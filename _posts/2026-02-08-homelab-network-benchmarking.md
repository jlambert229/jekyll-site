---
title: "Homelab Network Performance Testing and Benchmarking"
subtitle: "Baseline your network, storage, and compute performance. iperf3, fio, sysbench - know your limits before hitting them in production."
date: 2026-02-08
tags:
  - homelab
  - networking
  - performance
  - benchmarking
  - storage
  - nfs
  - troubleshooting
categories:
  - Homelab Infrastructure
readtime: true
---
Performance testing for your homelab. Measure network bandwidth, NFS throughput, storage IOPS, and CPU performance. Establish baselines before things break.
<!--more-->

> *"Why is Plex buffering?"* - Questions answered by having performance baselines

---

## Why Benchmark Your Homelab

Ran a media stack for eight months. Plex started buffering during peak hours - usually when 2–3 people were streaming. Was it:
- Network congestion? (1 Gbps switch, but maybe something weird)
- NFS too slow? (Synology DS920+ over Cat6)
- Proxmox CPU bottleneck? (i7-10700, shouldn't be)
- Storage IOPS maxed out? (RAID 5, 4× WD Red)

Had no baseline. *"In the beginning the Universe was created. This has made a lot of people very angry."* - Hitchhiker's Guide. Plex buffering makes people angry too. Benchmarks tell you why. Without them, you're guessing. Spent three evenings guessing, replacing cables, rebooting things.

> *"You can't improve what you don't measure. Baselines turn gut feeling into data."* - SRE performance principle

In one case: 100 Mbps link negotiation on one interface (bad cable or NIC, should've been 1 Gbps). iperf3 would've caught it in 30 seconds. In another: the TV was on 2.4 GHz Wi-Fi and the router was two rooms away - 40 Mbps actual. Moving to 5 GHz fixed it. The homelab was fine. The problem was often *outside* the homelab.

**Benchmarking gives you:**

- **Performance baselines** - Know what "normal" looks like
- **Bottleneck identification** - Find weak links before they cause problems
- **Upgrade justification** - Prove you need 10 GbE (or prove you don't)
- **Troubleshooting data** - "It used to get 900 Mbps, now it's 100 Mbps" beats "it feels slow"

This post covers practical benchmarks for homelab scenarios: network throughput, NFS storage, disk IOPS, CPU performance.

---

## Test Environment

My setup (adjust commands for yours):

```
┌─────────────────────────────────────────────────────────────┐
│  Network: 1 Gbps managed switch (EdgeSwitch 24)             │
│                                                              │
│  Nodes:                                                      │
│   • Proxmox VE (192.168.2.10)          - Intel i7, 32 GB   │
│   • Synology NAS (192.168.2.10)       - Celeron, 8 GB     │
│   • Talos K8s CP (192.168.2.70)        - 2 vCPU, 4 GB      │
│   • Talos K8s W1 (192.168.2.80)        - 2 vCPU, 4 GB      │
│   • Talos K8s W2 (192.168.2.81)        - 2 vCPU, 4 GB      │
│   • EdgeRouter X (192.168.2.1)         - Router            │
│                                                              │
│  Storage: Synology 4-bay NAS (RAID 5, NFS export)          │
└─────────────────────────────────────────────────────────────┘
```

---

## Network Performance (iperf3)

**Tests:** TCP/UDP bandwidth between nodes, identifies link speed issues, switch bottlenecks, and NIC problems.

### Install iperf3

**Proxmox / Linux:**

```bash
apt update && apt install iperf3
```

**Synology:**

Via SSH:

```bash
sudo synopkg install iperf3
# Or use Docker
docker run -d --name iperf3-server --network host mlabbe/iperf3 -s
```

**Kubernetes (for testing pod → NAS bandwidth):**

```bash
kubectl run iperf3-client -n default --rm -it --image=mlabbe/iperf3 -- sh
```

### Test: Proxmox ↔ Synology

**On Synology (server):**

```bash
iperf3 -s
```

**On Proxmox (client):**

```bash
iperf3 -c 192.168.2.10 -t 30 -i 5
```

**Flags:**
- `-t 30` - Test duration (30 seconds)
- `-i 5` - Report interval (every 5 seconds)

**Expected results (1 Gbps network):**

```
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-30.00  sec  3.28 GBytes   940 Mbits/sec   sender
[  5]   0.00-30.00  sec  3.28 GBytes   939 Mbits/sec   receiver
```

**What's normal:**
- **900-940 Mbps:** Perfect (TCP overhead ~6%)
- **700-900 Mbps:** Good (might have light interference)
- **400-700 Mbps:** Poor (duplex mismatch, cable issue, congestion)
- **< 400 Mbps:** Bad (serious problem)

<div class="admonition admonition-tip" markdown="1">
<div class="admonition-title">💡 Tip</div>
<div class="admonition-content" markdown="1">

Run tests at different times (peak vs. off-peak hours). If throughput drops during evenings, you have network congestion (kids streaming, backups running, etc.).

</div>
</div>

### Test: Parallel Streams (Simulate Multi-User Load)

```bash
iperf3 -c 192.168.2.10 -P 10 -t 30
```

**`-P 10`:** 10 parallel streams (simulates multiple users)

**Expected:**
- Aggregate bandwidth should still hit 900+ Mbps
- If it drops significantly, switch/router is the bottleneck

### Test: UDP Bandwidth (Jitter and Packet Loss)

```bash
iperf3 -c 192.168.2.10 -u -b 1G -t 30
```

**Flags:**
- `-u` - UDP mode
- `-b 1G` - Target bandwidth (1 Gbps)

**Expected output:**

```
[  5]   0.00-30.00  sec  3.58 GBytes  1.03 Gbits/sec  0.012 ms  0/2619842 (0%)
```

**What to watch:**
- **Jitter < 1 ms:** Excellent
- **Packet loss 0%:** Perfect
- **Packet loss > 1%:** Problem (bad cable, switch dropping packets)

### Test: K8s Pod → Synology NAS

Verify pods can reach full network speed (important for NFS-backed PVCs):

```bash
# Start iperf3 server on Synology (if not running)
ssh jlambert@192.168.2.10 "iperf3 -s -D"

# From K8s cluster
kubectl run iperf3-client --rm -it --image=mlabbe/iperf3 -- iperf3 -c 192.168.2.10 -t 30
```

**Expected:** Same ~940 Mbps

**If lower:**
- Check CNI overhead (Flannel/Calico add encapsulation)
- Check pod network policies (throttling?)
- Check NAS CPU usage during test (might be maxed)

---

## NFS Performance

**Tests:** Sequential read/write, random I/O, metadata operations. Critical for media servers and K8s PVCs.

### Baseline: Local Disk on NAS

SSH to Synology, test raw storage performance:

```bash
ssh jlambert@192.168.2.10

# Write test (create 10 GB file)
dd if=/dev/zero of=/volume1/test-write.img bs=1M count=10240 conv=fdatasync
# Note the MB/s

# Read test (read the file)
dd if=/volume1/test-write.img of=/dev/null bs=1M count=10240
# Note the MB/s

# Cleanup
rm /volume1/test-write.img
```

**My results (Synology DS920+, RAID 5, 4x4TB WD Red):**

- Write: **220 MB/s**
- Read: **280 MB/s**

This is the storage ceiling. NFS will be slower (network + protocol overhead).

### Test: NFS from Proxmox

Mount NFS share temporarily:

```bash
# On Proxmox
mkdir /mnt/nfs-test
mount -t nfs 192.168.2.10:/volume1/nfs01 /mnt/nfs-test

# Write test
dd if=/dev/zero of=/mnt/nfs-test/test-write.img bs=1M count=10240 conv=fdatasync

# Read test
dd if=/mnt/nfs-test/test-write.img of=/dev/null bs=1M count=10240

# Cleanup
rm /mnt/nfs-test/test-write.img
umount /mnt/nfs-test
```

**My results:**

- Write: **110 MB/s** (50% of local)
- Read: **115 MB/s** (41% of local)

**Overhead breakdown:**
- Network: ~6% (TCP)
- NFS protocol: ~10%
- Synology CPU (NFS daemon): ~30%

**What's normal (1 Gbps network):**
- **100-115 MB/s:** Expected max (network bandwidth limit)
- **70-100 MB/s:** Good (some NFS overhead)
- **< 70 MB/s:** Poor (investigate)

<div class="admonition admonition-info" markdown="1">
<div class="admonition-title">ℹ️ Info</div>
<div class="admonition-content" markdown="1">

1 Gbps = 125 MB/s theoretical. NFS overhead + network protocol overhead = ~110 MB/s realistic max.

</div>
</div>

### Test: Random I/O with fio

`fio` (Flexible I/O Tester) simulates real workloads:

**Install:**

```bash
apt install fio
```

**Random read (4K blocks, simulates database/VM workload):**

```bash
fio --name=random-read \
    --ioengine=libaio \
    --rw=randread \
    --bs=4k \
    --size=1G \
    --numjobs=4 \
    --runtime=30 \
    --group_reporting \
    --directory=/mnt/nfs-test
```

**Output:**

```
random-read: (groupid=0, jobs=4): err= 0: pid=12345
  read: IOPS=8432, BW=32.9MiB/s (34.5MB/s)(987MiB/30001msec)
```

**Key metrics:**
- **IOPS:** 8432 (operations per second)
- **Bandwidth:** 32.9 MiB/s

**Random write (4K blocks):**

```bash
fio --name=random-write \
    --ioengine=libaio \
    --rw=randwrite \
    --bs=4k \
    --size=1G \
    --numjobs=4 \
    --runtime=30 \
    --group_reporting \
    --directory=/mnt/nfs-test
```

**My results (NFS on Synology RAID 5):**

- Random read IOPS: **8,000-9,000**
- Random write IOPS: **1,500-2,000** (RAID 5 write penalty)

**Sequential read (simulates streaming media):**

```bash
fio --name=sequential-read \
    --ioengine=libaio \
    --rw=read \
    --bs=1M \
    --size=2G \
    --numjobs=1 \
    --runtime=30 \
    --directory=/mnt/nfs-test
```

**Expected:** 110-115 MB/s (matches network limit)

---

## K8s PVC Performance

**Test NFS-backed persistent volumes** (media stack scenario):

Deploy a test pod with fio:

```yaml
# fio-test.yaml
apiVersion: v1
kind: Pod
metadata:
  name: fio-test
  namespace: default
spec:
  containers:
  - name: fio
    image: nixery.dev/shell/fio
    command: ["sleep", "3600"]
    volumeMounts:
    - name: test-pvc
      mountPath: /data
  volumes:
  - name: test-pvc
    persistentVolumeClaim:
      claimName: fio-test-pvc
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: fio-test-pvc
  namespace: default
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: nfs-appdata
  resources:
    requests:
      storage: 5Gi
```

Apply and test:

```bash
kubectl apply -f fio-test.yaml
kubectl wait --for=condition=ready pod/fio-test

# Sequential write (simulates Plex recording)
kubectl exec fio-test -- fio --name=seq-write \
    --ioengine=libaio --rw=write --bs=1M --size=2G \
    --numjobs=1 --runtime=30 --directory=/data

# Random read (simulates database queries)
kubectl exec fio-test -- fio --name=rand-read \
    --ioengine=libaio --rw=randread --bs=4k --size=1G \
    --numjobs=4 --runtime=30 --directory=/data

# Cleanup
kubectl delete -f fio-test.yaml
```

**Expected:** Similar to direct NFS mount (100-115 MB/s sequential, 8k IOPS random read)

**If slower:**
- CSI driver overhead
- Pod CPU limits throttling
- NFS mount options (check `values.yaml` for NFS CSI driver)

---

## CPU and Compute Performance

### sysbench: CPU Benchmark

**Install:**

```bash
apt install sysbench
```

**Single-threaded performance:**

```bash
sysbench cpu --threads=1 --time=30 run
```

**Output:**

```
CPU speed:
    events per second:  1234.56
```

**Multi-threaded performance:**

```bash
sysbench cpu --threads=$(nproc) --time=30 run
```

**My results:**

| Host | Single-thread (events/s) | Multi-thread (events/s) |
|------|--------------------------|-------------------------|
| Proxmox (i7-6700) | 1850 | 7200 (4 cores) |
| Talos K8s Worker VM | 1200 | 2400 (2 vCPU) |
| Synology (Celeron J4125) | 650 | 2100 (4 cores) |

**Use cases:**
- Plex transcoding: ~1500 events/s per stream (software transcode)
- K8s etcd: Single-thread performance matters most

### Stress Test: Sustained Load

**Install stress-ng:**

```bash
apt install stress-ng
```

**CPU + memory stress (10 minutes):**

```bash
stress-ng --cpu $(nproc) --vm 2 --vm-bytes 50% --timeout 10m --metrics
```

Watch:
- CPU temp: `watch -n 1 sensors` (if `lm-sensors` installed)
- Throttling: `dmesg | grep -i "cpu clock throttled"`

**Expected:**
- Temps stable under 80°C (desktop CPUs)
- No throttling messages
- System remains responsive

If temps spike above 90°C or you see throttling: airflow problem, thermal paste dried out, or insufficient cooling.

---

## Results Summary: My Homelab Baselines

Document these for future comparison:

| Test | Value | Notes |
|------|-------|-------|
| **Network** |
| Proxmox ↔ Synology TCP | 940 Mbps | Expected max for 1 Gbps |
| K8s Pod → Synology TCP | 935 Mbps | Negligible overhead |
| UDP packet loss | 0% | No congestion |
| **Storage** |
| Synology local write | 220 MB/s | RAID 5 ceiling |
| Synology local read | 280 MB/s | RAID 5 ceiling |
| NFS write (Proxmox) | 110 MB/s | Network-limited |
| NFS read (Proxmox) | 115 MB/s | Network-limited |
| NFS random read IOPS | 8,500 | Good for RAID 5 |
| NFS random write IOPS | 1,800 | RAID 5 write penalty |
| K8s PVC write | 108 MB/s | CSI driver overhead ~2% |
| **Compute** |
| Proxmox CPU (single) | 1850 events/s | Sufficient for etcd |
| Proxmox CPU (multi) | 7200 events/s | ~2 Plex transcodes |
| K8s Worker CPU (single) | 1200 events/s | VM overhead ~35% |
| K8s Worker CPU (multi) | 2400 events/s | ~1 Plex transcode |

---

## Troubleshooting Playbook

### "My network is slow"

1. **Baseline test:** iperf3 between two nodes
   - < 400 Mbps → Check link speed: `ethtool eth0 | grep Speed`
   - Duplex mismatch: `ethtool eth0 | grep Duplex` (should be "Full")
2. **Switch stats:** Check for errors
   - EdgeSwitch: Web UI → Ports → Look for "CRC Errors" or "Collisions"
3. **Cable test:** Swap cable, re-test

### "NFS is slow"

1. **Network first:** iperf3 to NAS (if network is slow, NFS will be slow)
2. **NFS vs local:** Compare `dd` write speeds on NFS vs NAS local disk
   - If local is also slow → disk problem (RAID rebuild? Failing drive?)
3. **NFS mount options:** Check mount options:
   ```bash
   mount | grep nfs
   # Look for: rsize=1048576,wsize=1048576 (1 MB buffers)
   ```
4. **NAS CPU:** SSH to NAS, check CPU during transfer:
   ```bash
   top
   # Is nfsd process at 100%?
   ```

### "Plex is buffering"

Systematically test each layer:

1. **Network:** iperf3 from Plex server to client device (should be > 50 Mbps for 1080p, > 100 Mbps for 4K)
2. **Storage:** fio test on media PVC (should sustain > 50 MB/s read)
3. **CPU:** Check Plex transcoding load:
   ```bash
   kubectl top pod -n media -l app.kubernetes.io/name=plex
   # Is CPU at limit?
   ```
4. **Client:** Is the client forcing transcode? (Check Plex dashboard → Now Playing → Transcode reason)

---

## Automating Benchmarks

Create a script to run quarterly:

```bash
#!/bin/bash
# homelab-benchmark.sh

OUTPUT="benchmark-$(date +%Y%m%d).txt"

echo "=== Homelab Benchmark Report ===" | tee "$OUTPUT"
echo "Date: $(date)" | tee -a "$OUTPUT"
echo "" | tee -a "$OUTPUT"

echo "--- Network: Proxmox → Synology ---" | tee -a "$OUTPUT"
iperf3 -c 192.168.2.10 -t 10 | grep sender | tee -a "$OUTPUT"

echo "" | tee -a "$OUTPUT"
echo "--- NFS Write ---" | tee -a "$OUTPUT"
dd if=/dev/zero of=/mnt/nfs-test/benchmark.img bs=1M count=1024 conv=fdatasync 2>&1 | grep copied | tee -a "$OUTPUT"

echo "" | tee -a "$OUTPUT"
echo "--- NFS Read ---" | tee -a "$OUTPUT"
dd if=/mnt/nfs-test/benchmark.img of=/dev/null bs=1M 2>&1 | grep copied | tee -a "$OUTPUT"

rm /mnt/nfs-test/benchmark.img

echo "" | tee -a "$OUTPUT"
echo "--- CPU (single-thread) ---" | tee -a "$OUTPUT"
sysbench cpu --threads=1 --time=10 run | grep "events per second" | tee -a "$OUTPUT"

echo "" | tee -a "$OUTPUT"
echo "=== Benchmark Complete ===" | tee -a "$OUTPUT"
echo "Report saved: $OUTPUT"
```

Run quarterly, compare results over time. Detect degradation before it causes outages.

---

## What I Learned

### 1. Baselines Prevent Wild Goose Chases

Plex was buffering. I blamed the network. Upgraded a cable. Buffering. Blamed the NAS. Checked disk health. Fine. Blamed Kubernetes. Recreated the PVC. Buffering. Three evenings of guessing. Finally ran benchmarks. Network: 940 Mbps wired. NFS: 110 MB/s. Not the problem. Turned out the TV was on 2.4 GHz Wi-Fi (40 Mbps actual) and the router was two rooms away. Moved the TV to 5 GHz. Buffering gone. The homelab was fine. The *last* place I looked was the actual problem. I'd have found that in 10 minutes if I'd had baselines. Without them, I was throwing darts. Baselines turn "something is slow" into "here's exactly what's slow."

### 2. 1 Gbps Is Fine for Most Homelabs

I was *convinced* I needed 10 GbE. "Future-proofing." "Proper homelab." I ran benchmarks first. Plex 4K: 60-80 Mbps. NFS during backups: 110 MB/s. Peak household: under 300 Mbps. I'd have spent $800 on NICs and a switch for throughput I'd never use. The benchmarks saved me from myself. 1 Gbps is fine. Your wallet will thank you.

### 3. NFS Overhead Is Real but Acceptable

50% overhead vs local disk (220 → 110 MB/s). I was disappointed at first. Then I did the math: 110 MB/s streams multiple 4K movies, runs Kubernetes PVCs, and handles backups. The bottleneck was never NFS for my workload. It was my expectations. Accept the overhead. Measure whether it actually matters for *you*.

### 4. CPU Performance Matters More Than You Think

Unbelievable but true: I upgraded from a Celeron to an i7 for Proxmox. Same RAM. Same storage. Kubernetes felt *noticeably* faster. etcd operations, pod scheduling - everything snappier. I ran sysbench after. Single-thread: 650 → 1850 events/s. Consensus writes are single-threaded. I'd felt it before I measured it. CPU matters more than we give it credit for.

### 5. Random I/O Is the Real Bottleneck

Sequential reads: 115 MB/s. "Great!" Random 4K reads: 8,500 IOPS. Sounds impressive. That's 33 MB/s. I tried running a Postgres database on NFS. It was *painful*. Databases do random I/O. NFS is built for sequential. If you're running databases on your homelab, use local SSDs or iSCSI. Trust me on this one.

---

## What's Next

You have performance baselines for your homelab. Network bandwidth, NFS throughput, storage IOPS, and CPU performance documented.

**Optional enhancements:**

- **Grafana dashboards** - Visualize trends over time (network throughput degrading?)
- **10 GbE upgrade** - If benchmarks show you're hitting 1 Gbps limits
- **SSD caching** - Add SSD read/write cache to Synology for IOPS boost
- **Automated regression testing** - Run benchmarks after every major change (DSM update, kernel upgrade)

The core workflow works. Baseline, test, compare. Find bottlenecks before they find you.

---

## Related

- [Media Stack on K8s](/2026-02-08-k8s-media-stack/) - Plex buffering is often a network benchmark problem
- [Proxmox Backup Server](/2026-02-08-proxmox-backup-server/) - NFS throughput matters for backup performance

## References

- [iperf3 Documentation](https://iperf.fr/)
- [fio Documentation](https://fio.readthedocs.io/)
- [sysbench Documentation](https://github.com/akopytov/sysbench)
- [Linux I/O Performance Testing](https://www.kernel.org/doc/Documentation/iostats.txt)
- [NFS Performance Tuning](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/performance_tuning_guide/sect-red_hat_enterprise_linux-performance_tuning_guide-storage_and_file_systems-nfs)
