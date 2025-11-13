# Dashy Deployment

Dashy is a lightweight, self-hosted personal dashboard application. It provides a customizable homepage with quick links to all your services and applications.

## Deployment

```bash
ansible-playbook playbooks/apps/dashy/deploy_dashy.yaml
```

## Access

- **URL**: `https://dash.{{ domain }}`
- **Default Theme**: Colorful
- **Port**: 80 (behind Traefik)

## Directory Structure

After deployment, the following directories are created:

```
/mnt/dashy/
├── .env                 # Environment variables
├── compose.yaml         # Docker Compose configuration
└── config/
    └── conf.yml         # Dashy configuration (main customization file)
```

## Configuration

Dashy configuration is stored in `/mnt/dashy/config/conf.yml`. Edit this file to:

### Add New Service Links

To add a new service to your dashboard, add a new item under a section:

```yaml
- title: My Service
  description: Service Description
  icon: fas fa-icon-name
  url: https://service.{{ domain }}
```

### Available Icon Families

- **FontAwesome 6**: `fas fa-*` (default)
- **Monochrome icons**: `mono *`
- **Emoji**: Use emoji directly

Find icons at: [FontAwesome Icon Search](https://fontawesome.com/search)

### Customize Sections

Each section can have its own layout settings:

```yaml
sections:
  - name: My Section
    displayData:
      rows: 2          # Number of rows
      cols: 3          # Number of columns
      gap: 1rem        # Space between items
    items: []
```

### Available Themes

Set the theme in `appConfig`:

- `colorful` (default) - Bright, vibrant colors
- `darkly` - Dark background with light text
- `nord` - Arctic-inspired palette
- `dracula` - Dark with purple accent
- `gruvbox` - Warm retro colors
- `material` - Material design colors

Example:

```yaml
appConfig:
  theme: dracula
```

### Layout Options

Customize the default layout in `appConfig`:

```yaml
appConfig:
  layout: auto          # auto, grid, list, or hex
  iconSize: medium      # small, medium, large
  language: en          # en, de, fr, es, etc.
  showSplashScreen: false
```

## Customization Examples

### Add Authentication

To enable basic authentication:

```yaml
appConfig:
  auth:
    enableAuth: true
    users:
      - user: admin
        pass: your-secure-password
```

**Note**: Password will be visible in the config file. Use environment variables instead for production.

### Enable Web Search

To enable the built-in web search feature:

```yaml
appConfig:
  webSearch:
    disableWebSearch: false
    searchEngine: duckduckgo  # duckduckgo, google, bing
```

### Custom Color Scheme

Create a custom color scheme in `appConfig`:

```yaml
appConfig:
  customColors:
    primary: '#7d5ba6'
    secondary: '#c43a5f'
```

## After Editing Configuration

After making changes to `/mnt/dashy/config/conf.yml`:

1. The Docker container will automatically reload the configuration
2. No container restart is needed
3. Refresh your browser to see the changes

To manually recreate the container:

```bash
cd /mnt/dashy
docker compose down
docker compose up -d
```

## Backup and Recovery

Dashy stores all configuration in `conf.yml`. To backup:

```bash
cp /mnt/dashy/config/conf.yml /path/to/backup/conf.yml
```

The configuration is also stored on your ZFS dataset for automatic snapshots.

## Full Documentation

For complete configuration options, see: [Dashy Documentation](https://dashy.to/docs/configuration)

## Integration with Other Services

Dashy automatically discovers services running on your infrastructure. Add links for:

- Traefik Dashboard
- Monitoring services (Grafana, Prometheus)
- Media services (Immich, Paperless)
- Automation services (n8n)
- Finance apps (Actual Budget)
- Custom internal tools

## Troubleshooting

### Configuration Changes Not Applied

1. Check the configuration file syntax (it's YAML)
2. Look at container logs: `docker logs dashy`
3. Restart the container: `cd /mnt/dashy && docker compose restart`

### Icons Not Loading

1. Verify icon names at [FontAwesome Search](https://fontawesome.com/search)
2. Use the correct prefix (`fas`, `far`, `fab`, etc.)
3. Example: `fas fa-home`, `fab fa-github`

### Theme Not Applied

1. Clear browser cache
2. Try a different browser
3. Check `appConfig.theme` is set correctly

## Performance

- Dashy is lightweight (~50MB image)
- Typically uses <100MB RAM
- Fast load times with static content
- No database required