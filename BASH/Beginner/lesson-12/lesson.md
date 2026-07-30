# Lesson 12: File Tests

## History & Origins

File test operators are part of the Unix `test` command (`[`), which appeared in Unix V7 (1979). The original `test` had basic operators like `-r`, `-w`, `-x`, `-f`, `-d`. As Unix evolved, more operators were added: `-e` (exists, SysV), `-L` (symlink, 4.3BSD), `-N` (modified since read, SysV), `-O` (owned by EUID, POSIX), `-G` (group owned, POSIX).

The naming convention is simple: `-` followed by a letter that mnemonically represents the test. `-f` = "file" (regular file), `-d` = "directory", `-r` = "readable", `-w` = "writable", `-x` = "executable", `-s` = "size" (greater than zero), `-L` = "Link", `-N` = "New" (modified since read), `-O` = "Owned", `-G` = "Group".

The comparison operators `-nt` and `-ot` come from "newer than" and "older than." The `-ef` operator (same file, same inode) is less known but powerful for checking hard links.

These operators are built into bash's `test` builtin and `[[ ]]` keyword, but the external `/usr/bin/[` still exists for POSIX compliance. On modern systems, running `[` always uses the bash builtin unless you explicitly call `/usr/bin/[`.

Fun historical tidbit: The original Seventh Edition Unix `test` man page listed all operators alphabetically, which is why the order seems random. The `-a` (AND) and `-o` (OR) operators were added later and were always considered a mistake by the POSIX committee, who recommend using separate `[ ]` invocations with `&&`/`||` instead.

## Syntax Reference

### All file test operators
```bash
[ -a "$path" ]   # Path exists (deprecated, use -e)
[ -e "$path" ]   # Path exists (any file type)
[ -f "$path" ]   # Regular file exists
[ -d "$path" ]   # Directory exists
[ -h "$path" ]   # Path is a symbolic link (same as -L)
[ -L "$path" ]   # Path is a symbolic link
[ -b "$path" ]   # Block device (e.g., /dev/sda)
[ -c "$path" ]   # Character device (e.g., /dev/tty)
[ -p "$path" ]   # Named pipe (FIFO)
[ -S "$path" ]   # Socket (e.g., /var/run/docker.sock)
[ -t fd ]        # File descriptor is a terminal
[ -r "$path" ]   # Readable by current effective user ID
[ -w "$path" ]   # Writable by current effective user ID
[ -x "$path" ]   # Executable (file) or searchable (directory)
[ -s "$path" ]   # Size > 0
[ -N "$path" ]   # Modified since last read
[ -O "$path" ]   # Owned by current effective user ID
[ -G "$path" ]   # Group-owned by current effective group ID
[ -k "$path" ]   # Sticky bit is set
[ -u "$path" ]   # SUID (Set User ID) bit is set
[ -g "$path" ]   # SGID (Set Group ID) bit is set
```

### Comparison operators
```bash
[ "$p1" -nt "$p2" ]  # p1 newer than p2 (modification time)
[ "$p1" -ot "$p2" ]  # p1 older than p2
[ "$p1" -ef "$p2" ]  # Same file (same inode)
```

### Combining tests (POSIX style)
```bash
[ ! -e "$path" ]              # Negation
[ -r "$path" -a -w "$path" ]  # AND (both readable AND writable)
[ -r "$path" -o -w "$path" ]  # OR (readable OR writable)
```

### Combining tests (bash [[ ]] style)
```bash
[[ -f "$path" && -r "$path" ]]   # AND
[[ -d "$path" || -L "$path" ]]   # OR
[[ ! -e "$path" ]]               # Negation
```

### Finding test exit codes
```bash
test -f /etc/passwd; echo $?      # 0 if file exists
[ -f /etc/passwd ]; echo $?       # Same thing
[[ -f /etc/passwd ]]; echo $?     # Bash keyword
```

## Under the Hood

### What Happens When You Run `[ -f /etc/passwd ]`

