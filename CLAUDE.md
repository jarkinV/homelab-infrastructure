# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Proxmox automation repository using Ansible to manage VMs, LXC containers, ZFS storage, and Docker applications. The infrastructure is designed for a homelab environment with automated provisioning and configuration.

## Running Playbooks

All playbooks are run from the repository root using `ansible-playbook`:

```bash
# Network operations (setup NAT for internal network)
ansible-playbook playbooks/network/setup_nat_bridge.yaml
ansible-playbook playbooks/network/check_nat_status.yaml
ansible-playbook playbooks/network/setup_wifi.yaml  # Setup WiFi connection on Proxmox host

# VM operations
ansible-playbook playbooks/vm/create_ubuntu_cloud_template.yaml
ansible-playbook playbooks/vm/clone_and_start_vm.yaml -e "new_vm_id=101 new_vm_name=ubuntu-vm-101"
ansible-playbook playbooks/vm/setup_docker.yaml
ansible-playbook playbooks/vm/attach_zfs_datasets.yaml -e "vm_id=100"

# LXC container operations
ansible-playbook playbooks/lxc/create_nat_container.yaml  # NAT network (10.0.0.X)
ansible-playbook playbooks/lxc/create_container.yaml      # Direct network (192.168.X.X)
ansible-playbook playbooks/lxc/install_docker.yaml
ansible-playbook playbooks/lxc/update_and_upgrade.yaml
ansible-playbook playbooks/lxc/setup_docker_container.yaml  # Complete setup: create + update + docker
ansible-playbook playbooks/lxc/install_tailscale.yaml      # VPN access to internal network
ansible-playbook playbooks/lxc/setup_dnsmasq.yaml          # Internal DNS server

# ZFS storage operations
ansible-playbook playbooks/zfs/create_zfs_pool.yaml
ansible-playbook playbooks/zfs/configure_zfs_storage.yaml
ansible-playbook playbooks/zfs/add_zfs_storage.yaml
ansible-playbook playbooks/zfs/attach_zfs_datasets.yaml -e "ct_id=100"  # Attach datasets to LXC
ansible-playbook playbooks/zfs/install_sanoid.yaml         # Automated ZFS snapshots

# Application deployments
ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
ansible-playbook playbooks/apps/actual/deploy_actual.yaml
ansible-playbook playbooks/apps/paperless/deploy_paperless.yaml
ansible-playbook playbooks/apps/monitoring/deploy_monitoring.yaml  # Prometheus + Grafana + Loki observability stack
```

## Architecture

### Network Architecture

The repository supports two networking modes:

**1. NAT Network (Recommended)**

LXC containers and VMs reside in an internal NAT network (10.0.0.0/24) isolated from the external network. This provides:
- Stable internal IP addresses independent of external network changes
- Network isolation and security
- Easy switching between different routers/networks
- Centralized access control via port forwarding

Architecture:
```
External Network (192.168.X.0/24)
         ↓
    [vmbr0] - Proxmox external interface
         ↓
    iptables (NAT + DNAT for port forwarding)
         ↓
    [vmbr1] - NAT gateway (10.0.0.1/24)
         ↓
    LXC/VM internal network (10.0.0.10, 10.0.0.11, ...)
```

Setup:
```bash
# Setup NAT bridge with port forwarding for Traefik (80, 443)
ansible-playbook playbooks/network/setup_nat_bridge.yaml

# Create LXC in NAT network
ansible-playbook playbooks/lxc/create_nat_container.yaml \
  -e "ct_id=100 ct_hostname=traefik ct_ip_address=10.0.0.10/24"
```

The NAT bridge (vmbr1) is configured in `vars/global.yml`:
- `proxmox_nat_bridge: vmbr1`
- `proxmox_nat_network: 10.0.0.0/24`
- `proxmox_nat_gateway: 10.0.0.1/24`

Port forwarding rules are defined in `playbooks/network/setup_nat_bridge.yaml` using iptables DNAT. By default, ports 80 and 443 are forwarded to 10.0.0.10 (Traefik LXC).

