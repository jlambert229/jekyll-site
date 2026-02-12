---
title: "Running Pi-hole on Synology NAS with Docker"
subtitle: "DNS-based ad blocking on your NAS with automated deployment, version-controlled config, and router integration."
date: 2026-02-08
tags:
  - synology
  - docker
  - pihole
  - dns
  - homelab
  - networking
  - automation
categories:
  - Homelab Networking
readtime: true
---
Network-wide ad blocking and local DNS running on your NAS. Automated deployment, configuration as code, and router integration.
<!--more-->

> *"I should buy a Raspberry Pi for this..."* - Before realizing you already have a server running 24/7

---

## Why Synology Instead of a Pi

Ran Pi-hole on a Raspberry Pi 3 for two years. SD card died - classic Pi death. Lost everything: custom blocklists (oisd, Energized), local DNS overrides (`nas.local`, `router.lan`), 50+ whitelisted domains I'd accumulated (banks, government sites, random things that broke). Spent an evening rebuilding from memory. Got maybe 60% back. The Pi-hole Teleporter export I'd "meaning to set up" never happened.

My Synology was sitting there running 24/7 with RAID, automated backups, and Docker already installed. Why was I maintaining a separate device that could fail?

Moved Pi-hole to the Synology. Same functionality, no dedicated hardware, and actual backups.

> *"Consolidate services onto infrastructure you already maintain. More devices, more failure modes."* - Homelab simplification

---

## Architecture Decision: macvlan + Shim

Pi-hole needs port 53 (DNS). Synology DSM might want port 53 for its own services. Using host networking works but ties you to the NAS's IP and complicates port management.

**macvlan gives Pi-hole its own dedicated IP on your LAN.** Clean separation. Pi-hole gets its own IP, Synology keeps its management IP.

The catch: by design, the host can't communicate with macvlan containers directly. This means the Synology itself can't use Pi-hole for DNS, which breaks:
- DSM's own DNS resolution
- Other Docker containers needing DNS
- Accessing Pi-hole's web UI by hostname from the NAS

**The solution: a macvlan shim.** A small network interface on the host that bridges the gap. *"The answer is 42."* - Hitchhiker's Guide. The question was "why can't the Synology reach its own Pi-hole?" macvlan isolation. The shim is the answer. Took me an afternoon to figure out. Works perfectly.

---

## Repo Structure

Everything's in Git for reproducibility:

```
pihole-synology-docker/
├── docker-compose.yml          # Pi-hole container with macvlan
├── .env.example                # Password and timezone template
├── deploy.sh                   # Automated deployment from workstation
├── verify.sh                   # Comprehensive health checks
├── update-router-dns.sh        # Router DHCP automation (template)
├── macvlan-shim.sh             # Solves host ↔ container networking
├── apply-config.sh             # Push config changes to running instance
├── backup.sh                   # Automated backup (cron-ready)
└── config/
    ├── adlists.csv             # Curated blocklists with tiers
    ├── whitelist.txt           # Pre-emptive false positive fixes
    └── 99-custom.conf          # Local DNS + conditional forwarding
```

Configuration as code. Destroy and rebuild Pi-hole in 5 minutes.

