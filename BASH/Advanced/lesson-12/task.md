# Task 12: Secure Temp File Handler & TOCTOU Detection Toolkit

## Objective

Build a secure temp file handler library, write a TOCTOU vulnerability detector that scans shell scripts for race conditions, and create a sticky bit auditing tool. This task focuses on writing defensive code and detecting vulnerabilities in existing scripts.

## Requirements (8 sub-tasks)

### Sub-task 1: Secure Temp File Library
Create a library `lib/safe_temp.sh` with the following functions:
- `safe_temp_file [template]` — Creates a temp file with mktemp, registers cleanup trap, returns path
- `safe_temp_dir [template]` — Creates a temp directory with mktemp -d, registers cleanup trap, returns path
- `safe_temp_cleanup_all` — Removes all temp files/directories created by the library (for manual cleanup)
- All functions must be idempotent (safe to call multiple times)
- All functions must handle SIGTERM, SIGINT, and EXIT for cleanup

### Sub-task 2: TOCTOU Detector
Write `toctou_detector.sh` that scans files/directories for common TOCTOU patterns:
- Predictable temp names (`$$.tmp`, `/tmp/.*.pid`, hardcoded paths in `/tmp/`)
- Check-then-use patterns (`[ -f file ] && ... > file`)
- Non-atomic file operations (separate check and update)
- Missing trap cleanup for temp files
- Use of `mktemp -u` (dangerous)
- Hardcoded paths in `/tmp/` or `/var/tmp/`
- Flags each finding with filename, line number, and severity

### Sub-task 3: Fix Vulnerable Script
Given the following vulnerable script, rewrite it securely:

```bash
#!/bin/bash
# VULNERABLE — contains TOCTOU and race conditions
LOCKFILE=/tmp/process.lock
TEMPFILE=/tmp/tempdata.txt

if [ ! -f "$LOCKFILE" ]; then
  echo "$$" > "$LOCKFILE"
  if [ ! -f "$TEMPFILE" ]; then
    echo "initial" > "$TEMPFILE"
  fi
  echo "data" >> "$TEMPFILE"
  rm -f "$LOCKFILE"
fi
```

Your rewrite must:
- Use mktemp for all temp files
- Use flock for locking (not a PID file)
- Add proper trap cleanup
- Add `set -euo pipefail`
- Handle signals gracefully

### Sub-task 4: Sticky Bit Auditor
Write `sticky_bit_checker.sh` that:
- Finds all world-writable directories WITHOUT the sticky bit
- Reports each with path, permissions, and owner
- Flags critical system directories (/etc, /bin, /usr/bin, /opt) as HIGH severity
- Provides `--fix` option that adds the sticky bit (with confirmation)
- Provides `--report` option for machine-readable output

### Sub-task 5: /proc Exposure Scanner
Write `proc_exposure_scan.sh` that:
- Checks /proc mount options: `mount | grep /proc`
- Reports if `/proc` is mounted without `hidepid=2`
- Checks world-readable `/proc/PID/fd/` entries for sensitive paths
- Checks `/proc/PID/environ` accessibility
- Reports which processes have open FDs to sensitive files (shadow, passwd, keys)
- Rates exposure level: LOW (hidepid=2), MEDIUM (hidepid=1), HIGH (no hidepid)

### Sub-task 6: Atomic File Update Function
Write a function `atomic_write` that:
- Writes content to a temp file in the same directory (same filesystem for atomic rename)
- Uses `mv` for atomic rename to the final path
- Preserves original file permissions if the target exists
- Handles concurrent writers (use flock on a lock file)
- Logs the operation with timestamps

### Sub-task 7: Integration Test Framework
Write a test script that:
- Creates a vulnerable temp scenario (with race window simulation)
- Runs your detector against it — should find the vulnerability
- Runs your sticky bit checker
- Verifies that the secure version passes all checks
- Reports PASS/FAIL for each test case
- Measures and reports race window detection rate

## Bonus Challenges

1. **Race demonstration:** Write a script that DEMONSTRATES a TOCTOU race by running 100 parallel attacker processes against a vulnerable handler. Time how long it takes to win the race.

2. **NFS detection:** Add NFS detection to the sticky bit checker — NFS volumes can't properly enforce sticky bits.

3. **Seccomp integration:** Research and describe how seccomp-bpf profiles could prevent TOCTOU attacks (e.g., blocking `access()` after `open()`).

