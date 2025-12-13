# Cloudflare Firewall Rules

This document describes the recommended Cloudflare configuration for the CGNAT VPS Gateway architecture.

## DNS Configuration

### A Record

Create an A record pointing to your VPS:

| Type | Name | Content | Proxy status | TTL |
|------|------|---------|--------------|-----|
| A | app | 203.0.113.10 | Proxied (orange cloud) | Auto |

**Important**: The proxy status must be **Proxied** (orange cloud) to enable Cloudflare's WAF and hide your VPS IP.

### Multiple Applications

For multiple subdomains:

| Type | Name | Content | Proxy status |
|------|------|---------|--------------|
| A | app | 203.0.113.10 | Proxied |
| A | media | 203.0.113.10 | Proxied |
| A | cloud | 203.0.113.10 | Proxied |

## SSL/TLS Configuration

### Encryption Mode

Navigate to **SSL/TLS** → **Overview** and set mode to:

**Full (Strict)**

This ensures:

- Cloudflare encrypts traffic to your origin
- Cloudflare validates your origin certificate
- End-to-end encryption with certificate verification

### Origin Certificate

1. Go to **SSL/TLS** → **Origin Server**
2. Click **Create Certificate**
3. Configure:
   - Private key type: **RSA (2048)**
   - Hostnames: `*.example.com, example.com`
   - Validity: **15 years**
4. Download certificate and private key
5. Install on VPS (see `vps/traefik/README.md`)

### Minimum TLS Version

Navigate to **SSL/TLS** → **Edge Certificates**:

- Minimum TLS Version: **TLS 1.2** (or TLS 1.3 for maximum security)
- TLS 1.3: **Enabled**
- Automatic HTTPS Rewrites: **Enabled**
- Always Use HTTPS: **Enabled**

## Firewall Rules

### VPS Firewall (Not Cloudflare)

Your VPS firewall should only allow Cloudflare IPs on port 443.

Get current Cloudflare IP ranges:

- IPv4: <https://www.cloudflare.com/ips-v4>
- IPv6: <https://www.cloudflare.com/ips-v6>

**UFW example:**

```bash
# Deny all incoming by default
sudo ufw default deny incoming

# Allow SSH (adjust as needed)
sudo ufw allow 22/tcp

# Allow WireGuard
sudo ufw allow 51820/udp

# Allow HTTPS only from Cloudflare IPs
# IPv4 ranges (as of 2024 - verify current ranges)
sudo ufw allow from 173.245.48.0/20 to any port 443 proto tcp
sudo ufw allow from 103.21.244.0/22 to any port 443 proto tcp
sudo ufw allow from 103.22.200.0/22 to any port 443 proto tcp
sudo ufw allow from 103.31.4.0/22 to any port 443 proto tcp
sudo ufw allow from 141.101.64.0/18 to any port 443 proto tcp
sudo ufw allow from 108.162.192.0/18 to any port 443 proto tcp
sudo ufw allow from 190.93.240.0/20 to any port 443 proto tcp
sudo ufw allow from 188.114.96.0/20 to any port 443 proto tcp
sudo ufw allow from 197.234.240.0/22 to any port 443 proto tcp
sudo ufw allow from 198.41.128.0/17 to any port 443 proto tcp
sudo ufw allow from 162.158.0.0/15 to any port 443 proto tcp
sudo ufw allow from 104.16.0.0/13 to any port 443 proto tcp
sudo ufw allow from 104.24.0.0/14 to any port 443 proto tcp
sudo ufw allow from 172.64.0.0/13 to any port 443 proto tcp
sudo ufw allow from 131.0.72.0/22 to any port 443 proto tcp

# Enable firewall
sudo ufw enable
```

**Automated script:**

```bash
#!/bin/bash
# Update Cloudflare IP rules for UFW

# Remove old Cloudflare rules
sudo ufw status numbered | grep "443/tcp" | awk -F"[][]" '{print $2}' | sort -rn | xargs -I {} sudo ufw --force delete {}

# Add current Cloudflare IPs
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    sudo ufw allow from $ip to any port 443 proto tcp
done

# Reload
sudo ufw reload
```

## WAF Rules (Security → WAF)

### Managed Rules

Enable Cloudflare Managed Ruleset:

1. Go to **Security** → **WAF** → **Managed rules**
2. Enable **Cloudflare Managed Ruleset**
3. Configure sensitivity based on your application

### Custom Rules

Create custom rules for additional protection:

#### Block Non-Browser Traffic (Optional)

Block requests that don't look like browsers:

```
Rule name: Block non-browser traffic
Expression: (not cf.client.bot and not http.user_agent contains "Mozilla")
Action: Block
```

> **Warning**: This may block legitimate API clients. Use carefully.

#### Rate Limiting

Create a rate limiting rule:

```
Rule name: Rate limit login attempts
Expression: (http.request.uri.path contains "/login" or http.request.uri.path contains "/api/auth")
Rate: 10 requests per minute per IP
Action: Block for 1 hour
```

#### Geographic Restrictions (Optional)

If you only access from specific countries:

```
Rule name: Block non-allowed countries
Expression: (not ip.geoip.country in {"US" "DE" "CZ"})
Action: Block
```

Adjust country codes as needed.

## Bot Fight Mode

Navigate to **Security** → **Bots**:

- Bot Fight Mode: **Enabled** (or Super Bot Fight Mode on paid plans)
- This adds challenges for suspected bots

## DDoS Protection

Cloudflare provides DDoS protection automatically. For additional control:

1. Go to **Security** → **DDoS**
2. Review and adjust sensitivity levels
3. Enable **HTTP DDoS attack protection**

## Page Rules (Optional)

For specific application behavior:

### Cache Static Assets

```
URL: *example.com/static/*
Setting: Cache Level → Cache Everything
Edge Cache TTL: 1 month
```

### Disable Security for Webhooks

If your application receives webhooks:

```
URL: *example.com/api/webhooks/*
Setting: Security Level → Essentially Off
```

> **Warning**: Only use for trusted webhook sources.

## Monitoring

### Analytics

Monitor traffic in **Analytics** → **Traffic**:

- Watch for unusual spikes
- Identify attack patterns
- Track geographic distribution

### Security Events

Review blocked requests in **Security** → **Events**:

- Verify legitimate traffic isn't blocked
- Adjust rules based on false positives

## Recommended Cloudflare Settings Checklist

| Setting | Location | Value |
|---------|----------|-------|
| SSL Mode | SSL/TLS → Overview | Full (Strict) |
| Minimum TLS | SSL/TLS → Edge Certificates | 1.2 or 1.3 |
| Always HTTPS | SSL/TLS → Edge Certificates | Enabled |
| HSTS | SSL/TLS → Edge Certificates | Enabled (careful!) |
| Auto Minify | Speed → Optimization | Enabled |
| Brotli | Speed → Optimization | Enabled |
| HTTP/2 | Network | Enabled |
| HTTP/3 | Network | Enabled |
| 0-RTT | Network | Enabled |
| WebSockets | Network | Enabled |
| Bot Fight Mode | Security → Bots | Enabled |
| Browser Integrity Check | Security → Settings | Enabled |
| Privacy Pass | Security → Settings | Enabled |

## Troubleshooting

### 520 Error (Web Server Unknown Error)

- VPS not responding or misconfigured
- Check Traefik logs: `docker logs traefik`
- Verify origin certificate is installed

### 521 Error (Web Server is Down)

- Traefik not running
- Firewall blocking Cloudflare IPs
- Check: `docker ps` and `ufw status`

### 522 Error (Connection Timed Out)

- VPS unreachable
- Firewall too restrictive
- Network issue between Cloudflare and VPS

### 523 Error (Origin is Unreachable)

- DNS misconfigured
- VPS IP changed
- Check A record in Cloudflare DNS

### 525 Error (SSL Handshake Failed)

- Origin certificate issue
- Check certificate is valid and matches domain
- Verify SSL mode is "Full (Strict)"

### 526 Error (Invalid SSL Certificate)

- Origin certificate expired or invalid
- Certificate doesn't cover the requested hostname
- Regenerate origin certificate

## Additional Resources

- [Cloudflare IP Ranges](https://www.cloudflare.com/ips/)
- [Cloudflare SSL/TLS Encryption Modes](https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/)
- [Cloudflare WAF Documentation](https://developers.cloudflare.com/waf/)
- [Cloudflare Firewall Rules](https://developers.cloudflare.com/firewall/)
