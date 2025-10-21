# Modular Monitoring Stack Deployment

This directory contains a modular approach to deploying the monitoring stack, allowing you to start with basic monitoring and incrementally add features as needed.

## Deployment Phases

### Phase 1: Basic Monitoring (Required)

Deploy Prometheus + Grafana + cAdvisor + Node Exporter for basic metrics collection and visualization.

```bash
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_basic.yaml
```

**What you get:**
- Prometheus: Metrics collection at https://prometheus.yourdomain.com
- Grafana: Dashboards at https://grafana.yourdomain.com
- cAdvisor: Docker container metrics (CPU, memory, network)
- Node Exporter: Host system metrics (CPU, memory, disk)

**Required secrets in vars/secrets.yml:**
```yaml
domain: "yourdomain.com"
grafana_admin_password: "your-secure-password"
```

**Next steps:**
1. Login to Grafana (admin / grafana_admin_password)
2. Add Prometheus data source: http://prometheus:9090
3. Import dashboards:
   - Docker Monitoring: Dashboard ID 193
   - Host System: Dashboard ID 1860
   - Traefik: Dashboard ID 15489

### Phase 2: Logging (Optional)

Add Loki and Promtail for centralized log aggregation from all Docker containers.

```bash
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_logging.yaml
```

**What you get:**
- Loki: Log aggregation with 7-day retention
- Promtail: Automatic log collection from all Docker containers

**Next steps:**
1. Add Loki data source in Grafana: http://loki:3100
2. Import log dashboard: Dashboard ID 13639

### Phase 3: Alerting (Optional)

Add Alertmanager with Telegram and Email notifications for critical events.

```bash
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_alerts.yaml
```

**What you get:**
- Alertmanager: Alert routing and notification
- Alert rules: Container status, resource usage, database health
- Telegram notifications for warnings
- Telegram + Email for critical alerts

**Required secrets in vars/secrets.yml:**
```yaml
telegram_bot_token: "1234567890:ABCdefGHIjklMNOpqrsTUVwxyz"
telegram_chat_id: "123456789"
smtp_host: "smtp.gmail.com:587"
smtp_from: "alerts@yourdomain.com"
smtp_username: "your-email@gmail.com"
smtp_password: "your-app-password"
smtp_to: "your-email@gmail.com"
```

**Test alerts:**
```bash
# Stop a container to trigger alert
docker stop actual
# Wait 2 minutes for ContainerDown alert
# Check Telegram and email
```

### Phase 4: Application Exporters (Optional)

Add PostgreSQL and Redis exporters for monitoring Paperless-ngx database and cache.

```bash
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_exporters.yaml
```

**What you get:**
- PostgreSQL Exporter: Paperless database metrics
- Redis Exporter: Paperless cache metrics

**Required secrets in vars/secrets.yml:**
```yaml
paperless_postgres_password: "your-paperless-db-password"
```

**Next steps:**
1. Import database dashboards in Grafana:
   - PostgreSQL: Dashboard ID 9628
   - Redis: Dashboard ID 11835

## Complete Deployment (All-in-One)

If you want to deploy everything at once:

```bash
ansible-playbook playbooks/apps/monitoring/deploy_monitoring.yaml
```

This deploys all phases in a single run. Requires all secrets configured.

## Deployment Order

The modular playbooks must be run in order:
1. **Basic** (required first)
2. **Logging** (optional, independent of alerts)
3. **Alerts** (optional, independent of logging)
4. **Exporters** (optional, can be added anytime after basic)

## Templates and Configuration

### Basic Deployment Templates
- `.env-basic.j2` - Domain and Grafana password only
- `compose-basic.yaml.j2` - 4 services (Prometheus, Grafana, cAdvisor, Node Exporter)
- `prometheus-basic.yml.j2` - Basic scrape configs without alert rules

### Full Deployment Templates
- `.env.j2` - All secrets (Grafana, Telegram, SMTP, database)
- `compose.yaml.j2` - All 9 services
- `prometheus.yml.j2` - All scrape configs with alert rules
- `alert-rules.yml.j2` - Alert definitions
- `alertmanager.yml.j2` - Notification routing
- `loki-config.yml.j2` - Log aggregation config
- `promtail-config.yml.j2` - Log collection config

