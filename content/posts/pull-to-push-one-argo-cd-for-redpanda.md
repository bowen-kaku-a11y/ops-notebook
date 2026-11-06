---
title: "From Pull to Push: One Argo CD for Every Redpanda Cluster"
date: 2026-11-05
description: "My home lab used to give every Kubernetes cluster its own Argo CD, each pulling its own manifests. For Redpanda I switched to one central Argo CD that pushes to every cluster, reaching a NAT'd NAS through Teleport with a bot that holds one-hour certificates. Here's why the push model is the better fit for a fleet of streaming clusters, what it costs, and the access-control surprise that turned a 'namespace-only' role into a cluster-wide one without anyone noticing."
---

# From Pull to Push: One Argo CD for Every Redpanda Cluster

My lab has a few small Kubernetes clusters: a cloud VM (`public1`, called **phoenix** in the Redpanda fleet), a k3s on my NAS at home (**qnap-dev**), and an ops node. Until this week each one ran its own Argo CD. Each Argo CD watched its own folder in Git and pulled its own manifests. That's the pull model, and for base components (the Teleport agent, namespaces, access rules) it still works well: each of those clusters heals itself, and a dead Argo CD only hurts its own cluster.

For Redpanda I wanted something else. A common way to run a fleet of streaming clusters is one central Argo CD with one Application per cluster, all generated from a single list. So I moved Redpanda to a push model: one Argo CD on the cloud VM deploys Redpanda to every cluster, including the NAS that sits behind home NAT.

This post covers why, how the central Argo CD reaches a cluster it can't route to, and what the change costs.

## What changed

**Before (pull):** every cluster's Argo CD synced a folder like `qnap-dev/argocd/apps/`. Adding Redpanda would have meant one Redpanda folder per cluster, each with its own copy of the chart, versions and values.

**After (push for Redpanda, pull for everything else):**

```
central Argo CD (cloud VM)
└─ redpanda-management                     reads clusters.yaml
   ├─ redpanda-operator-phoenix            operator, pinned per cluster
   ├─ redpanda-clusters-phoenix
   │  └─ redpanda-cluster-phoenix-logstream    one Redpanda cluster
   ├─ redpanda-operator-qnap-dev           same, through Teleport
   └─ redpanda-clusters-qnap-dev
      └─ redpanda-cluster-qnap-dev-logstream
```

Each cluster's own GitOps still owns the boring, cluster-local parts: the `redpanda` namespace with its Pod Security labels and its admission policies. On the NAS that also includes the RBAC binding the central Argo CD uses through Teleport. The cloud VM needs no binding, because the central Argo CD runs there and deploys in-cluster. The central Argo CD owns only Redpanda. I also removed the Argo CD on the ops node entirely: it was spending about 500 MiB of a 5.5 GiB node to manage a single app.

## Why push fits a fleet better

**One list is the fleet.** `clusters.yaml` names every Kubernetes cluster and the operator version it runs. A second file names every Redpanda cluster inside them. To add a cluster you add a few lines. With pull, the same information would be scattered across one folder per cluster, and "what is the fleet running?" would mean reading all of them.

**Version pinning per cluster, in one place.** Each cluster points at a vendored operator chart directory: `chartPath: charts/operator-26.2.5`. Upgrading the operator means adding `charts/operator-26.3.x/` and moving one cluster's pointer per pull request. A staged rollout becomes a sequence of one-line diffs, each limited to one cluster. Broker upgrades work the same way through a per-cluster image-tag override. Operator and broker upgrades always go in separate PRs.

**One render shows the whole fleet.** A small script expands every layer locally: the management chart, then each generated Application, then each Redpanda custom resource. A PR diff shows exactly which clusters change. With pull, the equivalent check means rendering each cluster's folder separately.

**One place to see drift.** The central Argo CD UI lists every Redpanda cluster with its sync and health state. With pull, I would have had to open one UI per cluster, one of them behind a NAT.

**Base components stay local.** I didn't push everything. On the clusters that keep their own Argo CD, the Teleport agent, namespaces and RBAC stay pull-based. If the central Argo CD is down, those clusters keep their own access layer, and Redpanda keeps running. Only new syncs wait. (The ops node is the exception: it now has no Argo CD at all, and its few components are deployed by a script.)

## Reaching a cluster behind NAT

Push needs a route from the central Argo CD to every cluster's API server. The NAS has no inbound ports, and I wasn't going to open any.

Teleport was already in place for my own `kubectl` access, so the central Argo CD uses it too:

1. **tbot** runs next to the central Argo CD and joins Teleport as a machine identity (a "bot"). It joins with the Kubernetes in-cluster method: it shows its projected ServiceAccount token, and the auth server checks it with a TokenReview. No join secret is stored anywhere.
2. tbot requests a certificate for the Teleport role `argocd-redpanda`. The certificate lasts at most one hour and is renewed every twenty minutes.
3. For every Kubernetes cluster whose Teleport label matches `cluster=qnap-dev`, tbot writes an Argo CD cluster Secret. The Secret points at the Teleport proxy and carries the short-lived certificate.
4. Argo CD talks to the NAS through the Teleport tunnel, the same tunnel my laptop uses.

