---
title: "Letting Atlantis Manage Teleport Without Pretending It's Safe"
date: 2026-11-04
description: "Moving my home lab's Teleport roles into Terraform, driven by Atlantis, with a bot that joins using a projected ServiceAccount token instead of a stored secret. The plumbing took an afternoon. The harder part was being honest about what the least-privilege role actually protects: my first draft claimed there were no secrets involved and that a deny list kept the bot from locking me out, and both claims were wrong."
---

# Letting Atlantis Manage Teleport Without Pretending It's Safe

My lab's Teleport roles had been created by hand with `tctl`, one at a time, over a few weeks. The production setup I'm copying keeps them in a Terraform repo and applies them from pull requests. I wanted the same here: Teleport roles as code, planned by Atlantis on every PR, applied only when I comment.

The work came in two phases. Phase 0 gave Atlantis an identity in Teleport. Phase 1 adopted one existing role into Terraform without changing it. Both are done. Most of what I learned is about where the safety boundary really sits, which wasn't where I first wrote it down.

## 1. Decide who owns what before writing any HCL

Not everything in Teleport belongs in the same place, and production already shows the split:

- **Roles**, and later **join tokens**, go into a new `teleport/` Terraform workspace, applied by Atlantis. Tokens wait for phase 2: for now Atlantis's role can't touch them at all (section 3 explains why).
- **Apps** don't go into Terraform at all. Each cluster's Teleport agent declares its apps in its Helm values, and Argo CD deploys them. Production does the same.
- **Atlantis's own Teleport identity** (its bot and its join token) goes into a `teleport-bootstrap/` folder that Atlantis never sees. I apply it by hand with `tctl`.

The last rule copies one I already had for cloud infrastructure: the folder that defines the machine and identity Atlantis runs on is not an Atlantis project. If Atlantis can apply the thing that lets it log in, one bad apply can remove its own access, and then nothing can apply the fix.

## 2. "No stored secret" is not the same as "no secret"

Atlantis runs on a different cluster from the Teleport auth server, so the bot joins with Teleport's `kubernetes` join method in `static_jwks` mode:

- Atlantis's pod gets a projected ServiceAccount token whose audience is the Teleport cluster name.
- The join token in Teleport embeds the public half of that cluster's ServiceAccount signing key.
- Teleport checks the signature offline and issues the bot a certificate.

Nothing secret is stored in git or in a Kubernetes Secret.

My first draft summed that up as "no secrets involved". A review caught it. The projected token file **is** a credential: it's valid for ten minutes and readable by every Terraform run in that pod. A `terraform plan` runs code from the PR branch, so any PR to the infra repo could read the file and use the bot's identity. It wouldn't need a merge or an apply, just a plan.

In my lab that limits it to "people who can open PRs on a private repo with one collaborator". That's acceptable, but only once it's written down as a risk instead of a guarantee. I wrote it down and hardened the rest:

- The pod no longer mounts its Kubernetes API token.
- Bot certificates last at most 30 minutes.
- The `mergeable` apply requirement moved into the Atlantis server config, so a PR can't remove it (section 4 covers how little it actually guarantees).

## 3. A deny list can't protect roles by name

The built-in `terraform-provider` role can manage about forty resource types, including users, bots, locks, auth settings and tokens. I replaced it with a custom role, `terraform-atlantis`:

```yaml
spec:
  options:
    max_session_ttl: 30m
  allow:
    rules:
      - resources: [role]
        verbs: [list, create, read, update, delete]
  deny:
    rules:
      - resources: [token, scoped_token]   # reading a token can leak a join secret
        verbs: ['*']
      - resources: [user, bot, lock, cluster_auth_preference, github, oidc, saml, ...]
        verbs: [create, update, delete]
```

