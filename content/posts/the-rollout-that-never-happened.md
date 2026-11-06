---
title: "The Rollout That Never Happened: Helm Upgraded, the Pods Didn't"
date: 2026-10-03
description: "I rebuilt the image, loaded it, and ran helm upgrade. Everything reported success — and the pod kept serving the old binary for eight days. A fixed image tag makes change invisible to Kubernetes; here's the mechanism and the fix."
---

# The Rollout That Never Happened: Helm Upgraded, the Pods Didn't

I shipped a new UI for a small internal dashboard I run on a local kind cluster. The sequence felt airtight:

1. Rebuild the Docker image with the new frontend.
2. Load it into kind with `kind load docker-image`.
3. Run `helm upgrade --install` and watch it complete.
4. Open the dashboard.

The UI was the old one. Not cached-in-the-browser old — I checked. `kubectl get pods` told the real story: the same single pod, **eight days old**, happily running the binary from the first deploy. Every command I ran had succeeded. Nothing had ever rolled.

## Why nothing rolled

Kubernetes rolls a Deployment when — and only when — the **[pod template changes](/posts/redpanda-operator-decommission-controller-pvc/)**. Helm renders your chart, compares the result with what's live, and applies the diff. My chart pinned the image like this:

```yaml
image:
  repository: my-dashboard
  tag: dev
  pullPolicy: IfNotPresent
```

Before the rebuild, the rendered template said `my-dashboard:dev`. After the rebuild, it said... `my-dashboard:dev`. Byte-identical. Helm truthfully reported a successful upgrade of a release whose spec had not changed, Kubernetes truthfully changed nothing, and the old pod — which was healthy and matched the spec — had no reason to die.

The uncomfortable part is what a tag actually is: **a mutable label, not an identity.** The string `dev` pointed at one image on Monday and a different image on Friday. Everything in the cluster reasons about strings; the fact that the content behind the string changed is information that exists only on your build machine (and, after `kind load`, in the node's image cache). Nothing carries it into the pod template, so as far as the scheduler is concerned, Friday's deploy never happened.

`IfNotPresent` makes this quieter still. Its whole job is "don't fetch if you already have something with this name" — a sensible default that also guarantees a stale cache is never questioned. In my case the missing piece wasn't even the pull; `kind load` had already overwritten the node's cached image, so a *new* pod would have run the new code. There was simply never a new pod.

## The fix: put the image's identity in the pod template

The fix is to smuggle the one fact Kubernetes needs — "the content changed" — into the rendered template. The image's content ID is exactly that fact:

```bash
IMAGE_ID="$(docker image inspect my-dashboard:dev --format '{{.Id}}')"
helm upgrade --install my-dashboard ./chart \
  --set image.tag=dev \
  --set "imageID=${IMAGE_ID}"
```

And in the Deployment template, render it as an annotation:

```yaml
template:
  metadata:
    labels:
      app: my-dashboard
    {{- if .Values.imageID }}
    annotations:
      example.dev/image-id: {{ .Values.imageID | quote }}
    {{- end }}
```

Now the workflow is honest again. Same content → same ID → same template → no rollout, correctly. New build → new ID → annotation changes → pod template hash changes → Kubernetes rolls the pods, every time, while the human-friendly tag stays `dev`. My deploy script now computes the ID and passes it automatically, so there's no extra step to forget.

This is the same family as the classic `checksum/config` annotation trick for ConfigMaps and Secrets — ConfigMaps have the identical problem (content changes, template doesn't, pods never restart), and the ecosystem's answer was the same: hash the content into the template.

## The alternatives, honestly assessed

- **Unique tag per build** (git SHA, build number): the "proper" answer in CI with a registry, and what I'd do in production. Locally, with `kind load` and no registry in the loop, it means editing values on every iteration — which is how I talked myself into the fixed `dev` tag in the first place.
- **`kubectl rollout restart`**: works, and it's a fine manual hammer. It's also a step that exists only in your memory, which is to say it will eventually not happen — silently, exactly like this bug.
- **The image-ID annotation**: keeps the local workflow one-command, makes rollouts a property of the system instead of the operator. That's the one I kept.

## Verifying the fix

With the fix merged, I redeployed the same way as before — rebuild, `kind load`, the deploy script (which now passes the image ID on its own). This time the eight-day-old pod was replaced: a new pod came up running the new build, and the endpoint the old binary didn't have — the one that had been returning 404 all week — now serves. Just as important is the negative case: redeploying again with no changes leaves the pod alone, because the image ID, and therefore the pod template, is unchanged. Rollouts now happen exactly when the content changes — no sooner, no later.

## The lesson I keep relearning

"Deployed successfully" means *the spec was applied successfully*. If your change doesn't alter the spec, you have deployed nothing, and every health check downstream will happily pass against the old version — health checks verify that pods are up, not that they're new. Mutable tags hide change. Whatever you ship, make the change visible to the thing that schedules it.
