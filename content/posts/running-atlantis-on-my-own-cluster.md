---
title: "Running Atlantis on My Own Cluster: Eight Lessons from One Long Evening"
date: 2026-10-23
description: "Atlantis installed on a small k3s ops node to drive Terraform for an Oracle Cloud infra repo, proven end-to-end with a comment-only pull request before any real change was allowed through. Eight lessons from that evening: a pod that runs happily on a dead token, a webhook chain debugged one gate at a time, a 401 caused by a group OCID, and a plan that wanted to destroy a VM over a variable Atlantis could not see."
---

# Running Atlantis on My Own Cluster: Eight Lessons from One Long Evening

The goal was PR-driven Terraform for my Oracle Cloud infra repo: Atlantis plans on every pull request, applies only when I comment, and leaves the whole audit trail in the PR itself. Atlantis runs on the small k3s node I keep for ops tooling — official Helm chart, managed by that node's own Argo CD. Atlantis also got its own least-privilege cloud identity: a dedicated user that can read broadly, manage the instance and network families, and touch objects in exactly one bucket — the one holding Terraform state — with no IAM rights at all.

Before any real change was allowed through this pipeline, it had to prove itself on a harness that could do no damage: a pull request that changed nothing but comments in both Terraform projects. Every gate had to fire in order — webhook delivery, signature check, repo-config parsing, clone, plan, locks — and both projects had to come back "No changes." It took one long evening. The comment-only PR flushed out five separate failures on its way through — one of them a plan to destroy a VM — which is the best argument I know for boring acceptance tests.

## 1. A `Running` pod says nothing about its credentials

**Symptom.** Atlantis was up: pod Running, health checks green, UI reachable. And completely unable to talk to GitHub.

**Why it happens.** The first GitHub token I pasted into the cluster Secret was simply wrong. With a personal access token, Atlantis does not validate it at startup — it boots happily and only discovers the problem when it tries to use it. Checked from inside the pod, with the token exactly as injected, GitHub's answer was unambiguous:

```
401 Bad credentials
```

**Fix.** Treat credential verification as a step of its own, not a side effect of the pod starting: exec into the pod and call the API with the injected environment before you declare the install done. "The container is up" and "the identity works" are separate claims, and only one of them was true.

## 2. Debug the webhook chain one gate at a time

**Symptom.** The acceptance PR opened — and no plan comment appeared.

**Why it happens.** A webhook crosses several gates, and each one fails with a different voice. GitHub's delivery log said the events *were* being sent; the Atlantis log said what happened to them. First contact had earned a 502 — the service genuinely wasn't up yet. Later deliveries got further:

```
400
payload signature check failed
```

The HMAC secret on the GitHub side and the one in the cluster Secret didn't match — two manual pastes of one long string, diverging somewhere. After that was fixed, deliveries passed signature and died at repo-config parsing instead (lesson 3). Every failure moved one layer deeper than the last — paradoxically, the sign of a healthy chain: each gate rejected the event for a *new* reason.

**Fix.** Read the two logs together — the sender's delivery record and the receiver's log line, matched by timestamp — and fix exactly the gate that's complaining. For the secret specifically: re-set **both** sides from the single value in the password manager, rather than guessing which paste went wrong.

## 3. A repo cannot lower its own apply gates

**Symptom.** With signatures passing, Atlantis rejected my repo's own config file:

```
parsing atlantis.yaml: repo config not allowed to set 'apply_requirements' key: server-side config needs 'allowed_overrides: [apply_requirements]'
```

**Why it happens.** My `atlantis.yaml` set `apply_requirements: [mergeable]` — apply only to mergeable PRs. Reasonable, but Atlantis treats the apply gate as a server-admin decision: if any repo could set its own requirements, a repo could also *weaken* them. So the key is refused unless the server-side repo config allows that repo to override it.

**Fix.** One stanza in the server-side repo config, scoped as narrowly as the error message suggests: this one repo, this one key, `allowed_overrides: [apply_requirements]`. Everything else stays default-denied. The error message, for once, is the documentation.

## 4. The 401 that wasn't an auth problem — it was an identity problem

**Symptom.** Plans finally ran, and every Oracle Cloud API call in both projects failed the same way:

```
Error: 401-NotAuthenticated, The required information to complete authentication was not provided or was incorrect.
```

Odd detail: the S3-compatible state backend worked fine. State read, plan started, refresh began — then the provider calls died.

**Why it happens.** I eliminated the usual suspects in order. Clock skew — the classic OCI signature killer — was ruled out: NTP synced, seconds off at most. The private key parsed and was the right key. The fingerprint: I derived it from the private key in the Secret, and it matched the configured fingerprint character for character. What was left was the user OCID — and there it was. The value injected into the Secret began with `ocid1.group.`. I had pasted the **group** OCID at the user prompt. The provider was signing requests while claiming to be a user group, and OCI was quite correctly unimpressed.

This is not the first time a hand-pasted OCID has cost me a 401: [rebuilding the Terraform workspace on a new machine](/posts/terraform-new-machine-401/) ended in the same place, with a near-twin identifier, and that post already concluded that identifiers should be diffed by machine, not by eye. I had written the lesson down; I had not yet built it in.