**2. Direct Network (Alternative)**

LXC containers receive IPs directly from the external network (192.168.X.X). This is simpler but less flexible.

```bash
ansible-playbook playbooks/lxc/create_container.yaml
```

**3. WiFi Connectivity**

The Proxmox host can connect to WiFi networks using USB WiFi adapters. The playbook automates the setup:

- Installs required packages: `wireless-tools`, `wpasupplicant`, `iw`
- Configures WPA supplicant with encrypted credentials
- Creates systemd service for automatic WiFi connection on boot
- Brings up the WiFi interface

WiFi interface is configured in `vars/global.yml` (`wifi_interface`), and credentials (SSID and password) are stored in `vars/secrets.yml`.

```bash
# Setup WiFi connection
ansible-playbook playbooks/network/setup_wifi.yaml
```

The playbook creates a systemd service (`wpa_supplicant@<interface>`) that automatically connects to the configured network on boot and handles reconnections.

### Inventory Management

- **Static hosts**: Defined in `inventory.ini` for Proxmox host and LXC containers
- **Dynamic VM inventory**: The `clone_and_start_vm.yaml` playbook automatically updates both `vars/vm_inventory.yml` and `inventory.ini` when new VMs are created
- **Host groups**:
  - `proxmox_hosts`: Proxmox hypervisor (root access)
  - `lxc_containers`: LXC containers in external network (192.168.X.X)
  - `lxc_internal_containers`: LXC containers in NAT network (10.0.0.X) - requires ProxyJump through Proxmox host
  - `ubuntu_vms`: Ubuntu cloud-init VMs (auto-managed section in inventory.ini)

**Accessing internal containers**: Containers in the NAT network (10.0.0.X) require SSH ProxyJump configuration:
```ini
[lxc_internal_containers]
traefik ansible_host=10.0.0.10 ansible_ssh_common_args='-o ProxyJump=proxmox'
```
This allows Ansible to connect through the Proxmox host to reach isolated internal containers.

### Variable Management

The repository uses a two-tier variable system:

- **`vars/global.yml`**: Non-sensitive shared configuration (template IDs, storage names, network bridges, default VM resources)
- **`vars/secrets.yml`**: Sensitive data (SSH keys, passwords, API tokens) - gitignored

Required variables in `vars/secrets.yml`:
- `ssh_public_keys`: SSH public keys for VM/container access
- `ct_password`: Default password for LXC containers
- `cf_dns_api_token`: Cloudflare API token for DNS challenges (Traefik)
- `domain`: Primary domain name for services
- `traefik_dashboard_credentials`: HTTP basic auth for Traefik dashboard
- `paperless_postgres_password`: PostgreSQL password for Paperless
- `tailscale_auth_key`: Tailscale authentication key for VPN setup
- `grafana_admin_password`: Grafana admin password (monitoring stack)
- `telegram_bot_token`: Telegram bot token for alerts (monitoring stack)
- `telegram_chat_id`: Telegram chat ID for alerts (monitoring stack)
- `smtp_host`: SMTP server for email alerts (monitoring stack)
- `smtp_from`: Email sender address (monitoring stack)
- `smtp_username`: SMTP username (monitoring stack)
- `smtp_password`: SMTP password (monitoring stack)
- `smtp_to`: Email recipient for alerts (monitoring stack)
- `wifi_ssid`: WiFi network SSID
- `wifi_password`: WiFi network password

Most playbooks load both files via `vars_files`. Secrets are referenced using `{{ variable_name }}` from secrets.yml.

### VM Provisioning Flow

1. **Template creation** (`create_ubuntu_cloud_template.yaml`): Downloads Ubuntu cloud image, creates a cloud-init enabled template VM (ID 8000 by default)
2. **VM cloning** (`clone_and_start_vm.yaml`): Clones from template, resizes disk, configures resources, starts VM, waits for IP via guest agent, automatically updates inventory files
3. **Post-provisioning**: Additional playbooks like `setup_docker.yaml` can be run against the new VM

