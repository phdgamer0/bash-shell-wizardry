# Lesson 20: Capstone — Security Scanner

## History & Origins

Security auditing in Unix dates to the 1980s with tools like `COPS` (Computer Oracle and Password System, 1989) by Dan Farmer and Gene Spafford. COPS checked for common security issues: world-writable files, weak passwords, SUID binaries, and permissions problems. It was written in shell and Perl.

In 1995, Dan Farmer released `SATAN` (Security Administrator Tool for Analyzing Networks), which added network scanning to host auditing. Its controversial name led to derivative tools like `SAINT` and `SARAH`. These tools established the pattern of modular security checks with severity ratings.

`Lynis` (2011, by Cisofy) is the modern open-source security auditing tool written in shell. It performs hundreds of checks across system hardening, malware scanning, and compliance (CIS, PCI-DSS, HIPAA). Its architecture mirrors what we build: modular audit functions with severity ratings and a summary report.

`CIS Benchmarks` (Center for Internet Security) provide the standard for system hardening assessments. Each benchmark contains specific checks like "Ensure permissions on /etc/shadow are 600" or "Ensure unnecessary SUID binaries are removed." These checks translate directly into Bash audit functions.

The principle of "defense in depth" drives security scanning: no single check catches everything, but the combination of user audit, SUID scan, cron audit, writable check, services audit, and process audit provides comprehensive coverage.

## Syntax Reference

### File Permission Checking

```bash
# Check if world-writable
[[ $(stat -c '%a' "$file") =~ ^[0-9]{3}[0-9]*[2-7]$ ]]

# Check SUID bit set
[[ -u "$file" ]]

# Check SGID bit set
[[ -g "$file" ]]

# Check sticky bit
[[ -k "$file" ]]

# Get owner
stat -c '%U' "$file"

# Get permissions in octal
stat -c '%a' "$file"

# Get permissions in symbolic
stat -c '%A' "$file"
```

### Process Inspection

```bash
# Find processes running from suspicious directories
ps -eo pid,comm,args --no-headers | grep -E '/tmp|/dev/shm'

# Check if process binary exists
readlink -f /proc/$pid/exe

# Get process environment
cat /proc/$pid/environ 2>/dev/null | tr '\0' '\n'

# Check listening ports
ss -tlnp4
ss -ulnp4

# Check established connections
ss -tlnp  # Listening
ss -tup   # Connected
```

### User Audit

```bash
# Users with UID 0
awk -F: '$3 == 0 {print $1}' /etc/passwd

# Users with empty passwords
getent shadow | awk -F: '$2 == "" || $2 == "!" {print $1}'

# Users with last login > 90 days
lastlog -b 90 | tail -n+2 | awk '{print $1}'

# SSH root login status
grep -E '^PermitRootLogin\s+' /etc/ssh/sshd_config

# Sudoers file check
grep -v '^#' /etc/sudoers | grep -v '^$'
```

### File Search

```bash
# Find all SUID binaries (worldwide)
find / -type f -perm -4000 2>/dev/null

# Find all SGID binaries
find / -type f -perm -2000 2>/dev/null

# Find world-writable directories in critical paths
find /etc /bin /usr/bin /usr/lib -type d -perm -0002 2>/dev/null

# Find files without owners
find / -nouser -o -nogroup 2>/dev/null

# Find files with +x on critical data
find /etc -type f -perm /111 2>/dev/null
```

## Under the Hood

### How `stat` Decodes Permissions

The `stat -c '%a'` output is octal. Each digit represents owner (0-7), group (0-7), other (0-7). The values are:
- 0: --- (no permission)
- 1: --x (execute)
- 2: -w- (write)
- 3: -wx (write+execute)
- 4: r-- (read)
- 5: r-x (read+execute)
- 6: rw- (read+write)
- 7: rwx (read+write+execute)

A file is world-writable if the last digit is 2, 3, 6, or 7 (write bit set for "other"). The check `[[ $perms =~ [237]$ ]]` matches this.

For SUID detection, `stat -c '%a'` prepends a fourth digit for SUID/SGID/sticky:
- 4xxx: SUID set
- 2xxx: SGID set
- 1xxx: Sticky bit set
- 6xxx: Both SUID and SGID set

### How `/proc` Reveals Process Information