1. **Shell parsing**: Bash parses the statement, identifies `[` as a builtin.
2. **Argument preparation**: Shell expands variables. `-f`, `/etc/passwd`, `]` are three arguments.
3. **Execution**: Bash runs the `test` builtin.
4. **System call**: `test` calls `stat("/etc/passwd", &statbuf)` — a kernel system call.
5. **Kernel**: VFS layer traverses the filesystem, reads the inode metadata, returns it.
6. **Check**: `test` checks `S_ISREG(statbuf.st_mode)` for regular file.
7. **Result**: Returns 0 if true, 1 otherwise.

### The stat System Call

Every file test operator calls `stat()`, `lstat()`, or `access()`:

| Operator | Syscall | What It Checks |
|----------|---------|----------------|
| `-e` | `stat()` | File exists |
| `-f` | `stat()` | `S_ISREG(st_mode)` |
| `-d` | `stat()` | `S_ISDIR(st_mode)` |
| `-L` | `lstat()` | Checks link itself, not target |
| `-r` | `access(R_OK)` | EUID + mode bits + capabilities |
| `-w` | `access(W_OK)` | EUID + mode bits |
| `-x` | `access(X_OK)` | EUID + mode bits |
| `-s` | `stat()` | `st_size > 0` |
| `-N` | `stat()` | `st_atime < st_mtime` |
| `-O` | `stat()` | `st_uid == geteuid()` |
| `-G` | `stat()` | `st_gid == getegid()` |
| `-u` | `stat()` | `st_mode & S_ISUID` |
| `-g` | `stat()` | `st_mode & S_ISGID` |
| `-k` | `stat()` | `st_mode & S_ISVTX` |
| `-nt` | `stat()` both | Compare `st_mtime` |
| `-ot` | `stat()` both | Compare `st_mtime` |
| `-ef` | `stat()` both | Compare `st_dev` + `st_ino` |

### strace Trace

```bash
$ strace -e trace=stat,newfstatat,lstat,access bash -c '[ -f /etc/passwd ] && echo yes'
newfstatat(AT_FDCWD, "/etc/passwd", {st_mode=S_IFREG|0644, ...}, 0) = 0
write(1, "yes\n", 4)                    = 4
```

### The EUID Bypass

When running as root, the kernel bypasses permission checks. Root has `CAP_DAC_OVERRIDE`:
- `-r` returns true even on `chmod 000` files
- `-w` returns true even on read-only files
- `-x` still requires at least one execute bit

### Follow vs No-follow

`stat()` follows symlinks. `lstat()` does not. This means:
- `[ -f /symlink ]` — if target is a file, `stat` tells us. Returns true.
- `[ -d /symlink ]` — if target is a directory, returns true.
- `[ -L /symlink ]` — uses `lstat`, sees symlink regardless of target. Returns true.
- `[ -f /symlink ] && [ ! -L /symlink ]` — true only if regular file AND not a symlink.

## Core Examples (12)

### Example 1: Check File Type
```bash
$ path="/etc"
if [ -e "$path" ]; then
    if [ -d "$path" ]; then
        echo "$path is a directory"
    elif [ -f "$path" ]; then
        echo "$path is a regular file"
    elif [ -L "$path" ]; then
        echo "$path is a symlink"
    fi
fi
/etc is a directory
```
The nested if first checks existence, then drills into type. `-e` catches everything, then specific tests narrow it.

### Example 2: Permission Checks
```bash
$ test -r /etc/shadow && echo "readable" || echo "not readable"
not readable
$ test -w /tmp && echo "writable" || echo "not writable"
writable
$ test -x /home/phd && echo "accessible" || echo "not accessible"
accessible
```
`test` without brackets is the same command. These check what the current user can actually DO.

### Example 3: Negation
```bash
$ if [ ! -f /tmp/lockfile ]; then
    echo "Lockfile absent - safe to proceed"
fi
Lockfile absent - safe to proceed
$ if [ ! -w /etc/hosts ]; then
    echo "Cannot write to hosts file"
fi
Cannot write to hosts file
```
Negation with `!` flips the result. Common for guard clauses.

