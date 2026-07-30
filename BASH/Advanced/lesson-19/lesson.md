# Lesson 19: Capstone — Monitoring System

## History & Origins

System monitoring is as old as Unix itself. In the 1970s, Unix administrators used `ps`, `df`, and `iostat` to check system health manually. The first automated monitoring scripts were cron jobs that emailed root when disk space ran low.

The early 1990s brought `mon` (a general-purpose monitoring daemon) and `Big Brother` (the first web-based monitoring system). Nagios (2002, originally NetSaint) popularized the plugin architecture where monitoring checks are standalone scripts — many written in Bash or shell. Today, Prometheus (2012) uses a pull model with exporters (also often shell scripts), while legacy systems still run Nagios plugins.

Bash-based monitoring remains ubiquitous because:
- Every Linux system has Bash
- No dependencies to install on monitored hosts
- Shell scripts can read `/proc` and `/sys` directly
- Simple threshold checking requires no external tools

The "monitoring loop" pattern — collect, check, log, alert, sleep, repeat — is the same today as it was in 1990. What changed is scale: modern systems monitor thousands of hosts, use time-series databases, and employ complex alerting logic like exponential backoff and consecutive-failure counting.

## Syntax Reference

### /proc Filesystem Metrics

| File | Content | Metric |
|------|---------|--------|
| `/proc/stat` | CPU time (user, nice, system, idle, iowait, irq, softirq, steal) | CPU usage % |
| `/proc/meminfo` | MemTotal, MemFree, Buffers, Cached, SwapTotal, SwapFree | Memory usage % |
| `/proc/net/dev` | Per-interface RX/TX bytes and packets | Network throughput |
| `/proc/loadavg` | 1/5/15 min load averages | System load |
| `/proc/uptime` | Uptime in seconds, idle time | System uptime |
| `/proc/diskstats` | Per-disk I/O operations and sectors | Disk I/O |

### Key Commands

```bash
# CPU: two samples required for delta
get_cpu_usage() {
  local cpu line
  read -r cpu user nice system idle iowait irq softirq steal < /proc/stat
  local total=$((user + nice + system + idle + iowait + irq + softirq + steal))
  local used=$((total - idle))
  sleep 1
  read -r cpu user nice system idle iowait irq softirq steal < /proc/stat
  local total2=$((user + nice + system + idle + iowait + irq + softirq + steal))
  local used2=$((total2 - idle))
  local delta_used=$((used2 - used))
  local delta_total=$((total2 - total))
  echo $((delta_used * 100 / delta_total))
}

# Memory
get_mem_usage() {
  free -m | awk '/^Mem:/ {printf "%.1f", $3/$2 * 100}'
}

# Disk
get_disk_usage() {
  df "$1" | awk 'NR==2 {gsub(/%/,""); print $5}'
}

# Network rate
get_net_rate() {
  local iface=$1
  local rx1 tx1 rx2 tx2
  read -r _ _ rx1 _ _ _ _ _ tx1 _ < <(grep "$iface:" /proc/net/dev)
  sleep 1
  read -r _ _ rx2 _ _ _ _ _ tx2 _ < <(grep "$iface:" /proc/net/dev)
  echo "$(( (rx2 - rx1) / 1024 )) $(( (tx2 - tx1) / 1024 ))"
}

# Load average
get_load_avg() {
  read -r load1 load5 load15 _ < /proc/loadavg
  echo "$load1 $load5 $load15"
}

# Process count
get_proc_count() {
  echo "$(ls -1 /proc/ | grep -c '^[0-9]')"
}
```

### Signal Handling

```bash
running=true
trap 'running=false; log info "Shutting down gracefully..."' SIGTERM SIGINT
trap 'log error "Unexpected error at line $LINENO"; cleanup; exit 1' ERR
trap 'cleanup; exit' EXIT
```

### Log Rotation

```bash
if [[ -f "$logfile" && $(stat -c%s "$logfile") -gt 10485760 ]]; then
  mv "$logfile" "${logfile}.1"
  gzip "${logfile}.1"
fi
```

## Under the Hood

### How /proc/stat CPU Calculation Works

