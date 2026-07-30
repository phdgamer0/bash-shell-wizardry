# Task 20: Capstone — Comprehensive Security Scanner

## Objective

Build a comprehensive system security scanner with colored output, severity ratings, JSON output option, and a summary report. Must include at least 7 audit modules covering ports, users, SUID/SGID binaries, cron jobs, writable directories, listening services, and processes.

## Requirements

### Modules (Minimum 7)

#### 1. Port Scanner Module

Port Scan: Check top 25 TCP ports on localhost (or specified host)
- Use /dev/tcp built-in with timeout
- Top ports: 21,22,23,25,53,80,110,111,135,139,143,443,445,993,995,1433,1521,2049,3306,3389,5432,5900,6379,8080,8443
- Report open ports with service name
- Flag unexpected open ports (non-standard for that port number)
- Support --target HOST to scan remote hosts
- Support --port-range START-END for custom range

#### 2. User Audit Module

User Audit: Comprehensive user account review
- Non-root users with UID 0 (HIGH)
- Users with empty passwords (HIGH)
- Inactive accounts (>90 days since last login) (MEDIUM)
- SSH root login configuration (HIGH if enabled)
- Users in sudo/wheel group without password (MEDIUM)
- Duplicate UIDs (MEDIUM)
- Users with valid login shell but no password (MEDIUM)
- Last password change > 1 year ago (LOW)

#### 3. SUID/SGID Scanner Module

SUID/SGID Scan: Find all setuid/setgid binaries
- Find all files with SUID (perm -4000) or SGID (perm -2000) set
- Verify against package manager (dpkg/rpm) — flag unowned as HIGH
- Report SUID files with world-writable permissions (HIGH)
- Report SUID files in non-standard locations (/tmp, /dev/shm, /home) (HIGH)
- Count known vs unknown SUID files
- Cache results to allow diff-based change detection

#### 4. Cron Audit Module

Cron Audit: Enumerate all scheduled jobs
- Check /etc/crontab, /etc/cron.d/, /etc/cron.hourly|daily|weekly|monthly/
- Check /var/spool/cron/crontabs/ for per-user crontabs
- Flag world-writable scripts referenced in cron entries (HIGH)
- Flag cron entries pointing to non-existent scripts (MEDIUM)
- Flag cron entries running as root when not necessary (LOW)
- Report total cron entries by directory
- Detect .dpkg-dist / .rpmsave backups mixed in cron dirs

#### 5. Writable Directory Check Module

Writable Directory Check: Find dangerous world-writable locations
- Check critical dirs: /etc, /bin, /sbin, /usr/bin, /usr/sbin, /usr/lib, /boot, /opt
- Check /tmp and /dev/shm for correct permissions (1777 expected)
- Check /root for world-readable/writable (HIGH)
- Check /var/www permissions if directory exists (MEDIUM)
- Report world-writable files in /etc (HIGH)

#### 6. Listening Services Module

Services Scan: Identify all listening network services
- Parse ss -tlnp4 output for TCP services
- Parse ss -ulnp4 output for UDP services
- Match ports to known services (dictionary of 30+ common ports)
- Flag unknown services on privileged ports (<1024) as MEDIUM
- Flag services listening on 0.0.0.0 when only localhost expected (MEDIUM)
- Report process name, PID, port, and interface for each service

#### 7. Process Audit Module

Process Audit: Inspect running processes for anomalies
- Processes running from /tmp, /dev/shm (HIGH)
- Processes with deleted binary (HIGH)
- Processes running as root that don't need it (MEDIUM)
- Processes with open listening sockets but unknown binary (MEDIUM)
- High resource consumers (CPU > 50%, MEM > 50%) (LOW)
- Total process count vs system limits
- Zombie processes (LOW)

#### 8. Additional Module Options (Choose 1+)

- SSH Configuration Audit: Check sshd_config for secure settings
- Package Integrity: Check modified files via debsums/rpm --verify
- Filesystem Mounts: Check noexec, nosuid, nodev options
- Kernel Parameter Audit: Check sysctl security settings
- Docker Security: Check container permissions and configurations
- Orphaned File Check: Files with no user/group owner

### Output Modes