Full source: [pihole-synology-docker on GitHub](https://github.com/search?q=pihole-synology-docker&type=repositories)

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

The repo includes automated deployment scripts. You can deploy manually by following the Docker Compose section, or use the automation. I'll show both approaches.

</div>
</div>

---

## Docker Compose Configuration

Pi-hole gets its own IP via macvlan:

```20:67:pihole-synology-docker/docker-compose.yml
services:
  pihole:
    container_name: pihole
    image: pihole/pihole:latest
    restart: unless-stopped
    hostname: pihole
    networks:
      pihole_net:
        # ════════════════════════════════════════════════════════════
        # CONFIGURATION: Set your Pi-hole's IP address here
        # ════════════════════════════════════════════════════════════
        # Pick an unused IP on your LAN (outside DHCP range).
        # Example: 192.168.1.53 (matching DNS port)
        ipv4_address: 192.168.1.53
    environment:
      TZ: ${TZ:-UTC}
      FTLCONF_webserver_api_password: ${PIHOLE_PASSWORD:-}
      FTLCONF_dns_upstreams: '1.1.1.1;1.0.0.1'    # Cloudflare (change if preferred)
      FTLCONF_dns_listeningMode: all
      DNSMASQ_USER: root    # Required on Synology - see pihole/docker-pi-hole#963
    volumes:
      - /volume1/docker/pihole/etc-pihole:/etc/pihole
      - /volume1/docker/pihole/etc-dnsmasq.d:/etc/dnsmasq.d
    cap_add:
      - NET_ADMIN
    dns:
      - 127.0.0.1       # Pi-hole uses itself for DNS
      - 1.1.1.1          # Fallback during container startup

networks:
  pihole_net:
    driver: macvlan
    driver_opts:
      # ════════════════════════════════════════════════════════════
      # CONFIGURATION: Set your NAS's network interface here
      # ════════════════════════════════════════════════════════════
      # Check with: ssh your-nas "ip addr"
      # Common values: eth0, eth1, bond0
      parent: eth0
    ipam:
      config:
        # ════════════════════════════════════════════════════════════
        # CONFIGURATION: Set your LAN network details here
        # ════════════════════════════════════════════════════════════
        - subnet: 192.168.1.0/24       # Your LAN subnet
          gateway: 192.168.1.1          # Your router/gateway IP
          ip_range: 192.168.1.53/32     # Pi-hole's IP (must match ipv4_address above)
                                         # /32 restricts Docker to exactly this one IP
```

**Key configuration notes:**

- **DNSMASQ_USER: root** - Critical for Synology. DSM has non-standard user permissions. Without this, Pi-hole can't write to its volumes. I spent an hour debugging "permission denied" errors before finding this in a GitHub issue.
- **dns: 127.0.0.1** - Pi-hole uses itself for DNS lookups. Prevents dependency loops during startup.
- **ip_range: /32** - Restricts Docker to only this one IP. Without this, Docker can allocate additional IPs from your subnet, causing conflicts.
- **parent interface** - Must be your primary NIC. Check with `ip addr` on the Synology CLI.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Pick an unused IP outside your DHCP range. I use .53 to match standard DNS (like 8.8.8.8), but use what works for your network. Document it before you forget.

</div>
</div>

Create `.env` file with your settings:

```bash
# .env
PIHOLE_PASSWORD=your-secure-password-here
TZ=America/New_York
```

---

## The macvlan Shim Problem and Solution

Deploy the container and you'll hit the problem immediately: your Synology can't reach Pi-hole.

```bash
# From Synology CLI
ping <PIHOLE_IP>
# No response
```

macvlan isolation. By design, the host can't talk to macvlan containers. This breaks:
- DSM DNS resolution
- Other containers needing DNS
- Accessing Pi-hole web UI by hostname

**The fix: a network shim.** Create a macvlan interface on the host that bridges the gap:

```17:42:pihole-synology-docker/macvlan-shim.sh
set -euo pipefail

# ═════════════════════════════════════════════════════════════════════
# CONFIGURATION (must match docker-compose.yml)
# ═════════════════════════════════════════════════════════════════════
PARENT_IF="eth0"                 # Your NAS's primary network interface
SHIM_IF="macvlan-shim"
SHIM_IP="192.168.1.200"          # Unused LAN IP for the shim (must not conflict!)
PIHOLE_IP="192.168.1.53"         # Pi-hole's macvlan IP (must match docker-compose.yml)
# ═════════════════════════════════════════════════════════════════════

start() {
    echo "Creating macvlan shim: ${SHIM_IF} (${SHIM_IP}) → ${PIHOLE_IP}"

    # Remove existing shim if present (idempotent)
    ip link del "$SHIM_IF" 2>/dev/null || true

    # Create macvlan shim on the same parent as the Pi-hole container
    ip link add "$SHIM_IF" link "$PARENT_IF" type macvlan mode bridge
    ip addr add "${SHIM_IP}/32" dev "$SHIM_IF"
    ip link set "$SHIM_IF" up

    # Route Pi-hole traffic through the shim
    ip route add "${PIHOLE_IP}/32" dev "$SHIM_IF"

    echo "Done. Synology can now reach Pi-hole at ${PIHOLE_IP}"
}
```

Run this after starting the container. Now the Synology can reach Pi-hole.

[View full script](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/macvlan-shim.sh)

**Make it persist across reboots:**

Copy the script to DSM's boot hooks:

```bash
sudo cp macvlan-shim.sh /usr/local/etc/rc.d/
sudo chmod 755 /usr/local/etc/rc.d/macvlan-shim.sh
```

DSM runs scripts in `/usr/local/etc/rc.d/` on boot. Your shim survives reboots.

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

This solution took me an afternoon to figure out. macvlan isolation is well-documented, but the shim solution isn't obvious. I found it buried in Docker networking docs and adapted it for Synology.

</div>
</div>

---

## Automated Deployment (Recommended)

The repo includes [`deploy.sh`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/deploy.sh) - runs from your workstation over SSH:

```bash
./deploy.sh
```

What it does:
1. Preflight checks (SSH, Container Manager running)
2. Creates data directories on NAS
3. Prompts for password and timezone
4. Copies all files to NAS
5. Pulls image and starts container
6. Activates macvlan shim
7. Persists shim in rc.d for reboots

Output shows each step with colored status. Takes ~2 minutes depending on image pull.

```43:50:pihole-synology-docker/deploy.sh
# ─────────────────────────────────────────────
step "1/7" "Preflight checks"
# ─────────────────────────────────────────────
info "Testing SSH to ${NAS_HOST}..."
ssh -o ConnectTimeout=5 -o BatchMode=yes "${NAS_HOST}" "true" 2>/dev/null \
    || fail "Cannot SSH to ${NAS_HOST}. Check key auth and connectivity."
ok "SSH connection"
```

**Verify everything works:**

```bash
./verify.sh
```

Checks:
- Container running without restart loops
- DNS resolution working
- Ad domains blocked
- Web UI reachable
- macvlan shim active

All green? You're ready for network-wide deployment.

[View verify.sh](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/verify.sh)

---

## Configuration Management

The repo includes curated config files. Apply them with [`apply-config.sh`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/apply-config.sh):

```bash
./apply-config.sh
```

### Blocklists (config/adlists.csv)

Three tiers defined in [`config/adlists.csv`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/config/adlists.csv):

```14:27:pihole-synology-docker/config/adlists.csv
# ──────────────────────────────────────────────────────────────
# Essential - start here
# ──────────────────────────────────────────────────────────────

# StevenBlack unified hosts - ads + malware (Pi-hole default)
https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts,1,StevenBlack unified hosts (default)

# OISD Big - one of the most popular all-in-one lists
# Curated from 50+ sources, aggressive dedup, actively maintained
https://big.oisd.nl/domainswild2,1,OISD Big - comprehensive all-in-one

# HaGeZi Multi Pro - fast-growing, excellent maintenance
# Blocks ads, tracking, metrics, telemetry, phishing, malware
https://raw.githubusercontent.com/hagezi/dns-blocklists/main/domains/pro.txt,1,HaGeZi Multi Pro - ads/tracking/malware
```

**Essential** (enabled by default):
- StevenBlack unified hosts (~130k domains)
- OISD Big (~1.5M domains, well-maintained)
- HaGeZi Multi Pro (ads/tracking/malware)

**Recommended** (enabled):
- Firebog curated lists (AdGuard, EasyList, EasyPrivacy)
- Malware/phishing protection (DandelionSprout, DigitalSide)

**Aggressive** (disabled by default):
- HaGeZi Ultimate (very aggressive, expect breakage)
- OISD NSFW (adult content filter)

I started with all tiers enabled. Broke multiple shopping sites. Spent an evening debugging. Now I use Essential + Recommended. Works for 99% of use cases.

### Whitelist (config/whitelist.txt)

Pre-emptive fixes for common false positives. The repo includes ~100 domains that aggressive blocklists often catch:

```10:23:pihole-synology-docker/config/whitelist.txt
# ──────────────────────────────────────────────────────────────
# Microsoft - login, updates, services
# ──────────────────────────────────────────────────────────────
login.microsoftonline.com
login.live.com
outlook.office365.com
products.office.com
c.s-microsoft.com
i.s-microsoft.com
dl.delivery.mp.microsoft.com
geo-prod.do.dsp.mp.microsoft.com
displaycatalog.mp.microsoft.com
sls.update.microsoft.com.akadns.net
fe3cr.delivery.mp.microsoft.com
```

Categories covered:
- Microsoft login/updates
- Apple services and captive portal
- Google safe browsing and fonts
- Amazon/Alexa
- Streaming (Netflix, Spotify, YouTube)
- Gaming (Steam, PlayStation, Xbox)
- Samsung Smart TV services

[View full whitelist](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/config/whitelist.txt)

This is the collective pain of community testing. These domains get blocked by overzealous lists and break things.

### Local DNS (config/99-custom.conf)

Add local hostname resolution via dnsmasq using [`config/99-custom.conf`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/config/99-custom.conf):

```30:47:pihole-synology-docker/config/99-custom.conf
# ──────────────────────────────────────────────────────────────
# Local DNS records - homelab devices
# ──────────────────────────────────────────────────────────────
# Faster than going through the router's DNS. Edit to match your hosts.
#
# Format: host-record=hostname,ip[,ipv6][,ttl]
#         address=/hostname/ip  (wildcards supported)

# Examples (uncomment and customize):
# host-record=nas.lan,192.168.1.2
# host-record=pihole.lan,192.168.1.53
# host-record=router.lan,192.168.1.1

# Wildcard for Kubernetes ingress or other homelab services
# Example: *.homelab.lan → Traefik ingress controller
# address=/.homelab.lan/192.168.1.100

# Example: Specific services
# host-record=plex.homelab,192.168.1.101
```

Also includes conditional forwarding - sends reverse DNS lookups to your router so local hostnames resolve properly.

Apply configuration:

```bash
./apply-config.sh              # Apply everything
./apply-config.sh --adlists    # Only update blocklists
./apply-config.sh --dry-run    # Preview changes
```

---

## Router Integration

Manual DHCP configuration works, but automation is better. The repo includes [`update-router-dns.sh`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/update-router-dns.sh) - a template for EdgeRouter/UniFi/VyOS routers:

```bash
./update-router-dns.sh
```

```35:47:pihole-synology-docker/update-router-dns.sh
# ─────────────────────────────────────────────
step "1/4" "Preflight"
# ─────────────────────────────────────────────
info "Testing SSH to ${ROUTER_HOST}..."
ssh -o ConnectTimeout=5 -o BatchMode=yes "${ROUTER_HOST}" "true" 2>/dev/null \
    || fail "Cannot SSH to ${ROUTER_HOST}. Check key auth and connectivity."
ok "SSH connection"

info "Testing Pi-hole DNS before cutting over..."
if dig +short +timeout=5 +tries=1 @"${PIHOLE_IP}" google.com >/dev/null 2>&1; then
    ok "Pi-hole is resolving queries at ${PIHOLE_IP}"
else
    fail "Pi-hole is NOT responding at ${PIHOLE_IP}. Run ./verify.sh first."
fi
```

What it does:
1. Tests Pi-hole before cutover
2. Shows current DHCP DNS config
3. Updates all DHCP scopes to advertise Pi-hole
4. Commits and saves router config
5. Shows commands for forcing client DHCP renewal

Adapt this template for your router. The pattern works anywhere - SSH in, change DHCP config, commit.

<div class="admonition admonition-warning" markdown="1">
<div class="admonition-title">⚠️ Warning</div>
<div class="admonition-content" markdown="1">

Existing DHCP clients keep their old DNS until lease renewal (typically 24 hours). Force renewal or wait.

</div>
</div>

---

## Backup Strategy

Automated backups via [`backup.sh`](https://github.com/YOUR-USERNAME/pihole-synology-docker/blob/main/backup.sh):

```15:16:pihole-synology-docker/backup.sh
# Cron (daily at 3am):
#   0 3 * * * /volume1/docker/pihole/backup.sh 2>&1 | logger -t pihole-backup
```

Run manually to test:

```bash
sudo /volume1/docker/pihole/backup.sh --verbose
```

Each backup creates two artifacts:
1. **Teleporter export** - Pi-hole's built-in backup (settings, lists, DNS records)
2. **Volume snapshot** - Full tar of config directories (nuclear restore option)

Keeps 14 days of history. Auto-rotates old backups.

**Why two formats?**

Teleporter is portable. Import it via web UI. Works across Pi-hole versions.

Volume snapshot is the nuclear option. Everything on disk. Use it when Teleporter fails or you need byte-for-byte restoration.

I've used both. Teleporter for config changes I regretted. Volume snapshot when I accidentally upgraded to a broken Pi-hole version.

---

## Resource Usage

Tested on a 4-bay NAS (Celeron J-series, 8GB RAM):
- CPU: <1% idle, <5% during blocklist updates
- RAM: ~150MB
- Storage: ~500MB (container + config)

Query response times:
- Cached: <1ms
- Blocked: <1ms  
- Forwarded to upstream: 10-20ms

Your network DNS is now faster than before. Blocked queries don't leave your network. No round trip to an ad server just to get blocked.

---

## Troubleshooting

### macvlan Shim Not Working

**Symptom:** Synology can't ping Pi-hole container.

```bash
ping <PIHOLE_IP>
# Destination Host Unreachable
```

**Check:**

```bash
# Is the shim interface up?
ip addr show macvlan-shim

# Does the route exist?
ip route | grep <PIHOLE_IP>
```

If missing, the rc.d script didn't run. Check:

```bash
ls -l /usr/local/etc/rc.d/macvlan-shim.sh
# Should be executable (755) and owned by root
```

Manually run the shim script to test. If it works, reboot and verify it survives.

### Container Won't Start - Port Conflict

**Symptom:** Container exits immediately with port binding errors.

**Check:** DSM's DNS Server package conflicts with port 53.

```bash
sudo netstat -tlnp | grep :53
```

If DSM DNS Server is running:
1. Package Center → DNS Server → Stop
2. Disable it permanently

macvlan *should* avoid this conflict (separate IP), but sometimes DSM binds to `0.0.0.0:53` which blocks everything.

### Ads Still Showing

**Symptom:** Devices still get ads after cutover.

**Debug from the device:**

```bash
nslookup google.com
```

Look at `Server:` line. Should be your Pi-hole IP. If not:

- **Router DHCP not updated** - Verify router config, force DHCP renewal
- **Device has static DNS** - Check device network settings
- **Device hardcodes DNS** - Chromecast, IoT devices, some smart TVs bypass your DNS

For hardcoded DNS, you need router firewall rules to redirect DNS queries. Different battle.

### Sites Broken After Aggressive Blocklists

**Symptom:** Shopping sites, login pages, or apps randomly fail.

Pi-hole admin → Query Log. Find the blocked domain causing the issue.

**Quick fix:**

```bash
# SSH to Synology
docker exec pihole pihole -w example.com
```

Or add to `config/whitelist.txt` and rerun `./apply-config.sh` for version control.

Common false positives:
- Microsoft telemetry (breaks Windows Update sometimes)
- Apple metrics (breaks iCloud/App Store)
- CDN domains (breaks images/scripts on various sites)

The repo's whitelist.txt has the most common ones pre-loaded.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

When debugging a broken site: Admin panel → Settings → Disable blocking for 5 minutes. Site works now? It's Pi-hole. Check query log to find what was blocked.

</div>
</div>

### DNS Slow After DSM Update

DSM updates can reset network settings. Verify:

```bash
# Container still running?
docker ps | grep pihole

# macvlan shim still active?
ip addr show macvlan-shim

# DSM DNS config?
cat /etc/resolv.conf
```

Fix as needed - restart container, re-run shim script, update DSM DNS settings (Control Panel → Network).

---

## What I Learned

### 1. macvlan + Shim > Host Networking

Most guides use host networking. "Simpler." I tried that. DSM 7.2 updated. Port 53 conflict - Synology's DNS Server package or some internal service wanted 53. Pi-hole couldn't bind. The entire house lost DNS. My spouse's work VPN wouldn't connect (it needed to resolve internal hostnames). "Is the internet down?" Technically no. Practically yes. Took an hour to figure out it was a port conflict, not an outage. The macvlan + shim setup took an afternoon to figure out. It has never conflicted since. The shim isn't obvious. But it solves the problem permanently. Worth the investment.

### 2. Configuration as Code Matters

First time I set up Pi-hole, I clicked through the web UI for an hour. Custom blocklists. Regex rules. Local DNS overrides. Felt productive. Then I upgraded DSM. The container got recreated. Default config. Everything gone. I stood in my kitchen at 10pm rebuilding from memory. I got maybe 60% of it. Now everything's in Git. Blocklists, whitelist, local DNS. Lose the container? `./deploy.sh && ./apply-config.sh`. Five minutes. Never again.

### 3. Automated Backups Are Not Optional

"It's just DNS config. I'll remember." Famous last words. I was testing a regex to block a particularly aggressive ad network. The regex was wrong. It blocked... something critical. Pi-hole broke. No backup. I rebuilt from memory. Missed half my whitelist. Two weeks later I discovered my smart TV couldn't reach its update servers. Whitelisted. Again. Now `backup.sh` runs daily. I've restored twice. Both times I said a silent prayer of thanks to past-me.

### 4. Conservative Blocklists, Then Expand

Day one: enabled every aggressive blocklist. "Maximum ad blocking." Broke the mortgage payment site. Broke the DMV renewal. Broke something my kid needed for school. The house was in revolt. I spent an evening whitelisting domains I'd never heard of. Now I use Essential + Recommended. Blocks ~95% of ads. Doesn't break life. If you want more, enable Aggressive and accept that you will be whitelisting things at 9pm when someone needs to submit homework.

### 5. Local DNS Is the Killer Feature

I set this up for ad blocking. Ads are gone. Great. But the real win? `nas.homelab` instead of 192.168.1.10. `plex.media.lan`. `*.homelab.lan` for every Kubernetes service. I stopped opening my "homelab IP cheat sheet" document. The family stopped asking "what's the address for the movie thing again?" Local DNS is the silent quality-of-life upgrade nobody talks about. This alone justifies Pi-hole.

---

## What's Next

You have network-wide ad blocking running on your NAS. Automated deployment. Version-controlled config. Actual backups. No dedicated hardware, no SD card failures.

**Optional enhancements:**

- **Group management** - Different blocklists for kids' devices vs. adults
- **Regex blacklisting** - Block entire advertising networks with patterns
- **Query analytics** - See what your IoT devices are really phoning home about (prepare to be disturbed)
- **Conditional forwarding** - Better integration with local hostnames from DHCP

The core setup is production-ready. These are optimizations for when you're bored and want to tinker.

---

## Related

- [Vaultwarden on Kubernetes](/2026-02-08-vaultwarden-self-hosted-passwords/) - Another self-hosted service; uses same Traefik + cert-manager pattern
- [K8s Homelab Infrastructure](/2026-02-08-k8s-homelab-infra/) - Local DNS records for `plex.media.lan`, `*.homelab.lan`

## References

- [Pi-hole Docker Hub](https://hub.docker.com/r/pihole/pihole)
- [Pi-hole Documentation](https://docs.pi-hole.net/)
- [StevenBlack hosts](https://github.com/StevenBlack/hosts)
- [OISD blocklist](https://oisd.nl/)