### Example 4: Combining Tests
```bash
$ file="/etc/passwd"
$ if [ -f "$file" ] && [ -r "$file" ]; then
    echo "File exists, is regular, and is readable"
fi
File exists, is regular, and is readable
```
`&&` between two `[ ]` commands is clearer than `-a` inside one `[ ]`.

### Example 5: Newer/Older
```bash
$ touch /tmp/newfile
$ touch -t 202001010000 /tmp/oldfile
$ if [ /tmp/newfile -nt /tmp/oldfile ]; then
    echo "newfile is newer"
  elif [ /tmp/newfile -ot /tmp/oldfile ]; then
    echo "newfile is older"
  else
    echo "same age"
  fi
newfile is newer
```
`-nt` and `-ot` compare modification timestamps.

### Example 6: Empty vs Non-Empty
```bash
$ touch /tmp/empty
$ echo "data" > /tmp/nonempty
$ [ -s /tmp/empty ] && echo "has content" || echo "empty"
empty
$ [ -s /tmp/nonempty ] && echo "has content" || echo "empty"
has content
```
`-s` checks if size > 0 bytes.

### Example 7: Special Files
```bash
$ [ -c /dev/tty ] && echo "Character device"
Character device
$ [ -b /dev/sda ] && echo "Block device"
Block device
$ [ -S /var/run/docker.sock ] 2>/dev/null && echo "Socket" || echo "No Docker socket"
No Docker socket
$ [ -p /tmp/myfifo ] 2>/dev/null && echo "Named pipe" || echo "No FIFO"
No FIFO
```

### Example 8: SUID, SGID, Sticky Bit
```bash
$ [ -u /usr/bin/sudo ] && echo "SUID"
SUID
$ ls -l /usr/bin/sudo
-rwsr-xr-x 1 root root ...  # 's' where 'x' should be
$ [ -k /tmp ] && echo "Sticky bit set"
Sticky bit set
$ ls -ld /tmp
drwxrwxrwt ...  # 't' at the end
```
`-u` = SUID, `-g` = SGID, `-k` = sticky bit.

### Example 9: -ef Same File Detection
```bash
$ touch /tmp/original
$ ln /tmp/original /tmp/hardlink
$ ls -li /tmp/original /tmp/hardlink
12345 -rw-r--r-- 2 ... /tmp/original
12345 -rw-r--r-- 2 ... /tmp/hardlink
$ if [ /tmp/original -ef /tmp/hardlink ]; then
    echo "Same file (hard link)"
  fi
Same file (hard link)
```
`-ef` compares device and inode numbers.

### Example 10: -N Modified Since Read
```bash
$ echo "hello" > /tmp/newfile
$ cat /tmp/newfile
$ [ -N /tmp/newfile ] && echo "Modified since read" || echo "Not modified"
Not modified
$ echo "more" >> /tmp/newfile
$ [ -N /tmp/newfile ] && echo "Modified since read" || echo "Not modified"
Modified since read
```
`-N` compares atime vs mtime.

### Example 11: File Descriptor is TTY
```bash
$ [ -t 0 ] && echo "stdin is terminal" || echo "stdin is pipe/file"
stdin is terminal
$ echo "hello" | [ -t 0 ] && echo "stdin is terminal" || echo "stdin is pipe"
stdin is pipe
```
`-t` checks if fd is connected to a terminal.