The Linux kernel tracks time spent in various CPU states as jiffies (typically 100Hz = 10ms per jiffy). These counters are monotonically increasing since boot. To calculate CPU usage percentage over an interval:

1. Read total jiffies and idle jiffies at time T1
2. Sleep for interval (usually 1 second)
3. Read total jiffies and idle jiffies at time T2
4. `usage = (total_T2 - total_T1) - (idle_T2 - idle_T1)`
5. `percent = usage / (total_T2 - total_T1) * 100`

The idle counter includes `idle + iowait`. Some monitoring tools separate iowait as a distinct metric because high iowait indicates disk bottleneck.

### How /proc/net/dev Works

The kernel maintains per-interface byte/packet counters that reset at interface down/up. The counters are 64-bit unsigned integers (they wrap after ~584 years at 10Gbps). RX counters include all received bytes including headers. TX counters include all transmitted bytes including headers.

### How df Gets Filesystem Data

`df` reads from `/proc/mounts` for mount points and uses `statfs()` syscall to get block counts. The formula is:
- `used_percent = (total - available) / total * 100`
- Note: `available` reserves blocks for root (typically 5%), so `used + available` may be less than `total`.

### Signal Delivery

When you `kill -TERM <pid>`, the kernel delivers SIGTERM to the process. Bash's `trap` catches the signal and executes the handler. Crucially, if the process is in a `sleep` call, the signal interrupts sleep immediately — the handler runs, and `sleep` returns non-zero. The `set -e` option can cause unexpected exit if sleep returns non-zero due to signal.

## Core Examples

### Example 1: Complete CPU Monitor with RATE Detection

```bash
get_cpu_usage() {
  local cpu user nice system idle iowait irq softirq steal
  local total1 idle1 total2 idle2
  read -r cpu user nice system idle iowait irq softirq steal < /proc/stat
  total1=$((user + nice + system + idle + iowait + irq + softirq + steal))
  idle1=$idle
  sleep 1
  read -r cpu user nice system idle iowait irq softirq steal < /proc/stat
  total2=$((user + nice + system + idle + iowait + irq + softirq + steal))
  idle2=$idle
  local delta_total=$((total2 - total1))
  local delta_idle=$((idle2 - idle1))
  local delta_used=$((delta_total - delta_idle))
  if ((delta_total == 0)); then
    echo 0
  else
    echo $((delta_used * 100 / delta_total))
  fi
}
```

⚠️ **TRAP:** Division by zero is possible if delta_total is 0 (extremely rare but happens on heavily throttled systems). Always guard against divide-by-zero.

### Example 2: Memory Monitor with Swap

```bash
get_memory_stats() {
  local mem_total mem_free mem_avail swap_total swap_free
  mem_total=$(awk '/^MemTotal:/ {print $2}' /proc/meminfo)
  mem_free=$(awk '/^MemAvailable:/ {print $2}' /proc/meminfo)
  swap_total=$(awk '/^SwapTotal:/ {print $2}' /proc/meminfo)
  swap_free=$(awk '/^SwapFree:/ {print $2}' /proc/meminfo)
  local mem_used_percent=$(( (mem_total - mem_free) * 100 / mem_total ))
  local swap_used_percent=0
  if ((swap_total > 0)); then
    swap_used_percent=$(( (swap_total - swap_free) * 100 / swap_total ))
  fi
  echo "mem=${mem_used_percent} swap=${swap_used_percent}"
}
```

### Example 3: Disk Monitor with Inode Check

```bash
get_disk_stats() {
  local mount_point=$1
  local usage inode_usage
  usage=$(df "$mount_point" --output=pcent 2>/dev/null | tail -1 | tr -d '% ')
  inode_usage=$(df "$mount_point" --output=ipcent 2>/dev/null | tail -1 | tr -d '% ')
  echo "disk=${usage:-0} inode=${inode_usage:-0}"
}

# Multiple mount points
monitor_disks() {
  for mp in / /home /var /tmp; do
    if mountpoint -q "$mp" 2>/dev/null; then
      local stats
      stats=$(get_disk_stats "$mp")
      log info "DISK $mp: $stats"
    fi
  done
}
```

