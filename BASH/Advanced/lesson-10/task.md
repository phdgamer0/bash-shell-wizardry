# Task 10: Privilege Escalation Audit Script

## Objective
Write a comprehensive security audit script that checks all common privilege escalation vectors on a Linux system. The script should produce a color-coded report with severity ratings, actionable recommendations, and explain the risk of each finding. This is a DEFENSE exercise — the goal is to identify and close attack paths before attackers find them.

## Requirements

1. **SUID/SGID scan:** Find all SUID/SGID files, classify them by risk (known-safe system binaries vs. custom/unowned binaries)
2. **Sudo audit:** Parse `sudo -l` output for dangerous commands (shell escapes, interpreters, package managers)
3. **Capabilities:** Use `getcap` to find files with dangerous capabilities
4. **Writable PATH:** Check for writable directories in PATH
5. **World-writable root files:** Find files owned by root that anyone can write
6. **Cron analysis:** Check for writable cron scripts and directories
7. **Kernel security:** Check ASLR, yama ptrace_scope, and other kernel protections
8. **Reporting:** Severity ratings (HIGH/MEDIUM/LOW), summary at end, remediation suggestions
9. **Logging:** Write results to a timestamped log file for historical comparison
10. **Delta mode:** Compare current scan with previous scan to detect new/changed findings

## Sub-tasks (9 cumulative)

### Task 10.1: SUID/SGID Scanner

```bash
#!/bin/bash
# privesc_audit.sh — SUID component
RED='\033[0;31m'; GREEN='\033[0;32m'; YELLOW='\033[1;33m'; NC='\033[0m'
HIGH=0; MEDIUM=0; LOW=0

scan_suid() {
  echo "=== [SUID/SGID] Scanning ==="
  local suid_files=()
  while IFS= read -r -d '' file; do
    suid_files+=("$file")
  done < <(find / -type f \( -perm -4000 -o -perm -2000 \) -print0 2>/dev/null)
  
  echo "Found ${#suid_files[@]} SUID/SGID binaries"
  
  for file in "${suid_files[@]}"; do
    local severity="LOW"
    local owner=$(stat -c '%U' "$file" 2>/dev/null)
    local type=""
    [ -u "$file" ] && type+="SUID "
    [ -g "$file" ] && type+="SGID "
    
    # Check if owned by a package (Debian/Ubuntu)
    local pkg=""
    if command -v dpkg &>/dev/null; then
      pkg=$(dpkg -S "$file" 2>/dev/null || echo "UNOWNED")
    elif command -v rpm &>/dev/null; then
      pkg=$(rpm -qf "$file" 2>/dev/null || echo "UNOWNED")
    else
      pkg="UNKNOWN (no package manager)"
    fi
    
    # Classify severity
    case "$file" in
      /usr/bin/passwd|/usr/bin/su|/usr/bin/sudo|/usr/bin/newgrp|/usr/bin/chsh|/usr/bin/chfn|/usr/bin/mount|/usr/bin/umount|/usr/bin/pkexec)
        severity="LOW"  # Known safe system binaries
        ;;
      /opt/*|/tmp/*|/home/*|/var/tmp/*|/dev/*)
        severity="HIGH"  # Suspicious locations
        ;;
      /usr/local/*)
        severity="MEDIUM"  # Less common, possibly custom
        ;;
      *)
        if [[ "$pkg" == "UNOWNED" ]]; then
          severity="HIGH"  # Not from any package — suspicious
        elif [[ "$owner" != "root" ]]; then
          severity="HIGH"  # Not owned by root — unusual
        else
          severity="MEDIUM"
        fi
        ;;
    esac
    
    # Special: SUID on scripts is ignored, note this
    local is_script=0
    file "$file" 2>/dev/null | grep -qE '(script|text executable)' && is_script=1
    
    case $severity in
      HIGH)   echo -e "  ${RED}HIGH${NC}: $file ($type|$pkg)$([ $is_script -eq 1 ] && echo ' [SCRIPT — SUID IGNORED BY KERNEL]')"; ((HIGH++)) ;;
      MEDIUM) echo -e "  ${YELLOW}MEDIUM${NC}: $file ($type|$pkg)"; ((MEDIUM++)) ;;
      LOW)    echo -e "  ${GREEN}LOW${NC}: $file ($type|$pkg)"; ((LOW++)) ;;
    esac
  done
}
```

### Task 10.2: Sudo Audit

