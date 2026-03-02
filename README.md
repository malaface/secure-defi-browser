# Secure DeFi Browser

A hardened, VPN-routed remote browser for DeFi and Web3 operations. All traffic exits through a WireGuard VPN tunnel — your real IP is never exposed to the sites you visit.

```
┌──────────────────────────────────────────────────────────┐
│                     Docker Host                          │
│                                                          │
│  ┌───────────┐    network_mode     ┌──────────────────┐  │
│  │  Gluetun  │◄════════════════════│  Brave (KasmVNC) │  │
│  │  (VPN)    │  all traffic flows  │  Port 6901/HTTPS │  │
│  │           │  through VPN only   │                  │  │
│  └─────┬─────┘                     └──────────────────┘  │
│        │ WireGuard                                       │
│        │                                                 │
│        │           ┌────────────────────┐                │
│        │           │  cloudflared       │                │
│        │           │  (Tunnel Agent)    │                │
│        │           └────────┬───────────┘                │
└────────┼────────────────────┼────────────────────────────┘
         │                    │
         ▼                    ▼
    ┌─────────┐       ┌──────────────┐
    │ VPN     │       │  Cloudflare  │
    │ Server  │       │  Edge        │
    └────┬────┘       └──────┬───────┘
         │                   │
         ▼                   ▼
    Public Internet     Zero Trust Access
    (VPN exit IP)       (https://your-domain.com)
```

## Why these technologies?

### Brave over Firefox/Chrome

- **Built-in wallet** — Native support for Ethereum, Solana, and other chains without extensions
- **Shields** — Aggressive ad/tracker blocking enabled by default, critical for DeFi sites loaded with trackers
- **Fingerprint protection** — Randomizes browser fingerprint, making cross-site tracking harder
- **No telemetry risk** — Unlike Chrome, no data sent to Google while you interact with your wallets

### Gluetun over running VPN on the host