### Example 12: Path Validation Script
```bash
$ cat > ~/validate_path.sh << 'EOF'
#!/bin/bash
path="$1"
errors=0

[ -e "$path" ] || { echo "ERROR: $path does not exist"; errors=1; }
[ -r "$path" ] || { echo "ERROR: $path not readable"; errors=1; }

if [ -d "$path" ]; then
    [ -x "$path" ] || { echo "ERROR: $path not searchable"; errors=1; }
fi

if [ -f "$path" ]; then
    [ -s "$path" ] || echo "WARNING: $path is empty"
fi

exit $errors
EOF
$ ./validate_path.sh /etc
$ ./validate_path.sh /root
ERROR: /root does not exist
```

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Startup scripts**: `[ -f /var/lock/service.lock ]` prevents double-starting
- **Log rotation**: `[ -s /var/log/service.log ]` before rotating empty logs
- **Backup verification**: `[ -s backup.tar.gz ]` after creating backup
- **Mount checks**: `[ -d /mnt/backup ] && df /mnt/backup`
- **Service validation**: `[ -x /usr/sbin/sshd ]` before restart

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Build scripts**: `[ -f Makefile ] && make`
- **File processing**: `[ -s "$input" ] && process < "$input"`
- **Validation**: `[ -r "$config" ] && source "$config"`
- **Temp files**: `[ -d /tmp/build ] && rm -rf /tmp/build`
- **Caching**: `[ cache/"$key" -nt source-file ]` to decide rebuild

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Writable shadow**: `[ -w /etc/shadow ] && cat /etc/shadow`
- **SUID enumeration**: `[ -u /usr/bin/pkexec ]` for LPE
- **Writable dirs in PATH**: `[ -w /usr/local/bin ]` to plant binaries
- **TOCTOU races**: Check then use — file can swap between operations
- **Cron job abuse**: Finding writable cron directories

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **File integrity**: `[ -N /bin/ls ]` — was it examined?
- **Permission auditing**: `find / -perm -4000 -o -perm -2000`
- **Rootkit detection**: `[ -f /bin/ls -a -x /bin/ls -a -s /bin/ls ]`
- **Permission check**: `[ "$(stat -c '%a' /etc/shadow)" = "400" ]`
- **Backdoor detection**: Unexpected owners on cron files

## Memory Aids

- **`-f`**: "regular File" (not dir, not device)
- **`-d`**: Shape of D for "Directory"
- **`-r`/`-w`/`-x`**: "Read"/"Write"/"eXecute"
- **`-s`**: "Size" > 0
- **`-L`**: "Link" (symboLic)
- **`-u`/`-g`/`-k`**: "User" (SUID), "Group" (SGID), "sticKy" (sticky)
- **`-nt`/`-ot`**: "Newer Than" / "Older Than"
- **`-ef`**: "Equal File" — same inode
- **`-N`**: "New" — modified since read
- **`-O`/`-G`**: "Owned" / "Group owned"
- **`-b`/`-c`/`-p`/`-S`**: Block, Character, Pipe, Socket

## Trap Vault (12 traps)

### Trap 1: -x on Directories
**Problem:** A directory passes `-x` but is not "executable."
**Example:**
```bash
$ [ -x /tmp ] && echo "executable"
executable
```
**Why:** `-x` on a directory means "searchable" (enter it), not executable.
**Fix:** `[ -d "$dir" -a -x "$dir" ]` for "accessible directory."

### Trap 2: -L vs -f — Symlinks
**Problem:** `-L` true for symlinks but `-f` follows the link.
**Example:**
```bash
$ ln -s /etc/passwd /tmp/mylink
$ [ -L /tmp/mylink ] && echo "is symlink"
is symlink
$ [ -f /tmp/mylink ] && echo "is file"
is file  # target is a file!
```
**Why:** `-f` uses `stat()` which follows symlinks.
**Fix:** `[ -f "$path" ] && [ ! -L "$path" ]` for "regular file, not symlink."

### Trap 3: Root Bypasses -r
**Problem:** Root can read chmod 000 files.
**Example:**
```bash
# As root:
$ chmod 000 /tmp/secret
$ [ -r /tmp/secret ] && echo "readable"
readable
```
**Why:** Root has `CAP_DAC_OVERRIDE`.
**Fix:** `sudo -u nobody test -r /tmp/secret` to test as non-root.