```bash
audit_sudo() {
  echo
  echo "=== [SUDO] Auditing sudo privileges ==="
  
  if ! command -v sudo &>/dev/null; then
    echo "  sudo not installed"
    return
  fi
  
  local sudo_output
  sudo_output=$(sudo -l 2>/dev/null)
  if [ $? -ne 0 ]; then
    echo "  Could not run sudo -l (may need password)"
    return
  fi
  
  # Check for dangerous commands
  local dangerous_commands=(
    "less" "more" "vim" "vi" "nano" "man"
    "python" "python3" "perl" "ruby" "lua"
    "awk" "sed" "find" "tcpdump" "strace"
    "bash" "sh" "zsh" "fish"
    "apt" "apt-get" "dpkg" "rpm" "yum" "dnf"
    "tmux" "screen" "script"
    "env" "setenv" "reset"
  )
  
  echo "$sudo_output" | while read -r line; do
    echo "  $line"
    for cmd in "${dangerous_commands[@]}"; do
      if echo "$line" | grep -qi "$cmd"; then
        echo -e "    ${RED}HIGH:${NC} '$cmd' can be used to escape to a shell"
        echo "    Technique: $cmd can execute shell commands via ! or :! or system()"
      fi
    done
    
    if echo "$line" | grep -qi "NOPASSWD"; then
      echo -e "    ${YELLOW}MEDIUM:${NC} NOPASSWD — no password required"
    fi
  done
  
  # Check env_keep for dangerous variables
  if echo "$sudo_output" | grep -q "env_keep"; then
    echo
    echo "  Checking environment preservation..."
    echo "$sudo_output" | grep "env_keep" | while read -r env_line; do
      if echo "$env_line" | grep -qE '(LD_PRELOAD|LD_LIBRARY_PATH|PYTHONPATH|PERL5LIB)'; then
        echo -e "    ${RED}HIGH:${NC} Dangerous env_keep: $env_line"
        echo "    LD_PRELOAD preservation allows arbitrary code execution via sudo"
      fi
    done
  fi
}
```

### Task 10.3: Capabilities Scanner

```bash
scan_capabilities() {
  echo
  echo "=== [CAPABILITIES] Scanning ==="
  
  if ! command -v getcap &>/dev/null; then
    echo "  getcap not installed (install libcap-ng-utils)"
    return
  fi
  
  local dangerous_caps=(
    "cap_dac_override"
    "cap_dac_read_search"
    "cap_setuid"
    "cap_setgid"
    "cap_sys_admin"
    "cap_sys_ptrace"
    "cap_sys_module"
    "cap_sys_rawio"
    "cap_sys_chroot"
    "cap_sys_boot"
    "cap_linux_immutable"
    "cap_net_admin"
  )
  
  local caps_found
  caps_found=$(getcap -r / 2>/dev/null)
  
  if [ -z "$caps_found" ]; then
    echo "  No capabilities found (or filesystem doesn't support xattrs)"
    return
  fi
  
  echo "$caps_found" | while read -r line; do
    local file="${line% *}"
    local caps="${line#* }"
    local severity="LOW"
    
    for dc in "${dangerous_caps[@]}"; do
      if echo "$caps" | grep -qi "$dc"; then
        severity="HIGH"
        echo -e "  ${RED}HIGH${NC}: $file"
        echo "    Capability: $caps"
        case "$dc" in
          cap_dac_override) echo "    Risk: Can write any file (bypasses permission checks)" ;;
          cap_dac_read_search) echo "    Risk: Can read any file (including /etc/shadow)" ;;
          cap_setuid) echo "    Risk: Can change UID to any user (become root)" ;;
          cap_sys_admin) echo "    Risk: Many privileged operations" ;;
          cap_sys_ptrace) echo "    Risk: Can debug/inject any process" ;;
        esac
        break
      fi
    done
    
    if [ "$severity" == "LOW" ]; then
      echo -e "  ${GREEN}LOW${NC}: $file"
    fi
  done
}
```

### Task 10.4: Writable PATH Detector

```bash
scan_path() {
  echo
  echo "=== [PATH] Writable directories in PATH ==="
  
  local IFS=':'
  local found=0
  for dir in $PATH; do
    if [ -w "$dir" ]; then
      echo -e "  ${RED}Writable${NC}: $dir"
      echo "    Risk: Attacker can place malicious executable that gets priority"
      echo "    Check: Is $dir before /usr/bin in PATH? $(echo "$PATH" | tr ':' '\n' | grep -n "$dir" | head -1)"
      ((found++))
    fi
  done
  [ "$found" -eq 0 ] && echo -e "  ${GREEN}PASS${NC}: No writable directories in PATH"
}
```

