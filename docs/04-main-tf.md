# main.tf Walkthrough

This section explains each block of a minimal `main.tf` for an OVHcloud Magic Castle
deployment. All values use placeholders — replace them with your own.

---

## Provider configuration

```hcl
terraform {
  required_version = ">= 1.5.7"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.54"
    }
  }
}
```

Magic Castle on OVHcloud uses the **OpenStack** provider, not an OVHcloud-specific one.
OVHcloud Public Cloud runs on OpenStack, so all standard OpenStack resources apply.

No provider credentials go here — they are picked up from the environment variables
set by sourcing your OpenStack RC file.

---

## Magic Castle module

```hcl
module "magic_castle" {
  source = "git::https://github.com/ComputeCanada/magic_castle.git//ovh?ref=main"

  config_git_url = "https://github.com/ComputeCanada/puppet-magic_castle.git"
  config_version = "main"
```

- `source`: pulls the OVHcloud-specific Magic Castle Terraform module directly from GitHub.
  Pin to a tag (e.g. `?ref=13.3.0`) for reproducibility.
- `config_git_url`: the Puppet configuration repository. This is where the software stack
  is defined. To add Nextflow, Apptainer, and S3FS, you replace this with a fork.
  See [Bioinformatics stack](06-bioinfo-stack.md).
- `config_version`: the branch or tag to use in the Puppet repo.

---

## Cluster identity

```hcl
  cluster_name = "mycluster"
  domain       = "example.com"
```

- `cluster_name`: becomes the hostname prefix. Nodes will be named `mycluster-login1`,
  `mycluster-mgmt1`, etc.
- `domain`: used for internal DNS and SSL certificates. It does not need to be a
  registered domain — it is only used internally.

---

## Image

```hcl
  image = "Rocky Linux 9"
```

Magic Castle requires a **Red Hat-family** OS: Rocky Linux, AlmaLinux, or CentOS Stream.
**Ubuntu and Debian are not supported.**

On OVHcloud, check available images:

```bash
openstack image list --public | grep -i rocky
```

Use the exact name as it appears in the output.

> **Regional note**: image availability varies by region. If your preferred image is not
> available, check another region or upload your own via Control Panel → Public Cloud →
> **Storage** → **Object Storage** → **Import an image**.

---

## SSH key

```hcl
  nb_users     = 10
  public_keys  = ["ssh-ed25519 AAAA... your-key-comment"]
```

- `nb_users`: the number of local user accounts to create on the cluster.
- `public_keys`: list of SSH public keys that will be authorized for all user accounts.
  Add your own key here.

---

## Instances

```hcl
  instances = {
    login  = { type = "b2-7",   tags = ["login", "public", "proxy"], count = 1 }
    mgmt   = { type = "b2-7",   tags = ["puppet", "nfs", "slurm-controller"], count = 1 }
    node   = { type = "b2-15",  tags = ["node"], count = 2 }
  }
```

### OVHcloud instance types (flavors)

OVHcloud uses a different naming convention from AWS or GCP:

| Flavor | vCPUs | RAM   | Use case |
|--------|-------|-------|----------|
| `b2-7`  | 2    | 7 GB  | Login/mgmt — interactive work |
| `b2-15` | 4    | 15 GB | Light compute — small analyses |
| `b2-30` | 8    | 30 GB | Standard genomics workloads |
| `b2-60` | 16   | 60 GB | WGS, multi-sample analyses |
| `b2-120`| 32   | 120 GB| Heavy parallel work |
| `r2-30` | 2    | 30 GB | Memory-intensive (variant calling) |
| `r2-60` | 4    | 60 GB | High-memory analyses |

Check what is available in your region before writing `main.tf`:

```bash
openstack flavor list | grep -E "^| b2| r2| c2"
```

> **Friction point**: not all flavors are available in all regions. GRA (Gravelines) and
> SBG (Strasbourg) have the widest selection. If a flavor is unavailable, Terraform
> will fail with an obscure error about no valid hosts. Pick an alternative flavor and retry.

### Tags

Tags define what services run on each node:

| Tag | Service |
|-----|---------|
| `login` | SSH gateway |
| `public` | Gets a floating IP |
| `proxy` | JupyterHub reverse proxy |
| `puppet` | Puppet master |
| `nfs` | NFS server (exports /home, /project) |
| `slurm-controller` | SLURM `slurmctld` daemon |
| `node` | SLURM compute node (`slurmd`) |

---

## Volumes

```hcl
  volumes = {
    nfs = {
      home    = 50
      project = 100
      scratch = 50
    }
  }
```

Volumes are attached to the `nfs`-tagged node (mgmt1 by default) and exported to the
cluster via NFS.

Sizes are in GB. Adjust based on your expected data volume.

### Attaching a pre-existing volume by UUID

If you have a persistent volume (e.g. a reference database) that should survive cluster
destruction and redeployment:

```hcl
  volumes = {
    nfs = {
      home    = 50
      project = 100
    }
    node1 = {
      references = "<your-volume-uuid>"   # attach by existing UUID
    }
  }
```

This is covered in detail in [OVHcloud specifics](05-ovhcloud-specifics.md#persistent-volumes).

---

## Network

```hcl
  network = {
    type = "arbutus"   # or use an explicit network name
  }
```

Or reference the network you created in [Network setup](03-network-setup.md):

```hcl
  network = {
    name = "magic-castle-net"
  }
```

---

## Complete minimal example

```hcl
terraform {
  required_version = ">= 1.5.7"
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.54"
    }
  }
}

module "magic_castle" {
  source = "git::https://github.com/ComputeCanada/magic_castle.git//ovh?ref=main"

  config_git_url = "https://github.com/ComputeCanada/puppet-magic_castle.git"
  config_version = "main"

  cluster_name = "mycluster"
  domain       = "example.com"
  image        = "Rocky Linux 9"

  nb_users    = 5
  public_keys = ["ssh-ed25519 AAAA... your-key-comment"]

  instances = {
    login = { type = "b2-7",  tags = ["login", "public", "proxy"], count = 1 }
    mgmt  = { type = "b2-7",  tags = ["puppet", "nfs", "slurm-controller"], count = 1 }
    node  = { type = "b2-15", tags = ["node"], count = 2 }
  }

  volumes = {
    nfs = {
      home    = 50
      project = 100
    }
  }

  network = {
    name = "magic-castle-net"
  }
}

output "login_ip" {
  value = module.magic_castle.public_ip
}
```
