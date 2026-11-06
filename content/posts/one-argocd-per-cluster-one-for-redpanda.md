---
title: "One Argo CD per Cluster, and One for Redpanda"
date: 2026-11-04
description: "My home lab runs an Argo CD on each cluster, and I want to keep that while also running Redpanda the way production does: one central Argo CD deploying it to every cluster through Teleport. This post covers the half that's built: isolation that makes a wrong kube context fail closed, a disk-pressure incident that turned out to be 53 GB of unused disk, and the target-cluster side of a narrow access chain. It also covers the half that's still planned, including a 'cheap rename' I had to take back."
---

# One Argo CD per Cluster, and One for Redpanda

> **Update (2026-11-05):** the "planned" half of this post changed when it was built. Redpanda now runs on the Redpanda Operator, with a layout copied from a common fleet structure. Because the operator is cluster-scoped, the NAS cluster binds the central Argo CD's group to `cluster-admin`, not to a namespace-scoped `admin`, and the narrowing moved into a Teleport role v8. The access chain described below is the earlier design. See [From Pull to Push](/posts/pull-to-push-one-argo-cd-for-redpanda/) and [Redpanda on Two Clusters](/posts/redpanda-on-two-clusters-what-got-in-the-way/) for what was actually deployed.

My lab has three small clusters: a cloud node (public1), an ops node, and a k3s on my NAS (qnap-dev). Each runs its own Argo CD, which manages that cluster's base components: the Teleport agent, tools, namespaces, access rules. Production splits things the same way. Infrastructure is managed per cluster, so a broken Argo CD only affects its own cluster.

Redpanda is the exception, in production and soon in the lab. A single central Argo CD has one Application per cluster, `redpanda-cluster-<name>`, in a `redpanda` project, each with a different destination. The central control plane is shared, but upgrades stay independent: each cluster's Redpanda is its own Application, synced on its own schedule. I want the lab to work the same way: one minimal Redpanda on public1 and one on qnap-dev, both deployed by public1's Argo CD.

This post is about half of that job. The isolation work and the qnap-dev side of the access are merged and running. The central side (`tbot`, the Teleport role, the `redpanda` project and the ApplicationSet) is the next set of PRs. I'll say which is which as I go.

## 1. Make the wrong cluster fail closed (done)

Two Argo CDs that can both write to the same cluster is how you get them fighting, so this part came first.

The clusters share one repo, and Application names repeat: public1 and qnap-dev both have a `teleport-agent`. Running `kubectl apply` with the wrong kube context used to deploy one cluster's app onto the other without any warning.

The fix uses AppProjects:

- public1 and qnap-dev each get an AppProject named after the cluster, and every Application there must use it.
- That project **only exists on its own cluster**.
- The built-in `default` project is locked on both: no sources, no destinations.

Now an Application applied to the wrong one of those two clusters refers to a project that cluster doesn't have, and Argo CD marks it `InvalidSpecError` instead of deploying it. The mistake is still possible, but it can no longer do any damage. The ops cluster isn't covered yet: its one Application still uses `default`, which is on the follow-up list.

Bootstrap became one script, `argocd-bootstrap.sh <cluster> <install|project|root>`, with one settings file per cluster. Before it touches anything it checks the node name, so the context really points at the cluster it thinks it does. It refuses downgrades. It uses `kubectl apply --server-side --force-conflicts`. Server-side because the ApplicationSet CRD is too large for the annotation a client-side apply writes, and `--force-conflicts` because the first installs were client-side applied, and server-side apply over those stops on field-manager conflicts.

Review found two problems in the first version:

- **A private key copied into an annotation.** The script printed a command that created the repo deploy-key Secret with a client-side `apply`. Client-side apply saves the whole object in the `last-applied-configuration` annotation, so the private key would have been stored twice, once as plain text in metadata. That step now uses server-side apply.
- **A destructive command with no context.** The README's migration step deleted the old root Application but never said which cluster to run it on. On the wrong context it would have deleted another cluster's root. The migration now runs inside the `root` step, after the node-name check, and the README just says "run the root step".

## 2. Disk pressure that was really unused disk (done)

While this was going on, public1 had 25 `Failed` pods left over from a DiskPressure eviction five days earlier. Deleting them was easy. The cause was more interesting:

```
sda                100G
└─sda3             44.5G
  ├─ocivolume-root 29.5G  /
  └─ocivolume-oled 15G    /var/oled
```

The boot volume was 100 GB, but the image's partition only used 44.5 GB, and the root logical volume was 29.5 GB. Oracle Linux doesn't grow the partition on first boot. `oci-growfs` grew it online, with no reboot and no pod restarts, and `/` went from 29.5 GB to 83 GB. One detail remains: the node's `ephemeral-storage` capacity still shows the old number, because kubelet reads it only at startup. Eviction uses live free space, so it's correct already, and the number will update at the next restart. Running `oci-growfs` is going into the host's init steps, so the next rebuilt machine doesn't start with the same problem.