### Task 10.5: World-Writable Root Files Scanner

```bash
scan_world_writable() {
  echo
  echo "=== [WORLD-WRITABLE] Files owned by root, writable by anyone ==="
  
  local found=0
  while IFS= read -r -d '' file; do
    # Skip /proc, /sys, /dev (pseudo-filesystems)
    case "$file" in /proc/*|/sys/*|/dev/*) continue ;; esac
    
    # Check if it's a regular file or script
    if [ -f "$file" ]; then
      local perms=$(stat -c '%a' "$file" 2>/dev/null)
      local type=$(file "$file" 2>/dev/null)
      echo -e "  ${RED}HIGH${NC}: $file (perms: $perms)"
      echo "    Type: $type"
      if echo "$type" | grep -qiE '(script|shell|bash|python|perl|ruby)'; then
        echo "    EXTRA RISK: This is a script — modifiable code executed by root"
      fi
      ((found++))
    fi
  done < <(find / -xdev -user root -perm -0002 -type f -print0 2>/dev/null)
  
  [ "$found" -eq 0 ] && echo -e "  ${GREEN}PASS${NC}: No world-writable root-owned files"
  echo "  Found $found files"
}
```

### Task 10.6: Cron Job Auditor

```bash
scan_cron() {
  echo
  echo "=== [CRON] Auditing cron jobs ==="
  
  local cron_dirs=("/etc/cron.d" "/etc/cron.daily" "/etc/cron.hourly" "/etc/cron.weekly" "/etc/cron.monthly")
  
  for dir in "${cron_dirs[@]}"; do
    if [ ! -d "$dir" ]; then
      continue
    fi
    
    local perms=$(stat -c '%a' "$dir" 2>/dev/null)
    if [ -w "$dir" ]; then
      echo -e "  ${RED}HIGH${NC}: Cron directory is writable: $dir ($perms)"
    fi
    
    for file in "$dir"/*; do
      [ -f "$file" ] || continue
      if [ -w "$file" ]; then
        echo -e "  ${RED}HIGH${NC}: Cron script is writable: $file"
        echo "    Owner: $(stat -c '%U' "$file")"
      fi
    done
  done
  
  # Check /etc/crontab
  if [ -f /etc/crontab ] && [ -w /etc/crontab ]; then
    echo -e "  ${RED}CRITICAL${NC}: /etc/crontab is writable!"
  fi
}
```

### Task 10.7: Kernel Security Check

```bash
scan_kernel() {
  echo
  echo "=== [KERNEL] Security mitigation checks ==="
  
  # ASLR
  local aslr=$(cat /proc/sys/kernel/randomize_va_space 2>/dev/null)
  case $aslr in
    0) echo -e "  ${RED}HIGH${NC}: ASLR is DISABLED (0)" ;;
    1) echo -e "  ${YELLOW}MEDIUM${NC}: ASLR is partial (1)" ;;
    2) echo -e "  ${GREEN}LOW${NC}: ASLR is full (2)" ;;
    *) echo -e "  ${YELLOW}UNKNOWN${NC}: ASLR value: $aslr" ;;
  esac
  
  # ptrace_scope
  local ptrace=$(cat /proc/sys/kernel/yama/ptrace_scope 2>/dev/null)
  case $ptrace in
    0) echo -e "  ${RED}HIGH${NC}: ptrace unrestricted (0) — any process can debug any other" ;;
    1) echo -e "  ${GREEN}LOW${NC}: ptrace restricted (1)" ;;
    2) echo -e "  ${GREEN}LOW${NC}: ptrace admin-only (2)" ;;
    3) echo -e "  ${GREEN}LOW${NC}: ptrace disabled (3)" ;;
    *) echo -e "  ${YELLOW}MEDIUM${NC}: ptrace_scope not supported (Yama not loaded)" ;;
  esac
  
  # kptr_restrict
  local kptr=$(cat /proc/sys/kernel/kptr_restrict 2>/dev/null)
  case $kptr in
    0) echo -e "  ${YELLOW}MEDIUM${NC}: kptr_restrict = 0 (kernel pointers visible to all)" ;;
    1) echo -e "  ${GREEN}LOW${NC}: kptr_restrict = 1 (restricted)" ;;
    2) echo -e "  ${GREEN}LOW${NC}: kptr_restrict = 2 (hidden from all non-root)" ;;
  esac
  
  # dmesg_restrict
  local dmesg=$(cat /proc/sys/kernel/dmesg_restrict 2>/dev/null)
  case $dmesg in
    0) echo -e "  ${YELLOW}MEDIUM${NC}: dmesg_restrict = 0 (anyone can read kernel log)" ;;
    1) echo -e "  ${GREEN}LOW${NC}: dmesg_restrict = 1 (kernel log restricted)" ;;
  esac
  
  # Check kernel version for known exploit mitigations
  local version=$(uname -r)
  echo "  Kernel: $version"
}
```