### Trap 4: -s Returns True for Directories
**Problem:** `-s` on an empty directory returns true.
**Example:**
```bash
$ mkdir /tmp/emptydir
$ [ -s /tmp/emptydir ] && echo "has content" || echo "empty"
has content
```
**Why:** Directories have minimum size (4096+ bytes).
**Fix:** Use `find "$path" -maxdepth 0 -empty` for directories.

### Trap 5: TOCTOU Race Condition
**Problem:** File passes test but changes before use.
**Example:**
```bash
$ if [ -f /tmp/lockfile ]; then
    # BETWEEN CHECK AND USE, another process removed it!
    rm /tmp/lockfile  # Might fail
  fi
```
**Why:** Time of Check to Time of Use race.
**Fix:** Try the operation directly: `rm /tmp/lockfile 2>/dev/null || true`.

### Trap 6: Permissions + sudo
**Problem:** `-w` false but user can sudo.
**Example:**
```bash
$ [ -w /etc/hosts ] && echo "writable" || echo "not writable"
not writable
$ sudo -n true && sudo tee -a /etc/hosts <<< "..."  # Works!
```
**Fix:** Test sudo separately.

### Trap 7: -e Returns True for Broken Symlinks
**Problem:** `-e` true for symlinks to nothing.
**Example:**
```bash
$ ln -s /nonexistent /tmp/broken
$ [ -e /tmp/broken ] && echo "exists"
exists  # But target doesn't exist!
```
**Why:** `-e` uses `lstat()` — the symlink itself exists.
**Fix:** `[ -e "$path" ] && [ ! -L "$path" ]` to exclude symlinks.

### Trap 8: -t 0 Surprises
**Problem:** `-t 0` changes with pipes.
**Example:**
```bash
$ echo "" | [ -t 0 ] && echo "terminal" || echo "pipe"
pipe
```
**Fix:** Remember piping changes stdin fd.

### Trap 9: -N Is Unreliable
**Problem:** `-N` rarely true on modern Linux.
**Example:**
```bash
$ vim /tmp/file  # edit and save
$ [ -N /tmp/file ] && echo "modified" || echo "not"
not
```
**Why:** Vim reads file (sets atime). Also, `relatime` mount option.
**Fix:** Don't rely on `-N` for critical checks.

### Trap 10: -ef Across Mount Points
**Problem:** Files on different filesystems never match.
**Example:**
```bash
$ [ /etc/hosts -ef /proc/self/mounts ] && echo "same"
```
**Why:** `-ef` compares device AND inode. Different devices = never same.
**Fix:** By design — they ARE different files.

### Trap 11: -nt Without Existence Checks
**Problem:** Non-existent file in `-nt` comparison.
**Example:**
```bash
$ [ /nonexistent -nt /tmp/real ] && echo "newer" || echo "not newer"
not newer
```
**Why:** If either file doesn't exist, result is false.
**Fix:** Check existence first.

### Trap 12: -a and -o Deprecated
**Problem:** `-a`/`-o` inside `[ ]` are ambiguous.
**Example:**
```bash
$ [ -n "$var" -a "$var" -gt 5 ]  # Fragile!
```
**Why:** POSIX marks them as obsolescent. They fail with certain values.
**Fix:** `[ -n "$var" ] && [ "$var" -gt 5 ]`

## See It In The Wild

### Try this now:
```bash
# 1. Count all file types on your system
$ find /etc -type f 2>/dev/null | wc -l
$ find /etc -type d 2>/dev/null | wc -l
$ find /etc -type l 2>/dev/null | wc -l

# 2. Check what you can and can't read
$ for f in /etc/shadow /etc/passwd /var/log/syslog; do
    [ -r "$f" ] && echo "readable: $f" || echo "not readable: $f"
  done

# 3. Find all SUID binaries
$ find / -type f -perm -4000 2>/dev/null

# 4. Compare two files' timestamps
$ touch /tmp/a
$ sleep 2
$ touch /tmp/b
$ [ /tmp/b -nt /tmp/a ] && echo "b is newer"

# 5. Check for empty files in /tmp
$ for f in /tmp/*; do
    [ -f "$f" ] && [ ! -s "$f" ] && echo "Empty: $f"
  done 2>/dev/null
```

