---
title: "The Validation That Didn't: Two set -e Bugs That Passed Their Tests"
date: 2026-11-12
description: "I wrote a sync script that pulls a list of network routes from git, validates each line, and refuses to apply bad data. It had two bugs that let bad data through anyway — and the unit tests went green on both. The culprits were a process substitution that couldn't abort its parent, and an IP octet quietly parsed as octal."
---

# The Validation That Didn't: Two set -e Bugs That Passed Their Tests

I automated a small chore on a VM I run: a timer pulls a list of network routes from a git repo every few minutes, validates every line, renders a config file, and reloads the service only when the content actually changed. The whole point of the script is to be *safe* — to refuse bad data rather than push a broken config to live clients. I wrote it with `set -euo pipefail` at the top, a validation function that rejects any malformed or disallowed entry, and a rollback path for good measure.

Then I tested it against a fake repo, fed it deliberately broken input, and watched it do the wrong thing twice — while my first round of tests reported success. Both bugs are the kind that reads as correct until you make it fail on purpose. Here's each one.

## Bug one: a `die` that couldn't kill the script

The validator read a file line by line and rejected the whole batch on the first bad entry:

```bash
validate() {        # prints clean lines to stdout, or dies on bad input
  while read -r line; do
    is_valid "$line" || die "bad entry: $line"
    echo "$line"
  done < "$1"
}
```

The caller fed it through a loop that rendered each validated line into the output config:

```bash
while read -r entry; do
  render "$entry" >> "$out"
done < <(validate "$file")      # <-- process substitution
```

Feed this a file with a bad line in the middle, and the expected behavior is: `die` fires, the script exits non-zero, the output file is never written. What actually happened: the output config got the lines *before* the bad one and then just... stopped, cleanly, exit 0. Half a route list, applied to live clients, reported as success.

The mechanism is the process substitution `< <(validate "$file")`. That `validate` runs in a **subshell**. When it calls `die` (a `printf` to stderr plus `exit 1`), the `exit` terminates *the subshell*, not the parent. The parent's `while read` loop sees the pipe close, concludes "end of input," and carries on with whatever it managed to read. `set -e` doesn't save you here: the subshell's non-zero exit is the exit status of a process-substitution producer, which the parent never checks. The guard was real; its death was local.

The fix is to run the validator somewhere whose exit status the parent actually inspects — a command substitution in an assignment:

```bash
clean=$(validate "$file") || exit 1    # parent sees the failure
while read -r entry; do
  render "$entry" >> "$out"
done <<< "$clean"
```

Now `die` inside `validate` ends the subshell with status 1, the `$(...)` assignment propagates that status, and `|| exit 1` stops the main script before a single line is rendered. Same validator, different plumbing — and the plumbing was the whole bug.

## Bug two: an IP octet that was secretly octal

The per-line validator checked, among other things, that each dotted-decimal octet was in range:

```bash
for octet in ${ip//./ }; do
  (( octet <= 255 )) || return 1
done
```

This looks unimpeachable. It is not. Bash arithmetic treats a numeric literal with a **leading zero as octal**. So an octet written `08` or `09` isn't "eight" or "nine" — it's an invalid octal literal, and `(( 08 <= 255 ))` throws:

```
value too great for base (error token is "08")
```

Under `set -e`, that thrown error aborts the entire script — not with my friendly `bad entry:` message, but with a raw Bash arithmetic error, mid-run. One mistyped line in the route file (`192.0.08.0/24`) takes down every sync, with an error message that points at Bash internals instead of at the offending line.

And the quieter half: `001` *is* a valid octal literal — it's one. So `192.0.2.001` sails through validation as though it were `192.0.2.1`, which is a different address. The check that was supposed to be strict was simultaneously too fragile (crashes on `08`) and too lax (accepts `001`).

Two fixes stack cleanly. Reject leading zeros at the format check — no legitimate dotted-decimal octet has them — so a lone `0` is fine but `08` and `001` are rejected as malformed before arithmetic ever runs. Then, as defense in depth, force base-10 in the comparison so arithmetic can never reinterpret a value:

```bash
(( 10#$octet <= 255 )) || return 1
```

`10#08` is eight, `10#001` is one, and nothing throws. Belt and suspenders: the regex rejects the shapes that cause trouble, and the `10#` makes the arithmetic honest even if something slips past.

## There's a third one, and it's the same family

While I was in there, a related trap: counting matches with `grep -c` under `set -e`.

```bash
count=$(grep -c '^route' "$file")
(( count > 0 )) || die "no routes found — refusing to apply"
```

`grep` exits **1 when it finds zero matches**. In an assignment, that non-zero status still trips `set -e`, so on an empty file the script aborts on the `grep` line — and the friendly `die` below it, the one that was supposed to explain "no routes found," is dead code it never reaches. The fix is a boring `|| true`:

```bash
count=$(grep -c '^route' "$file" || true)
```

The common thread across all three: `set -e` and the commands that *legitimately* return non-zero (`grep` finding nothing, `cmp`/`diff` reporting a difference, a validator in a subshell) interact in ways that are invisible until the non-zero case actually occurs. In an `if` condition `set -e` is suppressed, so these feel safe — but in a bare statement or an assignment, they're live wires.

## Why the tests went green

Here's the part that bothers me most. I had unit tests. They passed. They passed *because they weren't specific enough.*

The empty-file test asserted two things: the script exits non-zero, and the output config is unchanged. Both were true whether the friendly guard fired **or** the script aborted on the raw `grep` error — the test couldn't tell those two worlds apart. It confirmed the symptom (didn't apply bad data) while the mechanism underneath (which safeguard actually stopped it) was broken. A test that checks "it failed" without checking *how* it failed will happily bless the wrong failure path.

The bad-data test had the same shape until I tightened it. The fix was to assert on the **specific** message — `grep "no routes found"`, not just "exit code non-zero" — so the test fails if the script aborts for the wrong reason. The moment I did that, the dead-code guard revealed itself.

(A smaller cousin of the same mistake bit me in another test: I was grepping output for `"offline"` to confirm a negative result, not noticing that the positive message, `"online"`, is a **substring** of it. The grep matched both pages. Full-phrase assertions only.)

## What I'm keeping

Three concrete reflexes came out of this, and I wrote them down where I'll see them before the next shell script:

1. **A validator that must abort the whole run can't live in a process substitution.** Capture it with `$(...)` and check the status, or the `die` only kills a subshell nobody is listening to.
2. **Never do bare `(( n <= x ))` on externally-sourced numbers.** Leading zeros mean octal; use `10#$n`, and reject leading zeros at the format check too.
3. **Tests must assert on the specific failure, not just that something failed.** "Exited non-zero and didn't apply" is satisfied by the correct guard *and* by a crash two lines earlier. If the test can't tell them apart, it isn't testing the guard.

The meta-lesson is older than any of these: a safety check you never watched fail is a hypothesis, not a safeguard. The bugs weren't in the validation logic I was proud of — they were in the wiring around it, the part that decides whether "this is invalid" actually stops anything. I only found them by feeding the script the exact garbage it was built to reject and watching, byte by byte, what came out the other side.
