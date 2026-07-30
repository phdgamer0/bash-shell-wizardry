# Task 19: Capstone — System Monitoring Daemon

## Objective

Build a production-grade monitoring daemon that checks 5+ system metrics, logs results with rotation, alerts when thresholds are exceeded (with exponential backoff), and performs auto-recovery actions. Must handle graceful shutdown, run as a daemon, and support configuration via config file.

## Requirements

### Metrics to Monitor

1. **CPU Usage (%)** — Two-sample delta from `/proc/stat`
2. **Memory Usage (%)** — From `/proc/meminfo` or `free`
3. **Disk Usage (%)** — From `df` for multiple mount points
4. **Inode Usage (%)** — From `df -i` for multiple mount points
5. **Network Bandwidth (KB/s)** — RX/TX rates from `/proc/net/dev`
6. **System Load Average** — 1/5/15 min from `/proc/loadavg`
7. **Process Count** — Number of running processes

### Threshold Configuration

Config file at `/etc/monitor/monitor.conf` with these defaults:

```bash
THRESHOLD_CPU=90
THRESHOLD_MEM=85
THRESHOLD_DISK=90
THRESHOLD_INODE=80
THRESHOLD_LOAD_1MIN=4.0
THRESHOLD_PROC_COUNT=500
CHECK_INTERVAL=30
LOG_FILE="/var/log/monitor/monitor.log"
ALERT_LOG="/var/log/monitor/alerts.log"
ALERT_EMAIL=""
ALERT_COOLDOWN=300
MAX_CONSECUTIVE=3
MONITOR_INTERFACES="eth0 lo"
MONITOR_MOUNTS="/:/home:/var"
SERVICES_TO_RESTART=(nginx sshd)
ENABLE_RECOVERY=true
ENABLE_NOTIFY=false
```

### Daemon Behavior

- Run as a background daemon with `--daemon` flag
- Write PID to `/var/run/monitor.pid`
- Handle `SIGTERM` and `SIGINT` for graceful shutdown
- Print summary on exit: runtime, total checks, max values
- Support `--foreground` mode for debugging

### Alerting

- Log alerts to dedicated alert log file (`ALERT_LOG`)
- Support desktop notification via `notify-send` (if ENABLE_NOTIFY=true)
- Support syslog via `logger`
- Support email (if ALERT_EMAIL is set)
- Implement exponential backoff: first alert immediately, then cooldown doubles with each consecutive alert (capped at 1 hour)
- Reset cooldown when metric returns below threshold

### Alert Severity Levels

- **INFO**: Metric approaching threshold (80-90%)
- **WARN**: Metric exceeds threshold (90-100%)
- **CRITICAL**: Metric exceeds threshold for MAX_CONSECUTIVE checks
- **RECOVERY**: Metric returns below threshold after being in alert state

### Auto-Recovery

Recovery actions trigger only after MAX_CONSECUTIVE consecutive CRITICAL alerts. Implement these recoveries:

- **CPU > 95%**: Restart services listed in `SERVICES_TO_RESTART`
- **Disk > 95%**: Clean `/tmp` files older than 1 day, rotate logs in `/var/log` older than 7 days
- **Memory > 95%**: Sync filesystems and drop caches (`echo 3 > /proc/sys/vm/drop_caches`)
- **Process count > threshold**: Kill oldest processes from /tmp

### Logging

- Log file: configured via `LOG_FILE`
- Log format: `[2026-07-31 10:00:00] [LEVEL] CPU=45% MEM=62% DISK=78% NET_RX=1.2KB/s NET_TX=0.5KB/s LOAD=1.5`
- Auto-rotate log at 10MB (compress old logs with gzip)
- Keep up to 5 rotated logs
- Colored output when logging to terminal

### Health Check Endpoint

- Listen on port 8080 (configurable) and return JSON health report
- Implement with `nc` or Bash built-in `/dev/tcp` redirection
- Return 200 OK with current metrics and status

## Sub-tasks

### 1. Project Structure

```
monitor/
├── monitor.sh           # Main daemon script
├── lib/
│   ├── metrics.sh       # Metric collection functions
│   ├── alert.sh         # Alerting functions
│   ├── recovery.sh      # Auto-recovery functions
│   ├── logging.sh       # Logging with rotation
│   └── config.sh        # Config file loading
```

### 2. Config Library (lib/config.sh)

- Load config from `/etc/monitor/monitor.conf` or `./monitor.conf`
- Fallback to hardcoded defaults
- Validate config values (must be numeric where expected)
- Log which config file was loaded
- Watch config file for changes (optional, extra credit)

### 3. Metrics Library (lib/metrics.sh)

Implement these functions:

- `get_cpu_usage()` — Returns CPU usage percentage
- `get_memory_usage()` — Returns memory usage percentage
- `get_disk_usage(mount_point)` — Returns disk usage percentage
- `get_inode_usage(mount_point)` — Returns inode usage percentage
- `get_network_rate(interface)` — Returns RX and TX rates in KB/s
- `get_load_average()` — Returns 1/5/15 min load averages
- `get_process_count()` — Returns number of running processes
- `get_uptime()` — Returns system uptime in seconds
- `collect_all_metrics()` — Returns formatted string of all metrics

### 4. Alert Library (lib/alert.sh)

- `check_threshold(metric, value, threshold)` — Returns true if threshold exceeded
- `send_alert(severity, metric, value, threshold)` — Send to all configured channels
- `send_notification(msg)` — Desktop notification via notify-send
- `send_syslog(msg)` — Write to syslog
- `send_email(subject, body)` — Send email if email configured
- `is_throttled(metric)` — Check if metric is in cooldown period
- `get_cooldown_seconds(metric)` — Calculate exponential backoff

### 5. Recovery Library (lib/recovery.sh)

- `recover_cpu()` — Restart services when CPU critically high
- `recover_disk(mount_point)` — Clean files when disk critically full
- `recover_memory()` — Drop caches when memory critically low
- `recover_processes()` — Kill oldest /tmp processes
- `log_recovery(action, result)` — Log recovery attempt
- `should_recover(metric)` — Check if MAX_CONSECUTIVE is reached

### 6. Logging Library (lib/logging.sh)

- `log(level, message)` — Write to log file with timestamp
- `rotate_log()` — Check size and rotate if needed
- `log_alert(severity, metric, value, threshold)` — Write to alert log
- `log_summary()` — Print session summary on exit

### 7. Main Daemon (monitor.sh)

- Parse CLI arguments: `--daemon`, `--foreground`, `--config FILE`, `--interval N`, `--oneshot`
- Load config
- Daemonize if `--daemon` (double fork, redirect fds)
- Write PID file
- Set up signal handlers (SIGTERM, SIGINT, SIGHUP)
- Main loop: collect, check, log, alert, recover, sleep
- Track consecutive failures per metric
- Generate and log summary on shutdown

### 8. Health Check (Extra Credit)

- Listen on TCP port 8080
- Accept connections and return JSON metrics
- Handle multiple connections (fork a subshell for each)
- Configurable port via `HEALTH_PORT` in config

## Expected Output

### Startup (foreground mode)

```
$ sudo ./monitor.sh --foreground --config ./monitor.conf
[2026-07-31 10:00:00] [INFO] Monitor starting (PID: 12345)
[2026-07-31 10:00:00] [INFO] Config loaded from: ./monitor.conf
[2026-07-31 10:00:00] [INFO] Check interval: 30s
[2026-07-31 10:00:00] [INFO] Logging to: /var/log/monitor/monitor.log
[2026-07-31 10:00:00] [INFO] Monitoring: CPU MEM DISK INODE NET LOAD PROC
[2026-07-31 10:00:00] [INFO] Check #1 — CPU=23% MEM=45% DISK=67% INODE=34% NET_RX=1.2KB/s NET_TX=0.5KB/s LOAD=0.5 PROC=128
[2026-07-31 10:00:30] [INFO] Check #2 — CPU=45% MEM=46% DISK=67% INODE=34% NET_RX=0.8KB/s NET_TX=0.4KB/s LOAD=0.6 PROC=127
[2026-07-31 10:01:00] [WARN] CPU=91% > THRESHOLD_CPU=90 (Check #3)
[2026-07-31 10:01:00] [INFO] Check #3 — CPU=91% MEM=47% DISK=67% INODE=34% NET_RX=2.1KB/s NET_TX=1.1KB/s LOAD=1.2 PROC=130
```

### Startup (daemon mode)

```
$ sudo ./monitor.sh --daemon
[INFO] Daemonized (PID: 12345)
$ ps aux | grep monitor
root     12345  0.1  0.2  12876  2344 ?        S    10:00   0:00 /bin/bash ./monitor.sh --daemon
$ cat /var/run/monitor.pid
12345
```

### Alert Sequence

```
[2026-07-31 10:01:00] [WARN] CPU at 91% (threshold: 90%)
[2026-07-31 10:01:30] [WARN] CPU at 93% (threshold: 90%)
[2026-07-31 10:02:00] [CRITICAL] CPU at 96% for 3 consecutive checks — initiating recovery
[2026-07-31 10:02:00] [RECOVERY] Restarting nginx (attempt 1)
[2026-07-31 10:02:01] [RECOVERY] nginx restart successful
[2026-07-31 10:02:30] [INFO] CPU at 12% — RECOVERED
[2026-07-31 10:02:30] [RECOVERY] CPU recovered after restarting nginx
```

### Alert Log

