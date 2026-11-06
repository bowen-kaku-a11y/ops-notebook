---
title: "The Username That Vanished: Five Layers of Debugging a Browser SSH Gateway"
date: 2026-10-16
description: "We put a web SSH client behind an auth gateway so the lab cluster needs no public ports. It then failed in five different ways — a crash loop, a proxy rewriting Host, a missing key, and finally an upstream bug that silently dropped the username from the ssh command whenever a ?host= override was used."
---

# The Username That Vanished: Five Layers of Debugging a Browser SSH Gateway

The practice cluster's admin surfaces have been moving [behind an authenticating proxy](/posts/putting-argocd-behind-teleport/): log in once, pass MFA, and the proxy forwards you to web UIs that listen on no public port at all. The natural next step was a browser-based SSH client (WeTTY) next to them — open `ssh.lab.example.com`, get a terminal, connect to a default lab VM, or append `?host=<ip>` to reach any other machine. A ClusterIP Service, one more application registration in the proxy, done.

It took five separate fixes, in five different layers, before a browser session survived end to end. Each layer lied a little: the page rendered, [the pod was Running](/posts/running-atlantis-on-my-own-cluster/), the handshake returned 200 — and the session still died. This is the debugging trail, layer by layer, because the *order* the evidence arrived in is the useful part.

## Layer 1: it refuses to start, on principle

The first rollout CrashLooped. The current WeTTY build exits at startup unless at least one allowed origin is configured — a sensible hardening default that is, unfortunately, mandatory. One argument later (`--allowed-origin https://ssh.lab.example.com`) the pod stayed up.

## Layer 2: a client that never SSHes

The page now loaded and showed a login prompt — inside the container. WeTTY runs as root in its image, and as root it defaults to spawning the local `login` program instead of `ssh`. Unless you pass `--force-ssh`, your "SSH client" is a terminal into its own pod. Worse, an earlier preconfigured version of this deployment had shipped without that flag too, which meant it had *never actually SSHed anywhere* — it just looked like it did. A UI that renders is not a client that connects.

## Layer 3: the proxy rewrites Host, the handshake dies

Through the proxy, the terminal was a black screen. The browser's network tab showed the socket.io handshake returning 403 — wrapped, confusingly, in a 503 synthesized by WeTTY's own service worker when its fetch failed offline-style. (The service worker was a red herring; it only reports failures, it doesn't cause them.)

The mechanism, once we read WeTTY's handshake code: if a request carries an `Origin` header, it must be on the allowlist. A same-origin GET — exactly what a browser sends for the polling handshake — carries *no* `Origin`, so WeTTY falls back to checking the `Host` header against the hosts derived from the allowlist. Our proxy, forwarding to the backend Service, rewrites `Host` to the cluster-internal name (`webssh.webtools.svc.cluster.local:3000`). That host was on nobody's allowlist, so every real browser handshake was rejected.

Why didn't testing catch it? Because the hand-rolled curl checks sent an `Origin` header explicitly — they passed through the first branch of the check and never exercised the fallback the browser actually hits. The fix was deliberately narrow: allowlist exactly that one internal host as a second origin. Not "disable origin checking" — one exact entry.

## Layer 4: the pod has no key, and the target takes nothing else

Handshake fixed, the prompt appeared — and the session exited within a second of typing a username. Reproducing the gateway's exact `ssh` invocation inside the pod produced the real error in the clear:

```
deploy@10.0.0.10: Permission denied (publickey,gssapi-keyex,gssapi-with-mic).
```

Two facts combine here. The target's sshd (a stock cloud image) offers *only* publickey authentication — there is no password prompt to fall back to, which is why the exit felt instantaneous. And the ssh process WeTTY spawns can only use keys that exist inside the pod — of which there were none.

The answer is the jump-host pattern: give the gateway its own identity. We generated a dedicated keypair for the tool. The private half lives only in a namespace Secret — never in git — mounted read-only as the pod's default identity file (mode `0400`), so pod rebuilds come back with the same identity. The public half goes into each target's `authorized_keys` once, with a recognizable comment; revoking the gateway on a target is deleting that one line. After the rollout, we verified from inside the new pod, with WeTTY's own ssh options, that `whoami` returned the expected user on two different targets — *before* asking a human to test the page again.

## Layer 5: the username that vanished

And still: with `?host=203.0.113.50` (a second test VM), the browser session exited instantly, while the same flow against the default target worked. This time we watched both ends of a live attempt simultaneously. The target's sshd log, at the exact second of the browser attempt, recorded:

```
Connection closed by authenticating user root 192.0.2.1 port 8146 [preauth]
```

*Root.* The gateway's debug output showed why — the command it generated was:

```
ssh -t -p 22 -o PreferredAuthentications=publickey,password ... -- 203.0.113.50
```

No `user@` at all, so ssh fell back to the pod's local user, root. The bug is in WeTTY's command builder: it first composes the destination as `user@default-host`, then spreads the URL overrides over the result object. A `?host=` override therefore replaces the *entire string*, silently discarding the `user@` half. Every username source — the interactive prompt, a `/ssh/<user>` path, a configured default — is clobbered the same way. It is an upstream bug, and it only fires on the override path, which is why every test that skipped `?host=` stayed green.

The fix ships as a ConfigMap that overlays a patched copy of that one file in the image: when an override host is present, the username is re-attached to it. The patch file carries a header documenting the bug and the maintenance rule it creates — on every image upgrade, diff upstream's file against the overlay, and delete the overlay the day upstream fixes the spread order. Verified in a throwaway pod before merging: generated command `deploy@203.0.113.50`, and the socket flow landed on a real shell prompt on the target.

## What the five layers taught us

**Test the path the user actually takes.** Three of the five layers hid behind tests that were *almost* the real thing: curl with a hand-set `Origin`, ssh typed by hand instead of spawned by the gateway, a URL without the parameter the user always uses. Every one of those tests was green while the product was broken.

**Read both ends' logs.** The client side said "exits immediately" through several rounds of guessing. The target's sshd log said "authenticating user root" in one line and ended the argument. Correlating two machines' clocks beat another hour of staring at the gateway.

**[Verify before you ask for a retest](/posts/the-rollout-that-never-happened/).** Each round of "try it now" that fails costs trust. The rule we ended with: the fix isn't ready for a human until the same command, the same headers, or the same URL form has passed under our own hands first.

Two honest limits. The password-auth path is still unverified — every target in this lab is key-only, so that branch has never run. And a mounted patch over an upstream file is technical debt with a known interest payment: it must be re-diffed on upgrades, or it will quietly preserve a bug upstream already fixed. Both are written down where the next maintainer will find them — in the patch header, next to the code.