### Example 4: Network Monitor with Bandwidth Tracking

```bash
get_network_rates() {
  local iface=$1
  local rx1 tx1 rx2 tx2
  local line
  line=$(grep "$iface:" /proc/net/dev 2>/dev/null) || {
    echo "iface=$iface rx=0 tx=0 error=1"
    return
  }
  set -- $line
  # $2 = RX bytes, $10 = TX bytes
  rx1=$2; tx1=${10}
  sleep 1
  line=$(grep "$iface:" /proc/net/dev)
  set -- $line
  rx2=$2; tx2=${10}
  local rx_rate=$(( (rx2 - rx1) / 1024 )) # KB/s
  local tx_rate=$(( (tx2 - tx1) / 1024 )) # KB/s
  echo "iface=$iface rx=$rx_rate tx=$tx_rate error=0"
}
```

⚠️ **TRAP:** The field positions in `/proc/net/dev` are whitespace-separated, but the interface line may start with spaces. Use `set -- $line` (word splitting) rather than parsing specific columns to handle inconsistent spacing.

### Example 5: Threshold Alerting with Exponential Backoff

```bash
declare -A alert_count
declare -A last_alert

check_threshold() {
  local metric=$1 current_value=$2 threshold=$3
  local cooldown=${4:-300} # 5 minute default cooldown
  local max_consecutive=${5:-3}

  if (( $(echo "$current_value > $threshold" | bc -l) )); then
    ((alert_count[$metric]++))
    local now
    now=$(date +%s)
    local last=${last_alert[$metric]:-0}
    if (( (now - last) > cooldown )); then
      local severity="WARN"
      if (( alert_count[$metric] >= max_consecutive )); then
        severity="CRITICAL"
        auto_recover "$metric"
      fi
      send_alert "$severity" "$metric" "$current_value" "$threshold"
      last_alert[$metric]=$now
    fi
  else
    alert_count[$metric]=0
  fi
}
```

### Example 6: Auto-Recovery Actions

```bash
recover_service() {
  local service=$1
  log warn "Attempting restart of $service"
  if systemctl restart "$service" 2>/dev/null; then
    log info "Successfully restarted $service"
    return 0
  else
    log error "Failed to restart $service"
    return 1
  fi
}

recover_disk() {
  local mount_point=$1
  log warn "Disk full on $mount_point, cleaning temp files"
  local freed=0
  if [[ "$mount_point" == "/" ]] || [[ "$mount_point" == "/tmp" ]]; then
    freed=$(find /tmp -type f -atime +1 -delete -printf '.' 2>/dev/null | wc -c)
  fi
  # Rotate old logs
  find /var/log -name '*.log.*' -mtime +7 -delete 2>/dev/null
  log info "Cleaned up approximately ${freed} temp files"
}

recover_memory() {
  log warn "Memory critically low, clearing page cache"
  sync && echo 3 > /proc/sys/vm/drop_caches 2>/dev/null || true
  log info "Page cache cleared (non-destructive)"
}
```

### Example 7: Main Monitoring Loop

```bash
#!/bin/bash
set -euo pipefail

INTERVAL=30
THRESHOLD_CPU=90
THRESHOLD_MEM=85
THRESHOLD_DISK=90
ALERT_EMAIL=""
LOG_FILE="/var/log/monitor/monitor.log"
PID_FILE="/var/run/monitor.pid"
running=true
check_count=0

trap 'running=false' SIGTERM SIGINT
trap 'log info "Monitor stopped after ${check_count} checks"' EXIT

log() {
  local level=$1 msg=$2
  local timestamp
  timestamp=$(date '+%Y-%m-%d %H:%M:%S')
  echo "[$timestamp] [$level] $msg" >> "$LOG_FILE"
  [[ -t 1 ]] && echo "[$timestamp] [$level] $msg"
}

while $running; do
  ((check_count++))
  log "INFO" "Check #${check_count} starting"
  cpu=$(get_cpu_usage)
  mem=$(get_memory_stats | cut -d= -f2 | cut -d' ' -f1)
  disk=$(get_disk_stats / | cut -d= -f2 | cut -d' ' -f1)
  net=$(get_network_rates eth0)
  log "INFO" "CPU=${cpu}% MEM=${mem}% DISK=${disk}% NET={${net}}"
  [[ $cpu -gt $THRESHOLD_CPU ]] && check_threshold "CPU" "$cpu" "$THRESHOLD_CPU"
  [[ $mem -gt $THRESHOLD_MEM ]] && check_threshold "MEM" "$mem" "$THRESHOLD_MEM"
  [[ $disk -gt $THRESHOLD_DISK ]] && check_threshold "DISK" "$disk" "$THRESHOLD_DISK"
  sleep "$INTERVAL"
done
```