### Task 10.8: Logging and Historical Comparison

```bash
log_findings() {
  local log_dir="/var/log/privesc_audit"
  mkdir -p "$log_dir" 2>/dev/null
  
  local timestamp=$(date '+%Y%m%d_%H%M%S')
  local log_file="$log_dir/audit_$timestamp.log"
  local prev_file=$(ls -t "$log_dir"/audit_*.log 2>/dev/null | head -1)
  
  # Write current findings
  {
    echo "=== Privilege Escalation Audit ==="
    echo "Date: $(date)"
    echo "Host: $(hostname)"
    echo "User: $(whoami)"
    echo
    echo "HIGH:   $HIGH"
    echo "MEDIUM: $MEDIUM"
    echo "LOW:    $LOW"
    echo
    echo "Kernel: $(uname -a)"
    echo "Packages: $(dpkg -l 2>/dev/null | wc -l) packages"
  } > "$log_file"
  
  # Compare with previous if exists
  if [ -n "$prev_file" ] && [ -f "$prev_file" ]; then
    echo
    echo "=== Changes from previous audit ==="
    diff --changed-group-format='%>' --unchanged-group-format='' \
      "$prev_file" "$log_file" 2>/dev/null || echo "  No changes detected"
  fi
  
  echo "Log saved to: $log_file"
}
```

### Task 10.9: Integration and Reporting

```bash
#!/bin/bash
# privesc_audit.sh — Complete integration

main() {
  echo "============================================"
  echo "  PRIVILEGE ESCALATION AUDIT"
  echo "  $(date)"
  echo "============================================"
  echo
  
  scan_suid
  audit_sudo
  scan_capabilities
  scan_path
  scan_world_writable
  scan_cron
  scan_kernel
  
  echo
  echo "============================================"
  echo "  SUMMARY"
  echo "============================================"
  echo -e "  ${RED}HIGH:   $HIGH${NC}"
  echo -e "  ${YELLOW}MEDIUM: $MEDIUM${NC}"
  echo -e "  ${GREEN}LOW:    $LOW${NC}"
  echo "  Total:  $((HIGH + MEDIUM + LOW))"
  echo
  
  log_findings
  
  echo
  echo "============================================"
  echo "  RECOMMENDATIONS"
  echo "============================================"
  [ "$HIGH" -gt 0 ] && echo "  - Investigate all HIGH findings immediately"
  [ "$MEDIUM" -gt 0 ] && echo "  - Schedule MEDIUM findings for next maintenance window"
  echo "  - Run this audit regularly (e.g., weekly cron)"
  echo "  - Keep kernel and packages up to date"
  echo "  - Remove unnecessary SUID binaries and capabilities"
}

main "$@"
```

## Bonus Challenges

1. **Bonus A:** Implement a `--fix` mode that automatically applies known-safe fixes (e.g., removing SUID from binaries not in the package manager database).

2. **Bonus B:** Add container-aware detection — detect if running in Docker and check container-specific privesc vectors (capabilities, mounts, seccomp).

3. **Bonus C:** Create a visualization of audit results using ASCII bar charts or HTML output.

4. **Bonus D:** Integrate with CVE databases — check kernel version and package versions against known CVEs.

5. **Bonus E:** Build a continuous monitoring mode that watches for new SUID binaries, new capabilities, or changed sudoers using `inotify` on key files.

## Hints

<details>
<summary>Hint 1: Package manager integration</summary>

Use `dpkg -S` (Debian) or `rpm -qf` (Red Hat) to check if a file belongs to a package. Unowned files are more suspicious.
</details>

<details>
<summary>Hint 2: Dangerous sudo commands</summary>

Any command that can execute a subshell is dangerous: `less`, `vim`, `more`, `man`, `awk`, `find`, `python`, `perl`, `bash`.
</details>

