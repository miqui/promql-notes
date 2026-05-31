# Prometheus Metric Types & PromQL Reference

> Focused on API monitoring and Linux node observability.

---

## Mental Model: Choosing the Right Type

| Question | Answer → Type |
|---|---|
| Does the value only ever go up? | **Counter** |
| Does it represent the current state ("right now")? | **Gauge** |
| Do you need fleet-wide latency percentiles? | **Histogram** |
| Do you need per-instance quantiles from a runtime? | **Summary** |

---

## 1. Counter

### What it is

A monotonically increasing value that only goes up (or resets to zero on process restart). Never use for values that can decrease.

**Always wrap in `rate()` or `increase()`** — raw counter values are rarely useful on their own.

### Use when tracking

- HTTP requests total
- Error counts
- Bytes sent / received
- DNS lookups
- Syscalls

### PromQL Queries

```promql
# Request throughput per second (5 min window) — foundational API traffic metric
rate(http_requests_total{job="api"}[5m])
```

```promql
# 5xx error ratio — the most common API SLO signal
rate(http_requests_total{status=~"5.."}[5m])
  / rate(http_requests_total[5m])
```

```promql
# Top 5 hottest routes by request rate
topk(5, rate(http_requests_total{job="api"}[5m])) by (route)
```

```promql
# Context switches per minute — high values indicate CPU contention on Linux nodes
increase(node_context_switches_total[1m])
```

```promql
# Outbound network throughput in bits/sec (excludes loopback)
rate(node_network_transmit_bytes_total{device!="lo"}[5m]) * 8
```

```promql
# Disk IOPS — reads/sec per device
rate(node_disk_reads_completed_total[5m])
```

> **The Counter trap:** never read a raw counter in a dashboard or alert — it resets to 0 on every restart. Always use `rate()` for per-second throughput or `increase()` for delta over a window.

---

## 2. Gauge

### What it is

A point-in-time snapshot of a value that can go up or down. Read it raw — no `rate()` needed. Represents the current state of the system.

### Use when tracking

- CPU utilization
- Memory used
- Active / in-flight connections
- Queue depth
- Goroutine count
- Open file descriptors

### PromQL Queries

```promql
# Memory utilization ratio (0–1) — more accurate than MemFree alone
1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

```promql
# CPU usage % per node — the classic Linux CPU formula
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```

```promql
# File descriptor saturation — alerts before "too many open files" crashes your API process
process_open_fds{job="api"} / process_max_fds{job="api"}
```

```promql
# Concurrent in-flight requests — spike = overload; flat during traffic = pool exhaustion
http_requests_in_flight{job="api"}
```

```promql
# Load average normalized by CPU count — >1.0 means the run queue is growing
node_load1 / count by(instance)(node_cpu_seconds_total{mode="idle"})
```

```promql
# Root filesystem free ratio — alert before the disk fills
node_filesystem_avail_bytes{mountpoint="/"}
  / node_filesystem_size_bytes{mountpoint="/"}
```

---

## 3. Histogram

### What it is

Stores observed values in configurable buckets (`_bucket`), along with a running sum (`_sum`) and count (`_count`). Quantiles are computed **server-side** using `histogram_quantile()`. This is the best metric type for latency.

### Use when tracking

- Request latency / response time
- Response payload size
- Database query duration
- Queue wait time
- Inbound request size

### PromQL Queries

```promql
# p99 API latency — your most important SLO signal for tail latency
histogram_quantile(0.99,
  rate(http_request_duration_seconds_bucket{job="api"}[5m])
)
```

```promql
# p95 latency broken down by route — find the slow endpoints
histogram_quantile(0.95,
  rate(http_request_duration_seconds_bucket{job="api"}[5m])
) by (route)
```

```promql
# Average latency — cheaper to compute but hides tail outliers; use alongside p99
rate(http_request_duration_seconds_sum[5m])
  / rate(http_request_duration_seconds_count[5m])
