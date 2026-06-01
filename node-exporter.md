# Node Exporter Metrics: CPU, Memory & IOPS Pressure

> **SRE Reference Guide** — PromQL queries and alerting rules for Linux host observability using the USE method (Utilization, Saturation, Errors).

***

## Overview

Node Exporter is a Prometheus exporter for hardware and OS metrics exposed by \*NIX kernels[1]. It scrapes from Linux kernel interfaces (`/proc/stat`, `/proc/meminfo`, `/proc/diskstats`) and exposes them as Prometheus metrics at the `/metrics` endpoint[2]. This guide covers the three most critical SRE signal categories: **CPU**, **Memory**, and **IOPS/Disk Pressure**, structured around the USE method.

***

## CPU Metrics

The core raw metric is `node_cpu_seconds_total` — a counter tracking cumulative CPU time per mode (idle, user, system, iowait, irq, softirq, steal) per CPU core[3][2]. All CPU utilization queries derive from this via `rate()` over a time window.

### PromQL Queries

| Signal | PromQL |
|--------|--------|
| **Utilization** (all cores avg) | `1 - avg(rate(node_cpu_seconds_total{mode="idle"}[1m]))` |
| **Utilization %** (per node) | `100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100)` |
| **iowait pressure** | `avg by(instance) (rate(node_cpu_seconds_total{mode="iowait"}[5m]))` |
| **User-space CPU** | `avg by(instance) (rate(node_cpu_seconds_total{mode="user"}[5m]))` |
| **Kernel-space CPU** | `avg by(instance) (rate(node_cpu_seconds_total{mode="system"}[5m]))` |
| **Steal time** (VMs) | `avg by(instance) (rate(node_cpu_seconds_total{mode="steal"}[5m]))` |
| **Saturation** (load vs cores) | `node_load1 / count by(instance)(sum by(instance,cpu)(node_cpu_seconds_total))` |
| **Load averages** | `node_load1`, `node_load5`, `node_load15` |

### Key Signals to Watch

- **iowait** > 20% signals disk I/O is blocking CPU threads — correlates directly with disk saturation[3]
- **steal** > 10% on VMs indicates CPU time is being stolen by the hypervisor — a cloud noisy-neighbor issue
- **Load average > number of vCPUs** indicates run queue saturation — processes are waiting to be scheduled[4]
- Node Exporter provides `node_load1`, `node_load5`, and `node_load15` for 1, 5, and 15-minute load averages[4]

***

## Memory Metrics

Memory metrics are sourced from `/proc/meminfo`[2]. The preferred signal for available memory is `node_memory_MemAvailable_bytes` (not `MemFree`), which accounts for reclaimable page cache and buffers — matching the behavior of the `free` command[5].

### PromQL Queries

| Signal | PromQL |
|--------|--------|
| **Total memory** | `node_memory_MemTotal_bytes` |
| **Available memory** | `node_memory_MemAvailable_bytes` |
| **Used memory** | `node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes` |
| **Utilization %** | `1 - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes) / node_memory_MemTotal_bytes` |
| **Available %** | `node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100` |
| **Major page faults (pressure)** | `rate(node_vmstat_pgmajfault[1m])` |
| **Swap activity (saturation)** | `rate(node_vmstat_pgpgin[1m]) + rate(node_vmstat_pgpgout[1m])` |
| **OOM kill events** | `increase(node_vmstat_oom_kill[1m])` |

### Key Signals to Watch

- `node_vmstat_pgmajfault` is the most reliable indicator of **true memory pressure** — major page faults mean the kernel is fetching pages from disk, causing latency spikes[1][4]
- High swap page-in/out (`pgpgin`/`pgpgout`) rates indicate the system is actively swapping, which degrades application performance significantly
- OOM kills (`node_vmstat_oom_kill`) signal critical memory exhaustion — workloads are being terminated by the kernel
- Available memory below 10% of total is a common alerting threshold for low-memory warnings[3]

***

## IOPS / Disk Pressure Metrics

Disk metrics come from `/proc/diskstats`[2] and are exposed under the `node_disk_*` namespace. The most important saturation metric is `node_disk_io_time_seconds_total`, which represents time the disk was busy — analogous to `%util` from `iostat`[4].

### PromQL Queries

| Signal | PromQL |
|--------|--------|
| **Read IOPS** | `rate(node_disk_reads_completed_total[1m])` |
| **Write IOPS** | `rate(node_disk_writes_completed_total[1m])` |
| **Read throughput** | `rate(node_disk_read_bytes_total[1m])` |
| **Write throughput** | `rate(node_disk_written_bytes_total[1m])` |
| **I/O utilization %** | `rate(node_disk_io_time_seconds_total[1m]) * 100` |
| **I/O wait queue (saturation)** | `rate(node_disk_io_time_weighted_seconds_total[1m])` |
| **Read latency** | `rate(node_disk_read_time_seconds_total[1m]) / rate(node_disk_reads_completed_total[1m])` |
| **Write latency** | `rate(node_disk_write_time_seconds_total[1m]) / rate(node_disk_writes_completed_total[1m])` |
| **Filesystem available** | `node_filesystem_avail_bytes{mountpoint!~"/boot\|/boot/efi"}` |
| **Filesystem used %** | `1 - node_filesystem_avail_bytes / node_filesystem_size_bytes` |

