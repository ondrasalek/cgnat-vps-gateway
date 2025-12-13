# Security Hardening Guide

This document provides security best practices for the CGNAT VPS Gateway architecture.

## Threat Model

### Attack Surface

| Component | Exposure | Risk Level |
|-----------|----------|------------|
| Cloudflare | Public | Low (managed service) |
| VPS :443 | Cloudflare only | Medium |
| VPS :51820 | Public (WireGuard) | Medium |
| VPS SSH | Public | High |
| WireGuard Tunnel | Encrypted | Low |
| Home Server | WireGuard only | Low |

### Attack Vectors

1. **Direct VPS attack**: Mitigated by firewall (Cloudflare IPs only on :443)
2. **WireGuard attacks**: Limited by cryptographic authentication
3. **SSH brute force**: Mitigated by key-only auth and fail2ban
4. **Application vulnerabilities**: Mitigated by Cloudflare WAF
5. **Home network exposure**: Home IP never revealed

## VPS Hardening

### 1. SSH Security

#### Disable Password Authentication

```bash
sudo nano /etc/ssh/sshd_config
```

```
PasswordAuthentication no
PermitRootLogin no
PubkeyAuthentication yes
PermitEmptyPasswords no
MaxAuthTries 3
```

```bash
sudo systemctl restart sshd
```

#### Use SSH Keys Only

```bash
# Generate key locally (if not exists)
ssh-keygen -t ed25519 -C "your-email@example.com"

# Copy to server
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@vps-ip
```

#### Change SSH Port (Optional)

```bash
# /etc/ssh/sshd_config
Port 2222
```

Update firewall accordingly.

### 2. Firewall Configuration

#### UFW (Recommended)

```bash
# Reset firewall
sudo ufw reset

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (change port if modified)
sudo ufw allow 22/tcp

# Allow WireGuard
sudo ufw allow 51820/udp

# Allow HTTPS only from Cloudflare IPs
# (see cloudflare/firewall-rules.md for full list)
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
    sudo ufw allow from $ip to any port 443 proto tcp
done

# Enable
sudo ufw enable
```

#### Verify Rules

```bash
sudo ufw status verbose
```

### 3. Fail2ban

Install and configure fail2ban to prevent brute force:

```bash
sudo apt install fail2ban
```

Create `/etc/fail2ban/jail.local`:

```ini
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 3
ignoreip = 127.0.0.1/8

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 1d
```

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

### 4. Automatic Updates

Enable unattended security updates:

```bash
sudo apt install unattended-upgrades
sudo dpkg-reconfigure unattended-upgrades
```

Edit `/etc/apt/apt.conf.d/50unattended-upgrades`:

```
Unattended-Upgrade::Allowed-Origins {
    "${distro_id}:${distro_codename}-security";
};
Unattended-Upgrade::Automatic-Reboot "true";
Unattended-Upgrade::Automatic-Reboot-Time "03:00";
```

### 5. Kernel Hardening

Add to `/etc/sysctl.conf`:

```bash
# IP spoofing protection
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1

# Disable source routing
net.ipv4.conf.all.accept_source_route = 0
net.ipv4.conf.default.accept_source_route = 0

# Ignore ICMP redirect
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0

# Disable ICMP redirect sending
net.ipv4.conf.all.send_redirects = 0
net.ipv4.conf.default.send_redirects = 0

# Log suspicious packets
net.ipv4.conf.all.log_martians = 1

# Ignore broadcast pings
net.ipv4.icmp_echo_ignore_broadcasts = 1

# SYN flood protection
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 2048
net.ipv4.tcp_synack_retries = 2

# Enable IP forwarding (required for WireGuard)
net.ipv4.ip_forward = 1
```

Apply:

```bash
sudo sysctl -p
```

## WireGuard Security

### 1. Key Management

#### Generate Strong Keys

```bash
# Use WireGuard's key generation (uses Curve25519)
wg genkey | sudo tee /etc/wireguard/privatekey | wg pubkey | sudo tee /etc/wireguard/publickey
```

#### Secure Key Permissions

```bash
sudo chmod 600 /etc/wireguard/privatekey
sudo chmod 644 /etc/wireguard/publickey
```

#### Key Rotation

Rotate keys periodically (every 6-12 months):

1. Generate new keys on both sides
2. Update configurations
3. Restart WireGuard on both VPS and home server
4. Securely delete old keys

### 2. Pre-shared Keys

Add an extra layer of post-quantum security:

```bash
# Generate on one side
wg genpsk | sudo tee /etc/wireguard/preshared.key
sudo chmod 600 /etc/wireguard/preshared.key
```

Add to both configurations:

```ini
[Peer]
...
PresharedKey = <contents of preshared.key>
```

### 3. Restrict Allowed IPs

Minimize the attack surface:

**VPS side** - only allow home server's specific IP:

```ini
AllowedIPs = 10.8.0.2/32
```

**Home server side** - only allow VPS's specific IP:

```ini
AllowedIPs = 10.8.0.1/32
```

Don't use `0.0.0.0/0` unless routing all traffic through VPN.

## Docker Security

