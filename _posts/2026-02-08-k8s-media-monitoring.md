---
title: "Monitoring a Kubernetes Media Stack with Grafana"
subtitle: "Production-grade monitoring for Plex and friends: Grafana dashboards, Prometheus alerts, and email notifications."
date: 2026-02-08
tags:
  - kubernetes
  - grafana
  - prometheus
  - monitoring
  - alerting
categories:
  - Kubernetes Homelab
readtime: true
---
Production-grade monitoring for a Kubernetes media stack: custom Grafana dashboards, intelligent Prometheus alerts, and formatted email notifications via Gmail SMTP.
<!--more-->

> *"Hope is not a strategy."* - Everyone who checked the logs too late

---

## The Problem

Running 8+ media apps on Kubernetes without monitoring is flying blind. Is Plex streaming or just "Running"? Which app is eating RAM? Did Sonarr crash at 3am?

You need observability before users complain.

---

## Architecture

```
Prometheus
  ├─► Scrapes metrics (cAdvisor, kube-state-metrics, kubelet)
  ├─► Evaluates alert rules every 30s
  └─► Sends alerts to Alertmanager
       └─► Routes to Gmail SMTP with HTML templates

Grafana
  ├─► Queries Prometheus for metrics
  ├─► Displays custom dashboards
  └─► Includes pre-built K8s dashboards
```

---

## Deployment

### kube-prometheus-stack

Single Helm chart includes everything: Prometheus, Grafana, Alertmanager, node-exporter, kube-state-metrics.

```bash
helm repo add prometheus-community \
  https://prometheus-community.github.io/helm-charts

helm install kube-prometheus-stack \
  prometheus-community/kube-prometheus-stack \
  -n media \
  -f apps/monitoring/values.yaml
```

### Configuration

```yaml
# apps/monitoring/values.yaml
grafana:
  enabled: true
  adminPassword: admin
  
  initChownData:
    enabled: false  # Causes permission errors on Talos
  
  ingress:
    enabled: true
    ingressClassName: traefik
    hosts:
      - grafana.media.lan
    tls:
      - hosts:
          - grafana.media.lan
  
  persistence:
    enabled: true
    storageClassName: local-path  # SQLite needs fast I/O
    size: 5Gi
  
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi
  
  defaultDashboardsEnabled: true

prometheus:
  enabled: true
  
  prometheusSpec:
    retention: 7d
    
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-appdata  # See storage notes below
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
    
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 2Gi

alertmanager:
  enabled: true
  
  config:
    global:
      resolve_timeout: 5m
      smtp_from: 'your-email@gmail.com'
      smtp_smarthost: 'smtp.gmail.com:587'
      smtp_auth_username: 'your-email@gmail.com'
      smtp_auth_password: 'your-gmail-app-password'
      smtp_require_tls: true
    
    route:
      group_by: ['alertname', 'cluster', 'service']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
      receiver: 'email'
    
    receivers:
      - name: 'email'
        email_configs:
          - to: 'your-email@gmail.com'
            send_resolved: true
            headers:
              Subject: '{{ if eq .Status "firing" }}🚨{{ else }}✅{{ end }} [{{ .Status | toUpper }}] {{ .GroupLabels.alertname }}'
            html: |
              <!DOCTYPE html>
              <html>
              <head>
                <style>
                  .critical { background-color: #ffebee; border-left: 5px solid #c62828; }
                  .warning { background-color: #fff3e0; border-left: 5px solid #f57c00; }
                  .resolved { background-color: #e8f5e9; border-left: 5px solid #388e3c; }
                </style>
              </head>
              <body>
                <h1>{{ if eq .Status "firing" }}🚨 Alert Firing{{ else }}✅ Alert Resolved{{ end }}</h1>
                {{ range .Alerts }}
                <div class="alert {{ .Labels.severity }}">
                  <h2>{{ .Labels.alertname }}</h2>
                  <p><strong>Summary:</strong> {{ .Annotations.summary }}</p>
                  <p><strong>Description:</strong> {{ .Annotations.description }}</p>
                  <p><strong>Started:</strong> {{ .StartsAt }}</p>
                </div>
                {{ end }}
              </body>
              </html>
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Get your Gmail app password at https://myaccount.google.com/apppasswords. Requires 2FA enabled on your Google account.

</div>
</div>

---

## Storage Decisions

### Why NFS for Prometheus?

Tried `local-path` first. Got permission errors:

```
Error: open /prometheus/queries.active: permission denied
```

**Root cause:** Talos Linux's strict security + Prometheus's file permissions = conflict

**Solution:** Use NFS storage (more lenient permissions)

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-appdata  # Not local-path
```

