# Task 13: Defense Toolkit

## Objective

Build a modular defense toolkit with three primary components: a file integrity monitor, a process anomaly detector, and an SSH login alert system. The toolkit must log to a dedicated directory, produce severity-rated reports, and run in both one-shot and daemon modes.

## Requirements (8 sub-tasks)

### Sub-task 1: File Integrity Monitor
Write `modules/integrity.sh` with:
- `init_baseline WATCH_DIR` — Recursively find all regular files, compute SHA256 hashes, store in `~/.defense_toolkit/baselines/`
- `check_integrity WATCH_DIR` — Compare current hashes against baseline, report modified/new/deleted files
- `update_baseline WATCH_DIR` — Re-compute baseline (use after authorized changes)
- All findings reported with severity: MODIFIED=HIGH, NEW=MEDIUM, DELETED=HIGH, OK=INFO

### Sub-task 2: Process Anomaly Detector
Write `modules/process_mon.sh` that checks:
- Processes running from /tmp, /dev/shm, /var/tmp (HIGH)
- Processes with missing executables (readlink /proc/PID/exe fails) (CRITICAL)
- Processes with hidden names (containing control chars, leading dots, spaces) (HIGH)
- Processes whose PPID is unexpected (e.g., a network service spawning bash) (MEDIUM)
- Processes listening on unexpected ports > 1024 without known service (MEDIUM)

### Sub-task 3: SSH Login Alert
Write `modules/ssh_monitor.sh` that:
- Tails `/var/log/auth.log` in real-time
- Parses successful logins: `Accepted password|publickey` — user, IP, port, auth method
- Parses failed logins: `Failed password` — user, IP, attempt count
- Tracks failed attempts per IP — alert if > 3 in 60 seconds (MEDIUM -> HIGH if > 10)
- Logs all events with timestamps to `/var/log/defense_toolkit/ssh.log`

### Sub-task 4: Logging Library
Write `lib/logging.sh` with:
- `init_logging LOG_DIR` — Create log directory with secure permissions (750)
- `log_event LEVEL MESSAGE` — Write to log file with ISO8601 timestamp
- `log_rotate` — Rotate log when > 10MB (compress old, keep 5 rotations)
- Support for both text and JSON log formats

### Sub-task 5: Main Dispatch
Write `defense_toolkit.sh` that:
- Sources all modules from `modules/` and `lib/`
- Supports `--all`, `--integrity`, `--process`, `--ssh`, `--daemon` modes
- `--daemon` runs SSH monitor continuously in background, integrity check every hour, process check every 5 minutes
- Handles SIGTERM/SIGINT for graceful shutdown in daemon mode
- Writes PID file to `/var/run/defense_toolkit.pid`
- Config file support: `/etc/defense_toolkit.conf` or `~/.config/defense_toolkit.conf`

### Sub-task 6: Alerting
Write `lib/alert.sh` with:
- `send_notification URGENCY TITLE MESSAGE` — Desktop notification via notify-send
- `send_syslog URGENCY TITLE MESSAGE` — via logger
- `send_json URGENCY TITLE MESSAGE` — JSON to stdout (for SIEM integration)
- Threshold-based deduplication: don't re-alert on the same finding within 5 minutes

### Sub-task 7: Reporting
Write `lib/report.sh` with:
- `generate_report` — Summary of all findings since last report
- Severity counts: CRITICAL, HIGH, MEDIUM, LOW, INFO
- Timeline of events
- Top 5 most frequent findings
- Exit code based on highest severity found

### Sub-task 8: Test Suite
Write `test/test_defense.sh` that:
- Creates test scenarios (modified files, suspicious processes, fake SSH logins)
- Runs each module and verifies correct detection
- Tests daemon mode start/stop
- Tests log rotation
- Reports PASS/FAIL for each test case

## Bonus Challenges