## Directory Structure After Deployment

```
/mnt/zfs/docker/monitoring/
├── .env                          # Environment variables
├── compose.yaml                  # Docker Compose (grows with each phase)
├── config/
│   ├── prometheus/
│   │   ├── prometheus.yml       # Scrape configs (updated by each phase)
│   │   └── alert-rules.yml      # Added in Phase 3
│   ├── alertmanager/
│   │   └── alertmanager.yml     # Added in Phase 3
│   ├── loki/
│   │   └── loki-config.yml      # Added in Phase 2
│   └── promtail/
│       └── promtail-config.yml  # Added in Phase 2
└── data/
    ├── grafana/                  # Dashboards and settings
    ├── prometheus/               # Metrics data (7 days)
    ├── loki/                     # Log data (7 days)
    └── alertmanager/             # Alert state
```

## Advantages of Modular Deployment

1. **Start Simple**: Begin with just metrics and visualization
2. **Lower Resource Usage**: Only deploy what you need
3. **Easier Troubleshooting**: Add one component at a time
4. **Incremental Learning**: Understand each component before adding more
5. **Flexible Secrets**: Only need relevant secrets for each phase

## Resource Usage by Phase

**Phase 1 (Basic):**
- RAM: ~400MB
- Disk: 1-2GB (7 days metrics)

**Phase 2 (+ Logging):**
- RAM: +150MB
- Disk: +500MB-1GB (7 days logs)

**Phase 3 (+ Alerting):**
- RAM: +50MB
- Disk: Minimal

**Phase 4 (+ Exporters):**
- RAM: +40MB
- Disk: Minimal

**Total (All Phases): ~640MB RAM, ~2-3GB disk**

## When to Use Which Approach

### Use Modular Deployment When:
- Learning observability stack for the first time
- Limited resources (small homelab)
- Want to understand each component
- Only need basic monitoring initially
- Don't want to configure alerts yet

### Use Complete Deployment When:
- Want full observability immediately
- Have all secrets configured
- Know exactly what you need
- Running production services
- Need comprehensive monitoring from day one

## Common Workflows

### Workflow 1: Start Small, Add Features
```bash
# Week 1: Basic monitoring
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_basic.yaml

# Week 2: Add logging
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_logging.yaml

# Week 3: Add alerts
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_alerts.yaml

# Week 4: Add database monitoring
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_exporters.yaml
```

### Workflow 2: Basic + Alerts (Skip Logging)
```bash
# Just metrics and alerts, no log aggregation
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_basic.yaml
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_alerts.yaml
```

### Workflow 3: Everything at Once
```bash
# All features in one go
ansible-playbook playbooks/apps/monitoring/deploy_monitoring.yaml
```

## Troubleshooting

### Phase Dependencies
Each playbook checks if basic monitoring is deployed first. If you see:
```
TASK [Fail if basic monitoring not deployed]
fatal: [traefik]: FAILED! => {"msg": "Basic monitoring stack not found..."}
```
Run `deploy_monitoring_basic.yaml` first.

### Missing Secrets
Playbooks validate required secrets before deployment. If you see:
```
TASK [Check required secrets]
fatal: [traefik]: FAILED! => {"msg": "Missing required secrets..."}
```
Add the required secrets to `vars/secrets.yml` (see phase documentation above).

### Idempotency
All playbooks are idempotent and can be re-run safely. They:
- Check if services already exist before adding them
- Update configurations without duplicating entries
- Only restart services when configurations change

## Maintenance

### Updating Components
Re-run any playbook to update its components:
```bash
# Update basic monitoring (Prometheus, Grafana versions)
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_basic.yaml

# Update alert rules
ansible-playbook playbooks/apps/monitoring/deploy_monitoring_alerts.yaml
```

### Removing Components
To remove logging or alerting:
1. Edit `/mnt/zfs/docker/monitoring/compose.yaml`
2. Remove the service definitions
3. Run: `docker compose up -d` (removes deleted services)

## See Also

- `README.md` - Complete monitoring stack documentation
- `playbooks/apps/monitoring/deploy_monitoring.yaml` - All-in-one deployment
- `docs/recovery-guide.md` - Backup and recovery procedures
