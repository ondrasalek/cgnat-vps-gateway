# Troubleshooting Guide

This document covers common issues and solutions for the CGNAT VPS Gateway architecture.

## Quick Diagnostic Commands

Run these first to identify the problem:

```bash
# VPS
sudo wg show                     # WireGuard status
docker ps                        # Container status
docker logs traefik              # Traefik logs
ss -tlnp | grep 443             # Port 443 listener
ping 10.8.0.2                   # Ping home server

# Home Server
sudo wg show                     # WireGuard status
docker ps                        # Container status
ping 10.8.0.1                   # Ping VPS
curl http://localhost:2283      # Test application
```

## WireGuard Issues

### No WireGuard Handshake

**Symptom**: `sudo wg show` shows no "latest handshake" or very old timestamp.

**Check on VPS:**

```bash
# Is WireGuard running?
sudo wg show

# Is the port open?
sudo ss -ulnp | grep 51820

# Check firewall
sudo ufw status | grep 51820
```

**Check on Home Server:**

```bash
# Is WireGuard running?
sudo wg show

# Can you reach VPS?
nc -vzu 203.0.113.10 51820
```

**Common causes:**

| Cause | Solution |
|-------|----------|
| VPS firewall blocking | `sudo ufw allow 51820/udp` |
| WireGuard not started | `sudo wg-quick up wg0` |
| Wrong public keys | Verify keys match on both sides |
| Wrong endpoint IP/port | Check `Endpoint` in home server config |

### Handshake Works, But No Connectivity

**Symptom**: Handshake shows recent timestamp, but `ping 10.8.0.2` fails.

**Check:**

```bash
# IP forwarding on VPS
cat /proc/sys/net/ipv4/ip_forward
# Should be 1

# Check routing
ip route | grep wg0

# Check AllowedIPs
sudo wg show
```

**Solution:**

```bash
# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### Connection Drops Periodically

**Symptom**: Tunnel works for a while, then stops.

**Cause**: NAT mapping expires because PersistentKeepalive is not set or too high.

**Solution** (on home server config):

```ini
[Peer]
...
PersistentKeepalive = 25
```

If still happening:

```ini
# Try lower value
PersistentKeepalive = 15
```

### Slow Transfer Speeds

**Symptom**: Data transfers are slower than expected.

**Diagnostic:**

```bash
# Check for packet loss
ping -c 100 10.8.0.2

# Check MTU
ping -M do -s 1400 10.8.0.2
```

**Solutions:**

1. **MTU issues** - reduce MTU:

```ini
[Interface]
MTU = 1280
```

2. **Packet loss** - check network quality:

```bash
mtr 10.8.0.2
```

3. **CPU bottleneck** (unlikely) - check with:

```bash
top
```

## Traefik Issues

### 502 Bad Gateway

**Symptom**: Browser shows "502 Bad Gateway".

**Cause**: Traefik can't reach the backend (socat).

**Check:**

```bash
# Is socat running?
docker ps | grep socat

# Is socat listening?
ss -tlnp | grep 2283

# Can Traefik reach it?
docker exec traefik wget -q -O- http://host.docker.internal:2283
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| socat not running | `docker compose up -d` |
| Wrong port in Traefik config | Check `dynamic.yml` service URL |
| WireGuard down | Restart WireGuard |

### 404 Not Found

**Symptom**: Browser shows "404 page not found".

**Cause**: No matching router rule in Traefik.

**Check:**

```bash
# View Traefik routers (if API enabled)
curl http://localhost:8080/api/http/routers

# Check configuration
cat traefik/dynamic.yml
```

**Solutions:**

- Verify the `Host()` rule matches the requested domain
- Check domain DNS points to VPS
- Ensure router is using correct entrypoint

### TLS Handshake Failure

**Symptom**: Browser shows SSL/TLS error.

**Check:**

```bash
# Test TLS locally
openssl s_client -connect localhost:443 -servername app.example.com

# Check certificate files exist
ls -la traefik/certs/
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| Certificate missing | Regenerate origin certificate from Cloudflare |
| Wrong file paths in config | Verify paths in `dynamic.yml` |
| Certificate doesn't match domain | Regenerate with correct hostnames |

### Traefik Won't Start

**Symptom**: Container exits immediately.

**Check:**

```bash
docker logs traefik
```

**Common causes:**

- YAML syntax error in config files
- Missing certificate files
- Port already in use

**Validate config:**

```bash
# Check YAML syntax
python3 -c "import yaml; yaml.safe_load(open('traefik/traefik.yml'))"
python3 -c "import yaml; yaml.safe_load(open('traefik/dynamic.yml'))"
```

## socat Issues

### Connection Refused on socat Port

**Symptom**: `nc -vz localhost 2283` shows connection refused.

**Check:**

```bash
# Is container running?
docker ps | grep socat

