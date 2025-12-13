# WireGuard Configuration (VPS)

This directory contains the WireGuard VPN configuration for the VPS server.

## Overview

The VPS acts as the WireGuard **server**, listening for incoming connections from the home server. Since the home server is behind CGNAT, it initiates the connection outbound, bypassing NAT restrictions.

## Prerequisites

### Install WireGuard

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install wireguard wireguard-tools
```

**CentOS/RHEL:**

```bash
sudo dnf install epel-release
sudo dnf install wireguard-tools
```

## Setup

### 1. Generate Keys

```bash
# Create key directory
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

# Generate VPS key pair
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey

# Set secure permissions
sudo chmod 600 privatekey
sudo chmod 644 publickey

# View your public key (share this with home server)
cat publickey
```

### 2. Configure WireGuard

```bash
# Copy example configuration
sudo cp wg0.conf.example /etc/wireguard/wg0.conf

# Edit configuration
sudo nano /etc/wireguard/wg0.conf
```

Replace:

- `[VPS_PRIVATE_KEY]` with contents of `/etc/wireguard/privatekey`
- `[HOME_PUBLIC_KEY]` with the public key from your home server

### 3. Enable IP Forwarding

```bash
# Enable immediately
sudo sysctl -w net.ipv4.ip_forward=1

# Enable permanently
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

### 4. Configure Firewall

Allow WireGuard port and configure forwarding:

**UFW (Ubuntu):**

```bash
sudo ufw allow 51820/udp
sudo ufw reload
```

**firewalld (CentOS/RHEL):**

```bash
sudo firewall-cmd --permanent --add-port=51820/udp
sudo firewall-cmd --reload
```

### 5. Start WireGuard

```bash
# Start interface
sudo wg-quick up wg0

# Enable on boot
sudo systemctl enable wg-quick@wg0
```

### 6. Verify Connection

```bash
# Check interface status
sudo wg show

# Expected output after home server connects:
# interface: wg0
#   public key: <VPS_PUBLIC_KEY>
#   private key: (hidden)
#   listening port: 51820
#
# peer: <HOME_PUBLIC_KEY>
#   endpoint: <HOME_IP>:random_port
#   allowed ips: 10.8.0.2/32
#   latest handshake: X seconds ago
#   transfer: X received, X sent
```

## Key Configuration Items

### MTU

```
MTU = 1280
```

**Why 1280?**

- Guaranteed to work through any IPv4/IPv6 network
- Avoids fragmentation issues that break connections
- Slightly lower throughput but more reliable

**Optimization:** If your VPS and home network both support it, test higher values:

```bash
# Test with ping
ping -M do -s 1400 10.8.0.2
```

If successful, you can increase MTU to ~1420.

### PersistentKeepalive

```
PersistentKeepalive = 25
```

**Critical for CGNAT!** This ensures the home server sends keepalive packets every 25 seconds, preventing NAT mappings from expiring.

Without this:

- NAT typically expires mappings after 30-60 seconds of inactivity
- The tunnel would break whenever there's no traffic
- The home server wouldn't be reachable

### AllowedIPs

```
AllowedIPs = 10.8.0.2/32
```

This routes only the home server's WireGuard IP through the tunnel. For the VPS, we don't need to route additional subnets since traffic flows from the internet to the home server, not the other way around.

## Security Best Practices

### 1. Key Security

- **Never commit private keys** to version control
- Store keys with `chmod 600`
- Regenerate keys if compromised

### 2. Pre-shared Key (Optional)

Add an extra layer of security with a pre-shared key:

```bash
# Generate on either VPS or home server
wg genpsk | sudo tee /etc/wireguard/preshared.key
sudo chmod 600 /etc/wireguard/preshared.key
```

Add to both configurations:

```
PresharedKey = <contents of preshared.key>
```

### 3. Restrict WireGuard Port Access

If possible, restrict the WireGuard port to your home IP (if static):

```bash
# UFW example
sudo ufw allow from HOME_IP to any port 51820 proto udp
```

However, this doesn't work with CGNAT since your public IP changes.

## Troubleshooting

### No Handshake

```bash
sudo wg show
# If "latest handshake" is empty or very old
```

Causes:

1. Home server not started WireGuard
2. Firewall blocking port 51820/udp
3. Incorrect public keys
4. VPS endpoint not reachable from home

### Connection Drops

If the tunnel frequently disconnects:

1. Check PersistentKeepalive is set on **home server**
2. Verify MTU isn't causing fragmentation
3. Check for packet loss: `ping -c 100 10.8.0.2`

### Can't Reach 10.8.0.2

1. Verify IP forwarding: `cat /proc/sys/net/ipv4/ip_forward`
2. Check WireGuard is up: `ip a show wg0`
3. Test from VPS: `ping 10.8.0.2`

## Common Commands

```bash
# Start/stop interface
sudo wg-quick up wg0
sudo wg-quick down wg0

# View status
sudo wg show

# View configuration
sudo wg showconf wg0

# Monitor in real-time
watch -n 1 sudo wg show

# Check interface IP
ip addr show wg0

# View routing table
ip route | grep wg0
```
