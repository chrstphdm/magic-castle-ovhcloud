# Architecture

---

## What Magic Castle deploys

A Magic Castle cluster has four types of nodes. Each runs a specific role and is sized
differently.

```
Internet
    │
    ▼
[login node]  ← SSH entry point, job submission, file transfers
    │
    ├── [mgmt node]   ← Puppet server, LDAP, SLURM controller, DNS
    │
    └── [compute nodes]  ← Where jobs actually run (1 to N instances)

Shared storage: NFS exported from mgmt node (or external volume)
```

---

## Node roles

### Login node (`login1`)

- The only node with a public IP (floating IP)
- Users SSH here to submit jobs, transfer data, edit scripts
- Runs the SLURM client but not the scheduler
- Should be sized for interactive work: 2–4 vCPUs, 8–16 GB RAM
- **Not** where compute happens — jobs run on compute nodes

### Management node (`mgmt1`)

- No public IP — only reachable from within the cluster
- Runs: Puppet master, OpenLDAP, SLURM controller (`slurmctld`), NFS server
- Critical: if this node is down, job submission stops
- Sized conservatively: 2–4 vCPUs, 4–8 GB RAM (it manages, doesn't compute)
- Exports `/home` and `/project` via NFS to all other nodes

### Compute nodes (`node[1-N]`)

- No public IP
- Run SLURM daemon (`slurmd`)
- Mount shared filesystems from mgmt node
- Sized for your workload: genomics pipelines typically need 8–32 vCPUs and 32–128 GB RAM
- You define how many and which flavors in `main.tf`

### Node naming

Magic Castle uses Ansible/Puppet naming conventions. The default names are `login1`,
`mgmt1`, `node1`, `node2`, etc. You can customise them in `main.tf`.

---

## Storage options

Magic Castle provides two storage approaches on OVHcloud:

### NFS from mgmt node (default)

The mgmt node exports `/home` and `/project` (and `/scratch` if configured) via NFS.
All nodes mount these at boot.

- Pros: simple, no extra cost, works out of the box
- Cons: single point of failure, bandwidth limited by mgmt node's disk I/O
- Good for: small clusters, development, ephemeral workloads

### Attached volumes (OVHcloud Block Storage)

Additional persistent volumes can be attached to the mgmt node and exported via NFS,
or attached directly to individual nodes.

- **Classic volumes**: standard SATA-backed, cheaper, sufficient for most genomics data
- **High-speed-gen2 volumes**: NVMe-backed, ~3× faster I/O, use for databases or scratch

See [OVHcloud specifics](05-ovhcloud-specifics.md#volume-types) for how to choose and
[main.tf walkthrough](04-main-tf.md#volumes) for configuration.

### OVHcloud Object Storage (S3-compatible)

OVHcloud Object Storage is S3-compatible and can be mounted on the cluster using `s3fs`.
This is the recommended approach for large reference databases and output data that
outlive the cluster.

See [Bioinformatics stack](06-bioinfo-stack.md#s3fs) for mounting configuration.

---

## Network topology

All cluster nodes communicate on a private network. Only the login node has a floating IP.

```
OVHcloud Private Network (10.x.x.0/24)
├── login1    10.x.x.10   + floating IP (public)
├── mgmt1     10.x.x.11   (private only)
├── node1     10.x.x.20   (private only)
└── node2     10.x.x.21   (private only)
```

You must create this private network **before** running Magic Castle.
See [Network setup](03-network-setup.md).

---

## Software stack

Magic Castle configures the cluster using Puppet. On first boot, all nodes pull their
configuration from the Puppet master on `mgmt1`. This takes 10–20 minutes.

Default software stack includes:
- SLURM workload manager
- OpenLDAP for user management
- FreeIPA (optional) for Kerberos/GSSAPI
- JupyterHub (optional)
- Lmod environment modules
- EESSI software stack (3000+ scientific applications)

For bioinformatics workloads, the default stack lacks Nextflow, Apptainer, and S3FS.
See [Bioinformatics stack](06-bioinfo-stack.md) for how to add them.
