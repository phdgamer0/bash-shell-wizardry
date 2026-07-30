# Lesson 12: Exploitation — File System Attacks (DEFENSE FOCUSED)

## History & Origins

File system attacks are older than Unix itself. The concept of a "race condition" — where the timing gap between two operations can be exploited — was first described in the context of time-sharing systems in the 1960s. As soon as computers had concurrent processes accessing shared resources, attackers found ways to slip in between.

The specific class of TOCTOU (Time-of-Check, Time-of-Use) bugs earned its name in 1974 when researchers at Bell Labs noticed that checking a file's permissions and then opening it could be subverted if the file changed between the two operations. Since then, TOCTOU has appeared in everything from the original `sendmail` debug mode exploit to modern container escape tools.

Symlink attacks emerged alongside world-writable directories like `/tmp`. In 1984, the first documented symlink attack on `/tmp` allowed local users to overwrite root-owned files. This pattern — create a file in `/tmp` that's actually a symlink to `/etc/shadow` — has been rediscovered by pentesters and malware authors for four decades.

The `/proc` filesystem (introduced in Linux 1.0, 1994) opened an entirely new attack surface. `/proc/PID/fd/` exposes open file descriptors, `/proc/PID/root` links to the process's root directory, and `/proc/PID/cwd` shows the current working directory. These pseudo-files have enabled countless privilege escalation and information disclosure attacks.

Container technology (2000s-2010s) didn't eliminate these problems — it amplified them. Misconfigured container mounts, shared `/tmp` volumes, and `/proc` access inside containers created new TOCTOU and symlink race opportunities, now with the added complexity of namespace boundaries.

## Syntax Reference

```
# Secure temp file creation
mktemp /tmp/template.XXXXXX         # Create temp file with random suffix
mktemp -d /tmp/tempdir.XXXXXX        # Create temp directory
mktemp -u /tmp/template.XXXXXX       # DANGEROUS: generate name without creating file
tempfile=$(mktemp)                    # With no template, uses /tmp/tmp.XXXXXXXXXX

# File test operators (CHECK operations)
[ -f "$file" ]      # True if file exists (regular file)
[ -e "$path" ]      # True if path exists (any type)
[ -L "$path" ]      # True if path is a symlink
[ -w "$path" ]      # True if writable by current user
[ -r "$path" ]      # True if readable
[ -O "$path" ]      # True if owned by current user

# File descriptor operations
exec 3> "$file"     # Open FD 3 for writing (TRUNCATE mode)
exec 4>> "$file"    # Open FD 4 for appending
exec 5< "$file"     # Open FD 5 for reading
exec 3>&-           # Close FD 3
flock -n 3          # Non-blocking lock on FD 3 (flock utility)

# Permissions and attributes
stat -c '%a' file   # Get permissions in octal (e.g., 644)
stat -c '%u %g' file # Get UID and GID
chmod +t dir        # Set sticky bit on directory
getfacl file        # Get ACL entries
lsattr file         # Show file attributes (immutable, append-only)

# Process-related file access
ls -la /proc/PID/fd/     # List open file descriptors
ls -la /proc/PID/root/   # Symlink to process's root directory
ls -la /proc/PID/cwd/    # Symlink to process's cwd
readlink /proc/PID/exe   # Path to the executable

# Cleanup and safety
trap 'rm -f "$tmpfile"' EXIT   # Remove temp file on script exit
trap 'rm -rf "$tmpdir"' EXIT   # Remove temp directory on exit
set -euo pipefail               # Fail-safe mode
```

## Under the Hood (GO DEEP)

### The TOCTOU Race Window — Precise Mechanics

TOCTOU vulnerabilities exploit the gap between TWO system calls — the CHECK and the USE. Here's what happens at the kernel level:

**Vulnerable pattern:**
```
[CHECK]  access("/tmp/lock", F_OK)  →  returns -1 (file not found)
         ╰── RACE WINDOW ──╯         ← attacker creates symlink here
[USE]    open("/tmp/lock", O_CREAT|O_WRONLY)  →  follows symlink to /etc/shadow
```

The race window is the time between the `access()` syscall returning and the `open()` syscall executing. In a single-threaded script, this window is small but measurable:
- Context switch to another process: ~100 microseconds
- Attacker process running in parallel: the window repeats every scheduler timeslice (~4ms on a typical kernel)

With 100 concurrent attacker processes each trying to create the symlink, the race almost always succeeds within seconds. This is called a "symlink race."

**Secure pattern:**
```
open("/tmp/temp.XXXXXX", O_CREAT|O_EXCL|O_RDONLY)  →  atomic create-or-fail
```

The `O_CREAT | O_EXCL` flags tell the kernel: "create this file ONLY if it doesn't exist, and fail if it does." This is atomic — no race window exists because the check and create are a single syscall.

### Kernel-Level File Operations

**File Descriptor Internals:**

When a process opens a file, the kernel:
1. Traverses the pathname, resolving symlinks along the way
2. Performs permission checks against the process's UID/GID/capabilities
3. Allocates a file descriptor (small integer) pointing to the kernel's file struct
4. The file struct contains: the current file position, the inode pointer, the open flags (O_RDONLY, O_WRONLY, etc.), and the refcount

The critical thing: once a file descriptor is opened, it points to a specific inode. Even if the original path is deleted or replaced, the FD stays valid for the open file. This is why holding an open FD is safe — the race has already passed.

**Symlink Resolution:**