#### Text Mode (default)

Colored output with severity icons and organized modules.

#### JSON Mode (--output json)

Machine-readable JSON with all findings, severity counts, and metadata.

#### CSV Mode (--output csv)

Comma-separated values for import into spreadsheets.

### Command-Line Interface

```
scanner.sh [options]

Options:
  -m, --modules MODULES    Comma-separated list of modules to run (default: all)
  -q, --quick              Quick scan (skip port scan, skip deep file search)
  -f, --full               Full scan (all modules, deep port scan, process inspection)
  -o, --output FORMAT      Output format: text, json, csv (default: text)
  -t, --target HOST        Target host for port scan (default: 127.0.0.1)
  -c, --config FILE        Config file path
  --baseline FILE          Compare against baseline from previous scan
  --no-color               Disable colored output
  -h, --help               Show help
  --version                Show version
```

### Configuration File

Location: /etc/scanner/scanner.conf or ~/.config/scanner/scanner.conf

```bash
# scanner.conf
SCAN_MODE="quick"
OUTPUT_FORMAT="text"
TARGET_HOST="127.0.0.1"
PORT_RANGE="1-1024"
TOP_PORTS_ONLY=true
INACTIVE_DAYS=90
KNOWN_SUID_FILE="/var/lib/scanner/known_suid.txt"
SUID_CACHE_FILE="/var/lib/scanner/suid_cache.txt"
ALERT_EMAIL=""
EXCLUDE_USERS="daemon:bin:sys:sync:games:man:lp:mail:news:uucp"
EXCLUDE_MODULES="docker"
```

### Finding Scoring

Each finding contributes to a numerical score:

- HIGH: 10 points
- MEDIUM: 3 points
- LOW: 1 point
- INFO: 0 points

Score interpretation:
- 0-5: PASS (no issues)
- 6-20: WARNING (review recommended)
- 21+: FAIL (action required)

### Exit Codes

- 0: PASS (no HIGH findings)
- 1: WARNING (MEDIUM findings but no HIGH)
- 2: FAIL (HIGH findings exist)
- 3: Error (config error, permission denied, etc.)

## Sub-tasks

### 1. Project Structure

```
scanner/
├── scanner.sh                # Main entry point
├── lib/
│   ├── audit.sh              # Audit framework (header, finding, report)
│   ├── output.sh             # Output formatting (text, json, csv)
│   ├── config.sh             # Config file loading
│   └── utils.sh              # Utility functions
├── modules/
│   ├── port_scan.sh          # Port scanning
│   ├── user_audit.sh         # User audit
│   ├── suid_scan.sh          # SUID/SGID scan
│   ├── cron_audit.sh         # Cron audit
│   ├── writable_check.sh     # Writable directory check
│   ├── services_scan.sh      # Listening services
│   ├── process_audit.sh      # Process audit
│   └── (optional extras)     # SSH audit, mounts, kernel, etc.
└── conf/
    └── scanner.conf          # Default config
```

### 2. Audit Framework (lib/audit.sh)

- audit_header(title) — Print section header with formatting
- audit_finding(severity, message) — Record finding with severity, increment counter
- audit_summary() — Print/serialize summary of all findings
- init_counters() — Reset severity counters
- get_score() — Calculate numerical security score
- get_status(score) — Return PASS/WARNING/FAIL based on score
- dedup_findings() — Remove duplicate findings

### 3. Output Module (lib/output.sh)

- output_text() — Format findings as colored text
- output_json() — Format findings as JSON
- output_csv() — Format findings as CSV
- json_escape(string) — Escape special characters for JSON
- csv_escape(string) — Escape commas and quotes for CSV
- show_banner() — Display tool banner with version

### 4. Config Module (lib/config.sh)

- Load from multiple search paths
- Validate config values
- Handle boolean, numeric, and list config types
- Support --config FILE override

### 5. Implement Each Audit Module

Each module in modules/ should:
- Be a standalone script sourced by main
- Define a main function (e.g., run_user_audit)
- Source the audit framework for finding reporting
- Handle errors gracefully (2>/dev/null for permission errors)
- Support both quick and full scan modes via MODE variable
- Return count of findings by severity