# Check container logs
docker logs socat-immich
```

**Solutions:**

| Issue | Solution |
|-------|----------|
| Container not running | `docker compose up -d` |
| Port conflict | Check if another process uses the port |
| Bind error | Ensure WireGuard is up if binding to 10.8.0.2 |

### socat Timeout

**Symptom**: Connection established but times out.

**Cause**: socat forwards connection, but destination is unreachable.

**VPS socat (can't reach home):**

```bash
# Test WireGuard connectivity
ping 10.8.0.2

# Check home server socat is listening
# (run on home server)
ss -tlnp | grep 2283
```

**Home socat (can't reach application):**

```bash
# Test application directly
curl http://localhost:2283

# Check application is running
docker ps | grep immich
```

## Cloudflare Issues

### 520 Error

**Meaning**: Web server returned an unknown error.

**Cause**: Traefik returned unexpected response.

**Check:**

```bash
docker logs traefik
```

### 521 Error

**Meaning**: Web server is down.

**Cause**: VPS not responding on :443.

**Check:**

```bash
# Is Traefik running?
docker ps | grep traefik

# Is port open?
ss -tlnp | grep 443

# Test locally
curl -k https://localhost
```

### 522 Error

**Meaning**: Connection timed out.

**Cause**: Cloudflare couldn't establish TCP connection.

**Check:**

- VPS is running
- Firewall allows Cloudflare IPs
- Port 443 is listening

### 523 Error

**Meaning**: Origin is unreachable.

**Cause**: DNS or routing issue.

**Check:**

- Cloudflare DNS A record is correct
- VPS IP hasn't changed

### 525 Error

**Meaning**: SSL handshake failed.

**Check:**

```bash
# Verify certificate
openssl s_client -connect localhost:443 -servername app.example.com
```

**Solutions:**

- Regenerate origin certificate
- Verify Cloudflare SSL mode is "Full (Strict)"

### 526 Error

**Meaning**: Invalid SSL certificate.

**Check:**

- Certificate covers the requested domain
- Certificate hasn't expired
- Certificate is from Cloudflare (not self-signed)

## Application-Specific Issues

### Immich Not Loading

**Symptoms**:

- Page loads but stuck on "Loading..."
- API errors in browser console

**Check:**

```bash
# On home server
docker ps | grep immich
docker logs immich_server

# Test API
curl http://localhost:2283/api/server-info/ping
```

### WebSocket Not Working

**Symptom**: Real-time features don't work.

**Cause**: WebSocket upgrade not forwarded.

**Traefik config:**

```yaml
http:
  services:
    myapp:
      loadBalancer:
        servers:
          - url: "http://host.docker.internal:8080"
        passHostHeader: true
```

**Cloudflare:**

- Network → WebSockets: Enabled

### Large File Uploads Fail

**Symptom**: Upload starts but fails partway.

**Causes:**

1. Cloudflare timeout (free plan: 100 seconds)
2. Request size limit (free plan: 100MB)

**Solutions:**

- For large files, consider direct upload outside Cloudflare
- Upgrade to paid Cloudflare plan
- Use chunked uploads in application

## Network Debugging

### Trace Full Request Path

```bash
# 1. Test from public internet
curl -v https://app.example.com

# 2. Test Cloudflare to VPS
# (on VPS, temporarily allow all traffic to :443)
curl -k https://vps-ip

# 3. Test Traefik to socat
docker exec traefik wget -q -O- http://host.docker.internal:2283

# 4. Test VPS to home via WireGuard
ping 10.8.0.2
curl http://10.8.0.2:2283

# 5. Test home socat to application
# (on home server)
curl http://localhost:2283
```

### Packet Capture

For deep debugging:

```bash
# VPS: Capture WireGuard traffic
sudo tcpdump -i wg0 -n

# Home: Capture application traffic
sudo tcpdump -i lo port 2283 -n

# Capture with detailed output
sudo tcpdump -i wg0 -n -v -X
```

### MTU Path Discovery

Find optimal MTU:

```bash
# Binary search for working MTU
for size in 1500 1400 1300 1280 1200; do
    if ping -M do -c 1 -s $size 10.8.0.2 > /dev/null 2>&1; then
        echo "MTU $size: OK"
    else
        echo "MTU $size: FAIL"
    fi
done
```

## Log Locations

| Service | Location |
|---------|----------|
| WireGuard | `journalctl -u wg-quick@wg0` |
| Traefik | `docker logs traefik` |
| socat | `docker logs socat-immich` |
| UFW | `/var/log/ufw.log` |
| SSH | `/var/log/auth.log` |
| System | `/var/log/syslog` |

## Getting Help

If you're still stuck:

1. **Collect information:**
   - `sudo wg show` output (redact keys!)
   - Relevant Docker logs
   - Network test results

2. **Check component isolation:**
   - Does each component work independently?
   - Where exactly does the chain break?

3. **Simplify:**
   - Test with minimal configuration
   - Remove middlewares temporarily
   - Test without Cloudflare (direct to VPS IP)

4. **Document:**
   - What changed before it broke?
   - What error messages appear?
   - What have you already tried?
