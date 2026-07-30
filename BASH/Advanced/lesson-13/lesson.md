# Lesson 13: Exploitation — Defense & Detection (DEFENSE FOCUSED)

## History & Origins

Defense against system exploitation has always been an arms race. In the early days, defense meant "don't let untrusted users log in." As systems became networked and multi-user, the need for automated detection grew.

The concept of file integrity monitoring (FIM) dates to the 1990s with Tripwire (1992), the first tool that computed cryptographic hashes of system files and alerted on changes. Gene Kim and Dr. Eugene Spafford created Tripwire after the 1988 Morris Worm showed how hard it was to detect file modifications.

The audit subsystem (`auditd`) arrived in the Linux 2.6 kernel (2003), providing kernel-level logging of security-relevant events. Before auditd, administrators relied on `syslog` and application-level logging — too easy for attackers to tamper with.

`inotify` (2005) replaced the older `dnotify` mechanism, giving user-space programs the ability to monitor filesystem events in real-time. This was a game-changer: instead of polling files for changes every N seconds, you could now receive immediate notifications.

Shell history monitoring has existed as long as shells themselves, but `HISTTIMEFORMAT` (added in bash 3.0) made it possible to timestamp history entries. Attackers quickly learned to disable it or edit the history file directly.

Modern defense-in-depth for bash environments combines:
- **Prevention:** Restricted shells, SELinux, capabilities, read-only filesystems
- **Detection:** Integrity monitoring, auditd, inotify watchers, log analysis
- **Response:** Automated containment, alerting, forensic capture

This lesson builds custom detection tools using only bash and standard Linux utilities — no agents, no commercial tools, no dependencies.

## Syntax Reference

```
# File Integrity
sha256sum /path/to/file              # Compute SHA-256 hash
sha256sum -c checksums.sha256         # Verify against a checksum file
find /etc -type f -exec sha256sum {} \; > baseline.sha256
diff <(sort baseline.sha256) <(find /etc -type f -exec sha256sum {} \; | sort)

# Inotify (inotifywait from inotify-tools)
inotifywait -m -r -e modify,create,delete /path   # Monitor events
inotifywait -m /path --format '%w%f %e %T' --timefmt '%H:%M:%S'
inotifywatch -t 60 /path                           # Collect event stats

# auditd
auditctl -w /path/to/file -p wa -k key_name        # Add watch rule
auditctl -l                                         # List rules
ausearch -k key_name --start today                  # Search audit log
aureport -f                                         # File access report

# Process Monitoring
ps -eo pid,ppid,user,cwd,args
ls -la /proc/PID/exe /proc/PID/root /proc/PID/fd/
lsof -i -P -n                                       # Network connections
ss -tulanp                                          # Listening sockets

# Shell History
history                                             # Show history
history -w                                          # Write to HISTFILE
HISTTIMEFORMAT='%F %T '                             # With timestamps
HISTFILESIZE=10000                                   # Max lines in file
HISTSIZE=5000                                       # Max lines in memory
HISTFILE=~/.bash_history                            # History file path

# Log Management
logger -t mytag "message"                           # Send to syslog
tail -Fn0 /var/log/auth.log                         # Follow log in real-time
grep 'Accepted\|Failed password' /var/log/auth.log  # SSH login analysis
journalctl -u service-name -f                       # systemd journal follow

# Network Monitoring
tcpdump -i any port 80 -w capture.pcap             # Packet capture
ngrep -d any -W byline port 22                      # Pattern match on wire
ss -tnp                                             # TCP connections with PID
```

## Under the Hood (GO DEEP)

### Kernel/OS Mechanics of Detection Tools

**auditd Internals:**

The Linux Audit subsystem lives in the kernel. Key components:
1. **auditctl** — user-space tool that sends rules to the kernel via `netlink` socket
2. **kauditd** — kernel thread that generates audit events
3. **auditd** — user-space daemon that reads events from the kernel via `netlink` and writes to `/var/log/audit/audit.log`

When an audit rule matches (e.g., `auditctl -w /etc/shadow -p wa`), the kernel:
1. Intercepts the syscall (open, write, chmod, etc.)
2. Checks if the target inode matches any watched path
3. If yes, allocates an audit record with: timestamp, syscall info, UID, path, result
4. Sends the record via `netlink_unicast()` to user-space auditd
5. auditd writes it to disk