### 6. Main Entry Point (scanner.sh)

- Parse CLI arguments
- Load config
- Source lib files
- Source selected modules
- Run each module in sequence
- Generate report in selected format
- Exit with appropriate code

### 7. Baseline Comparison (--baseline)

- Save scan results to a file
- On subsequent runs with --baseline FILE, compare findings
- Report new findings since last scan
- Report resolved findings since last scan
- Format: JSON lines with finding hash for dedup

### 8. Initialization and Testing

- Generate test environment with known security issues:
  - Create a world-writable directory in /tmp
  - Create a test user with weak password
  - Create a test SUID binary (script)
  - Add a cron entry pointing to a world-writable script
- Run scanner and verify all issues are detected
- Run scanner as non-root and verify graceful degradation

## Expected Output

### Quick Scan (Text Mode)

```
$ ./scanner.sh --quick --no-color

=== Security Scanner v1.0 ===
Scan time: Fri Jul 31 10:00:00 UTC 2026
Host: myserver (192.168.1.100)
Kernel: 6.1.0-10-amd64
Mode: Quick

=== User Audit ===
  [HIGH] Non-root user with UID 0: bob (should be root only)
  [HIGH] SSH root login with password enabled
  [MEDIUM] User 'guest' inactive for 120 days (last: 2026-04-01)
  [INFO] All users have password set

=== SUID/SGID Audit ===
  [HIGH] SUID binary not from package: /opt/custom/bin/helper
  [INFO] /usr/bin/su — packaged (dpkg: coreutils)
  [INFO] /usr/bin/sudo — packaged (dpkg: sudo)
  [INFO] Known SUID/SGID: 12
  [INFO] Unknown SUID/SGID: 1

=== Cron Audit ===
  [HIGH] World-writable cron script: /opt/scripts/backup.sh (from /etc/cron.d/backup)
  [INFO] /etc/crontab: 0 2 * * * root /usr/local/bin/daily-backup
  [INFO] Total cron entries: 8

=== Writable Directory Check ===
  [HIGH] World-writable critical directory: /opt/staging (777)
  [INFO] No world-writable directories in /etc, /bin, /usr/bin

=== Process Audit ===
  [HIGH] Process running from suspicious location: PID 3456 (/tmp/.hidden/exploit)
  [HIGH] Deleted binary in use: PID 4567 (/tmp/deleted-script)
  [MEDIUM] PID 5678 (custom-daemon) — no binary on disk
  [INFO] Total processes: 128

=== SECURITY SCAN SUMMARY ===
HIGH:   3
MEDIUM: 2
LOW:    0
INFO:   18
Score: 36/100
Status: FAIL — Action required
```

### Full Scan with Port Scanner

```
$ ./scanner.sh --full

=== Port Scan (Top 25) ===
  [INFO] Port 22 (ssh) OPEN — expected
  [INFO] Port 80 (http) OPEN — expected
  [INFO] Port 443 (https) OPEN — expected
  [INFO] Port 3306 (mysql) OPEN — internal only (expected)
  [MEDIUM] Port 9999 (unknown) OPEN — unknown service

=== Services Scan ===
  [INFO] sshd:22 (PID: 1234) — listening on 0.0.0.0
  [INFO] nginx:80 (PID: 2345) — listening on 0.0.0.0
  [INFO] nginx:443 (PID: 2345) — listening on 0.0.0.0
  [MEDIUM] mysql:3306 (PID: 3456) — listening on 0.0.0.0 (should be 127.0.0.1)
```

### JSON Output

```
$ ./scanner.sh --full --output json | jq .
{
  "scan": {
    "timestamp": 1722412800,
    "host": "myserver",
    "kernel": "6.1.0-10-amd64",
    "mode": "full"
  },
  "summary": {
    "high": 3,
    "medium": 2,
    "low": 0,
    "info": 18,
    "score": 36,
    "status": "FAIL"
  },
  "modules": {
    "port_scan": {"findings": 4, "high": 0, "medium": 1},
    "user_audit": {"findings": 4, "high": 2, "medium": 1},
    "suid_scan": {"findings": 5, "high": 1, "medium": 0},
    "cron_audit": {"findings": 3, "high": 1, "medium": 0},
    "writable_check": {"findings": 2, "high": 1, "medium": 0},
    "services_scan": {"findings": 4, "high": 0, "medium": 1},
    "process_audit": {"findings": 4, "high": 2, "medium": 1}
  },
  "findings": [
    {"severity": "HIGH", "module": "user_audit", "message": "Non-root user with UID 0: bob"},
    {"severity": "HIGH", "module": "suid_scan", "message": "SUID binary not from package: /opt/custom/bin/helper"}
  ]
}
```