### 1. Non-root Containers

Where possible, run containers as non-root:

```yaml
services:
  myapp:
    image: myapp:latest
    user: "1000:1000"
```

### 2. Read-only Filesystem

Mount containers as read-only where possible:

```yaml
services:
  myapp:
    image: myapp:latest
    read_only: true
    tmpfs:
      - /tmp
```

### 3. Resource Limits

Prevent resource exhaustion:

```yaml
services:
  myapp:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M
```

### 4. Security Options

Add security options:

```yaml
services:
  myapp:
    image: myapp:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE  # Only if needed
```

### 5. Docker Daemon Security

Edit `/etc/docker/daemon.json`:

```json
{
  "icc": false,
  "userns-remap": "default",
  "no-new-privileges": true,
  "live-restore": true
}
```

## Traefik Security

### 1. Disable Dashboard in Production

```yaml
# traefik.yml
api:
  dashboard: false
  insecure: false
```

### 2. Security Headers

Configure in `dynamic.yml`:

```yaml
http:
  middlewares:
    security-headers:
      headers:
        frameDeny: true
        contentTypeNosniff: true
        browserXssFilter: true
        referrerPolicy: "strict-origin-when-cross-origin"
        stsSeconds: 31536000
        stsIncludeSubdomains: true
        stsPreload: true
```

### 3. TLS Configuration

Use modern TLS settings:

```yaml
tls:
  options:
    modern:
      minVersion: VersionTLS13
      sniStrict: true
```

### 4. Rate Limiting

Apply rate limiting middleware:

```yaml
http:
  middlewares:
    rate-limit:
      rateLimit:
        average: 100
        burst: 200
        period: 1s
```

## Monitoring and Logging

### 1. Centralized Logging

Forward logs to a central location:

```yaml
services:
  traefik:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
```

### 2. Monitor WireGuard

Create a monitoring script:

```bash
#!/bin/bash
# Check WireGuard status

PEER_HANDSHAKE=$(sudo wg show wg0 latest-handshakes | awk '{print $2}')
CURRENT_TIME=$(date +%s)
DIFF=$((CURRENT_TIME - PEER_HANDSHAKE))

# Alert if last handshake was more than 5 minutes ago
if [ $DIFF -gt 300 ]; then
    echo "WARNING: WireGuard handshake is stale ($DIFF seconds)"
    # Add alerting here (email, webhook, etc.)
fi
```

### 3. Log Analysis

Monitor for suspicious activity:

```bash
# Failed SSH attempts
grep "Failed password" /var/log/auth.log | tail -20

# UFW blocks
grep "UFW BLOCK" /var/log/syslog | tail -20

# Traefik errors
docker logs traefik 2>&1 | grep -i error | tail -20
```

## Backup and Recovery

### 1. Backup WireGuard Keys

Securely backup your keys:

```bash
# Create encrypted backup
tar czf - /etc/wireguard/*.key | gpg -c > wireguard-keys-backup.tar.gz.gpg

# Store in secure location (not on the same server!)
```

### 2. Configuration Backup

Regularly backup configurations:

```bash
# VPS configuration
tar czf vps-config-$(date +%Y%m%d).tar.gz \
    /etc/wireguard/wg0.conf \
    /path/to/docker-compose.yml \
    /path/to/traefik/

# Encrypt and store securely
```

### 3. Recovery Plan

Document your recovery procedure:

1. Provision new VPS
2. Install WireGuard and Docker
3. Restore configurations
4. Update Cloudflare DNS
5. Regenerate origin certificates
6. Update home server WireGuard endpoint

## Regular Security Tasks

### Weekly

- [ ] Review failed SSH attempts
- [ ] Check WireGuard handshake status
- [ ] Review Cloudflare security events

### Monthly

- [ ] Update all packages: `sudo apt update && sudo apt upgrade`
- [ ] Review Docker image updates
- [ ] Check disk space and logs
- [ ] Verify backup integrity

### Quarterly

- [ ] Rotate SSH keys
- [ ] Review firewall rules
- [ ] Update Cloudflare IP ranges in firewall
- [ ] Review and prune Docker images

### Annually

- [ ] Rotate WireGuard keys
- [ ] Rotate pre-shared keys
- [ ] Review and update security policies
- [ ] Renew any expiring certificates
- [ ] Perform security audit

## Security Checklist

| Item | Status |
|------|--------|
| SSH key-only authentication | ☐ |
| SSH root login disabled | ☐ |
| Firewall enabled (UFW) | ☐ |
| Only Cloudflare IPs allowed on :443 | ☐ |
| Fail2ban configured | ☐ |
| Automatic security updates enabled | ☐ |
| WireGuard keys secured (chmod 600) | ☐ |
| Pre-shared key configured | ☐ |
| Docker containers run as non-root | ☐ |
| Traefik dashboard disabled | ☐ |
| Security headers configured | ☐ |
| Rate limiting enabled | ☐ |
| Cloudflare SSL mode set to Full (Strict) | ☐ |
| Cloudflare WAF enabled | ☐ |
| Backup procedure documented and tested | ☐ |
| Monitoring configured | ☐ |
