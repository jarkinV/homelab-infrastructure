# dnsmasq DNS Server Setup

This playbook sets up dnsmasq as a local DNS server on an LXC container in the internal NAT network to provide DNS resolution for the internal domain.

## Overview

The DNS server resolves `*.varshavskyi.com` to `10.0.0.10` (Traefik LXC) for easy access to services behind Traefik reverse proxy without needing to edit `/etc/hosts` files.

## Configuration

The dnsmasq configuration (`/etc/dnsmasq.conf`) includes:

- **Wildcard DNS**: `*.varshavskyi.com` → `10.0.0.10`
- **Listening addresses**: `127.0.0.1`, `10.0.0.10`
- **Upstream DNS**: Google DNS (`8.8.8.8`, `8.8.4.4`)
- **Logging**: Query logs at `/var/log/dnsmasq.log`

## Usage

### Run the playbook

```bash
ansible-playbook playbooks/lxc/setup_dnsmasq.yaml
```

### Configure clients to use the DNS server

On machines in the internal network (10.0.0.0/24), set DNS to `10.0.0.10`.

**On Linux/Unix:**
```bash
# Edit /etc/resolv.conf (temporary)
echo "nameserver 10.0.0.10" | sudo tee /etc/resolv.conf

# Or configure via systemd-resolved (permanent)
sudo systemd-resolve --interface eth0 --set-dns 10.0.0.10
```

**On Docker containers:**
```yaml
# In docker-compose.yaml
dns:
  - 10.0.0.10
  - 8.8.8.8
```

**On Proxmox LXC containers:**
Edit `/etc/resolv.conf` inside the container or set DNS in Proxmox GUI.

### Test DNS resolution

```bash
# Test from any machine with 10.0.0.10 as DNS
dig @10.0.0.10 traefik.varshavskyi.com
nslookup app.varshavskyi.com 10.0.0.10

# Expected result: Should resolve to 10.0.0.10
```

### View query logs

```bash
# SSH into the DNS server LXC
ssh root@10.0.0.10 -J proxmox

# Tail the log file
tail -f /var/log/dnsmasq.log
```

## Architecture

```
Client Machine
     ↓ DNS query: app.varshavskyi.com
     ↓
[10.0.0.10] dnsmasq DNS Server
     ↓ resolves to 10.0.0.10
     ↓
[10.0.0.10] Traefik Reverse Proxy
     ↓ routes to appropriate container
     ↓
Docker Container (actual app)
```

## Benefits

1. **No /etc/hosts editing**: All `*.varshavskyi.com` domains automatically resolve
2. **Easy service discovery**: New services behind Traefik work immediately
3. **Centralized DNS**: Change resolution in one place
4. **Internal isolation**: Domain only resolves within internal network

## Troubleshooting

### Check dnsmasq status
```bash
ssh root@10.0.0.10 -J proxmox
systemctl status dnsmasq
```

### Test local resolution
```bash
# From the DNS server itself
dig @127.0.0.1 test.varshavskyi.com
```

### Check logs for errors
```bash
journalctl -u dnsmasq -f
tail -f /var/log/dnsmasq.log
```

### Restart dnsmasq
```bash
ssh root@10.0.0.10 -J proxmox
systemctl restart dnsmasq
```

## Configuration Files

- **Main config**: `/etc/dnsmasq.conf`
- **Backup**: `/etc/dnsmasq.conf.backup`
- **Logs**: `/var/log/dnsmasq.log`

## Customization

To add more domains or change IP addresses, edit the playbook and modify the dnsmasq configuration section:

```yaml
# Example: Add another domain
address=/anotherdomain.com/10.0.0.20
```

Then re-run the playbook to apply changes.