1. **Remote logging:** Add syslog forwarding to a remote server via `logger -n`
2. **File quarantine:** Add option to move modified files to a quarantine directory instead of just reporting
3. **Baseline signing:** GPG-sign the baseline files and verify signature before use
4. **eBPF alternative:** Research and document how eBPF-based monitoring (bpftrace, bcc) would improve detection vs. bash-based monitoring
5. **Honeypot integration:** Create a fake "secret" file (honeypot) that triggers an immediate alert if read

## Hints

<details>
<summary>Hint 1: File integrity with find + sha256sum</summary>

```bash
init_baseline() {
  local watch_dir="$1" name=$(echo "$watch_dir" | tr '/' '_')
  local baseline="$BASELINE_DIR/${name}.sha256"
  find "$watch_dir" -type f -exec sha256sum {} \; 2>/dev/null > "$baseline"
  echo "Baseline created: $baseline ($(wc -l < "$baseline") files)"
}

check_integrity() {
  local watch_dir="$1" name=$(echo "$watch_dir" | tr '/' '_')
  local baseline="$BASELINE_DIR/${name}.sha256"
  [ ! -f "$baseline" ] && { echo "No baseline for $watch_dir"; return 1; }

  # Check modified files
  while IFS= read -r result; do
    file=$(echo "$result" | cut -d: -f1)
    echo "HIGH|MODIFIED|$file"
  done < <(sha256sum -c "$baseline" 2>&1 | grep ': FAILED$')

  # Check new files
  while IFS= read -r file; do
    echo "MEDIUM|NEW|$file"
  done < <(comm -13 <(awk '{print $2}' "$baseline" | sort) <(find "$watch_dir" -type f | sort) 2>/dev/null)

  # Check deleted files
  while IFS= read -r file; do
    echo "HIGH|DELETED|$file"
  done < <(comm -23 <(awk '{print $2}' "$baseline" | sort) <(find "$watch_dir" -type f | sort) 2>/dev/null)
}
```
</details>

<details>
<summary>Hint 2: Process anomaly patterns</summary>

```bash
check_process_anomalies() {
  local tmp_pids=""

  # Processes from suspicious locations
  echo "CHECK: Processes from temp directories"
  ps -eo pid,user,args | grep -E '\s(/tmp|/dev/shm|/var/tmp)\S+' || true

  # Missing executables
  echo "CHECK: Missing executables"
  for pid in /proc/[0-9]*; do
    p=$(basename "$pid")
    cmdline=$(tr '\0' ' ' < "$pid/cmdline" 2>/dev/null)
    exe=$(readlink -f "$pid/exe" 2>/dev/null)
    if [ -z "$exe" ] && [ "$p" -gt 10 ]; then
      echo "PID $p ($cmdline) — executable not found on disk"
    fi
  done

  # Processes renamed to look like kernel threads
  echo "CHECK: Masquerading processes"
  for pid in /proc/[0-9]*; do
    cmdline=$(tr '\0' ' ' < "$pid/cmdline" 2>/dev/null)
    exe=$(readlink -f "$pid/exe" 2>/dev/null)
    if echo "$cmdline" | grep -qE '\[kworker|\[kthread|\[watchdog|\[ksoftirqd'; then
      if [ "$exe" != "/sbin/kthreadd" ] && [ "$exe" != "/" ]; then
        echo "PID $pid claims to be kernel thread but exe is $exe"
      fi
    fi
  done
}
```
</details>

<details>
<summary>Hint 3: SSH login alert with rate limiting</summary>

