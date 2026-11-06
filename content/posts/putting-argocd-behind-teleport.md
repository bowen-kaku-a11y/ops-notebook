---
title: "Putting Argo CD Behind Teleport: App Access With MFA on a Single-Node Cluster"
date: 2026-10-17
description: "Our Argo CD was only reachable through an SSH tunnel — safe, but it taught us nothing about gated application access. We deployed Teleport inside the same k3s cluster, GitOps-managed by Argo CD itself, and put the UI behind login + MFA on port 443. Three traps along the way: a Helm value that moved and failed silently, repo credentials matched by exact URL string, and a join token name that is not a secret."
---

# Putting Argo CD Behind Teleport: App Access With MFA on a Single-Node Cluster

Our practice cluster is a single k3s node on a small ARM cloud VM, and Argo CD already manages everything in it — including, recursively, itself. The one surface that stayed deliberately unglamorous was Argo CD's own UI: no Ingress, no NodePort, nothing public. Reaching it meant an SSH tunnel plus `kubectl port-forward`, every time.

That setup is safe, and it teaches you almost nothing. The skill worth practicing is gated application access: a real login with MFA in front of an admin UI, per-application hostnames, certificates that renew themselves, and an audit trail of who opened what. So we deployed Teleport *inside* the cluster it would protect, had Argo CD manage it like any other workload, and moved the UI behind it. The deployment itself was an evening's work. The three traps it contained are the article.

## Design: everything declarative, nothing on the host

Two decisions shaped everything else.

**In-cluster Helm deployment, not a host binary.** Teleport ships two charts: `teleport-cluster` (auth + proxy) and `teleport-kube-agent` (the app service that registers applications). Running them as ordinary workloads means the whole gateway is values files and Application objects. We ran the cluster chart in standalone mode with its multiplex listener, so the proxy owns host port 443 and speaks web, SSH, and app traffic on that one port — one firewall rule, opened through the same Terraform-managed security list workflow we already had.

**No CDN access product in front.** DNS is two plain A records — `teleport.example.com` and `*.teleport.example.com` — pointing at the VM, DNS-only, no proxying. Certificates come from Teleport's built-in ACME support (Let's Encrypt), per hostname. The point of the lab is to learn the gateway, not to rent one.

GitOps wiring followed the pattern the cluster already used: a root Application watches an `apps/` directory; one Application per Teleport chart; each chart Application is multi-source — the chart from the vendor's Helm repo, and a `$values` reference back into our own config repo so the values live next to everything else. Argo CD would, in other words, deploy the thing that was about to gate Argo CD. That circularity is fine as long as the break-glass path survives it — more on that in the last section.

## Freeing port 443: the value that moved

Port 443 on the node belonged to the cluster's bundled ingress (Traefik, via k3s's ServiceLB). Before Teleport's proxy could take it, that Service had to become ClusterIP. The bundled chart had been upgraded to the 40.x line, and in that line the Service type key moved:

```yaml
# chart 40.x: the type lives under service.spec —
# the old top-level key is silently ignored
service:
  spec:
    type: ClusterIP
```

We set the *old* key first. The Helm upgrade completed, Argo CD reported Synced, and the Service did not change at all. A renamed values key fails in the worst possible way in Helm: no error, no warning, just a value nobody reads anymore. The lesson generalizes beyond this chart — when an upgrade "succeeds" but the object doesn't move, diff the rendered output or read the object back; the exit code is not evidence. (The failure shape is worth learning to recognize on sight: [a silent no-op upgrade](/posts/the-rollout-that-never-happened/) looks exactly like success until you read the object back.)

Verifying the port was actually free had its own trap. `ss` showed no listener on 443 even *before* the change, because ServiceLB doesn't listen on the host at all: it programs a hostPort plus iptables DNAT rules that forward to the svclb pod. A port can look empty and still be fully occupied. The check that tells the truth is the NAT table:

```bash
iptables -t nat -S | grep 443   # CNI-DN rules pointing at the svclb pod IP
```

Only when those rules disappeared was 443 genuinely Teleport's to bind.

## GitOps wiring: credentials match by exact string

With the charts rendering, the agent Application stuck in ComparisonError:

```
ComparisonError: failed to get git client for repo
```