Nothing on the NAS listens for the central Argo CD. Revoking the bot works two ways, with different timing. A Teleport **lock** takes effect immediately and rejects certificates already issued. **Deleting the join token** only stops new certificates, so the current one keeps working until it expires, within the hour. Either way, everything already deployed keeps running.

## The cost, written down

**The central Argo CD is effectively cluster-admin everywhere.** The Redpanda Operator is cluster-scoped: it ships ClusterRoles that can create ClusterRoles, ClusterRoleBindings and CRDs. Anything that installs it needs equivalent rights on that cluster. On the NAS, the Kubernetes group the bot maps to is bound to `cluster-admin`. That comes with choosing a central installer for a cluster-scoped operator. The honest summary is that **anyone who can merge to the Redpanda repo administers every cluster in it**. That includes the cloud VM that hosts Teleport's auth server, and cluster-admin there is a path to everything Teleport fronts: every enrolled cluster, node and app, not just the Redpanda ones.

**So the guardrails target mistakes, not attackers.** I added three layers that each catch an accident:

- **Two AppProjects:** a high-privilege one for the operator and the generated Applications, and a low-privilege one that may only deliver `Redpanda` custom resources into the `redpanda` namespace.
- **Admission policies:** in the `redpanda` namespace, only ClusterIP Services without `externalIPs`, and no Ingress. The operator creates Services itself, outside Argo CD, so a project whitelist can't see them. An admission policy sees every writer.
- **A guard against escaping the projects:** an admission policy requires Applications whose names follow the fleet's naming pattern (`redpanda-management`, `redpanda-operator-*`, `redpanda-clusters-*`, `redpanda-cluster-*`) to use one of those two projects. A project can't restrict which project its child Applications declare, so without this, writing the cluster's main project into a template would bypass both whitelists. The guard matches on name, not on origin: a generated Application with a name outside that pattern isn't covered.

**Teleport becomes a dependency for syncing.** If Teleport is down, the central Argo CD can't reach the NAS. Workloads are unaffected; only changes wait.

**Whoever reads the Argo CD namespace's Secrets gets the bot's access.** tbot writes the cluster certificate into a Secret in Argo CD's namespace, and the chart's Role lets tbot read and write every Secret there. So any person or workload that can read Secrets in that namespace can take the certificate and use it until it expires.

**Some churn.** tbot rewrites the cluster Secret every twenty minutes, and Argo CD refreshes that cluster's cache each time. The bot only sees one namespace plus cluster-scoped resources, so the cost is small, but it isn't zero.

## The surprise: a namespace that wasn't a limit

The Teleport role was supposed to be the main place where access got narrowed: read cluster-scoped resources, read and write in `redpanda`, write only the operator's own ClusterRoles and ClusterRoleBindings (named `redpanda-operator-*`), and nothing in other namespaces. That ClusterRoleBinding write is needed to install the operator, and it is also why the bot is effectively cluster-admin: whoever can write a ClusterRoleBinding can bind themselves to `cluster-admin`. My first version used role version 7 with a rule like this:

```hcl
kind      = "*"
namespace = "redpanda"
verbs     = ["get", "list", "watch", "create", "update", "patch", "delete"]
```

That looks like "everything, but only in `redpanda`." The Teleport documentation says otherwise. In v7, `kind: "*"` "allows access to all resources including the cluster-wide ones **regardless of the namespace field**." So the rule granted write access to the whole cluster. Earlier, when the NAS side only had a namespace-scoped RoleBinding, Kubernetes RBAC hid the problem. Once the binding became `cluster-admin`, the Teleport rule was the only thing narrowing access, and it narrowed nothing.

Role version 8 has the semantics I wanted:

- `namespace: ""` matches cluster-scoped resources only.
- `namespace: "redpanda"` matches that namespace and never cluster-scoped resources.
- `kind` takes plural resource names with an `api_group`, and it covers CRDs.

The final role has four rules: cluster-scoped read-only, read/write in `redpanda`, and write on `clusterroles` and `clusterrolebindings` named `redpanda-operator-*`. One caveat I haven't verified yet: a Kubernetes `create` carries the object name in the request body, not the URL, so I still need to test whether Teleport enforces that name filter on creates. Until then I treat the name limit as unproven.

Two more surprises came while switching:

- The Terraform provider normalizes `namespace = ""` to `null`, so the plan failed with "Provider produced invalid plan." In config, cluster-scoped rules have to say `namespace = null`.
- Role v8 rejects the SAML IdP option (`idp`) that v7 roles carry by default. Neither the plan nor validation caught it; the server refused at apply time. The failed apply changed nothing, so it was safe, but it was a surprise.

## What I'm keeping from this

- **Push for the fleet, pull for the floor.** One central control plane for the thing you upgrade in waves, and per-cluster self-healing for the access layer it depends on.
- **Reach clusters through the access layer you already trust.** A bot with one-hour certificates beats opening an API server to the internet or copying long-lived kubeconfigs.
- **Write the blast radius down in plain words.** "Merge rights here = cluster-admin there" belongs in the README, not in someone's head.
- **Read the matching rules of your access tool, not just the field names.** `namespace: "redpanda"` looked like a limit and wasn't. The fix was one version number away, but only after reading the docs.