- **Container-level isolation** — Only the browser goes through the VPN; your other services keep their normal routing
- **Kill switch built-in** — If the VPN drops, Gluetun's firewall blocks all traffic automatically. No IP leak possible
- **60+ VPN providers** — Works with ProtonVPN, Mullvad, NordVPN, Surfshark, IVPN, and [many more](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
- **`network_mode: "service:gluetun"`** — The browser container has no network interface of its own. It physically cannot bypass the VPN

### KasmVNC over traditional VNC

- **Browser-based access** — Connect via any browser at `https://localhost:6901`, no VNC client needed
- **HTTPS by default** — Self-signed TLS out of the box, encrypted connection even locally
- **Better performance** — WebSocket-based streaming is faster and more efficient than classic VNC protocols
- **Clipboard support** — Copy/paste works between your local machine and the remote browser

### Cloudflare Tunnel over port forwarding

- **No open ports** — Your server exposes nothing to the internet; the tunnel connects outbound to Cloudflare
- **Zero Trust Access** — Enforce email/OTP/SSO authentication before anyone can reach the browser
- **DDoS protection** — Cloudflare's network sits in front of your service
- **Token-based setup** — Single environment variable, no config files or credentials to manage

### Pinned versions over `latest`

All images use fixed version tags (`kasmweb/brave:1.14.0`, `qmcgaw/gluetun:v3.40.0`) instead of `:latest`:

- **Supply chain safety** — A compromised `:latest` tag could inject malicious code into your DeFi environment
- **Reproducibility** — Your setup works the same today as it will in 6 months
- **Controlled updates** — You choose when to upgrade by reviewing changelogs first

## Quick start

### 1. Clone and configure

```bash
git clone https://github.com/malaface/secure-defi-browser.git
cd secure-defi-browser

cp .env.example .env
# Edit .env with your credentials
```

### 2. Get your VPN credentials

**ProtonVPN (recommended):**
1. Go to [ProtonVPN Account](https://account.protonvpn.com/) > **WireGuard** > **Create credentials**
2. Copy the private key and address into your `.env`

**Other providers:** Check [Gluetun's provider list](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers) for provider-specific env vars.

### 3. Set up Cloudflare Tunnel (optional but recommended)

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) > **Networks** > **Tunnels**
2. Click **Create a tunnel** > name it (e.g., `secure-browser`)
3. Copy the tunnel token into `CLOUDFLARE_TUNNEL_TOKEN` in your `.env`
4. Add a **Public Hostname**:
   - Subdomain: `defi` (or whatever you prefer)
   - Domain: your Cloudflare domain
   - Service type: `HTTPS`
   - URL: `secure-browser-gluetun:6901`
   - Under **TLS** > enable **No TLS Verify** (required for KasmVNC's self-signed cert)
5. **MANDATORY:** Go to **Access** > **Applications** > create a policy for your hostname

> **Why `No TLS Verify`?** KasmVNC uses a self-signed HTTPS certificate. The cloudflared agent needs to skip TLS verification for the internal connection. This is safe because the public-facing connection (user → Cloudflare) uses Cloudflare's valid certificate, and Zero Trust Access is the real authentication layer.

### 4. Start the stack

```bash
docker compose up -d
```

### 5. Verify everything works

```bash
# Check Gluetun connected to VPN
docker logs secure-browser-gluetun

# Verify exit IP is from VPN, not your real IP
docker exec secure-browser-gluetun wget -qO- http://ip-api.com/line/?fields=query,country

# Check all containers are running
docker compose ps
```

### 6. Access the browser

**Locally:** Open **https://localhost:6901** — accept the self-signed certificate, log in with your `BROWSER_USER`/`BROWSER_PASSWORD`.

**Remotely:** Open your Cloudflare hostname (e.g., `https://defi.yourdomain.com`) — authenticate through Cloudflare Access first, then log in to KasmVNC.

**Verify VPN inside the browser:** Navigate to [https://ipleak.net](https://ipleak.net) — it should show the VPN server's IP and country, not yours.

## Updating

### Check for new versions

- **Brave:** [kasmtech/workspaces-images releases](https://github.com/kasmtech/workspaces-images/releases)
- **Gluetun:** [qdm12/gluetun releases](https://github.com/qdm12/gluetun/releases)

### Update process

```bash
# 1. Review the changelog for breaking changes

# 2. Update the image tag in docker-compose.yml
#    Example: kasmweb/brave:1.14.0 → kasmweb/brave:1.15.0

# 3. Pull the new images
docker compose pull

# 4. Recreate containers with new images (volumes are preserved)
docker compose up -d

# 5. Verify VPN is still working
docker exec secure-browser-gluetun wget -qO- http://ip-api.com/line/?fields=query,country

# 6. Open https://localhost:6901 and verify everything works
```

Your browser profile (bookmarks, extensions, wallet data) is stored in a Docker volume and **survives updates**.

### Rollback

If something breaks after an update:

```bash
# 1. Revert the image tag in docker-compose.yml to the previous version

# 2. Recreate with the old image
docker compose up -d
```

## Service management

```bash
# Start all services (VPN + Browser + Tunnel)
docker compose up -d

# Stop all services (preserves data)
docker compose down

# View logs
docker compose logs -f              # All services
docker compose logs -f gluetun      # VPN only
docker compose logs -f brave        # Browser only
docker compose logs -f cloudflared  # Tunnel only

# Restart after config change
docker compose down && docker compose up -d

# Check resource usage
docker stats secure-browser-gluetun secure-browser-brave secure-browser-cloudflared

# Full cleanup (WARNING: deletes browser profile and VPN data)
docker compose down -v
```

## Security notes

- The browser port (`6901`) is bound to `127.0.0.1` — it is **not accessible** from other machines on your network
- Gluetun's firewall (`FIREWALL_INPUT_PORTS`) only allows port `6901` inbound through the VPN interface
- `FIREWALL_OUTBOUND_SUBNETS` allows Docker-internal communication (needed for cloudflared to reach gluetun)
- Resource limits prevent the browser from consuming all host memory (2GB cap for Brave, 256MB for Gluetun/cloudflared)
- Log rotation is configured on all services to prevent disk exhaustion
- The Cloudflare Tunnel token is the only credential needed — no config files or JSON credentials to manage

## File structure

```
secure-defi-browser/
├── docker-compose.yml   # All 3 services: Gluetun + Brave + cloudflared
├── .env.example         # Template for all credentials
├── .gitignore           # Excludes .env and secrets
└── README.md
```

## License

MIT
