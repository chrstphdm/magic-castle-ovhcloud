# Magic Castle on OVHcloud

Community guide for deploying a SLURM HPC cluster on OVHcloud using
[Magic Castle](https://github.com/ComputeCanada/magic_castle).

**Read the full guide**: https://chrstphdm.github.io/magic-castle-ovhcloud/

---

## What this guide covers

Magic Castle officially supports OVHcloud but the documentation leaves gaps.
This guide fills them based on real production deployments.

Topics covered:
- Account setup and quota management
- Network pre-requisites (the step most people miss)
- Annotated `main.tf` for OVHcloud with flavor selection guide
- OVHcloud-specific friction points (volume types, regions, credential encryption)
- Adding Nextflow, Apptainer, and S3FS via a custom Puppet fork
- Production SLURM tuning (scheduler, priorities, autoscaling decisions)
- Running Nextflow pipelines on the cluster with OVHcloud Object Storage
- Safe deployment and destruction workflow
- Troubleshooting OVHcloud-specific errors

## Quick links

- [Magic Castle GitHub](https://github.com/ComputeCanada/magic_castle)
- [Magic Castle documentation](https://computecanada.github.io/magic_castle/)
- [OVHcloud Public Cloud](https://www.ovhcloud.com/en/public-cloud/)

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
