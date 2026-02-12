---
title: "Network Monitoring with Uptime Kuma on Kubernetes"
subtitle: "Self-hosted uptime monitoring for all your homelab services. Single dashboard, push notifications, zero dependencies."
date: 2026-02-08
tags:
  - kubernetes
  - homelab
  - monitoring
  - helm
  - uptime-kuma
  - networking
categories:
  - Kubernetes Homelab
readtime: true
---
I found out Plex was down from a text message. Not from Uptime Kuma alerting me - from my brother asking why he couldn't watch The Office.
<!--more-->

This was six months into running my media stack. I checked Kubernetes. The pod had been crashlooping for a week. Health checks failing, restart count at 847, and I had no idea because I wasn't monitoring anything.

That's when I finally deployed Uptime Kuma.

---

## What Is It?

Uptime Kuma is monitoring software. Think Pingdom or UptimeRobot, but self-hosted. Single container, SQLite backend, web UI. It checks if your stuff is up and yells at you when it's not.

I monitor everything now:
- HTTP endpoints (all the *arr apps, Proxmox, Pi-hole)
- Network devices (NAS, router, switches)
- TCP ports (DNS, SSH, Kubernetes API)
- SSL certificates (get alerted 14 days before expiration)

When something goes down, I get a Discord notification within 60 seconds. No more finding out from family members.

---

## Deployment

The deployment is straightforward. I'm using the same `bjw-s/app-template` Helm chart I use for everything else. 2GB storage for the SQLite database, minimal resource limits (it barely uses anything), and Traefik ingress.

```bash
helm repo add bjw-s https://bjw-s-labs.github.io/helm-charts/
kubectl create namespace monitoring
helm upgrade --install uptime-kuma bjw-s/app-template -n monitoring -f values.yaml
```

Point DNS at Traefik: `uptime.media.lan` → your Traefik LoadBalancer IP.

First time you load the UI, it'll ask you to create an admin account. Pick a good password - there's no password reset without database access.

## Setting Up Monitors

I set up Discord notifications first. Created a webhook in my Discord server, pasted the URL into Uptime Kuma, tested it. Works perfectly. Every alert shows up in Discord within seconds.

Then I started adding monitors. The UI makes it easy - add a monitor, pick HTTP/TCP/Ping, set the target, choose check interval.

**What I monitor:**
- **HTTP** - All the web UIs (Sonarr, Radarr, Prowlarr, Plex, Proxmox, Pi-hole)
- **TCP** - Critical services (Pi-hole DNS on port 53, K8s API, SSH)
- **Ping** - Network gear (NAS, router, switches)
- **SSL certs** - Anything with HTTPS (gets annoying when certs expire unexpectedly)

Most checks run every 60 seconds. I set retries to 3 before alerting - cuts down on false positives when the network hiccups.

For SSL monitoring, I set the alert threshold to 14 days. Gives me two weeks to renew before things break. Saved me once when cert-manager silently failed and my Let's Encrypt cert was about to expire.

One cool feature: status pages. I created a public status page at `uptime.media.lan/status/homelab` that shows uptime for all my services. Now when family asks if something's down, I just send them the link.

## Backups

Everything lives in a SQLite database at `/app/data/kuma.db`. I run a cronjob that backs it up daily:

```bash
kubectl exec -n monitoring deploy/uptime-kuma -- sqlite3 /app/data/kuma.db ".backup /tmp/backup.db"
kubectl cp monitoring/uptime-kuma-xxx:/tmp/backup.db ./backups/kuma-$(date +%Y%m%d).db
```

Keeps the last 7 backups. I've never needed to restore, but it's there if I do.

## Things I Learned

**Alert fatigue is real.** My first week, I had checks running every 30 seconds with alerts on first failure. My network stuttered for 10 seconds and I got 50 Discord notifications. Now everything checks every 60 seconds with 3 retries before alerting.

**Monitor the monitors.** Uptime Kuma can go down too. I have a separate cron job on another machine that curls the Uptime Kuma homepage every 5 minutes and alerts if it's unreachable. Meta.

**Ping, TCP, and HTTP are different things.** A server can respond to ping (network is up) but the TCP port check fails (service crashed). Or both pass but HTTP returns 500 (app is broken). Use all three for critical services.

**SSL cert monitoring is underrated.** Got alerted 14 days before my cert expired. cert-manager had silently failed weeks earlier. Fixed it before anyone noticed. That one notification paid for the entire setup.

**Status pages are for non-technical users.** My family doesn't understand Kubernetes. They do understand "Plex is down, I'm fixing it" on a status page. Made the public status page and sent them the link. No more "is it down?" texts.

## Resources

It barely uses anything. ~120 MB RAM, <1% CPU idle, maybe 5% CPU when running checks. 50 monitors checking every 60 seconds, total network traffic is negligible.

The SQLite database is currently 480 MB after 8 months with 180-day retention. 2GB storage is plenty.

## Worth It?

Absolutely. Takes 10 minutes to deploy. Saves you from finding out things are broken via angry family members or missed Plex streams.

I sleep better knowing that if something goes down at 3am, I'll get a Discord ping immediately. Before Uptime Kuma, I'd discover outages days later by accident.

Set it up. You'll thank yourself.

---

[Uptime Kuma](https://github.com/louislam/uptime-kuma) • [bjw-s chart](https://bjw-s-labs.github.io/helm-charts/docs/app-template/)