```

```promql
# Fraction of requests served within 500 ms — direct SLO compliance ratio
rate(http_request_duration_seconds_bucket{le="0.5",job="api"}[5m])
  / rate(http_request_duration_seconds_count{job="api"}[5m])
```

```promql
# p99 disk I/O latency on Linux — slow tail I/O appears here before throughput drops
histogram_quantile(0.99,
  rate(node_disk_io_time_seconds_bucket[5m])
)
```

```promql
# Median inbound request payload size — useful for capacity planning and DDoS detection
histogram_quantile(0.5,
  rate(http_request_size_bytes_bucket{job="api"}[5m])
)
```

---

## 4. Summary

### What it is

Pre-computed quantiles calculated **on the client side** at scrape time, exposed as `{quantile="0.99"}` labels. Accurate for a single instance but **cannot be aggregated across instances**. Use Histogram when you need fleet-wide percentiles.

### Use when tracking

- Single-instance percentiles
- Go GC pause time (exposed automatically by the Go runtime)
- JVM GC duration
- gRPC call latency

### PromQL Queries

```promql
# p99 Go GC stop-the-world pause in ms — exposed automatically by the Go runtime
go_gc_duration_seconds{quantile="0.99"} * 1000
```

```promql
# Median GC pause — compare to p99 to gauge pause variability
go_gc_duration_seconds{quantile="0.5"}
```

```promql
# Average RPC call duration — computed from _sum / _count (safe to aggregate)
rate(rpc_duration_seconds_sum[5m])
  / rate(rpc_duration_seconds_count[5m])
```

```promql
# Alert expression — fires when p99 latency exceeds 1 s on any single instance
http_request_duration_seconds{quantile="0.99", job="api"} > 1
```

```promql
# ⚠️ Mathematically INCORRECT for multi-instance deployments
# Use Histogram + histogram_quantile() instead for fleet-wide p95
avg(http_request_duration_seconds{quantile="0.95"})
```

> **The Histogram vs Summary tradeoff:** Summary quantiles are computed on the client — exact for one replica but cannot be mathematically averaged across replicas. Histogram buckets are additive, so `histogram_quantile()` works correctly across your entire fleet behind a load balancer.

---

## Quick Reference Cheat Sheet

| Metric Type | Raw useful? | Aggregatable? | Quantiles? | Best for |
|---|---|---|---|---|
| Counter | ❌ (use `rate()`) | ✅ | ❌ | Throughput, error rate |
| Gauge | ✅ | ✅ | ❌ | Utilization, queue depth |
| Histogram | Partial | ✅ | ✅ server-side | Latency SLOs, fleet-wide p99 |
| Summary | ✅ | ⚠️ only `_sum`/`_count` | ✅ client-side | Single-instance runtime metrics |

---

## Common Function Reference

| Function | Works on | Purpose |
|---|---|---|
| `rate(m[5m])` | Counter | Per-second rate over window |
| `increase(m[1m])` | Counter | Total increase over window |
| `irate(m[5m])` | Counter | Instantaneous rate (last two samples) |
| `histogram_quantile(φ, …)` | Histogram | Compute quantile from buckets |
| `avg_over_time(m[5m])` | Gauge | Average gauge value over window |
| `max_over_time(m[5m])` | Gauge | Peak gauge value over window |
| `topk(n, expr)` | Any | Top N time series by value |
| `sum by (label)` | Any | Aggregate across label dimension |

---

## node_exporter Label Reference

These labels appear on most `node_*` metrics and are useful for filtering:

| Label | Example values | Use |
|---|---|---|
| `instance` | `10.0.0.1:9100` | Per-host filtering |
| `device` | `sda`, `eth0`, `lo` | Disk / network device |
| `mountpoint` | `/`, `/data` | Filesystem |
| `mode` | `idle`, `user`, `system`, `iowait` | CPU mode breakdown |
| `job` | `node`, `api` | Scrape job grouping |

---

*Requires `node_exporter` for Linux node metrics. Requires standard Prometheus client instrumentation for API metrics.*