**auditd is NOT real-time** — there's a small delay between event and logging. For true real-time monitoring, use `inotify`.

**inotify Kernel Mechanics:**

inotify uses three syscalls:
- `inotify_init()` — creates an inotify instance (returns a file descriptor)
- `inotify_add_watch(fd, path, mask)` — adds a watch on a path for specified events
- `inotify_rm_watch(fd, wd)` — removes a watch

When a watched event occurs, the kernel writes an `inotify_event` struct to the FD:
```c
struct inotify_event {
    int      wd;       /* Watch descriptor */
    uint32_t mask;     /* Mask of events */
    uint32_t cookie;   /* Unique cookie for related events (e.g., rename) */
    uint32_t len;      /* Size of name field */
    char     name[];   /* Optional name (for directory watches) */
};
```

The key limitation: inotify watches are per-inode, not per-path. If a watched file is deleted and recreated, the new file has a new inode, and the old watch is lost. `inotifywait -m` handles this for directories by re-adding watches, but for individual files you must handle IN_DELETE_SELF.

**Max watch limit:** `/proc/sys/fs/inotify/max_user_watches` (default 8192) limits the number of watches per user. When exceeded, `inotify_add_watch()` returns -1 with `ENOSPC`. This is why monitoring `/` recursively will quickly exhaust watches.

**Shell History File Mechanics:**

bash writes history to `$HISTFILE` (default `~/.bash_history`) when:
- The shell exits (if `histappend` is set, it appends; otherwise, overwrites)
- `history -w` is called
- `history -a` appends the current session's new entries

The history file is a plain text file. Attackers exploit this:
- `unset HISTFILE` before running commands — no history written
- `set +o history` — disables history for the current session
- `history -c && history -w` — clears and rewrites history with nothing
- Directly editing `~/.bash_history` — remove incriminating lines
- Symlink `~/.bash_history` to `/dev/null` — discards all history

**Defense:** Set `HISTFILE` to a location only root can write, or use `auditd` to log all commands via `pam_tty_audit.so`.

### What strace Reveals

```bash
# inotifywait initialization:
$ strace -f inotifywait -m /tmp/test 2>&1 | head -10
inotify_init()                          = 3
inotify_add_watch(3, "/tmp/test", IN_MODIFY|IN_CREATE|IN_DELETE) = 1
read(3,                                  # Blocks here waiting for events

# auditd rule application:
$ sudo strace auditctl -w /etc/shadow -p wa -k shadow_watch 2>&1
socket(AF_NETLINK, SOCK_RAW|SOCK_CLOEXEC, NETLINK_AUDIT) = 3
sendto(3, {nlmsg_type=AUDIT_ADD_RULE, ...}, 96, 0, ...) = 96

# sha256sum reading a file:
$ strace sha256sum /etc/passwd 2>&1 | grep open
openat(AT_FDCWD, "/etc/passwd", O_RDONLY) = 3

# Process audit:
$ strace -e openat ps aux 2>&1 | grep -E '^openat.*/proc'
openat(AT_FDCWD, "/proc/version", O_RDONLY) = 3
openat(AT_FDCWD, "/proc/1/status", O_RDONLY) = 4
openat(AT_FDCWD, "/proc/1/cmdline", O_RDONLY) = 4
# ps reads /proc directly — no special permissions needed for our own processes
```

### Security Model Interactions

**SELinux and Auditing:**
- SELinux generates AUDIT events for denials automatically (type=AVC)
- These appear in the audit log and can be monitored with `ausearch -m AVC`
- SELinux can also confine auditd itself (auditd_t domain) — restricting what auditd can access

**Capabilities and Detection:**
- `CAP_AUDIT_WRITE` — allows a process to write to the audit log (used by container runtimes)
- `CAP_AUDIT_CONTROL` — allows managing audit rules (needed by auditctl)
- `CAP_SYS_PTRACE` — needed for inspecting other processes' /proc entries
- Detection scripts often run as root to access all /proc entries — this is also the attacker's first target

**Namespaces and Visibility:**
- PID namespaces: processes inside a container can only see their own namespace
- Mount namespaces: /proc inside a container is the container's proc, not the host's
- To detect host-level issues from inside a container, you need host PID namespace (`--pid=host`) and host /proc

### Process/Memory Implications

