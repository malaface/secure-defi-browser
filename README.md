# Secure DeFi Browser

A hardened, VPN-routed remote desktop for DeFi and Web3 operations. All traffic exits through a WireGuard VPN tunnel — your real IP is never exposed to the sites you visit.

```
  User (remote)
       |
       v
+------------------+
|  Cloudflare      |  Layer 1: Zero Trust Access (email OTP / SSO)
|  Access          |  Only authorized users reach the server
+--------+---------+
         |
         v
+------------------+
|  cloudflared     |  Outbound tunnel (no ports exposed to internet)
|                  |  Points to: http://caddy:8443
+--------+---------+
         |
         v
+------------------+
|  Caddy           |  Layer 2: HTTP Basic Auth (username + password)
|  :8443           |  Reverse proxy to KasmVNC
+--------+---------+
         |
         v
+------------------+
|  Gluetun         |  VPN Gateway (WireGuard)
|  :6901           |  Kill switch: if VPN drops, all traffic is blocked
+--------+---------+
         | network_mode: "service:gluetun"
         v
+------------------+
|  Ubuntu Desktop  |  Full XFCE desktop - install any browser + wallets
|  (KasmVNC)       |  Real IP never exposed
+------------------+
```

## Why Ubuntu Desktop instead of a single-browser image?

Previous versions used `kasmweb/brave:1.14.0` (Brave-only). We migrated to `kasmweb/ubuntu-jammy-desktop:1.16.1` for these reasons:

- **Multi-browser flexibility** — DeFi dApps have variable compatibility. Some work better on Chrome, others on Firefox or Brave. With a full desktop you can install any browser and switch as needed
- **Wallet extension freedom** — Many protocols require MetaMask or specific wallet extensions that may not be available or work well in Brave's built-in wallet. Install any extension on any browser
- **Full desktop environment** — File manager, terminal, calculator, screenshot tools, and other utilities needed during trading sessions
- **Persistent workspace** — The entire `/home/kasm-user` directory is stored in a Docker volume. Browser profiles, bookmarks, wallet data, and desktop settings survive restarts and updates

The security architecture is identical — `network_mode: "service:gluetun"` forces all traffic through the VPN regardless of which browser you use.

### Gluetun over running VPN on the host

- **Container-level isolation** — Only the desktop goes through the VPN; your other services keep their normal routing
- **Kill switch built-in** — If the VPN drops, Gluetun's firewall blocks all traffic automatically. No IP leak possible
- **60+ VPN providers** — Works with ProtonVPN, Mullvad, NordVPN, Surfshark, IVPN, and [many more](https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers)
- **`network_mode: "service:gluetun"`** — The desktop container has no network interface of its own. It physically cannot bypass the VPN

### Caddy as auth proxy

- **Solves dual Basic Auth** — KasmVNC and Cloudflare both use Basic Auth. The browser can only send one `Authorization` header. Caddy handles its own auth, then injects KasmVNC credentials automatically via `header_up`
- **Lightweight** — Alpine-based, 128MB memory limit
- **No TLS management needed** — Cloudflare handles public TLS; Caddy runs plain HTTP internally

### KasmVNC over traditional VNC

- **Browser-based access** — Connect via any browser at `https://localhost:6901`, no VNC client needed
- **HTTPS by default** — Self-signed TLS out of the box, encrypted connection even locally
- **Better performance** — WebSocket-based streaming is faster and more efficient than classic VNC protocols
- **Clipboard support** — Copy/paste works between your local machine and the remote desktop

### Cloudflare Tunnel over port forwarding

- **No open ports** — Your server exposes nothing to the internet; the tunnel connects outbound to Cloudflare
- **Zero Trust Access** — Enforce email/OTP/SSO authentication before anyone can reach the desktop
- **DDoS protection** — Cloudflare's network sits in front of your service
- **Token-based setup** — Single environment variable, no config files or credentials to manage

### Pinned versions over `latest`

All images use fixed version tags (`kasmweb/ubuntu-jammy-desktop:1.16.1`, `qmcgaw/gluetun:v3.41.1`) instead of `:latest`:

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

### 3. Configure Caddy Basic Auth

