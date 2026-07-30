# Task 7: `find` Deep Dive

## Overview

Write a system-audit script `audit.sh` that uses `find` for five different search tasks. This script demonstrates practical system administration with `find`.

## Sub-tasks

### 1. Find Large Files

Find all files under a target directory larger than 100 MB. Print size and path.

```bash
#!/bin/bash
TARGET="${1:-/}"
echo "=== Large Files (>100MB) ==="
find "$TARGET" -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```

**Step-by-step:** `-type f` limits to regular files. `-size +100M` selects files larger than 100 MB. `-exec ls -lh {} \;` shows detailed info. `2>/dev/null` suppresses permission errors.

<details>
<summary>Hint: Exclude /proc, /sys, /dev for speed</summary>
```bash
find "$TARGET" -path /proc -prune -o -path /sys -prune -o -type f -size +100M -exec ls -lh {} \; 2>/dev/null
```
The `-prune` prevents descending into virtual filesystems.
</details>

### 2. Find SUID/SGID Binaries

Find all files in `/usr` with SUID or SGID permissions set. Log them to `suid_audit.log`.

```bash
echo "=== SUID/SGID Binaries ==="
count=$(find /usr -type f \( -perm /4000 -o -perm /2000 \) -ls 2>/dev/null | tee suid_audit.log | wc -l)
echo "Logged $count binaries to suid_audit.log"
```

**Alternative — separate SUID and SGID:**
```bash
echo "=== SUID Binaries ==="
find /usr -type f -perm /4000 -ls 2>/dev/null | tee suid_audit.log
echo "=== SGID Binaries ==="
find /usr -type f -perm /2000 -ls 2>/dev/null | tee -a suid_audit.log
```

<details>
<summary>Hint: -perm /4000 vs -perm -4000</summary>
`-perm /4000` = any SUID bit set. `-perm -4000` = all SUID bits set (only 4000 is one bit, so same here). But for `-perm -644`, it means "at least rw-r--r--" (all those bits must be set).
</details>

### 3. Find Recently Modified Configs

Find files under `/etc` modified in the last 7 days. Show mtime and path.

```bash
echo "=== Recently Modified Configs ==="
find /etc -type f -mtime -7 -exec ls -ld {} \; 2>/dev/null
```

**Better output with custom format (GNU find):**
```bash
find /etc -type f -mtime -7 -printf "%Td days ago: %p\n" 2>/dev/null
```

<details>
<summary>Hint: Using mtime vs mmin</summary>
`-mtime -7` = modified less than 7×24 hours ago. `-mmin -10080` = same but in minutes. `-mtime 7` = exactly 7 days ago (between 7 and 8 days). `-mtime +7` = more than 7 days ago.
</details>

### 4. Prune .git and Find Shell Scripts

Search for all `*.sh` files but exclude `.git` directories.

```bash
echo "=== Shell Scripts (excluding .git) ==="
find . -type d -name ".git" -prune -o -type f -name "*.sh" -print
```

**Alternative — prune multiple directories:**
```bash
find . \( -name ".git" -o -name "node_modules" -o -name "__pycache__" \) -prune \
    -o -type f -name "*.sh" -print
```

**Memory aid:** Think of the pattern as "If it's a .git dir, prune (and skip). Otherwise (-o), find .sh files and print."

<details>
<summary>Hint: Testing prune behavior</summary>
```bash
# See what gets pruned
find . -type d -name ".git" -prune -print
# The -print here shows you which directories are being pruned
```
</details>

### 5. Batch Permission Fix

Find all `.sh` files and ensure they're executable. Find all `.md` files and set to 644.

```bash
echo "=== Permission Fix ==="
sh_count=$(find . -type f -name "*.sh" -exec chmod +x {} + -print | wc -l)
md_count=$(find . -type f -name "*.md" -exec chmod 644 {} + -print | wc -l)
echo "Fixed $sh_count .sh files — made executable"
echo "Fixed $md_count .md files — set to 644"
```

**Alternative with progress:**
```bash
find . -type f -name "*.sh" -print -exec chmod +x {} \;
```

### 6. Bonus: Find by Depth

```bash
# Files modified today, only 2 levels deep
find . -maxdepth 2 -type f -mtime 0

# Files at least 3 levels deep
find . -mindepth 3 -type f -name "*.log"
```

### 7. Bonus: Find Orphaned Files

```bash
echo "=== Orphaned Files (no user/group) ==="
find / -nouser -o -nogroup -ls 2>/dev/null
```

### 8. Bonus: Find Duplicate Files by Size

```bash
echo "=== Potential Duplicates (same size) ==="
find . -type f -printf "%s\t%p\n" | sort -k1,1n | uniq -D -w8 -f1
```

## Expected Output

```
$ ./audit.sh /
=== Large Files (>100MB) ===
-rw-r--r-- 1 root root 150M Jul 28 /var/log/syslog
-rw-r--r-- 1 root root 120M Jul 27 /var/cache/pkg/pkg.tar.gz

=== SUID/SGID Binaries ===
Logged 12 binaries to suid_audit.log

=== Recently Modified Configs ===
-rw-r--r-- 1 root root 1234 Jul 28 /etc/apt/sources.list
-rw-r--r-- 1 root root 5678 Jul 29 /etc/ssh/sshd_config

=== Shell Scripts (excluding .git) ===
./scripts/deploy.sh
./setup.sh
./tests/test_suite.sh

=== Permission Fix ===
Fixed 3 .sh files — made executable
Fixed 5 .md files — set to 644
```

## Self-Check

- What's the difference between `-exec {} \;` and `-exec {} +`?
- Why does `-delete` imply `-depth`? What problem does this solve?
- How do you safely pipe `find` output to `xargs`?
- What does `-prune` do, and how is it combined with `-o`?
- How would you find all empty files in a directory tree?
- What's the difference between `-perm /4000` and `-perm -4000`?
- How do you exclude multiple directories from find?

### Sub-task 6: find + exec for bulk rename

Write a one-liner (or script) using `find -exec` that renames all `.txt` files in a directory tree to `.bak` by appending `.bak` to the existing name (e.g., `notes.txt` → `notes.txt.bak`). Test it.

<details>
<summary>Hint 1</summary>

Use `find . -name "*.txt" -exec sh -c 'mv "$1" "$1.bak"' _ {} \;
</details>

### Sub-task 7: Time-based differential backup

Write a script that:
1. Creates a timestamp file
2. Runs a command (simulate with `touch`)
3. Uses `find -newer` to list all files modified after the timestamp
4. Archives them with `tar`

<details>
<summary>Hint 1</summary>

Use `touch /tmp/timestamp` to create the reference point.
</details>

<details>
<summary>Hint 2</summary>

`find . -newer /tmp/timestamp -type f -print0 | xargs -0 tar -czf backup.tar.gz`
</details>