### CSV Output

```
$ ./scanner.sh --full --output csv
severity,module,message
HIGH,user_audit,Non-root user with UID 0: bob
HIGH,user_audit,SSH root login with password enabled
HIGH,suid_scan,SUID binary not from package: /opt/custom/bin/helper
HIGH,cron_audit,World-writable cron script: /opt/scripts/backup.sh
HIGH,writable_check,World-writable critical directory: /opt/staging
HIGH,process_audit,Process running from suspicious location: PID 3456
HIGH,process_audit,Deleted binary in use: PID 4567
MEDIUM,user_audit,User guest inactive for 120 days
MEDIUM,port_scan,Port 9999 (unknown) OPEN
MEDIUM,process_audit,PID 5678 — no binary on disk
```

### Baseline Comparison

```
$ ./scanner.sh --baseline /var/lib/scanner/baseline.json
--- Security Scanner — Baseline Comparison ---
Baseline: /var/lib/scanner/baseline.json (2026-07-30 10:00:00)

NEW findings since baseline:
  [HIGH] Non-root user with UID 0: bob (NEW)
  [HIGH] SUID binary not from package: /opt/custom/bin/helper (NEW)

RESOLVED findings since baseline:
  [INFO] World-writable directory /opt/staging — FIXED

No change: 15 findings (unchanged since baseline)
```

### Error Handling

```
$ ./scanner.sh --target
[ERROR] --target requires an argument

$ ./scanner.sh --invalid-flag
[ERROR] Unknown option: --invalid-flag
Usage: scanner.sh [options]

$ ./scanner.sh --output xml
[ERROR] Invalid output format: xml (valid: text, json, csv)

$ ./scanner.sh --modules nonexistent
[ERROR] Unknown module: nonexistent
Available modules: port_scan, user_audit, suid_scan, cron_audit, writable_check, services_scan, process_audit
```

### Permission Denied (Non-Root)

```
$ ./scanner.sh --quick
[INFO] Running as user: alice (not root — some checks will be limited)

=== User Audit ===
[WARN] Cannot read /etc/shadow (permission denied) — skipping password checks
[INFO] Checking /etc/passwd only (accessible to all users)
...
```

### Test Environment Results

```
$ ./scanner.sh --full --output text
...

=== SECURITY SCAN SUMMARY ===
HIGH:   5
MEDIUM: 3
LOW:    1
INFO:   22
Score: 60/100
Status: FAIL — Action required
Exit code: 2
```

## Hints

<details>
<summary>Hint 1: Module Discovery and Loading</summary>

```bash
# Auto-discover modules in modules/ directory
load_modules() {
  local selected=("$@")
  if [[ ${#selected[@]} -eq 0 ]]; then
    for mod in "$SCRIPT_DIR/modules"/*.sh; do
      source "$mod"
    done
  else
    for mod in "${selected[@]}"; do
      local mod_file="$SCRIPT_DIR/modules/${mod}.sh"
      if [[ -f "$mod_file" ]]; then
        source "$mod_file"
      else
        echo "ERROR: Module not found: $mod" >&2
        return 1
      fi
    done
  fi
}
```
</details>

<details>
<summary>Hint 2: Finding Deduplication</summary>

```bash
declare -A finding_hashes

record_finding() {
  local severity=$1 message=$2
  local hash
  hash=$(echo "$message" | md5sum 2>/dev/null | cut -d' ' -f1 || echo "$message" | sha1sum | cut -d' ' -f1)
  if [[ -z "${finding_hashes[$hash]}" ]]; then
    finding_hashes[$hash]=1
    echo "$severity|$message" >> "$FINDINGS_FILE"
    ((severity_count[$severity]++))
  fi
}
```
</details>