**Integrity Monitoring:**
- Computing SHA256 of `/etc` (1000+ files) takes 10-30 seconds on modern hardware
- During this time, the script uses ~100% CPU for one core
- Memory usage is minimal: each hash is ~60 bytes, stored in a file
- For large directories, schedule scans during off-peak hours

**inotify Watchers:**
- Each watch consumes ~1KB of kernel memory
- 8192 default watches = ~8MB kernel memory
- The `inotifywait` process itself is small (~1MB RSS) but runs continuously
- On low-memory systems, reduce watch count or use polling instead

**auditd:**
- auditd runs as a daemon with ~10MB RSS
- Kernel audit buffer is 100MB by default (check `/proc/sys/kernel/audit_backlog_limit`)
- Under high event load, the backlog can fill up and events are dropped (`audit: backlog limit exceeded`)
- Log files grow quickly — configure `max_log_file` and `max_log_file_action` in `/etc/audit/auditd.conf`

## Core Examples (15 total)

### Example 1: File integrity baseline creation

```bash
$ BASEDIR="$HOME/.integrity_db"
$ mkdir -p "$BASEDIR"
$ find /etc -type f -exec sha256sum {} \; 2>/dev/null > "$BASEDIR/etc_baseline.sha256"
$ wc -l "$BASEDIR/etc_baseline.sha256"
1432 /home/user/.integrity_db/etc_baseline.sha256
```

**Anatomy:** Creates a SHA256 hash of every regular file in `/etc`. Each line: `<hash>  <path>`. The two spaces between hash and path are important for `sha256sum -c` to work.

**Variations:** For Docker containers, baseline the image layer. For system files, baseline against package manager hashes (`dpkg --verify` on Debian, `rpm -V` on Red Hat).

**Edge case:** Files that change frequently (logs, lock files, temporary configs) generate false positives. Exclude patterns: `-not -name '*.log' -not -path '*/lock'`.

### Example 2: Integrity check — detecting changes

```bash
$ sha256sum -c "$BASEDIR/etc_baseline.sha256" 2>&1 | grep -v ': OK$'
```

**Output if compromised:**
```
/etc/shadow: FAILED
/etc/crontab: FAILED
/opt/backup.sh: FAILED (new file — not in baseline)
```

**Anatomy:** `sha256sum -c` reads the hash file, computes each file's current hash, and reports FAILED for mismatches. Files not in the baseline at all are not checked — you need a separate new-file detection step.

**Variations:** Check for new files: `diff <(awk '{print $2}' baseline) <(find /etc -type f | sort)`.

**Edge case:** If the hash file itself is tampered with, the check is meaningless. Store the hash file on read-only media or sign it with GPG.

### Example 3: Real-time file monitoring with inotifywait

```bash
$ inotifywait -m -r /etc/myapp -e modify,create,delete --format '%w%f %e %T' --timefmt '%H:%M:%S'
```

**Output:**
```
/etc/myapp/config.yml MODIFY 10:00:05
/etc/myapp/plugins/ CREATE 10:01:12
```

**Anatomy:** `-m` (monitor) keeps running after the first event. `-r` recurses into subdirectories. `-e` specifies event types. `--format` controls output (path, event, time).

**Variations:** Use `--exclude '\.(log|swp)$'` to ignore specific patterns. Use `--excludei` for case-insensitive exclusion.

**Edge case:** inotify watches consume kernel memory. `max_user_watches` default is 8192. Monitoring `/etc` recursively might use hundreds of watches depending on depth. Check with `cat /proc/sys/fs/inotify/max_user_watches`.

### Example 4: Process anomaly detection — suspicious locations

```bash
$ ps -eo pid,args | grep -E '/tmp/|/dev/shm/|/var/tmp/'
```

**Output if suspicious:**
```
3456 /tmp/cryptominer --config=pool.xyz
4567 /dev/shm/.hidden/script.sh
```

**Anatomy:** Legitimate processes rarely run from `/tmp`, `/dev/shm`, or `/var/tmp`. These are temp directories — running executables from them is a strong indicator of malicious activity.

**Variations:** Check for hidden filenames: `ps -eo pid,args | grep -E '^\.|/\.'`. Check for processes with spaces in the path (anti-forensics): `ps -eo pid,args | grep -E '/tmp/[A-Za-z0-9]+ +[A-Za-z]'`.

**Edge case:** Some legitimate applications use `/tmp` for unpacking and running. Steam, some installers, and self-extracting archives do this. Check the context — a process named `java` in `/tmp` is suspicious; an installer unpacker may be legitimate.

