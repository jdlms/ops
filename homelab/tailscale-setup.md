# MacBook Network & Tailscale Setup

## Overview

This MacBook runs FortiClient (corporate requirement) alongside Tailscale for homelab access. The App Store version of Tailscale conflicts with FortiClient's network extensions, so the Homebrew standalone version is required.

## Current Setup

### Tailscale (Homebrew Standalone)

The standalone version uses a userspace `tailscaled` daemon instead of macOS Network Extensions, which coexists with FortiClient.

**Installation:**
```bash
brew install tailscale
sudo brew services start tailscale
tailscale up
```

**Post-reboot:** Tailscale must be started manually with root privileges:
```bash
sudo brew services start tailscale
```

**Upgrade note:** Due to root ownership, upgrades require:
```bash
sudo brew services stop tailscale
sudo brew upgrade tailscale
sudo brew services start tailscale
```

### Network Locations

Two macOS network locations are configured:

| Location | Use Case | Tailscale DNS | Notes |
|----------|----------|---------------|-------|
| Home | Home network via Pi-hole | `accept-dns=true` | DNS queries go through Tailscale to Pi-hole |
| Automatic | Corporate/default | `accept-dns=false` | Uses local network DNS, FortiClient handles corporate routing |

### Pi-hole Integration

Home network routes DNS through a Pi-hole via Tailscale. When `accept-dns=true`, Tailscale uses the DNS settings from your tailnet (presumably pointing to the Pi-hole's Tailscale IP).

## Current Aliases

```bash
alias network-home="osascript -e 'do shell script \"networksetup -switchtolocation \\\"Home\\\"\" with administrator privileges' && tailscale set --accept-dns=true"
alias network-automatic="osascript -e 'do shell script \"networksetup -switchtolocation \\\"Automatic\\\"\" with administrator privileges' && tailscale set --accept-dns=false"
```

## Proposed Alias Refactor

Separate network location switching from Tailscale/VPN control for more flexibility:

```bash
# --- Network Location ---
alias net-home="osascript -e 'do shell script \"networksetup -switchtolocation \\\"Home\\\"\" with administrator privileges'"
alias net-auto="osascript -e 'do shell script \"networksetup -switchtolocation \\\"Automatic\\\"\" with administrator privileges'"

# --- Tailscale ---
alias tailscale-start="sudo brew services start tailscale"  # Run after reboot
alias ts-dns-on="tailscale set --accept-dns=true && tailscale dns status"
alias ts-dns-off="tailscale set --accept-dns=false && tailscale dns status"
alias ts-status="tailscale status"
alias ts-up="tailscale up"
alias ts-down="tailscale down"

# --- Compound: Common Scenarios ---
# At home, full Tailscale DNS through Pi-hole
alias mode-home="net-home && ts-dns-on"

# Corporate network, Tailscale connected but not handling DNS
alias mode-work="net-auto && ts-dns-off"

# Home network but need FortiClient VPN (Tailscale stays connected, DNS off to avoid conflicts)
alias mode-home-vpn="net-home && ts-dns-off"
```

## FortiClient + Tailscale Coexistence

When using FortiClient VPN while at home:

1. **Routing conflicts:** FortiClient may push routes that overlap with Tailscale. Both can run, but be aware traffic may route unexpectedly.

2. **DNS conflicts:** If both Tailscale (`accept-dns=true`) and FortiClient push DNS, you'll have issues. Disable Tailscale DNS when using FortiClient: `tailscale set --accept-dns=false`

3. **Split tunnel consideration:** If FortiClient is configured for split tunneling, Tailscale traffic to your homelab IPs (100.x.x.x) should still work. If FortiClient forces all traffic through the VPN, Tailscale connectivity may break.

**Test your setup:**
```bash
# Check current routes
netstat -rn | head -30

# Check Tailscale connectivity
tailscale ping <your-pihole-tailscale-ip>

# Check DNS resolution
tailscale dns status
scutil --dns | head -40
```

## Troubleshooting

### Tailscale won't start after reboot
```bash
sudo brew services start tailscale
```

### "failed to connect to local Tailscale service"
```bash
pgrep -fl tailscaled  # Should show the daemon
sudo brew services restart tailscale
```

### DNS not resolving through Pi-hole
```bash
tailscale dns status           # Check Tailscale DNS config
tailscale set --accept-dns=true
scutil --dns | grep nameserver # Verify system DNS
```

### Check for conflicting network extensions
```bash
systemextensionsctl list
```

## Files & Paths

| Item | Path |
|------|------|
| tailscale binary | `/opt/homebrew/bin/tailscale` |
| tailscaled binary | `/opt/homebrew/opt/tailscale/bin/tailscaled` |
| tailscaled logs | `/opt/homebrew/var/log/tailscaled.log` |
| Tailscale state | `/var/lib/tailscale/` (root-owned) |