When the kernel encounters a symlink during path resolution:
1. It reads the symlink target (stored in the inode's data blocks)
2. It continues path resolution starting from the target path
3. This is recursive — symlinks can point to symlinks (with a limit of `MAXSYMLINKS` = 40 on Linux)

Symlinks in world-writable directories are dangerous because:
- The symlink itself is owned by the attacker
- The target can be anywhere the attacker chooses
- The symlink can be swapped after the check passes

### Trapdoor: The `/proc` Filesystem

`/proc` is a pseudo-filesystem that exposes kernel data structures as files. The key attack vectors:

**`/proc/PID/fd/`:** Lists all open file descriptors. If a root process has an FD open to `/etc/shadow`, any user can read it (if permissions allow — modern kernels restrict this).

**`/proc/PID/root/`:** Symlink to the process's root directory. If the process is inside a chroot, this points to the chroot root. But if you have `CAP_SYS_PTRACE` (or are root and ptrace is unrestricted), you can follow this to escape the chroot.

**`/proc/PID/cwd/`:** Symlink to the process's current working directory. If a process is in a directory you can't normally access, reading `/proc/PID/cwd/` bypasses the restriction.

**`/proc/self/`:** Convenience symlink to the current process's `/proc` directory.

**`/proc/1/root/`** : If accessible, shows the host filesystem from inside a container. Container runtimes restrict this via user namespaces and `--pid=host`.

### Sticky Bit on Directories

The sticky bit (`chmod +t`) on directories changes deletion semantics:
- **Without sticky bit:** Any user with write+execute permission on a directory can delete or rename any file inside it, regardless of file ownership.
- **With sticky bit:** Only the file owner, directory owner, or root can delete or rename files inside the directory.

`/tmp` has the sticky bit set (`drwxrwxrwt`). If it didn't, any user could delete any other user's temp files. This is the standard defense against temp file attacks.

**But the sticky bit does NOT protect against:**
- Opening files (anyone can read/write world-readable/writable files in `/tmp`)
- Symlink races (the symlink is created by the attacker, not deleted)
- File content modification (if the file is world-writable, anyone can truncate or write to it)

### chroot and Container Filesystem Isolation

**chroot:** Changes the root directory for a process and its children. It:
- Does NOT change the working directory (chdir outside the new root is possible after chroot)
- Does NOT close open file descriptors (Fds to outside the chroot remain open)
- Does NOT drop privileges (root inside chroot is still root)
- Does NOT isolate /proc (unless mounted separately)
- Is NOT a security mechanism — it's a filesystem restriction tool

**chroot escape via /proc:**
```bash
# Inside chroot as root:
chroot /path/to/newroot
# If /proc is mounted:
mkdir -p /newroot
mount --bind /proc/1/root /newroot
chroot /newroot
# Now you're in the host root!
```

### What strace Reveals

```bash
# Vulnerable TOCTOU pattern:
$ strace -f bash -c 'if [ -f /tmp/lock ]; then echo locked; fi'
stat("/tmp/lock", {st_mode=...}) = 0    # CHECK
write(1, "locked\n", 7)                 # USE (separate syscall)

# Secure atomic create:
$ strace -f bash -c 'mktemp /tmp/test.XXXXXX'
open("/tmp/test.", O_RDWR|O_CREAT|O_EXCL, 0600) = 3
# Note: ONE syscall, atomic. O_EXCL ensures the file didn't exist.

# Sticky bit on /tmp:
$ stat /tmp 2>&1 | head -2
  File: /tmp
  Size: 4096        Blocks: 8          IO Block: 4096   directory
Device: 801h/2049d   Inode: 1234567     Links: 11
Access: (1777/drwxrwxrwt)  # <-- the 't' at the end is the sticky bit
# 1777 = 1 (sticky) + 777 (rwx for all)
```

### Security Model Interactions

**Capabilities and TOCTOU:**
- `CAP_DAC_OVERRIDE`: Bypass permission checks — a process with this capability can write to any file, regardless of the TOCTOU protection
- `CAP_DAC_READ_SEARCH`: Bypass file read checks
- `CAP_SYS_PTRACE`: Allows ptracing any process — enables `/proc` attacks

**SELinux and File Operations:**
- SELinux type enforcement applies to BOTH the check AND the use
- A process confined to type `httpd_t` cannot write to files of type `shadow_t`
- This provides a second layer: even if a TOCTOU race succeeds, SELinux may block the final operation

**User Namespaces:**
- Inside a user namespace, the process appears to be root but lacks real `CAP_SYS_ADMIN`
- Mount operations that require `CAP_SYS_ADMIN` in the initial namespace may still work inside the user namespace
- This enables container escape TOCTOUs that target host filesystem mounts

## Core Examples (15 total)

### Example 1: TOCTOU — The classic PID file vulnerability

```bash
# VULNERABLE — never use this pattern
if [ ! -f /tmp/process.lock ]; then
  echo "$$" > /tmp/process.lock
  # ... critical section ...
  rm -f /tmp/process.lock
fi
```

**Anatomy:** The `[ ! -f /tmp/process.lock ]` check reads the file status. Between that check and the `echo "$$" > /tmp/process.lock` write, an attacker can create a symlink at `/tmp/process.lock` pointing to `/etc/shadow`. The redirect `>` then overwrites `/etc/shadow`.

**Variations:** PID files, socket files, lock files, temp db files — any file in a world-writable directory is vulnerable to this pattern.

**Edge case:** Even if you use `[ -f ]` AND check with `[ -L ]` separately, the race still exists between the checks and the use. Only atomic operations (O_EXCL, mktemp) are safe.

### Example 2: Secure temp file creation with mktemp

```bash
tmpfile=$(mktemp /tmp/myscript.XXXXXX)
trap 'rm -f "$tmpfile"' EXIT
echo "data" > "$tmpfile"
```

**Anatomy:** `mktemp` generates a random suffix (replacing XXXXXX) and creates the file atomically with `O_CREAT|O_EXCL`. The `trap` ensures cleanup even if the script is killed or errors.

**Variations:** `mktemp -d` creates a directory instead of a file. Without a template, uses `TMPDIR` or `/tmp`. `tempfile` is the Debian-specific variant (same concept).

**Edge case:** `mktemp -u` generates a name WITHOUT creating the file. Using this is just as dangerous as hardcoding a temp name — the race window exists between the generation and your use.

### Example 3: Secure temp directory with cleanup

```bash
tmpdir=$(mktemp -d /tmp/myscript.XXXXXX)
trap 'rm -rf "$tmpdir"' EXIT
cp "$1" "$tmpdir/input"
process "$tmpdir/input" > "$tmpdir/output"
mv "$tmpdir/output" "/final/$(date +%s).out"
```

**Anatomy:** Processing files in a private temp directory with restrictive permissions (by default, `mktemp -d` creates with 700). The `trap` cleans up on any exit path.

**Edge case:** `$tmpdir` could be very long if `TMPDIR` is set to a path with long components. Always quote variable expansions.

### Example 4: Symlink race detection function

```bash
check_symlink_race() {
  local path="$1"
  if [ -L "$path" ]; then
    echo "WARNING: $path is a symlink -> $(readlink "$path")"
    return 1
  fi
  # Double-check: the file might have been replaced between -L check and readlink
  if [ -L "$path" ]; then
    echo "CRITICAL: Symlink race in progress on $path"
    return 2
  fi
  return 0
}
```

**Anatomy:** Double-checking reduces the race window but doesn't eliminate it. This is a best-effort detection function, not a prevention function.

**Edge case:** An attacker can still race between the first `-L` check and the second one. Only atomic operations truly prevent this.

### Example 5: Sticky bit verification

```bash
check_sticky_bit() {
  local dir="$1"
  local perms=$(stat -c '%a' "$dir" 2>/dev/null)
  # Check if world-writable (any of the last 3 octal digits are 2,3,6,7)
  if [ "${perms: -1}" -ge 2 ] || [ "${perms: -2:1}" -ge 2 ]; then
    local sticky=$(stat -c '%t' "$dir" 2>/dev/null)
    # stat -c '%t' gives type (not sticky info) — use '%a' and check for 't'
    local ls_output=$(ls -ld "$dir" 2>/dev/null)
    if ! echo "$ls_output" | grep -q 't$' && ! echo "$ls_output" | grep -q 't '; then
      echo "WARNING: $dir is world-writable WITHOUT sticky bit!"
    fi
  fi
}
```

**Anatomy:** The sticky bit is shown as 't' in the last position of the permission string (`drwxrwxrwt`). Without it, world-writable directories are wide open.

**Edge case:** Some filesystems don't support the sticky bit (FAT, NTFS, FUSE). Directories on these filesystems are always vulnerable.

### Example 6: Race-safe lock file using flock

```bash
lockfile=/tmp/myscript.lock
exec 3>"$lockfile"
if ! flock -n 3; then
  echo "Another instance is running"
  exit 1
fi
# Critical section — the lock is held on FD 3
echo "Running exclusive operation..."
flock -u 3  # Release
```

**Anatomy:** `flock` uses the kernel's file locking mechanism. The FD 3 is opened on the lock file. `flock -n 3` tries to acquire a write lock, failing immediately if another process holds it. The lock is automatically released when FD 3 closes (or on process exit).

**Variations:** `flock -s` for shared (read) locks. `flock -x` for exclusive (write) locks. Without `-n`, it blocks until the lock is available.

**Edge case:** The lock file itself must be created securely. Opening with `exec 3>` creates/truncates the file, but that's fine — the lock is on the FD, not the file content.

### Example 7: chroot detection and prevention

```bash
# Check if we're in a chroot
is_chroot() {
  if ! grep -q '/' /proc/1/mountinfo 2>/dev/null; then
    # /proc/1/mountinfo doesn't show '/' — we're in a chroot
    return 0
  fi
  # Compare /proc/1/root with our own root
  if [ "$(stat -c '%d:%i' /)" != "$(stat -c '%d:%i' /proc/1/root/ 2>/dev/null)" ]; then
    return 0  # Different inode — we're chrooted
  fi
  return 1
}
```

**Anatomy:** A chroot changes the process's root directory. Comparing inode numbers of `/` and `/proc/1/root/` reveals the difference.

**Edge case:** This check itself accesses `/proc` — which the attacker may have already manipulated. Defense in depth: don't run untrusted code as root, even in a chroot.

### Example 8: Secure script with set -euo pipefail

```bash
#!/bin/bash
set -euo pipefail

safe_temp() {
  local tmp
  tmp=$(mktemp "$1.XXXXXX") || { echo "mktemp failed"; exit 1; }
  trap 'rm -f "$tmp"' RETURN
  echo "$tmp"
}

process_file() {
  local input="$1"
  local tmp=$(safe_temp "process")
  cat "$input" > "$tmp"
  echo "Processed: $tmp"
}

process_file "$1"
```

**Anatomy:** `set -euo pipefail` catches potential failures:
- `-e`: Exit on any command failure (non-zero exit)
- `-u`: Treat unset variables as errors
- `-o pipefail`: Fail if any command in a pipeline fails (not just the last)

**Edge case:** `set -e` can be surprising — it catches failures in conditions (`if`, `while`, `||`, `&&`). Use `|| true` for expected failures.

### Example 9: Inode-based file identity check

```bash
# Check if two paths point to the SAME file
same_file() {
  local f1="$1" f2="$2"
  local info1=$(stat -c '%d:%i' "$f1" 2>/dev/null)
  local info2=$(stat -c '%d:%i' "$f2" 2>/dev/null)
  [ "$info1" = "$info2" ]
}

# Example: verify the file hasn't been swapped
check_identity() {
  local path="$1" expected_dev_ino="$2"
  local current=$(stat -c '%d:%i' "$path" 2>/dev/null)
  [ "$current" = "$expected_dev_ino" ]
}
```

**Anatomy:** Each file has a unique (device, inode) pair. If the path was a symlink that was swapped, the inode changes. This is a stronger check than `-f` because symlinks have different inodes than their targets.

**Edge case:** An attacker can still race between the inode check and the file use. The inode check is just another CHECK — the USE still happens later.

### Example 10: /proc/self/fd/ exposure detection

```bash
# Check what file descriptors are exposed to other users
check_fd_exposure() {
  local pid=$$
  for fd in /proc/$pid/fd/*; do
    local target=$(readlink "$fd" 2>/dev/null)
    local perms=$(stat -c '%a' "$fd" 2>/dev/null)
    if [ -n "$target" ] && [ "$target" != "$fd" ]; then
      echo "FD $(basename $fd) -> $target (perms $perms)"
    fi
  done
}
```

**Anatomy:** `/proc/PID/fd/` lists all open file descriptors. In restrictive systems, only the owning user can read this. But if `/proc` is mounted with default options, any user can see which files a process has open.

**Variations:** `/proc/PID/fdinfo/` contains additional info like file position and flags.

**Edge case:** Attackers can also read `/proc/PID/environ` to get environment variables (which may contain secrets like API keys, passwords). Mount `/proc` with `hidepid=2` on multi-user systems.

### Example 11: World-writable directory finder

```bash
find / -type d -perm -0002 ! -perm -1000 2>/dev/null
```

**Anatomy:** `-perm -0002` matches directories where the "other write" bit is set. `! -perm -1000` excludes those with the sticky bit. Output is paths of dangerous directories.

**Variations:** Check specific critical locations: `/etc`, `/bin`, `/usr/bin`, `/opt`, `/var/www`. World-writable permissions on any of these is a severe issue.

**Edge case:** Some directories legitimately need to be world-writable (like `/tmp`). The sticky bit is what makes them safe. Without it, they're a security hole.

### Example 12: Race-safe data processing pipeline

```bash
#!/bin/bash
set -euo pipefail

INPUT="$1"
WORKDIR=$(mktemp -d)
trap 'rm -rf "$WORKDIR"' EXIT

# Copy input to safe directory (atomic)
cp "$INPUT" "$WORKDIR/input"

# Process in WORKDIR — no races possible
grep 'ERROR' "$WORKDIR/input" > "$WORKDIR/errors"
awk '{sum+=$NF} END {print sum}' "$WORKDIR/input" > "$WORKDIR/total"

# Move output to final location
mv "$WORKDIR/errors" "/output/errors.$(date +%s).txt"
mv "$WORKDIR/total" "/output/total.$(date +%s).txt"
```

**Anatomy:** All processing happens in `$WORKDIR` (created with `mktemp -d`, permissions 700). No other process can access this directory. The `mv` to output is not atomic across filesystems but is atomic within the same filesystem (rename syscall).

**Edge case:** If `INPUT` is read from a world-writable location, the `cp` itself could be subject to a TOCTOU race. Combine with mktemp at the first read point.

### Example 13: abusing the .lockfile race for privilege escalation

```bash
# VULNERABLE setuid script pattern:
#!/bin/bash
# This is a setuid-root script
LOCKFILE=/tmp/$(whoami).lock

if [ ! -f "$LOCKFILE" ]; then
  echo "$$" > "$LOCKFILE"
  # ... privileged operation ...
  rm -f "$LOCKFILE"
fi
```

**Anatomy:** A setuid script that uses a predictable temp path based on `$(whoami)`. An attacker creates a symlink `ln -s /etc/sudoers /tmp/attacker.lock` BEFORE the script runs (or during the race window). The script then overwrites `/etc/sudoers` with `"$$"` (a small number), potentially corrupting it or adding backdoor access.

**Edge case:** Setuid SHELL SCRIPTS are inherently dangerous on most Unix systems. Many modern systems ignore the setuid bit on interpreted scripts for this exact reason.

### Example 14: Directory traversal via symlink in temp

```bash
# VULNERNABLE temp handler:
TMPFILE=/tmp/backup.$$
tar cf "$TMPFILE" /home/user
# If attacker creates /tmp/backup.$$ -> /etc/shadow before tar runs
# tar follows the symlink and clobbers /etc/shadow
```

**Anatomy:** The PID-based temp filename is predictable. An attacker can pre-create a symlink with that name pointing to a sensitive file.

**Variations:** Any archiving or backup script that constructs temp filenames from PIDs, timestamps, or other predictable values is vulnerable.

**Edge case:** Even with random names, if the file creation isn't using O_EXCL, the race still exists. Always use `mktemp`.

### Example 15: Safe FD-based operations

```bash
# SECURE: Open the file once, keep the FD
exec 3< "/var/log/syslog"
# Read line by line from FD 3
while IFS= read -r line <&3; do
  process "$line"
done
exec 3<&-  # Close when done
```

**Anatomy:** Opening the file ONCE and keeping the FD eliminates TOCTOU for that file. The FD points to the specific inode at the time of opening. Even if the original path is renamed, replaced, or deleted, the FD remains valid.

**Variations:** Same approach for writing: `exec 4>> /var/log/app.log` ensures you're appending to the original file, not to a replacement.

**Edge case:** FDs are per-process. Forked child processes inherit parent FDs. This can accidentally leak access to sensitive files.

## Real-World Use Cases

### FOR the OS

- **mktemp** in system maintenance scripts (log processing, temp file creation)
- **flock** for cron job serialization (preventing overlapping runs)
- **chmod +t** on `/tmp`, `/var/tmp`, shared directories
- **Kernel O_EXCL** flag in filesystem operations

### WITH the OS

- **Lock files** for database connections, web server pid files
- **Temp files** for sorting large datasets (sort uses `/tmp`)
- **Atomic renames** for configuration file updates (write new, rename old)
- **Flock-based mutex** for multi-process coordination

### AGAINST the OS

- **Symlink races** against setuid scripts and temp file handlers
- **TOCTOU on /proc** to read other processes' file descriptors
- **chroot escapes** via `/proc/1/root` or leftover FDs
- **temp file hijacking** to overwrite sensitive files
- **PID file race** to trick monitoring systems

### FOR DEFENSE

- **Always use mktemp** — never construct temp paths manually
- **Set sticky bit** on world-writable directories
- **Set file creation mask (umask)** to restrict permissions
- **Use O_EXCL or flock** for atomic operations
- **Mount /proc with hidepid** on multi-user systems
- **Don't use setuid shell scripts** — use a compiled wrapper or capabilities
- **Set immutable bit** on critical files with `chattr +i`

## Memory Aids

**"CHECK THEN USE = YOU LOSE"** — The fundamental TOCTOU vulnerability. If you check a file's existence/permissions and then separately use it, there's a race window.

**"O_EXCL IS YOUR FRIEND"** — The open-with-exclusive flag on mktemp is the standard defense: create file ONLY if it doesn't exist, atomically.

**"FD ONCE, USE FOREVER"** — Open the file once and keep the file descriptor. FDs are immune to path-based races.

**"STICKY /tmp SAVES SCRIPTS"** — The sticky bit on /tmp prevents users from deleting each other's temp files.

**"proc ROOT is the escape route"** — chroot escapes almost always involve /proc/PID/root. Mount /proc with hidepid.

## Trap Vault (15 traps)

**Trap 1:** `mktemp -u` generates a random name but DOES NOT create the file. The race window between the generation and your use of the name is wide open. Never use `-u` for security purposes.

**Trap 2:** `[ -f "$file" ] && [ ! -L "$file" ]` is STILL vulnerable to TOCTOU. Both checks happen separately from the USE. Between the last check and the file access, the file can be replaced.

**Trap 3:** The race window is measured in MICROSECONDS but attackers use thousands of parallel processes to win the race. With 1000 attackers all racing to create symlinks, the window collapses to milliseconds.

**Trap 4:** `cp` followed by `chmod` on the copy is a TOCTOU: the file content was written under one set of permissions, then changed. An attacker who can write to the destination directory can replace the file between the `cp` and the `chmod`.

**Trap 5:** `chroot` is NOT a security mechanism. A root process inside a chroot can escape if it can see `/proc/1/root` or if any file descriptors were inherited from outside.

**Trap 6:** PID-based temp filenames (`/tmp/script.$$`) are predictable. `$$` is the shell's PID, which is easy to guess via `/proc/$$/stat`. Attackers can pre-create symlinks at predictable paths.

**Trap 7:** The sticky bit does NOT prevent file reading or writing — it only prevents deletion. A world-readable secret file in `/tmp` is still readable by anyone.

**Trap 8:** `ln -sf /etc/shadow /tmp/mylock` creates a NEW symlink even if one already exists. The attacker doesn't need to delete the lock file — they just update the symlink target.

**Trap 9:** `set -e` doesn't catch errors in conditions, like `if [ -f /tmp/lock ]; then`. Always use `set -euo pipefail` AND check for failures explicitly.

**Trap 10:** Named pipes (FIFOs) in `/tmp` can be exploited similarly to symlinks. An attacker creates a FIFO instead of a regular file, and the script blocks waiting for input.

**Trap 11:** `exec 3> "$file"` truncates the file before any lock is acquired. If another process had the file open, it loses its content. Use `exec 3>> "$file"` (append mode) for lock files.

**Trap 12:** Docker volumes mounted from the host create shared filesystem access. A privileged container can exploit TOCTOU races on the host filesystem via the mounted volume.

**Trap 13:** `umask` affects file creation permissions. A umask of `000` creates world-writable files. Always set `umask 077` in scripts that create temp files.

**Trap 14:** NFS does NOT properly support O_EXCL on all versions. On NFSv3, O_EXCL falls back to a non-atomic check-and-create. Use lock files or SSHFS instead.

**Trap 15:** `tmpfs` (memory-backed) filesystems like `/dev/shm` do not enforce the sticky bit as reliably as disk-based filesystems. Check the mount options.

## See It In The Wild

- **Local root exploit CVE-2018-14665 (Xorg):** A TOCTOU vulnerability in Xorg's `-modulepath` argument allowed local users to overwrite arbitrary files by racing a file check and a file write, gaining root on many distros.

- **Docker container escape (shocker exploit):** Achieved by accessing `/proc/1/root/` from inside a container that had the `CAP_DAC_READ_SEARCH` capability and `/proc` mounted. The exploit: `cat /proc/1/root/etc/shadow`.

- **Symlink attack on MIT-Kerberos:** A race condition in ksu (Kerberized superuser) allowed local users to overwrite any file on the system by racing a `lstat()` check with a `chmod()` call on a file in `/tmp`.

- **Sendmail debug mode (1988 Morris Worm):** The original worm exploited a TOCTOU in sendmail's debug mode, where sendmail would execute commands from an input file that changed between validation and execution.

- **Git clone TOCTOU (CVE-2014-9390):** Git for Windows handled symlinks in case-insensitive filesystems incorrectly, allowing repository clones that overwrite `.git/config` via a TOCTOU in the checkout logic.

## Check Your Understanding (10 questions)

1. **Q:** What precisely is the TOCTOU race window, and what two system calls are involved? **A:** The window between a CHECK syscall (like `access()` or `stat()`) and a USE syscall (like `open()` or `write()`). An attacker changes the file between the two calls.

2. **Q:** Why does `mktemp` eliminate the TOCTOU problem for temp file creation? **A:** It uses `open()` with `O_CREAT|O_EXCL` flags, which atomically checks for existence AND creates the file in a single syscall. There's no race window.

3. **Q:** How does the sticky bit protect `/tmp`, and what doesn't it protect against? **A:** It prevents users from deleting or renaming files they don't own. It does NOT prevent reading, writing, or creating symlinks.

4. **Q:** Why is `mktemp -u` considered dangerous? **A:** It only generates a name without creating the file. An attacker can create a symlink at that path between the name generation and the file creation.

5. **Q:** How can a root process escape a chroot jail? Name two methods. **A:** (1) Mount `proc` and use `mount --bind /proc/1/root /newroot; chroot /newroot`. (2) Inherit a file descriptor to outside the chroot and use it to access the host filesystem.

6. **Q:** What does `set -euo pipefail` do, and how does it help with file system attack defense? **A:** `-e` exits on errors. `-u` errors on unset variables. `-o pipefail` catches pipeline failures. Together they prevent scripts from continuing after a race condition causes a command to fail.

7. **Q:** How does `flock` work, and why is it safer than PID-based lock files? **A:** `flock` uses the kernel's file locking mechanism on an open file descriptor. Unlike PID files, it's atomic and auto-released when the process dies.

8. **Q:** What is the difference between a symlink and a hard link in the context of TOCTOU attacks? **A:** Symlinks point to paths (can cross filesystems, can dangle). Hard links share the same inode (cannot cross filesystems, cannot be made to directories on Linux). Symlinks are the primary TOCTOU vector.

9. **Q:** Why is `/proc/1/root/` a security concern inside containers? **A:** It's a symlink to the host's root filesystem. If accessible from inside a container (requires appropriate capabilities), it allows reading/writing any file on the host.

10. **Q:** What is the recommended umask setting for scripts that handle temporary files, and why? **A:** `umask 077` ensures files are created with rwx------ (700) or rw------- (600) permissions. This prevents other users from reading or writing the temporary files.