⚠️ **TRAP:** The main loop variable `$running` is checked at the top of each iteration. After `SIGTERM`, the current `sleep` is interrupted and `$running` is set to false. The loop exits on the next iteration. Without the `sleep $INTERVAL` being interruptible, shutdown could take up to `$INTERVAL` seconds.

### Example 8: Alert with Desktop Notification

```bash
send_alert() {
  local severity=$1 metric=$2 value=$3 threshold=$4
  local msg="[${severity}] ${metric}: ${value}% (threshold: ${threshold}%)"
  log "$severity" "$msg"
  
  # Desktop notification
  if command -v notify-send &>/dev/null; then
    local urgency="normal"
    [[ "$severity" == "CRITICAL" ]] && urgency="critical"
    notify-send -u "$urgency" "Monitor Alert" "$msg"
  fi
  
  # Email
  if [[ -n "$ALERT_EMAIL" ]]; then
    echo "$msg" | mail -s "Monitor $severity: $metric" "$ALERT_EMAIL"
  fi
  
  # Syslog
  logger -t monitor -p user.warn "$msg"
}
```

### Example 9: Metrics Over Time (Trend Tracking)

```bash
declare -A metric_history
MAX_HISTORY=60

record_metric() {
  local metric=$1 value=$2
  metric_history["$metric"]+="${value},"
  # Trim to max entries
  local count
  count=$(echo "${metric_history[$metric]}" | tr ',' '\n' | wc -l)
  if ((count > MAX_HISTORY)); then
    metric_history["$metric"]="${metric_history[$metric]#*,}"
  fi
}

get_trend() {
  local metric=$1
  local values
  values="${metric_history[$metric]}"
  if [[ -z "$values" ]]; then
    echo "stable"
    return
  fi
  local first last
  first=$(echo "$values" | cut -d, -f1)
  last=$(echo "$values" | tr ',' '\n' | tail -1)
  if ((last > first + 10)); then
    echo "increasing"
  elif ((last < first - 10)); then
    echo "decreasing"
  else
    echo "stable"
  fi
}
```

### Example 10: Daemon Mode with PID File

```bash
become_daemon() {
  local pid_file=$1
  # Fork to background
  if [[ -n "$*" ]]; then
    exec "$0" --daemon-internal "$@"
  fi
}

daemonize() {
  # Double fork to detach from terminal
  if [[ "$1" == "--daemon-internal" ]]; then
    shift
    # First fork
    if [[ -n "$PID_FILE" ]]; then
      echo $$ > "$PID_FILE"
    fi
    # Redirect standard fds
    exec 0</dev/null
    exec 1>>"$LOG_FILE"
    exec 2>&1
    main "$@"
  fi
}
```

⚠️ **TRAP:** PID files are advisory. An old PID file from a crashed instance will point to a non-existent process or (worse) a recycled PID belonging to a different process. Check if the PID in the file is still running before writing.

### Example 11: Config File for Monitoring Tool

```bash
# /etc/monitor/monitor.conf
THRESHOLD_CPU=90
THRESHOLD_MEM=85
THRESHOLD_DISK=90
THRESHOLD_INODE=80
THRESHOLD_LOAD=4.0
CHECK_INTERVAL=30
LOG_FILE="/var/log/monitor/monitor.log"
ALERT_EMAIL="admin@example.com"
ALERT_COOLDOWN=300
MAX_CONSECUTIVE=3
MONITOR_INTERFACES="eth0 eth1"
MONITOR_MOUNTS="/ /home /var"
SERVICES_TO_RESTART=(nginx sshd postgresql)
ENABLE_RECOVERY=true
```