4. **Kernel module monitor:** Write an inotify-based watcher that monitors key system directories for TOCTOU attacks in real time.

5. **Setuid script database:** Build a database of known vulnerable setuid script patterns and add a checker for them.

## Hints

<details>
<summary>Hint 1: Safe temp function with trap stacking</summary>

```bash
safe_temp_file() {
  local template="${1:-mytemp}"
  local tmp
  tmp=$(mktemp "/tmp/${template}.XXXXXX") || return 1
  # Store for cleanup
  TEMP_FILES+=("$tmp")
  # Cleanup on exit
  trap 'rm -f "${TEMP_FILES[@]}"' EXIT
  trap 'rm -f "${TEMP_FILES[@]}"; exit' SIGTERM SIGINT
  echo "$tmp"
}
```
</details>

<details>
<summary>Hint 2: TOCTOU pattern detection with regex</summary>

```bash
patterns=(
  '\[[^]]*-f[^]]*\].*[>|]'     # Check -f then redirect
  '\$\$\.tmp'                    # PID-based temp
  '/tmp/.*\.pid'                 # PID files in /tmp
  'mktemp -u'                    # Unsafe mktemp
  '/tmp/[A-Za-z]*\.[A-Za-z]*'   # Hardcoded temp paths
)

scan_file() {
  local file="$1"
  for i in "${!patterns[@]}"; do
    while IFS= read -r line; do
      lineno=$(echo "$line" | cut -d: -f1)
      content=$(echo "$line" | cut -d: -f2-)
      echo "LINE $lineno: PATTERN $i: $content"
    done < <(grep -nE "${patterns[$i]}" "$file" 2>/dev/null)
  done
}
```
</details>

<details>
<summary>Hint 3: Fixed secure version of the vulnerable script</summary>

```bash
#!/bin/bash
set -euo pipefail

WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT

LOCKFILE="$WORKDIR/process.lock"
TEMPFILE="$WORKDIR/tempdata.txt"

exec 3>"$LOCKFILE"
if flock -n 3; then
  # Safe to use the lock
  echo "$$" > "$LOCKFILE"
  if [ ! -f "$TEMPFILE" ]; then
    echo "initial" > "$TEMPFILE"
  fi
  echo "data" >> "$TEMPFILE"
fi
exec 3>&-
```
</details>

<details>
<summary>Hint 4: Sticky bit finder with classification</summary>

```bash
find_world_writable_no_sticky() {
  find / -type d -perm -0002 ! -perm -1000 2>/dev/null | while read dir; do
    perms=$(stat -c '%a' "$dir" 2>/dev/null)
    owner=$(stat -c '%U' "$dir" 2>/dev/null)
    severity="LOW"
    for critical in /etc /bin /sbin /usr/bin /usr/sbin /opt /var/www; do
      if [ "$dir" = "$critical" ] || [ "${dir#$critical}" != "$dir" ]; then
        severity="HIGH"
        break
      fi
    done
    echo "$severity|$dir|$perms|$owner"
  done
}
```
</details>

<details>
<summary>Hint 5: Proc exposure check</summary>

```bash
check_proc_mount() {
  local mount_opts
  mount_opts=$(mount | grep ' /proc ' | awk '{print $6}')
  if echo "$mount_opts" | grep -q 'hidepid=2'; then
    echo "PROTECTED: /proc mounted with hidepid=2"
  elif echo "$mount_opts" | grep -q 'hidepid=1'; then
    echo "PARTIAL: /proc mounted with hidepid=1 (visible to same group)"
  else
    echo "EXPOSED: /proc mounted without hidepid"
  fi
}
```
</details>

<details>
<summary>Hint 6: Atomic write function</summary>

```bash
atomic_write() {
  local target="$1" content="$2"
  local dir=$(dirname "$target")
  local tmp=$(mktemp "$dir/.atomic.XXXXXX")
  echo "$content" > "$tmp"
  # Preserve original perms
  if [ -f "$target" ]; then
    chmod --reference="$target" "$tmp"
    chown --reference="$target" "$tmp"
  fi
  mv "$tmp" "$target"
}
```
</details>

<details>
<summary>Hint 7: Race simulation for testing</summary>

