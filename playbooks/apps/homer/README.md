# Homer Dashboard Deployment

This playbook deploys [Homer](https://github.com/bastienwirtz/homer), a dead simple static homepage for your server to keep your services on hand, from a simple YAML configuration file.

## Features

- **Simple Configuration**: YAML-based configuration
- **Fast & Lightweight**: Static page, no backend required
- **Beautiful UI**: Based on Bulma CSS framework
- **Responsive**: Works great on mobile devices
- **Dark Mode**: Built-in dark/light theme support
- **Search**: Quick service search functionality
- **Icons**: FontAwesome & custom logo support
- **No Authentication**: Simple static page (use Traefik auth if needed)

## Prerequisites

- LXC container in the internal network (NAT) with Docker installed
- Traefik reverse proxy deployed and configured
- Domain configured in `vars/secrets.yml`
- ZFS dataset `docker/homer` created (optional but recommended)

## ZFS Dataset Setup (Recommended)

Create a dedicated ZFS dataset for Homer data:

```bash
# On Proxmox host
zfs create rpool/docker/homer
zfs set mountpoint=/mnt/homer rpool/docker/homer

# Attach to LXC container (e.g., container 100)
ansible-playbook playbooks/zfs/attach_zfs_datasets.yaml -e "ct_id=100"
```

## Deployment

Deploy Homer using the playbook:

```bash
ansible-playbook playbooks/apps/homer/deploy_homer.yaml
```

The playbook will:
1. Create necessary directories
2. Generate configuration files from templates
3. Deploy Homer using Docker Compose
4. Configure Traefik labels for HTTPS access

## Access

After deployment, access Homer at: **https://homer.{{ domain }}**

## Configuration

### Main Configuration File

The configuration is stored in `/mnt/homer/assets/config.yml`. This file is generated from the Jinja2 template during deployment.

To customize your dashboard:

1. Edit the template: `playbooks/apps/homer/config.yml.j2`
2. Or manually edit: `/mnt/homer/assets/config.yml` on the container
3. Changes to the file are reflected immediately (no container restart needed)

### Configuration Structure

```yaml
title: "Homer Dashboard"
subtitle: "Services"

# Optional theme customization
theme: default
colors:
  light:
    highlight-primary: "#3367d6"
    # ... more colors
  dark:
    highlight-primary: "#3367d6"
    # ... more colors

# Services
services:
  - name: "Group Name"
    icon: "fas fa-cloud"
    items:
      - name: "Service Name"
        logo: "https://url-to-logo.png"
        subtitle: "Description"
        tag: "category"
        url: "https://service.domain.com"
        target: "_blank"
```

### Adding New Services

Add services to the `services` section in the config file:

```yaml
services:
  - name: "My Apps"
    icon: "fas fa-server"
    items:
      - name: "My Service"
        logo: "assets/tools/sample.png"  # or URL
        subtitle: "Service description"
        tag: "app"
        url: "https://myservice.domain.com"
        target: "_blank"
```

### Themes

Homer supports several built-in themes. Change the `theme` value in config.yml:

- `default` - Classic Homer theme
- `sui` - Semantic UI inspired theme

You can also customize colors directly in the config file.

### Custom Icons & Logos

1. **FontAwesome Icons**: Use any FontAwesome icon
   ```yaml
   icon: "fas fa-cloud"
   ```

2. **Custom Logos**: Place images in `/mnt/homer/assets/` and reference them
   ```yaml
   logo: "assets/my-logo.png"
   ```

3. **External URLs**: Use direct image URLs
   ```yaml
   logo: "https://example.com/logo.png"
   ```

### Message Banner

Display a welcome message or announcement:

```yaml
message:
  style: "is-dark"  # is-dark, is-warning, is-info, is-success
  title: "Welcome!"
  icon: "fa fa-grin"
  content: "This is your Homer dashboard."
```

## File Structure

```
/mnt/homer/
├── .env                    # Environment variables (generated)
├── compose.yaml            # Docker Compose configuration (generated)
└── assets/
    ├── config.yml          # Main configuration file (generated)
    └── (custom images)     # Optional custom logos/icons
```

## Docker Compose Configuration

The deployment creates a single container:

- **homer**: Main Homer application
  - Image: `b4bz/homer:latest`
  - Port: 8080 (internal, accessed via Traefik)
  - Volume: `/mnt/homer/assets` mounted to `/www/assets`
  - Environment: `INIT_ASSETS=0` (don't overwrite config on restart)

## Idempotency

The playbook is fully idempotent:
- Re-running the playbook will update configuration if changed
- Container is recreated only when `.env`, `compose.yaml`, or `config.yml` changes
- Safe to run multiple times

## Updating Configuration

To update Homer configuration:

```bash
# Method 1: Edit template and re-run playbook (recommended)
vim playbooks/apps/homer/config.yml.j2
ansible-playbook playbooks/apps/homer/deploy_homer.yaml

# Method 2: Edit config directly on container (temporary)
vim /mnt/homer/assets/config.yml
# No restart needed - changes reflected immediately
```

## Troubleshooting

### Check Container Status

```bash
# On the LXC container
cd /mnt/homer
docker compose ps
docker compose logs homer
```

### Verify Configuration

```bash
# Check if config file exists
cat /mnt/homer/assets/config.yml

# Check file permissions
ls -la /mnt/homer/assets/
```

### Access Issues

If you can't access Homer via browser:

1. Check Traefik dashboard for route configuration
2. Verify DNS resolution: `nslookup homer.{{ domain }}`
3. Check container logs: `docker compose logs homer`
4. Verify Traefik labels: `docker inspect homer`

### Configuration Not Updating

Homer uses static file serving. If changes don't appear:

1. Clear browser cache (Ctrl+Shift+R)
2. Check file was actually modified: `cat /mnt/homer/assets/config.yml`
3. Verify file permissions allow container to read it

### Icons Not Loading

If FontAwesome icons don't load:

1. Check internet connectivity from container
2. Verify CDN is accessible
3. Use alternative icon format or custom images

## Customization Examples

### Minimal Dashboard

```yaml
title: "Services"
subtitle: ""
header: true
footer: false

services:
  - name: "Apps"
    items:
      - name: "Service 1"
        url: "https://app1.domain.com"
      - name: "Service 2"
        url: "https://app2.domain.com"
```

### With Status Checks

Homer doesn't have built-in health checks, but you can use tag colors to indicate status:

```yaml
- name: "Service"
  tag: "production"
  tagstyle: "is-success"  # is-success, is-warning, is-danger, is-info
```

### Multiple Columns

Homer automatically arranges items in columns based on screen size. Control this with CSS classes if needed.

## Backup

Homer configuration is simple to backup:

```bash
# Backup configuration
cp /mnt/homer/assets/config.yml /backup/homer-config-$(date +%F).yml

# If using ZFS, snapshot the dataset
zfs snapshot rpool/docker/homer@backup-$(date +%F)
```

## Restore

```bash
# Restore from backup file
cp /backup/homer-config-YYYY-MM-DD.yml /mnt/homer/assets/config.yml

# Or restore from ZFS snapshot
zfs rollback rpool/docker/homer@backup-YYYY-MM-DD
```

## Comparison with Dashy

| Feature | Homer | Dashy |
|---------|-------|-------|
| Configuration | YAML | YAML |
| Backend | Static | Node.js |
| Resource Usage | Very Low (~10MB) | Low (~50MB) |
| Features | Basic | Advanced |
| Speed | Very Fast | Fast |
| Themes | 2 built-in | 10+ themes |
| Status Checks | Manual tags | Built-in |
| Search | Basic | Advanced |
| Widgets | No | Yes |
| Auth | No (use Traefik) | Optional |

**Use Homer if**: You want a simple, fast, static dashboard
**Use Dashy if**: You need advanced features, widgets, and built-in auth

## Uninstallation

To remove Homer:

```bash
# Stop and remove container
cd /mnt/homer
docker compose down

# Remove data (optional)
rm -rf /mnt/homer

# Remove ZFS dataset (optional)
zfs destroy rpool/docker/homer
```

## Resources

- **Official Repository**: https://github.com/bastienwirtz/homer
- **Documentation**: https://github.com/bastienwirtz/homer/blob/main/docs/configuration.md
- **Demo**: https://homer-demo.netlify.app/
- **Icon Library**: https://fontawesome.com/icons
- **Dashboard Icons**: https://github.com/walkxcode/dashboard-icons

## Support

For issues specific to:
- **Homer application**: Check the [official repository](https://github.com/bastienwirtz/homer/issues)
- **This deployment**: Check Ansible playbook logs and container logs
- **Traefik routing**: Check Traefik dashboard and logs