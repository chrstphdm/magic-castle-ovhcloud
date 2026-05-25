# Troubleshooting

Common errors on OVHcloud Magic Castle deployments, with their causes and fixes.

---

## Terraform errors

### `Error: Post "https://auth.cloud.ovh.net/v2.0/tokens": 404`

**Cause**: you are using an OpenStack RC File v2 instead of v3.

**Fix**: re-download the RC file from Control Panel → Users & Roles → **Download OpenStack RC File v3**.

---

### `Error: No valid host was found`

**Cause**: the instance flavor you requested is not available in your region.

**Fix**:
```bash
openstack flavor list | grep b2
# Check which flavors exist in your current region
```
Adjust `type` in `instances` block of `main.tf` to a flavor that is actually available.

---

### `Error: Error creating openstack_networking_floatingip_v2`

**Cause**: your project has reached the floating IP quota (typically 1–3 by default).

**Fix**:
```bash
openstack floating ip list
# Release any unused floating IPs
openstack floating ip delete <ip-id>
```
Or request a quota increase in the Control Panel.

---

### `Error: Error creating openstack_networking_network_v2: ... already exists`

**Cause**: a network with the same name already exists in your project.

**Fix**: use a unique network name in `main.tf`, or import the existing network into
Terraform state:
```bash
terraform import openstack_networking_network_v2.cluster_net <network-id>
```

---

### `terraform destroy` hangs on security groups

**Cause**: OVHcloud security groups cannot be deleted while instances that use them
still exist (or are in a partially-deleted state).

**Fix**:
```bash
# List stuck resources
terraform state list

# Force-destroy instances first
terraform destroy -target=module.magic_castle.openstack_compute_instance_v2.instances

# Then retry the full destroy
terraform destroy
```

---

## Cluster connectivity

### SSH connection refused after `terraform apply`

**Cause**: the login node is still booting. It takes 2–5 minutes after `apply` completes.

**Fix**: wait and retry. If it persists after 10 minutes:
```bash
openstack server list
openstack server show <login-node-id>   # check ACTIVE status and console log
openstack console log show <login-node-id>
```

---

### Node shows as `down` or `drained` in `sinfo`

**Cause**: SLURM daemon (`slurmd`) failed to start, or Puppet hasn't finished yet.

**Fix**:
```bash
# On the affected node (via login node)
ssh node1
systemctl status slurmd
journalctl -u slurmd -n 50

# If slurmd is stopped, start it
systemctl start slurmd

# If the node is drained manually, undrain it
# (from login node or mgmt node)
scontrol update nodename=node1 state=resume
```

---

### Puppet never finishes

**Cause**: Puppet master on mgmt1 is unreachable (DNS issue, mgmt node not up yet),
or the Puppet repository URL is wrong.

**Fix**:
```bash
# On login node, check connectivity to mgmt1
ping mgmt1
ssh mgmt1

# Check Puppet master service
ssh mgmt1 systemctl status puppetserver

# Check the Puppet repo URL matches config_git_url in main.tf
ssh mgmt1 grep git /etc/puppetlabs/puppet/hiera.yaml
```

---

## Storage issues

### S3FS mount fails silently

**Cause**: wrong credentials, wrong endpoint URL, or wrong bucket name.

**Debug**:
```bash
# Unmount and remount with debug output
fusermount -u /mnt/references
s3fs <bucket-name> /mnt/references \
  -o passwd_file=/etc/passwd-s3fs \
  -o use_path_request_style \
  -o url=https://s3.gra.perf.cloud.ovh.net \
  -o dbglevel=info \
  -f    # run in foreground to see errors
```

Common mistakes:
- Using virtual-hosted-style URL (`bucket.s3.region...`) instead of path-style
- Wrong region in the endpoint URL
- Credentials generated for a different OVHcloud project or user

---

### `/home` not mounted on compute nodes

**Cause**: NFS server (mgmt1) is not running, or the compute node failed to mount at boot.

**Fix**:
```bash
# On mgmt1: verify NFS is exporting
showmount -e mgmt1

# On compute node: force remount
mount -a
# or
mount mgmt1:/home /home
```

---

## SLURM issues

### Jobs stay `PENDING` indefinitely

Common reasons:

```bash
# Check the reason column
squeue -o "%.10i %.8T %R"

# Reasons and fixes:
# Resources       → not enough CPUs/memory available; check sinfo
# ReqNodeNotAvail → requested node is down; check sinfo
# Priority        → waiting for higher-priority jobs; normal, wait
# QOSMaxCpuPerUserLimit → per-user CPU limit reached; reduce concurrency
```

---

### Nextflow pipeline stalls — jobs never submitted

**Cause**: Nextflow launched before SLURM was ready (Puppet still running).

**Fix**:
```bash
# Verify SLURM is accepting jobs
sinfo
srun --pty hostname   # interactive test job

# If sinfo shows no nodes, Puppet hasn't finished yet
# Wait and recheck
```

---

## Cost surprises

### Unexpected billing after cluster destroy

Floating IPs and volumes may survive `terraform destroy` if Terraform failed to delete them.

**Check for orphaned resources**:
```bash
openstack floating ip list
openstack volume list
openstack server list
```

Delete any resources that should have been removed. OVHcloud bills for floating IPs
and volumes even when not attached to any instance.