```
$ cat /var/log/monitor/alerts.log
2026-07-31 10:01:00|WARN|CPU|91|90
2026-07-31 10:01:30|WARN|CPU|93|90
2026-07-31 10:02:00|CRITICAL|CPU|96|90
2026-07-31 10:02:00|RECOVERY|nginx|restarted|success
2026-07-31 10:02:30|RECOVERY|CPU|normalized
```

### Disk Full Recovery

```
[2026-07-31 11:00:00] [WARN] DISK=/ at 91% (threshold: 90%)
[2026-07-31 11:00:30] [WARN] DISK=/ at 92% (threshold: 90%)
[2026-07-31 11:01:00] [CRITICAL] DISK=/ at 93% for 3 consecutive
[2026-07-31 11:01:00] [RECOVERY] Cleaning temp files on /
[2026-07-31 11:01:01] [RECOVERY] Removed 45 temp files, freed 230MB
[2026-07-31 11:01:00] [RECOVERY] Rotating old logs in /var/log
[2026-07-31 11:01:02] [RECOVERY] Rotated 3 logs, freed 45MB
[2026-07-31 11:01:30] [INFO] DISK=/ at 71% — RECOVERED
```

### Shutdown

```
$ kill -TERM 12345
[2026-07-31 11:30:00] [INFO] SIGTERM received, shutting down gracefully...
[2026-07-31 11:30:00] [INFO] Monitor stopped after 180 checks
[2026-07-31 11:30:00] [INFO] Runtime: 1h 30m 0s
[2026-07-31 11:30:00] [INFO] Session summary:
  Total checks:     180
  Alerts sent:      12 (WARN: 8, CRITICAL: 2, RECOVERY: 2)
  Recoveries:       2 (successful: 2, failed: 0)
  Max CPU:          96%
  Max MEM:          78%
  Max DISK:         93%
  Avg NET_RX:       1.5 KB/s
  Avg NET_TX:       0.6 KB/s
[INFO] PID file removed: /var/run/monitor.pid
```

### Oneshot Mode

```
$ ./monitor.sh --oneshot
[2026-07-31 12:00:00] [INFO] Check #1 — CPU=34% MEM=52% DISK=67% INODE=34% NET_RX=1.0KB/s NET_TX=0.4KB/s LOAD=0.8 PROC=132
[2026-07-31 12:00:00] [INFO] All metrics within thresholds — OK
```

### Health Check Response

```
$ curl http://localhost:8080/health
{
  "timestamp": 1722412800,
  "host": "monitor-server",
  "status": "ok",
  "checks": 180,
  "uptime_seconds": 5400,
  "metrics": {
    "cpu": {"value": 34, "unit": "%", "status": "ok"},
    "memory": {"value": 52, "unit": "%", "status": "ok"},
    "disk": {
      "/": {"value": 67, "unit": "%", "status": "ok"},
      "/home": {"value": 42, "unit": "%", "status": "ok"},
      "/var": {"value": 71, "unit": "%", "status": "warn"}
    },
    "network": {
      "eth0": {"rx_kbps": 1.0, "tx_kbps": 0.4},
      "lo": {"rx_kbps": 0.1, "tx_kbps": 0.1}
    },
    "load": {"1min": 0.8, "5min": 0.6, "15min": 0.5},
    "processes": {"count": 132, "threshold": 500, "status": "ok"}
  },
  "alerts_active": 0,
  "recoveries": 2
}
```

### Config Validation

```
$ sudo ./monitor.sh --config /nonexistent/config
[ERROR] Config file not found: /nonexistent/config
[INFO] Using built-in defaults

$ sudo ./monitor.sh --config /etc/monitor/monitor.conf.typo
[ERROR] Invalid value in config: THRESHOLD_CPU=abc (must be numeric)
[ERROR] Using built-in defaults
```

## Hints

<details>
<summary>Hint 1: Daemonization Double Fork</summary>

```bash
daemonize() {
  # First fork
  case "$(fork)" in
    0) ;; # Child continues
    *) exit 0 ;; # Parent exits
  esac
  # Create new session
  command setsid </dev/null >/dev/null 2>&1 || true
  # Second fork
  case "$(fork)" in
    0) ;; # Grandchild continues
    *) exit 0 ;; # Child exits
  esac
  # Redirect stdin/stdout/stderr
  exec 0</dev/null
  exec 1>>"$LOG_FILE"
  exec 2>&1
  # Write PID file
  echo $$ > "$PID_FILE"
}
```
</details>

<details>
<summary>Hint 2: Exponential Backoff for Alerts</summary>

```bash
get_cooldown() {
  local metric=$1
  local consecutive=${alert_count[$metric]:-0}
  local base_cooldown=${ALERT_COOLDOWN:-300}
  local max_cooldown=${MAX_ALERT_COOLDOWN:-3600}
  # Exponential: base * 2^consecutive, capped at max
  local cooldown=$(( base_cooldown * (2 ** consecutive) ))
  (( cooldown > max_cooldown )) && cooldown=$max_cooldown
  echo "$cooldown"
}
```
</details>