### Example 5: Process anomaly — check for missing executables

```bash
$ for pid in /proc/[0-9]*; do
    exe=$(readlink -f "$pid/exe" 2>/dev/null) || echo "PID $(basename $pid): missing executable"
  done
```

**Output:**
```
PID 5678: missing executable
```

**Anatomy:** `/proc/PID/exe` is a symlink to the executable's path. If the binary has been deleted (the file is gone but the process is still running), `readlink` fails. This is a classic technique — run a payload, delete the binary, and the process runs from kernel memory (the inode still exists until the process exits).

**Variations:** Check `readlink -f /proc/PID/exe` against expected paths. A process claiming to be `sshd` but running from `/tmp` is immediately suspicious.

**Edge case:** Some kernel processes have no userspace executable (`kworker`, `kthreadd`). Their `/proc/PID/exe` is a broken symlink by design. Filter them by parent PID (kthreadd is PPID 2).

### Example 6: auditd — Watch critical files

```bash
$ sudo auditctl -w /etc/shadow -p wa -k shadow_watch
$ sudo auditctl -w /etc/crontab -p wa -k cron_watch
$ sudo auditctl -w /etc/ssh/sshd_config -p wa -k sshd_config_watch
$ sudo auditctl -l
```

**Output:**
```
-w /etc/shadow -p wa -k shadow_watch
-w /etc/crontab -p wa -k cron_watch
-w /etc/ssh/sshd_config -p wa -k sshd_config_watch
```

**Anatomy:** `-w` specifies the path. `-p wa` watches for write and attribute changes. `-k` is a key (label) for searching. Without adding to `/etc/audit/rules.d/`, rules are lost on reboot.

**Variations:** Monitor directories recursively: `auditctl -w /etc/ -p wa`. Monitor syscalls directly: `auditctl -a exit,always -S open -F uid!=0`.

**Edge case:** Too many audit rules can flood the audit log and cause performance issues. Be selective. A rule like `-w /etc -p wa` on a busy system generates thousands of events per minute.

### Example 7: ausearch — Searching audit logs

```bash
$ sudo ausearch -k shadow_watch --start today --format text
```

**Output:**
```
----
time->Wed Jul 31 10:00:00 2026
type=PROCTITLE msg=audit(1722416400.123:456): proctitle=76696D202F6574632F736861646F77
type=PATH msg=audit(1722416400.123:456): item=0 name="/etc/shadow" inode=123456 dev=08:01 mode=0100644 ouid=0 ogid=0 rdev=00:00 nametype=NORMAL cap_fp=0 cap_fi=...
type=SYSCALL msg=audit(1722416400.123:456): arch=c000003e syscall=2 success=yes exit=3 ... comm="vim" exe="/usr/bin/vim.basic"
```

**Anatomy:** Each audit event includes:
- `PROCTITLE` — hex-encoded process title (decoded: `vim /etc/shadow`)
- `PATH` — the file accessed, with inode and mode
- `SYSCALL` — the syscall number (2 = open), with comm (command name) and exe (binary path)

**Variations:** `ausearch -ui 0` for events by root. `ausearch -c vim` for events by command name. `aureport -f` for summary.

**Edge case:** The `proctitle` field is hex-encoded. Decode with `echo '76696D20...' | xxd -r -p`. The comm field is truncated to 15 characters on older kernels.

### Example 8: SSH login monitor

```bash
$ tail -Fn0 /var/log/auth.log | while read line; do
    if echo "$line" | grep -qE 'Accepted (password|publickey)'; then
      user=$(echo "$line" | awk '{print $9}')
      ip=$(echo "$line" | awk '{print $11}')
      logger -t ssh-monitor "LOGIN: user=$user from=$ip"
    fi
  done
```

**Anatomy:** `tail -Fn0` follows the file, reading new lines as they're appended. The `-F` flag handles log rotation (follows the file by inode, not name). The `while read` loop processes each line.

**Variations:** Use `journalctl -u sshd -f` on systemd systems. Check for failed attempts too: `Failed password for`. Alert on root login attempts.

**Edge case:** On busy systems, auth.log may rotate mid-read. `tail -F` handles this by tracking inode, but if rotation happens and many lines are written quickly, you might miss some.

### Example 9: Defense toolkit — modular design

