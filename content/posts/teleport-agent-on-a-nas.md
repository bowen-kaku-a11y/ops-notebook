---
title: "A Teleport SSH Node on a NAS, Deployed Through a GUI: Four Traps"
date: 2026-10-30
description: "Adding a Teleport SSH target is usually an evening's work: an agent dials out over 443 and the box is reachable through the reverse tunnel. This time the box was a consumer NAS administered only through its web console — no host shell, no docker build, and a daemon that couldn't reach Docker Hub. Four traps followed: a GUI that hides the host, an image pull that timed out while egress worked fine, a dollar sign that Compose ate before the container ever saw it, and two agents sharing one node name."
---

# A Teleport SSH Node on a NAS, Deployed Through a GUI: Four Traps

We already run a small Teleport cluster in our k3s lab, and [the UI for everything else lives behind it](/posts/putting-argocd-behind-teleport/). Adding a new SSH target to that cluster is normally trivial: drop an agent on the box, it dials out to the proxy over 443, and `tsh ssh` reaches it through the reverse tunnel. No inbound ports, no public IP, and the login, MFA, and audit trail come for free.

This time the box was a consumer NAS at home, administered only through its web console. No shell on the appliance, no `docker build`, no `ssh` into the firmware OS. That one constraint — plus a NAS that turned out to be unable to pull from Docker Hub — turned the usual evening's job into four traps worth writing down. None of them, it's worth saying up front, are about Teleport.

## Design: a container, and knowing what you reach

Two decisions framed the rest.

**Run the agent in a container, and reach the container.** Teleport ships its own SSH server (`ssh_service`), so the container needs no `sshd` — just the `teleport` binary dialing out. `tsh ssh root@<node>` then lands you *inside that container*, not in the appliance's firmware OS. That's deliberate. The appliance OS is not an ordinary Linux, and whatever you install there tends not to survive a firmware update; a container with a named volume is the stable, reproducible unit. When the container needs to touch files on the NAS, you bind-mount them in.

**Be self-contained, because there is no host shell.** With only a web console there is no `docker build` and nowhere to drop a config file for a bind mount. So everything lives in one Compose file pasted into the GUI: the image is one already present on the box, the Teleport config is written by the container's start command, the host identity (certificates) lives in a named volume, and the short-lived join token arrives as an environment variable. The skeleton:

```yaml
services:
  node:
    image: ubuntu:jammy-20221130
    restart: unless-stopped
    environment:
      JOIN_TOKEN: "<paste a node join token>"
    entrypoint: ["bash", "-lc"]
    command:
      - |
        cat > /etc/teleport.yaml <<'CFG'
        version: v3
        teleport:
          nodename: home-node
          proxy_server: teleport.example.com:443
          join_params: { method: token, token_name: /var/lib/teleport/join-token }
        ssh_service: { enabled: true, labels: { env: nas } }
        auth_service: { enabled: false }
        proxy_service: { enabled: false }
        CFG
        # install teleport, write the token, then exec it — details below
    volumes:
      - teleport-data:/var/lib/teleport
volumes:
  teleport-data:
```

Because the base image is minimal and recreated fresh, the start command installs the tools and the `teleport` binary on every boot — the price of not baking a custom image, which would need the `docker build` we don't have. The certificates persist in `teleport-data`, so a rebuild rejoins from saved state rather than from scratch.

## Trap 1: the image pull that timed out while everything else worked

**Symptom.** Creating the application fails at once, before anything runs:

```
failed to pull image ubuntu:22.04
[Get "https://registry-1.docker.io/v2/": request canceled while waiting
 for connection (Client.Timeout exceeded while awaiting headers)]
```

**Why.** The appliance's container daemon could not reach `registry-1.docker.io`. What made this confusing is that minutes earlier, inside a container on the same box, `apt-get` and a `curl | bash` vendor install had both worked fine. Outbound to package mirrors and CDNs was healthy; only the registry pull hung. Registry reachability is a separate network path from in-container egress — one working tells you nothing about the other.