### Example 12: Systemd Integration

```bash
# /etc/systemd/system/monitor.service
[Unit]
Description=System Monitoring Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/monitor.sh --config /etc/monitor/monitor.conf
PIDFile=/var/run/monitor.pid
Restart=on-failure
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
```

### Example 13: Check Results to JSON

```bash
generate_json_report() {
  local cpu=$1 mem=$2 disk=$3 net_rx=$4 net_tx=$5
  cat <<-JSON
{
  "timestamp": $(date +%s),
  "host": "$(hostname)",
  "metrics": {
    "cpu": { "usage": $cpu, "unit": "%" },
    "memory": { "usage": $mem, "unit": "%" },
    "disk": { "usage": $disk, "unit": "%" },
    "network": { "rx_kbps": $net_rx, "tx_kbps": $net_tx }
  },
  "alerts": [],
  "status": "ok"
}
JSON
}
```

### Example 14: Historical Data to CSV

```bash
export_csv() {
  local logfile=$1
  echo "timestamp,cpu,mem,disk,net"
  grep -oP '\[.*?\] \[INFO\] CPU=\K[0-9]+' "$logfile" > /tmp/cpu_vals
  grep -oP 'MEM=\K[0-9]+' "$logfile" > /tmp/mem_vals
  grep -oP 'DISK=\K[0-9]+' "$logfile" > /tmp/disk_vals
  paste -d',' /tmp/cpu_vals /tmp/mem_vals /tmp/disk_vals
}
```

### Example 15: Health Check Endpoint

```bash
health_check() {
  local port=${1:-8080}
  local response="HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n"
  response+=$(generate_json_report "$cpu" "$mem" "$disk" "$net_rx" "$net_tx")
  echo -e "$response" | nc -l -p "$port" -q 1 2>/dev/null || true
}
```

## Real-World Use Cases

1. **Nagios/Icinga plugins** — Thousands of Nagios plugins are Bash scripts that check disk, CPU, memory, services, and return OK/WARNING/CRITICAL with performance data.

2. **Prometheus node_exporter textfile collector** — The `--collector.textfile.directory` option lets cron jobs write metrics via Bash scripts. The node_exporter reads the text files and exposes them as Prometheus metrics.

3. **Netdata** — While written in C, Netdata's plugin architecture allows external plugins written in any language including Bash, integrating directly with the metrics dashboard.

4. **Monit** — A lightweight monitoring tool that uses Bash-like configuration syntax and can run arbitrary shell commands for custom checks.

5. **AWS CloudWatch custom metrics** — Bash scripts collect system metrics and push them to CloudWatch via the AWS CLI: `aws cloudwatch put-metric-data`.

6. **Server health dashboards** — Internal IT dashboards often run Bash scripts on each server, collecting metrics and posting JSON to a central API.

7. **CI/CD pipeline health** — Jenkins/GitLab CI jobs use Bash to check environment health before deployments, with threshold alerts that fail the pipeline.

## Memory Aids

- **SCLAR**: Sample, Calculate, Log, Alert, Recover — the monitoring pipeline.
- **Two Samples Rule**: CPU and network rates ALWAYS need two readings.
- **COOLDOWN**: Consecutive + Override + Offset + Limit + Delay + Wait + Notify — alert backoff strategy.
- **PID File Check**: "If it exists, check /proc; if not running, remove stale PID."
- **3-3-3 Rule:** 3 metrics (CPU, MEM, DISK), 3 severity levels (WARN, CRITICAL, RECOVERY), 3 recovery strategies (restart, clean, cache drop).

## Trap Vault

1. **⚠️ TRAP:** Single-sample CPU reading. Reading `/proc/stat` once gives total jiffies since boot, not a rate. Always take two samples with a sleep interval.

2. **⚠️ TRAP:** Division by zero in CPU calculation. If the system is idle for the entire interval, delta_total could theoretically be 0 on a tickless kernel. Guard with `if ((delta_total == 0))`.

