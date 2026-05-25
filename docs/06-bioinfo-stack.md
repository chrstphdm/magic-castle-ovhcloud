# Bioinformatics Software Stack

The default Magic Castle Puppet configuration does not include Nextflow, Apptainer,
or S3FS. This section explains how to add them via a custom Puppet fork.

---

## Why a Puppet fork

Magic Castle configures software using [Puppet](https://puppet.com/) and a dedicated
configuration repository (`puppet-magic_castle`). The official repo covers the base
cluster (SLURM, LDAP, modules, EESSI) but intentionally stays generic.

To add bioinformatics tools, you maintain a fork that extends the official config.

The fork approach:
1. Fork `https://github.com/ComputeCanada/puppet-magic_castle` on GitHub
2. Add your customisations as a new branch (e.g. `feature/bioinfo-stack`)
3. Point your `main.tf` to your fork:

```hcl
module "magic_castle" {
  ...
  config_git_url = "https://github.com/<your-username>/puppet-magic_castle.git"
  config_version = "feature/bioinfo-stack"
}
```

> **Important**: the fork must remain compatible with the base Magic Castle version you
> are using. Check the base repo's release notes when upgrading.

---

## Adding Nextflow

### What to configure

Nextflow requires:
- Java (OpenJDK 17 or 21)
- Nextflow binary itself
- Optional: `NXF_HOME` pointing to a shared directory (avoids re-downloading plugins per user)

### In your Puppet manifest

In your fork, add to the relevant profile (e.g. `profile::base` or a new `profile::nextflow`):

```puppet
# Install Java
package { 'java-21-openjdk-headless':
  ensure => installed,
}

# Download and install Nextflow
exec { 'install_nextflow':
  command => '/bin/bash -c "curl -fsSL https://get.nextflow.io | bash && mv nextflow /usr/local/bin/ && chmod +x /usr/local/bin/nextflow"',
  creates => '/usr/local/bin/nextflow',
  require => Package['java-21-openjdk-headless'],
}
```

Pin the Nextflow version for reproducibility:

```puppet
exec { 'install_nextflow':
  command => '/bin/bash -c "curl -fsSL https://github.com/nextflow-io/nextflow/releases/download/v<VERSION>/nextflow > /usr/local/bin/nextflow && chmod +x /usr/local/bin/nextflow"',
  creates => '/usr/local/bin/nextflow',
}
```

Replace `<VERSION>` with the desired version (e.g. `24.10.0`).

---

## Adding Apptainer (Singularity)

### Why non-SUID

By default, Apptainer's `starter-suid` binary is installed with SUID root, which allows
unprivileged users to create fakeroot environments. On many HPC clusters, system
administrators disable SUID executables for security reasons.

For tighter security or if your OVHcloud instances run with SELinux enforcing:

```puppet
# Install Apptainer without SUID
package { 'apptainer':
  ensure => '1.4.5',   # pin version
}

exec { 'disable_apptainer_suid':
  command => '/bin/chmod u-s /usr/libexec/apptainer/bin/starter-suid',
  onlyif  => '/bin/test -u /usr/libexec/apptainer/bin/starter-suid',
  require => Package['apptainer'],
}
```

### Container cache on shared storage

Configure Apptainer to use a shared cache directory to avoid each user downloading
the same container images independently:

```bash
# In /etc/environment or via Puppet:
APPTAINER_CACHEDIR=/project/containers/cache
APPTAINER_TMPDIR=/scratch/apptainer_tmp
```

---

## S3FS: mounting OVHcloud Object Storage

S3FS mounts an S3-compatible bucket as a POSIX filesystem. Use it to mount reference
databases or large input datasets stored in OVHcloud Object Storage.

### Install S3FS via Puppet

```puppet
package { 's3fs-fuse':
  ensure => installed,
}
```

### S3FS credentials {#s3fs-credentials}

S3FS reads credentials from `~/.passwd-s3fs` or `/etc/passwd-s3fs`. Format:

```
<access-key>:<secret-key>
```

```bash
echo "<access-key>:<secret-key>" > /etc/passwd-s3fs
chmod 600 /etc/passwd-s3fs
```

**Never hardcode credentials in your Puppet repository if it is public.**

Options for secure credential delivery:
- Pass as Terraform variables and write via a `file` Terraform resource
- Use Puppet Hiera with eyaml encryption (PKCS7 keys)
- Use OVHcloud's native secrets management if available in your region

### Mount a bucket via /etc/fstab

```
s3fs#<bucket-name> /mnt/references fuse _netdev,allow_other,use_path_request_style,url=https://s3.gra.perf.cloud.ovh.net,passwd_file=/etc/passwd-s3fs 0 0
```

Replace `<bucket-name>` with your OVHcloud Object Storage container name and adjust
the region in the URL.

### Mount via Puppet

```puppet
mount { '/mnt/references':
  ensure  => mounted,
  device  => 's3fs#<bucket-name>',
  fstype  => 'fuse',
  options => '_netdev,allow_other,use_path_request_style,url=https://s3.gra.perf.cloud.ovh.net,passwd_file=/etc/passwd-s3fs',
  require => [Package['s3fs-fuse'], File['/etc/passwd-s3fs']],
}
```

### Mounting multiple buckets

For multiple buckets (input data, output, reference databases):

```puppet
$s3_mounts = {
  '/mnt/references' => '<bucket-references>',
  '/mnt/inputs'     => '<bucket-inputs>',
  '/mnt/outputs'    => '<bucket-outputs>',
}

$s3_mounts.each |$mountpoint, $bucket| {
  file { $mountpoint:
    ensure => directory,
  }
  mount { $mountpoint:
    ensure  => mounted,
    device  => "s3fs#${bucket}",
    fstype  => 'fuse',
    options => '_netdev,allow_other,...',
    require => [Package['s3fs-fuse'], File[$mountpoint]],
  }
}
```

---

## Verifying the stack

After Puppet has converged (see [Deploy and destroy](09-deployment.md)):

```bash
# SSH to login node
ssh rocky@<login-ip> -i ~/.ssh/mc_ovh

# Check Nextflow
nextflow -version

# Check Apptainer
apptainer version

# Check S3 mount
ls /mnt/references/
df -h /mnt/references
```