```bash
$ cat > defense_kit.sh << 'EOF'
#!/bin/bash
COMMAND="${1:-help}"

case "$COMMAND" in
  fswatch)
    inotifywait -m -r "$2" -e modify,create,delete,move --format '%w%f %e %T' --timefmt '%H:%M:%S'
    ;;
  hashcheck)
    [ -f "$2" ] && sha256sum -c "$2" 2>&1 | grep -v ': OK$' || echo "Usage: $0 hashcheck <baseline_file>"
    ;;
  procanomaly)
    echo "=== Processes in /tmp ==="
    ps -eo pid,user,args | grep -E '/tmp/|/dev/shm/|/var/tmp/'
    echo "=== Deleted executables ==="
    for pid in /proc/[0-9]*; do
      readlink -f "$pid/exe" &>/dev/null || echo "  PID $(basename $pid) has no executable"
    done
    ;;
  logwatch)
    local logfile="${2:-/var/log/syslog}"
    tail -Fn0 "$logfile" | grep --line-buffered -E 'FAILED|ERROR|Unauthorized|Accepted'
    ;;
  *)
    echo "Usage: $0 {fswatch|hashcheck|procanomaly|logwatch} [args]"
    exit 1
    ;;
esac
EOF
```

**Anatomy:** Modular dispatch with `case`. Each command is a self-contained function. Arguments pass through from the top-level parser.

**Variations:** Add JSON output mode. Add logging mode that writes to a file. Add notification mode (desktop alert, email, webhook).

### Example 10: inotify-based reverse shell detector

```bash
$ inotifywait -m /proc -e create 2>/dev/null | while read dir ev file; do
    [[ "$file" =~ ^[0-9]+$ ]] || continue
    cmdline=$(tr '\0' ' ' < /proc/$file/cmdline 2>/dev/null) || continue
    if echo "$cmdline" | grep -qE '/dev/tcp|bash -i|nc -e|ncat'; then
      echo "SUSPICIOUS: PID $file — $cmdline"
      logger -t revshell-detect "Reverse shell detected: PID=$file CMD=$cmdline"
    fi
  done
```

**Anatomy:** Monitors `/proc` for new directories (which are PID directories). When a new PID appears, reads its cmdline and checks for reverse shell patterns.

**Variations:** Monitor for `execve` syscalls via auditd. Check for processes with network connections but no listening ports (outbound-only).

**Edge case:** This is CPU-intensive on busy systems (every process creation triggers a check). For production, use auditd or eBPF instead.

### Example 11: Integrity monitoring with new-file detection

```bash
$ cat > advanced_integrity.sh << 'EOF'
BASEDIR="$HOME/.integrity_db"
WATCH_DIR="${1:-/etc}"
BASELINE="$BASEDIR/$(echo "$WATCH_DIR" | tr '/' '_').sha256"

check_integrity() {
  # Check modified files
  local modified=$(sha256sum -c "$BASELINE" 2>&1 | grep ': FAILED$')
  echo "=== MODIFIED FILES ==="
  echo "$modified" | sed 's/: FAILED//' | while read file; do
    echo "  MODIFIED: $file"
  done

  # Find new files
  echo "=== NEW FILES ==="
  diff <(awk '{print $2}' "$BASELINE" | sort) \
       <(find "$WATCH_DIR" -type f | sort) | grep '^>' | sed 's/^>/  NEW:/'

  # Find deleted files
  echo "=== DELETED FILES ==="
  diff <(awk '{print $2}' "$BASELINE" | sort) \
       <(find "$WATCH_DIR" -type f | sort) | grep '^<' | sed 's/^</  DELETED:/'
}
EOF
```

**Anatomy:** Three-way diff: modified (hash mismatch), new (in filesystem but not in baseline), deleted (in baseline but not in filesystem).

**Edge case:** Files that are legitimately modified (config updates, package installations) trigger alerts. Exclude known changeable files or update the baseline after authorized changes.

### Example 12: Process ancestry analysis

```bash
$ cat > process_tree.sh << 'EOF'
show_tree() {
  local pid=${1:-1}
  local indent=${2:-0}
  local padding=$(printf '%*s' "$indent" '')
  local cmdline=$(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null || echo "[kernel thread]")
  echo "${padding}├─ PID $pid: $cmdline"
  for child in /proc/$pid/task/$pid/children 2>/dev/null; do
    [ -f "$child" ] && for cpid in $(cat "$child" 2>/dev/null); do
      show_tree "$cpid" $((indent + 2))
    done
  done
}
show_tree
EOF
```

