# WireGuard Configuration (Home Server)

This directory contains the WireGuard VPN configuration for the home server.

## Overview

The home server is the WireGuard **client**, initiating an outbound connection to the VPS. This is the key to bypassing CGNAT:

1. **CGNAT blocks inbound connections** - Your ISP shares a public IP among many customers
2. **Outbound connections work normally** - The home server connects out to the VPS
3. **WireGuard maintains the tunnel** - Once established, bidirectional communication works
4. **PersistentKeepalive prevents NAT timeout** - Regular packets keep the NAT mapping alive

## Prerequisites

### Confirm You're Behind CGNAT

```bash
# Check your public IP
curl -s ifconfig.me

# Check your router's WAN IP
# (access router admin panel or check via router's API)

# If these IPs are different, you're likely behind CGNAT
# CGNAT uses RFC 6598 address space: 100.64.0.0/10
```

### Install WireGuard

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install wireguard wireguard-tools
```

**Synology NAS:**

1. Install via Package Center (Community packages)
2. Or use the `runq/wireguard` Docker image

**Other NAS systems:**
Check your vendor's documentation for WireGuard support.

## Setup

### 1. Generate Keys

```bash
# Create key directory
sudo mkdir -p /etc/wireguard
cd /etc/wireguard

# Generate home server key pair
wg genkey | sudo tee privatekey | wg pubkey | sudo tee publickey

# Set secure permissions
sudo chmod 600 privatekey
sudo chmod 644 publickey

# View your public key (share this with VPS)
cat publickey
```

### 2. Get VPS Public Key

On the VPS, run:

```bash
cat /etc/wireguard/publickey
```

Copy this key to use in the home server configuration.

### 3. Configure WireGuard

```bash
# Copy example configuration
sudo cp wg0.conf.example /etc/wireguard/wg0.conf

# Edit configuration
sudo nano /etc/wireguard/wg0.conf
```

Replace:

- `[HOME_PRIVATE_KEY]` with contents of `/etc/wireguard/privatekey`
- `[VPS_PUBLIC_KEY]` with the public key from your VPS
- `203.0.113.10` with your VPS's actual public IP

### 4. Update VPS Configuration

On the VPS, add the home server's public key:

```bash
sudo nano /etc/wireguard/wg0.conf
```

Replace `[HOME_PUBLIC_KEY]` with your home server's public key.

Restart WireGuard on VPS:

```bash
sudo wg-quick down wg0
sudo wg-quick up wg0
```

### 5. Start WireGuard on Home Server

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

# Expected output:
# interface: wg0
#   public key: <HOME_PUBLIC_KEY>
#   private key: (hidden)
#   listening port: random
#
# peer: <VPS_PUBLIC_KEY>
#   endpoint: 203.0.113.10:51820
#   allowed ips: 10.8.0.1/32
#   latest handshake: X seconds ago
#   transfer: X received, X sent

# Test connectivity to VPS
ping 10.8.0.1
```

## Critical Configuration: PersistentKeepalive

### Why It's Required

When behind CGNAT, your ISP's NAT device tracks outbound connections using a mapping table:

```
Home IP:Port  <-->  Public IP:Port  <-->  VPS IP:Port
```

This mapping **expires if inactive** (typically 30-60 seconds). Without keepalives:

1. Home server establishes tunnel
2. No traffic for 60 seconds
3. NAT mapping expires
4. VPS can no longer reach home server
5. Tunnel appears broken

### How PersistentKeepalive Works

```
Home Server                CGNAT                   VPS
     │                       │                       │
     │ ──── Keepalive ────▶  │                       │
     │                       │ ── Keepalive ──────▶  │
     │                       │ (mapping refreshed)   │
     │                       │                       │
     │      25 seconds later...                      │
     │                       │                       │
     │ ──── Keepalive ────▶  │                       │
     │                       │ ── Keepalive ──────▶  │
     │                       │ (mapping refreshed)   │
     │                       │                       │
```

The setting `PersistentKeepalive = 25` sends a keepalive every 25 seconds, ensuring the NAT mapping never expires.

### Troubleshooting Keepalive Issues

If the tunnel still drops:

1. **Try lower interval**: `PersistentKeepalive = 15`
2. **Check ISP throttling**: Some ISPs aggressively drop UDP
3. **Try different VPS port**: Some networks filter port 51820

## MTU Considerations

### Why 1280?

```
Standard Ethernet MTU:    1500 bytes
WireGuard overhead:       ~60 bytes
CGNAT encapsulation:      ~20-40 bytes (varies)
Safety margin:            ~80 bytes
────────────────────────────────────
Safe WireGuard MTU:       1280 bytes
```

MTU 1280 is the IPv6 minimum MTU, guaranteed to work through any network path.

### Symptoms of MTU Problems

- Large file transfers stall
- SSH works, but SCP hangs
- Web pages partially load
- Video streaming buffers indefinitely

### Testing MTU

```bash
# Test if larger MTU works (run from home server)
ping -M do -s 1400 10.8.0.1

# If this works, try higher values
ping -M do -s 1450 10.8.0.1

# Maximum that works = safe MTU for your network
# Subtract ~28 bytes for ICMP header
```

## Security Best Practices

### 1. Key Security

- **Never commit private keys**
- Store with `chmod 600`
- Regenerate if compromised

### 2. Pre-shared Key

Add extra security with a pre-shared key:

```bash
# Generate on VPS
wg genpsk > preshared.key

# Share securely with home server
# Add to both configurations:
# PresharedKey = <contents of preshared.key>
```

### 3. Minimize Exposure

The configuration only allows `10.8.0.1/32` (VPS) through the tunnel. This means:

- Only VPS-bound traffic uses the tunnel
- Normal internet traffic uses your regular connection
- Your home network isn't exposed through the VPS

## Common Commands

```bash
# Start/stop interface
sudo wg-quick up wg0
sudo wg-quick down wg0

# View status
sudo wg show

# View detailed status
sudo wg show all

# Restart to apply changes
sudo wg-quick down wg0 && sudo wg-quick up wg0

# View real-time statistics
watch -n 1 sudo wg show

# Check interface IP
ip addr show wg0

# View routing
ip route | grep wg0
```

## Troubleshooting

### No Handshake

If `sudo wg show` shows no "latest handshake":

1. **Check VPS WireGuard is running**: `sudo wg show` on VPS
2. **Verify endpoint is reachable**: `nc -vzu 203.0.113.10 51820`
3. **Check keys match**: Ensure VPS has home server's public key
4. **Check firewall**: VPS must allow UDP 51820 inbound

### Handshake Works, But No Connectivity

1. **Check IP forwarding on VPS**: `cat /proc/sys/net/ipv4/ip_forward`
2. **Check routes**: `ip route get 10.8.0.1`
3. **Check firewall on both ends**

### Connection Drops After a While

1. **Ensure PersistentKeepalive is set** (on home server)
2. **Check ISP isn't throttling UDP**
3. **Try different WireGuard port**

### Slow Transfer Speeds

1. **Check MTU settings**
2. **Test without WireGuard**: `iperf3` between VPS and home
3. **Check for packet loss**: `ping -c 100 10.8.0.1`