**Trade-off:** Historical metrics aren't latency-sensitive. NFS network overhead is acceptable.

### Why local-path for Grafana?

Grafana uses SQLite. Random I/O needs fast storage:

```yaml
grafana:
  persistence:
    storageClassName: local-path  # Fast SSD
```

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

This is the same split as the media apps: databases on local-path, large sequential data on NFS.

</div>
</div>

---

## Custom Dashboard

Grafana ships with excellent Kubernetes dashboards. But I wanted **media stack-specific** panels.

### Auto-Import via ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-media-stack
  namespace: media
  labels:
    grafana_dashboard: "1"  # Sidecar watches for this label
data:
  media-stack-overview.json: |
    {
      "title": "Media Stack Overview",
      "panels": [
        {
          "title": "CPU Usage by App",
          "targets": [{
            "expr": "sum(rate(container_cpu_usage_seconds_total{namespace=\"media\", pod=~\"plex.*|sonarr.*|radarr.*\"}[5m])) by (pod)"
          }]
        },
        {
          "title": "Memory Usage by App",
          "targets": [{
            "expr": "sum(container_memory_working_set_bytes{namespace=\"media\", pod=~\"plex.*|sonarr.*|radarr.*\"}) by (pod)"
          }]
        },
        {
          "title": "Pod Status",
          "targets": [{
            "expr": "kube_pod_status_ready{namespace=\"media\", condition=\"true\", pod=~\"plex.*|sonarr.*|radarr.*\"}"
          }]
        },
        {
          "title": "Network I/O",
          "targets": [{
            "expr": "sum(rate(container_network_receive_bytes_total{namespace=\"media\", pod=~\"plex.*|sabnzbd.*\"}[5m])) by (pod)"
          }]
        },
        {
          "title": "Storage Usage",
          "targets": [{
            "expr": "100 * sum(kubelet_volume_stats_used_bytes{persistentvolumeclaim=\"media-data\"}) / sum(kubelet_volume_stats_capacity_bytes{persistentvolumeclaim=\"media-data\"})"
          }]
        }
      ]
    }
```

Deploy:

```bash
kubectl apply -f dashboards/media-stack-overview.yaml
```

Grafana's sidecar container watches for `grafana_dashboard: "1"` labels and imports automatically. No manual clicking.

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Check import status: `kubectl logs -n media -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard`

</div>
</div>

---

## Alert Rules

### Philosophy

Every alert should be:
1. **Actionable** - You can fix it
2. **Urgent** - Needs timely response
3. **Real** - Actually indicates a problem

Don't alert on noise.

### Configuration

```yaml
# monitoring/prometheus-rules.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: media-stack-alerts
  namespace: media
  labels:
    prometheus: kube-prometheus-stack-prometheus
    role: alert-rules