The clone playbook handles inventory updates automatically, adding new VMs to both the YAML tracking file and the Ansible inventory.

### LXC Container Workflow

LXC containers are used for privileged workloads like Docker. The following playbooks are available:

**Container Creation:**

**`create_nat_container.yaml`** (Recommended): Creates containers in NAT network (10.0.0.0/24)
- Internal network isolation
- Stable IP addresses independent of external network
- Requires NAT bridge to be configured first
- Example: `ansible-playbook playbooks/lxc/create_nat_container.yaml -e "ct_id=100 ct_hostname=traefik ct_ip_address=10.0.0.10/24"`

**`create_container.yaml`**: Creates containers in external network (192.168.X.X)
- Direct IP from external DHCP/static assignment
- Simpler setup but less flexible
- Example: `ansible-playbook playbooks/lxc/create_container.yaml -e "ct_id=222 ct_ip_address=192.168.8.222/24"`

Both playbooks configure:
- Nesting enabled (required for Docker)
- Privileged mode (root in container = root on host)
- Static IP configuration
- SSH key injection from `vars/secrets.yml`

**Post-Creation Setup:**

**`update_and_upgrade.yaml`**: Updates all packages and upgrades the system
- Runs apt update && apt upgrade
- Should be run after container creation

**`install_docker.yaml`**: Installs Docker Engine and Docker Compose v2
- Adds Docker's official GPG key and repository
- Installs docker-ce, docker-ce-cli, containerd.io
- Installs docker-compose-plugin

**`setup_docker_container.yaml`**: Meta-playbook orchestrating complete setup
- Imports create_container.yaml → update_and_upgrade.yaml → install_docker.yaml
- One-command setup for fully configured Docker-ready containers
- Example: `ansible-playbook playbooks/lxc/setup_docker_container.yaml -e "ct_id=222 ct_ip_address=192.168.8.222/24"`

**Network Services:**

**`install_tailscale.yaml`**: Install Tailscale VPN for remote access
- Target: `lxc_internal_containers` group
- Advertises internal network (10.0.0.0/24) as Tailscale subnet route
- Enables Tailscale SSH for secure remote access
- Requires `tailscale_auth_key` in vars/secrets.yml
- See "Remote Access with Tailscale VPN" section below

**`setup_dnsmasq.yaml`**: Configure internal DNS server
- Target: `lxc_internal_containers` group
- Provides wildcard DNS (*.domain → 10.0.0.10)
- Resolves internal services to Traefik proxy
- Integrates with Tailscale for VPN DNS resolution
- See "Internal DNS and Service Discovery" section below

### ZFS Storage Architecture

The ZFS setup is designed for Docker data persistence:

1. **Pool creation** (`create_zfs_pool.yaml`): Creates RAIDZ1 pool from disk serials with compression and optimized settings
2. **Dataset configuration** (`configure_zfs_storage.yaml`): Creates individual ZFS datasets for each Docker app (traefik, actual, paperless, etc.)
3. **VM attachment** (`attach_zfs_datasets.yaml` in playbooks/vm/): Uses 9p filesystem (virtio-9p-pci) to share ZFS datasets with VMs without NFS overhead
4. **LXC attachment** (`attach_zfs_datasets.yaml` in playbooks/zfs/): Mounts ZFS datasets as bind mounts in LXC containers

**VM vs LXC ZFS Access:**
- **VMs**: Use 9p (virtio-9p-pci) filesystem sharing - datasets mounted via virtio, no block device needed
- **LXC**: Use bind mounts (mp0, mp1, etc.) - datasets mounted directly as container mount points

Both methods allow the Proxmox host to maintain direct ZFS control while containers/VMs access the data.

### Backup and Recovery

The repository uses Sanoid for automated ZFS snapshot management:

**`install_sanoid.yaml`**: Install and configure automated ZFS snapshots
- Target: `proxmox_hosts`
- Installs sanoid package
- Configures snapshot retention policies from `docs/sanoid.conf`
- Sets up systemd timer for automated snapshots
- Three snapshot templates available:
  - **minimal**: Keeps 1 daily, 1 weekly, 1 monthly snapshot
  - **production**: Keeps 7 daily, 4 weekly, 3 monthly snapshots

Configuration file: `/etc/sanoid/sanoid.conf`
Recovery procedures: See `docs/recovery-guide.md` (Ukrainian)

Example retention policy for production datasets:
```
[docker/traefik]
use_template = production
recursive = yes
```

The recovery guide covers:
- Restoring from ZFS snapshots
- Recovering Proxmox backups
- Rebuilding containers from snapshots
- Emergency procedures

### Internal DNS and Service Discovery

The repository includes a dnsmasq-based internal DNS server for wildcard domain resolution:

**`setup_dnsmasq.yaml`**: Configure dnsmasq DNS server on internal LXC
- Target: `lxc_internal_containers` group
- Provides wildcard DNS resolution (*.domain.com → 10.0.0.10)
- All subdomains resolve to Traefik reverse proxy
- Listens on all interfaces (0.0.0.0) for network-wide availability
- Integrates with Tailscale DNS for VPN clients
- Includes validation and testing of DNS resolution

Configuration: `/etc/dnsmasq.conf`
Logs: `/var/log/dnsmasq.log`

Example DNS flow:
```
Client requests: app.varshavskyi.com
         ↓
    dnsmasq (10.0.0.10)
         ↓
    Returns: 10.0.0.10
         ↓
    Traefik proxy routes to correct Docker container
```

For detailed setup instructions, see `playbooks/lxc/README_dnsmasq.md`

### Remote Access with Tailscale VPN

Tailscale provides secure remote access to the internal NAT network:

**`install_tailscale.yaml`**: Install and configure Tailscale VPN
- Target: `lxc_internal_containers` group
- Installs Tailscale from official repository
- Authenticates using auth key from `vars/secrets.yml`
- Advertises internal network (10.0.0.0/24) as subnet route
- Enables Tailscale SSH for secure remote access
- Requires `tailscale_auth_key` in secrets.yml

**Key capabilities:**
- Remote access to internal 10.0.0.0/24 network from anywhere
- Secure SSH access without exposing ports
- DNS resolution through dnsmasq (port 53)
- Access internal services via VPN tunnel

**Setup workflow:**
1. Create internal LXC container (10.0.0.X)
2. Run `install_tailscale.yaml` to enable VPN
3. Run `setup_dnsmasq.yaml` for DNS (optional)
4. Approve subnet routes in Tailscale admin console
5. Access internal services from any Tailscale-connected device

### Application Deployment Pattern

Application playbooks (in `playbooks/apps/`) follow a consistent pattern:
- Target `lxc_internal_containers` host group (NAT network)
- Load variables from `vars/global.yml` and `vars/secrets.yml`
- Create directory structure with proper permissions
- Generate `.env` files with secrets (escaped for Docker Compose)
- Deploy using `community.docker.docker_compose_v2` module
- Recreate containers when environment files change

Traefik is the ingress controller, using Cloudflare DNS challenge for wildcard SSL certificates.

### Deployed Applications

The repository includes playbooks for deploying the following applications:

**`traefik/deploy_traefik.yaml`**: Traefik reverse proxy and ingress controller
- Automatic SSL/TLS with Let's Encrypt
- Cloudflare DNS challenge for wildcard certificates (*.domain.com)
- Dashboard with HTTP basic auth at traefik.domain.com
- Routes traffic to other Docker containers
- Prometheus metrics enabled (`:8082` endpoint for monitoring)
- ZFS dataset: `docker/traefik`
- Docker networks: proxy, tunnel, monitoring

