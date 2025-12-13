# Architecture Diagram

This directory contains the architecture diagram for the CGNAT VPS Gateway solution.

## Files

- `diagram.mmd` - Mermaid source file
- `diagram.png` - Rendered PNG (placeholder)

## Rendering the Diagram

### Option 1: Mermaid CLI

```bash
# Install mermaid-cli
npm install -g @mermaid-js/mermaid-cli

# Render to PNG
mmdc -i diagram.mmd -o diagram.png -b transparent -w 1600

# Render to SVG
mmdc -i diagram.mmd -o diagram.svg -b transparent
```

### Option 2: Mermaid Live Editor

1. Visit [mermaid.live](https://mermaid.live)
2. Paste the contents of `diagram.mmd`
3. Export as PNG or SVG

### Option 3: GitHub Rendering

GitHub automatically renders Mermaid diagrams in Markdown files. View the diagram directly in the main README.md.

### Option 4: VS Code Extension

Install the "Mermaid Preview" extension in VS Code to preview the diagram while editing.

## Diagram Overview

The diagram shows the complete traffic flow:

1. **User** makes HTTPS request to `app.example.com`
2. **Cloudflare** handles DNS, WAF, and TLS termination
3. **VPS** runs Traefik (reverse proxy) and socat (TCP bridge)
4. **WireGuard tunnel** connects VPS to home server
5. **Home server** runs socat bridge and the target application

### Key Components

| Component | Role | Network |
|-----------|------|---------|
| Cloudflare | Public-facing WAF and TLS | Internet |
| Traefik | HTTPS routing on VPS | VPS public IP |
| socat (VPS) | TCP bridge to WireGuard network | VPS → 10.8.0.0/24 |
| WireGuard | Encrypted tunnel | 10.8.0.0/24 |
| socat (Home) | TCP bridge to application | Home → localhost |
| Application | Target service (e.g., Immich) | localhost |