**Fix.** We didn't need the registry. The box already had an Ubuntu image from earlier use, so point the service at the exact tag that is present and tell Compose not to call out when the image is already local:

```yaml
    image: ubuntu:jammy-20221130   # a tag already in the local image list
    pull_policy: missing           # already local -> don't pull
```

The tag has to match byte-for-byte. `ubuntu:22.04` and `ubuntu:jammy-20221130` are the same bits under two names, and `pull_policy: missing` keys on the *name*, not the content — ask for a name the box doesn't have and it goes back to Docker Hub and hangs again. Read the appliance's image list and copy the tag verbatim.

## Trap 2: the dollar sign Compose ate

This is the one that cost the most time.

**Symptom.** The container runs, Teleport starts, and then loops forever on:

```
Failed to establish connection to cluster ...
open /var/lib/teleport/join-token: no such file or directory
```

The config points `token_name` at that file and the file is not there. Yet the start command plainly writes it, from an environment variable that is plainly set in the Compose file.

**Why.** Compose runs its own variable interpolation over the entire file *before* the container exists. A bare `$JOIN_TOKEN` inside the service's `command` is never handed to the container's shell: Compose expands it first, against *its own* environment — the shell that launched Compose, or a `.env` file — finds nothing, and substitutes the empty string. By the time the in-container shell runs, the reference is already gone. The `environment:` block did put `JOIN_TOKEN` *inside* the container, but the command meant to read it had been rewritten to read nothing. So the guard `[ -n "$JOIN_TOKEN" ]` was testing an empty string, the token file was never written, and Teleport had nothing to join with.

**Fix.** Escape the literal dollar as `$$`, so Compose passes a single `$` through and the container's shell does the expansion:

```yaml
    command:
      - |
        if [ -n "$$JOIN_TOKEN" ]; then
          printf '%s' "$$JOIN_TOKEN" > /var/lib/teleport/join-token
        fi
        exec teleport start -c /etc/teleport.yaml
```

The same rule applies to every `$` you intend for the container and not for Compose. The tell is a value that is mysteriously empty at runtime even though `environment:` sets it — go looking for an un-doubled `$` in `command` or `entrypoint`. (An aside: the join token here is not a secret worth guarding — it is single-use and short-lived, [which is the point about Teleport join tokens](/posts/putting-argocd-behind-teleport/). The bug was never that the token leaked; it was that it never arrived.)

## Trap 3: two agents, one name

**Symptom.** `tsh ssh root@home-node` fails with an ambiguity error, and `tsh ls` shows two rows with the identical name.

**Why.** We had first brought the agent up by hand in a throwaway container to prove the path, then rebuilt it properly as the Compose application — same node name, same labels — and for a while both were running. Teleport identifies a node by a host UUID written into its data directory on first join, not by its node name. The hand-run container and the Compose container each generated their own UUID (the Compose one started from an empty named volume), so the cluster saw two distinct nodes that merely shared a name. A name matching more than one node can't be resolved, so the connection is refused rather than guessed.

**Fix.** Don't run two. Bring up the replacement, confirm it joined, then stop the original; a stopped agent stops heartbeating and its node record ages out on its own within minutes (or `tctl nodes rm <uuid>` to drop it now — by UUID, since the name is ambiguous). The part that bites is cleanup: *stopping* the container is not enough if the appliance is set to auto-start it on boot. Disable that, or delete the throwaway container outright, or it comes back after the next reboot and re-creates the collision.

## What they had in common

Four failures, and the agent behaved correctly through all of them. The breaks were in the layer we were driving it through: a GUI that hides the host and takes `docker build` off the table, a registry path that fails independently of egress, a Compose interpolation pass that runs before the shell, and an identity model that counts UUIDs rather than names. Deploying through an appliance console removes the shell you would normally debug in, so the debugging moves to reading container logs and knowing where each layer draws its line.

The payoff is the same as for any other Teleport target: one more box reachable with a login, MFA, and an audit trail, over a single outbound 443, with nothing at all listening on the way in.
