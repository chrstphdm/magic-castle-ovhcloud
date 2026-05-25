# SLURM Tuning

The default Magic Castle SLURM configuration is conservative. This section covers
production-grade settings for bioinformatics workloads on a cloud cluster.

---

## Autoscaling: enable or disable?

Magic Castle supports SLURM autoscaling — nodes are suspended when idle and resumed
when a job is submitted. This reduces cloud costs but adds job start latency (2–5
minutes for a node to resume on OVHcloud).

### When to disable autoscaling

Disable autoscaling (`SuspendTime=-1`) when:
- You need immediate job start (interactive analyses, time-sensitive pipelines)
- You pay by the hour and the suspend/resume overhead is not worth the saving
- Your cluster runs continuously during a project sprint

### When to keep autoscaling

Keep autoscaling when:
- The cluster is used intermittently (overnight pipeline runs, weekly analyses)
- Cost control is a priority
- Job latency of a few minutes is acceptable

### Configuration in Puppet

In your Puppet configuration (typically `slurm.conf` template):

```ini
# Disable autoscaling — nodes always stay up
SuspendTime=-1

# Enable autoscaling — suspend after 5 minutes idle
# SuspendTime=300
# ResumeTimeout=600    # how long to wait for a node to resume before marking it down
```

---

## Scheduler: backfill

SLURM's default scheduler is FIFO (first in, first out). For a shared cluster with
mixed job sizes, **backfill** scheduling fills gaps left by large jobs with smaller
jobs that can complete before the large job's resources are needed.

```ini
SchedulerType=sched/backfill
SchedulerParameters=bf_continue,bf_max_job_test=200,bf_resolution=600
```

- `bf_continue`: allows backfill to run continuously in the background
- `bf_max_job_test`: maximum jobs to consider in each backfill cycle
- `bf_resolution`: time resolution for backfill (in seconds)

---

## Multi-factor priority

The default SLURM priority is FIFO. For a shared cluster where fairness matters
(multiple users or projects), enable multi-factor priority (MFPQ):

```ini
PriorityType=priority/multifactor
PriorityWeightJobSize=100
PriorityWeightAge=500
PriorityWeightPartition=100
PriorityWeightFairshare=1000
PriorityDecayHalfLife=7-0    # jobs decay toward zero priority over 7 days
PriorityMaxAge=14-0           # maximum age bonus (2 weeks)
```

Key parameters:
- **Age weight** (`PriorityWeightAge`): rewards jobs that have been waiting longer.
  This prevents starvation of smaller jobs behind large ones.
- **Fairshare weight** (`PriorityWeightFairshare`): penalises users or groups that
  have used more than their fair share of resources recently.

---

## Partition configuration

Magic Castle creates a single default partition. For bioinformatics workloads, splitting
into multiple partitions by resource type avoids misuse:

```ini
PartitionName=cpubase    Nodes=node[1-4]  Default=YES  MaxTime=7-00:00:00  State=UP
PartitionName=highmem    Nodes=node[5-6]  Default=NO   MaxTime=3-00:00:00  State=UP
PartitionName=debug      Nodes=node1      Default=NO   MaxTime=00:30:00    State=UP
```

- `cpubase`: standard compute nodes, default partition for most jobs
- `highmem`: high-memory nodes (r2-series flavors) for variant calling or assembly
- `debug`: single node, short time limit — for testing scripts quickly

### In Puppet

SLURM partition configuration is managed via the Magic Castle Puppet module. Override
in your fork's Hiera data:

```yaml
# data/common.yaml
slurm::partitions:
  cpubase:
    nodes: "node[1-4]"
    default: true
    max_time: "7-00:00:00"
  highmem:
    nodes: "node[5-6]"
    default: false
    max_time: "3-00:00:00"
```

---

## Resource limits

Prevent single users from monopolising the cluster:

```ini
# Global limits
MaxJobsPerUser=50
MaxSubmitJobsPerUser=200

# Per-partition limits (in partition definition)
PartitionName=cpubase  MaxCPUsPerUser=32  MaxMemPerUser=128G
```

---

## NTP configuration

SLURM requires time synchronisation between nodes. The default Magic Castle config
uses standard public NTP servers. If your organisation requires specific NTP servers
(e.g. for compliance or low latency), configure them in Puppet:

```puppet
# /etc/chrony.conf or via Puppet
$ntp_servers = ['0.fr.pool.ntp.org', '1.fr.pool.ntp.org']
```

---

## Monitoring and debugging

```bash
# Node state
sinfo -o "%N %T %C"    # node, state, CPUs (allocated/idle/other/total)

# Queue state
squeue -o "%.10i %.9P %.30j %.8u %.8T %.10M %.6D %R"

# Job accounting
sacct -u <username> --format=JobID,JobName,Elapsed,State,MaxRSS

# SLURM log
journalctl -u slurmctld -n 100 -f   # on mgmt node
journalctl -u slurmd -n 100 -f      # on compute nodes
```
