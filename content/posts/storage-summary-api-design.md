---
title: "From kubectl to Code: Designing a Storage Summary API"
date: 2026-10-10
description: "Answering 'where is our storage waste?' used to mean joining kubectl output, EBS volume IDs, and Prometheus by hand. We replaced the ritual with one small API — and designed its two defining traits on purpose: aggressive caching, and degrading gracefully instead of failing."
---

# From kubectl to Code: Designing a Storage Summary API

Every storage cost review used to start with the same ritual. `kubectl get pvc` for what we had provisioned. A second pass to map each volume to its EBS volume ID. Then Prometheus for `kubelet_volume_stats_used_bytes`, to see what was actually used. Three sources, joined by hand, in a spreadsheet, by a human — every single time someone asked the simplest question in storage operations: *where is the waste?*

Our [kubelet volume stats guide](/posts/kubelet-volume-stats-prometheus-guide/) ended with a paragraph about the summary view we built on top of those metrics. This post is that paragraph, expanded: a small Go service with exactly two endpoints, and the two design decisions that define it — **cache aggressively, and degrade gracefully**.

## One endpoint, one question

The API answers one question, so it has one shape. `GET /api/summary` returns a single object: every volume with its provisioned and used gigabytes, the storage price we pay per GB-month, and totals that price the waste:

```json
{
  "generatedAt": "2026-10-10T13:00:00Z",
  "cacheTTLSeconds": 60,
  "degraded": false,
  "priceUSDPerGBMonth": 0.08,
  "totals": {
    "volumes": 42,
    "provisionedGB": 51200,
    "usedGB": 22940,
    "wastedUSDPerMonth": 2260
  },
  "volumes": [ "..." ]
}
```

A second endpoint, `GET /api/history?days=7` (clamped to 1–30), serves the trend. That's the whole surface.

The scope discipline matters more than the shape. This is not a monitoring system — Grafana already exists, alerts already exist. The moment a summary API grows query builders and dashboards of its own, you've built a second, worse Prometheus. Two endpoints, one question, no apologies.

## Cache aggressively, on purpose

Every summary is cached for 60 seconds (configurable via `CACHE_TTL_SECONDS`), and a request that lands within the TTL gets the cached object — no Kubernetes API calls, no PromQL.

Sixty seconds sounds sloppy until you think about the physics of the data. EBS volumes do not fill up in seconds; a volume's used bytes move at the speed of data ingestion, which for log and streaming workloads means the number you computed a minute ago is, for cost purposes, the same number. What the cache buys is real: a dashboard that auto-refreshes every minute, plus every human who opens it, costs the apiserver and Prometheus **one collection per minute total** instead of one per viewer per refresh. The cache isn't an optimization we added later. It's the reason the service is allowed to exist on a production cluster at all.

## Degradation is a field, not a failure

The second decision: this service essentially never returns a 500.

Construction is the first place that shows. If the Kubernetes client can't be built, or Prometheus isn't configured, the service logs it, starts anyway, and reports the gap in every payload. Collection is the second: if listing volumes fails, or the usage query errors, the summary still returns — with `"degraded": true` and an `errors` array saying exactly which source failed and why. The history endpoint follows the same rule: no Prometheus, no data — but the payload is a well-formed, empty, degraded response, not an error page.

The UI honors the contract. It renders the numbers it has and shows the degraded state on the page, instead of silently drawing a reassuring row of zeros. This is the part people resist — *shouldn't it just fail loudly?* — and the incident math argues otherwise. The moment you most need a storage dashboard is during an incident, which is precisely when its dependencies are likeliest to be limping. A dashboard that shows you 90% of the truth with a warning banner beats a dashboard that shows you nothing with great integrity. Degraded is information. A 500 is the absence of it.

## "Unknown" is not zero

The subtlest bug we shipped was representing a missing usage number as zero. Zero is a measurement: it says the volume is empty, it feeds the totals, it prices the waste at 100%. But sometimes the number isn't zero — it's *absent*, because kubelet never emitted the series for that volume at all. On hostPath-backed volumes, [the series doesn't exist](/posts/kubelet-volume-stats-prometheus-guide/), and no amount of querying will conjure it.

So in the payload, `usedGB` is nullable, and absence stays absence all the way into the totals — which then have to be honest about what they are. Our totals sum the volumes whose usage is known, and say so: `volumesWithUsage` counts them, and `partial: true` is set whenever at least one volume reported nothing. An earlier version withheld the totals entirely if any volume was missing its number, which in practice meant the totals were null forever and the dashboard's headline was always blank. Partial sums with a label turned out to be strictly more useful than perfect sums that never arrive.

## What breaks at 10x

At our scale — dozens of volumes — a full collection is cheap, and the design above is almost all cache and honesty. At ten times the PVCs, three things bend first: the collection itself wants to become incremental instead of a full list-and-query sweep; the single flat volume list wants per-team and per-cluster rollups, because nobody reviews 4,000 rows; and the history cache, keyed by day-range, wants a real retention story instead of a map in memory.

None of that changes the two decisions. If anything, scale argues harder for them: the more expensive a fresh answer is, the more a cache earns its keep — and the more dependencies a collection touches, the more certain it is that one of them is having a bad day when you ask.

The ritual is gone. The cost review now starts with the answer already on the screen — and when the screen says *degraded*, we know exactly which third of the truth to go and check by hand.
