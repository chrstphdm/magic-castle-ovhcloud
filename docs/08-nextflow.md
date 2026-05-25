# Nextflow on Magic Castle

This section covers configuring Nextflow to submit jobs to the SLURM cluster deployed
by Magic Castle, and staging data from OVHcloud Object Storage.

---

## SLURM executor configuration

Create a `nextflow.config` in your pipeline or home directory:

```groovy
process {
    executor = 'slurm'
    queue    = 'cpubase'         // matches your SLURM partition name
    clusterOptions = '--account=default'
}

executor {
    name      = 'slurm'
    queueSize = 50               // max concurrent jobs submitted to SLURM
    pollInterval = '30 sec'      // how often Nextflow checks job status
    submitRateLimit = '10/1min'  // max job submissions per minute (be kind to slurmctld)
}
```

### Per-process resource requests

```groovy
process {
    executor = 'slurm'
    queue    = 'cpubase'

    withLabel: 'small' {
        cpus   = 2
        memory = '8 GB'
        time   = '2h'
    }
    withLabel: 'medium' {
        cpus   = 8
        memory = '32 GB'
        time   = '12h'
    }
    withLabel: 'highmem' {
        cpus   = 4
        memory = '64 GB'
        time   = '24h'
        queue  = 'highmem'       // send to the high-memory partition
    }
}
```

---

## Apptainer (Singularity) integration

With Apptainer installed on the cluster (see [Bioinformatics stack](06-bioinfo-stack.md)),
Nextflow can run each process inside a container automatically:

```groovy
apptainer {
    enabled    = true
    autoMounts = true
    cacheDir   = '/project/containers/cache'
}
```

- `autoMounts = true`: automatically mounts `/home`, `/project`, `/scratch` inside
  containers — avoids the common error of "file not found" inside a container.
- `cacheDir`: shared cache so containers are pulled once and reused by all users.

### Using a container per process

```groovy
process ALIGN {
    container 'docker://quay.io/biocontainers/bwa-mem2:2.2.1--he513fc3_0'
    ...
}
```

Nextflow pulls the Docker image and converts it to an Apptainer `.sif` file on first use.

---

## Data staging from OVHcloud Object Storage

### Using S3 as pipeline input/output

If your reference data and input files are in OVHcloud Object Storage (mounted via
S3FS or accessed directly), configure Nextflow's S3 support:

```groovy
aws {
    accessKey = '<your-ovhcloud-s3-access-key>'
    secretKey = '<your-ovhcloud-s3-secret-key>'
    client {
        endpoint = 'https://s3.gra.perf.cloud.ovh.net'
        s3PathStyleAccess = true   // required for OVHcloud — path-style, not virtual-hosted
    }
}
```

> **Security**: do not hardcode credentials in `nextflow.config`. Use environment
> variables instead:
> ```bash
> export AWS_ACCESS_KEY_ID=<key>
> export AWS_SECRET_ACCESS_KEY=<secret>
> ```
> Nextflow picks these up automatically.

### S3 as working directory

```groovy
workDir = 's3://<bucket-name>/nextflow-work'
```

This stores intermediate files in Object Storage instead of local disk — useful for
large pipelines where local scratch space is insufficient. Adds latency but removes
disk pressure.

### Mixed local + S3

The most common pattern for large genomics workloads:

```groovy
// Input: from S3 (reference data already there)
// Work: local NFS scratch (fast I/O during pipeline)
// Output: push back to S3

params {
    genome   = 's3://<references-bucket>/GRCh38/genome.fa'
    outdir   = 's3://<results-bucket>/run-20240115'
}

workDir = '/scratch/nextflow-work'  // local NFS
```

---

## SLURM job naming and accounting

Nextflow submits each process as a SLURM job. By default, job names are long and hard
to read in `squeue`. Add a job name prefix:

```groovy
process {
    executor       = 'slurm'
    clusterOptions = "--job-name=nf-${task.process.tokenize(':').last()}"
}
```

### Job tracking

```bash
# See all Nextflow jobs currently running
squeue -u <your-username>

# Past job details (after pipeline finishes)
sacct -u <your-username> --format=JobID,JobName,Elapsed,MaxRSS,State --starttime=2024-01-01
```

---

## Profiles for reproducibility

Define named profiles in `nextflow.config` to switch between local testing and cluster
execution without editing config files:

```groovy
profiles {

    standard {
        process.executor = 'local'
    }

    cluster {
        process.executor   = 'slurm'
        process.queue      = 'cpubase'
        apptainer.enabled  = true
        apptainer.autoMounts = true
        executor.queueSize = 50
    }

    test {
        process.executor = 'local'
        params.input     = 'test_data/'
        params.genome    = 'test_data/genome_chr22.fa'
    }

}
```

Run on the cluster:

```bash
nextflow run main.nf -profile cluster -params-file params.yaml
```

---

## Common issues

### Jobs submitted but never start

Check SLURM queue and node state:
```bash
squeue            # are jobs PENDING or RUNNING?
sinfo             # are nodes idle, drained, or down?
scontrol show node node1  # detailed node state
```

### Container not found

If Apptainer cannot pull a container:
```bash
# Check disk space on cacheDir
df -h /project/containers/cache

# Pull manually to see the error
apptainer pull docker://quay.io/biocontainers/<tool>:<tag>
```

### Pipeline stalls after Terraform apply

Nextflow was likely started before Puppet finished configuring SLURM.
Wait for `sinfo` to show nodes in `idle` state before submitting pipelines.
