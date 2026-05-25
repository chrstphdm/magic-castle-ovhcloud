# Magic Castle on OVHcloud

A community guide for deploying a SLURM HPC cluster on OVHcloud using
[Magic Castle](https://github.com/ComputeCanada/magic_castle).

---

## What this guide is

Magic Castle is a Terraform-based tool that deploys a fully configured SLURM cluster on
cloud infrastructure. It handles node provisioning, network setup, shared storage, user
management, and software stack configuration — in one `terraform apply`.

OVHcloud is an officially supported provider, but the existing documentation leaves several
gaps that will cost you hours. This guide fills them.

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
- [ ] A project with sufficient [quotas](#quotas) (vCPUs, RAM, volumes, floating IPs)
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

## Quick links

- [Magic Castle GitHub](https://github.com/ComputeCanada/magic_castle)
- [Magic Castle documentation](https://computecanada.github.io/magic_castle/)
- [OVHcloud Public Cloud](https://www.ovhcloud.com/en/public-cloud/)
- [OpenStack Terraform provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest)
