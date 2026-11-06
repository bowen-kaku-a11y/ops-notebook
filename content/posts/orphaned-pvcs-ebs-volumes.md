---
title: "Orphaned PVCs: Finding EBS Volumes Nothing Mounts Anymore"
date: 2026-10-15
description: "Every storage dashboard is built on kubelet stats, and kubelet only sees volumes a pod has mounted. The claims nobody mounts are invisible — but their EBS volumes still bill by the hour. Here is how to list them, and how to tell a true orphan from two look-alikes that are not safe to delete."
---

# Orphaned PVCs: Finding EBS Volumes Nothing Mounts Anymore

Every storage dashboard starts from the same data source: kubelet
volume stats. Our [guide to those metrics](/posts/kubelet-volume-stats-prometheus-guide/)
covers how they work and where they mislead. This post is about the
blind spot they all share.

Kubelet reports stats for volumes that a pod on its node has
mounted. A PersistentVolumeClaim that no pod mounts produces no
series at all — no capacity, no used bytes, no row in the table.
It is not that the dashboard shows it as zero. It does not show it
at all.

Meanwhile the volume behind that claim is very real. On EBS CSI
storage, a provisioned volume bills by the hour whether anything
mounts it or not, and it keeps billing until someone deletes it.
Workloads get deleted, namespaces get torn down, StatefulSets get
scaled to zero and forgotten — and each leftover claim quietly
survives, invisible to the tool you would use to look for waste.

One forgotten 100 GiB volume is only about $8 a month — easy to
ignore. But orphans accumulate in herds: a torn-down test
environment here, a replaced StatefulSet there, and a year later
the sweep finds twenty of them — $160 a month for storage that
holds nothing anyone wants.

## Three different silences

When a claim has no usage data, it can mean three very different
things. We learned to separate them the hard way, in a lab cluster
and in our own code:

1. **Nothing mounts it.** The workload is gone and the claim is a
   leftover. This is the true orphan candidate — the only one of
   the three that is safe to act on.
2. **Something mounts it, but the provisioner emits no stats.**
   In a controlled A/B test on the same node and the same kubelet,
   CSI-backed volumes emitted `kubelet_volume_stats_*` series and
   hostPath/local-path volumes emitted none at all. A mounted,
   actively used volume can be exactly as silent as an orphan.
3. **Stats exist, but they do not measure the volume.** Where the
   CSI hostpath driver does report `used_bytes`, the number is the
   node filesystem's statfs figure, not the volume's own usage.
   Silence is not the only failure mode; a confident wrong number
   is the other one.

Deleting on "no data" alone will eventually destroy a live volume
of type 2. Trusting the totals will quietly absorb type 3. The
job is telling type 1 apart from its look-alikes — and for that
you need an inventory that does not come from kubelet at all.

## The inventory query

kube-state-metrics sees every claim in the cluster, mounted or
not, because it reads the API server rather than kubelet. Its
`kube_persistentvolumeclaim_info` series is the full inventory.
The other half of the picture is
`kube_pod_spec_volumes_persistentvolumeclaims_info`, which lists
every claim referenced by some pod's volume spec. Subtract the
second from the first with an anti-join, and the remainder is
your candidate list:

```promql
kube_persistentvolumeclaim_info
unless on (namespace, persistentvolumeclaim)
kube_pod_spec_volumes_persistentvolumeclaims_info
```

We ran this query against a lab cluster (kube-state-metrics
v2.13, Prometheus) with one mounted and one never-mounted claim
before trusting it here: it returned exactly the never-mounted
claim. Two details from that run are worth knowing:

- The claim a pod references is excluded even if the pod is only
  Pending — the metric reads pod *specs*, not mounts. Treat the
  result as candidates to inspect, not a deletion list.
- `kube_persistentvolumeclaim_info` carries a `volumename`
  label. A candidate with an empty `volumename` has nothing
  provisioned yet (common with WaitForFirstConsumer storage
  classes) and is not billing. The ones that cost money are
  Bound, with a volume name — cross-check with
  `kube_persistentvolumeclaim_status_phase{phase="Bound"}`.

## Before you delete

A candidate is a lead, not a verdict. Before removing anything,
we check four things:

- **Status and reclaim policy.** A Bound claim whose PV is set to
  Retain will leave the volume behind even after the claim is
  deleted — which may be the point, or may just move the orphan
  one level down where nobody looks.
- **Who used to own it.** The claim name and namespace usually
  identify the workload. If that workload was temporarily scaled
  away, the "orphan" is next month's outage.
- **Whether a StatefulSet will recreate it.** StatefulSet claims
  come from volumeClaimTemplates; deleting the PVC of a set that
  still exists just moves the problem to its next scale-up, and
  deleting the set without its claims is how many orphans are
  born in the first place.
- **A snapshot first.** An EBS snapshot of the volume before
  deletion is cheap insurance with an expiry date — keep it for
  one review cycle, then delete that too.

## Make your totals admit what they cannot see

The same blindness exists inside any tool that sums kubelet
stats, including ours. Our first totals summed the volumes whose
usage was known and silently treated the rest as zero — the
dashboard equivalent of assuming silence means empty. The fix,
described in [the summary API post](/posts/storage-summary-api-design/),
was to sum only known volumes and mark the result partial
(`volumesWithUsage` and a `partial` flag, with a visible notice
in the UI). A total that admits it is partial leads you to the
orphan question. A total that pretends to be complete hides it.

## The quarterly sweep

Orphan hunting works best as a calendar habit, not a rescue
operation. Ours rides along with the
[rightsizing review](/posts/ebs-rightsizing-redpanda-kubernetes/):
run the inventory query, work the candidate list, check the four
boxes, snapshot, delete, and record what was reclaimed next to
the rightsizing numbers. Five minutes in a quiet quarter — and
the alternative, a volume from a workload nobody remembers,
billing silently for a year.
