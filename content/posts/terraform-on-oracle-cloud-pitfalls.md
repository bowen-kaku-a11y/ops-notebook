---
title: "Terraform on Oracle Cloud: Eight Pitfalls I Hit So You Don't Have To"
date: 2026-10-23
description: "A free-tier OCI lab, Terraform managing instances, VCN and security rules, and a state file that later moved to OCI Object Storage's S3-compatible backend. Eight traps from that work: an import ID that is not what you assume, a 401 with two stacked causes, a plan that never goes clean, free-tier capacity rules nobody writes down, and an S3 backend that rejects every upload with 501 NotImplemented."
---

# Terraform on Oracle Cloud: Eight Pitfalls I Hit So You Don't Have To

My lab runs on Oracle Cloud's free tier: one small ARM instance, a VCN, and Terraform managing everything underneath — instances, security rules, public IPs. Later the state file moved off my workstation into OCI Object Storage, behind its S3-compatible API.

None of the pitfalls below are exotic. Each one cost real debugging time because the error message pointed somewhere unhelpful, or because an assumption imported from another cloud turned out to be wrong. Every section follows the same shape: the symptom, why it happens, and the fix that stuck.

## 1. The default security list imports under its own OCID

**Symptom.** Adopting a live VCN into Terraform, the import of the *default* security list fails or produces a generated config that makes no sense — even though you passed what looks like the right identifier.

**Why it happens.** A VCN has a default security list, and it is tempting to assume the VCN's OCID identifies it. It doesn't. The security list is its own resource with its own OCID, and that OCID — not the VCN's — is the import ID.

**Fix.** Copy the security list's OCID from the console or from the provider's generated output, and trust the generated config over your assumption about which identifier "should" work:

```bash
terraform import \
  oci_core_default_security_list.main \
  ocid1.securitylist.oc1..xxxx
```

The broader lesson recurs in this list: OCI identifiers look interchangeable in a console listing and are not.

## 2. A bare IP is not a source — the API wants CIDR

**Symptom.** Adding an ingress rule whose source is a single host address fails with a 400 from the OCI API.

**Why it happens.** The security-rule API expects a CIDR block, always. A bare address like `203.0.113.10` is rejected as malformed — there is no "single host" shorthand.

**Fix.** Write the host as a /32:

```hcl
ingress_security_rules {
  protocol = "6" # TCP
  source   = "203.0.113.10/32"
}
```

One character class of difference, one hard API error. Every source in every rule gets a prefix length, no exceptions.

## 3. A 401 with two causes stacked on top of each other

**Symptom.** The OCI provider returns 401 Unauthorized. The API key is correct, the fingerprint matches, and it still fails — fix one thing and it *still* fails.

**Why it happens.** There were two independent faults. First, my workstation's clock was about 30 minutes off, and OCI rejects request signatures outside a small time window. Second, the `user_ocid` in `terraform.tfvars` was a garbled near-copy of the **tenancy** OCID, not the user OCID at all — diffing tfvars against `~/.oci/config` exposed it. Either fault alone produces the same 401, so fixing one changed nothing observable.

**Fix.** Sync the clock, and never hand-copy identifiers again — pull the values mechanically from the CLI config the provider already trusts:

```bash
# ~/.oci/config is the source of truth;
# tfvars gets its values from there,
# by script or copy-paste of whole lines,
# never by retyping.
```

When an auth error survives a correct-looking fix, suspect a second cause instead of re-checking the first one harder.

## 4. The plan that never goes clean: floating image references

**Symptom.** Weeks after a successful apply, `terraform plan` starts showing a diff on an instance you haven't touched — and it never goes away.

**Why it happens.** The configuration looks up the OS image through a data source that returns *the latest* published image. Oracle publishes a new image, the data source returns a new ID, and plan dutifully reports drift. But an instance's boot image is fixed at creation; applying the diff would not — and should not — change anything real.

**Fix.** Pin the image to an explicit OCID, or tell Terraform the attribute is settled at birth:

```hcl
lifecycle {
  ignore_changes = [
    source_details[0].source_id
  ]
}
```

A data source that means "latest" is a diff generator. Anything it feeds needs a pin or an exemption.

## 5. Free-tier capacity is a pool, and it is per-domain

**Symptom.** An instance creation fails with `LimitExceeded` in a tenancy that is visibly using almost nothing — and the console never labels your ARM instances "Always Free" the way the marketing page implies.

