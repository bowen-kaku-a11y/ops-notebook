---
title: "From Console Clicks to Code: Adopting a Live VM into Terraform"
date: 2026-10-12
description: "Our practice cluster's apps were already GitOps-managed, but the VM underneath was pure console clicks. We adopted it into Terraform — instance first, firewall second — without rebuilding anything, and kept the two mistakes that taught us the most."
---

# From Console Clicks to Code: Adopting a Live VM into Terraform

Our little practice stack had a split personality. Everything *inside* the cluster was already code: [Argo CD reconciles the cluster's contents from Git](/posts/putting-argocd-behind-teleport/), and upgrading Argo CD itself is a one-line change to a pinned version. Everything *under* the cluster was console clicks: a free-tier ARM VM (an OCI `VM.Standard.A1.Flex`, 2 OCPU / 12 GB) created by hand, its virtual network built by a setup wizard, its firewall rules whatever the wizard chose. If that VM died, the rebuild procedure was "remember what you clicked."

The goal was adoption, not migration: bring the existing VM and its network under Terraform **without recreating anything**. The k3s node and the demo app it serves — a static page behind NodePort 30080 — were not supposed to notice.

## Recon before code

Terraform talks to OCI with an API signing key, so step zero was generating a key pair in the console and keeping the private key on the workstation, nowhere else. After that, the first configuration we wrote contained no resources at all — just data sources:

```hcl
data "oci_core_vcns" "discovery" {
  compartment_id = var.tenancy_ocid
}

data "oci_core_subnets" "discovery" {
  compartment_id = var.tenancy_ocid
}
```

plus outputs projecting their names, IDs, and CIDR blocks. A `terraform plan` at this point changes nothing; it is a map. It proved the credentials worked and told us exactly which VCN (a /16) and subnet (a /24) the VM lives in — facts every later step would be checked against.

## Lesson one: import writes the ledger, not the world

Adopting the instance used Terraform's declarative import. An `import` block names the resource we intend to manage and the real object's ID; the configuration for it does not exist yet, and that is deliberate — we let Terraform draft it:

```
terraform plan -generate-config-out=generated.tf
```

The generated file is a claim, not a fact, so we reviewed it against the recon data before believing a word: shape `VM.Standard.A1.Flex`, 2 OCPU and 12 GB in `shape_config`, and the subnet ID in `create_vnic_details` matching the one discovery had printed. All three matched. Renamed to `instance.tf`, the sequence `plan` → `apply` → `plan` ended with the most satisfying line in Terraform:

```
No changes. Your infrastructure matches the configuration.
```

That apply created nothing. An import is a ledger entry: a resource that already existed is now registered as managed, with its real ID recorded in state. From here on, if someone edits this VM in the console, `plan` will report the drift. The instance had been "wild"; now it is on the books.

## The firewall is a different animal

Opening the demo to the internet meant editing the subnet's security list — OCI's firewall rule set. Two unknowns had to fall first.

**Which list?** Our discovery outputs did not project the one field we needed. Subnets carry a `security_list_ids` attribute, and a VCN carries a `default_security_list_id`; adding one line to each output and re-running `plan` produced two IDs. They were identical, which settled it: the subnet uses the VCN's **default** security list (the display name, visible once we generated its config, confirmed it: "Default Security List for …"). That matters because a default list is a different Terraform resource (`oci_core_default_security_list`) than a standalone one.

**What is already in it?** This is the step that makes firewall adoption safe. In Terraform, a security list is *fully authoritative*: the configuration is the entire rule set, and anything not in the configuration is removed on apply. Hand-writing a config containing only our new rule would have deleted SSH. So we harvested first — import block plus `-generate-config-out` again — and got the complete existing set: one egress rule (all protocols, everywhere) and three ingress rules (ICMP from inside the VCN, ICMP type 3 code 4 from anywhere for path MTU, and TCP 22 from anywhere). Only then did we append exactly one rule:

```hcl
ingress_security_rules {
  protocol    = "6"
  source      = "203.0.113.7/32"
  description = "demo NodePort 30080"
  tcp_options {
    max = 30080
    min = 30080
  }
}
```

Port 30080, TCP, from one home IP as a /32 — not `0.0.0.0/0`. A practice demo does not need the whole internet, and least privilege costs nothing here: if the home IP changes, the fix is editing this one line and re-applying.

## Two mistakes, both cheap

The plan for this change *looked* alarming: the three existing ingress rules appeared as removed (`-`) and re-added (`+`). Reading them field by field, every re-added rule was identical to its removed twin; the only difference was a `description` shown as "(known after apply)". Rule collections here are unordered sets with a computed field, so adding one element re-renders the others. The real gate was the summary line — **1 to import, 0 to add, 1 to change, 0 to destroy** — plus that field-by-field check. The update itself is a single API call carrying the full list, applied atomically.

We had also made two genuine errors on the way, and both are worth keeping:

- **Wrong import ID.** We first tried importing the default security list using the VCN's ID and got `Cannot import non-existent remote object`. For this provider, the import ID is the security list's own ID — and the generated config set `manage_default_resource_id` to that same list ID, contradicting what we remembered from the documentation. When generated config and recollection disagree, believe the generator; it was derived from live state.
- **A bare IP is not a CIDR.** The new rule's source went in as a plain IP address, and `plan` displayed it without complaint. `apply` then failed with a 400 from the API: *unable to parse*. Plan validates structure; [only the API validates values](/posts/terraform-new-machine-401/), and a CIDR needs its prefix — `203.0.113.7/32`, not `203.0.113.7`. The failure cost nothing: the import had already landed, the rejected update changed zero rules, and the retry was a one-character-class fix.

## Where this leaves the stack

The demo now answers at the node's public address on port 30080 — from exactly one source IP, granted by exactly one rule, which lives in exactly one file. Both layers of the practice stack are code now: Terraform below the cluster, GitOps inside it, each layer able to report drift in its own domain.

The honest summary of the method: recon with read-only data sources, adopt with import blocks, harvest before you edit anything authoritative, gate every apply on the plan summary, and treat failed applies as information rather than damage. Next rung on the ladder: the opposite direction — create a resource from scratch with Terraform, use it, and destroy it on purpose. Adoption taught us how Terraform meets reality; creation is where it starts making it.
