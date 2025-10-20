# Monitoring Stack Setup Guide

This guide covers the deployment and configuration of a complete observability stack with Prometheus, Grafana, Loki, and Alertmanager for monitoring Docker services.

## Overview

The monitoring stack provides:
- **Metrics Collection**: Prometheus scrapes metrics from Docker containers, host systems, and applications
- **Visualization**: Grafana dashboards for metrics and logs
- **Log Aggregation**: Loki collects and indexes logs from all Docker containers
- **Alerting**: Alertmanager routes alerts to Telegram and Email

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Monitoring Stack                         │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Prometheus ────> Alertmanager ────> Telegram/Email         │
│      ↑                                                       │
│      │ scrapes metrics                                       │
│      │                                                       │
│  ┌───┴──────┬──────────┬──────────┬──────────┬──────────┐  │
│  │          │          │          │          │          │  │
│  cAdvisor  Node    PostgreSQL  Redis     Traefik   Promtail│
│  (Docker)  Exporter (Paperless) (Paperless)        (Logs)  │
│            (Host)                                      ↓     │
│                                                      Loki    │
│                                                        ↓     │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Grafana Dashboards                      │   │
│  │  - Docker Containers  - Host Metrics  - Logs        │   │
│  │  - PostgreSQL        - Redis          - Traefik     │   │
│  └─────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

## Components

### Core Services

1. **Prometheus** (`:9090`)
   - Metrics collection and storage
   - 7-day retention period
   - Scrapes all exporters every 15 seconds
   - Web UI: `https://prometheus.yourdomain.com`

2. **Grafana** (`:3000`)
   - Visualization dashboards
   - Data source integration (Prometheus + Loki)
   - Pre-configured with admin credentials
   - Web UI: `https://grafana.yourdomain.com`

3. **Alertmanager** (`:9093`)
   - Alert routing and deduplication
   - Telegram bot integration
   - SMTP email notifications
   - Severity-based routing (critical → email + telegram, warning → telegram only)

4. **Loki** (`:3100`)
   - Log aggregation and indexing
   - 7-day log retention
   - Integrates with Promtail for log collection

5. **Promtail** (`:9080`)
   - Docker log collection
   - Automatic container discovery
   - Labels logs with container metadata

### Exporters

6. **cAdvisor** (`:8080`)
   - Docker container metrics
   - CPU, memory, network, disk I/O per container
   - Lightweight resource monitoring

7. **Node Exporter** (`:9100`)
   - Host system metrics
   - CPU, memory, disk, network for LXC host
   - System-level monitoring

8. **PostgreSQL Exporter** (`:9187`)
   - Paperless database metrics
   - Connection pool, query performance
   - Database health monitoring

9. **Redis Exporter** (`:9121`)
   - Paperless cache metrics
   - Memory usage, hit rate, keys

## Prerequisites

### 1. Required Secrets

Add these variables to `vars/secrets.yml`:

```yaml
# Grafana
grafana_admin_password: "your-secure-password"

# Alertmanager - Telegram
telegram_bot_token: "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"
telegram_chat_id: "123456789"

# Alertmanager - Email (SMTP)
smtp_host: "smtp.gmail.com:587"
smtp_from: "alerts@yourdomain.com"
smtp_username: "your-email@gmail.com"
smtp_password: "your-app-password"
smtp_to: "your-email@gmail.com"

# Paperless (required for database exporter)
paperless_postgres_password: "your-paperless-db-password"
```

### 2. Telegram Bot Setup

1. Create a Telegram bot:
   - Message `@BotFather` on Telegram
   - Send `/newbot` and follow instructions
   - Save the bot token

2. Get your chat ID:
   - Message your new bot
   - Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
   - Find your chat ID in the JSON response

### 3. Email Setup (Gmail Example)

For Gmail:
1. Enable 2-factor authentication on your Google account
2. Generate an app password: https://myaccount.google.com/apppasswords
3. Use the app password (16 characters, no spaces) as `smtp_password`

For other providers, adjust `smtp_host` accordingly:
- Outlook: `smtp.office365.com:587`
- Yahoo: `smtp.mail.yahoo.com:587`
- Custom SMTP: `smtp.yourserver.com:587`

### 4. ZFS Dataset

The monitoring stack requires a ZFS dataset. This was automatically added to the configuration files, but you need to create it:

```bash
# Create ZFS dataset
ansible-playbook playbooks/zfs/configure_zfs_storage.yaml

# Attach to Traefik LXC (container 100)
ansible-playbook playbooks/zfs/attach_zfs_datasets.yaml -e "ct_id=100"
```

