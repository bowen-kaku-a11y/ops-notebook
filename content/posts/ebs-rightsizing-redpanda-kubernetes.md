---
title: "How We Cut ~$1,600/Month on EBS by Rightsizing Redpanda Volumes on Kubernetes"
date: 2026-09-27
description: "We were paying for 20 TiB of EBS gp3 for two Redpanda clusters while using less than half of it. Here's the exact measurement and migration process we used to rightsize stateful volumes you can't shrink in place."
---

# How We Cut ~$1,600/Month on EBS by Rightsizing Redpanda Volumes on Kubernetes

EBS is the quiet line item on your AWS bill. Nobody provisions a 2 TiB volume because they need 2 TiB *today* — they provision it because "data grows" and nobody wants a 3 a.m. page about a full disk. Two years later you're paying for 20 TiB of `gp3` across two Redpanda clusters while actually using less than half of it.

In our case the math was blunt: 20 TiB of gp3 at $0.08/GB-month is roughly **$1,600/month** before IOPS and throughput charges. Here's how we measured real usage, picked safe target sizes, and migrated stateful volumes that AWS won't let you shrink in place.

## Step 1: Measure actual usage, not provisioned size

`kubectl get pvc` tells you what you *asked for*. What you need is what the filesystem actually holds. We pull this from Prometheus — every kubelet already exposes it:

```promql
# Used bytes per persistent volume, by claim
kubelet_volume_stats_used_bytes{persistentvolumeclaim=~"data-.*"}
```

Pair it with capacity to get utilization:

```promql
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes
```

We built a [small summary view](/posts/storage-summary-api-design/) (top volumes by usage, cluster-wide utilization) on top of these two metrics. The headline number: **cluster-wide utilization under 45%**. Some volumes sat at 20%.

A few caveats we learned while trusting this metric:

- It only reports for **currently mounted** volumes. Unmounted orphan PVCs are invisible — [find those separately](/posts/orphaned-pvcs-ebs-volumes/).
- It reflects the filesystem's view, so reserved blocks (ext4's 5% default) count as used.
- It lags by a few minutes. Fine for capacity planning, not for alerting on sudden spikes.

## Step 2: Pick target sizes with honest headroom

The naive move is "set the volume to current usage + 10%." Don't. Redpanda (like Kafka) needs headroom for segment rolling, and EBS expansion is easy while shrinking is effectively impossible. Our rule:

> **target = p95 usage over 90 days × 1.5, rounded up to the next 100 GiB**

The 90-day p95 captures growth bursts and seasonal peaks; the 1.5× multiplier covers segment churn plus a safety margin. For our two production clusters this brought the fleet from 20 TiB provisioned down to roughly 11 TiB — same safety posture, ~45% less spend.

Also check IOPS/throughput separately. gp3 gives you 3,000 IOPS and 125 MiB/s baseline free; we were paying for provisioned extras on volumes whose actual throughput never left baseline. Rightsize those too — it's a second, smaller saving hiding inside the first.

## Step 3: Migrate — because EBS volumes only grow

Here's the part that stops most teams: **you cannot shrink an EBS volume.** The migration for a Redpanda cluster looks like this:

1. **Provision the new, smaller PVCs** alongside the old ones (new StatefulSet or additional data directories, depending on your operator version).
2. **Let Redpanda rebalance.** Redpanda supports partition movement between brokers — move partitions onto the new volumes, then [decommission the old brokers/volumes](/posts/redpanda-operator-decommission-controller-pvc/). This is online and doesn't require downtime if your replication factor is ≥ 3 and you move one broker at a time.
3. **Verify** every partition has healthy replicas on the new volumes (`rpk cluster partitions` is your friend).
4. **Delete the old PVCs** — and this is the step people forget — then delete the underlying EBS volumes. A deleted PVC with `Retain` reclaim policy leaves the EBS volume (and the bill) behind.

Do it one availability zone at a time. Never migrate two brokers' worth of data simultaneously; you want quorum intact if something goes sideways.

## Step 4: Prove the savings

After migration, the bill tells the story, but we also kept the dashboard: provisioned vs. used TiB per cluster, per month. Two numbers worth putting in front of your manager:

| | Before | After |
|---|---|---|
| Provisioned | ~20 TiB | ~11 TiB |
| Monthly EBS cost | ~$1,600 | ~$880 |
| p95 utilization | <45% | ~70% |

Utilization at ~70% is the sweet spot for stateful streaming workloads: comfortable headroom, no waste.

## The checklist

1. Measure with `kubelet_volume_stats_used_bytes` / `capacity_bytes` over 90 days.
2. Target = p95 × 1.5, rounded up.
3. Audit provisioned IOPS/throughput separately — baseline is often enough.
4. Migrate via partition movement, one broker/AZ at a time.
5. Delete old PVCs **and** confirm the EBS volumes are gone.
6. Keep the dashboard. Storage creeps back up; review quarterly.

Rightsizing isn't a project, it's a habit. The first pass pays for the dashboard; the dashboard pays for every pass after that.
