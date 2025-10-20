# Traefik Reverse Proxy Deployment

This directory contains the Ansible playbook and configuration templates for deploying Traefik reverse proxy with Docker Compose.

## Directory Structure

```
playbooks/apps/traefik/
├── deploy_traefik.yaml    # Main Ansible playbook
├── compose.yaml.j2         # Docker Compose configuration template
├── traefik.yaml.j2         # Traefik static configuration template
├── .env.j2                 # Environment variables template
└── README.md               # This file
```

## Configuration Files

### `traefik.yaml.j2`
Traefik static configuration that defines:
- Entry points (HTTP, HTTPS)
- Let's Encrypt certificate resolver with Cloudflare DNS challenge
- Docker provider settings
- Logging configuration
- API dashboard settings

**Note:** To use Let's Encrypt staging server for testing, uncomment the staging CA line in the template.

### `compose.yaml.j2`
Docker Compose file that configures:
- Traefik container with v3.5.3 image
- Port mappings (80, 443)
- Prometheus metrics endpoint (8082)
- Volume mounts for configuration and logs
- Docker networks (proxy, tunnel, monitoring)
- Traefik dashboard labels and routing

**Templated variables:**
- `{{ traefik_config_dir }}` - Path to Traefik configuration directory
- `{{ traefik_log_dir }}` - Path to Traefik log directory

### `.env.j2`
Environment variables used by Docker Compose:
- `CF_DNS_API_TOKEN` - Cloudflare API token for DNS challenges
- `TRAEFIK_DASHBOARD_CREDENTIALS` - HTTP basic auth credentials
- `DOMAIN` - Primary domain name

## Deployment

### Prerequisites

1. **Required variables in `vars/secrets.yml`:**
   ```yaml
   cf_dns_api_token: "your-cloudflare-api-token"
   traefik_dashboard_credentials: "admin:$$apr1$$hash..."  # htpasswd format
   domain: "yourdomain.com"
   ```

2. **Generate basic auth credentials:**
   ```bash
   # Install apache2-utils
   apt install apache2-utils

   # Generate password hash
   htpasswd -nb admin your_password
   # Output: admin:$apr1$...

   # Important: Escape $ signs by doubling them in secrets.yml
   # Example: admin:$$apr1$$hash...
   ```

3. **ZFS dataset:** Traefik data should be on ZFS dataset `/mnt/traefik`

### Deploy Traefik

```bash
# From repository root
ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
```

### What the Playbook Does

1. **Setup Docker:**
   - Ensures Docker service is running
   - Creates Docker networks: `proxy`, `tunnel`, `monitoring`

2. **Create directory structure:**
   - `/mnt/traefik/config/` - Configuration files
   - `/mnt/traefik/logs/` - Access and error logs

3. **Deploy configuration files:**
   - `traefik.yaml` - Static configuration
   - `acme.json` - Let's Encrypt certificates (created with 0600 permissions)
   - `.env` - Environment variables
   - `compose.yaml` - Docker Compose configuration

4. **Deploy container:**
   - Pulls Traefik image (if not present)
   - Starts Traefik container
   - Configures automatic restarts

5. **Idempotent updates:**
   - Tracks changes to `traefik.yaml`, `compose.yaml`, and `.env`
   - Automatically recreates container when configuration changes

## Monitoring Integration

Traefik is configured to export Prometheus metrics on port 8082:
- `--metrics.prometheus=true`
- `--metrics.prometheus.addEntryPointsLabels=true`
- `--metrics.prometheus.addServicesLabels=true`
- `--entryPoints.metrics.address=:8082`

These metrics are scraped by Prometheus when the monitoring stack is deployed.

## Accessing Traefik

### Dashboard
- URL: `https://traefik.yourdomain.com`
- Authentication: HTTP Basic Auth (configured in secrets.yml)
- Features: View routers, services, middlewares, and certificate status

### Logs
- Access log: `/mnt/traefik/logs/access.log`
- Error log: `/mnt/traefik/logs/traefik.log`

```bash
# View live logs
docker logs -f traefik

# View access log
tail -f /mnt/traefik/logs/access.log
```

## Customization

### Change Traefik Version

Edit `compose.yaml.j2`:
```yaml
image: traefik:v3.5.3  # Change version here
```

### Add Custom Middlewares

Add to `traefik.yaml.j2` or use Docker labels in service deployments.

### Use Let's Encrypt Staging

For testing, edit `traefik.yaml.j2`:
```yaml
certificatesResolvers:
  cloudflare:
    acme:
      caServer: "https://acme-staging-v02.api.letsencrypt.org/directory"
```

## Troubleshooting

### Certificate Not Generated

1. Check Cloudflare API token has correct permissions:
   - Zone:DNS:Edit
   - Zone:Zone:Read

2. Check Traefik logs:
   ```bash
   docker logs traefik
   ```

3. Verify DNS propagation:
   ```bash
   dig @1.1.1.1 yourdomain.com
   ```

### Dashboard Not Accessible

1. Verify basic auth credentials:
   ```bash
   # Test credentials
   curl -u admin:password https://traefik.yourdomain.com
   ```

2. Check DNS resolves to correct IP:
   ```bash
   nslookup traefik.yourdomain.com
   ```

### Metrics Not Available

1. Check Prometheus can reach Traefik:
   ```bash
   # From monitoring container
   curl http://traefik:8082/metrics
   ```

2. Verify Traefik is on monitoring network:
   ```bash
   docker network inspect monitoring
   ```

## Updating Configuration

After modifying any template file, redeploy:

```bash
ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
```

The playbook will:
1. Detect configuration changes
2. Update files on target host
3. Recreate Traefik container with new configuration

## Networks

Traefik connects to three Docker networks:

- **proxy**: Main network for routing traffic to application containers
- **tunnel**: For Cloudflare Tunnel integration
- **monitoring**: For Prometheus metrics collection

Applications that need to be accessible via Traefik should be on the `proxy` network.

## Security Considerations

1. **Dashboard access:** Protected by HTTP basic auth
2. **API token:** Stored in `.env` with 0600 permissions
3. **Certificates:** Stored in `acme.json` with 0600 permissions
4. **Container security:** Runs with `no-new-privileges:true`

## Related Documentation

- [Traefik Documentation](https://doc.traefik.io/traefik/)
- [Cloudflare DNS Challenge](https://doc.traefik.io/traefik/https/acme/#dnschallenge)
- [Docker Provider](https://doc.traefik.io/traefik/providers/docker/)
- Main project documentation: `../../CLAUDE.md`