**Anatomy:** Recursively reads `/proc/PID/task/PID/children` to build a process tree. Useful for spotting unexpected parent-child relationships (e.g., a reverse shell whose parent is `nginx`).

**Variations:** Check for processes whose PPID no longer exists (zombie/ghost processes). Check for multiple processes with the same PID namespace.

**Edge case:** Reading `/proc/PID/task/PID/children` requires `CAP_SYS_PTRACE` or root. Without it, children lists are empty.

### Example 13: Desktop security alert (notify-send)

```bash
$ cat > security_alert.sh << 'EOF'
send_alert() {
  local urgency="${1:-normal}" title="$2" message="$3"
  if command -v notify-send &>/dev/null; then
    notify-send -u "$urgency" "$title" "$message"
  fi
  logger -t security-alert "$title: $message"
  if [ "$urgency" = "critical" ]; then
    # For critical alerts, also write to a dedicated file
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $title: $message" >> /var/log/security/critical.log
  fi
}

# Example usage:
send_alert "critical" "ROOT LOGIN DETECTED" "root login from 10.0.0.5 at $(date)"
send_alert "normal" "File integrity check complete" "No changes detected"
EOF
```

**Anatomy:** Wraps `notify-send` and `logger` into a single alert function. `-u critical` makes the notification persistent on the desktop.

**Variations:** Add email integration with `mail`. Add webhook with `curl`. Add SMS with `twilio-cli`.

### Example 14: Log rotation watcher

```bash
$ cat > log_rotation_detector.sh << 'EOF'
WATCH_DIR="${1:-/var/log}"
DETECTED_LOG=$(mktemp)

inotifywait -m "$WATCH_DIR" -e delete -e move --format '%w%f %e' |
while read path event; do
  if [[ "$path" == *.log ]] && [[ "$path" != *.gz ]]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') — $path was $event" >> "$DETECTED_LOG"
    logger -t log-watcher "Log file $event: $path"
  fi
done
EOF
```

**Anatomy:** Watches for log file deletion or moves. Attackers often delete log files to cover tracks. This detects the deletion immediately.

**Variations:** Combine with `logrotate` integration — if a log rotates normally, it's expected. If it disappears unexpectedly, it's suspicious.

**Edge case:** Log rotation via `logrotate` also triggers delete/move events. Your detection script needs to distinguish between expected rotation and malicious deletion. Check `/etc/logrotate.d/` for expected patterns.

### Example 15: Comprehensive defense dashboard

```bash
$ cat > dashboard.sh << 'EOF'
while true; do
  clear
  echo "=== SYSTEM DEFENSE DASHBOARD ==="
  echo "Time: $(date)"
  echo ""
  echo "--- File Integrity ---"
  sha256sum -c "$HOME/.integrity_db/etc_baseline.sha256" 2>&1 | grep -v ': OK$' | head -5
  echo ""
  echo "--- Suspicious Processes ---"
  ps -eo pid,user,args --sort=-%mem | grep -E '/tmp|/dev/shm' | head -5
  echo ""
  echo "--- Listening Services ---"
  ss -tlnp4 | tail -n+2 | head -10
  echo ""
  echo "--- Recent SSH Logins ---"
  tail -5 /var/log/auth.log 2>/dev/null | grep -E 'Accepted|Failed'
  sleep 10
done
EOF
```

**Anatomy:** Continuous monitoring dashboard that refreshes every 10 seconds. Shows integrity status, suspicious processes, listening services, and SSH login events.

**Variations:** Add color coding (red for critical, yellow for warnings). Add network traffic spikes. Add disk usage warnings.

**Edge case:** The dashboard itself uses CPU. On production systems, keep the refresh interval high (30-60 seconds) or disable it.

## Real-World Use Cases

### FOR the OS

- File integrity monitoring (Tripwire, AIDE, OSSEC)
- System auditing (auditd for compliance: PCI-DSS, HIPAA, SOC2)
- Log monitoring (fail2ban, logwatch, SEC)
- Process monitoring (monit, supervisor)

### WITH the OS

- inotify triggers for config reload (haproxy reload on config change)
- auditd for intrusion detection (watch /etc/passwd modifications)
- sha256sum for package verification (dpkg --verify)
- tail -F for real-time log aggregation (logstash, fluentd)