```bash
# Generate a bcrypt hash for your chosen password
docker run --rm caddy:2-alpine caddy hash-password --plaintext 'YOUR_PASSWORD'
```

Edit `Caddyfile`:
- Replace `PASTE_YOUR_CADDY_HASH_HERE` with the generated hash (includes the `$2a$14$...` prefix)
- Optionally change the username `defi` to whatever you prefer

Then generate the KasmVNC auth header:

```bash
# YOUR_KASMVNC_PASSWORD must match BROWSER_PASSWORD in your .env
echo -n "kasm_user:YOUR_KASMVNC_PASSWORD" | base64
```

Replace `PASTE_YOUR_KASMVNC_BASE64_HERE` in the Caddyfile with the output.

### 4. Set up Cloudflare Tunnel (optional but recommended)

1. Go to [Cloudflare Zero Trust Dashboard](https://one.dash.cloudflare.com/) > **Networks** > **Tunnels**
2. Click **Create a tunnel** > name it (e.g., `secure-browser`)
3. Copy the tunnel token into `CLOUDFLARE_TUNNEL_TOKEN` in your `.env`
4. Add a **Public Hostname**:
   - Subdomain: `defi` (or whatever you prefer)
   - Domain: your Cloudflare domain
   - Service type: `HTTP`
   - URL: `secure-browser-caddy:8443`
5. **MANDATORY:** Go to **Access** > **Applications** > create a policy for your hostname

### 5. Start the stack

```bash
docker compose up -d
```

### 6. Verify everything works

```bash
# Check Gluetun connected to VPN
docker logs secure-browser-gluetun

# Verify exit IP is from VPN, not your real IP
docker exec secure-browser-gluetun wget -qO- http://ip-api.com/line/?fields=query,country

# Check all containers are running
docker compose ps
```

### 7. Access the desktop

**Locally:** Open **https://localhost:6901** — accept the self-signed certificate, log in with your `BROWSER_PASSWORD`.

**Remotely:** Open your Cloudflare hostname (e.g., `https://defi.yourdomain.com`) — authenticate through Cloudflare Access first, then enter Caddy Basic Auth credentials. KasmVNC auth is handled automatically by Caddy.

**Verify VPN inside the desktop:** Open a browser and navigate to [https://ipleak.net](https://ipleak.net) — it should show the VPN server's IP and country, not yours.

### 8. Install your browsers and wallets

Once inside the desktop, open a terminal and install what you need:

```bash
# Brave
sudo curl -fsSLo /usr/share/keyrings/brave-browser-archive-keyring.gpg https://brave-browser-apt-release.s3.brave.com/brave-browser-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/brave-browser-archive-keyring.gpg] https://brave-browser-apt-release.s3.brave.com/ stable main" | sudo tee /etc/apt/sources.list.d/brave-browser-release.list
sudo apt update && sudo apt install -y brave-browser

# Chrome
wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb
sudo dpkg -i google-chrome-stable_current_amd64.deb
sudo apt --fix-broken install -y

# Firefox is pre-installed
```

All installations persist across restarts thanks to the `desktop_home` volume.

## Updating

### Check for new versions

- **Ubuntu Desktop:** [kasmtech/workspaces-images releases](https://github.com/kasmtech/workspaces-images/releases)
- **Gluetun:** [qdm12/gluetun releases](https://github.com/qdm12/gluetun/releases)

### Update process

```bash
# 1. Review the changelog for breaking changes

# 2. Update the image tag in docker-compose.yml
#    Example: kasmweb/ubuntu-jammy-desktop:1.16.1 -> kasmweb/ubuntu-jammy-desktop:1.17.0

# 3. Pull the new images
docker compose pull

# 4. Recreate containers with new images (volumes are preserved)
docker compose up -d

# 5. Verify VPN is still working
docker exec secure-browser-gluetun wget -qO- http://ip-api.com/line/?fields=query,country

# 6. Open https://localhost:6901 and verify everything works
```

Your desktop profile (browsers, bookmarks, extensions, wallet data) is stored in a Docker volume and **survives updates**.

### Rollback

If something breaks after an update:

```bash
# 1. Revert the image tag in docker-compose.yml to the previous version

# 2. Recreate with the old image
docker compose up -d
```

## Service management

```bash
# Start all services (VPN + Desktop + Caddy + Tunnel)
docker compose up -d

# Stop all services (preserves data)
docker compose down

# View logs
docker compose logs -f              # All services
docker compose logs -f gluetun      # VPN only
docker compose logs -f desktop      # Desktop only
docker compose logs -f caddy        # Reverse proxy
docker compose logs -f cloudflared  # Tunnel only

# Restart after config change
docker compose down && docker compose up -d

# Check resource usage
docker stats secure-browser-gluetun secure-browser-desktop secure-browser-caddy secure-browser-cloudflared

# Full cleanup (WARNING: deletes desktop profile and VPN data)
docker compose down -v
```

## Resource recommendations

The desktop container is limited to **3 GB RAM** and **2 CPU cores** by default. Tips to stay within limits:

- Close browser tabs you are not actively using — each tab is a separate process consuming 50-200 MB
- Avoid running multiple browsers simultaneously unless necessary
- If you need more resources, adjust the `deploy.resources.limits` in `docker-compose.yml`

## Troubleshooting

### VPN not connecting

```bash
docker logs secure-browser-gluetun 2>&1 | tail -30
```

- **i/o timeout errors:** WireGuard handshake is failing silently. Verify your `WIREGUARD_PRIVATE_KEY` and `WIREGUARD_ADDRESSES` match what your VPN provider gave you. Try regenerating credentials.
- **`VPN_SERVER_COUNTRIES` must use full country names:** `Switzerland`, `United States`, `Netherlands` — NOT `ch#1`, `us-free#56`.

### XFCE desktop crashes inside container

If you see `Unable to load a failsafe session` or `Permission denied` on `.config/xfce4`:

```bash
# The volume mount may have corrupted permissions. Clean restart:
docker compose down -v    # WARNING: deletes desktop profile
docker compose up -d      # Fresh start with correct permissions
```

The `desktop_home` volume mounts at `/home/kasm-user` (the full home directory). Mounting at a subdirectory causes Docker to create parent directories as root, breaking XFCE's ability to create its config.

### 401 Unauthorized through Cloudflare Tunnel

This happens when two Basic Auth layers compete for the same HTTP `Authorization` header. The Caddy reverse proxy solves this by:
1. Validating its own Basic Auth credentials
2. Replacing the `Authorization` header with KasmVNC credentials via `header_up`

Make sure your Cloudflare Tunnel points to `http://secure-browser-caddy:8443` (not directly to gluetun:6901).

### Healthcheck failures on Gluetun v3.41+

Gluetun v3.41+ requires authentication for `/v1/publicip/ip`. The healthcheck uses `/v1/vpn/status` instead, which works without auth.

## Security notes

- The desktop port (`6901`) is bound to `127.0.0.1` — it is **not accessible** from other machines on your network
- Gluetun's firewall (`FIREWALL_INPUT_PORTS`) only allows port `6901` inbound through the VPN interface
- `FIREWALL_OUTBOUND_SUBNETS` allows Docker-internal communication (needed for caddy/cloudflared to reach gluetun)
- Resource limits prevent the desktop from consuming all host memory (3GB cap for Desktop, 256MB for Gluetun/cloudflared)
- Log rotation is configured on all services to prevent disk exhaustion
- `DOT=off` disables DNS over TLS inside Gluetun to prevent timeout issues with some VPN providers. DNS queries go to `9.9.9.9` (Quad9) in plaintext through the VPN tunnel, which is still encrypted end-to-end by WireGuard
- The Cloudflare Tunnel token is the only credential needed — no config files or JSON credentials to manage

## File structure

```
secure-defi-browser/
├── docker-compose.yml   # All 4 services: Gluetun + Desktop + Caddy + cloudflared
├── Caddyfile            # Reverse proxy config with Basic Auth
├── .env.example         # Template for all credentials
├── .gitignore           # Excludes .env and secrets
├── SECURITY.md          # Security policy and vulnerability reporting
├── CONTRIBUTING.md      # Contribution guidelines
└── README.md
```

## License

MIT
