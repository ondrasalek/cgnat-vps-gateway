# socat TCP Bridge (VPS)

This directory contains the socat configuration for the VPS side of the tunnel.

## What is socat?

socat (SOcket CAT) is a command-line utility that establishes bidirectional data streams between two endpoints. In this architecture, it acts as a simple TCP bridge.

## Why socat?

| Alternative | Problem |
|-------------|---------|
| Direct Traefik routing | Traefik would need complex configuration for WireGuard IPs |
| iptables NAT | Less transparent, harder to debug |
| nginx streams | Heavier, more complex configuration |
| HAProxy | Overkill for simple TCP forwarding |

socat provides:

- **Simplicity**: One-line command per application
- **Transparency**: Easy to understand and debug
- **Reliability**: Battle-tested, minimal dependencies
- **Flexibility**: Easy to add/remove bridges

## How It Works

```
                         VPS
┌─────────────────────────────────────────────────┐
│                                                 │
│  Traefik                socat                   │
│  (:443)  ──────────▶  (:2283)                   │
│                          │                      │
│                          │                      │
│                          ▼                      │
│                    WireGuard (wg0)              │
│                     10.8.0.1                    │
│                          │                      │
└──────────────────────────┼──────────────────────┘
                           │
                    WireGuard Tunnel
                           │
┌──────────────────────────┼──────────────────────┐
│                     10.8.0.2                    │
│                    WireGuard (wg0)              │
│                          │                      │
│                          ▼                      │
│                    socat (:2283)                │
│                          │                      │
│                          ▼                      │
│                  Application (:2283)            │
│                                                 │
│                    Home Server                  │
└─────────────────────────────────────────────────┘
```

## Usage

### Option 1: Integrated (Recommended)

socat is included in the main `../docker-compose.yml`. No separate action needed.

```bash
cd ..
docker compose up -d
```

### Option 2: Standalone

If you need to run socat separately:

```bash
cd socat
docker compose up -d
```

## Command Breakdown

```
TCP4-LISTEN:2283,fork,reuseaddr TCP4:10.8.0.2:2283
```

| Part | Meaning |
|------|---------|
| `TCP4-LISTEN:2283` | Listen on IPv4 TCP port 2283 |
| `fork` | Fork a new process for each connection |
| `reuseaddr` | Reuse the port immediately after restart |
| `TCP4:10.8.0.2:2283` | Forward to this address and port |

## Adding a New Application

### 1. Determine the port

Find what port your application uses. Common examples:

| Application | Port |
|-------------|------|
| Immich | 2283 |
| Jellyfin | 8096 |
| Plex | 32400 |
| Nextcloud | 80/443 or 8080 |
| Home Assistant | 8123 |
| Gitea | 3000 |
| Vaultwarden | 80 or 8080 |

### 2. Add socat service

Edit `docker-compose.yml` or `../docker-compose.yml`:

```yaml
socat-jellyfin:
  image: alpine/socat:latest
  container_name: socat-jellyfin
  restart: unless-stopped
  command: TCP4-LISTEN:8096,fork,reuseaddr TCP4:10.8.0.2:8096
  network_mode: host
```

### 3. Update Traefik

Add routing in `../traefik/dynamic.yml`:

```yaml
http:
  routers:
    jellyfin:
      rule: "Host(`media.example.com`)"
      entryPoints:
        - websecure
      service: jellyfin
      tls:
        options: modern

  services:
    jellyfin:
      loadBalancer:
        servers:
          - url: "http://host.docker.internal:8096"
```

### 4. Restart services

```bash
docker compose restart
```

## Troubleshooting

### Verify socat is listening

```bash
# Check if port is open
ss -tlnp | grep 2283

# Or with netstat
netstat -tlnp | grep 2283
```

### Test connectivity

```bash
# Test WireGuard connectivity
ping 10.8.0.2

# Test socat bridge (requires netcat)
nc -vz localhost 2283

# Full test with curl (if HTTP application)
curl -v http://localhost:2283
```

### View logs

```bash
# Docker logs
docker logs -f socat-immich

# For debugging, add verbose flag to command:
# command: -d -d TCP4-LISTEN:2283,fork,reuseaddr TCP4:10.8.0.2:2283
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection refused | socat not running | Check `docker ps` |
| No route to host | WireGuard tunnel down | Check `wg show` |
| Connection timeout | Firewall blocking | Check VPS firewall rules |
| Port already in use | Another process on port | Find with `ss -tlnp` |

## Performance Considerations

socat adds minimal overhead:

- **CPU**: Negligible for most use cases
- **Memory**: ~5-10MB per container
- **Latency**: Sub-millisecond addition

For high-throughput applications, consider:

- Increasing system buffer sizes
- Using `TCP4-LISTEN:...,rcvbuf=262144,sndbuf=262144`
- Monitoring with `iftop` or `nethogs`

## Security

socat itself provides no encryption or authentication. Security comes from:

1. **WireGuard**: All traffic through the tunnel is encrypted
2. **Cloudflare**: TLS termination and WAF
3. **Firewall**: Only allow connections from localhost/Traefik

The `network_mode: host` is required to access the WireGuard interface, but means socat listens on all interfaces. Ensure your firewall only allows necessary ports from Cloudflare IPs.
