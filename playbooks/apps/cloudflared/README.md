# Cloudflare Tunnel Deployment

This playbook deploys [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) (cloudflared) to expose internal services to the internet securely without opening firewall ports or exposing your home IP address.

## Overview

Cloudflare Tunnel creates an encrypted tunnel between your internal network and Cloudflare's edge network. This allows you to:

- **Expose internal services** (like your self-hosted apps) to the internet
- **No port forwarding required** - no need to open ports in your router
- **Hide your home IP** - services are accessed through Cloudflare's network
- **Free SSL/TLS** - automatic HTTPS for all exposed services
- **DDoS protection** - Cloudflare's network protects against attacks
- **Access control** - use Cloudflare Access for authentication

## Architecture

```
Internet Users
     ↓
Cloudflare Edge Network (DDoS protection, SSL, caching)
     ↓
Cloudflare Tunnel (encrypted tunnel)
     ↓
cloudflared container (10.0.0.X)
     ↓
Traefik proxy (10.0.0.10)
     ↓
Internal Docker services
```

## Prerequisites

1. **Cloudflare account** with a domain added to Cloudflare
2. **Cloudflare Zero Trust account** (free tier available)
3. **LXC container** in the NAT network (10.0.0.0/24)
4. **Traefik** or other internal reverse proxy (recommended)

## Setup Instructions

### 1. Create a Cloudflare Tunnel

1. Go to **Cloudflare Zero Trust Dashboard**: https://one.dash.cloudflare.com/
2. Navigate to **Networks** → **Tunnels**
3. Click **Create a tunnel**
4. Choose **Cloudflared** as the connector type
5. Give your tunnel a name (e.g., "homelab-tunnel")
6. Click **Save tunnel**
7. **Copy the tunnel token** - it looks like: `eyJhIjoiX...` (long base64 string)

### 2. Add Tunnel Token to Secrets

Add the tunnel token to `vars/secrets.yml`:

```yaml
# Cloudflare Tunnel
cloudflared_tunnel_token: "eyJhIjoiXXXXXXXXXXXXXXXXXXXXXXXX"
```

**Important**: Keep this token secure! It provides access to your tunnel.

### 3. Deploy Cloudflared

Run the deployment playbook:

```bash
ansible-playbook playbooks/apps/cloudflared/deploy_cloudflared.yaml
```

The playbook will:
- Create `/mnt/cloudflared/` directory
- Generate `.env` file with tunnel token (encrypted)
- Generate `compose.yaml` for cloudflared
- Start the cloudflared container
- Connect to Cloudflare's edge network

### 4. Configure Public Hostnames

After deployment, configure which services to expose:

1. Go back to **Cloudflare Zero Trust** → **Networks** → **Tunnels**
2. Click on your tunnel name
3. Go to **Public Hostname** tab
4. Click **Add a public hostname**

Example configurations:

**Expose Traefik Dashboard:**
- **Subdomain**: `traefik`
- **Domain**: `yourdomain.com`
- **Type**: `HTTP`
- **URL**: `10.0.0.10:80` (Traefik's HTTP port)

**Expose any service through Traefik:**
- **Subdomain**: `app` (e.g., paperless, jellyfin, grafana)
- **Domain**: `yourdomain.com`
- **Type**: `HTTP`
- **URL**: `10.0.0.10:80` (Traefik handles routing)

**Expose service directly (bypass Traefik):**
- **Subdomain**: `service`
- **Domain**: `yourdomain.com`
- **Type**: `HTTP`
- **URL**: `10.0.0.11:8080` (direct to service)

### 5. Verify Tunnel is Connected

Check tunnel status:

```bash
# On the LXC container
docker logs cloudflared

# Look for: "Connection <UUID> registered"
# Should see 4 connections registered (Cloudflare uses 4 for redundancy)
```

Test access from the internet:

```bash
curl https://traefik.yourdomain.com
```

## Deployment Details

### Docker Compose Stack

- **Container**: `cloudflared`
- **Image**: `cloudflare/cloudflared:latest`
- **Network**: `proxy` (connects to Traefik and other services)
- **Restart policy**: `unless-stopped`
- **Configuration**: Tunnel token in `.env` file

### Directory Structure

```
/mnt/cloudflared/
├── .env              # Tunnel token (sensitive, mode 0600)
└── compose.yaml      # Docker Compose configuration
```

### Resource Usage

- **Memory**: ~20-30MB RAM
- **CPU**: Minimal (only active during traffic)
- **Disk**: ~50MB for container image
- **Network**: Outbound only (to Cloudflare edge)

## Integration with Traefik

If you're using Traefik as your internal reverse proxy:

1. **Expose Traefik's HTTP port** (80) through Cloudflare Tunnel
2. **Point all subdomains** to Traefik (10.0.0.10:80)
3. **Let Traefik handle routing** based on hostnames
4. **Traefik provides SSL** termination internally

This way, you only need one Cloudflare Tunnel public hostname pointing to Traefik, and Traefik routes to all internal services based on the hostname.

Example Cloudflare Tunnel configuration:
- `*.yourdomain.com` → `HTTP://10.0.0.10:80` (Traefik)

Traefik then routes:
- `paperless.yourdomain.com` → paperless container
- `jellyfin.yourdomain.com` → jellyfin container
- `grafana.yourdomain.com` → grafana container
- etc.

## Security Considerations

### Tunnel Token Security

- **Never commit** the tunnel token to git
- **Store securely** in `vars/secrets.yml` (gitignored)
- **Rotate regularly** by creating a new tunnel and updating the token
- **Limit access** to the LXC container with the tunnel

### Access Control

Consider adding Cloudflare Access for authentication:

1. Go to **Cloudflare Zero Trust** → **Access** → **Applications**
2. Create an application for your domain
3. Add authentication policies (email, Google, GitHub, etc.)
4. Require authentication before reaching your services

This adds a login page before users can access your services.

### Rate Limiting

Enable rate limiting in Cloudflare:

1. Go to **Cloudflare Dashboard** → **Security** → **WAF**
2. Create rate limiting rules to prevent abuse
3. Example: Limit requests to 100/minute per IP

### DDoS Protection

Cloudflare automatically provides:
- Layer 3/4 DDoS protection (network layer)
- Layer 7 DDoS protection (application layer)
- Bot management and filtering

## Troubleshooting

### Tunnel Not Connecting

Check container logs:
```bash
docker logs cloudflared
```

Common issues:
- **Invalid token**: Verify token in `.env` file matches Cloudflare dashboard
- **Network issues**: Ensure container can reach Cloudflare (outbound HTTPS)
- **Firewall blocking**: Check if outbound traffic to Cloudflare is allowed

### Service Not Accessible

1. **Check tunnel status** in Cloudflare dashboard (should show "Healthy")
2. **Verify public hostname configuration** (subdomain, domain, URL)
3. **Test internal connectivity**:
   ```bash
   # From cloudflared container
   docker exec -it cloudflared sh
   wget -O- http://10.0.0.10:80  # Test Traefik
   ```
4. **Check DNS propagation**: May take a few minutes for DNS to update

### SSL/TLS Errors

Cloudflare Tunnel automatically handles SSL/TLS. If you see errors:

1. **Check Cloudflare SSL mode**: Should be "Full" or "Flexible" (not "Full (strict)")
2. **Disable HSTS** temporarily to test
3. **Check Traefik configuration** if using internal SSL

### High Memory Usage

Cloudflared is very lightweight (~20-30MB). If you see high memory:

1. **Check for memory leaks**: Restart container
   ```bash
   docker restart cloudflared
   ```
2. **Update to latest version**: Pull latest image
   ```bash
   cd /mnt/cloudflared
   docker compose pull
   docker compose up -d
   ```

## Updating Cloudflared

Update to the latest version:

```bash
ansible-playbook playbooks/apps/cloudflared/deploy_cloudflared.yaml
```

Or manually:

```bash
cd /mnt/cloudflared
docker compose pull
docker compose up -d
```

## Removing the Tunnel

1. **Stop and remove the container**:
   ```bash
   cd /mnt/cloudflared
   docker compose down
   ```

2. **Delete the tunnel in Cloudflare**:
   - Go to **Cloudflare Zero Trust** → **Networks** → **Tunnels**
   - Click on your tunnel → **Delete**

3. **Remove configuration**:
   ```bash
   rm -rf /mnt/cloudflared
   ```

4. **Remove from secrets.yml**:
   - Delete `cloudflared_tunnel_token` variable

## Alternative: Using with Multiple Tunnels

You can run multiple tunnels for different purposes:

1. **Create multiple tunnels** in Cloudflare (e.g., "homelab", "test")
2. **Deploy to different directories**:
   - Modify `cloudflared_base_dir` in playbook
   - Use different tokens
3. **Isolate services** by tunnel (production vs development)

## Comparison with Port Forwarding

| Feature | Cloudflare Tunnel | Port Forwarding |
|---------|------------------|-----------------|
| Expose home IP | ❌ No | ✅ Yes |
| Firewall changes | ❌ Not needed | ✅ Required |
| DDoS protection | ✅ Built-in | ❌ None |
| Free SSL/TLS | ✅ Yes | ❌ Need Let's Encrypt |
| Access control | ✅ Cloudflare Access | ❌ DIY |
| Setup complexity | ⚠️ Medium | ✅ Simple |
| Performance | ✅ Fast (Cloudflare edge) | ✅ Direct |

## Resources

- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/)
- [Cloudflare Access Documentation](https://developers.cloudflare.com/cloudflare-one/applications/)
- [cloudflared Docker Hub](https://hub.docker.com/r/cloudflare/cloudflared)

## Support

For issues:
1. Check container logs: `docker logs cloudflared`
2. Verify tunnel status in Cloudflare dashboard
3. Check Cloudflare Tunnel documentation
4. Review Ansible playbook output for errors