<details>
<summary>Hint 3: Log Rotation with Copy-Truncate</summary>

```bash
rotate_log() {
  local log_file=$1
  local max_size=${2:-10485760}
  [[ ! -f "$log_file" ]] && return
  local size
  size=$(stat -c%s "$log_file" 2>/dev/null) || return
  if ((size > max_size)); then
    # Copy and truncate (safer than mv for open file handles)
    cp "$log_file" "${log_file}.1"
    : > "$log_file"
    gzip "${log_file}.1" 2>/dev/null || true
    # Clean old logs (keep last 5)
    ls -t "${log_file}".*.gz 2>/dev/null | tail -n +6 | xargs rm -f 2>/dev/null || true
  fi
}
```
</details>

<details>
<summary>Hint 4: Health Check with /dev/tcp</summary>

```bash
health_check_server() {
  local port=${HEALTH_PORT:-8080}
  while $running; do
    # Listen using bash built-in TCP
    exec 3<>"/dev/tcp/0.0.0.0/$port" 2>/dev/null || {
      sleep 5
      continue
    }
    # Read HTTP request
    read -r request <&3
    # Send response
    echo -e "HTTP/1.1 200 OK\r\nContent-Type: application/json\r\n\r\n$(generate_json_report)" >&3
    exec 3>&-
  done
}
```
</details>

<details>
<summary>Hint 5: Consecutive Failure Tracking</summary>

```bash
declare -A consecutive_failures

check_and_alert() {
  local metric=$1 value=$2 threshold=$3
  if (( $(echo "$value > $threshold" | bc -l) )); then
    ((consecutive_failures[$metric]++))
    local count=${consecutive_failures[$metric]}
    if ((count >= MAX_CONSECUTIVE)); then
      send_alert "CRITICAL" "$metric" "$value" "$threshold"
      $ENABLE_RECOVERY && trigger_recovery "$metric"
    elif ((count == 1)); then
      send_alert "WARN" "$metric" "$value" "$threshold"
    fi
  else
    if ((consecutive_failures[$metric] >= MAX_CONSECUTIVE)); then
      send_alert "RECOVERY" "$metric" "$value" "$threshold"
    elif ((consecutive_failures[$metric] > 0)); then
      log info "$metric recovered (was above threshold for ${consecutive_failures[$metric]} checks)"
    fi
    consecutive_failures[$metric]=0
  fi
}
```
</details>

<details>
<summary>Hint 6: Timeout for Recovery Commands</summary>

```bash
safe_restart() {
  local service=$1
  local timeout=${2:-30}
  timeout "$timeout" systemctl restart "$service" 2>/dev/null
  local result=$?
  if ((result == 124)); then
    log error "Timeout restarting $service (killed after ${timeout}s)"
    return 1
  elif ((result != 0)); then
    log error "Failed to restart $service (exit code: $result)"
    return 1
  fi
  log info "Successfully restarted $service"
  return 0
}
```
</details>

<details>
<summary>Hint 7: SIGHUP Config Reload</summary>

```bash
reload_config() {
  log info "SIGHUP received, reloading config"
  load_config "$CONFIG_FILE"
  log info "Config reloaded"
}

trap 'reload_config' SIGHUP
```
</details>

<details>
<summary>Hint 8: CPU Temperature (Extra Metric)</summary>

```bash
get_cpu_temp() {
  local temp
  if [[ -f /sys/class/thermal/thermal_zone0/temp ]]; then
    temp=$(cat /sys/class/thermal/thermal_zone0/temp)
    echo "$((temp / 1000))" # Convert millidegrees to Celsius
  elif command -v sensors &>/dev/null; then
    sensors -u 2>/dev/null | awk '/temp1_input/ {print $2; exit}'
  else
    echo "0"
  fi
}
```
</details>

## Self-Check

1. Why must CPU and network metrics use delta measurements instead of single readings?

2. How does the exponential backoff (cooldown) for alerts work, and what problem does it solve?

3. Why is `trap 'running=false' SIGTERM` better than `trap 'exit' SIGTERM` in a monitoring loop?

4. What happens if a recovery action itself fails? Design a strategy for handling this.

5. How would you add alert throttling to prevent alert storms when a metric hovers right at the threshold?

6. What is the difference between `MemFree` and `MemAvailable` in `/proc/meminfo`?

7. Why should log files use copy-truncate rotation instead of move-and-create when the script has the file open?

8. How would you implement a `--oneshot` mode for cron integration?

9. What issues can arise from running `systemctl restart` inside a monitoring check, and how does `timeout` help?

10. How would you add support for alerting via Slack webhook or other HTTP endpoint?