This creates `/mnt/monitoring` inside the Traefik LXC container.

## Deployment

### 1. Deploy the Stack

```bash
ansible-playbook playbooks/apps/deploy_monitoring.yaml
```

This will:
- Create necessary directories
- Generate configuration files for all services
- Deploy all containers via Docker Compose
- Set up Traefik routes for Grafana and Prometheus

### 2. Enable Traefik Metrics

**If Traefik was already deployed**, you need to redeploy it to enable metrics:

```bash
ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
```

This will automatically add:
- Prometheus metrics endpoint (`:8082`)
- Monitoring network connection
- Required command-line flags

**Note:** The Traefik playbook (`traefik/deploy_traefik.yaml`) now includes Prometheus metrics configuration by default. New Traefik deployments will have metrics enabled automatically.

## Grafana Configuration

### 1. Login

Visit `https://grafana.yourdomain.com` and login with:
- Username: `admin`
- Password: (value from `grafana_admin_password` in secrets.yml)

### 2. Add Data Sources

**Add Prometheus:**
1. Go to **Connections** → **Data Sources** → **Add data source**
2. Select **Prometheus**
3. Set URL: `http://prometheus:9090`
4. Click **Save & Test**

**Add Loki:**
1. Go to **Connections** → **Data Sources** → **Add data source**
2. Select **Loki**
3. Set URL: `http://loki:3100`
4. Click **Save & Test**

### 3. Import Dashboards

Grafana has a large library of community dashboards. Import these recommended ones:

**Docker Monitoring:**
- Dashboard ID: `193`
- Name: "Docker containers (cAdvisor)"
- Shows: CPU, memory, network for all containers

**Host System:**
- Dashboard ID: `1860`
- Name: "Node Exporter Full"
- Shows: System metrics, disk, CPU, memory

**Traefik:**
- Dashboard ID: `15489`
- Name: "Traefik Official Dashboard"
- Shows: HTTP requests, response times, error rates

**PostgreSQL:**
- Dashboard ID: `9628`
- Name: "PostgreSQL Database"
- Shows: Database connections, queries, locks

**Redis:**
- Dashboard ID: `11835`
- Name: "Redis Dashboard for Prometheus"
- Shows: Memory, commands, hit rate

**Logs:**
- Dashboard ID: `13639`
- Name: "Logs / App"
- Shows: Loki logs with filtering

**Import process:**
1. Click **Dashboards** → **New** → **Import**
2. Enter dashboard ID
3. Select Prometheus/Loki as data source
4. Click **Import**

## Alert Configuration

### Default Alerts

The stack includes pre-configured alerts:

**Container Alerts:**
- `ContainerDown`: Container has been down for 2+ minutes (Critical)
- `ContainerHighCPU`: Container using >80% CPU for 5+ minutes (Warning)
- `ContainerHighMemory`: Container using >90% memory for 5+ minutes (Warning)

**Host Alerts:**
- `HostHighCPU`: Host CPU usage >80% for 5+ minutes (Warning)
- `HostHighMemory`: Host memory usage >90% for 5+ minutes (Critical)
- `HostDiskSpaceLow`: Disk space <10% (Critical)
- `HostDiskSpaceWarning`: Disk space <20% (Warning)

**Database Alerts:**
- `PostgreSQLDown`: PostgreSQL has been down for 2+ minutes (Critical)
- `RedisDown`: Redis has been down for 2+ minutes (Critical)

**Traefik Alerts:**
- `TraefikHighErrorRate`: 5xx error rate >0.05 req/sec for 5+ minutes (Warning)

### Alert Routing

- **Critical alerts** → Telegram + Email
- **Warning alerts** → Telegram only
- Alerts are grouped by: alertname, cluster, service
- Repeat interval: 12 hours

### Testing Alerts

**Test Telegram notifications:**
```bash
# Stop a container to trigger ContainerDown alert
docker stop actual

# Wait 2 minutes
# Check Telegram for alert notification

# Restart container
docker start actual
```

**Test email notifications:**
```bash
# Trigger critical alert by filling disk space
# Or manually test via Alertmanager API
curl -X POST http://10.0.0.10:9093/api/v1/alerts -d '[{
  "labels": {"alertname": "TestAlert", "severity": "critical"},
  "annotations": {"summary": "Test alert"}
}]'
```

### Customizing Alerts

Alert rules are in `/mnt/monitoring/config/prometheus/alert-rules.yml`. After editing:

```bash
# Reload Prometheus configuration
docker exec prometheus killall -HUP prometheus

# Or restart the container
cd /mnt/monitoring
docker compose restart prometheus
```