Each running process has a directory in `/proc/$PID/`:
- `exe` — Symlink to the executable
- `cwd` — Symlink to current working directory
- `cmdline` — Full command line (null-separated)
- `environ` — Environment variables (null-separated)
- `fd/` — Open file descriptors
- `maps` — Memory-mapped regions
- `status` — Process status including UID/GID

A process running from `/tmp` where the binary has been deleted will have `exe` pointing to `(deleted)`.

### How `ss` Differs from `netstat`

`ss` (socket statistics) reads from `/proc/net/tcp`, `/proc/net/udp`, and `/proc/net/unix` directly. It's significantly faster than `netstat` which uses `procfs` parsing. The output format:
- `tcp LISTEN 0 128 0.0.0.0:22 0.0.0.0:* users:(("sshd",pid=1234,fd=3))`
- Fields: protocol, state, recv-q, send-q, local, remote, process info

### How Package Managers Track File Ownership

- Debian/Ubuntu: `dpkg -S /path/to/file` returns the package name
- RHEL/CentOS: `rpm -qf /path/to/file` returns the package name
- These tools maintain a database of every installed file and its package origin
- SUID binaries not belonging to any package are high-risk (custom SUID binaries are rare and suspicious)

## Core Examples

### Example 1: Complete User Audit Module

```bash
audit_users() {
  audit_header "User Audit"
  local high=0 medium=0

  # UID 0 check
  while IFS=: read -r user uid; do
    [[ $uid -eq 0 && $user != "root" ]] && {
      audit_finding HIGH "Non-root user with UID 0: $user"
      ((high++))
    }
  done < <(getent passwd | awk -F: '{print $1, $3}')

  # Empty password check
  while IFS=: read -r user pass; do
    [[ -z "$pass" || "$pass" == "!" || "$pass" == "*" ]] && continue
    passwd -S "$user" 2>/dev/null | grep -q ' NP ' && {
      audit_finding HIGH "User with no password: $user"
      ((high++))
    }
  done < /etc/shadow 2>/dev/null || audit_finding INFO "Cannot read /etc/shadow (not root)"

  # Inactive account check
  local inactive_days=90
  while IFS= read -r user; do
    local last
    last=$(lastlog -u "$user" 2>/dev/null | tail -1 | awk '{for(i=4;i<=NF;i++) printf "%s ", $i}')
    [[ -n "$last" ]] && audit_finding MEDIUM "Inactive user (90d+): $user (last: $last)"
  done < <(lastlog -b "$inactive_days" 2>/dev/null | tail -n+2 | awk '{print $1}')

  # SSH root login
  local root_login
  root_login=$(grep -E '^\s*PermitRootLogin\s+' /etc/ssh/sshd_config 2>/dev/null | awk '{print $2}')
  if [[ "$root_login" == "yes" ]] || [[ "$root_login" == "prohibit-password" ]]; then
    audit_finding HIGH "SSH root login: $root_login"
    ((high++))
  else
    audit_finding INFO "SSH root login: ${root_login:-not configured (default: prohibit-password)}"
  fi
}
```

⚠️ **TRAP:** Reading `/etc/shadow` requires root. Always handle the permission error gracefully with `2>/dev/null` and report if checks were skipped.

### Example 2: SUID/SGID Scanner with Package Verification

```bash
audit_suid() {
  audit_header "SUID/SGID Audit"
  local known_suid=0 unknown=0

  while IFS= read -r -d '' file; do
    if command -v dpkg &>/dev/null; then
      if dpkg -S "$file" &>/dev/null; then
        audit_finding INFO "SUID (packaged): $file"
        ((known_suid++))
      else
        audit_finding HIGH "SUID (unowned): $file"
        ((unknown++))
      fi
    elif command -v rpm &>/dev/null; then
      if rpm -qf "$file" &>/dev/null; then
        audit_finding INFO "SUID (packaged): $file"
        ((known_suid++))
      else
        audit_finding HIGH "SUID (unowned): $file"
        ((unknown++))
      fi
    else
      audit_finding INFO "SUID: $file (package check skipped)"
      ((known_suid++))
    fi
  done < <(find / -type f \( -perm -4000 -o -perm -2000 \) -print0 2>/dev/null)

  audit_finding INFO "Known SUID/SGID: $known_suid"
  [[ $unknown -gt 0 ]] && audit_finding HIGH "Unknown SUID/SGID: $unknown"
}
```

### Example 3: Complete Cron Audit

