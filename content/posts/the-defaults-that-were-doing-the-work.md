---
title: "The Defaults That Were Doing the Work"
date: 2026-11-03
description: "Standing up a self-managed modern k3s, a Teleport agent, and ArgoCD on a locked-down NAS — one that can't reach Docker Hub, boots a vendor k3s frozen at an EOL version, and has a root filesystem the size of a postage stamp. Nothing went wrong that was hard, exactly. What went wrong was that every convenience I'd stopped noticing — a reachable registry, a roomy /root, a project that creates itself, an identity that survives a config change, a Helm upgrade that just connects — turned out to be load-bearing, and the environment removed them one at a time."
---

# The Defaults That Were Doing the Work

The plan was dull on paper: run a current k3s on a NAS, register it with Teleport for audited `kubectl`, install ArgoCD, and expose one internal app through the tunnel. I'd done each of these before. What I hadn't done was all of them on a box this *constrained*: a vendor-managed k3s locked at an end-of-life version with no upgrade path, a network that can't reach Docker Hub, a root filesystem measured in megabytes, and a reverse-tunnel as the only way in.

None of the problems that followed were clever. Every one was a **default I'd been relying on without knowing it** — a small convenience quietly doing real work — and this environment took them away one by one. That's the whole post: a list of invisible dependencies, made visible by their absence.

## The cluster you're given vs. the cluster you run

First default to go: *the managed thing is a good starting point.* The NAS ships a one-click Kubernetes — a single-node k3s inside its container platform. But it's pinned to an EOL release, its version is bolted to the firmware, and there's no upgrade button. Its bundled container runtime is old enough that its default seccomp profile rejects a syscall modern glibc needs, so hardened workloads crash on it (a story I [told separately](/posts/when-a-bundled-k3s-cant-run-threads/)).

The move that unblocked everything was to stop treating the managed cluster as the foundation. k3s is just a binary in a container; I ran my *own* current version as a second container next to the managed one — host port 6444 instead of 6443, its own data directory, `--restart unless-stopped`. Modern runtime, modern API, a version I control. The managed cluster still sits there, untouched, doing whatever the vendor wants. The lesson is almost a reflex now: **when a managed component is the thing blocking you, the fastest path is often to run the unmanaged version beside it rather than to fight the one you were given.**

## The registry you can reach

I ran the new k3s and watched its own system pods — the pause container, CoreDNS — fail to pull. `dial tcp … i/o timeout` to the Docker Hub registry. This network simply can't reach it.

Two sub-defaults died here. The obvious one: *images come from where the manifest says.* They don't, if you can't route there; you point the runtime at a mirror that proxies the blocked registries. k3s takes a `registries.yaml` for exactly this. The *non*-obvious one, which cost me a wasted restart: **`registries.yaml` is read once, at startup.** I edited it on a running cluster, re-checked, and saw the identical failure with the identical timestamp — because the running k3s had never re-read the file. It wasn't a stale cache or a wrong endpoint; it was that the config only takes effect on boot. And one more layer: a single mirror line isn't enough, because the images don't all live in one place. The cluster's own components, the GitOps controller's image, the app's image — Docker Hub, a container registry, a couple of others — each needed its own mirror entry. I added them all up front and restarted once, instead of discovering each blocked registry the hard way as a new `ImagePullBackOff` three steps later.

## The filesystem with room to spare

The Teleport agent is a Helm chart. `helm upgrade` got partway and died: `no space left on device`, writing to `~/.cache/helm`. On this box `/root` is a tiny read-mostly partition; Helm's default cache, config, and data directories all live under `$HOME`, and the chart download alone overran it.

*Your home directory has space* is a default you never think about until a appliance OS puts `$HOME` on something minuscule. Helm respects `HELM_CACHE_HOME`, `HELM_CONFIG_HOME`, `HELM_DATA_HOME`, and `TMPDIR`; I pointed all four at the big data volume and the upgrade went through. Nothing deep — but it's the kind of thing that stalls you precisely because it's so far from where you're looking.

## The chart that tolerates an empty list

