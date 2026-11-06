---
title: "The Kube Agent That Lived Next Door"
date: 2026-11-03
description: "A follow-up to the bundled k3s that couldn't create a thread — the post where I'd decided the cluster couldn't host the agent at all, and was wrong. It can; the chart just needed Unconfined. I ran the agent outside the cluster anyway, because it's the cleaner separation, and hit two traps worth more than the original mistake: a kubeconfig that was valid where it was born and useless once moved, and a NAS that couldn't reach Docker Hub. Both have the same moral — make the thing self-contained before you move it."
---

# The Kube Agent That Lived Next Door

[Last time](/posts/when-a-bundled-k3s-cant-run-threads/) I tried to register a QNAP Container Station k3s for Teleport Kubernetes Access by installing the agent *inside* the cluster, watched it `SIGABRT` on startup, and — as that post's correction lays out — talked myself into the wrong conclusion: that the runtime couldn't host *any* threaded image. It can. The crash came from the Helm chart's `seccompProfile: RuntimeDefault` meeting an old `libseccomp`; dropping the pod to `Unconfined` would have let the in-cluster agent run. But by the time I understood that, I'd already built the agent to run *outside* the cluster — and that turned out to be the better design regardless, so I kept it. This is that build.

The idea is small once you say it out loud: **the Teleport Kubernetes Service doesn't have to run inside the cluster it serves.** An in-pod agent is the convenient default — it inherits the pod's ServiceAccount and talks to `kubernetes.default.svc` — but there's a second mode where the agent runs anywhere and reaches the API server through a kubeconfig. So the agent goes into Container Station's Docker engine — where the SSH node has run happily for weeks, and where nothing has to be told to relax its seccomp — and points a kubeconfig at the k3s API.

## The shape

Three decisions fall out of the constraints:

- **Where it runs:** a plain container under Docker, not k3s. Same box, different runtime — one where nothing has to be told to loosen its seccomp.
- **How it reaches the API:** the k3s API server listens on the NAS loopback and isn't exposed to the LAN (a `curl` to `:6443` from another host times out — the firewall drops it). So the container runs with host networking and the kubeconfig points at `https://127.0.0.1:6443`. The k3s serving cert has `127.0.0.1` in its SAN list, so TLS still verifies — no `insecure-skip-tls-verify`.
- **How Teleport names it:** in kubeconfig mode the cluster's Teleport name comes from the kubeconfig **context name**, which may only contain alphanumerics, `.`, `_`, and `-`. So the context gets renamed to something clean, and the agent attaches a couple of labels (`cluster`, `env`) purely so the cluster is recognisable in `tsh kube ls`.

The config is unremarkable: `kubernetes_service` enabled, `kubeconfig_file` pointed at the mounted file, the usual outbound-only join over 443. No inbound ports on the NAS, same as the SSH node. On paper, done.

Then it crashed anyway, twice, for reasons that had nothing to do with Teleport.

## Trap one: a kubeconfig that only worked where it was born

First start, the agent aborted with:

```text
no Kubernetes credentials found; kubernetes_service requires either a
valid kubeconfig_file or to run inside of a Kubernetes pod
```

Which was maddening, because I had *tested* that kubeconfig. I'd dumped it with `kubectl config view --raw`, and `kubectl --kubeconfig ./that-file get nodes` worked fine. The file was obviously valid.

It was valid **on the host**. The kubeconfig that Container Station's `kubectl` wrapper emits refers to its client certificate and key *by file path*, not by embedded data:

```yaml
users:
- name: default
  user:
    client-certificate: /some/host/path/client.crt   # a path, not data
    client-key: /some/host/path/client.key
```

`kubectl config view --raw` faithfully preserves those paths. On the host the paths resolve, so every test passed. Mounted into the agent container, the paths point at nothing — the cert files aren't there — so Teleport correctly concludes it has no credentials. A kubeconfig is not automatically a portable artifact; it can be a set of *pointers* to things sitting next to it.

The fix is one flag: `--flatten`.

```bash
kubectl config view --raw --flatten > kubeconfig
```

`--flatten` reads those referenced files *now*, while the paths still resolve, and inlines their contents as `client-certificate-data` / `client-key-data`. The result is self-contained and travels. I added a check to the generator script so it refuses to hand over a kubeconfig that still has zero embedded credentials — because "it works on the box I made it on" is exactly the failure that doesn't announce itself.

This rhymes with the lesson from the first post. There it was a crash dump that read as a config error; here it's a kubeconfig that reads as portable and isn't. Both are *works-here-fails-there* bugs, and both hide behind a thing that superficially succeeded.

## Trap two: the registry you can't reach

Second start, the plan was to build the agent image from `ubuntu:22.04`. The NAS's Docker couldn't pull it:

```text
Get "https://registry-1.docker.io/v2/": ... Client.Timeout exceeded
```

This box reaches the internet when it wants a Teleport release, but the container registry is, for whatever local networking reason, out of reach. No base image, no build.

But I didn't need Docker Hub. There was already a container running on this exact box with the exact Teleport version I wanted: the SSH node from the first post. A running container *is* a filesystem with the binary in it, and you can snapshot it:

```bash
docker commit <running-ssh-agent> teleport-local:18.11.4
```

That's now a local image with Teleport baked in, no pull required. Run the kube agent from it, override the entrypoint to `teleport start`, mount the flattened kubeconfig and config over whatever the snapshot carried, and the mounts shadow the SSH node's own identity so the two agents never cross wires. In a sealed environment, **your running containers are your registry.** The images you need may already be on the box, one `commit` away.

## The role that was already there

One thing that *didn't* go wrong is worth noting, because it's the opposite of the usual Kubernetes-access yak-shave. I'd written a dedicated Teleport role for this cluster — label selector, `kubernetes_groups`, the works — and was about to apply it when I checked what my account already had. The existing `kube-access` role was `kubernetes_labels: '*':'*'` with `kubernetes_groups: ['system:masters']`: a deliberately broad "practice clusters, as admin" grant. A wildcard label selector matches a cluster the instant it registers, labels or none. The new cluster was already covered; the dedicated role would have been a second thing granting exactly what the first already granted, plus the small, real risk of locking myself out by fumbling a `--set-roles` list. I deleted it before it shipped. Least privilege is the right instinct, but a role that duplicates one you already carry isn't least privilege — it's just more surface.

## What I'm keeping from this

- **Make it self-contained before you move it.** A kubeconfig full of file paths, an image that only exists as a pull — both work until they leave home. `--flatten` the kubeconfig; `commit` the container. Portability is a property you add on purpose.
- **"It worked when I tested it" needs a second question: where?** The host and the container are different machines sharing a kernel. Test the artifact from where it will actually run, or at least from somewhere that can't see the host's filesystem.
- **The agent doesn't have to live in the thing it manages.** Even when it could — and here, with `Unconfined`, it could — running the control plane as a separate process is a cleaner separation: the audited door runs on a runtime I control and can upgrade, pointed at the cluster behind it. I arrived at it the embarrassing way, by first believing in-cluster was impossible; it's a good pattern even once you know it isn't.
- **Check what you already have before you grant more.** The best role change is sometimes the one you don't make.

The payoff is the same `kubectl` I'd have gotten from the in-cluster agent — MFA, full session audit, no inbound ports. The dev-tier question from the first post still stands, but on its real terms now: an end-of-life k8s 1.21 and the seccomp trade-off, not the imaginary wall I'd first put in front of it. Either way, the audited door is open.