```bash
audit_cron() {
  audit_header "Cron & Scheduled Tasks"
  local cron_dirs=(
    /etc/crontab
    /etc/cron.d
    /etc/cron.hourly
    /etc/cron.daily
    /etc/cron.weekly
    /etc/cron.monthly
    /var/spool/cron/crontabs
  )
  local total_entries=0 world_writable=0

  for loc in "${cron_dirs[@]}"; do
    if [[ -f "$loc" ]]; then
      while IFS= read -r line; do
        [[ "$line" == \#* || -z "$line" ]] && continue
        audit_finding INFO "Cron: $loc — $line"
        ((total_entries++))
        # Extract script path and check if world-writable
        local script_path
        script_path=$(echo "$line" | grep -oP '/\S+\.\w+' | head -1)
        if [[ -n "$script_path" && -f "$script_path" ]]; then
          if [[ $(stat -c '%a' "$script_path") =~ [237]$ ]]; then
            audit_finding HIGH "World-writable cron script: $script_path (from $loc)"
            ((world_writable++))
          fi
        fi
      done < "$loc"
    elif [[ -d "$loc" ]]; then
      while IFS= read -r cronfile; do
        [[ "$cronfile" == *.dpkg-dist || "$cronfile" == *.rpmsave ]] && continue
        audit_finding INFO "Cron file: $cronfile"
        while IFS= read -r line; do
          [[ "$line" == \#* || -z "$line" ]] && continue
          audit_finding INFO "Cron entry: $cronfile — $line"
          ((total_entries++))
        done < "$cronfile"
      done < <(find "$loc" -type f 2>/dev/null)
    fi
  done

  audit_finding INFO "Total cron entries: $total_entries"
  [[ $world_writable -gt 0 ]] && audit_finding HIGH "World-writable cron scripts: $world_writable"
}
```

⚠️ **TRAP:** Cron files can have `.dpkg-dist` or `.rpmsave` suffixes from package upgrades. These are backup files and should not be treated as active cron jobs. Filter them out.

### Example 4: Writable Directory Check

```bash
audit_writable() {
  audit_header "World-Writable Directory Check"
  local critical_dirs=(
    /etc /bin /sbin /lib /lib64
    /usr/bin /usr/sbin /usr/lib
    /opt /boot
    /root
  )
  local found=0

  for d in "${critical_dirs[@]}"; do
    if [[ -d "$d" ]]; then
      local perms
      perms=$(stat -c '%a' "$d" 2>/dev/null)
      # Check if world-writable (last digit matches [237])
      if [[ $perms =~ [237]$ ]]; then
        audit_finding HIGH "World-writable critical directory: $d ($perms)"
        ((found++))
      fi
    fi
  done

  # Also check /tmp and /dev/shm permissions
  for d in /tmp /dev/shm; do
    if [[ -d "$d" ]]; then
      local perms sticky
      perms=$(stat -c '%a' "$d" 2>/dev/null)
      sticky=$(stat -c '%a' "$d" 2>/dev/null)
      # /tmp should be 1777 (world-writable with sticky bit)
      if [[ "$perms" != "1777" ]] && [[ "$perms" != "777" ]] && [[ "$perms" != "1777" ]]; then
        audit_finding MEDIUM "Non-standard /tmp permissions: $perms (expected 1777)"
      fi
    fi
  done

  [[ $found -eq 0 ]] && audit_finding INFO "No world-writable critical directories found"
}
```

### Example 5: Listening Services Audit

```bash
audit_services() {
  audit_header "Listening Services"
  local known_ports=(
    [22]=sshd [80]=http [443]=https [53]=dnsmasq
    [25]=postfix [3306]=mysql [5432]=postgres
    [6379]=redis [8080]=proxy [8443]=proxy
  )
  local unknown=0

  while read -r proto recvq sendq local foreign state program; do
    [[ -z "$local" ]] && continue
    local port
    port="${local##*:}"
    local addr="${local%:*}"

    case $proto in
      tcp) ;;
      udp6|tcp6) ;;
      *) continue ;;
    esac

    # Extract process name from program column
    local proc_name
    proc_name=$(echo "$program" | grep -oP 'users:\(\("\K[^"]+' || echo "unknown")

    if [[ ${known_ports[$port]+_} ]]; then
      audit_finding INFO "Known service: $proc_name on port $port ($proto)"
    elif [[ $port -le 1024 ]]; then
      audit_finding MEDIUM "Unknown service on privileged port: $proc_name:$port"
      ((unknown++))
    elif [[ $port -gt 1024 ]]; then
      audit_finding LOW "Service on non-standard port: $proc_name:$port"
    fi
  done < <(ss -tlnp4 2>/dev/null | tail -n+2)

  [[ $unknown -gt 0 ]] && audit_finding MEDIUM "Unknown services on privileged ports: $unknown"
}
```