**Fix.** Paste the right OCID, obviously — but the durable fix is one layer up. Identifiers have type prefixes for a reason; a one-glance prefix check (`ocid1.user.` or it isn't a user) would have caught this in a second. Afterwards I added input guards to the deploy script: the user-OCID prompt re-prompts until the prefix is right, the fingerprint prompt validates the colon-hex format, and the GitHub token is verified against the API *before* the Secret is written (lesson 1, automated). Paste errors should fail at the prompt, not three debugging rounds later wearing a different costume.

## 5. A local-only override is invisible to Atlantis

**Symptom.** The root project planned clean. The second project — a disposable lesson VM — did not:

```
Plan: 1 to add, 0 to change, 1 to destroy.
# oci_core_instance.lesson3 must be replaced
```

**Why it happens.** That VM lives in the second availability domain, because when I created it, the first AD had no capacity for the free shape and I passed `ad_index = 1` — locally, as a command-line override, never written into the code. Atlantis ran with the code default, `0`, computed the first AD, and since availability domain is a recreate-only attribute, it planned to destroy the VM and rebuild it in the wrong AD. Only one diff line carried the `# forces replacement` marker: the AD. The floating "latest OS image" reference did *not* cause this — an existing `lifecycle { ignore_changes }` on the image held firm; its diff lines were knock-on noise from the replacement.

**Fix.** Codify reality: the variable default in the code now says `1`, with the reason written in its description. Any value that exists only in one runner's local invocation is a trap for every other runner — my laptop, Atlantis, a future CI — and plan will find it at the worst moment.

## 6. Plan locks are a feature with sharp edges

**Symptom.** The `ad_index` fix went up as its own PR — and couldn't plan at all. Atlantis told me why, politely:

```
This project is currently locked by an unapplied plan from pull #10. To continue, delete the lock from #10 or apply that plan and merge the pull request.
```

**Why it happens.** The acceptance PR's successful plans had locked both projects — correctly: Atlantis won't let two PRs race applies against the same workspace. But one of those plans was the VM-destroying one. Applying it was never an option, so the locks had to be discarded deliberately, via `atlantis unlock` on the holding PR. My first attempt at this command was `atlantis unclock`, which earned a courteous "unknown command" reply listing the real ones. Even the typo was instructive: the bot answers in the PR, in the open, where the mistake becomes part of the record.

**Fix.** Respect the lock workflow: unlock discards plans and releases locks without applying anything; then re-plan the PR you actually care about. With the locks released, the fix PR planned a clean, code-only change and merged. And read lock messages as information — they name the holding PR, usually the thing you forgot you left open.

## 7. Atlantis plans the branch head, not main

**Symptom.** With the `ad_index` fix on main, I re-planned the acceptance PR — and it *still* showed the destroy.

**Why it happens.** Atlantis checks out the pull request's branch, verbatim. The acceptance PR's branch was two commits behind main and simply did not contain the fix yet. Nothing in the GitHub UI had warned me: the PR showed "no conflicts, mergeable," which I had unconsciously read as "current." It isn't. GitHub only surfaces the out-of-date state prominently when a branch-protection rule requires branches to be up to date before merging — a setting this repo doesn't have.

**Fix.** Update the branch (one merge of main into it), let Atlantis re-plan, and watch both projects come back "No changes." The durable lesson: mergeable means *can* be merged, not *contains* main. Anything that reads the branch head — plans, CI, previews — sees the branch exactly as stale as it is.

## 8. Keep the identity boring, and never let the tool manage its own floor

**Symptom.** None — this is the lesson that produced no incident, which is rather the point.

**Why it matters.** Atlantis's cloud identity was itself created with Terraform, in a separate bootstrap workspace. The API key pair was generated on my own machine; Terraform uploaded only the public key, so the private key never entered state. The Object Storage secret key was deliberately kept *out* of Terraform entirely — the resource that creates one writes its value into state, which is writing the key on the door it locks. It lives in the password manager, injected into the cluster by hand. And the bootstrap workspace — the thing that creates Atlantis's host node and its identity — is never managed by Atlantis. A tool that can recreate the floor it stands on is a self-reference paradox with a blast radius.

**Fix.** None needed. Least privilege held up all evening: when things broke, they broke as *this* identity, in *this* scope, and the audit trail showed it.

## Closing: the acceptance PR earned its keep

It would have been easy to skip the comment-only PR and go straight to a real change. Had I done that, the first real PR through this pipeline would have carried a plan to destroy and recreate a VM in the wrong availability domain — behind an apply gate I might well have approved, because the rest of the plan looked fine. Instead, the harness PR caught the group-OCID paste and the invisible local override while the stakes were zero, and forced two permanent improvements on the way: server-side config scoped to one key, and a deploy script that validates what you paste into it.

That gap closed later the same day. On October 24, I commented `atlantis apply` on the comment-only PR, and Atlantis returned "Apply complete! 0 added, 0 changed, 0 destroyed" — the whole chain, from comment trigger through execution to result, with nothing to change. A real change PR will still be the first apply that actually changes something, with the mergeable requirement doing its job, but the apply path itself is no longer untested. Everything upstream of that moment — delivery, signature, parsing, auth, planning, locking — has now failed at least once, in public, in the PR, and been fixed. That's what an acceptance test is for.
