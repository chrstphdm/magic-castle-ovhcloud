# Network Setup

**This step is mandatory and must be done before running `terraform apply`.**

Magic Castle does not create the OVHcloud private network for you — it expects one to
already exist. Skipping this step causes a cryptic Terraform error about missing network
resources.

---

## What you need to create

Three interconnected resources in your OVHcloud project:

1. **Private network** — the virtual LAN that all cluster nodes connect to
2. **Subnet** — the IP address range within the network (e.g. `10.10.0.0/24`)
3. **Router** — connects the private network to the public internet (required for nodes
   to download packages during Puppet configuration)

---

## Option A: Create via OVHcloud Control Panel (recommended for beginners)

1. Control Panel → Public Cloud → **Network** → **Private Networks** → **Create a private network**
2. Name it (e.g. `magic-castle-net`), select your region, choose a VLAN ID or leave default
3. Configure the subnet: e.g. `10.10.0.0/24`, gateway `10.10.0.1`, enable DHCP
4. Go to **Routers** → **Create a router** → attach it to the private network
5. Add a gateway interface on the router pointing to the external (public) network

Note the network **name** or **ID** — you will reference it in `main.tf`.

---

## Option B: Create via Terraform (infrastructure-as-code approach)

Create a separate `network.tf` file alongside your `main.tf`:

```hcl
terraform {
  required_providers {
    openstack = {
      source  = "terraform-provider-openstack/openstack"
      version = "~> 1.54"
    }
  }
}

resource "openstack_networking_network_v2" "cluster_net" {
  name           = "magic-castle-net"
  admin_state_up = true
}

resource "openstack_networking_subnet_v2" "cluster_subnet" {
  name            = "magic-castle-subnet"
  network_id      = openstack_networking_network_v2.cluster_net.id
  cidr            = "10.10.0.0/24"
  ip_version      = 4
  dns_nameservers = ["213.186.33.99", "8.8.8.8"]  # OVHcloud DNS + Google fallback
}

resource "openstack_networking_router_v2" "cluster_router" {
  name                = "magic-castle-router"
  external_network_id = data.openstack_networking_network_v2.ext_net.id
}

resource "openstack_networking_router_interface_v2" "cluster_router_iface" {
  router_id = openstack_networking_router_v2.cluster_router.id
  subnet_id = openstack_networking_subnet_v2.cluster_subnet.id
}

# Retrieve the external network ID (the public/internet network in your region)
data "openstack_networking_network_v2" "ext_net" {
  name = "Ext-Net"   # This is the standard name on OVHcloud
}
```

```bash
terraform init
terraform apply -target=openstack_networking_network_v2.cluster_net \
                -target=openstack_networking_subnet_v2.cluster_subnet \
                -target=openstack_networking_router_v2.cluster_router \
                -target=openstack_networking_router_interface_v2.cluster_router_iface
```

> **Why a separate apply?** Creating network infrastructure before the cluster resources
> avoids dependency ordering issues and makes it easier to destroy/recreate the cluster
> without touching the network (useful during development).

---

## OVHcloud-specific: the external network name

On OVHcloud, the external network (the one connected to the internet) is always named
`Ext-Net`. This is different from AWS (`igw-*`) or GCP (`default`) naming conventions.

If the `data` source lookup fails, list available networks:

```bash
source openstack-rc-<region>.rc
openstack network list
# Look for the one with "External" in its type
```

---

## Verify the network is ready

```bash
openstack network list
# Should show your private network

openstack subnet list
# Should show your subnet with the CIDR you defined

openstack router list
# Should show your router with an external gateway set
```

All three must exist before proceeding to `main.tf` configuration.

---

## Reusing an existing network across deployments

If you plan to destroy and recreate the cluster (common during setup and testing),
keep the network infrastructure in a **separate Terraform state** or create it manually.

This way, you can run `terraform destroy` on the cluster without destroying the network,
and redeploy faster without waiting for the network to be recreated.