### AGAINST the OS

- Attackers disable auditd: `systemctl stop auditd`
- Attackers clear history: `history -c && history -w; unset HISTFILE`
- Attackers stop inotify watchers: kill the monitoring process, or exhaust watch limits
- Attackers modify baselines: if they have root, they can update the SHA256 files
- Attackers use `LD_PRELOAD` to hook `open()` and hide from integrity checks

### FOR DEFENSE

- **Immutable baselines:** Store integrity hashes on read-only media or a remote syslog server
- **Protected processes:** Run monitors as a dedicated user with limited permissions
- **Log shipping:** Ship ALL logs to a remote syslog server immediately (before the attacker can delete them)
- **Watch the watchers:** Monitor your monitoring processes with a secondary mechanism
- **Kernel-level monitoring:** Use eBPF for detection that can't be bypassed by user-space rootkits
- **Hardened auditd:** Set `space_left_action = SUSPEND` and `action_mail_acct = root` in auditd.conf

## Memory Aids

**"HASH IT, WATCH IT, LOG IT"** — Three-layer defense:
- **HASH:** File integrity baselines detect changes after the fact
- **WATCH:** inotify detects changes in real-time
- **LOG:** auditd provides forensic evidence for investigation

**"PROCESS TRIAD: PID-PPID-EXE"** — The three things to check about any suspicious process:
- What is its PID? (When did it start? Is it new?)
- What is its PPID? (Who spawned it? Is that expected?)
- What is its EXE? (Is the binary on disk? Is it from a known location?)

**"5 SIGNS OF COMPROMISE"** — Things every detection system should check:
1. Processes running from /tmp or /dev/shm
2. Unexpected outbound network connections
3. Modified system binaries or configurations
4. New cron jobs or systemd services
5. Deleted or truncated log files

## Trap Vault (15 traps)

**Trap 1:** `sha256sum -c` only checks files IN the baseline. New malicious files are NOT detected unless you also run a new-file scan. Always check both.

**Trap 2:** inotify watches are per-inode, not per-path. If a monitored file is deleted and recreated (common with editor save operations), the watch on the old inode is lost and the new file is NOT monitored.

**Trap 3:** `auditctl` rules are lost on reboot unless persisted in `/etc/audit/rules.d/`. The command `auditctl -l` shows current rules — if none are shown after boot, rules weren't persisted.

**Trap 4:** An attacker with root can stop auditd (`systemctl stop auditd`) or modify rules (`auditctl -D`). Monitor auditd as a service, and consider a secondary remote logging mechanism.

**Trap 5:** `/proc/PID/cmdline` is truncated at 4096 bytes and uses null separators. `tr '\0' ' '` converts it to readable form but may lose argument boundaries. Use `strings /proc/PID/cmdline` for a safer read.

**Trap 6:** Processes can rename themselves via `prctl(PR_SET_NAME)` or by modifying `argv[0]`. A malicious process can appear as `[kworker/0:0]` in `ps` output. Check `/proc/PID/exe` for the real binary.

**Trap 7:** `tail -F` tracks by inode and handles rotation. But if the log file is DELETED (not rotated), tail exits. Add `--retry` to keep trying if the file is recreated.

**Trap 8:** `inotifywait -m -r /` will exhaust all available watches immediately. The inotify watch limit (`max_user_watches`) is 8192 by default — not enough for a full filesystem.

**Trap 9:** Attackers can use `LD_PRELOAD` to hook `open()` and return fake contents to integrity checkers. The baseline hash file is read via `open()` — the rootkit can serve the old hash while the real file is different.

**Trap 10:** `ps -eo pid,args` shows the command line as stored in `/proc/PID/cmdline` at process creation. A process that execs a new binary keeps the original cmdline. Use `/proc/PID/exe` for the actual binary.

**Trap 11:** Shell history (`~/.bash_history`) is only written on shell EXIT (or with `history -w`). If the shell is killed with SIGKILL, history is lost. Attackers can use `kill -9 $$` to prevent history writing.

**Trap 12:** `set +o history` disables history for the current shell session, but the shell still writes previous sessions' history on exit. Attackers may combine `set +o history` with `kill -9` to leave no trace.

**Trap 13:** `ausearch -k some_key` searches the audit log for events with that key. If the attacker deleted the audit log file (or rotated it away), `ausearch` finds nothing. Check log file integrity too.

