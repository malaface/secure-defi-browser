# Contributing to Secure DeFi Browser

Thanks for your interest in contributing! This project values security, simplicity, and reliability.

## How to Contribute

### Reporting Bugs

1. Check [existing issues](https://github.com/malaface/secure-defi-browser/issues) to avoid duplicates
2. Open a new issue with:
   - Clear description of the problem
   - Steps to reproduce
   - Expected vs actual behavior
   - Docker and OS version (`docker --version`, `uname -a`)
   - Relevant container logs (redact any credentials)

**Important:** Never include credentials, tokens, private keys, or personal information in bug reports.

### Suggesting Features

Open an issue with the `enhancement` label. Describe:
- What problem does it solve?
- How should it work?
- Does it add complexity? (simpler is better)

### Submitting Pull Requests

1. **Fork** the repository
2. **Create a branch** from `main`:
   ```bash
   git checkout -b feature/your-feature-name
   ```
3. **Make your changes** — keep them focused on a single issue
4. **Test locally:**
   ```bash
   # Verify all 4 services start correctly
   docker compose up -d
   docker compose ps

   # Verify VPN works
   docker exec secure-browser-gluetun wget -qO- http://ip-api.com/line/?fields=query,country

   # Verify KasmVNC is accessible
   curl -sk -o /dev/null -w "%{http_code}" https://localhost:6901
   ```
5. **Commit** with a clear message:
   ```bash
   git commit -m "feat: short description of change"
   ```
6. **Push** and open a PR against `main`

### PR Requirements

- [ ] All 4 containers start and pass healthchecks
- [ ] VPN tunnel connects and routes traffic correctly
- [ ] No credentials, tokens, or personal data in the diff
- [ ] `.env.example` updated if new environment variables are added
- [ ] `README.md` updated if user-facing behavior changes
- [ ] Image versions are pinned (no `:latest` tags except cloudflared)

## Code Style

- **docker-compose.yml**: Use comments to explain non-obvious decisions. Group services logically with section headers.
- **Caddyfile**: Keep it minimal. Document any `header_up` or transport options.
- **Documentation**: Write for someone who has never used Docker or VPNs before. Be explicit about which commands to run and in what order.

## What We Look For

**Good contributions:**
- Security improvements (additional hardening, better defaults)
- Support for more VPN providers (with tested configurations)
- Bug fixes with clear reproduction steps
- Documentation improvements (troubleshooting, clearer setup instructions)
- Resource optimization (lower memory limits, better healthchecks)

**We will likely decline:**
- Features that add significant complexity without clear security benefit
- Changes that require additional services beyond the current 4-container stack
- Provider-specific customizations that should live in `.env` or user config
- Cosmetic changes without functional impact

## Security Vulnerabilities

**Do NOT open a public issue for security vulnerabilities.** See [SECURITY.md](SECURITY.md) for responsible disclosure instructions.

## License

By contributing, you agree that your contributions will be licensed under the MIT License.