### Key Signals to Watch

- `node_disk_io_time_seconds_total` approaching `1.0` (100% when multiplied) means the disk is fully saturated — applications will stall waiting for reads/writes[3]
- `node_disk_io_time_weighted_seconds_total` is the gold-standard IOPS pressure metric, equivalent to Linux `await` from `iostat -x`[4]
- Read/write latency above 1–5ms (HDDs) or 0.1–1ms (SSDs) typically indicates saturation
- Filesystem usage above 85% is a common alert threshold to avoid unexpected out-of-space failures

***

## Alerting Rules

These Prometheus alerting rules implement signal thresholds tuned for production SRE workflows[1][3].

```yaml
groups:
  - name: node_exporter_pressure_alerts
    rules:

      # ─── CPU ───────────────────────────────────────────────────────

      - alert: HighCPUUtilization
        expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 85
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU utilization on {{ $labels.instance }}"
          description: "CPU usage is {{ $value | humanize }}% (threshold: 85%)"

      - alert: HighCPUIOWait
        expr: avg by(instance)(rate(node_cpu_seconds_total{mode="iowait"}[5m])) > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High iowait on {{ $labels.instance }}"
          description: "iowait is {{ $value | humanizePercentage }} — disk may be saturated"

      - alert: CPUStealHigh
        expr: avg by(instance)(rate(node_cpu_seconds_total{mode="steal"}[5m])) > 0.1
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "High CPU steal on {{ $labels.instance }}"

      # ─── MEMORY ────────────────────────────────────────────────────

      - alert: LowMemoryAvailable
        expr: node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes * 100 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low available memory on {{ $labels.instance }}"
          description: "Only {{ $value | humanize }}% memory available"

      - alert: HostMemoryPressure
        expr: rate(node_vmstat_pgmajfault[1m]) > 1000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory pressure (major page faults) on {{ $labels.instance }}"
          description: "{{ $value | humanize }} major page faults/s"

      - alert: OOMKillDetected
        expr: increase(node_vmstat_oom_kill[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "OOM kill on {{ $labels.instance }}"
          description: "A process was killed by the OOM killer"

      # ─── DISK / IOPS ───────────────────────────────────────────────

      - alert: DiskIOSaturation
        expr: rate(node_disk_io_time_seconds_total[1m]) > 0.9
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Disk I/O saturation on {{ $labels.instance }} ({{ $labels.device }})"
          description: "Disk utilization is {{ $value | humanizePercentage }}"

      - alert: DiskFilesystemNearFull
        expr: 1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Filesystem nearly full on {{ $labels.instance }}:{{ $labels.mountpoint }}"
          description: "{{ $value | humanizePercentage }} used"

      - alert: DiskFilesystemFull
        expr: 1 - node_filesystem_avail_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes > 0.95
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Filesystem critically full on {{ $labels.instance }}:{{ $labels.mountpoint }}"
```

***

## Kubernetes Context

In Kubernetes environments, Node Exporter is typically deployed as a **DaemonSet**, ensuring one pod per node[6]. Metrics are scraped via `ServiceMonitor` (Prometheus Operator) or via static scrape configs. Common additions:

- Pair `node_exporter` with `kube-state-metrics` for pod-level resource correlation
- Use `topk(5, rate(node_disk_writes_completed_total[5m]))` to find highest IOPS-consuming nodes
- In cloud environments (AWS, GCP), high `steal` time alongside high IOPS often indicates the instance type needs an upgrade (e.g., general-purpose → storage-optimized)
- For EBS/PD-backed volumes, correlate `node_disk_io_time_weighted_seconds_total` with cloud-provider IOPS quota metrics

***

## Quick Reference: Signal-to-Metric Mapping

| Pressure Type | Primary Metric | Saturation Metric | Threshold |
|---------------|---------------|-------------------|-----------|
| CPU Utilization | `node_cpu_seconds_total{mode="idle"}` | `node_load1` / core count | > 85% util, load > 1.0/core |
| CPU iowait | `node_cpu_seconds_total{mode="iowait"}` | — | > 20% |
| Memory | `node_memory_MemAvailable_bytes` | `node_vmstat_pgmajfault` | < 10% avail, > 1000 faults/s |
| Swap | `node_vmstat_pgpgin` + `pgpgout` | — | Any sustained activity |
| Disk I/O | `node_disk_io_time_seconds_total` | `node_disk_io_time_weighted_seconds_total` | > 90% util |
| Filesystem | `node_filesystem_avail_bytes` | — | > 85% used |