## 3. Name the deployment, not the machine (planned)

Production gives each Redpanda cluster a logical name, separate from the infrastructure underneath it. I'm copying that: public1's Redpanda will be **phoenix**, and qnap-dev's will just be **qnap-dev**. The node, the Teleport cluster name and the agent all keep their infrastructure names.

The plan is to fix everything inside the cluster: the Helm release is always `redpanda` and the namespace is always `redpanda`. The logical name shows up only in the Argo CD Application name and in one `clusters.yaml` entry.

My first draft said that makes a rename a one-line change, with the pods never noticing. Review pointed out that's wrong as stated. Renaming the entry makes the ApplicationSet delete `redpanda-cluster-phoenix` and create a new Application. Deleting an Application with the resources finalizer cascades, so Argo CD would delete the StatefulSet and Redpanda would go down until the new Application recreated it. The PVCs would survive, but that's still an outage, not a cheap rename. To make it actually cheap, the ApplicationSet needs `syncPolicy.preserveResourcesOnDeletion: true`. Then the old Application lets go, and the new one adopts the same release in the same namespace. I'll set that from the start and test a rename before calling it cheap again.

## 4. The access chain, one link at a time (half done)

The plan is for public1's Argo CD to reach qnap-dev through Teleport. A `tbot` running next to Argo CD will get short-lived certificates and register the clusters it can reach as Argo CD cluster Secrets, pointed at the Teleport proxy. The access is meant to be narrow. It's a chain, and each link limits the next:

1. **Teleport label (live).** qnap-dev's agent advertises `cluster=qnap-dev`. The agent rolled automatically when I added the label, because the chart puts a `checksum/config` annotation on the pod template. It came back within a minute.
2. **Teleport role (in review).** `argocd-redpanda` matches `kubernetes_labels: {cluster: [qnap-dev]}`, allows Kubernetes resources only in namespace `redpanda`, and maps to the Kubernetes group `argocd-redpanda`. It's written in Terraform and will be applied through Atlantis, like the other roles.
3. **Kubernetes RBAC (live).** On qnap-dev, a RoleBinding in `redpanda` gives that group the built-in `admin` ClusterRole, **inside that namespace only**.
4. **Argo CD project (next).** The central `redpanda` project will allow only that destination and namespace, and `tbot` will register the cluster with `clusterResources: false`.

Two questions came up in review about the live links, and both were worth checking on the cluster:

- **Is `admin` bigger than it looks?** `admin` is an aggregated role, so any CRD that adds itself to `aggregate-to-admin` extends it. On qnap-dev, only the built-in roles and a metrics read rule were aggregated. That's fine for now, and worth checking again after installing an operator.
- **Does Redpanda need anything cluster-wide?** A local minimal render of the chart contained only namespaced resources: no ClusterRole and no CRDs. If a future chart version adds some, this chain will block them, which is what I want.

## 5. The namespace belongs to the cluster, not the deployer (done)

The `redpanda` namespace is created by qnap-dev's own Argo CD, not the central one. That's the one place the two models meet, so I made the boundary explicit:

- **Pod Security `baseline` is enforced.** The Redpanda chart has a `tuning` init container that runs privileged with `SYS_RESOURCE` by default, and baseline rejects it. The Redpanda values will have to turn tuning off (`tuning.tune_aio_events: false`), or the first sync will fail. That's fine on a lab box, and it means a privileged pod can't land in this namespace unnoticed.
- **The local Argo CD won't delete it.** The namespace has `argocd.argoproj.io/sync-options: Prune=false,Delete=false`, so removing it from the local repo can't delete the namespace while Redpanda and its PVCs are in it. The rest of that app, the RoleBinding, still prunes normally.
- **The central Argo CD won't create it.** Its Applications won't set `CreateNamespace=true`. If the namespace is missing, the sync will fail instead of creating a namespace with no Pod Security labels.

## What I'm keeping from this

- **Make the mistake fail closed instead of trying to prevent it.** I'll still apply a manifest with the wrong context someday. On public1 or qnap-dev I'll get `InvalidSpecError` instead of an outage. Ops is next.
- **Read what your command does to metadata.** A client-side apply of a Secret puts the Secret into an annotation. It's an easy thing to miss when reviewing the YAML.
- **Check `lsblk` before resizing anything in the cloud.** The fix I'd planned was a Terraform change to grow the boot volume. The disk was already big enough.
- **A rename is only cheap if deletion is.** Fixed release and namespace names cover half of it. The other half is making sure the old Application lets go of the resources instead of deleting them.
- **Limit access at every layer, then check the one that can grow.** Label, role, RBAC and project each narrow the access. Aggregated ClusterRoles are the layer that can widen later without anyone editing it, so that's the one to look at again.
