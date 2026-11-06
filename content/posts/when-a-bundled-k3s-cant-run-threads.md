---
title: "The Bundled k3s That Couldn't Create a Thread"
date: 2026-11-03
description: "A QNAP NAS ships a one-click k3s inside Container Station, so I tried to register it for Teleport Kubernetes Access. The agent went into CrashLoopBackOff with pthread_create: Operation not permitted — an old seccomp profile rejecting clone3. I had the mechanism right and the conclusion wrong: I decided the cluster couldn't run any threaded image and said so, out loud, before running the one cheap experiment that would have corrected me. This is that debugging trail and the test that turned the verdict inside out."
---

# The Bundled k3s That Couldn't Create a Thread

My QNAP NAS already runs a Teleport SSH node — an agent in a container that dials out over 443 and shows up through the reverse tunnel. That part has been boring and reliable for weeks. QNAP's Container Station also offers a one-click Kubernetes cluster: a single-node k3s, "enable" checkbox, download a kubeconfig, done. So the obvious next move was to register that k3s for Teleport **Kubernetes Access** and get audited `kubectl` from anywhere, with no inbound ports.

The install went clean. The agent never ran. Chasing why it never ran was good debugging right up until the moment I stopped debugging and started concluding — and the conclusion was wrong in a way the next five minutes of work would have caught. I'm leaving the whole trail in, mistake and correction both, because the correction is the useful part.

## The symptom: a healthy install, a crashing pod

The agent is a Helm chart. It templated, applied, and reported `STATUS: deployed`. Then:

```console
$ kubectl -n teleport get pods
NAME                    READY   STATUS             RESTARTS   AGE
teleport-kube-agent-0   0/1     CrashLoopBackOff   1          82s
```

`kubectl describe` was no help — image pulled fine, container `Created` then `Started`, then `Back-off restarting failed container`. The reason a container dies is in its **logs**, and because it was restarting I needed the *previous* container's logs, not the current (empty) one:

```bash
kubectl -n teleport logs teleport-kube-agent-0 --previous
```

What came back was a Go runtime crash: a register dump, a hex stack, `goroutine 0 … [idle]`. Pages of it. And here's the first trap — I'd been reading it with `--tail`, which shows the *bottom* of the dump (registers and stacks), the least useful part. The cause is always on the first line:

```bash
kubectl -n teleport logs teleport-kube-agent-0 --previous 2>&1 | head -3
```

```text
runtime/cgo: pthread_create failed: Operation not permitted
SIGABRT: abort
```

**Lesson one:** when a container logs a crash dump, `head` it, don't `tail` it. The signal is the first line; everything after is the corpse.

## Ruling out the obvious wrong answers

`pthread_create failed: Operation not permitted` is not an application error, so I had to resist the usual suspects:

- **"The join token expired."** No. A bad token prints a readable one-line error and exits cleanly. A register dump means the process took a fatal signal before it ever got to joining anything. Re-signing the token three times (which I sheepishly did) changed nothing.
- **"The CPU is too old for the binary."** This NAS runs an Intel chip, and it was tempting to blame an illegal-instruction fault. But `SIGABRT` isn't `SIGILL`, and — the clincher — the *same Teleport version* was already running happily on this exact box as the SSH node. Same CPU, same kernel.
- **"The kernel is ancient."** `uname -r` said `5.10.60`. `clone3`, the syscall at the heart of this, landed in 5.3. The kernel was fine.

So: not the token, not the CPU, not the kernel. The binary runs on this hardware. It only dies *here*.

## A cause that fit

`pthread_create: Operation not permitted` is glibc reporting `EPERM` from the syscall it uses to spawn a thread. Modern glibc (2.34+) creates threads with **`clone3`**. If the container runtime's seccomp profile was built against an old `libseccomp` that predates `clone3`, the filter doesn't recognise the syscall number and — depending on its default action — returns `EPERM` instead of the `ENOSYS` that would let glibc fall back to plain `clone`. glibc treats `EPERM` as fatal, the C runtime can't make its first thread, and the process aborts before `main` does anything useful.

That mechanism is real, and it is exactly what was happening. Container Station's bundled containerd is from 2021; its default seccomp profile predates `clone3`. A **cgo / natively-threaded** image is the canary — Teleport is cgo, so it trips the wire immediately, while a pure-Go static binary creates threads through Go's own `clone` path and often sails through.

