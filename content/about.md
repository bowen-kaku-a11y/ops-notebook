---
title: "About"
date: 2026-09-27
description: "Who writes The Ops Notebook and why it exists."
---

# About The Ops Notebook

The Ops Notebook is a field journal for DevOps and cloud-native infrastructure, written by an operations engineer who runs live Kubernetes data platforms every day — including Redpanda (Kafka-compatible streaming) and Elasticsearch clusters backed by EBS CSI storage.

## Why this site exists

Most tutorials stop at "hello world." Real systems don't. This site documents the unglamorous middle: storage rightsizing, operator upgrades that recreate your PVCs, volume metrics that lie to you, and the cost math nobody puts in the README.

Every article is based on something that actually happened on a real cluster, with real numbers where it matters.

## What you'll find here

- **Kubernetes storage in practice** — EBS CSI, PVC lifecycle, volume monitoring
- **Streaming & search on K8s** — Redpanda and Elasticsearch operations
- **Cost optimization** — rightsizing with real dollar figures
- **Incident notes** — what broke, why, and the exact fix

## Editorial policy

- No sponsored posts disguised as tutorials.
- Commands and manifests are tested before publishing.
- Articles are updated when the underlying software changes behavior; the update date is shown at the top of each post.
