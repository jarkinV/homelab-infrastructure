# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Proxmox automation repository using Ansible to manage VMs, LXC containers, ZFS storage, and Docker applications. The infrastructure is designed for a homelab environment with automated provisioning and configuration.

## Running Playbooks

All playbooks are run from the repository root using `ansible-playbook`:

```bash
# VM operations
ansible-playbook playbooks/vm/create_ubuntu_cloud_template.yaml
ansible-playbook playbooks/vm/clone_and_start_vm.yaml -e "new_vm_id=101 new_vm_name=ubuntu-vm-101"
ansible-playbook playbooks/vm/setup_docker.yaml
ansible-playbook playbooks/vm/attach_zfs_datasets.yaml -e "vm_id=100"

# LXC container operations
ansible-playbook playbooks/lxc/create_container.yaml
ansible-playbook playbooks/lxc/install_docker.yaml
ansible-playbook playbooks/lxc/update_and_upgrade.yaml

# ZFS storage operations
ansible-playbook playbooks/zfs/create_zfs_pool.yaml
ansible-playbook playbooks/zfs/configure_zfs_storage.yaml
ansible-playbook playbooks/zfs/add_zfs_storage.yaml

# Application deployments
ansible-playbook playbooks/apps/deploy_traefik.yaml
ansible-playbook playbooks/apps/deploy_actual.yaml
```

## Architecture

### Inventory Management

- **Static hosts**: Defined in `inventory.ini` for Proxmox host and LXC containers
- **Dynamic VM inventory**: The `clone_and_start_vm.yaml` playbook automatically updates both `vars/vm_inventory.yml` and `inventory.ini` when new VMs are created
- **Host groups**:
  - `proxmox_hosts`: Proxmox hypervisor (root access)
  - `lxc_containers`: LXC containers (typically for Docker workloads)
  - `ubuntu_vms`: Ubuntu cloud-init VMs (auto-managed section in inventory.ini)

### Variable Management

The repository uses a two-tier variable system:

- **`vars/global.yml`**: Non-sensitive shared configuration (template IDs, storage names, network bridges, default VM resources)
- **`vars/secrets.yml`**: Sensitive data (SSH keys, passwords, API tokens) - gitignored

Most playbooks load both files via `vars_files`. Secrets are referenced using `{{ variable_name }}` from secrets.yml.

### VM Provisioning Flow

1. **Template creation** (`create_ubuntu_cloud_template.yaml`): Downloads Ubuntu cloud image, creates a cloud-init enabled template VM (ID 8000 by default)
2. **VM cloning** (`clone_and_start_vm.yaml`): Clones from template, resizes disk, configures resources, starts VM, waits for IP via guest agent, automatically updates inventory files
3. **Post-provisioning**: Additional playbooks like `setup_docker.yaml` can be run against the new VM

The clone playbook handles inventory updates automatically, adding new VMs to both the YAML tracking file and the Ansible inventory.

### LXC Container Workflow

LXC containers are used for privileged workloads like Docker. The `create_container.yaml` playbook creates Debian-based containers with:
- Nesting enabled (required for Docker)
- Privileged mode (root in container = root on host)
- Static IP configuration
- SSH key injection

### ZFS Storage Architecture

The ZFS setup is designed for Docker data persistence:

1. **Pool creation** (`create_zfs_pool.yaml`): Creates RAIDZ1 pool from disk serials with compression and optimized settings
2. **Dataset configuration** (`configure_zfs_storage.yaml`): Creates individual ZFS datasets for each Docker app (traefik, actual, paperless, etc.)
3. **VM attachment** (`attach_zfs_datasets.yaml`): Uses 9p filesystem (virtio-9p-pci) to share ZFS datasets with VMs without NFS overhead

Datasets are mounted inside VMs via 9p, not as block devices. This allows the Proxmox host to maintain direct ZFS control while VMs access the data.

### Application Deployment Pattern

Application playbooks (in `playbooks/apps/`) follow a consistent pattern:
- Target `lxc_containers` host group
- Load variables from `vars/global.yml` and `vars/secrets.yml`
- Create directory structure with proper permissions
- Generate `.env` files with secrets (escaped for Docker Compose)
- Deploy using `community.docker.docker_compose_v2` module
- Recreate containers when environment files change

Traefik is the ingress controller, using Cloudflare DNS challenge for wildcard SSL certificates.

## Key Implementation Details

### Cloud-Init User Data

The template creation uses Jinja2 templating to inject SSH keys from `vars/secrets.yml` into cloud-init user-data snippets stored in `/var/lib/vz/snippets/`. The snippet is referenced during VM creation via `--cicustom user=`.

### Guest Agent Requirement

The VM clone playbook waits for IP assignment by querying the QEMU guest agent. VMs must have qemu-guest-agent installed and running for this to work. The template creation playbook installs this automatically.

### Docker Compose v2

All Docker deployments use the `community.docker.docker_compose_v2` module (not the deprecated v1 module). Compose files use the modern `compose.yaml` filename.

### Inventory Auto-Update

The `clone_and_start_vm.yaml` playbook uses Jinja2 templating to preserve existing VM entries while adding new ones. It performs regex manipulation on `inventory.ini` to cleanly update the `[ubuntu_vms]` section.

## Sensitive Information

All sensitive data (SSH keys, passwords, API tokens, domain names) is stored in `vars/secrets.yml` and never committed. When adding new secrets:
1. Add the variable to `vars/secrets.yml`
2. Reference it in playbooks via `{{ variable_name }}`
3. Use `no_log: true` for tasks that display sensitive values
4. For Docker Compose .env files, escape dollar signs: `{{ variable | replace('$', '$$') }}`

## Common Patterns

**Checking existence before creation**: Most playbooks check if resources (VMs, containers, ZFS pools) exist before attempting creation to provide clear error messages.

**Idempotency**: Playbooks use `changed_when` and `failed_when` to ensure Ansible correctly tracks state changes even when using `command` or `shell` modules.

**Storage references**: Global variables like `proxmox_storage_vm` and `proxmox_network_bridge` centralize infrastructure configuration, making it easy to adapt to different Proxmox setups.