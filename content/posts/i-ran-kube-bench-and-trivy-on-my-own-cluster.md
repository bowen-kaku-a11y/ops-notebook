---
title: "I Ran kube-bench and Trivy on My Own Cluster: 17 Criticals, 980 Highs — and Why the Real List Was Six Items"
date: 2026-10-18
description: "My practice cluster quietly grew production-shaped — GitOps, an authenticating proxy, web tools — so I audited it with the tools an auditor would use: kube-bench against the CIS k3s profile, and Trivy on every running image. The scanners reported 17 criticals and 980 highs. Sorted by reachability, the list that actually mattered was six items, and the first fix shipped the same weekend."
---

# I Ran kube-bench and Trivy on My Own Cluster: 17 Criticals, 980 Highs — and Why the Real List Was Six Items

My lab cluster started life as a scratchpad: one k3s node on a small ARM cloud VM (2 vCPU, 12 GB) in a US region, somewhere to try things before touching anything that matters. Then it accumulated real services. A GitOps controller that manages everything, including itself. An authenticating proxy with MFA in front of every admin surface. A browser and an SSH client running as pods. A demo app. At some point "practice cluster" stopped being an honest description — it had become production-shaped, with production-shaped ways to be wrong.

So I audited it, with the same tools an auditor would bring: kube-bench against the CIS benchmark profile for k3s, Trivy against every image actually running, plus a pass over the API with `kubectl` and `jq` for the things scanners phrase badly. This post is what survived triage — a much shorter list than what the scanners printed.

## Method: install nothing in the cluster

The one deliberate choice: no new resident software. A scanner you install is one more workload to patch, and on a 2-core node it competes with the things it is scanning. Both tools ran one-shot from the host instead.

kube-bench ran as a local binary against the node it lives on, using the `k3s-cis-1.9` profile. Trivy ran as a host binary too. First, enumerate what is actually running — not what the manifests claim, what the API says is up:

```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{range .spec.containers[*]}{.image}{"\n"}{end}{end}' | sort -u
```

Thirteen images. Each went through Trivy at HIGH and CRITICAL only:

```bash
trivy image --severity HIGH,CRITICAL <image>
```

The alternative is trivy-operator: continuous scanning, results stored as `VulnerabilityReport` objects, Prometheus metrics, new pods scanned as they appear. It is the right answer at fleet scale. On one small node it is a resident scanner solving a problem a monthly one-shot re-run solves equally well — the images here change only when I change them. The audit is a diff I re-run, not a daemon I feed.

## The headline numbers lie politely

Raw totals: kube-bench reported **17 PASS, 4 FAIL, 33 WARN**. Trivy reported **17 CRITICAL and 980 HIGH** across the 13 images. Nine hundred and eighty sounds like a five-alarm fire. It is an inventory, not a risk ranking — and inventories become rankings only after you sort by reachability.

| Image | CRITICAL | HIGH |
| --- | ---: | ---: |
| Browser desktop (linuxserver/chromium) | 6 | 450 |
| Dex, the GitOps controller's OIDC sidecar | 5 | 141 |
| Argo CD itself | 2 | 137 |
| WeTTY (browser SSH client) | 2 | 62 |
| Redis (bundled with Argo CD) | 2 | 21 |
| Teleport (distroless) | 0 | 17 |
| k3s bundled system images | 0 | 6–88 |
| A small static-site image I build myself | 0 | 5 |

One line does most of the damage: the browser pod is a *full desktop environment* — window system, fonts, media libraries — and it contributes 450 of the 980 HIGHs by itself. Severity scores count vulnerable code, not reachable code. That browser serves exactly one user, me, through an authenticating proxy with MFA; nothing in this cluster is published raw. The cloud firewall in front of the VM opens exactly two ports, 22 and 443, and 443 belongs to the proxy. (The proxy setup, and why everything sits behind it, is written up in [Putting Argo CD Behind Teleport](/posts/putting-argocd-behind-teleport/).)

This is the rule-review instinct from firewall and WAF work, applied to scanner output: exposure first, severity second. A CRITICAL in a package no network path reaches is a patch note. A HIGH on something the internet can touch is a page. Sorted that way, the real list was six items.

## The six findings that survived triage

Worth saying first: the floor was solid. No privileged containers, no pods on the host network or host PID/IPC namespaces, no hostPath abuse, control-plane ports bound to localhost, sensitive files at `0600`. The findings below are defaults that were never tightened, not holes that were punched.

**1. Secrets are stored unencrypted at rest.** On a single-node k3s install the embedded datastore is SQLite, and secret encryption was never enabled:

```bash
$ k3s secrets-encrypt status
Encryption Status: Disabled, no configuration file found
```

Every Secret in the cluster sits in that file in plaintext, protected by exactly one control: file permissions. Enabling encryption means a restart plus re-encrypting existing Secrets — a maintenance window, not an edit — so this one is scheduled as its own job, and written up separately when it happens.

**2. Three apps run as the default ServiceAccount, and tokens are mounted almost everywhere.** The demo site, the browser pod, and the SSH client (the [WeTTY gateway from an earlier post](/posts/the-username-that-vanished/)) all ran as `default`, and counting pods whose token automount was not explicitly off gave 16 of 20:

```bash
kubectl get pods -A -o json | jq '[.items[] | select(.spec.automountServiceAccountToken != false)] | length'
```

None of these three apps ever calls the Kubernetes API. Every mounted token is pure attack surface: compromise the pod, inherit an API credential it never needed. These were kube-bench's FAILs 5.1.5 and 5.1.6, and they were the cheapest real fix on the list — see the end of this post.

