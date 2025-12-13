# Traefik Configuration

This directory contains the Traefik reverse proxy configuration for the VPS.

## Files

| File | Purpose |
|------|---------|
| `traefik.yml` | Static configuration (entrypoints, providers, logging) |
| `dynamic.yml` | Dynamic configuration (routers, services, TLS) |
| `certs/` | Cloudflare origin certificates (you must create this) |

## Setup

### 1. Create Certificate Directory

```bash
mkdir -p certs
```

### 2. Obtain Cloudflare Origin Certificate

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Select your domain
3. Go to **SSL/TLS** → **Origin Server**
4. Click **Create Certificate**
5. Settings:
   - Private key type: **RSA (2048)**
   - Hostnames: `*.example.com, example.com`
   - Validity: **15 years** (recommended for origin certs)
6. Copy the certificate to `certs/origin.pem`
7. Copy the private key to `certs/origin-key.pem`

```bash
# Set correct permissions
chmod 600 certs/origin-key.pem
chmod 644 certs/origin.pem
```

### 3. Configure Dynamic Routing

Edit `dynamic.yml` to:

1. Replace `app.example.com` with your actual domain
2. Add additional routers for other applications
3. Adjust middlewares as needed

## Security Considerations

### Why Origin Certificates?

Cloudflare origin certificates are:

- **Free and long-lived**: 15-year validity
- **Only valid through Cloudflare**: Attackers can't use them directly
- **Simpler than ACME**: No renewal automation needed
- **Trusted by Cloudflare**: Required for Full (Strict) SSL mode

### Headers

The `security-headers` middleware adds:

| Header | Purpose |
|--------|---------|
| `X-Frame-Options: DENY` | Prevent clickjacking |
| `X-Content-Type-Options: nosniff` | Prevent MIME sniffing |
| `X-XSS-Protection: 1` | Enable XSS filter |
| `Strict-Transport-Security` | Force HTTPS |
| `Referrer-Policy` | Control referrer information |

### Rate Limiting

Default rate limits:

- 100 requests/second average
- 200 request burst

Adjust in `dynamic.yml` based on your application needs.

## Adding New Applications

To expose a new application:

### 1. Add socat bridge in `../docker-compose.yml`

```yaml
socat-newapp:
  image: alpine/socat:latest
  container_name: socat-newapp
  restart: unless-stopped
  command: TCP4-LISTEN:8080,fork,reuseaddr TCP4:10.8.0.2:8080
  network_mode: host
```

### 2. Add router and service in `dynamic.yml`

```yaml
http:
  routers:
    newapp:
      rule: "Host(`newapp.example.com`)"
      entryPoints:
        - websecure
      service: newapp
      tls:
        options: modern
      middlewares:
        - security-headers

  services:
    newapp:
      loadBalancer:
        servers:
          - url: "http://host.docker.internal:8080"
        passHostHeader: true
```

### 3. Add DNS record in Cloudflare

Add an A record for `newapp.example.com` pointing to your VPS IP (proxied).

### 4. Restart services

```bash
docker compose restart traefik
```

## Troubleshooting

### Check Traefik logs

```bash
docker logs -f traefik
```

### Verify TLS certificate

```bash
openssl s_client -connect localhost:443 -servername app.example.com
```

### Test routing without Cloudflare

Add to `/etc/hosts`:

```
203.0.113.10 app.example.com
```

Then:

```bash
curl -k https://app.example.com
```

### Common Issues

| Issue | Solution |
|-------|----------|
| Certificate not found | Check file paths in dynamic.yml |
| Connection refused | Verify socat is running: `netstat -tlnp \| grep 2283` |
| TLS handshake failure | Ensure Cloudflare SSL mode is "Full (Strict)" |
| 502 Bad Gateway | Backend (socat/WireGuard) not reachable |