⚠️ **TRAP:** `ss` output format varies across distributions. Some use tabs, some use spaces. Field positions shift when IPv6 addresses are present. Always test on your target system.

### Example 6: Process Audit

```bash
audit_processes() {
  audit_header "Process Audit"
  local total=0 suspicious=0

  while read -r pid comm args; do
    ((total++))
    # Check for processes running from /tmp or /dev/shm
    if [[ "$args" == /tmp/* ]] || [[ "$args" == /dev/shm/* ]]; then
      audit_finding HIGH "Process running from suspicious location: PID $pid ($args)"
      ((suspicious++))
    fi
    # Check for deleted binaries
    if [[ -L "/proc/$pid/exe" ]] 2>/dev/null; then
      local exe_path
      exe_path=$(readlink -f "/proc/$pid/exe" 2>/dev/null) || {
        audit_finding HIGH "Deleted binary in use: PID $pid ($args)"
        ((suspicious++))
        continue
      }
      # Check if binary is world-writable
      if [[ -f "$exe_path" ]] && [[ $(stat -c '%a' "$exe_path" 2>/dev/null) =~ [237]$ ]]; then
        audit_finding HIGH "World-writable binary in use: PID $pid ($exe_path)"
        ((suspicious++))
      fi
    fi
    # Check for processes with empty cmdline (kernel threads)
    if [[ -z "$args" || "$args" == "["* ]]; then
      continue
    fi
  done < <(ps -eo pid,comm,args --no-headers 2>/dev/null)

  audit_finding INFO "Total processes: $total"
  [[ $suspicious -gt 0 ]] && audit_finding HIGH "Suspicious processes: $suspicious"
}
```

### Example 7: Port Scanner (Internal)

```bash
TOP_PORTS=(21 22 23 25 53 80 110 111 135 139 143 443 445 993 995 1433 1521 2049 3306 3389 5432 5900 6379 8080 8443)
declare -A PORT_NAMES=(
  [21]=ftp [22]=ssh [23]=telnet [25]=smtp [53]=dns
  [80]=http [110]=pop3 [111]=rpc [135]=msrpc [139]=netbios
  [143]=imap [443]=https [445]=smb [993]=imaps [995]=pop3s
  [1433]=mssql [1521]=oracle [2049]=nfs [3306]=mysql
  [3389]=rdp [5432]=postgres [5900]=vnc [6379]=redis [8080]=http-proxy [8443]=https-alt
)

port_scan() {
  audit_header "Port Scan (Top 20)"
  local target=${1:-127.0.0.1}
  local open=0

  for port in "${TOP_PORTS[@]}"; do
    # Use /dev/tcp for bash-builtin TCP check
    timeout 2 bash -c "echo >/dev/tcp/$target/$port" 2>/dev/null && {
      local service=${PORT_NAMES[$port]:-unknown}
      audit_finding INFO "Port $port ($service) OPEN"
      ((open++))
    }
  done

  audit_finding INFO "Open ports found: $open"
  [[ $open -gt 0 ]] && audit_finding INFO "Run with --full for detailed service enumeration"
}
```

### Example 8: Report Generation with Severity

```bash
declare -i global_high=0 global_medium=0 global_low=0 global_info=0

generate_report() {
  echo -e "\n${BOLD}${UNDERLINE}=== SECURITY SCAN SUMMARY ===${NC}"
  echo -e "Scan time: $(date)"
  echo -e "Host: $(hostname) ($(hostname -I 2>/dev/null | awk '{print $1}'))"
  echo -e "Kernel: $(uname -r)"
  echo -e "Uptime: $(uptime -p)"
  echo -e "\nFindings:"
  echo -e "  ${RED}HIGH:${NC}   $global_high"
  echo -e "  ${YELLOW}MEDIUM:${NC} $global_medium"
  echo -e "  ${BLUE}LOW:${NC}    $global_low"
  echo -e "  ${GREEN}INFO:${NC}   $global_info"
  echo -e "\nScore: $(( global_high * 10 + global_medium * 3 + global_low * 1 ))/100"

  if ((global_high > 0)); then
    echo -e "${RED}❌ ISSUES FOUND — Action required${NC}"
    return 1
  elif ((global_medium > 0)); then
    echo -e "${YELLOW}⚠️  WARNINGS — Review recommended${NC}"
    return 2
  else
    echo -e "${GREEN}✅ PASS — No issues found${NC}"
    return 0
  fi
}
```

