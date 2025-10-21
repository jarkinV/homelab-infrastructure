# Paperless-ngx Deployment

This directory contains the Ansible playbook and configuration templates for deploying Paperless-ngx with Docker Compose.

## Overview

[Paperless-ngx](https://docs.paperless-ngx.com/) is a document management system that transforms your physical documents into a searchable online archive. It features OCR, full-text search, tagging, and automatic organization.

## Directory Structure

```
playbooks/apps/paperless/
├── deploy_paperless.yaml    # Main Ansible playbook
├── compose.yaml.j2           # Docker Compose configuration template
├── .env.j2                   # Environment variables template
└── README.md                 # This file
```

## Configuration Files

### `.env.j2`
Environment variables used by Paperless and Docker Compose:
- `DOMAIN` - Primary domain name
- `POSTGRES_PASSWORD` - PostgreSQL database password
- `PAPERLESS_DBPASS` - Paperless database password (same as PostgreSQL)
- `PAPERLESS_REDIS` - Redis connection string
- `PAPERLESS_DBHOST` - PostgreSQL hostname
- `PAPERLESS_URL` - Public URL for Paperless
- `PAPERLESS_TIME_ZONE` - Timezone for timestamps
- `PAPERLESS_OCR_LANGUAGE` - Primary OCR language (eng)
- `PAPERLESS_OCR_LANGUAGES` - Additional OCR languages (pol)
- `PAPERLESS_FILENAME_FORMAT` - File naming pattern for exports

### `compose.yaml.j2`
Docker Compose file that configures:
- **Redis (broker)**: Message queue and caching
- **PostgreSQL (db)**: Database for metadata
- **Paperless webserver**: Main application

**Templated variables:**
- `{{ paperless_base_dir }}` - Base directory for Paperless (/mnt/paperless)

## Architecture

```
┌─────────────────────────────────────────┐
│        Paperless-ngx Stack              │
├─────────────────────────────────────────┤
│                                          │
│  paperless-web (Port 8000)              │
│       ↓              ↓                   │
│   PostgreSQL      Redis                 │
│   (metadata)     (cache/queue)          │
│                                          │
│  Volumes:                                │
│  - /config/web/data (app data)          │
│  - /media (documents)                   │
│  - /export (exported files)             │
│  - /consume (inbox for new docs)        │
└─────────────────────────────────────────┘
          ↓
     Traefik Proxy
          ↓
  https://paperless.domain.com
```

## Deployment

### Prerequisites

1. **Required variables in `vars/secrets.yml`:**
   ```yaml
   domain: "yourdomain.com"
   paperless_postgres_password: "secure-database-password"
   ```

2. **ZFS dataset:** Paperless data should be on ZFS dataset `/mnt/paperless`
   ```bash
   # Ensure paperless dataset is created
   ansible-playbook playbooks/zfs/configure_zfs_storage.yaml

   # Attach to Traefik LXC (container 100)
   ansible-playbook playbooks/zfs/attach_zfs_datasets.yaml -e "ct_id=100"
   ```

3. **Traefik:** Must be deployed and running for HTTPS routing
   ```bash
   ansible-playbook playbooks/apps/traefik/deploy_traefik.yaml
   ```

### Deploy Paperless-ngx

```bash
# From repository root
ansible-playbook playbooks/apps/paperless/deploy_paperless.yaml
```

### What the Playbook Does

1. **Setup Docker:**
   - Ensures Docker service is running
   - Creates Docker networks: `proxy`, `paperless-network`

2. **Create directory structure:**
   - `/mnt/paperless/config/redis` - Redis data
   - `/mnt/paperless/config/postgres` - PostgreSQL data
   - `/mnt/paperless/config/web/data` - Application data
   - `/mnt/paperless/media` - Document storage
   - `/mnt/paperless/export` - Exported documents
   - `/mnt/paperless/consume` - Inbox for new documents

3. **Deploy configuration files:**
   - `.env` - Environment variables
   - `compose.yaml` - Docker Compose configuration

4. **Deploy containers:**
   - Redis (message broker)
   - PostgreSQL (database)
   - Paperless webserver

5. **Idempotent updates:**
   - Tracks changes to `compose.yaml` and `.env`
   - Automatically recreates containers when configuration changes

## Accessing Paperless-ngx

### Web Interface
- URL: `https://paperless.yourdomain.com`
- First-time setup:
  1. Create admin user (CLI command below)
  2. Login with admin credentials
  3. Configure settings (OCR, storage paths, etc.)

### Create Admin User

```bash
# Create superuser account
docker exec -it paperless-web python3 manage.py createsuperuser

# Follow prompts to set username, email, and password
```

### Data Locations
- **Documents:** `/mnt/paperless/media/documents/`
- **Thumbnails:** `/mnt/paperless/media/thumbnails/`
- **Database:** `/mnt/paperless/config/postgres/`
- **App data:** `/mnt/paperless/config/web/data/`
- **Inbox:** `/mnt/paperless/consume/` (place new documents here)
- **Export:** `/mnt/paperless/export/` (exported documents)

### Logs
```bash
# View Paperless logs
docker logs -f paperless-web

# View PostgreSQL logs
docker logs paperless-postgres

# View Redis logs
docker logs paperless-redis
```

## Features

### Document Management
- **OCR:** Automatic text extraction from scanned documents
- **Full-text search:** Find documents by content
- **Tagging:** Organize with tags and correspondents
- **Automatic classification:** ML-based document sorting
- **Custom fields:** Add metadata to documents

### Workflows
- **Consume folder:** Drop files to automatically import
- **Email import:** Forward emails to import attachments
- **API:** Programmatic document management
- **Bulk operations:** Edit multiple documents at once

### Supported Formats
- **Documents:** PDF, TXT, MD, Office files (via conversion)
- **Images:** JPG, PNG, TIFF, WebP
- **Archives:** ZIP (extracts and imports contents)

## Usage Tips

### Adding Documents

1. **Via Web Upload:**
   - Navigate to web interface
   - Click "Upload" button
   - Select files

2. **Via Consume Folder:**
   ```bash
   # Copy documents to consume folder
   cp ~/documents/*.pdf /mnt/paperless/consume/

   # Paperless automatically processes files
   ```

3. **Via Email:**
   - Configure email settings in Paperless
   - Forward emails to configured address
   - Attachments automatically imported

### Organizing Documents

1. **Tags:** Create tags for categories (receipts, invoices, personal)
2. **Correspondents:** Link documents to people/companies
3. **Document types:** Classify by type (letter, contract, receipt)
4. **Custom fields:** Add metadata (invoice number, date range)

### Searching

- **Full-text:** Search document content
- **Tags:** Filter by tags
- **Date range:** Find documents by date
- **Correspondent:** Filter by sender/recipient
- **Combined:** Use multiple filters together

## Customization

### Change OCR Languages

Edit `.env.j2`:
```bash
PAPERLESS_OCR_LANGUAGE=eng         # Primary language
PAPERLESS_OCR_LANGUAGES=pol+deu    # Additional languages (Polish + German)
```

Available languages: `eng`, `deu`, `fra`, `spa`, `ita`, `pol`, `nld`, `por`, `rus`, etc.

### Change Timezone

Edit `.env.j2`:
```bash
PAPERLESS_TIME_ZONE=America/New_York  # Use your timezone
```

### Change Filename Format

Edit `.env.j2`:
```bash
PAPERLESS_FILENAME_FORMAT={created_year}/{correspondent}/{title}
```

Variables: `{created}`, `{created_year}`, `{created_month}`, `{correspondent}`, `{title}`, `{tags}`, `{document_type}`

### Change Paperless Version

Edit `compose.yaml.j2`:
```yaml
image: ghcr.io/paperless-ngx/paperless-ngx:2.18  # Change version
```

## Backup and Recovery

### Backup Strategy

Paperless data consists of:
1. **PostgreSQL database** - Document metadata
2. **Media files** - Actual documents
3. **Redis data** - Cache (optional to backup)

### Automated ZFS Snapshots

```bash
# Manual snapshot
zfs snapshot tank/paperless@backup-$(date +%Y%m%d)

# List snapshots
zfs list -t snapshot tank/paperless

# Automated via Sanoid (if configured)
# See docs/sanoid.conf
```

### Manual Export

```bash
# Stop containers
docker stop paperless-web paperless-postgres paperless-redis

# Backup PostgreSQL database
docker exec paperless-postgres pg_dump -U paperless paperless > paperless-db-$(date +%Y%m%d).sql

# Backup media files
tar -czf paperless-media-$(date +%Y%m%d).tar.gz /mnt/paperless/media/

# Backup config
tar -czf paperless-config-$(date +%Y%m%d).tar.gz /mnt/paperless/config/

# Start containers
docker start paperless-redis paperless-postgres paperless-web
```

### Restore from Snapshot

```bash
# List snapshots
zfs list -t snapshot tank/paperless

# Rollback to snapshot
docker stop paperless-web paperless-postgres paperless-redis
zfs rollback tank/paperless@backup-20250120
docker start paperless-redis paperless-postgres paperless-web
```

### Restore from Manual Backup

```bash
# Stop containers
docker stop paperless-web paperless-postgres paperless-redis

# Restore database
cat paperless-db-20250120.sql | docker exec -i paperless-postgres psql -U paperless paperless

# Restore media files
tar -xzf paperless-media-20250120.tar.gz -C /

# Restore config
tar -xzf paperless-config-20250120.tar.gz -C /

# Start containers
docker start paperless-redis paperless-postgres paperless-web
```

## Troubleshooting

### Container Won't Start

1. Check all container logs:
   ```bash
   docker logs paperless-web
   docker logs paperless-postgres
   docker logs paperless-redis
   ```

2. Verify database is ready:
   ```bash
   docker exec paperless-postgres pg_isready -U paperless
   ```

3. Check permissions:
   ```bash
   ls -la /mnt/paperless/
   ```

### OCR Not Working

1. Check OCR languages are installed:
   ```bash
   docker exec paperless-web ls /usr/share/tesseract-ocr/5/tessdata/
   ```

2. Test OCR manually:
   ```bash
   docker exec paperless-web ocrmypdf --version
   ```

3. Check document format is supported:
   - PDFs: Native support
   - Images: JPG, PNG, TIFF supported
   - Office docs: Converted to PDF first

### Documents Not Consuming

1. Check consume folder permissions:
   ```bash
   ls -la /mnt/paperless/consume/
   ```

2. Check Paperless logs:
   ```bash
   docker logs -f paperless-web | grep consume
   ```

3. Manually trigger consumption:
   ```bash
   docker exec paperless-web document_consumer
   ```

### Slow Performance

1. Check container resources:
   ```bash
   docker stats paperless-web paperless-postgres
   ```

2. Check PostgreSQL performance:
   ```bash
   docker exec paperless-postgres psql -U paperless -c "SELECT count(*) FROM documents_document;"
   ```

3. Optimize database:
   ```bash
   docker exec paperless-web python3 manage.py document_index reindex
   ```

### Database Issues

1. Check PostgreSQL logs:
   ```bash
   docker logs paperless-postgres
   ```

2. Connect to database:
   ```bash
   docker exec -it paperless-postgres psql -U paperless
   ```

3. Verify database integrity:
   ```bash
   docker exec paperless-postgres pg_dump -U paperless paperless > /dev/null
   ```

## Updating

### Update Paperless Version

1. Edit `compose.yaml.j2` with new version
2. Redeploy:
   ```bash
   ansible-playbook playbooks/apps/paperless/deploy_paperless.yaml
   ```

The playbook will:
1. Detect compose file change
2. Pull new images
3. Run database migrations automatically
4. Recreate containers with new version

### Manual Update

```bash
# Pull new images
docker compose -f /mnt/paperless/compose.yaml pull

# Stop and remove containers
docker compose -f /mnt/paperless/compose.yaml down

# Redeploy via playbook
ansible-playbook playbooks/apps/paperless/deploy_paperless.yaml
```

## Advanced Configuration

### Email Integration

1. Configure email in Paperless settings
2. Set up mail server (SMTP)
3. Create import rules
4. Forward emails to Paperless

### API Access

Paperless provides a REST API:

```bash
# Get API token from web interface
# Settings > API Tokens

# Example: List documents
curl -H "Authorization: Token YOUR_TOKEN" \
  https://paperless.yourdomain.com/api/documents/

# Upload document
curl -H "Authorization: Token YOUR_TOKEN" \
  -F "document=@file.pdf" \
  https://paperless.yourdomain.com/api/documents/post_document/
```

See [API documentation](https://docs.paperless-ngx.com/api/) for details.

### Mobile Apps

Paperless has companion mobile apps:
- **Paperless Mobile** (iOS/Android): Scan and upload on the go
- **Configure URL:** `https://paperless.yourdomain.com`
- **Authentication:** Use API token or username/password

### Custom Processing

Add custom post-consumption scripts:

1. Create script in container
2. Configure in Paperless settings
3. Scripts run after document processing

## Monitoring Integration

If the monitoring stack is deployed, Paperless metrics are available:

- **Container metrics:** CPU, memory, network via cAdvisor
- **PostgreSQL metrics:** Exported via postgres-exporter
- **Redis metrics:** Exported via redis-exporter
- **Application logs:** Available via Loki
- **Alerts:** Container down, database issues

Access via Grafana: `https://grafana.yourdomain.com`

Metrics include:
- Document processing rate
- Database query performance
- Storage usage
- OCR processing time

## Security Considerations

1. **HTTPS only:** All traffic encrypted via Traefik
2. **Network isolation:** Containers on isolated network
3. **Database security:** PostgreSQL not exposed externally
4. **Authentication:** Required for web access
5. **API tokens:** Use tokens instead of passwords for API
6. **Container security:** Runs with `no-new-privileges:true`
7. **Regular backups:** ZFS snapshots + manual exports

## Performance Tuning

### PostgreSQL Optimization

For large document libraries (>10,000 docs):

```bash
# Connect to database
docker exec -it paperless-postgres psql -U paperless

# Analyze tables
ANALYZE;

# Reindex
REINDEX DATABASE paperless;
```

### Redis Memory

Increase Redis memory limit if needed (edit `compose.yaml.j2`):
```yaml
broker:
  command: redis-server --maxmemory 512mb --maxmemory-policy allkeys-lru
```

### OCR Performance

- Use `PAPERLESS_OCR_MODE=skip` for documents that don't need OCR
- Enable `PAPERLESS_OCR_SKIP_ARCHIVE_FILE=always` for pre-OCR'd PDFs
- Use `PAPERLESS_OCR_PAGES=1` to only OCR first page

## Related Documentation

- [Paperless-ngx Official Docs](https://docs.paperless-ngx.com/)
- [Paperless-ngx GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [Docker Hub](https://hub.docker.com/r/paperlessngx/paperless-ngx)
- [API Documentation](https://docs.paperless-ngx.com/api/)
- Main project documentation: `../../../CLAUDE.md`

## Support

- **Community:** [Reddit r/paperless](https://reddit.com/r/paperless)
- **GitHub Discussions:** [Discussions](https://github.com/paperless-ngx/paperless-ngx/discussions)
- **GitHub Issues:** [Report bugs](https://github.com/paperless-ngx/paperless-ngx/issues)
- **Documentation:** [Official docs](https://docs.paperless-ngx.com/)
