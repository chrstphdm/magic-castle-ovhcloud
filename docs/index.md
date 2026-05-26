# Magic Castle on OVHcloud

A community guide for deploying a SLURM HPC cluster on OVHcloud using
[Magic Castle](https://github.com/ComputeCanada/magic_castle).

> **Tested with Magic Castle 12.2.0** on OVHcloud regions GRA and SBG.
> If you use a different version, some configuration options or behavior may differ —
> check the [Magic Castle changelog](https://github.com/ComputeCanada/magic_castle/releases).

---

## Why this guide exists

I've been deploying Magic Castle clusters on OVHcloud for years — for genomics pipelines,
bioinformatics platforms, and production HPC workloads. Every time, I hit the same walls:
scattered documentation, OVHcloud-specific quirks that nobody writes about, and hours lost
to problems that a single paragraph could have prevented.

This guide is the result of that accumulated experience. It's the document I wish I'd had
on day one.

**What you'll find here** is a mix of two things:

1. **The essentials from official docs, consolidated in one place.** Magic Castle, OVHcloud,
   OpenStack, Terraform, SLURM — the relevant bits from each, assembled into a single
   coherent walkthrough instead of five browser tabs.

2. **The things that aren't in any official doc.** The network setup that must happen
   *before* `terraform apply`. The volume types that silently don't work. The flavor naming
   inconsistencies across regions. The SLURM tuning that turns a demo cluster into a
   production one. These are lessons learned the hard way, over many deployments and many
   hours of debugging.

---

## Who this is for

**If you know SLURM but not Terraform**: this guide explains what Terraform does, why Magic
Castle uses it, and walks through each configuration step without assuming cloud experience.

**If you know Terraform but not SLURM**: this guide explains the SLURM architecture, what
each node type does, and what configuration choices matter for a bioinformatics workload.

---

## Prerequisites checklist

Before starting, make sure you have:

- [ ] An OVHcloud Public Cloud project (not a dedicated server — the Public Cloud section)
- [ ] A project with sufficient [quotas](01-prerequisites.md#quotas) (vCPUs, RAM, volumes, floating IPs)
- [ ] An SSH key pair registered in the OVHcloud console
- [ ] [Terraform](https://developer.hashicorp.com/terraform/install) ≥ 1.5.7 installed locally
- [ ] The OpenStack RC File **v3** downloaded from your OVHcloud project
- [ ] [Git](https://git-scm.com/) to clone Magic Castle

---

## Guide sections

| Section | What you will do |
|---------|-----------------|
| [Prerequisites](01-prerequisites.md) | Account setup, quotas, SSH keys, Terraform install |
| [Architecture](02-architecture.md) | What nodes exist and why |
| [Network setup](03-network-setup.md) | Create network/subnet/router before running Magic Castle |
| [main.tf walkthrough](04-main-tf.md) | Configure your cluster for OVHcloud |
| [OVHcloud specifics](05-ovhcloud-specifics.md) | What the official docs don't tell you |
| [Bioinformatics stack](06-bioinfo-stack.md) | Add Nextflow, Apptainer, and S3FS |
| [SLURM tuning](07-slurm-tuning.md) | Production scheduler configuration |
| [Nextflow on Magic Castle](08-nextflow.md) | Run pipelines on your cluster |
| [Deploy and destroy](09-deployment.md) | Safe deployment workflow |
| [Troubleshooting](10-troubleshooting.md) | OVHcloud-specific errors and fixes |

---

## About the author

I'm [Christophe Demay](https://chrstphdm.github.io/), a senior bioinformatics consultant.
I've spent years building and operating HPC clusters on OVHcloud for genomics and
bioinformatics workloads — from proof-of-concept setups to production platforms running
large-scale Nextflow pipelines.

This guide condenses everything I've learned into a single resource: the tips, the
workarounds, the configuration choices that actually matter, and the mistakes you don't
need to repeat. Every code example and configuration snippet has been written from scratch
using publicly available documentation — no proprietary code or client-specific material
is included.

I wrote this because the community deserves a straightforward path from zero to a working
cluster, without the hours of trial and error I went through.

---

## Quick links

- [Magic Castle GitHub](https://github.com/ComputeCanada/magic_castle)
- [Magic Castle documentation](https://computecanada.github.io/magic_castle/)
- [OVHcloud Public Cloud](https://www.ovhcloud.com/en/public-cloud/)
- [OpenStack Terraform provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest)