```bash
declare -A FAIL_COUNTS
FAIL_THRESHOLD=3
ALERTED_IPS=""

check_ssh_line() {
  local line="$1"

  # Successful login
  if echo "$line" | grep -qE 'Accepted (password|publickey)'; then
    local user=$(echo "$line" | awk '{print $9}')
    local ip=$(echo "$line" | awk '{print $11}')
    local method=$(echo "$line" | grep -oE 'password|publickey')
    log_event "INFO" "SSH_LOGIN: user=$user ip=$ip method=$method"
  fi

  # Failed login
  if echo "$line" | grep -q 'Failed password'; then
    local user=$(echo "$line" | awk '{print $9}')
    local ip=$(echo "$line" | awk '{print $11}')
    ((FAIL_COUNTS[$ip]++))
    local count=${FAIL_COUNTS[$ip]}

    if [ "$count" -eq "$FAIL_THRESHOLD" ]; then
      log_event "MEDIUM" "SSH_FAIL_THRESHOLD: ip=$ip count=$count user=$user"
      send_alert "normal" "SSH Brute Force" "IP $ip exceeded $FAIL_THRESHOLD failed attempts"
    fi
    if [ "$count" -eq 10 ]; then
      log_event "HIGH" "SSH_BRUTE_FORCE: ip=$ip count=$count user=$user"
    fi
  fi
}

# Usage: tail -Fn0 /var/log/auth.log | while read line; do check_ssh_line "$line"; done
```
</details>

<details>
<summary>Hint 4: Logging library with rotation</summary>

```bash
LOG_DIR="${LOG_DIR:-/var/log/defense_toolkit}"
LOG_FILE="$LOG_DIR/toolkit.log"
MAX_LOG_SIZE=$((10 * 1024 * 1024))  # 10MB

init_logging() {
  local dir="${1:-$LOG_DIR}"
  mkdir -p "$dir" 2>/dev/null || sudo mkdir -p "$dir"
  chmod 750 "$dir" 2>/dev/null
  LOG_DIR="$dir"
  LOG_FILE="$dir/toolkit.log"
}

log_event() {
  local level="$1" message="$2"
  local timestamp=$(date '+%Y-%m-%dT%H:%M:%S%z')
  echo "$timestamp|$level|$message" >> "$LOG_FILE"
  log_rotate
}

log_rotate() {
  [ -f "$LOG_FILE" ] || return
  local size=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
  [ "$size" -lt "$MAX_LOG_SIZE" ] && return

  for i in 4 3 2 1; do
    [ -f "${LOG_FILE}.${i}.gz" ] && mv "${LOG_FILE}.${i}.gz" "${LOG_FILE}.$((i+1)).gz"
  done
  mv "$LOG_FILE" "${LOG_FILE}.1"
  gzip "${LOG_FILE}.1"
  touch "$LOG_FILE"
}
```
</details>

<details>
<summary>Hint 5: Daemon mode with graceful shutdown</summary>

```bash
RUNNING=true
PID_FILE="/var/run/defense_toolkit.pid"

cleanup() {
  RUNNING=false
  log_event "INFO" "Shutting down defense toolkit"
  rm -f "$PID_FILE"
  exit 0
}

trap cleanup SIGTERM SIGINT

start_daemon() {
  log_event "INFO" "Starting defense toolkit daemon (PID $$)"
  echo $$ > "$PID_FILE"

  while $RUNNING; do
    check_integrity "/etc"
    check_process_anomalies
    sleep "$CHECK_INTERVAL"
  done
}
```
</details>

<details>
<summary>Hint 6: Testing with simulated events</summary>

```bash
test_ssh_detection() {
  echo "TEST: SSH login detection"
  # Simulate auth.log entries
  echo "Jul 31 10:00:00 host sshd[1234]: Accepted password for alice from 192.168.1.100 port 22 ssh2" | \
    while read line; do check_ssh_line "$line"; done

  # Verify
  grep -q 'SSH_LOGIN.*alice.*192.168.1.100' "$LOG_FILE" && echo "  PASS" || echo "  FAIL"
}

test_integrity() {
  echo "TEST: File integrity detection"
  init_baseline "/tmp/test_integrity"
  echo "CHANGED" > /tmp/test_integrity/test.txt
  check_integrity "/tmp/test_integrity" | grep -q 'MODIFIED' && echo "  PASS" || echo "  FAIL"
}
```
</details>

<details>
<summary>Hint 7: Main dispatcher with argument parsing</summary>

