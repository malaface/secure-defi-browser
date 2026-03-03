# Security Policy

## Supported Versions

| Version | Supported |
|---------|-----------|
| 1.0.x   | Yes       |

## Reporting a Vulnerability

If you discover a security vulnerability in this project, please report it responsibly.

**Do NOT open a public GitHub issue for security vulnerabilities.**

Instead, use [GitHub Security Advisories](https://github.com/malaface/secure-defi-browser/security/advisories/new) to report vulnerabilities privately.

Include:
- Description of the vulnerability
- Steps to reproduce
- Potential impact
- Suggested fix (if any)

We will review the advisory and work with you to address the issue before any public disclosure.

## Security Architecture

This project implements defense in depth with three independent security layers:

### Layer 1: Cloudflare Zero Trust Access
- Email OTP or SSO authentication before reaching the server
- Configured in the Cloudflare Dashboard, not in this repository
- No ports are exposed to the public internet

### Layer 2: Caddy HTTP Basic Auth
- Username + bcrypt-hashed password on the reverse proxy
- Even if Layer 1 is bypassed, credentials are required
- Password hashes use bcrypt with cost factor 14

### Layer 3: KasmVNC Session Password
- Final authentication before controlling the browser
- Handled automatically by Caddy's `header_up` when accessing remotely

### VPN Kill Switch
- Gluetun enforces a strict firewall: if the VPN tunnel drops, all browser traffic is blocked
- The browser container uses `network_mode: "service:gluetun"` — it has no independent network interface and physically cannot bypass the VPN

## Security Best Practices for Users

1. **Never commit `.env` files** — They contain VPN credentials and tunnel tokens
2. **Use strong, unique passwords** for each layer (Caddy, KasmVNC)
3. **Rotate VPN credentials periodically** — Regenerate WireGuard keys in your VPN provider's dashboard
4. **Keep images updated** — Check for security patches in Gluetun and Brave releases
5. **Restrict Cloudflare Access policies** — Only allow specific email addresses, not entire domains
6. **Review the Caddyfile** — Ensure no credentials are committed in plaintext (only bcrypt hashes)
7. **Monitor container logs** — Watch for unauthorized access attempts in Caddy and cloudflared logs

## Scope

The following are **in scope** for security reports:
- Authentication bypass in any of the three security layers
- VPN leak scenarios (browser traffic escaping the VPN tunnel)
- Container escape or privilege escalation
- Credential exposure in configuration files or logs
- Docker Compose misconfigurations that weaken the security model

The following are **out of scope**:
- Vulnerabilities in upstream images (Brave, Gluetun, Caddy, cloudflared) — report those to the respective projects
- Cloudflare Zero Trust configuration issues — those are user-specific
- Social engineering attacks
- Denial of service against the local Docker host