My first plan was to also deny edits to `editor`, to the bot's own role and to my user. That turned out to be impossible. A role's `where` clause can't protect specific roles by name. Any identity that can update roles can grant itself more power, so **being able to edit roles is being an admin**. The role's comment now says that plainly. The deny list stops a careless Terraform change from deleting my user or a connector. It won't stop someone who sets out to escalate. What stops them is that only I can open PRs, plus a way in that doesn't depend on Teleport: SSH to the host and run `tctl` inside the auth pod.

One more correction came from review. I had originally denied every verb on the protected kinds. Deny rules override allow rules, including the read access Teleport grants by default, so the provider's first plan could have failed while reading cluster settings. The fix was to deny only writes and keep reads. Tokens are the exception: reading one leaks its name, and for `token`-method joins the name is the join secret.

## 4. Move the apply requirement to the server, and know what it's worth

The `mergeable` requirement for apply used to live in the repo's `atlantis.yaml`, which means a PR can change it. It now lives in the server-side repo config, with no `allowed_overrides`. I set `import_requirements` the same way. That one only gates the `atlantis import` command, which is disabled by default anyway. It doesn't cover `import {}` blocks, which go through a normal `atlantis apply`, and it doesn't cover `atlantis state rm`.

Then the honest part. The repo is private on a free GitHub plan, so there's no branch protection, and without required reviews `mergeable` only means "no merge conflicts". Any conflict-free PR can be applied before it's merged. The gate that actually holds is that I'm the only one who comments `atlantis apply`. Moving the requirement to the server stops a PR from quietly deleting it, and that's all it does.

There was a side effect I didn't expect. Once the server owns those keys, a repo `atlantis.yaml` that still sets them **fails to parse**. That broke every open PR branched before the change until I rebased them. If you move a setting from repo to server, check which open branches still carry the old file.

## 5. Adopt a resource, don't recreate it

Phase 1 adopted the existing `chrome-only` role. The steps:

1. Add an `import {}` block.
2. Run `terraform plan -generate-config-out=generated.tf`, so Terraform writes the HCL from the live role instead of me typing it.
3. Delete `metadata.revision`, which the server sets on every write, and add `lifecycle { ignore_changes = [metadata.revision] }`.

The generated file is long. It writes every server default as an empty map, `cert_format = "standard"`, `max_session_ttl = "12h0m0s"` and so on. I wanted to trim it. Don't: these attributes aren't computed, so once you delete them the plan will try to remove them from the server on every run. The new role I'm writing for Argo CD copies the same full set of defaults for the same reason.

One difference remains. The provider changes `allow.kubernetes_resources` to `[]`, and neither `null` nor `[]` in HCL avoids it. The import plan came back `1 to import, 1 to change` instead of a clean import. After the apply, a fresh plan said `No changes`, and it has stayed that way. Knowing it's a one-time change made it easy to approve.

## 6. A webhook that arrives doesn't always mean a plan

Two PRs showed a successful webhook delivery in GitHub and no Atlantis comment. Atlantis was fine. Its log said `0 projects are to be planned based on their when_modified config`. Those PRs only changed host config files, so there was nothing to plan, and Atlantis only marked the checks green.

The opposite case caught me too. The `root` project lives at `.`, and the default `when_modified` matches `**/*.tf*`. Every change under `teleport/` also planned `root` and took its lock. Setting `when_modified: ["*.tf", ".terraform.lock.hcl"]` on root limits it to its own directory.

## What I'm keeping from this

- **Write the risk next to the control.** My two wrong claims, "no secrets" and "the deny list protects me", came from describing what the setup does instead of what an attacker could do with it. Each control now has a comment saying what it stops and what it doesn't.
- **Keep the identity bootstrap outside the system it bootstraps.** Atlantis can change roles but not its own bot or token. Someone still has to apply those by hand, and that's on purpose.
- **Adopt with generated config and keep the defaults.** The noisy file is what the server actually stores.
- **Have something review your own explanations, not just your code.** Every PR here got an automatic review, and the most useful findings were about wrong statements in comments and READMEs.