**3. No Pod Security Standards labels anywhere.** All eight namespaces lack `pod-security.kubernetes.io/enforce` labels. Nothing in the cluster would stop a future manifest — mine, or a chart's — from asking for a privileged pod. This is the root cause behind kube-bench's whole 5.2.x WARN family: the admission gate exists in the API server, it is simply not switched on.

**4. NetworkPolicy exists only where the vendor shipped it.** `kubectl get networkpolicies -A` returns seven rows, all in the GitOps controller's namespace, all shipped by the controller itself. Every other namespace — tools, the proxy, the demo, default — has none, so pod-to-pod traffic between namespaces is unrestricted. k3s ships an embedded policy controller that *would* enforce policies; what is missing is the policies.

**5. Most system components have no resource limits.** The GitOps family, the proxy, the bundled ingress — container after container with `resources.limits` unset. On a 12 GB single node this is an availability finding, not a footnote: one runaway container can OOM the node and take the entire lab, proxy and all, down with it.

**6. The image patch rhythm is two drifting tags and one upgrade train.** Two images track `:latest` (the browser and WeTTY), so their contents change whenever upstream republishes. Everything else is pinned — and here is the arithmetic that defused the CRITICAL count: 9 of the 17 CRITICALs (Dex's five, Redis's two, Argo CD's two, mostly one OpenSSL and one Go stdlib issue between them) belong to the GitOps controller's release train and clear with a single controller upgrade. Not thirteen emergencies. One upgrade, two tags to pin, and a browser image whose six CRITICALs had no upstream fix available at scan time — noted, watched, and accepted until there is one.

## Findings I deliberately did not fix

An audit you cannot argue with is a checklist, not an assessment. Three results I read, verified, and closed as *accepted*:

- **Wildcard permissions (FAIL 5.1.3).** The ClusterRoles with `*` verbs and resources are the GitOps controller's own shipped RBAC, the built-in system controllers, and the storage provisioner. That breadth is how those components are designed to work. The finding is real on paper and meaningless in context — the fix would be forking vendor RBAC.
- **Kubelet read-only port (FAIL 4.2.4).** The benchmark wants `--read-only-port=0` set explicitly. It is not set — and the port itself answers connection refused. I probed it rather than trusting either the config or the scanner. Closed port, open paperwork.
- **`system:masters` (WARN 5.1.7).** The stock admin kubeconfig on a single-node install authenticates as `system:masters`. Noted — and the API port it talks to is not reachable from the public internet, per the firewall above.

Recording *why* each of these is accepted matters more than the acceptance. Six months from now, the note is what separates "I decided this" from "nobody looked".

## The gotcha that cost real time: kube-bench vs. its own k3s profile

One tooling trap worth passing on, because it cost more time than any single check. kube-bench 0.16.0 ships the `k3s-cis-1.9` profile, but the profile's config is missing its `target_mapping` section — the table that tells kube-bench which targets (master, node, controlplane, etcd, policies) a benchmark covers. Point `--benchmark k3s-cis-1.9` at a perfectly healthy cluster and the run dies before executing a single check, complaining that the config file is missing its `target_mapping`.

The fix is to stop using the bundled config in place. Copy `cfg/` somewhere writable, append the missing block to `cfg/k3s-cis-1.9/config.yaml`:

```yaml
target_mapping:
  "k3s-cis-1.9":
    - "master"
    - "node"
    - "controlplane"
    - "etcd"
    - "policies"
```

and point kube-bench at the copy with `--config-dir`. Same binary, same checks — the profile just never told the runner how to assemble itself.

## Remediation, cheapest risk first — and the first fix is already in

The order I am working in:

1. Dedicated ServiceAccounts, token automount off *(done — below)*
2. Pod Security labels, warn/audit mode first, enforce after a quiet week
3. Default-deny NetworkPolicy in the namespaces that have none, with explicit allows
4. Pin the two `:latest` images; ride the controller upgrade that clears 9 CRITICALs
5. Secrets encryption at rest, in its own restart window
6. Resource limits on the system components, folded into whichever change touches them next

Item 1 shipped the same weekend, as a small GitOps pull request: +42 lines, −0. Each of the three apps got a dedicated ServiceAccount with no Role, no bindings, and automount disabled on the account itself as well as in the pod spec:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: wetty
  namespace: tools
automountServiceAccountToken: false
---
# in the Deployment's pod template:
spec:
  serviceAccountName: wetty
  automountServiceAccountToken: false
```

The zero-risk argument held up: the browser only needs its config volume, the SSH client's identity is a mounted Secret *file* (not an API read), and the demo serves static files. Verification was three checks — both Applications Synced and Healthy after the sync, the three new ServiceAccounts present, and the rolled pods' volume lists free of the one entry that proves a token is mounted:

```bash
kubectl get pod -n tools <pod> -o jsonpath='{.spec.volumes[*].name}'
# no kube-api-access-* volume in the output
```

The same gateway that fronts these apps also fronts `kubectl` itself now, which makes the token hygiene less academic: a stolen API credential is worth more when the API has a well-known front door, even an MFA-gated one.

The audit itself is now a monthly diff rather than an event. The one-shot scans re-run against the current image list, and the only results worth attention are a CRITICAL that was not there last month or a count that moved. Everything else on the list above is scheduled work with a known fix — which is, in the end, the difference between a scanner report and an assessment: the report ends in numbers, the assessment ends in a queue.