## Maintenance

### View Logs

```bash
# All services
cd /mnt/monitoring
docker compose logs -f

# Specific service
docker compose logs -f grafana
docker compose logs -f prometheus
docker compose logs -f alertmanager
```

### Check Service Health

```bash
# Prometheus targets
curl http://localhost:9090/api/v1/targets | jq

# Grafana health
curl http://localhost:3000/api/health

# Loki health
curl http://localhost:3100/ready
```

### Backup

Important directories to backup:
- `/mnt/monitoring/data/grafana` - Dashboards and settings
- `/mnt/monitoring/data/prometheus` - Metrics data (optional, retained for 7 days)
- `/mnt/monitoring/config/` - All configuration files

Since this is on a ZFS dataset, you can use Sanoid for automated snapshots.

### Update Services

```bash
cd /mnt/monitoring

# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d
```

## Troubleshooting

### Prometheus Can't Scrape Targets

**Check if target is reachable:**
```bash
# From inside Prometheus container
docker exec prometheus wget -O- http://cadvisor:8080/metrics
docker exec prometheus wget -O- http://node-exporter:9100/metrics
```

**Check Docker networks:**
```bash
docker network inspect monitoring
docker network inspect paperless-network
```

### Grafana Can't Connect to Data Sources

**Verify URLs:**
- Prometheus: `http://prometheus:9090` (not localhost)
- Loki: `http://loki:3100` (not localhost)

**Check containers are in same network:**
```bash
docker inspect grafana | grep -A 10 Networks
```

### Alerts Not Sending

**Check Alertmanager logs:**
```bash
docker compose logs alertmanager | grep -i error
```

**Verify secrets in .env:**
```bash
# Inside LXC container
cat /mnt/monitoring/.env
```

**Test Telegram bot:**
```bash
curl -X POST "https://api.telegram.org/bot<TOKEN>/sendMessage" \
  -d "chat_id=<CHAT_ID>" \
  -d "text=Test message"
```

### High Resource Usage

**Reduce Prometheus retention:**
Edit `/mnt/monitoring/compose.yaml`:
```yaml
command:
  - '--storage.tsdb.retention.time=3d'  # Change from 7d to 3d
```

**Reduce Loki retention:**
Edit `/mnt/monitoring/config/loki/loki-config.yml`:
```yaml
limits_config:
  retention_period: 72h  # Change from 168h
```

**Disable unused exporters:**
Comment out services in compose.yaml:
```yaml
# postgres-exporter-paperless:
#   image: ...
```

## Monitoring Additional Applications

To add monitoring for new applications (e.g., Immich):

### 1. Add Scrape Config

Edit `/mnt/monitoring/config/prometheus/prometheus.yml`:
```yaml
scrape_configs:
  - job_name: 'postgres-immich'
    static_configs:
      - targets: ['postgres-exporter-immich:9187']
```

### 2. Add Exporter to Compose

Edit `/mnt/monitoring/compose.yaml`:
```yaml
postgres-exporter-immich:
  image: prometheuscommunity/postgres-exporter:v0.16.0
  container_name: postgres-exporter-immich
  environment:
    - DATA_SOURCE_NAME=postgresql://user:pass@immich-postgres:5432/immich?sslmode=disable
  networks:
    - monitoring
    - immich-network
```

### 3. Restart Stack

```bash
cd /mnt/monitoring
docker compose up -d
```

## Security Considerations

1. **Grafana Password**: Change default password on first login
2. **Network Isolation**: Exporters are only accessible within the `monitoring` network
3. **Traefik Integration**: Prometheus and Grafana use SSL via Traefik
4. **Secret Management**: Sensitive data is in `.env` with `0600` permissions
5. **Email Credentials**: Use app passwords, not your main account password

## Performance Impact

**Expected Resource Usage:**
- Prometheus: ~200MB RAM, 1-2GB disk (7 days retention)
- Grafana: ~150MB RAM
- Loki: ~100MB RAM, 500MB-1GB disk (7 days logs)
- Alertmanager: ~50MB RAM
- cAdvisor: ~50MB RAM
- Node Exporter: ~20MB RAM
- Each additional exporter: ~20MB RAM

**Total overhead:** ~600MB RAM for full stack

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/grafana/latest/)
- [Loki Documentation](https://grafana.com/docs/loki/latest/)
- [Alertmanager Configuration](https://prometheus.io/docs/alerting/latest/configuration/)
- [Grafana Dashboards Library](https://grafana.com/grafana/dashboards/)