The agent was meant to serve two roles: Kubernetes access and application access, one agent, both doors — mirroring the production pattern. So I set `roles: "kube,app"` and left the app list empty, planning to fill it later. The chart refused to render: *app service is enabled, but no application source is enabled.*

The default I'd assumed: *you can turn a feature on now and configure it later.* Not this one — enabling the `app` role with an empty `apps` list is a contradiction the chart catches at template time. The fix was sequencing, not config: install with just the `kube` role now, and flip to `kube,app` **in the same change that adds the first app**. A half-enabled feature isn't a smaller version of the feature; sometimes it's an invalid state.

## The project that creates itself

ArgoCD installed cleanly in its headless "Core" flavor — controller, repo-server, Redis, no API server, no web UI. I applied the root app-of-apps and it went straight to `InvalidSpecError: project default does not exist`.

In a normal ArgoCD install, the `default` AppProject is created for you — by the API server, on startup. Core has no API server. So the one piece of bootstrap state that *every* Application references, and that you never think to create because it's always just *there*, isn't there. One permissive `AppProject` manifest later, the root app went `Synced / Healthy`. **The headless build doesn't just drop the UI; it drops everything the server-that-isn't-running would have done on your behalf** — and that includes invisible setup you've never had to name.

## The identity that outlives a config change

Flipping the agent from `kube` to `kube,app` should have been a one-line values change and an upgrade. Instead the pod came back `0/1`, logs looping on `the token is expired or not found`.

Here the buried default is subtle: *an agent's stored identity covers whatever you reconfigure it to do.* It doesn't. The agent had joined as `kube` only, and its cached identity was scoped to that. Adding the `app` role meant it needed a **new** identity, which meant re-joining, which meant a valid join token — and the token it was presenting was stale. The real twist: when I checked the control plane, the token I'd issued was perfectly valid, with time to spare. The invalid one was the *old* token still sitting in the Kubernetes Secret the agent reads. The lesson that generalizes past Teleport: when something authenticates with a credential pulled from a Secret, "the credential is bad" and "the Secret holds the wrong credential" look identical in the logs, and they are not the same bug. I stopped re-issuing tokens and started checking what was actually in the Secret.

## The orchestration that assumes a direct line

The clean way to apply the role change was `helm upgrade` through the Teleport-proxied `kubectl` context — manage the cluster the same audited way I reach it. Helm errored with a Teleport routing failure: `missing port in address … .kube.proxy …`. Helm's multi-step orchestration — its discovery calls, its three-way merge, its watches — leans on connection behavior the kube proxy doesn't reproduce, even though plain `kubectl` sails through the same path.

The default that broke: *the tool I drive the cluster with works wherever `kubectl` works.* Helm is `kubectl` plus a lot of orchestration, and the orchestration is where the leak was. The escape was to drop down a level. Single-resource operations — patch the agent's ConfigMap to enable the app service, overwrite the token Secret, delete the pod to force a clean re-join — all go through the proxy fine, because each is one ordinary API call. It cost me Helm's bookkeeping (the release record now disagrees with the live config until the next real upgrade reconciles it), but it got a working agent, and it's a reminder that **the layer you automate with is not always the layer that survives a hostile transport. When the high-level tool can't connect, the primitives underneath it often still can.**

## What I'm keeping from this

- **A "convenience" is just work someone else was doing for you.** A reachable registry, a roomy `$HOME`, a self-created default project, an identity that spans a role change, a Helm upgrade that connects — none of these are features you chose. They're defaults, and a constrained environment is a machine for revealing which ones were load-bearing.
- **Inventory the implicit dependencies before the environment does it for you.** Every stall above was something I'd never had to name. The ones that cost the most time were the quietest: a config file read only at boot, a token sitting stale in a Secret.
- **Prefer tools that degrade to primitives you can drive by hand.** The whole thing got unstuck when Helm-through-the-tunnel gave way to three `kubectl` calls. The high-level path is faster when it works; keep the low-level one in reach for when it doesn't.
- **"Managed," "headless," and "locked-down" each remove conveniences you didn't order removed.** That's not a reason to avoid them — it's a reason to expect the removal and to recognize the symptom when a default you never thought about turns out to be missing.