<details>
<summary>Hint 3: Port Scan with Timeout</summary>

```bash
check_port() {
  local host=$1 port=$2 timeout=${3:-2}
  # Use bash /dev/tcp with timeout
  timeout "$timeout" bash -c "echo >/dev/tcp/$host/$port" 2>/dev/null && return 0
  return 1
}

scan_ports() {
  local host=$1
  shift
  local ports=("$@")
  for port in "${ports[@]}"; do
    if check_port "$host" "$port"; then
      echo "OPEN: $port"
    fi
  done
}
```
</details>

<details>
<summary>Hint 4: JSON Output Construction</summary>

```bash
json_array() {
  local name=$1
  shift
  local items=("$@")
  printf '"%s": [' "$name"
  local first=true
  for item in "${items[@]}"; do
    $first || printf ","
    printf "%s" "$item"
    first=false
  done
  printf "]"
}
```
</details>

<details>
<summary>Hint 5: Baseline Comparison</summary>

```bash
compare_baseline() {
  local baseline_file=$1 current_file=$2
  local new_findings=()
  local resolved_findings=()

  while IFS='|' read -r sev msg; do
    if ! grep -q "|$msg" "$baseline_file"; then
      new_findings+=("$sev|$msg")
    fi
  done < "$current_file"

  while IFS='|' read -r sev msg; do
    if ! grep -q "|$msg" "$current_file"; then
      resolved_findings+=("$sev|$msg")
    fi
  done < "$baseline_file"
}
```
</details>

<details>
<summary>Hint 6: Module Execution with Timing</summary>

```bash
run_module() {
  local module_name=$1
  shift
  local start_time end_time elapsed
  start_time=$(date +%s%N)
  "run_${module_name}" "$@"
  end_time=$(date +%s%N)
  elapsed=$(( (end_time - start_time) / 1000000 ))
  log info "Module $module_name completed in ${elapsed}ms"
}
```
</details>

<details>
<summary>Hint 7: Color and Format Helpers</summary>

```bash
if [[ -t 1 ]]; then
  BOLD='\033[1m'; RED='\033[0;31m'
  GREEN='\033[0;32m'; YELLOW='\033[1;33m'
  BLUE='\033[0;34m'; NC='\033[0m'
  ICON_HIGH="[${RED}HIGH${NC}]"
  ICON_MED="[${YELLOW}MED${NC}]"
  ICON_LOW="[${BLUE}LOW${NC}]"
  ICON_INFO="[${GREEN}INFO${NC}]"
else
  BOLD=''; RED=''; GREEN=''; YELLOW=''; BLUE=''; NC=''
  ICON_HIGH="[HIGH]"
  ICON_MED="[MED]"
  ICON_LOW="[LOW]"
  ICON_INFO="[INFO]"
fi
```
</details>

<details>
<summary>Hint 8: Module Function Naming Convention</summary>

```bash
# Each module defines: run_MODULENAME
# Module: modules/port_scan.sh
run_port_scan() {
  audit_header "Port Scan"
  # ...implementation...
}

# Module: modules/user_audit.sh
run_user_audit() {
  audit_header "User Audit"
  # ...implementation...
}

# Main dispatch
for mod_func in $(declare -F | grep -oP 'run_\w+'); do
  run_module "${mod_func#run_}"
done
```
</details>

## Self-Check

1. Why should you run 2>/dev/null on many security scan commands like find, stat, and ss?

2. How would an attacker evade a simple SUID scan? What would you add to catch evasion?

3. Why are world-writable scripts in cron a HIGH severity finding?

4. How does JSON output help integrate with other security tools like SIEMs and dashboards?

5. What is the limitation of scanning only the top 20 ports? How would a full scan differ?

6. How does ss -tlnp4 differ from ss -tulnp? When would you use each?

7. Why must the findings counter be managed with a temp file rather than a global variable in subshells?

8. How would you distinguish between a legitimate custom SUID binary and a malicious one?

9. What false positives might a user audit produce, and how would you reduce them?

10. How would you add a --baseline mode that only reports findings different from a previous scan?
