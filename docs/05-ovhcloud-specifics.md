# OVHcloud Specifics

These are the things you will not find in the official Magic Castle or OVHcloud docs,
but that will cost you hours if you don't know them.

---

## Authentication: always use RC File v3

The Magic Castle OVHcloud docs mention the RC file but do not warn you that v2 is
incompatible. If you download the wrong version:

```
Error: Error creating OpenStack client: Post "https://auth.cloud.ovh.net/v2.0/tokens": 404
```

**Fix**: re-download from Control Panel → Users & Roles → your user → **Download OpenStack RC File v3**.

Also: if you have multiple OVHcloud projects, it is easy to source the wrong RC file.
Name your RC files explicitly: `openstack-rc-project-gra11.rc`, `openstack-rc-project-sbg5.rc`.

---

## Flavor availability varies by region

Not all instance types are available in every region. If you request `b2-120` in `SBG5`
and it is only available in `GRA11`, Terraform fails with:

```
Error: Error creating openstack_compute_instance_v2: ... No valid host was found
```

This error gives no indication that the problem is flavor availability.

**Before writing main.tf**, verify your flavors are available in your target region:

```bash
source openstack-rc-<region>.rc
openstack flavor list | grep b2
```

If your preferred flavor is unavailable, check an adjacent region (GRA11 ↔ SBG5).
The two largest French regions usually have the widest selection.

---

## Persistent volumes

By default, all OVHcloud Block Storage volumes attached by Magic Castle are created and
destroyed with the cluster. This is fine for scratch and home directories, but you do not
want your 500 GB reference genome database deleted every time you rebuild the cluster.

### Attach an existing volume by UUID

1. Create the volume manually in the OVHcloud Control Panel (or via Terraform separately)
2. Format and populate it once
3. Note its UUID
4. Reference it in `main.tf` to attach without recreating:

```hcl
  volumes = {
    nfs = {
      home    = 50
      project = 100
    }
    # Attach an existing volume — will NOT be destroyed with the cluster
    persistent_refs = "<your-volume-uuid>"
  }
```

> **Important**: the volume must be in the same region and availability zone as your
> cluster. You cannot attach a GRA11 volume to a SBG5 cluster.

### Volume types

OVHcloud offers two types of Block Storage:

| Type | Performance | Price | Use case |
|------|-------------|-------|----------|
| Classic | ~400 IOPS | Lower | Home dirs, project data, logs |
| High Speed gen2 | ~3000 IOPS (NVMe) | Higher | Reference databases, scratch, intensive I/O |

For most genomics pipelines, classic volumes for `/home` and `/project` and a
high-speed volume for reference databases is the right balance.

---

## Multi-region deployments

If you work across multiple OVHcloud regions (e.g. one cluster in GRA11 for EU data,
one in SBG5 for another project), use Terraform workspaces:

```bash
# Create a workspace per region
terraform workspace new gra11
terraform workspace new sbg5

# Switch and deploy
terraform workspace select gra11
source openstack-rc-gra11.rc
terraform apply -var-file=regions/gra11.tfvars

terraform workspace select sbg5
source openstack-rc-sbg5.rc
terraform apply -var-file=regions/sbg5.tfvars
```

Keep a `regions/` directory with per-region variable files:

```hcl
# regions/gra11.tfvars
cluster_name = "mycluster-gra"
image        = "Rocky Linux 9"   # verify exact name per region
node_flavor  = "b2-30"
```

> **Pitfall**: Terraform state is workspace-scoped. Never run `terraform destroy` without
> first verifying which workspace is active (`terraform workspace show`).

---

## OVHcloud Object Storage credentials

OVHcloud Object Storage is S3-compatible but uses its own credentials — separate from
your OpenStack login. You need **EC2 credentials** (access key + secret key).

Generate them:

```bash
openstack ec2 credentials create
# Output: access, secret keys
```

Or via Control Panel: Users & Roles → your user → **EC2 Credentials** tab.

The S3 endpoint for OVHcloud follows this pattern:
```
https://s3.<region>.perf.cloud.ovh.net
```
e.g. `https://s3.gra.perf.cloud.ovh.net` for the GRA region.

**Never hardcode these credentials in your Terraform files or Puppet configuration.**
Use environment variables, an encrypted YAML file, or a secrets manager.
See [Bioinformatics stack](06-bioinfo-stack.md#s3fs-credentials) for how to pass
them securely to the cluster.

---

## Floating IP and security groups

Magic Castle automatically creates a floating IP for the login node. However, if your
project quota only allows 1 floating IP and you already have one allocated elsewhere,
`terraform apply` will fail silently (the IP just won't be allocated, and the login node
will have no public access).

**Check before applying:**

```bash
openstack floating ip list
openstack quota show | grep floatingip
```

Release unused floating IPs in the Control Panel if you are at the limit.

---

## Puppet configuration delay

After `terraform apply` completes, the cluster is not immediately ready. Each node
bootstraps via Puppet, which takes **15–30 minutes** depending on the number of nodes
and the software stack.

You can monitor progress:

```bash
ssh rocky@<login-ip> -i ~/.ssh/mc_ovh
# Once connected:
journalctl -u puppet -f          # watch Puppet runs
systemctl status puppet           # check service state
tail -f /var/log/puppetlabs/puppet/puppet.log
```

The cluster is ready when `sinfo` returns your nodes in `idle` state:

```bash
sinfo
# PARTITION  AVAIL  TIMELIMIT  NODES  STATE  NODELIST
# cpubase*   up     infinite       2  idle   node[1-2]
```

See [Deploy and destroy](09-deployment.md) for a script that waits for Puppet to finish.
