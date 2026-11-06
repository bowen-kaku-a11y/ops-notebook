---
title: "Redpanda on Two Clusters: What Got in the Way"
date: 2026-11-05
description: "Rolling out a minimal Redpanda fleet with the Redpanda Operator to a cloud VM and a home NAS from one Argo CD. The cloud cluster came up on the first sync. The NAS got stuck on a registry mirror's allowlist, and a two-line pull request to someone else's repo unstuck it. Both now pass a produce/consume smoke test. Along the way: a chart setting that is silently ignored, telemetry that needs two switches to turn off, a disk guard that only works if it beats the kubelet, and a naming scheme that is cheap to read but expensive to change."
---

# Redpanda on Two Clusters: What Got in the Way

The goal was small on purpose: one Redpanda broker on each of two clusters, deployed by the Redpanda Operator, managed by one central Argo CD. One cluster is a 2-core cloud VM (`public1` in earlier posts; in the Redpanda fleet it's called **phoenix**). The other is a 4-core k3s on my NAS (**qnap-dev**). Every setting is the minimum: one broker, 400m CPU, 2 GiB of memory, 4 GiB of storage. These clusters exist to rehearse the operational work (staged upgrades, per-cluster version pins, retirement) without paying for a big fleet.

Both are up now and both pass a smoke test. phoenix came up on the first sync. qnap-dev got stuck for a reason that had nothing to do with Kubernetes. Both stories are below.

## The layout

The repo follows a common fleet layout:

```
clusters.yaml              k8s clusters + the operator version each one runs
charts/
  management/              per k8s cluster: an operator app and a "clusters" app
  operator-26.2.5/         vendored operator chart, one directory per version
  clusters/values.yaml     clusters.<k8s>.<name>: every Redpanda cluster
  cluster/                 one Redpanda custom resource
scripts/argo-render.sh     expands every layer locally for PR diffs
```

A Redpanda cluster is one entry:

```yaml
clusters:
  phoenix:
    logstream:
      scope: logstream
      redpanda:
        config:
          extraClusterConfiguration:
            storage_min_free_bytes:
              repr: "10737418240"
```

From that entry, the charts generate an Argo CD Application and a `Redpanda` custom resource, and the operator creates a StatefulSet. All three are named **`redpanda-cluster-phoenix-logstream`**.

## Names you can read, and can't cheaply change

I wanted one name everywhere: the Application, the custom resource, the StatefulSet, the Service, and the PVC (`datadir-redpanda-cluster-phoenix-logstream-0`). When something pages you, the name in the alert should be the name in Argo CD and the name in `kubectl`.

Two costs come with that:

- **Length.** The name becomes a Service name and pod hostname, which are DNS labels limited to 63 characters. The StatefulSet name also feeds a `controller-revision-hash` label, which caps the StatefulSet name at about 52. The chart then derives longer names such as `<name>-rpk-debug-bundle`, and I want those to fit in 63 as well. So I chose a conservative 46-character limit, which leaves 28 characters for "k8s cluster + Redpanda cluster." The chart now fails the render past that limit, and also when two entries would collide (`qnap` + `dev-logs` and `qnap-dev` + `logs` produce the same string).
- **Renames are migrations.** Once the cluster name is part of the StatefulSet and PVC names, renaming `phoenix` creates a new StatefulSet and new volumes. The old data doesn't follow. My earlier design kept the logical name only in Argo CD so renames were free. I traded that away for readability on purpose, and the README now says "pick names once."

## phoenix: up on the first sync

Merging two pull requests produced this in the central Argo CD, and everything was Synced and Healthy within a few minutes:

- The operator's pre-install hook Job installed nine CRDs.
- The operator Deployment came up.
- The `Redpanda` resource went Ready, with the status message "Cluster ready to service requests."
- The StatefulSet `redpanda-cluster-phoenix-logstream` ran 1/1, with a 4 GiB local-path PVC.
- The only Service was ClusterIP. The admission policy allowed it, as it should.

The smoke test was a topic, three messages, and a consumer, first from inside the pod and then through the Service DNS name that other clients will use:

```
$ rpk topic create smoke-test -p 1 -r 1
$ printf 'hello-1\nhello-2\nhello-3\n' | rpk topic produce smoke-test
$ rpk topic consume smoke-test -n 3 -o start -f '%o %v\n' \
    -X brokers=<service>.redpanda.svc.cluster.local:9093
0 hello-1
1 hello-2
2 hello-3
```

Memory in use: about 430 MiB for the broker and 110 MiB for the operator.

## Chart settings that don't do what they say

**A setting that is silently ignored.** I set `storage_min_free_bytes` under `config.cluster`, which is the documented place for cluster properties. The rendered bootstrap config still showed the chart's computed default of about 205 MiB. The chart (26.2.x) builds its bootstrap map with its own value first and then merges user values. In that merge the earlier value wins, so a user value for a key the chart already set never lands. The same happens to the `tunable` keys (`log_segment_size_min/max`, `kafka_connection_rate_limit` and others). Those work under `config.tunable`. `storage_min_free_bytes` only works under `config.extraClusterConfiguration`, which is merged last, and only in this form:

```yaml
extraClusterConfiguration:
  storage_min_free_bytes:
    repr: "10737418240"   # a string; a number fails the chart's schema
```

Keys the chart doesn't preset, such as `retention_bytes`, work fine under `config.cluster`. I checked each case by rendering the chart, not by trusting the values file comments.

**Telemetry needs two switches.** `logging.usageStats.enabled: false` removes the environment variables that the chart writes into the pod template. That one also matters for stability, because one of those variables is the Kubernetes version, so every k3s patch upgrade would otherwise restart the broker. It does not stop the broker's own reporter. That is the cluster property `enable_metrics_reporter: false`. Only after setting both did `rpk cluster config get enable_metrics_reporter` return `false`.

**The tuning init container is privileged.** By default the chart adds an init container with `privileged: true` and `SYS_RESOURCE` to tune kernel parameters. The `redpanda` namespace enforces Pod Security `baseline`, so that pod would be rejected. On a lab box the fix is `tuning.tune_aio_events: false`.

## A disk guard has to beat the kubelet

local-path PVCs aren't size-limited: the "4 GiB" volume is a directory on the node's root filesystem, which other services share. Two settings keep Redpanda from filling it:

- **Usage:** `retention_bytes` of 256 MiB per partition, plus 16 MiB segments, because retention only deletes closed segments and the open one adds to the total. This caps topics that delete old data, per partition. It doesn't cap compacted topics (`__consumer_offsets`, `_schemas`), the internal raft logs, or the number of partitions, so on its own it isn't a hard limit. The floor below is the real backstop.
- **Floor:** `storage_min_free_bytes` makes Redpanda refuse writes when free space on the filesystem that holds its data directory falls below a threshold. On a plain node that's the root filesystem; with a containerized k3s or bind mounts it may be a different volume, so check with `df` on the data directory. The threshold has to be **above the kubelet's eviction threshold** for that same filesystem. If it's below, the kubelet starts evicting other pods first. I read the kubelet config on both nodes: `evictionHard: nodefs.available 5%, imagefs.available 5%`. With 5% of phoenix's 83 GiB root at 4.1 GiB, I set the floor to 10 GiB. qnap-dev has a 516 GiB root, so 5% is 25.8 GiB, and I set its floor to 30 GiB.

The value differs per machine, so there's no shared default anymore. Every cluster must set its own, and the chart fails to render if one is missing or under 1 GiB.

## qnap-dev: stopped at the registry, not at Kubernetes

The central Argo CD reached qnap-dev through Teleport and synced. The operator's CRD Job was scheduled. Then:

```
Failed to pull image "docker.io/redpandadata/redpanda-operator:v26.2.5":
... unexpected status from HEAD request to https://mirror.example.com/v2/redpandadata/redpanda-operator/manifests/v26.2.5?ns=docker.io: 403 Forbidden
```

The NAS can't reach Docker Hub directly, so k3s sends every `docker.io` pull through a public mirror. That mirror serves only an **allowlist**, and the list (about 1,400 entries) includes `redpandadata/console` and `redpandadata/kminion` but not the broker or the operator. My config was correct. The images just weren't allowed through.

The options I weighed:

- **Ask the mirror to allow them.** I opened an issue and a two-line pull request to their allowlist, citing the official Helm chart lines and docs that reference the images. Their stated turnaround is up to five working days. This is the long-term fix: future upgrades will just work.
- **Import the images by hand.** On a machine that can reach Docker Hub, `docker pull --platform linux/amd64` both images and `docker save --platform linux/amd64` them into one tar. The NAS is amd64. On an arm64 VM or laptop, leaving out the flag gives you the wrong architecture, or a multi-arch index whose missing blobs make the import fail. Copy it into k3s's image directory, which k3s imports on every start. My k3s runs in a container, so that's the host directory bind-mounted at `/var/lib/rancher/k3s/agent/images/`. Then import it once now inside the container with `docker exec <k3s-container> k3s ctr -n k8s.io images import /var/lib/rancher/k3s/agent/images/<tar>`. The `k8s.io` namespace matters: a plain `ctr images import` lands in containerd's `default` namespace, where the kubelet never looks. With `IfNotPresent`, the stuck Job picks the images up on its next retry. Run the import right before the retry. An image that is imported but not yet used can be removed by the kubelet's image garbage collection if the node comes under disk pressure. It's quick, but every version bump would need it again.
- **Add another, unrestricted mirror as a fallback.** Possible, but a third-party mirror serving tags gives no integrity guarantee unless you pin digests. It also needs a k3s restart and affects every image on the node. Not now.

While those were pending, qnap-dev sat in ImagePullBackOff and kept retrying, which was harmless. I also prepared a change that added three unrestricted proxies after the first one as fallbacks. They served the same digests as Docker Hub when I checked, but they would apply to every `docker.io` image with no integrity check, so I treated them as a last resort.

They were never applied. **The allowlist pull request was merged about an hour after I opened it**, before I had touched the NAS. I removed the fallback change again. On its next retry the stuck CRD Job pulled the operator image through the mirror. The operator came up, the broker followed, and the central Argo CD showed every qnap-dev Application Synced and Healthy. Nobody touched the cluster in between. The fix was one pull request to the mirror's allowlist. Everything after that was retries, and everything the cluster needed was already declared in Git.

The checks afterwards:

- The `Redpanda` resource is Ready, and `rpk cluster health` reports healthy.
- `storage_min_free_bytes` is 30 GiB, `enable_metrics_reporter` is `false`, and the retention settings are as written.
- **The disk floor is sized against the right filesystem.** `df` on the broker's data directory shows the same 516 GiB volume the kubelet reports, so 5% of it is the right number to beat. local-path writes into a directory bind-mounted from the NAS data volume, and I had wanted to confirm that rather than assume it.
- The image digests on the node match Docker Hub's.
- The same smoke test as phoenix, through the Service DNS name: three messages in, three out.

Memory in use on the NAS: about 220 MiB for the broker and 90 MiB for the operator.

## What I'm keeping from this

- **Render before you believe a values file.** The most important setting in this rollout was silently ignored where the chart's own comments said to put it.
- **"Off" can take more than one switch.** For telemetry, check what the broker itself reports, not just what the chart renders.
- **Size safety thresholds against the next layer down.** A disk floor below the kubelet's eviction threshold does nothing useful.
- **The first failure on a new network is often not about your system.** The deployment logic was fine. A mirror's allowlist was the blocker, and the fix was a two-line pull request to someone else's repo, merged within the hour.
- **Let retries do the last mile.** Once the image was allowed, nothing else needed doing. The design rule that kept paying off was failing safely and retrying, rather than reaching in by hand.
