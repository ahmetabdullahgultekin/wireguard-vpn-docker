# Self-Hosted WireGuard VPN with wg-easy

A Docker-based WireGuard VPN server with web management UI, designed for personal use on a VPS. Uses [wg-easy](https://github.com/wg-easy/wg-easy) behind Traefik reverse proxy with automatic HTTPS.

## Architecture

```
Client (Phone/PC)              VPS (Docker Host)
     │                              │
     ├── WireGuard tunnel ──────────┤── wg-easy container
     │   (UDP, encrypted)           │   ├── WireGuard server
     │                              │   └── Web UI (:51821)
     │                              │
     │                              ├── Traefik container
     │                              │   ├── HTTPS termination
     │                              │   ├── Let's Encrypt certs
     │                              │   └── Routes vpn.domain.com → wg-easy:51821
     │                              │
     └──────────────────────────────┘
```

## What This Gives You

- **Full traffic encryption** between your device and the VPS
- **DNS leak protection** — DNS queries go through the tunnel
- **Web UI** to manage clients (add/remove devices, QR codes, traffic stats)
- **Auto-HTTPS** via Traefik + Let's Encrypt for the management interface
- **~50-80MB RAM** footprint (WireGuard runs in kernel space)

## Prerequisites

- A VPS with Docker and Docker Compose
- A domain with DNS pointing to your VPS
- Traefik reverse proxy (or adapt for your own setup)
- Linux kernel 5.6+ (WireGuard built-in) or `wireguard-tools` package

## Quick Start

### 1. Load WireGuard kernel module

```bash
sudo modprobe wireguard
echo "wireguard" | sudo tee /etc/modules-load.d/wireguard.conf

# Enable IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
```

### 2. Open WireGuard UDP port

```bash
sudo ufw allow 51820/udp comment "WireGuard VPN"
```

### 3. Generate password hash

```bash
docker run --rm ghcr.io/wg-easy/wg-easy wgpw 'YOUR_SECURE_PASSWORD'
```

### 4. Create `.env.prod`

```bash
cp .env.example .env.prod
# Edit .env.prod with your values:
# - WG_HOST: your server's public IP
# - PASSWORD_HASH: output from step 3 (double $$ for docker-compose)
```

### 5. Configure DNS

Add an A record for your VPN management subdomain pointing to your server IP:
```
vpn.yourdomain.com → A → YOUR_SERVER_IP
```

### 6. Start

```bash
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d
```

### 7. Connect

1. Open `https://vpn.yourdomain.com` in your browser
2. Log in with your password
3. Click "New Client" → name it (e.g., "iPhone", "Laptop")
4. Scan the QR code with the WireGuard app on your device

## Client Apps

| Platform | App |
|----------|-----|
| Android | [WireGuard](https://play.google.com/store/apps/details?id=com.wireguard.android) |
| iOS | [WireGuard](https://apps.apple.com/app/wireguard/id1441195209) |
| Windows | [WireGuard](https://www.wireguard.com/install/) |
| macOS | [WireGuard](https://apps.apple.com/app/wireguard/id1451685025) |
| Linux | `sudo apt install wireguard-tools` |

## Configuration

All configuration is in `.env.prod`. See `.env.example` for available options.

| Variable | Default | Description |
|----------|---------|-------------|
| `WG_HOST` | (required) | Server's public IPv4 address |
| `PASSWORD_HASH` | (required) | Bcrypt hash of web UI password |
| `WG_PORT` | `51820` | WireGuard UDP listen port |
| `WG_DEFAULT_DNS` | `1.1.1.1,8.8.8.8` | DNS servers pushed to clients |
| `WG_ALLOWED_IPS` | `0.0.0.0/0,::/0` | Traffic to route through VPN |
| `WG_PERSISTENT_KEEPALIVE` | `25` | Keep-alive interval (seconds) |
| `WG_DEFAULT_ADDRESS` | `10.8.0.x` | Client IP subnet |
| `WG_MTU` | `1420` | Tunnel MTU size |

## Traefik Integration

The `docker-compose.prod.yml` includes Traefik labels for automatic HTTPS routing. If you use a different reverse proxy, remove the `labels` section and expose port 51821 directly (not recommended for production).

## Without Traefik

If you don't use Traefik, add port 51821 mapping and remove the labels:

```yaml
services:
  wg-easy:
    # ... same config ...
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"  # Web UI - restrict with firewall!
    # Remove: labels, networks
```

**Important:** Restrict port 51821 access via firewall if exposing directly.

## Security Notes

- The web UI is password-protected (bcrypt hash)
- HTTPS is enforced via Traefik + Let's Encrypt
- WireGuard uses ChaCha20-Poly1305 encryption + Curve25519 key exchange
- Client private keys are generated server-side — the QR code contains the full config
- WireGuard UDP port (51820) only accepts authenticated peers
- No logs are kept by default

## Maintenance

```bash
# View logs
docker logs wg-easy -f --tail=50

# Restart
docker compose -f docker-compose.prod.yml --env-file .env.prod restart

# Update
docker compose -f docker-compose.prod.yml --env-file .env.prod pull
docker compose -f docker-compose.prod.yml --env-file .env.prod up -d

# Backup client configs
cp -r data/ /path/to/backup/
```

## License

MIT
