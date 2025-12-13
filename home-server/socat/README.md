# socat TCP Bridge (Home Server)

This directory contains the socat configuration for the home server side.

## Overview

On the home server, socat:

1. **Listens** on the WireGuard interface IP (10.8.0.2)
2. **Receives** traffic from the VPS through the tunnel
3. **Forwards** to local applications running on localhost

## Why Bind to WireGuard IP?

```bash
# This binds ONLY to the WireGuard interface
TCP4-LISTEN:2283,bind=10.8.0.2,fork,reuseaddr

# NOT this, which binds to all interfaces
TCP4-LISTEN:2283,fork,reuseaddr
```

Binding to `10.8.0.2` ensures:

- Traffic only accepted from the WireGuard tunnel
- Service not exposed on LAN
- Additional layer of security

## Traffic Flow

```
VPS (10.8.0.1)                    Home Server (10.8.0.2)
┌────────────────┐                ┌────────────────────────┐
│                │                │                        │
│  socat         │  WireGuard     │  socat                 │
│  :2283 ────────┼───Tunnel───────┼▶ 10.8.0.2:2283         │
│                │                │         │              │
└────────────────┘                │         ▼              │
                                  │  localhost:2283        │
                                  │  (Immich)              │
                                  │                        │
                                  └────────────────────────┘
```

## Usage

### Option 1: Integrated (Recommended)

socat is included in the main `../docker-compose.yml`:

```bash
cd ..
docker compose up -d
```

### Option 2: Standalone

```bash
docker compose up -d
```

## Connecting to Applications

### Application on Host (Direct)

Your application runs directly on the host (not in Docker):

```yaml
command: TCP4-LISTEN:2283,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:2283
network_mode: host
```

### Application in Same Docker Compose

Your application is defined in the same `docker-compose.yml`:

```yaml
services:
  socat-app:
    image: alpine/socat:latest
    command: TCP4-LISTEN:2283,fork,reuseaddr TCP4:my-app:2283
    ports:
      - "10.8.0.2:2283:2283"
    networks:
      - app-network
    depends_on:
      - my-app

  my-app:
    image: your-app:latest
    networks:
      - app-network

networks:
  app-network:
```

### Application in External Docker Network

Your application runs in a separate `docker-compose.yml`:

```yaml
# In your app's docker-compose.yml
networks:
  default:
    name: my-app-network

# In socat's docker-compose.yml
services:
  socat-app:
    image: alpine/socat:latest
    command: TCP4-LISTEN:2283,fork,reuseaddr TCP4:my-app-container:2283
    ports:
      - "10.8.0.2:2283:2283"
    networks:
      - my-app-network

networks:
  my-app-network:
    external: true
```

## Common Applications

### Immich

```yaml
socat-immich:
  image: alpine/socat:latest
  command: TCP4-LISTEN:2283,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:2283
  network_mode: host
```

Or connect to Immich container:

```yaml
socat-immich:
  command: TCP4-LISTEN:2283,fork,reuseaddr TCP4:immich_server:2283
  ports:
    - "10.8.0.2:2283:2283"
  networks:
    - immich_default

networks:
  immich_default:
    external: true
```

### Jellyfin

```yaml
socat-jellyfin:
  image: alpine/socat:latest
  command: TCP4-LISTEN:8096,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:8096
  network_mode: host
```

### Home Assistant

```yaml
socat-homeassistant:
  image: alpine/socat:latest
  command: TCP4-LISTEN:8123,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:8123
  network_mode: host
```

### Nextcloud

```yaml
socat-nextcloud:
  image: alpine/socat:latest
  command: TCP4-LISTEN:8080,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:8080
  network_mode: host
```

## Troubleshooting

### Check socat is Running

```bash
docker ps | grep socat
docker logs socat-immich
```

### Verify Binding

```bash
# Should show socat listening on 10.8.0.2:2283
ss -tlnp | grep 2283
netstat -tlnp | grep 2283
```

### Test Local Application

```bash
# Test if application responds locally
curl http://127.0.0.1:2283

# Test through socat (from WireGuard IP perspective)
curl http://10.8.0.2:2283
```

### Test from VPS

On the VPS:

```bash
# Should reach home server application
curl http://10.8.0.2:2283
```

### Common Issues

| Issue | Cause | Solution |
|-------|-------|----------|
| Connection refused | App not running | Check application status |
| No route to host | WireGuard down | Run `sudo wg show` |
| Connection timeout | Firewall blocking | Check host firewall rules |
| Bind failed | IP not available | Ensure WireGuard is up first |

### Debug Mode

Run socat with verbose output:

```yaml
command: -d -d TCP4-LISTEN:2283,bind=10.8.0.2,fork,reuseaddr TCP4:127.0.0.1:2283
```

Then check logs:

```bash
docker logs -f socat-immich
```

## Performance

socat adds minimal overhead:

- **Latency**: Sub-millisecond
- **Throughput**: Limited by network, not socat
- **Memory**: ~5-10MB per container
- **CPU**: Negligible

For optimal performance:

1. Ensure WireGuard MTU is appropriate
2. Monitor with `iftop` or `nethogs`
3. Place socat container near application (same host)