spec:
  groups:
    - name: media-stack
      interval: 30s
      rules:
        # CRITICAL: App down
        - alert: MediaAppPodDown
          expr: |
            kube_pod_status_phase{
              namespace="media",
              phase="Running",
              pod=~"plex.*|sonarr.*|radarr.*|sabnzbd.*"
            } == 0
          for: 2m
          labels:
            severity: critical
          annotations:
            summary: "{{ $labels.pod }} is down"
            description: "Pod has not been running for 2 minutes."
        
        # WARNING: High CPU
        - alert: MediaAppHighCPU
          expr: |
            sum(rate(container_cpu_usage_seconds_total{
              namespace="media",
              pod=~"plex.*|sonarr.*|radarr.*|sabnzbd.*"
            }[5m])) by (pod) > 0.8
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High CPU on {{ $labels.pod }}"
            description: "Pod using {{ $value | humanizePercentage }} CPU for 5+ minutes."
        
        # WARNING: High Memory
        - alert: MediaAppHighMemory
          expr: |
            sum(container_memory_working_set_bytes{
              namespace="media",
              pod=~"plex.*|sonarr.*|radarr.*"
            }) by (pod) /
            sum(container_spec_memory_limit_bytes{
              namespace="media",
              pod=~"plex.*|sonarr.*|radarr.*"
            }) by (pod) > 0.85
          for: 5m
          labels:
            severity: warning
          annotations:
            summary: "High memory on {{ $labels.pod }}"
            description: "Pod using {{ $value | humanizePercentage }} of memory limit."
        
        # WARNING: Storage full
        - alert: MediaStorageFull
          expr: |
            100 * sum(kubelet_volume_stats_used_bytes{
              namespace="media",
              persistentvolumeclaim="media-data"
            }) / sum(kubelet_volume_stats_capacity_bytes{
              namespace="media",
              persistentvolumeclaim="media-data"
            }) > 85
          for: 10m
          labels:
            severity: warning
          annotations:
            summary: "Media storage almost full"
            description: "NFS storage is {{ $value | humanize }}% full."
        
        # INFO: Container restarted
        - alert: MediaAppRestarting
          expr: |
            increase(kube_pod_container_status_restarts_total{
              namespace="media",
              pod=~"plex.*|sonarr.*|radarr.*"
            }[1h]) > 0
          labels:
            severity: info
          annotations:
            summary: "{{ $labels.pod }} restarted"
            description: "Pod restarted {{ $value }} time(s) in the last hour."
```

Deploy:

```bash
kubectl apply -f monitoring/prometheus-rules.yaml
```

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

The `for: 5m` clause prevents flapping. Plex CPU spikes for 30s during a transcode? Not worth alerting.

</div>
</div>

---

## Gmail SMTP Setup

### Get App Password

1. Enable 2-Step Verification: https://myaccount.google.com/security
2. Generate app password: https://myaccount.google.com/apppasswords
3. Name it "Kubernetes Alerts"
4. Copy the 16-character password

### Test SMTP

```bash
curl --url 'smtp://smtp.gmail.com:587' \
  --ssl-reqd \
  --mail-from 'your-email@gmail.com' \
  --mail-rcpt 'your-email@gmail.com' \
  --user 'your-email@gmail.com:app-password' \
  --upload-file - <<EOF
Subject: Test Alert

This is a test from Alertmanager.
EOF
```

If that succeeds, Alertmanager will work.

---

## Useful PromQL Queries

### Top CPU Consumers

```promql
topk(5, sum(rate(container_cpu_usage_seconds_total{namespace="media"}[5m])) by (pod))
```

### Memory as % of Limit

```promql
100 * container_memory_working_set_bytes / container_spec_memory_limit_bytes
```

### Pods with >5 Restarts (24h)

```promql
increase(kube_pod_container_status_restarts_total{namespace="media"}[24h]) > 5
```

### Storage Trend (7d)

```promql
kubelet_volume_stats_used_bytes{persistentvolumeclaim="media-data"}[7d]
```

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Run these in Grafana's Explore tab (left sidebar) to test queries before adding to dashboards.

</div>
</div>

---

## Troubleshooting

### Prometheus CrashLoopBackOff

**Symptom:**

```
Error: open /prometheus/queries.active: permission denied
```

**Fix:** Use NFS storage

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: nfs-appdata  # Not local-path
```

Talos Linux's strict security conflicts with Prometheus file permissions. NFS bypasses the issue.

### Dashboard Not Auto-Importing

**Check sidecar logs:**

```bash
kubectl logs -n media -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
```

