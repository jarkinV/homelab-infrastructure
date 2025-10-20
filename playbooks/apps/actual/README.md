# Actual Budget Deployment

This directory contains the Ansible playbook and configuration templates for deploying Actual Budget with Docker Compose.

## Overview

[Actual Budget](https://actualbudget.org/) is a local-first personal finance tool. It is 100% free and open source, written in NodeJS, with a simple and intuitive interface.

## Directory Structure

```
playbooks/apps/actual/
├── deploy_actual.yaml    # Main Ansible playbook
├── compose.yaml.j2       # Docker Compose configuration template
├── .env.j2               # Environment variables template
└── README.md             # This file
```

## Configuration Files

### `.env.j2`
Environment variables used by Docker Compose:
- `DATA_LOCATION` - Path to Actual Budget data directory
- `DOMAIN` - Primary domain name for Traefik routing

### `compose.yaml.j2`
Docker Compose file that configures:
- Actual Budget server container (v25.10.0)
- Volume mount for data persistence
- Traefik labels for HTTPS routing
- Docker network connection to proxy

**Templated variables:**
- `{{ actual_base_dir }}` - Base directory for Actual Budget (/mnt/actual)

## Deployment

### Prerequisites

1. **Required variables in `vars/secrets.yml`:**
   ```yaml
   domain: "yourdomain.com"
   ```

2. **ZFS dataset:** Actual data should be on ZFS dataset `/mnt/actual`
   ```bash
   # Ensure monitoring dataset is created
   ansible-playbook playbooks/zfs/configure_zfs_storage.yaml

   # Attach to Traefik LXC (container 100)
   ansible-playbook playbooks/zfs/attach_zfs_datasets.yaml -e "ct_id=100"
   ```

3. **Traefik:** Must be deployed and running for HTTPS routing
   ```bash
   ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
   ```

### Deploy Actual Budget

```bash
# From repository root
ansible-playbook playbooks/apps/actual/deploy_actual.yaml
```

### What the Playbook Does

1. **Setup Docker:**
   - Ensures Docker service is running
   - Creates Docker network: `proxy`

2. **Create directory structure:**
   - `/mnt/actual/` - Base directory
   - `/mnt/actual/data/` - Data storage (created by container)

3. **Deploy configuration files:**
   - `.env` - Environment variables
   - `compose.yaml` - Docker Compose configuration

4. **Deploy container:**
   - Pulls Actual Budget image (if not present)
   - Starts Actual Budget container
   - Configures automatic restarts

5. **Idempotent updates:**
   - Tracks changes to `compose.yaml` and `.env`
   - Automatically recreates container when configuration changes

## Accessing Actual Budget

### Web Interface
- URL: `https://budget.yourdomain.com`
- First-time setup:
  1. Create your first budget file
  2. Set up synchronization (optional)
  3. Import existing data (optional)

### Data Location
- Container data: `/mnt/actual/data/`
- Budget files: `/mnt/actual/data/server-files/`
- User data: `/mnt/actual/data/user-files/`

### Logs
```bash
# View live logs
docker logs -f actual

# View last 100 lines
docker logs --tail 100 actual
```

## Features

- **Local-first:** All data stored locally, sync optional
- **Budget management:** Zero-based budgeting system
- **Bank sync:** Optional bank account synchronization
- **Reports:** Income vs expenses, net worth tracking
- **Multi-device:** Sync across multiple devices (optional)
- **Import:** Import from YNAB, Mint, CSV

## Customization

### Change Actual Budget Version

Edit `compose.yaml.j2`:
```yaml
image: docker.io/actualbudget/actual-server:25.10.0  # Change version here
```

Then redeploy:
```bash
ansible-playbook playbooks/apps/actual/deploy_actual.yaml
```

### Change Subdomain

Edit `compose.yaml.j2`:
```yaml
- "traefik.http.routers.actual.rule=Host(`budget.${DOMAIN}`)"  # Change 'budget' here
```

### Add Monitoring Labels

To enable Prometheus metrics scraping (if available), add to `compose.yaml.j2`:
```yaml
labels:
  - "prometheus.io/scrape=true"
  - "prometheus.io/port=5006"
```

## Backup and Recovery

### Backup

Actual Budget data is stored on ZFS, which supports snapshots:

```bash
# Manual snapshot
zfs snapshot tank/actual@backup-$(date +%Y%m%d)

# List snapshots
zfs list -t snapshot tank/actual

# Automated snapshots via Sanoid (if configured)
# See docs/sanoid.conf for configuration
```

### Manual Backup

```bash
# Backup budget files
tar -czf actual-backup-$(date +%Y%m%d).tar.gz /mnt/actual/data/

# Copy to remote location
scp actual-backup-*.tar.gz user@backup-server:/backups/
```

### Restore from Snapshot

```bash
# List available snapshots
zfs list -t snapshot tank/actual

# Restore from snapshot
zfs rollback tank/actual@backup-20250120

# Or restore specific files
zfs send tank/actual@backup-20250120 | zfs receive tank/actual-restore
```

### Restore from Manual Backup

```bash
# Stop container
docker stop actual

# Extract backup
tar -xzf actual-backup-20250120.tar.gz -C /

# Start container
docker start actual
```

## Troubleshooting

### Container Won't Start

1. Check Docker logs:
   ```bash
   docker logs actual
   ```

2. Verify permissions:
   ```bash
   ls -la /mnt/actual/data/
   ```

3. Check if data directory exists:
   ```bash
   docker exec actual ls -la /data/
   ```

### Cannot Access via URL

1. Verify Traefik is running:
   ```bash
   docker ps | grep traefik
   ```

2. Check Traefik logs:
   ```bash
   docker logs traefik | grep actual
   ```

3. Verify DNS resolution:
   ```bash
   nslookup budget.yourdomain.com
   ```

4. Check container is on proxy network:
   ```bash
   docker network inspect proxy
   ```

### Data Not Persisting

1. Check volume mount:
   ```bash
   docker inspect actual | grep -A 5 Mounts
   ```

2. Verify data directory:
   ```bash
   ls -la /mnt/actual/data/
   ```

3. Check container logs for write errors:
   ```bash
   docker logs actual | grep -i error
   ```

### Slow Performance

1. Check container resources:
   ```bash
   docker stats actual
   ```

2. Check disk I/O:
   ```bash
   zpool iostat -v tank 1
   ```

3. Verify ZFS ARC hit rate:
   ```bash
   arc_summary | grep "Hit Rate"
   ```

## Updating

### Update Actual Budget Version

1. Edit `compose.yaml.j2` with new version
2. Redeploy:
   ```bash
   ansible-playbook playbooks/apps/actual/deploy_actual.yaml
   ```

The playbook will:
1. Detect compose file change
2. Pull new image
3. Recreate container with new version

### Manual Update

```bash
# Pull new image
docker pull actualbudget/actual-server:latest

# Stop and remove old container
docker stop actual
docker rm actual

# Redeploy via playbook
ansible-playbook playbooks/apps/actual/deploy_actual.yaml
```

## Security Considerations

1. **HTTPS only:** All traffic encrypted via Traefik
2. **Network isolation:** Container only accessible via proxy network
3. **Data security:** All data stored locally on ZFS with encryption option
4. **Container security:** Runs with `no-new-privileges:true`
5. **Updates:** Regular updates recommended for security patches

## Integration

### Bank Sync (Optional)

Actual Budget supports bank synchronization via SimpleFIN:

1. Sign up for SimpleFIN Bridge
2. Configure in Actual Budget settings
3. Link your bank accounts

### API Access

Actual Budget provides an API for automation:

```bash
# Example: Export budget data
curl -X GET https://budget.yourdomain.com/api/v1/budgets
```

See [Actual Budget API documentation](https://actualbudget.org/docs/api/) for details.

### Mobile App

Actual Budget has official mobile apps:
- iOS: Available on App Store
- Android: Available on Google Play

Configure sync in the app to connect to your server.

## Monitoring

If the monitoring stack is deployed, Actual Budget container metrics are available:

- **Container metrics:** CPU, memory, network via cAdvisor
- **Application logs:** Available via Loki
- **Alerts:** Container down alert configured

Access via Grafana: `https://grafana.yourdomain.com`

## Related Documentation

- [Actual Budget Official Docs](https://actualbudget.org/docs/)
- [Actual Budget GitHub](https://github.com/actualbudget/actual)
- [Docker Hub](https://hub.docker.com/r/actualbudget/actual-server)
- Main project documentation: `../../CLAUDE.md`

## Support

- **Community:** [Actual Budget Discord](https://discord.gg/pRYNYr4W5A)
- **GitHub Issues:** [Report bugs](https://github.com/actualbudget/actual/issues)
- **Documentation:** [Official docs](https://actualbudget.org/docs/)
