---
title: "Redpanda Operator 26.2.1 Recreated Our PVCs: Why We Removed the Decommission Controller"
date: 2026-09-27
description: "A routine operator upgrade changed a StatefulSet template label. The new StatefulSetDecommissioner treated it as a decommission event and started recreating PVCs. Here's the incident, the root cause, and the fix."
---

# Redpanda Operator 26.2.1 Recreated Our PVCs: Why We Removed the Decommission Controller

Some upgrades fail loudly. This one failed *helpfully* — the operator was doing exactly what it was designed to do, and that was the problem.

After upgrading to Redpanda operator **26.2.1**, we noticed PVCs being recreated on our clusters. For a stateful streaming platform, "recreated PVC" is a phrase that ruins your morning: a fresh empty volume where your partitions used to be.

## What happened

The 26.2.1 operator introduced (or newly enabled in our configuration) a **StatefulSetDecommissioner** — a controller whose job is to gracefully decommission brokers when the StatefulSet scales down or changes shape.

The trigger was innocent: the upgrade changed a **label in the StatefulSet's pod template**. To the decommissioner, a template change that alters the set of managed pods looked like brokers leaving the cluster. It began decommissioning — and as part of that flow, PVCs were recreated.

The pods came back. The data didn't, on the affected volumes.

## Why PVC recreation is the worst failure mode

A pod restart is routine. A PVC recreation is not:

- The new PVC binds to a **[new, empty EBS volume](/posts/ebs-rightsizing-redpanda-kubernetes/)**. Your segments are gone from that broker's perspective.
- With replication factor 3 and one broker affected, the cluster can recover from replicas — *if* the other replicas are healthy and *if* you catch it before a second failure.
- If two brokers are hit in sequence, you are one bad moment away from under-replicated partitions and potential data loss.

We caught it early, recovered from replicas, and then made a decision.

## The fix: remove the decommission controller

We **removed the decommission controller** from our deployment. The reasoning:

1. **We decommission brokers explicitly and rarely.** Broker removal is a deliberate, runbook-driven operation in our environment — not something that should ever be inferred from a template diff.
2. **The blast radius of a false positive is catastrophic.** A controller that can delete data volumes must have an extremely conservative trigger. "A label changed" is not conservative.
3. **We'd rather have a manual step than an automatic disaster.** Decommissioning one broker by hand twice a year costs minutes. An automated decommission bug costs sleep.

Concretely, this meant disabling the decommissioner component in the operator configuration and verifying — in a staging cluster first — that StatefulSet template changes no longer touched PVCs.

## Lessons we're keeping

**Pin your operator versions and read the changelog like it's an incident report.** The decommissioner wasn't a secret; it was in the release notes. We just didn't map "new controller" to "can recreate my volumes" before upgrading production.

**Test upgrades against [PVC immutability](/posts/orphaned-pvcs-ebs-volumes/).** Our staging upgrade checklist now includes an explicit assertion: after the operator upgrade, `kubectl get pvc` must show the same UIDs and creation timestamps. Any recreation is a failed test, full stop.

**Treat any controller with delete permissions on storage as critical infrastructure.** It gets the same scrutiny as your backup system: who triggers it, what the exact preconditions are, and what the rollback looks like.

**Template labels are load-bearing.** In the StatefulSet world, [labels in the pod template aren't cosmetic](/posts/the-rollout-that-never-happened/) — controllers watch them. Changing one can cascade through every controller that keys off pod identity. Diff your rendered manifests before applying operator upgrades, not after.

## The takeaway

Automation that manages stateless pods is a gift. Automation that can recreate stateful volumes needs to earn your trust one conservative trigger at a time — and until it does, the manual runbook wins. Our clusters have been quiet since the decommissioner left. We intend to keep it that way.