**Verify label:**

```yaml
labels:
  grafana_dashboard: "1"  # Must be string "1", not int 1
```

### Alerts Not Firing

**Check rule status:**

```bash
kubectl port-forward -n media svc/kube-prometheus-stack-prometheus 9090:9090
# Visit http://localhost:9090/rules
```

**Manually trigger test alert:**

```yaml
- alert: TestAlert
  expr: vector(1)  # Always true
  annotations:
    summary: "Test alert"
```

### Email Not Sending

**Check Alertmanager logs:**

```bash
kubectl logs -n media alertmanager-kube-prometheus-stack-alertmanager-0 -c alertmanager | grep -i smtp
```

**Common issues:**
- Wrong app password (16 chars, no spaces)
- 2FA not enabled on Gmail
- Port 587 blocked by firewall

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Test Gmail SMTP directly (curl command above) before debugging Alertmanager. Separate infrastructure from config issues.

</div>
</div>

---

## Alert Tuning

### Start with Too Many

Deploy all alerts, then tune down based on noise. Better to know what's noisy than miss real issues.

### Group Related Alerts

```yaml
route:
  group_by: ['alertname', 'cluster', 'service']
  group_wait: 10s
  group_interval: 10s
```

If 3 pods go down simultaneously, you get **one email** with 3 alerts, not 3 separate emails.

### Use `for:` to Prevent Flapping

```yaml
for: 5m  # Must be true for 5 consecutive minutes
```

Transient CPU spikes don't trigger alerts. Only sustained issues do.

### Send Resolved Notifications

```yaml
send_resolved: true
```

Getting the "all clear" email is satisfying. Always enable it.

---

## Operational Insights

### What Actually Gets Alerted (Last 7 Days)

- **3x** `MediaAppRestarting` (info) - Sonarr updated
- **1x** `MediaAppHighMemory` (warning) - Plex hit 85% but didn't OOM
- **0x** `MediaAppPodDown` (critical) - Stack stable

**Key insight:** Most restarts are planned (updates). Use `severity: info` so you're aware but not paged at 3am.

### Dashboards I Use Daily

1. **Media Stack Overview** - Quick health check
2. **Kubernetes / Compute Resources / Namespace** - When something feels slow
3. **Node Exporter / Nodes** - Host-level investigation

The pre-built Kubernetes dashboards are excellent. Don't reinvent them.

---

## Checklist

Before calling monitoring "done":

- [ ] Grafana accessible at `https://grafana.media.lan`
- [ ] Prometheus `/targets` page shows all scrape targets
- [ ] Custom dashboard auto-imported
- [ ] Alert rules loaded (`/rules` page)
- [ ] Test alert sent to email
- [ ] Resolved alert sent when cleared
- [ ] Dashboard panels show live data

---

## What's Next

**Enhance monitoring:**
- Add app-specific metrics (Plex API, SABnzbd queue)
- Create per-app dashboards
- Set up SLO tracking (99.9% uptime target)

**Improve alerting:**
- Silence windows during maintenance
- Add Slack/Discord integration
- Create runbooks linked to each alert

**External monitoring:**
- Uptime Robot for external checks
- Synthetic monitoring (test Plex stream every 5min)

---

## Related

- [Uptime Kuma](/post/k8s-uptime-kuma/) - Simpler: uptime checks, heartbeat monitors, status page
- [Media Stack Complete](/post/k8s-media-stack-complete/) - Full deploy including this monitoring
- [Media Stack basics](/post/k8s-media-stack/) - Plex, Sonarr, Radarr without Prometheus

## References

- [Source code for this post](https://github.com/YOUR-USERNAME/k8s-media-stack)
- [kube-prometheus-stack chart](https://github.com/prometheus-community/helm-charts/tree/main/charts/kube-prometheus-stack)
- [Prometheus Alerting](https://prometheus.io/docs/alerting/latest/overview/)
- [Grafana Provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/)
- [Building a Kubernetes Media Stack](/post/k8s-media-stack-complete/)