**`actual/deploy_actual.yaml`**: Actual Budget - personal finance manager
- Simple budget tracking and finance management
- Single container deployment
- Accessible at budget.domain.com via Traefik
- ZFS dataset: `docker/actual`
- Data persistence in /data volume

**`paperless/deploy_paperless.yaml`**: Paperless-ngx - document management system
- Multi-container stack (app + PostgreSQL + Redis)
- OCR support for scanned documents (English + Polish)
- Web interface at paperless.domain.com
- ZFS dataset: `docker/paperless`
- Components:
  - paperless-webserver: Main application
  - paperless-db: PostgreSQL database
  - paperless-redis: Redis cache
- Requires `paperless_postgres_password` in secrets.yml

**`monitoring/deploy_monitoring.yaml`**: Complete observability stack
- Comprehensive monitoring and logging solution for all Docker services
- ZFS dataset: `docker/monitoring`
- Configuration templates: `.env.j2`, `compose.yaml.j2`, `prometheus.yml.j2`, `alert-rules.yml.j2`, `alertmanager.yml.j2`, `loki-config.yml.j2`, `promtail-config.yml.j2`
- Components:
  - **Prometheus**: Metrics collection and storage (7-day retention) at prometheus.domain.com
  - **Grafana**: Visualization dashboards at grafana.domain.com
  - **Alertmanager**: Alert routing to Telegram and Email
  - **Loki**: Log aggregation (7-day retention)
  - **Promtail**: Docker log collection
  - **cAdvisor**: Docker container metrics
  - **Node Exporter**: Host system metrics
  - **PostgreSQL Exporter**: Database metrics (Paperless)
  - **Redis Exporter**: Cache metrics (Paperless)
- Requires additional secrets: `grafana_admin_password`, `telegram_bot_token`, `telegram_chat_id`, `smtp_*` variables
- Pre-configured alerts: container down, high CPU/memory, low disk space, database issues
- Full idempotency: tracks all 7 configuration files
- See `playbooks/apps/monitoring/README.md` for complete setup guide, Grafana dashboard imports, and troubleshooting

All applications use Traefik for SSL termination and routing, with persistent data stored on ZFS datasets.

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

## External Documentation

Additional documentation is available in the repository:

**Network Configuration:**
- `playbooks/network/README.md` - Comprehensive NAT network setup guide (Ukrainian, 218 lines)
  - Detailed NAT architecture explanation
  - Port forwarding configuration
  - Troubleshooting NAT connectivity issues
  - iptables rules and persistence

**DNS Server Setup:**
- `playbooks/lxc/README_dnsmasq.md` - dnsmasq DNS server guide (English, 133 lines)
  - Complete dnsmasq installation and configuration
  - Wildcard DNS setup for internal services
  - Integration with Tailscale VPN
  - Testing and validation procedures

**Backup and Recovery:**
- `docs/recovery-guide.md` - System recovery procedures (Ukrainian, 440 lines)
  - ZFS snapshot restoration
  - Proxmox backup recovery
  - Container rebuild from snapshots
  - Emergency recovery procedures
  - Step-by-step recovery workflows

**Snapshot Configuration:**
- `docs/sanoid.conf` - Sanoid snapshot retention policies (40 lines)
  - Minimal and production snapshot templates
  - Dataset-specific retention policies
  - Used by `playbooks/zfs/install_sanoid.yaml`

## Common Patterns

**Checking existence before creation**: Most playbooks check if resources (VMs, containers, ZFS pools) exist before attempting creation to provide clear error messages.

**Idempotency**: Playbooks use `changed_when` and `failed_when` to ensure Ansible correctly tracks state changes even when using `command` or `shell` modules.

**Storage references**: Global variables like `proxmox_storage_vm` and `proxmox_network_bridge` centralize infrastructure configuration, making it easy to adapt to different Proxmox setups.
- ansible playbooks should be idenpotent. If playbook deploy the app in docker and some configuration changed (env file, compose file) then next run playbook should apply these changes and restart the app in docker