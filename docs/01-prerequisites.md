# Prerequisites

---

## OVHcloud account and project

You need an OVHcloud **Public Cloud** project — not a VPS or dedicated server.
Public Cloud is OVHcloud's OpenStack-based infrastructure.

1. Go to the [OVHcloud Control Panel](https://www.ovh.com/manager/#/public-cloud/)
2. Create a new Public Cloud project (or use an existing one)
3. Note your **Project ID** (visible in the URL and project overview)

---

## Quotas

OVHcloud projects start with default quotas that are sufficient for a small test cluster.
For a production deployment you will likely need to increase them.

Default limits (vary by region — check your project's **Quota** page):

| Resource | Typical default |
|----------|----------------|
| vCPUs    | 20             |
| RAM      | 50 GB          |
| Volumes  | 10             |
| Volume storage | 200 GB   |
| Floating IPs | 1–3        |
| Security groups | 10       |

A minimal Magic Castle cluster (1 login + 1 mgmt + 2 compute nodes) uses roughly:
8–16 vCPUs, 16–32 GB RAM, 3 floating IPs, and 2–3 volumes.

To request a quota increase: Control Panel → Public Cloud project → **Quota and Regions** → **Increase my quota**.

---

## SSH key pair

Magic Castle configures SSH access on all nodes using a key you register in OVHcloud.

```bash
# Generate a key pair if you don't have one
ssh-keygen -t ed25519 -C "magic-castle" -f ~/.ssh/mc_ovh

# The public key to register is:
cat ~/.ssh/mc_ovh.pub
```

Register the public key:
Control Panel → Public Cloud → **SSH Keys** → **Add a key**

Note the key name — you will reference it in `main.tf`.

---

## Terraform

Magic Castle requires Terraform ≥ 1.5.7.

```bash
# Check current version
terraform version

# Install via package manager (Linux)
sudo apt install terraform         # Debian/Ubuntu (HashiCorp repo required)
brew install terraform             # macOS

# Or download directly from HashiCorp
# https://developer.hashicorp.com/terraform/install
```

> **Note for HPC users**: if you are running this from a login node or a shared machine,
> install Terraform in your `$HOME` — it is a single static binary.

---

## OpenStack RC File v3

Magic Castle uses the [OpenStack Terraform provider](https://registry.terraform.io/providers/terraform-provider-openstack/openstack/latest),
which authenticates via standard OpenStack environment variables.

**Download the RC file:**
Control Panel → Public Cloud project → **Users & Roles** → your user → **Download OpenStack RC File** → choose **Version 3** (v2 will not work).

The file looks like:

```bash
#!/usr/bin/env bash
export OS_AUTH_URL=https://auth.cloud.ovh.net/v3
export OS_TENANT_ID=<your-tenant-id>
export OS_TENANT_NAME="<your-project-name>"
export OS_USERNAME="<your-username>"
export OS_PASSWORD="<your-password>"
export OS_REGION_NAME="GRA11"
export OS_INTERFACE=public
export OS_IDENTITY_API_VERSION=3
```

**Source it before running Terraform:**

```bash
source ~/openstack-rc-gra11.rc
# Enter your password when prompted (unless you hardcoded it — avoid this)
```

> **Tip**: rename the file to include the region name (e.g. `openstack-rc-gra11.rc`)
> if you plan to deploy in multiple regions. Sourcing the wrong RC file is a common
> source of confusing errors.

---

## Clone Magic Castle

```bash
git clone https://github.com/ComputeCanada/magic_castle.git
cd magic_castle/ovh
ls
# main.tf.example  README.md  ...
```

Copy the example config:

```bash
cp main.tf.example main.tf
```

You will edit `main.tf` extensively — see [main.tf walkthrough](04-main-tf.md).

---

## References

- [Terraform installation](https://developer.hashicorp.com/terraform/install) — download and install Terraform for your platform
- [Magic Castle releases](https://github.com/ComputeCanada/magic_castle/releases) — pinned release tags for stable deployments
- [OVHcloud Public Cloud getting started](https://help.ovhcloud.com/csm/en-public-cloud-compute-getting-started?id=kb_article_view&sysparm_article=KB0051009) — creating your first Public Cloud project