Everything up to here is correct. The next paragraph is where I went off the road.

## The verdict I shipped too early

I generalised. If the runtime's seccomp eats `clone3`, I reasoned, then *nothing* that spawns an OS thread through modern glibc can run on this cluster — not Teleport, not Redpanda, not Postgres — and since this k3s was going to be the dev tier of a multi-cluster lab where a Redpanda cluster got its first upgrade, the cluster had just failed its entrance exam. I even tried the two standard seccomp escape hatches: `seccompProfile: Unconfined`, which *appeared* not to help, and `privileged: true`, which the chart's PodSecurityPolicy refused. Both "failures" hardened the story. I wrote the whole thing up as a dead end and moved on to running the agent elsewhere.

Every sentence in that paragraph is either wrong or unproven, and I shipped it as a conclusion.

## The test I should have run first

Here is the experiment that takes thirty seconds and that I skipped in favour of a narrative. Run one throwaway pod that does nothing but spawn a thread, once with no seccomp profile and once with the runtime default:

```bash
# default on this k8s 1.21 is Unconfined — no profile imposed
kubectl run t --image=python:3.12-slim --restart=Never --rm --attach -- \
  python3 -c "import threading; t=threading.Thread(target=lambda:None); t.start(); t.join(); print('THREAD_OK')"
# THREAD_OK
```

```bash
# same image, but ask for the runtime's default seccomp profile
kubectl run t --image=python:3.12-slim --restart=Never --rm --attach \
  --overrides='{"spec":{"securityContext":{"seccompProfile":{"type":"RuntimeDefault"}}}}' -- \
  python3 -c "import threading; threading.Thread(target=lambda:None).start()"
# RuntimeError: can't start new thread
```

Same image, modern glibc, one creates a thread and one can't. The variable isn't the cluster — it's the **seccomp profile**. The old `clone3`-eating filter only applies when a pod asks for `RuntimeDefault`. With no profile, which is the default on this k8s 1.21, `clone3` is never filtered and threads work fine.

So the Teleport agent didn't crash because "the cluster can't run threaded images." It crashed because the Helm chart sets `seccompProfile: RuntimeDefault` on its pod, as hardened charts do — and that one field, on this one old runtime, is the whole bug. And my `Unconfined` "escape hatch that didn't work"? I almost certainly patched the StatefulSet's template and then read the template back to confirm it, while the pod that kept crashing was never recreated with the new security context. I was reading my own edit, not the running container.

## What was actually true

- The cluster runs natively-threaded workloads just fine — Teleport, Redpanda, Chrome, anything — as long as the pod does **not** request `RuntimeDefault` seccomp. On this k8s that's the default, so most things just work; a hardened chart that pins `RuntimeDefault` is the thing to override to `Unconfined` here.
- The real limitations of this cluster are duller and smaller than the one I invented: it's an **end-of-life k8s 1.21**, bundled and version-locked by Container Station, so modern charts and APIs are the actual compatibility risk — and running `Unconfined` trades away the seccomp sandbox, which is a real if acceptable cost on a lab cluster behind an authenticating proxy.

The dev-tier question still stands on those grounds, but not on the one I'd failed it for.

## What I'm keeping from this

- **A root cause you haven't isolated with a controlled test is a story, not a diagnosis.** I had a mechanism that fit every symptom, which is exactly the condition under which you're most likely to stop too early. "Consistent with" is not "caused by." Change one variable, hold the rest, and watch — *before* you write the conclusion down, let alone publish it.
- **Read crash dumps top-down.** `head -3`, not `--tail`. That part I'd still tell anyone. The signal and the first error line are at the top; registers are noise.
- **Smoke-test the runtime with a throwaway pod before you build on it — and before you rule it out.** One `kubectl run` that spawns a thread would have turned a false "this cluster is disqualified" into a true "this chart needs `Unconfined` here." I wrote this exact lesson down the first time; the embarrassing part is that I wrote it *instead of* doing it.
- **Verify a fix landed on the running thing, not the template.** A patch you can read back in the spec is not proof the crashing pod ever restarted under it. Check the live object.