<details>
<summary>Hint 3: Capability levels</summary>

`cap_dac_override` and `cap_dac_read_search` are near-root for file access. `cap_setuid` is near-root for identity.
</details>

<details>
<summary>Hint 4: History log location</summary>

Store audit logs in `/var/log/privesc_audit/` with timestamps. Use symlinks for "latest" pointer.
</details>

<details>
<summary>Hint 5: Running as non-root</summary>

Some checks (like reading `/proc/PID/` of other processes) require root. Gracefully skip checks that need elevated privileges when running as a regular user.
</details>

## Expected Output

```bash
$ sudo ./privesc_audit.sh
============================================
  PRIVILEGE ESCALATION AUDIT
  Mon Jul 31 10:00:00 UTC 2026
============================================

=== [SUID/SGID] Scanning ===
Found 14 SUID/SGID binaries
  LOW: /usr/bin/su (SUID | passwd)
  LOW: /usr/bin/sudo (SUID | sudo)
  LOW: /usr/bin/passwd (SUID | passwd)
  HIGH: /opt/custom/helper (SUID | UNOWNED)
  MEDIUM: /usr/local/bin/legacy-tool (SUID | UNOWNED)

=== [SUDO] Auditing sudo privileges ===
User user may run:
  (root) NOPASSWD: /usr/bin/less
  HIGH: 'less' can be used to escape to a shell
  MEDIUM: NOPASSWD — no password required
  (root) /usr/sbin/service nginx restart

=== [CAPABILITIES] Scanning ===
  HIGH: /usr/bin/tar = cap_dac_read_search+ep
    Risk: Can read any file (including /etc/shadow)
  LOW: /usr/bin/ping = cap_net_raw+ep

=== [PATH] Writable directories in PATH ===
  Writable: /home/user/bin
    Risk: Attacker can place malicious executable

=== [WORLD-WRITABLE] Root-owned writable files ===
  HIGH: /opt/scripts/backup.sh (perms: 777)
    Type: Bourne-Again shell script

=== [CRON] Auditing cron jobs ===
  HIGH: /etc/cron.d/backup_job is writable

=== [KERNEL] Security mitigation checks ===
  LOW: ASLR is full (2)
  LOW: ptrace restricted (1)
  LOW: kptr_restrict = 1
  LOW: dmesg_restrict = 1

============================================
  SUMMARY
============================================
  HIGH:   4
  MEDIUM: 2
  LOW:    16

============================================
  RECOMMENDATIONS
============================================
  - Investigate all HIGH findings immediately
  - Schedule MEDIUM findings for next maintenance window
```

## Deep Self-Check

1. **SUID script test:** Create a simple shell script, set SUID on it (`chmod u+s`), and observe whether it runs with elevated privileges. What does `ps -eo pid,uid,euid,comm` show? Why does the kernel ignore SUID on scripts?

2. **Capability experiment:** Use `setcap cap_dac_read_search+ep /usr/bin/cat` (if permitted) and try to read `/etc/shadow` as a normal user with `/usr/bin/cat`. Does it work? Remove the capability after testing.

3. **PATH hijacking simulation:** Create a directory `~/evil`, prepend it to PATH, create a function/script called `ls` there that prints "Hijacked!". Run `ls`. What happens? This is exactly how PATH hijacking works.

4. **sudo -l analysis:** Run `sudo -l` on your system. Are any of the allowed commands in the dangerous list? Can you escape to a shell from any of them?

5. **World-writable file hunt:** Find all world-writable files on your system. How many are there? Classify them into "needs fixing" vs. "intentionally writable" (like `/tmp`).

6. **Kernel exploit mitigation score:** Based on your kernel version and security settings, how difficult would it be for a kernel exploit to work? Research your kernel version for known CVEs.

7. **Container privesc test:** If you have Docker, run a container with `--privileged` and check what capabilities you have inside with `capsh --print`. How does this differ from a non-privileged container?

8. **Cron exploitation simulation:** Create a world-writable cron script (in a test environment!) that runs `touch /tmp/pwned`. Wait for it to execute. Does it run as root? What UID creates the file?

9. **NFS no_root_squash test:** If you have NFS, check `/etc/exports` for `no_root_squash`. If found, research what it allows. Test by mounting the share as root and creating a SUID binary.

10. **Audit delta analysis:** Run the audit twice, making a change between runs (e.g., add a test SUID binary). Does the delta comparison correctly identify the change?