```bash
# Simulate an attacker racing a vulnerable script
attacker_loop() {
  local target="$1" symlink_target="$2"
  while true; do
    # Replace target with symlink to sensitive file
    rm -f "$target"
    ln -s "$symlink_target" "$target"
  done
}

# Run attacker in background
attacker_loop "/tmp/process.lock" "/etc/shadow" &
attacker_pid=$!

# Run vulnerable script
./vulnerable_script.sh

# Check if the attack succeeded
if [ -f "/etc/shadow.bak" ]; then
  echo "RACE WON: /etc/shadow was overwritten!"
fi
kill $attacker_pid
```
</details>

## Expected Output

```bash
$ ./toctou_detector.sh --scan /usr/local/bin --output text

=== TOCTOU Detector v1.0 ===
Scanning: /usr/local/bin
Files found: 23

=== Findings ===
[SECURE] /usr/local/bin/safe_process.sh — no TOCTOU patterns found

[HIGH] /usr/local/bin/process.sh
  Line 3:   LOCKFILE=/tmp/process.lock     — Hardcoded path in /tmp
  Line 5:   [ ! -f "$LOCKFILE" ] && ...    — Check-then-use race
  Line 6:   echo "$$" > "$LOCKFILE"        — PID-based temp (predictable)
  Line 19:  rm -f "$LOCKFILE"              — No trap for cleanup
  Score: 78/100 (HIGH)

[MEDIUM] /usr/local/bin/backup.sh
  Line 8:   TEMPFILE=/tmp/backup.$$.tar    — PID-based temp (predictable)
  Line 12:  cp "$file" "$TEMPFILE"         — No TOCTOU check on input
  Score: 45/100 (MEDIUM)

[LOW] /usr/local/bin/info.sh
  Line 15:  trap 'rm -f /tmp/tmp.$$' EXIT  — Hardcoded temp in trap
  Score: 20/100 (LOW)

=== Summary ===
🔴 HIGH:   1 (process.sh — immediate action required)
🟡 MEDIUM: 1 (backup.sh — review and fix)
🔵 LOW:    1 (info.sh — minor issue)
✅ Secure: 20

$ ./sticky_bit_checker.sh --audit --fix-interactive
=== Sticky Bit Auditor ===

🔴 HIGH /opt/staging (perms 777, owner root) — world-writable, NO sticky bit
   Fix? [Y/n] y
   ✅ Sticky bit set on /opt/staging

🟡 MEDIUM /var/www/shared (perms 775, owner www-data)
   — group-writable directory without sticky bit (lower risk)

🔵 LOW /mnt/nfs_share (perms 777, owner nobody)
   — world-writable, no sticky bit (NFS — sticky bit ineffective anyway)

✅ /tmp — sticky bit set (1777)
✅ /var/tmp — sticky bit set (1777)

=== Summary ===
Fixed: 1 directory
Remaining: 1 MEDIUM, 1 LOW

$ ./secure_rewrite.sh --test
=== Testing Fixed Script ===
✅ set -euo pipefail — present
✅ Temp files use mktemp — 2/2 confirmed
✅ Trap registered for cleanup — present
✅ Flock used instead of PID file — confirmed
✅ No hardcoded paths — 0 found
✅ Race-free lock mechanism — confirmed
✅ Handles SIGTERM/SIGINT — trap registered

All 7/7 checks passed. Script is race-condition-free.

$ ./proc_exposure_scan.sh
=== /proc Exposure Scanner ===
🔴 /proc mounted WITHOUT hidepid — all users can see process info
🔴 World-readable /proc/1234/environ — contains AWS_SECRET_KEY=...
🔴 Process 1234 has FD 5 -> /etc/shadow (world-readable)
🟡 Process 5678 has FD 3 -> /home/user/.ssh/id_rsa (limited read)
ℹ️  Total: 3 sensitive exposures found

Rating: HIGH — immediate action recommended
Recommendation: mount /proc with hidepid=2,invisible
```

## Self-Check Questions

1. Why is `mktemp` safer than `$$` for temp filenames? What makes `$$` predictable?

2. What is the exact race window between `[ -f /tmp/file ]` and `> /tmp/file`? How does an attacker exploit this window?

3. How does the sticky bit prevent file deletion attacks in `/tmp`? What doesn't it prevent?

4. Why is `mktemp -u` dangerous? When would it be acceptable to use it?

5. What does `trap 'rm -f "$tmp"' EXIT` guarantee? What if the script receives SIGKILL?

6. How does `flock` provide better locking than PID-based lock files? What happens if the locking process crashes?

7. Why is `/proc` exposure a security concern? What does `hidepid=2` do?

8. How does `mv` for atomic file updates work, and what are its limitations across filesystems?