### Example 9: JSON Output

```bash
json_report() {
  local findings_json=""
  while IFS='|' read -r severity title; do
    findings_json+="{\"severity\":\"$severity\",\"finding\":\"$(json_escape "$title")\"},"
  done < "$FINDINGS_FILE"
  findings_json="${findings_json%,}"

  cat <<-JSON
{
  "scan": {
    "timestamp": $(date +%s),
    "host": "$(hostname)",
    "kernel": "$(uname -r)"
  },
  "summary": {
    "high": $global_high,
    "medium": $global_medium,
    "low": $global_low,
    "info": $global_info
  },
  "findings": [$findings_json],
  "status": "$( ((global_high > 0)) && echo 'FAIL' || echo 'PASS' )"
}
JSON
}
```

### Example 10: Main Entry Point

```bash
#!/bin/bash
set -euo pipefail
SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)

source "$SCRIPT_DIR/lib/audit.sh"

SCAN_MODE="quick"
OUTPUT_FORMAT="text"
FINDINGS_FILE=$(mktemp)
trap 'rm -f "$FINDINGS_FILE"' EXIT

while getopts ":qfo:" opt; do
  case $opt in
    q) SCAN_MODE="quick" ;;
    f) SCAN_MODE="full" ;;
    o) OUTPUT_FORMAT="$OPTARG" ;;
    \?) echo "Usage: $0 [-q] [-f] [-o text|json]" >&2; exit 1 ;;
  esac
done

show_banner

if [[ "$SCAN_MODE" == "full" ]]; then
  port_scan "127.0.0.1"
fi
audit_users
audit_suid
audit_cron
audit_writable
audit_services
audit_processes

if [[ "$OUTPUT_FORMAT" == "json" ]]; then
  json_report
else
  generate_report
fi
```

⚠️ **TRAP:** `mktemp` creates a file. Always clean it up with `trap 'rm -f ...' EXIT`. If the script crashes before the trap is set, `/tmp` fills with temp files.

### Example 11: Package Integrity Check

```bash
audit_packages() {
  audit_header "Package Integrity"
  if command -v debsums &>/dev/null; then
    debsums -s 2>/dev/null | while IFS= read -r line; do
      audit_finding HIGH "Modified package file: $line"
    done
  elif command -v rpm --verify &>/dev/null; then
    rpm --verify -a 2>/dev/null | while IFS= read -r line; do
      audit_finding HIGH "Modified package file: $line"
    done
  else
    audit_finding INFO "Package verification tool not available (install debsums or rpm)"
  fi
}
```

### Example 12: FileSystem Mount Audit

```bash
audit_mounts() {
  audit_header "Filesystem Mount Audit"
  while IFS=' ' read -r device mount fstype opts _; do
    [[ "$device" == "none" || "$device" == "proc" || "$fstype" == "tmpfs" ]] && continue
    if [[ "$opts" == *"noexec"* ]]; then
      audit_finding INFO "$mount: noexec set (good)"
    else
      audit_finding MEDIUM "$mount: noexec NOT set (user could execute from $mount)"
    fi
    if [[ "$opts" == *"nosuid"* ]]; then
      audit_finding INFO "$mount: nosuid set (good)"
    else
      audit_finding MEDIUM "$mount: nosuid NOT set (SUID possible on $mount)"
    fi
  done < <(mount | grep '^/')
}
```

### Example 13: SSH Configuration Audit

