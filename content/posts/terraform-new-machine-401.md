---
title: "New Machine, Same Cloud: Rebuilding a Terraform Workspace After a 401"
date: 2026-10-16
description: "Our Terraform workspace lived on exactly one laptop — code uncommitted, state local. Rebuilding it on another machine ran into a 401 with two stacked causes: a clock thirty minutes slow, and a hand-copied OCID that was a near-twin of the wrong identifier."
---

# New Machine, Same Cloud: Rebuilding a Terraform Workspace After a 401

The [adoption work](/posts/terraform-import-adopt-live-vm/) ended well: the practice VM and its security list under Terraform, plan reporting `No changes`. One errand was left unfinished — the files adoption produced (the harvested security list config, the provider lock file) sat uncommitted in that laptop's clone, and the state lived there too, the only copy.

Today we are on a spare Linux VM with nothing on it: no Terraform, no API key, no clone. Rebuilding the workspace here turned into the most instructive debugging session of the series.

## The rule we set first: copy nothing

Two shortcuts present themselves: copy the API private key over from the old machine, and copy the state file with it. We rejected both.

**One key per machine.** Each machine generates its own RSA key pair and registers the public half; the private key never leaves the machine it was born on. A key is an identity *for a machine*: retire or lose the machine, revoke its key, and the others are untouched. A shared key copied everywhere has no such story.

**Regenerate instead of rescue.** Nearly everything in that workspace was *generated* — `plan -generate-config-out` projected the live cloud into HCL — and a projection can be re-projected anywhere the provider can authenticate. The genuinely stranded artifact was state, and declarative import rebuilds exactly the slice of it we need.

## The 401

Setup went by the book: install Terraform, generate a key pair, add the public key in the console (which writes a `~/.oci/config` for you), clone the repo, fill in `terraform.tfvars`. The first plan died with `401-NotAuthenticated` — "the required information to complete authentication was not provided or was incorrect."

The message accuses the credentials, so verify them locally, where proof is cheap:

```
openssl rsa -in ~/.oci/oci_api_key.pem \
  -pubout 2>/dev/null | diff - ~/.oci/oci_api_key_public.pem
openssl rsa -in ~/.oci/oci_api_key.pem \
  -pubout -outform DER 2>/dev/null | openssl md5 -c
```

The derived public key matched the uploaded file, and the MD5 fingerprint of the local private key matched the fingerprint in `tfvars` exactly. Key material: exonerated. But a request signature covers more than the key — it also binds the identity you claim and the time you claim it. Both were wrong.

## Fault one: a clock with no keeper

`date -u` read thirty minutes behind real UTC. Cloud signature validation allows only a few minutes of skew; beyond that, a perfectly signed request is indistinguishable from a replayed one, and gets rejected as an authentication failure. That is why a clock fault wears a 401 costume.

Nothing had ever been keeping this clock. `timedatectl set-ntp true` answered `NTP not supported` — no time service was installed at all, and the clock had been drifting freely since the VM was built. Installing chrony and stepping the clock (`chronyc -a makestep`) fixed the time; we re-ran the plan.

Still 401. The clock was a real fault — and a decoy: fixing it was necessary, not sufficient, and the dangerous move would have been stopping there.

## Fault two: the identifier that looked right

So we diffed the two files that are supposed to agree: `~/.oci/config`, written by the console flow, against `terraform.tfvars`, filled in by hand. Tenancy: same. Fingerprint: same. Region: same. `user_ocid`: different — and not randomly different. Schematically:

```
tenancy   ocid1.tenancy.oc1..aaaaaaaam7k2…wxyz
tfvars    ocid1.user.oc1..aaaaaaaam7k2…wxzy
config    ocid1.user.oc1..aaaaaaaab91d…pqrs
```

The value in `tfvars` was a near-twin of the *tenancy* OCID: the same long identifier body, a couple of characters transposed, wearing a `user` prefix. It named a user that does not exist, so the cloud found no key registered under that identity and returned 401 — correctly, from its side of the wire.

OCIDs run to ninety-odd characters of noise, and the human eye does not diff them — it pattern-matches the first familiar chunk and declares a match. This one had been retyped during setup, exactly where the error entered. The fix: stop transcribing identifiers, and pull the value from the file the console wrote:

```
USER_OCID=$(grep '^user=' ~/.oci/config | cut -d= -f2)
sed -i "s|^user_ocid.*|user_ocid = \"$USER_OCID\"|" terraform.tfvars
```

The next plan authenticated immediately.

## Recovery is a re-run, not a rescue

With authentication working, the adoption sequence ran again on this machine: import block, generate, twice over. One pleasant surprise: the instance config *had* been committed at some point — only the security list file and the lock file were genuinely stranded, and both are re-derivable.

The regenerated security list is the checkpoint: because generation harvests live state, it contained everything later sessions had added — the NodePort rule scoped to a single /32, the 443 rule from when the admin UI moved behind a Teleport deployment. Not the rule set of the day the file was lost; the rule set of today. Final plan:

```
No changes. Your infrastructure matches the configuration.
```

One discipline falls out: two state files now claim the same resources, so this machine's state is the working one; the other is never applied from again. Generated code is perfectly portable; state still insists on a single home — a remote backend, the honest next step.

## Proof: build something disposable

To prove the workspace end to end — creation, not just reads and imports — we made a resource from scratch in an isolated directory with its own state, so nothing it does can touch the adopted VM: a free-tier `VM.Standard.E2.1.Micro`, subnet located by display name rather than pasted OCID, this machine's SSH public key injected at launch.

The first apply failed with `400-LimitExceeded`. Not a capacity error — a quota one. Free-tier micro instances are limited *per availability domain*, and the first domain's free quota was already spoken for. One variable flip (`ad_index`) to a second domain, and the instance was created in under a minute; SSH in as `opc` confirmed it alive.

It stays for now, on purpose: its removal is written into its lifecycle — delete the resource block (and the outputs that reference it), apply, and let the plan's `1 to destroy` do the rest. No destroy-everything command for a workspace holding exactly one disposable thing.

## What the next machine switch looks like

- **Per-machine keys, registered and revoked separately.** Machines change; credential hygiene that assumes they don't will eventually strand you.
- **Generated code lives in git, committed the day it is generated.** A projection you can regenerate is only portable if the committed copy exists.
- **Debug a 401 in order:** key against fingerprint locally; the clock, including whether anything keeps it; then every identifier diffed against the console-written file, character for character, by machine.
- **Never retype an identifier that already exists in a file.** Copy it, or script the copy.

Next: give state its single home with a remote backend. After that, the workspace stops caring which machine we are sitting at.