## Check Your Understanding (7 questions)

1. What does `-x` mean for a directory vs a file?

2. If a symlink points to a directory, does `-f` return true or false? What about `-d`?

3. What is the difference between `-e` and `-f`?

4. What does `-s` test? What is its negation?

5. Why might `-r` return true even for a file with `chmod 000`?

6. How does `-nt` work? What does it return if one file doesn't exist?

7. What is the TOCTOU problem and how does it relate to file tests?

## Advanced Examples (extra)

### Example 13: Using -t to Check if stdin is Interactive
This is crucial for scripts that behave differently when piped:
```bash
$ cat > ~/color_output.sh << 'EOF'
#!/bin/bash
if [ -t 1 ]; then
    echo -e "\033[32mGreen text on terminal"
else
    echo "Plain text in pipe"
fi
EOF
$ ./color_output.sh
Green text on terminal
$ ./color_output.sh | cat
Plain text in pipe
```

### Example 14: Detecting Mount Points
Check if a directory is a mount point using device comparison:
```bash
$ check_mount() {
    local dir="$1"
    local parent=$(dirname "$dir")
    if [ "$dir" -ef "$parent" ]; then
        echo "$dir is NOT a mount point"
    else
        echo "$dir IS a mount point (different device)"
    fi
}
$ check_mount /mnt
/mnt IS a mount point (different device)
$ check_mount /root
/root is NOT a mount point
```

### Example 15: Checking File Change With -N
Despite relatime, -N can still work on recently created files:
```bash
$ tmp=$(mktemp)
$ echo "initial" > "$tmp"
$ cat "$tmp" > /dev/null
$ [ -N "$tmp" ] && echo "Modified since read"
$ echo "modified" > "$tmp"
$ [ -N "$tmp" ] && echo "Modified since read" || echo "Not modified"
Modified since read
$ rm "$tmp"
```

### Example 16: Security Audit — Find World-Writable Scripts in PATH
```bash
$ IFS=:
$ for dir in $PATH; do
    [ -d "$dir" ] || continue
    [ -w "$dir" ] && echo "WARNING: $dir is writable!"
    for f in "$dir"/*; do
        [ -f "$f" ] && [ -x "$f" ] && [ -w "$f" ] && echo "  $f is world-writable"
    done
  done 2>/dev/null
```

### Example 17: Check if a File is a Script
```bash
$ is_script() {
    local file="$1"
    [ -f "$file" ] && [ -r "$file" ] && [ -x "$file" ] &&
    head -1 "$file" | grep -q '^#!'
}
$ is_script /usr/bin/clear && echo "Is script" || echo "Not script"
Is script
$ is_script /bin/ls && echo "Is script" || echo "Not script"
Not script
```
This checks: is it a regular file? Readable? Executable? First line starts with `#!`? Only true for interpreted scripts.

## See It In The Wild

### Try this now:
```bash
# 1. Count all file types in /etc
$ echo "Regular files: $(find /etc -type f 2>/dev/null | wc -l)"
$ echo "Directories: $(find /etc -type d 2>/dev/null | wc -l)"
$ echo "Symlinks: $(find /etc -type l 2>/dev/null | wc -l)"
$ echo "Sockets: $(find /etc -type s 2>/dev/null | wc -l)"
$ echo "FIFOs: $(find /etc -type p 2>/dev/null | wc -l)"

# 2. Find files you can't read
$ find / -type f ! -readable 2>/dev/null | head -10

# 3. Test if your terminal supports color
$ [ -t 1 ] && echo "Terminal supports color output"

# 4. Find all SUID binaries (potential security risk)
$ find / -type f -perm -4000 2>/dev/null

# 5. Check if two files are hard links
$ ln /etc/hosts /tmp/hosts_link
$ [ /etc/hosts -ef /tmp/hosts_link ] && echo "Same file"
$ rm /tmp/hosts_link
```