```bash
audit_ssh() {
  audit_header "SSH Configuration Audit"
  local sshd_config="/etc/ssh/sshd_config"
  [[ ! -f "$sshd_config" ]] && { audit_finding INFO "SSH not installed"; return; }

  local checks=(
    "PermitRootLogin:no:no|prohibit-password"
    "PasswordAuthentication:yes:no"
    "X11Forwarding:yes:no"
    "MaxAuthTries:6:3"
    "ClientAliveInterval:0:300"
    "Protocol:2:2"
  )

  for check in "${checks[@]}"; do
    local key expected_good actual
    IFS=: read -r key _ expected_good <<< "$check"
    actual=$(grep -E "^\s*${key}\s+" "$sshd_config" 2>/dev/null | awk '{print $2}')
    local expected
    expected=$(echo "$expected_good" | cut -d'|' -f1)
    if [[ -z "$actual" ]]; then
      audit_finding MEDIUM "$key not set (default may be insecure)"
    elif [[ "$actual" == "yes" ]] || [[ "$actual" == "no" ]]; then
      if [[ "$expected_good" == *"$actual"* ]]; then
        audit_finding INFO "$key = $actual (good)"
      else
        audit_finding MEDIUM "$key = $actual (should be $expected)"
      fi
    fi
  done
}
```

### Example 14: Find Orphaned Files

```bash
audit_orphaned() {
  audit_header "Orphaned File Check"
  local orphans=0
  while IFS= read -r -d '' file; do
    audit_finding MEDIUM "Orphaned file (no owner): $file"
    ((orphans++))
  done < <(find / -nouser -o -nogroup -print0 2>/dev/null | head -50)
  [[ $orphans -eq 0 ]] && audit_finding INFO "No orphaned files found"
}
```

### Example 15: Docker Security Check

```bash
audit_docker() {
  audit_header "Docker Security"
  command -v docker &>/dev/null || { audit_finding INFO "Docker not installed"; return; }
  if docker info 2>/dev/null | grep -q "root"; then
    if groups "$(whoami)" 2>/dev/null | grep -q docker; then
      audit_finding MEDIUM "User in docker group (potential privilege escalation)"
    fi
  fi
  # Check running containers
  local containers
  containers=$(docker ps -q 2>/dev/null | wc -l)
  audit_finding INFO "Running containers: $containers"
  # Check for privileged containers
  while read -r cid; do
    if docker inspect "$cid" 2>/dev/null | grep -q '"Privileged": true'; then
      audit_finding HIGH "Privileged container running: $cid"
    fi
  done < <(docker ps -q 2>/dev/null)
}
```

## Real-World Use Cases

1. **Lynis** — The most popular open-source security auditing tool. Written in shell, it performs 300+ checks across system hardening, malware, file integrity, and custom tests. Its output format: warnings, suggestions, and a "hardening index" score.

2. **CIS-CAT** — Center for Internet Security's Configuration Assessment Tool. Compares system configuration against CIS Benchmarks. Enterprise tool but the underlying checks are the same Bash-level inspections.

3. **OSQuery** — While using SQL for queries, OSQuery exposes the same information: processes, listening ports, SUID binaries, users. The translation: our Bash `ps` commands become SQL `SELECT * FROM processes`.

4. **Security Onion** — A Linux distribution for security monitoring. Its setup scripts perform comprehensive system hardening checks before deployment, all in Bash.

5. **Bastille Linux** — An early (2000) hardening script that locked down Linux systems. It checked permissions, users, services, and applied hardening. Now largely replaced by `CIS` and `STIG` automation.

6. **PCI-DSS compliance scanners** — Many PCI compliance scanning tools run shell-based checks on POS systems to verify file permissions, user access, and configuration.

7. **Kubernetes Pod Security** — Admission controllers use Bash-like checks (via `kubectl` and `jq`) to verify pod security contexts, read-only root filesystems, and non-root users.

## Memory Aids

- **PUSCWP**: Ports, Users, SUID, Cron, Writable, Processes — the six core audit modules.
- **CHAMP**: Checks, High/Medium/Low, Alerts, Mitigate, Prevent — the audit workflow.
- **SEVEN P's**: Permissions, Passwords, Processes, Ports, Packages, Partitions, Production — what to check.
- **Permission Octal Cue**: r=4, w=2, x=1. Sum per group. World-writable check = last digit in [2,3,6,7].
- **TRAP Sequence**: Test (the system), Report (findings), Act (mitigate), Prevent (monitor).

## Trap Vault

1. **⚠️ TRAP:** `find / -type f -perm -4000` returns files that have the SUID bit set. The `-4000` means "at least bit 12 (octal 4000) is set." For SUID-only files, use `-perm -4000`. For SGID-only, `-perm -2000`. For both, `-perm -6000`.

2. **⚠️ TRAP:** `/proc` filesystem inspection reveals processes, but some entries are kernel threads (no `exe` link, cmdline in brackets). Filter `[...]` entries from process lists.