The cause is a matching rule that is easy to miss: Argo CD matches repository credentials by the **complete repoURL string**. Our config repo's credential — a deploy key — was registered under the repo's SSH URL. The chart source is a public Helm repository and needs no credential, so it never noticed. The `$values` source points back at our own config repo, so it does need the credential — and we had written it with the repo's HTTPS URL. HTTPS URL ≠ SSH URL, no credential matched, comparison failed.

The fix was one line: the values source now uses the SSH URL, byte-identical to the registered credential:

```yaml
sources:
  - repoURL: https://charts.releases.teleport.dev
    chart: teleport-kube-agent
    targetRevision: 18.11.4
    helm:
      valueFiles:
        - $values/teleport/values/agent.yaml
  - repoURL: git@git.example.com:platform/cluster-config.git
    ref: values
```

Nothing about the credential changed; only the spelling of the reference to it did.

## The agent join: a token name that is not a secret

The agent authenticates to the cluster using the Kubernetes join method: it presents its own pod ServiceAccount token, and the auth server checks it against a ProvisionToken resource whose rules name the allowed ServiceAccount. No join secret exists anywhere — not in git, not in a Secret.

It still failed, with `TokenName is required`. The chart's `joinParams` needs `tokenName` set explicitly even for delegated join methods:

```yaml
joinParams:
  method: kubernetes
  tokenName: "app-agent"
```

The name is a *selector*, not a credential: it tells the auth server which ProvisionToken's rules to validate the presented ServiceAccount token against. Ours is deliberately boring:

```yaml
kind: token
version: v2
metadata:
  name: app-agent
spec:
  roles: [App]
  join_method: kubernetes
  kubernetes:
    allow:
      - service_account: "teleport:teleport-kube-agent"
```

The proof in this scheme is the ServiceAccount token the platform issues to that one pod — the token *name* being public knowledge buys an attacker nothing.

## Verification

We verified from the outside in, the way a user (or an attacker) would meet the system:

| Check | Expected | Observed |
| --- | --- | --- |
| `curl -sI https://argocd.teleport.example.com` while logged out | Redirect to gateway login | `302` to the Teleport web login |
| TLS on `teleport.example.com` and the app subdomain | Valid public certificates | Let's Encrypt certs issued for both |
| `tctl apps ls` | The application is registered | `argocd` listed, pointing at the in-cluster Service |
| Audit log after a real login | Identity events recorded | TOTP login event and app session start, same user |
| Argo CD Applications | Everything converged | All Synced / Healthy, including the gateway's own apps |

Then the human test: one local user, roles `access` + `editor`, MFA enforced with TOTP and WebAuthn both permitted. Log in through the browser, pass the second factor, land in the Argo CD UI — which is served from a ClusterIP Service that no longer has any public path of its own.

## Two identity layers, on purpose

It is tempting to call the result double-login bureaucracy: Teleport makes you authenticate, then Argo CD makes you authenticate again. We kept both layers because they cover different gaps.

Argo CD's local admin account — the one this lab actually uses — has a password and no MFA. Teleport's local users get MFA enforced at the gateway, plus the audit log. So the outer layer supplies exactly the factor the inner one lacks, and its session is what an attacker has to defeat before the Argo CD login page is even reachable. The accounts are intentionally unrelated: a Teleport user is a *network* identity (which applications may you open), the Argo CD account is an *application* identity (what may you do once inside).

And the circularity from the design section has its answer: the Argo CD admin password stays as break-glass. If the gateway breaks — a bad values change, an expired cert, a wedged agent — the SSH tunnel and `port-forward` path still works, because that path never depended on Teleport. The gateway is the front door, not the only door; the only door is just no longer offered to the whole internet.

The same gateway has since taken on more tenants — a browser-based SSH client lives behind it too, and [debugging that integration](/posts/the-username-that-vanished/) produced a five-layer failure trail of its own. The Argo CD move came first, though, and it is still the template: one new Application object, one values file, zero new public ports beyond the gateway itself.

*The 443 firewall rule in this setup rides on the security list we had already adopted into Terraform; that adoption workflow is written up in [the Terraform adoption post](/posts/terraform-import-adopt-live-vm/). Storage housekeeping on the same cluster gets the same GitOps treatment — see [the orphaned PVCs post](/posts/orphaned-pvcs-ebs-volumes/) for the volumes nothing mounts anymore.*