3. **⚠️ TRAP:** `sleep` in a loop with SIGTERM handler. Without interruptible sleep, the script ignores SIGTERM for up to `$INTERVAL` seconds. The `trap` handler runs during signal delivery, which interrupts `sleep`.

4. **⚠️ TRAP:** Log file grows unbounded. A 1-second check interval writing 100 bytes per entry produces 8.6 MB per day. Without rotation, a month of data is 250+ MB.

5. **⚠️ TRAP:** PID file with stale PID. If the monitor crashes without cleanup, the PID file may contain a PID that was reassigned to another process. Always verify with `kill -0 $pid`.

6. **⚠️ TRAP:** Alert storms. A metric that fluctuates just above threshold sends an alert every check cycle. Implement cooldown periods and consecutive-failure counting.

7. **⚠️ TRAP:** Auto-recovery loops. If recovery actions fail, the next check cycle triggers recovery again, potentially damaging the system (e.g., restarting a database service every 30 seconds).

8. **⚠️ TRAP:** Units mismatch. `/proc/stat` jiffies, `/proc/meminfo` kilobytes, `df` blocks, network bytes. Always track units and convert consistently.

9. **⚠️ TRAP:** Blocking recovery. If `systemctl restart nginx` hangs (e.g., nginx is stuck stopping), the entire monitoring loop blocks. Use `timeout` for recovery commands.

10. **⚠️ TRAP:** Race condition in log rotation. If you `mv` the log file while the script has an open file descriptor, subsequent writes go to the moved file (now a different inode). Use `copytruncate` or close/reopen.

11. **⚠️ TRAP:** Memory monitoring with `free` vs `/proc/meminfo`. `free` shows "available" (includes reclaimable cache), while `MemAvailable` in `/proc/meminfo` is the kernel's best estimate. These can differ significantly.

12. **⚠️ TRAP:** Network counter overflow. 32-bit systems cap network counters at 4GB. If your interface handles high throughput, counters may wrap mid-interval, producing negative rates.

13. **⚠️ TRAP:** Filesystem stats on bind mounts. `df /var` on a bind mount may show the parent filesystem's stats, not the actual mount. Use `df --output=target` to verify.

14. **⚠️ TRAP:** `set -e` in the monitoring loop. If any command in the loop fails (e.g., `systemctl` returns non-zero), `set -e` exits the script. Use `|| true` for expected failures.

15. **⚠️ TRAP:** Running as root vs non-root. Many `/proc` counters require root. Network interface names differ (eth0 vs enp0s3). Test monitoring scripts as the user they'll run as.

## See It In The Wild

Nagios plugins are the canonical real-world example. The official Nagios Plugins project includes `check_disk` (checks disk usage with thresholds, perfdata output), `check_load` (checks load average), and `check_procs` (checks process counts) — all written as standalone scripts. They follow a strict output format:

```
OK - disk usage: 45% | /:45%;80;90 /home:23%;80;90
```

The pipe `|` separates human-readable from machine-readable (perfdata). This format is parsed by Nagios/Icinga and graphed by tools like PNP4Nagios.

Prometheus's `textfile` collector is another real-world pattern. A cron job runs a Bash script that writes metrics to a `.prom` file:

```bash
#!/bin/bash
echo "# HELP custom_metric Example custom metric"
echo "# TYPE custom_metric gauge"
echo "custom_metric $(get_value)"
```

The Prometheus node_exporter reads these files and exposes them on the metrics endpoint.

## Check Your Understanding

1. Why must CPU and network metrics use delta measurements (two samples)?

2. How does exponential backoff for alerting work, and why is it important?

3. Why is `trap 'running=false' SIGTERM` better than `trap 'exit' SIGTERM` in a monitoring loop?

4. What happens if a recovery action itself fails? How would you handle this?

5. How would you add alert throttling to prevent alert storms when a metric hovers right at the threshold?

6. What is the difference between `MemFree` and `MemAvailable` in `/proc/meminfo`?

7. Why should log files be rotated, and what issues arise if they aren't?

8. How would you add a health-check HTTP endpoint to the monitoring daemon?

9. What issues can arise from running `systemctl restart` inside a monitoring check?

10. How would you implement a `--oneshot` mode that runs one check cycle and exits (for cron integration)?