**Trap 14:** inotify on NFS/CIFS mount points behaves differently. Some events may not be generated at all. For network filesystems, use polling (`find -newer`) instead of inotify.

**Trap 15:** The `lsof -i` command shows network connections by reading `/proc/PID/fd/` and `/proc/net/`. A rootkit that hooks these /proc entries can hide connections. Use `ss` (which uses netlink) as a more reliable alternative.

## See It In The Wild

- **Log tampering in the Colonial Pipeline attack:** Attackers deleted logs from the pipeline's IT systems after deploying ransomware. Log shipping to a remote syslog server would have preserved forensic evidence.

- **Samsung data breach (2022):** Attackers modified file integrity monitoring baselines on compromised systems to hide their changes from weekly integrity scans. Immutable baseline storage would have prevented this.

- **SolarWinds Orion attack:** The attackers added legitimate-looking code to the Orion monitoring tool itself — the defenders' monitoring tool was the attack vector. Defense-in-depth means monitoring your monitors.

- **Crypto miners evading ps detection:** Miners renamed themselves to `[kworker/0:0]` using `prctl(PR_SET_NAME)` to appear as kernel threads. Detected by checking `/proc/PID/exe` (which resolves to something in /tmp, not a kernel path).

- **SSH brute force detection with fail2ban:** Uses `tail -F /var/log/auth.log` to detect repeated failed logins. The same technique can be adapted for any log-based detection.

- **Rootkit hiding from sha256sum:** The Jynx2 LD_PRELOAD rootkit hooked `open()` and `read()` to return clean file contents when integrity checkers accessed modified binaries. Detected by using a statically-linked binary for verification.

## Check Your Understanding (10 questions)

1. **Q:** What's the difference between `sha256sum -c` and a simple `find -exec sha256sum` for integrity checking? **A:** `sha256sum -c` compares against a stored baseline. `find -exec sha256sum` just computes current hashes — you need a separate diff step to detect changes.

2. **Q:** How does inotify differ from auditd for file change detection? **A:** inotify provides real-time event notification from the kernel via file descriptors. auditd logs events to disk for forensic analysis. inotify is better for real-time reaction; auditd is better for post-incident investigation.

3. **Q:** Why would a process have no executable in `/proc/PID/exe`? **A:** The original binary was deleted from disk after the process started. The process is still running with the inode open, but the path is gone. This is a strong indicator of malicious activity.

4. **Q:** How can an attacker evade `sha256sum` integrity checking if they have root access? **A:** They can (1) modify the baseline hash file to match the new file, (2) use LD_PRELOAD to hook `open()` and return fake file contents, or (3) modify files only on a filesystem that isn't checksummed.

5. **Q:** Why should detection logs be shipped to a remote syslog server? **A:** If an attacker gains root on a system, they can tamper with all local logs. Shipping logs to a remote server (ideally append-only) preserves forensic evidence even if the local system is fully compromised.

6. **Q:** What is the significance of `inotify_add_watch()` returning `ENOSPC`? **A:** It means the per-user watch limit (`max_user_watches`) has been exceeded. Increase it via `sysctl fs.inotify.max_user_watches`. Attackers may intentionally exhaust watch limits to blind monitoring.

7. **Q:** How does `tail -F` handle log rotation vs. `tail -f`? **A:** `-f` follows the file descriptor — if the file is rotated (renamed and a new file created), tail still follows the old (now renamed) file. `-F` follows by inode or filename — it detects when the watched file is replaced and opens the new file.

8. **Q:** What's the limitation of using `ps` for process monitoring? **A:** `ps` takes a snapshot of `/proc` at a point in time. Short-lived processes (spawning, executing, exiting in milliseconds) can easily be missed between `ps` runs. Use auditd or eBPF for comprehensive process monitoring.

9. **Q:** How can you detect that an attacker has disabled history recording? **A:** Check if `HISTFILE` is set and writable. Check `set -o | grep history`. Compare the number of lines in `~/.bash_history` with expected session activity. A sudden drop in history line count suggests clearing.

10. **Q:** Why is monitoring `/proc/PID/children` useful for defense? **A:** It reveals process ancestry. An unexpected parent-child relationship (e.g., a network daemon spawning a shell) is a strong indicator of compromise. A web server should not spawn `/bin/bash` with network connections.
