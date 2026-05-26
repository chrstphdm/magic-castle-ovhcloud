# Deploy and Destroy

A safe workflow for deploying, monitoring, and destroying your Magic Castle cluster.

---

## First deployment

```bash
# 1. Source OpenStack credentials
source openstack-rc-gra11.rc

# 2. Initialise Terraform (downloads providers and Magic Castle module)
terraform init

# 3. Preview what will be created
terraform plan

# 4. Apply — this creates all cloud resources
terraform apply
# Type 'yes' when prompted
```

`terraform apply` completes in 5–10 minutes and prints the login node's public IP.
The cluster is **not yet ready** — Puppet still needs to run on each node.

---

## Waiting for Puppet to finish

After `terraform apply`, SSH to the login node and wait for Puppet to converge:

```bash
LOGIN_IP=$(terraform output -raw login_ip)

# Wait for SSH to become available (may take 2–5 minutes after apply)
until ssh -o StrictHostKeyChecking=no -o ConnectTimeout=5 rocky@$LOGIN_IP -i ~/.ssh/mc_ovh "echo ok" 2>/dev/null; do
    echo "Waiting for SSH..."
    sleep 30
done

echo "SSH ready. Connecting..."
```

Once connected, monitor Puppet:

```bash
# Watch the Puppet log in real time
journalctl -u puppet -f

# Or check when the last Puppet run completed
puppet agent --last_run_summary
```

Puppet is done when all nodes appear in `sinfo`:

```bash
sinfo
# PARTITION  AVAIL  TIMELIMIT  NODES  STATE  NODELIST
# cpubase*   up     infinite       2  idle   node[1-2]
```

---

## Deployment checklist

After Puppet converges, verify:

```bash
# SLURM is functional
sinfo
squeue

# Shared filesystems are mounted on all nodes
ssh node1 df -h | grep -E "home|project|scratch"

# Software stack is installed (if using bioinformatics Puppet fork)
which nextflow && nextflow -version
apptainer version

# S3 mounts are accessible (if configured)
ls /mnt/references/
```

Submit a test job to confirm end-to-end function:

```bash
sbatch --wrap "hostname && sleep 10 && echo done" --output /home/rocky/test-%j.out
watch squeue
cat /home/rocky/test-<jobid>.out
```

---

## Deploying in multiple regions

Use Terraform workspaces to manage separate clusters per region:

```bash
# List workspaces
terraform workspace list

# Create and switch to a region workspace
terraform workspace new gra11
terraform workspace select gra11
source openstack-rc-gra11.rc
terraform apply -var-file=regions/gra11.tfvars

# Always verify active workspace before any destructive operation
terraform workspace show
```

---

## Scaling compute nodes

To add or remove compute nodes without destroying the cluster, update the `count` in
`main.tf` and apply:

```hcl
instances = {
  node = { type = "b2-30", tags = ["node"], count = 5 }  # was 2
}
```

```bash
terraform apply
```

New nodes will bootstrap via Puppet automatically. Existing nodes are not affected.

> **Before scaling down**: drain the nodes gracefully so running jobs are not killed.
> ```bash
> scontrol update nodename=node[4-5] state=drain reason="removing"
> # Wait for running jobs to finish
> sinfo  # nodes should show 'drained' not 'allocated'
> ```
> Then apply the reduced count.

---

## Suspend and resume the cluster

If you need to pause work (weekend, end of project sprint) but want to keep the cluster
state, stop the compute instances:

```bash
# Suspend compute nodes (stop billing for compute, keep volumes and login)
openstack server stop node1 node2 node3 node4

# Resume
openstack server start node1 node2 node3 node4
```

Note: you continue paying for attached volumes and the floating IP while suspended.

---

## Terraform state management

Terraform tracks every cloud resource it creates in a **state file** (`terraform.tfstate`).
If you lose this file, Terraform no longer knows what it deployed — you end up with
orphaned resources (VMs, volumes, floating IPs) still running and billing, with no way
to destroy them cleanly.

### Back up the state

At a minimum, copy your state file after every `apply` or `destroy`:

```bash
cp terraform.tfstate terraform.tfstate.backup.$(date +%Y%m%d-%H%M%S)
```

### Use a remote backend (recommended for teams)

For shared clusters or any deployment you cannot afford to lose, store the state in a
remote backend with locking:

```hcl
terraform {
  backend "s3" {
    bucket   = "my-tfstate-bucket"
    key      = "magic-castle/terraform.tfstate"
    region   = "gra"
    endpoint = "https://s3.gra.perf.cloud.ovh.net"

    skip_credentials_validation = true
    skip_region_validation      = true
    skip_metadata_api_check     = true
  }
}
```

This uses OVHcloud Object Storage as an S3-compatible backend. Benefits:

- **No local file to lose** — state is stored remotely and survives laptop failures
- **Locking** — prevents two people from running `terraform apply` simultaneously
- **Versioning** — if you enable bucket versioning, you can roll back to a previous state

> **If you already lost your state**: the resources still exist in OVHcloud. List them
> with `openstack server list`, `openstack volume list`, `openstack floating ip list`,
> and delete them manually via the CLI or the OVHcloud Control Panel.

---

## Destroying the cluster

```bash
# Always verify which workspace you are in first
terraform workspace show

terraform destroy
# Type 'yes' when prompted
```

This destroys all cluster resources **including volumes created by Magic Castle**.
Volumes you attached by UUID (pre-existing volumes) are not destroyed — only the
attachment is removed.

> **Data protection**: before destroying, make sure any important data is:
> - Pushed to OVHcloud Object Storage
> - Stored on a pre-existing volume (attached by UUID, not created by Magic Castle)
> - Downloaded locally

If destruction hangs (common with security groups or floating IPs that have dependencies):

```bash
# Force-remove the stuck resource
terraform state list                          # find the resource name
terraform destroy -target=<resource_address>  # destroy just that resource
terraform destroy                             # retry the full destroy
```

---

## References

- [Terraform CLI commands](https://developer.hashicorp.com/terraform/cli/commands) — reference for init, plan, apply, destroy, and state management
- [Terraform S3 backend](https://developer.hashicorp.com/terraform/language/backend/s3) — configuring remote state storage with an S3-compatible backend