```bash
SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
for lib in "$SCRIPT_DIR/lib"/*.sh; do source "$lib"; done
for mod in "$SCRIPT_DIR/modules"/*.sh; do source "$mod"; done

MODE="${1:-help}"
case "$MODE" in
  --all)
    init_logging
    init_baseline "/etc"
    check_integrity "/etc"
    check_process_anomalies
    ;;
  --daemon)
    init_logging
    start_daemon
    ;;
  --integrity)
    init_logging
    init_baseline "${2:-/etc}"
    check_integrity "${2:-/etc}"
    ;;
  --ssh)
    init_logging
    tail -Fn0 /var/log/auth.log | while read line; do
      check_ssh_line "$line"
    done
    ;;
  --help|*)
    echo "Usage: $0 {--all|--daemon|--integrity [dir]|--process|--ssh|--help}"
    ;;
esac
```
</details>

## Expected Output

```bash
$ sudo ./defense_toolkit.sh --daemon

=== Defense Toolkit v1.0 ===
[2026-07-31T10:00:00+0000] [INFO] Starting defense toolkit daemon (PID 12345)
[2026-07-31T10:00:00+0000] [INFO] Logging to /var/log/defense_toolkit/toolkit.log
[2026-07-31T10:00:00+0000] [INFO] Integrity check initialized for /etc (1432 files)
[2026-07-31T10:00:00+0000] [INFO] Process monitor initialized
[2026-07-31T10:00:00+0000] [INFO] SSH monitor initialized (following /var/log/auth.log)
[2026-07-31T10:00:05+0000] [HIGH] INTEGRITY: /etc/shadow — MODIFIED (hash: abc123 -> def456)
[2026-07-31T10:00:05+0000] [MEDIUM] INTEGRITY: /etc/hosts — NEW FILE
[2026-07-31T10:00:10+0000] [HIGH] PROCESS: PID 23456 (/tmp/.crypto) — running from /tmp
[2026-07-31T10:00:10+0000] [CRITICAL] PROCESS: PID 23457 — executable not found on disk
[2026-07-31T10:00:15+0000] [INFO] SSH_LOGIN: user=alice ip=192.168.1.100 method=publickey
[2026-07-31T10:00:20+0000] [MEDIUM] SSH_FAIL_THRESHOLD: ip=10.0.0.5 count=3 user=root
[2026-07-31T10:00:25+0000] [HIGH] SSH_BRUTE_FORCE: ip=10.0.0.5 count=10 user=admin

$ sudo ./defense_toolkit.sh --report

=== Defense Toolkit Report ===
Period: 2026-07-31 10:00:00 to 2026-07-31 11:00:00
Total events: 47

=== Findings by Severity ===
CRITICAL: 1  — Process with no executable
HIGH:     3  — File modified (2), Process in /tmp (1)
MEDIUM:   5  — SSH brute force (2), New files (3)
LOW:      2  — Unexpected PPID relationships
INFO:     36  — Normal operations

=== Top Findings ===
1. SSH login from 192.168.1.100 (12 times — authorized user)
2. Failed SSH from 10.0.0.5 (10 times — brute force)
3. File changes in /etc (3 files)

=== Summary ===
Overall: 🔴 ISSUES FOUND (1 CRITICAL, 3 HIGH)
Recommendation: Investigate PID 23457 (no executable) and /etc/shadow modification
```

## Self-Check Questions

1. How can an attacker evade `sha256sum` integrity checking if they have root access? What are three methods?

2. Why would a process have no binary on disk (exe missing)? What does this indicate about the attacker's methods?

3. How can an attacker prevent their commands from appearing in `auth.log`? What would need to be disabled?

4. What are the limitations of using `inotifywait` for security monitoring on a production system?

5. How would you protect the defense toolkit's own logs from tampering by an attacker?

6. Why is `tail -F` preferred over `tail -f` for log monitoring? What happens during log rotation?

7. How can an attacker exhaust inotify watch limits to blind your monitoring?

8. What's the difference between monitoring `/proc/PID/exe` vs `/proc/PID/cmdline` for process detection?
