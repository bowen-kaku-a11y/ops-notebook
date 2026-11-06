---
title: "Monitoring Kubernetes Volume Usage with kubelet_volume_stats_used_bytes (and Its Gotchas)"
date: 2026-09-28
description: "The kubelet already exposes per-volume usage metrics to Prometheus. Here's the PromQL we use for dashboards and alerts, plus the five gotchas that will mislead you if you trust the numbers blindly."
---

# Monitoring Kubernetes Volume Usage with kubelet_volume_stats_used_bytes (and Its Gotchas)

Every kubelet exposes per-volume filesystem metrics, and if you run Prometheus with kube-state-metrics or the standard Kubernetes scrape configs, you're probably already collecting them and ignoring them. These two metrics are the foundation of every storage dashboard we've built:

- `kubelet_volume_stats_used_bytes` — bytes used on the volume
- `kubelet_volume_stats_capacity_bytes` — total capacity
- (bonus) `kubelet_volume_stats_available_bytes` — bytes available

Here's how to turn them into something you'd actually page on.

## The essential queries

**Utilization per PVC:**

```promql
kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes
```

**Top 10 volumes by usage right now:**

```promql
topk(10, kubelet_volume_stats_used_bytes)
```

**Cluster-wide storage efficiency (provisioned vs. used):**

```promql
sum(kubelet_volume_stats_used_bytes) / sum(kubelet_volume_stats_capacity_bytes)
```

That last one is the number for your cost review meetings. Ours sat under 45% for months — which is what kicked off our [EBS rightsizing project](/posts/ebs-rightsizing-redpanda-kubernetes/).

**Predicting "disk full" with `predict_linear`:**

```promql
predict_linear(kubelet_volume_stats_used_bytes[7d], 14 * 24 * 3600)
  > kubelet_volume_stats_capacity_bytes
```

This fires when the 7-day trend says the volume fills within 14 days. It's noisy on bursty workloads, so pair it with a minimum-utilization threshold (e.g., only alert above 75% used) to cut false positives during low-usage growth spurts.

## Alerting rules we actually run

```yaml
groups:
  - name: volume-alerts
    rules:
      - alert: VolumeFillingUp
        expr: |
          predict_linear(kubelet_volume_stats_used_bytes[7d], 14*24*3600)
            > kubelet_volume_stats_capacity_bytes
            and
          kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.75
        for: 6h
        labels:
          severity: warning
        annotations:
          summary: "Volume {{ $labels.persistentvolumeclaim }} filling up within 14 days"

      - alert: VolumeCriticallyFull
        expr: kubelet_volume_stats_used_bytes / kubelet_volume_stats_capacity_bytes > 0.9
        for: 15m
        labels:
          severity: critical
        annotations:
          summary: "Volume {{ $labels.persistentvolumeclaim }} over 90% full"
```

The `for: 6h` on the predictive alert matters — `predict_linear` on a 7-day window swings wildly with short bursts. Six hours of sustained prediction is a real trend; six minutes is a deploy.

## The five gotchas

**1. Unmounted volumes are invisible.** The kubelet only reports stats for volumes mounted into a running pod. Orphaned PVCs — the ones silently billing you — never appear. Audit those with a separate query joining `kube_persistentvolumeclaim_info` against actual pod mounts.

**2. ext4 reserves 5% by default.** The metric reports the filesystem's view, so that reserved space counts as "used." A volume reporting 96% used on ext4 is effectively full for non-root processes. Set your critical threshold at 90%, not 95%.

**3. It lags.** Kubelet volume stats update on the order of minutes, and Prometheus scrapes on top of that. Don't build autoscaling on this metric; it's a planning and alerting signal, not a control loop.

**4. `available_bytes` ≠ `capacity - used`.** Reserved blocks, inodes, and filesystem overhead mean the three metrics don't reconcile exactly. Use `used / capacity` for utilization and `available` only for "can I write one more segment" sanity checks.

**5. Ephemeral storage has its own metrics.** These `volume_stats` metrics cover persistent volumes. For container ephemeral storage (emptyDir, logs, image layers), you want `kubelet_volume_stats_*` on the ephemeral volume or node-level `node_filesystem_*` metrics instead. Mixing the two is a classic dashboard bug.

## When the series doesn't exist at all

There's one failure mode worse than a misleading number: no series at all. We hit this in a local kind cluster — the PVC was Bound, a pod had it mounted and was writing to it, and Prometheus was scraping the kubelet successfully for other metrics. Every volume query came back empty, because the kubelet had never emitted the metrics in the first place.

The cause was the storage underneath. kind's default provisioner backs volumes with hostPath, and the kubelet doesn't collect volume stats for hostPath-backed volumes. No error, no warning — just an absent series.

Before you blame your scrape config, isolate it the way we did:

**1. Confirm a pod actually mounts the volume.** This rules out gotcha #1 above:

```bash
kubectl get pods -A -o json | \
  jq -r '.items[] | select(.spec.volumes[]?.persistentVolumeClaim.claimName == "<pvc>") | .metadata.name'
```

**2. Ask Prometheus directly, and check a metric that should exist:**

```promql
kubelet_volume_stats_used_bytes          # empty result
kubelet_running_pods                     # returns data → scraping works
```

If the second query returns data, Prometheus and the scrape config are fine. The metric was never produced.

**3. Ask the kubelet itself.** This is the decisive step:

```bash
kubectl get --raw "/api/v1/nodes/<node>/proxy/metrics" | grep kubelet_volume_stats
```

No output means the kubelet on that node emits no volume stats at all — there is nothing for Prometheus to scrape, and no relabeling or recording rule will fix it.

The practical takeaway: on kind and other local-path setups, expect this gap and don't burn an afternoon on it. If you genuinely need usage numbers inside a local cluster, switch the workload to a CSI-backed StorageClass (for example a CSI hostpath driver). On real CSI volumes — EBS CSI included — the metrics show up exactly as the rest of this guide describes.

## Building the summary view

Raw per-PVC time series get noisy at fleet scale. We aggregate into a cached summary API: per-cluster provisioned vs. used, top volumes by utilization, and week-over-week growth rate. The key design decision: **cache aggressively and degrade gracefully.** If Prometheus is down, serve the last good snapshot with a staleness banner instead of a blank dashboard — a storage dashboard that errors out during the incident where you need it is worse than no dashboard.

Two rollups cover 90% of questions:

```promql
# Provisioned vs used per StorageClass (find your expensive habits)
sum by (storageclass) (kubelet_volume_stats_capacity_bytes)
sum by (storageclass) (kubelet_volume_stats_used_bytes)

# Growth rate per week per PVC (find your future problems)
(sum by (persistentvolumeclaim) (kubelet_volume_stats_used_bytes)
 - sum by (persistentvolumeclaim) (kubelet_volume_stats_used_bytes offset 7d))
 / 7 / 24 / 3600
```

Start with utilization and the predictive alert. Add the cost rollup when finance starts asking questions — they will, and you'll have the answer before they finish the sentence.