**Why it happens.** Two separate rules collide. First, the A1 allowance is a **single tenancy-level pool** — but its size changed: since 15 June 2026 Oracle's docs scope the Always Free allowance to 1,500 OCPU-hours / 9,000 GB-hours a month (≈2 OCPU / 12 GB), half the 3,000 / 18,000 figure older guides quote. Whether paid tenancies still get the old 4 / 24 is unconfirmed — reported support replies say yes, public docs don't say so. Check your own tenancy's entitlement in the console; don't copy either number blindly. The console doesn't badge A1 instances as Always Free; the badge belongs to specific free shapes. Second, capacity for the small free shape is accounted **per availability domain**: one AD can be exhausted while another has room.

**Fix.** Budget the pool as one pool when you plan instance sizes, and when a creation returns `LimitExceeded`, retry in a different availability domain before assuming the tenancy is full. In my case the second AD accepted the instance immediately.

## 6. The reserved-IP swap is a two-step dance

**Symptom.** You want to replace an instance's ephemeral public IP with a reserved one — and the console simply doesn't offer "reserved" as an option where you expect it.

**Why it happens.** The VNIC edit flow won't swap address types in one move. The Reserved option only appears *after* the ephemeral address has been released.

**Fix.** In order: create the reserved IP object first; edit the VNIC and choose **No public IP** (this releases the ephemeral address); edit the VNIC again and the Reserved option is now available — attach the one you created.

It is worth the dance. A reserved IP costs nothing, even while unassigned. An ephemeral IP is reclaimed when its instance dies — and a DNS record still pointing at it can silently end up resolving to a stranger's machine.

## 7. Security lists are subnet-scoped; NSGs are machine-scoped

**Symptom.** You need to open a port for exactly one machine, and the obvious tool — the subnet's default security list — would open it for every instance in the subnet.

**Why it happens.** Security lists attach to subnets, so every rule is subnet-wide. Network Security Groups (NSGs) attach to VNICs, which is the granularity per-machine rules actually need. One bonus gotcha: unlike security lists, **NSGs have no implicit egress allow**. An NSG with only ingress rules leaves the instance unable to reach anything outbound.

**Fix.** Leave the shared default security list alone; give each machine its own NSG, with explicit egress:

```hcl
# per-machine rules live in an NSG,
# bound to that machine's VNIC:
#   - ingress 443 from anywhere (webhook)
#   - ingress 22 from my admin /32 only
#   - egress: explicit allow-all —
#     an NSG allows no egress by default
```

Isolation by editing the shared list is how one machine's exception becomes every machine's exposure.

## 8. The big one: the S3 backend rejects every upload with 501

**Symptom.** State migrated to OCI Object Storage's S3-compatible endpoint, every write fails the same way:

```
501 NotImplemented:
AWS chunked encoding
not supported
```

It breaks the state upload — and, with `use_lockfile` enabled, it breaks the lock file first, so the failure looks like a locking problem.

**Why it happens.** Terraform's S3 backend is built on AWS SDK Go v2, which by default streams `PutObject` bodies using **aws-chunked encoding** so it can compute request checksums on the fly. OCI's S3 compatibility layer doesn't implement aws-chunked encoding, so it rejects *every* upload — state and `.tflock` alike.

**Fix.** Tell the SDK to only calculate checksums when required, which restores plain uploads with a known content length. Do this with environment variables in every shell that runs Terraform — there is no backend-argument version of these two settings; putting them in `backend "s3"` fails on Terraform 1.16.6 with `Unsupported argument`:

```bash
export AWS_REQUEST_CHECKSUM_CALCULATION=when_required
export AWS_RESPONSE_CHECKSUM_VALIDATION=when_required
```

```hcl
terraform {
  backend "s3" {
    bucket = "tf-state"
    key    = "root/terraform.tfstate"
    region = "us-phoenix-1"

    endpoints = {
      s3 = "https://<namespace>.compat.objectstorage.us-phoenix-1.oraclecloud.com"
    }

    skip_region_validation      = true
    skip_credentials_validation = true
    skip_requesting_account_id  = true
    skip_metadata_api_check     = true
    skip_s3_checksum            = true
    use_path_style              = true
    use_lockfile                = true
  }
}
```

Only with those variables set does `use_lockfile` work; before it, the lock file is simply the first upload to hit the 501.

## Closing

Read back over the list, a pattern emerges: most of these are not bugs, they are *contracts* — identifiers that must match exactly, sources that must be CIDR, capacity that is pooled in units nobody displays, an encoding the compatibility layer never promised to speak. The cloud works the way its API says it works, not the way the console layout suggests. The durable habits are boring ones: copy identifiers mechanically, pin anything that floats, and when an error survives one fix, look for the second cause.