3. **⚠️ TRAP:** `ss -tlnp4` shows IPv4 TCP listening sockets with process info. Without `-p` (process), you don't see which program owns the socket. Without root, some process info is hidden.

4. **⚠️ TRAP:** Package manager verification (`dpkg -S`, `rpm -qf`) fails for files that were never part of a package. This is common for custom scripts in `/usr/local/bin` — these are legitimate but appear "unowned."

5. **⚠️ TRAP:** World-writable `/tmp` is normal (sticky bit: `1777`). Flagging all world-writable directories as security issues creates false positives. Only flag world-writable directories in critical paths.

6. **⚠️ TRAP:** `lastlog -b 90` shows users who haven't logged in for 90+ days. But system accounts (daemon, bin, sys) never log in — they're expected to be "inactive." Filter UIDs < 1000.

7. **⚠️ TRAP:** Cron entries may reference scripts that no longer exist. A cron job for `/opt/scripts/backup.sh` that has been deleted still appears in the cron file. Check existence of referenced scripts.

8. **⚠️ TRAP:** `ss` and `netstat` show different info. `ss` shows UNIX domain sockets by default; use `-t` for TCP, `-u` for UDP. Mixing them hides listening services.

9. **⚠️ TRAP:** File permissions revealed by `stat -c '%a'` on symlinks show the target's permissions, not the link's. Use `stat -L` to follow symlinks, or no flag to show link metadata.

10. **⚠️ TRAP:** `ps` output columns vary by system and `ps` variant (procps vs busybox). Always specify exact columns with `-o` rather than relying on default format.

11. **⚠️ TRAP:** `audit_finding` functions that increment counters inside `while read` loops run in a subshell if piped. Use process substitution (`< <(cmd)`) instead of pipes to avoid subshell counter loss.

12. **⚠️ TRAP:** SUID binaries in containers vs hosts. Container filesystems have their own SUID binaries. If scanning inside a container, you'll find SUID files in the container image — these are expected, not suspicious.

13. **⚠️ TRAP:** The `/etc/shadow` file requires root to read. If running as non-root, the scanner cannot check for empty passwords. Report this limitation rather than silently skipping the check.

14. **⚠️ TRAP:** Custom audit findings must be deduplicated. If the same SUID file appears in both the SUID scan and the process audit, it's reported twice. Use a findings file with dedup logic.

15. **⚠️ TRAP:** The scanner itself creates processes, opens files, and may trigger intrusion detection systems. Running a security scanner on a compromised system may alert the attacker.

## See It In The Wild

Lynis is the canonical real-world example of a shell-based security scanner. Its architecture mirrors this lesson: modular audit functions in `include/` directory, a main entry point (`lynis`), profile-based configuration, and severity-rated output.

Lynis checks include: `AUTH-9308` (Check password hashing methods), `FILE-6370` (Check SUID integrity), `KRNL-6000` (Check kernel hardening). Each check follows:
```bash
Register --id "AUTH-9308" --category "authentication" --description "Check password hashing algorithm"
if condition; then
  Report --warning --text "SHA256 not in use" --suggestion "Use SHA512 in /etc/shadow"
fi
```

The CIS Benchmark for Linux provides the actual compliance criteria. For example:
- "1.1.1.1 Ensure mounting of cramfs is disabled" — checked via `modprobe -n -v cramfs`
- "1.5.1 Ensure core dumps are restricted" — checked via `ulimit -c` and `/etc/security/limits.conf`
- "6.2.1 Ensure password fields are not empty" — checked via `awk -F: '($2 == "" )' /etc/shadow`

These CIS checks translate directly into the functions we've built.

## Check Your Understanding

1. Why should you run `2>/dev/null` on many security scan commands like `find` and `stat`?

2. How would an attacker evade a simple SUID scan? What would you add to catch evasion?

3. Why are world-writable scripts in cron a HIGH severity finding?

4. How does JSON output help integrate with other security tools like SIEMs?

5. What is the limitation of scanning only the top 20 ports? How would a full scan differ?

6. How does `ss -tlnp4` differ from `ss -tulnp`? When would you use each?

7. Why must the findings counter be managed with a temp file rather than a global variable in subshells?

8. How would you distinguish between a legitimate custom SUID binary and a malicious one?

9. What false positives might a user audit produce, and how would you reduce them?

10. How would you add a `--baseline` mode that only reports findings different from a previous scan?
