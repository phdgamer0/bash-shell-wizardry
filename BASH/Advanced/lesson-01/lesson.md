# Lesson 1: File Descriptors & exec

## History & Origins

The file descriptor model was one of the most radical simplifications Unix introduced in the early 1970s. Before Unix, operating systems had dozens of different I/O control blocks, each with its own API — there were separate system calls for tape drives, card readers, disk files, and terminals. Ken Thompson and Dennis Ritchie said: "Everything is a file." Every I/O object — a regular file, a pipe, a socket, a terminal, a device — is represented by an integer called a file descriptor, and the same read(2)/write(2) system calls work on all of them.

The three standard descriptors (0 = stdin, 1 = stdout, 2 = stderr) were established by the earliest Unix shells in 1971. The convention came from the teletype model: keyboard input on one channel, printed output on another, error messages on a third so they could be separately redirected. The exec builtin appeared in the Bourne shell (1977) and was inherited by bash. The ability to manipulate arbitrary FDs (beyond 0/1/2) came later; bash's `exec N>file` form was present by the mid-1980s. Dynamic FD allocation (`exec {var}>file`) appeared in bash 4.1 (2009), solving the long-standing problem of hardcoded FD conflicts.

## Syntax Reference

### Opening FDs
```
exec N<file       # Open file for reading on FD N
exec N>file       # Open file for writing (truncate) on FD N
exec N>>file      # Open file for writing (append) on FD N
exec N<>file      # Open file for reading AND writing on FD N
exec {var}<file   # Bash chooses FD number, stored in $var (bash 4.1+)
exec {var}>file   # Same for writing
exec {var}>>file  # Same for append
exec {var}<>file  # Same for read-write
```

### Closing FDs
```
exec N<&-         # Close FD N (input direction)
exec N>&-         # Close FD N (output direction)
exec N<&- N>&-    # Close FD N completely (both directions)
```

### Duplicating FDs
```
exec N<&M         # Duplicate FD M as FD N for reading
exec N>&M         # Duplicate FD M as FD N for writing
exec N<&M-        # Duplicate FD M as FD N for reading, then close M
exec N>&M-        # Duplicate FD M as FD N for writing, then close M (move)
```

### Moving FDs (bash 4.3+)
```
exec N<&M-        # Dup FD M onto N, then close M (move input)
exec N>&M-        # Dup FD M onto N, then close M (move output)
```

### Special file paths for FDs
```
/dev/fd/N          # Access FD N as a file name
/dev/stdin         # Synonym for /dev/fd/0
/dev/stdout        # Synonym for /dev/fd/1
/dev/stderr        # Synonym for /dev/fd/2
/proc/self/fd/N    # Linux procfs — same as /dev/fd/N but more info
```

### The `close-on-exec` flag (FD_CLOEXEC)
- By default, FDs remain open across exec. This can leak FDs into child processes.
- bash 4.1+ `exec {var}>file` opens with FD_CLOEXEC set automatically.
- Manual `exec N>file` does NOT set FD_CLOEXEC — the FD leaks to child processes.
- Force close-on-exec with `exec N<>file` then use `{var}>file` form.

## Under the Hood

### The Kernel's File Descriptor Table

Every process in Linux has a `struct files_struct` in the kernel, which contains an array of `struct file *` pointers — this is the file descriptor table. The size of this array is bounded by `RLIMIT_NOFILE` (default 1024 on most systems, but configurable up to the system-wide limit in `/proc/sys/fs/file-max`).

```
Process A                          Kernel Memory
+-----------------+                +--------------------------+
| files_struct    |                | inode / dentry / page    |
|   fd[0] --------+---> struct file ---> cache for /etc/hosts  |
|   fd[1] --------+---> struct file ---> /dev/pts/0 (terminal) |
|   fd[2] --------+---> struct file ---> /dev/pts/0 (terminal) |
|   fd[3] --------+---> struct file ---> /tmp/mylock           |
|   fd[4] --------+---> struct file ---> /tmp/output.txt       |
|   ...           |                +--------------------------+
+-----------------+
```

Each `struct file` contains:
- `f_pos` — current file offset (for regular files)
- `f_flags` — O_RDONLY, O_WRONLY, O_RDWR, O_APPEND, O_NONBLOCK, O_CLOEXEC
- `f_op` — pointer to the file operations struct (read, write, open, release, etc.)
- `f_count` — reference count (how many FDs point to this file)
- `f_inode` — pointer to the inode (the actual data on disk)

The `struct file` is shared when you `dup()` an FD. Both FDs point to the same `struct file`, sharing the same file offset. This is critical to understand: when you do `exec 4>&1`, FDs 1 and 4 share the same file offset and flags. A `write` to FD 4 advances the offset, and a subsequent `write` to FD 1 starts from the new position.

### What strace Shows

When you run `exec 3</etc/hostname`:

```
openat(AT_FDCWD, "/etc/hostname", O_RDONLY) = 3
```

The shell may additionally call `fcntl(3, F_SETFD, FD_CLOEXEC)` depending on how the FD was opened.

For a simple redirect `exec 3>file`:

```
openat(AT_FDCWD, "/tmp/test.log", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
```

Note O_TRUNC — that's why the file is truncated immediately. The file is opened and truncated before any data is written.

For `exec 3>>file`:

```
openat(AT_FDCWD, "/tmp/test.log", O_WRONLY|O_CREAT|O_APPEND, 0666) = 3
```

O_APPEND means every write goes to the end atomically, regardless of the current file offset.

For `exec 3<>file`:

```
openat(AT_FDCWD, "/tmp/rw.txt", O_RDWR|O_CREAT, 0666) = 3
```

O_RDWR — no truncation, no append. The file offset starts at 0.

### FD Closure Semantics

When you call `exec N<&-`, the kernel does `close(3) = 0`. The kernel decrements `f_count` on the `struct file`. If it reaches 0, the file is actually closed and resources are freed. If another FD still points to the same struct file (via dup), the file stays open.

### Security Model Interactions

**SELinux/AppArmor:** File descriptor operations are subject to the LSM (Linux Security Module) hooks. `exec 3>/etc/shadow` will fail with EACCES if SELinux policy denies write access to shadow, even if the Unix permissions would allow it. SELinux labels are checked at file open time, not at read/write time.

**Capabilities:** Opening files for reading requires `CAP_DAC_READ_SEARCH` only if the normal permission check would fail. Opening for writing requires `CAP_DAC_OVERRIDE`.

**Namespaces:** When you use `exec N<>/dev/tcp/host/port`, the socket creation goes through the network namespace. If you're in a container with restricted network, this will fail with EPERM.

### Memory Layout Implications

FDs themselves don't consume significant kernel memory — each `struct file` is about 200-400 bytes. The per-process FD table is `sizeof(struct file *) * rlimit_nofile`. At the default 1024, that's about 8KB per process. But the `struct file` entries themselves are shared across processes (if inherited from a common ancestor via fork or passed via Unix sockets).

The real memory concern: when you open a file for reading, the kernel may cache its pages in the page cache. Reading a 1GB file via FDs will use 1GB of page cache until the kernel evicts it. This is independent of the FD itself.

### Comparison with C/libc

In C, you use `open()`, `close()`, `dup2()`, `read()`, `write()`. In bash, `exec` is the equivalent:

| C function | Bash equivalent |
|---|---|
| `int fd = open("file", O_RDONLY)` | `exec N<file` |
| `int fd = open("file", O_WRONLY\|O_CREAT\|O_TRUNC, 0666)` | `exec N>file` |
| `close(fd)` | `exec N<&-` |
| `dup2(oldfd, newfd)` | `exec N<&M` or `exec N>&M` |
| `dup2(oldfd, newfd); close(oldfd)` | `exec N<&M-` (move) |
| `fcntl(fd, F_SETFD, FD_CLOEXEC)` | Not directly; use `{var}>file` for auto CLOEXEC |
| `lseek(fd, 0, SEEK_SET)` | Reopen the file |

The crucial difference: in C, you explicitly manage buffers with `stdio` (setvbuf, fflush). In bash, the shell and external commands handle buffering. When you write `echo "data" >&3`, bash's `echo` builtin writes to FD 3 using `write(3, "data\n", 5)`. No user-space buffering. But when you use `cat <&3`, cat reads in buffered mode, typically 8KB blocks, using `read()`.

### O_CLOEXEC and the `{var}>` Form

Before bash 4.1, every FD opened with `exec N>file` leaked to exec'd children. A child process (like a command run in the script) would inherit all open FDs. This is a security concern: a child could read from an FD it shouldn't have access to.

Bash 4.1 introduced `exec {var}>file` which opens with O_CLOEXEC, meaning the FD is automatically closed on any exec. Check this:

```bash
$ exec {fd}>/tmp/test.txt
$ bash -c 'ls -la /dev/fd/' 2>/dev/null | grep -c test
0  # Not inherited — CLOEXEC worked
```

But with `exec N>file`:

```bash
$ exec 5>/tmp/test.txt
$ bash -c 'ls -la /dev/fd/' 2>/dev/null | grep -c 5
1  # Inherited — FD leaked!
```

## Core Examples

### Example 1: Reading from a Custom FD

```bash
$ exec 3</etc/hostname
$ cat <&3
myhost
$ exec 3<&-
```

**Anatomy:**
1. `exec 3</etc/hostname` — opens `/etc/hostname` (O_RDONLY) and assigns it to FD 3
2. `cat <&3` — runs `cat` with FD 3 as stdin. `cat` reads the file to EOF
3. `exec 3<&-` — closes FD 3

**What if variations:**
- What if `/etc/hostname` doesn't exist? → exec fails, bash prints error, script continues unless `set -e`
- What if you read twice? → After `cat <&3` reads to EOF, the file offset is at end. A second `cat <&3` reads nothing.
- What if you rewind? → Use `exec 3</etc/hostname` again to reopen.

**Dangerous edge case:** If you forget to close FD 3 and the script continues for a long time, you've leaked an FD. If this happens in a loop, you'll exhaust the FD limit.

### Example 2: Writing to a Custom FD (Truncate Mode)

```bash
$ exec 4> /tmp/test.log
$ echo "Process started" >&4
$ echo "Process ended" >&4
$ exec 4<&-
$ cat /tmp/test.log
Process started
Process ended
```

**Anatomy:**
1. `exec 4>/tmp/test.log` — truncates the file to zero bytes and opens FD 4 for writing
2. `echo ... >&4` — writes to FD 4
3. `exec 4<&-` — closes FD 4

**TRAP:** The file is truncated immediately at step 1, even if you never write a single byte. `exec 4>file` is equivalent to `: >file` combined with opening for writing at offset 0 (not end).

```bash
$ exec 4>/tmp/important.txt   # Oops — file is now empty
$ exec 4<&-
$ cat /tmp/important.txt      # Empty!
```

### Example 3: Read-Write with `<>`

```bash
$ exec 5<> /tmp/rw.txt
$ echo "hello world" >&5
$ exec 5<&-                    # Close and reopen to rewind
$ exec 5<> /tmp/rw.txt
$ head -1 <&5
hello world
$ exec 5<&-
```

**Anatomy:**
1. `exec 5<>/tmp/rw.txt` — opens for read-write, no truncation, offset at 0
2. `echo "hello world" >&5` — writes "hello world\n" to the file, offset now at 12
3. `exec 5<&-` — closes
4. Reopen and read

Without closing/reopening, reading after writing gets nothing because the offset is at the end:

```bash
$ exec 5<>/tmp/rw.txt
$ echo "hello" >&5
$ cat <&5                     # Nothing — offset is at byte 6
$ exec 5<&-
```

### Example 4: Duplicating FDs — Redirecting Stderr

```bash
$ exec 2> /tmp/errors.log
$ ls /nonexistent
$ cat /tmp/errors.log
ls: cannot access '/nonexistent': No such file or directory
```

**Anatomy:**
1. `exec 2>/tmp/errors.log` — redirects stderr (FD 2) to the log file
2. `ls /nonexistent` — writes error to stderr, which goes to the log
3. stdout remains connected to the terminal

**Full save-and-restore pattern:**
```bash
exec 3>&2                     # Save stderr to FD 3
exec 2> /tmp/errors.log       # Redirect stderr
... commands ...
exec 2>&3                     # Restore stderr
exec 3<&-                     # Close saved FD
```

### Example 5: Multi-Stream Logging

```bash
$ exec 3> /tmp/stdout.log 4> /tmp/stderr.log
$ echo "Normal" >&3
$ echo "Error" >&4
$ exec 3<&- 4<&-
$ cat /tmp/stdout.log
Normal
```

**Anatomy:** Opening multiple FDs in one exec command. Each FD points to a different file.

### Example 6: Dynamic FD Assignment (bash 4.1+)

```bash
$ exec {log_fd}>/tmp/app.log
$ echo "log_fd=$log_fd"
log_fd=10
$ echo "Application started" >&$log_fd
$ exec {log_fd}>&-
```

**Anatomy:**
1. `{log_fd}>` — bash picks the first available FD above 9 and stores it in `log_fd`
2. The FD is automatically set with O_CLOEXEC
3. Reference it with `>&$log_fd` (note the `$`)
4. Close with `{log_fd}>&-` (no `$`)

**TRAP:** You must NOT put `$` when assigning (`{log_fd}>`) but you MUST put `$` when using (`>&$log_fd`). Closing also uses `{log_fd}>&-` without `$`.

```bash
$ # WRONG:
$ echo "data" >&log_fd        # Missing $
$ # Correct:
$ echo "data" >&$log_fd
$ # Closing WRONG:
$ exec $log_fd>&-             # Bash doesn't understand this for closing
$ # Correct:
$ exec {log_fd}>&-            # Bash knows to resolve the variable
```

### Example 7: The `/dev/fd/` Filesystem

```bash
$ echo Hello | cat /dev/fd/0
Hello
```

`/dev/fd/` is a symbolic link to `/proc/self/fd/` on Linux. It's a virtual filesystem that shows the current process's open file descriptors as named files. You can pass `/dev/fd/N` to any program that expects a file path.

**Use case:** Process substitution like `diff <(cmd1) <(cmd2)` internally uses `/dev/fd/`.

```bash
$ diff /dev/fd/3 /dev/fd/4 3< <(echo hello) 4< <(echo world)
1c1
< hello
---
> world
```

### Example 8: Lock File with flock

```bash
$ exec 6> /tmp/mylock.lock
$ flock -n 6 && echo "Lock acquired" || echo "Lock failed"
Lock acquired
$ exec 6<&-
```

**Anatomy:**
1. `exec 6>/tmp/mylock.lock` — opens/creates the lock file
2. `flock -n 6` — attempts to acquire an exclusive lock on FD 6. `-n` means non-blocking
3. `exec 6<&-` — releases the lock AND closes the FD (flock releases when FD is closed)

**What if variations:**
- Shared lock: `flock -s 6` — allows multiple readers
- Blocking lock: `flock 6` — waits until lock is available
- Timeout lock: `flock -w 5 6` — wait at most 5 seconds

**Dangerous edge case:** If your script crashes without closing the FD, the lock is released automatically (kernel closes all FDs on process exit). This is actually a feature — locks don't persist after death.

### Example 9: Discard Output with `/dev/null`

```bash
$ exec 3>/dev/null
$ echo "Silent" >&3
$ exec 3<&-
```

Redirecting to `/dev/null` is a special case: writes succeed but the data is discarded by the kernel's null driver.

### Example 10: Opening the Same File on Multiple FDs

```bash
$ exec 5</etc/hostname 6>/tmp/copy.txt
$ while IFS= read -r line <&5; do
>   echo "$line" >&6
> done
$ exec 5<&- 6<&-
```

**Anatomy:**
- FD 5 reads from source, FD 6 writes to destination
- Each FD has its own file offset (separate `struct file` entries)
- When FD 5 hits EOF, `read -r` returns false and the loop ends

### Example 11: FIFO with Bidirectional Open

```bash
$ pipe=$(mktemp -u)
$ mkfifo "$pipe"
$ exec 7<>"$pipe"              # Open FIFO for both reading and writing
$ echo "data" >&7              # Write to FIFO (non-blocking because write end is open)
$ read -r data <&7             # Read from FIFO
$ echo "$data"
data
$ rm "$pipe"
$ exec 7<&-
```

Opening a FIFO on both read and write prevents blocking. Without the read end also open, writing to a FIFO blocks until a reader connects.

### Example 12: FD Redirection to a Socket

```bash
$ exec 3<>/dev/tcp/example.com/80
$ printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
$ head -5 <&3
HTTP/1.1 200 OK
...
$ exec 3<&-
```

This only works if bash was compiled with `--enable-net-redirections`. On the kernel side, bash calls `socket(AF_INET, SOCK_STREAM, 0)`, then `connect(3, ...)`.

```bash
$ ls -la /dev/tcp
ls: cannot access '/dev/tcp': No such file or directory  # It's a bash builtin, not a device
```

### Example 13: FD Moving — Safe Cleanup

```bash
$ exec 5>/tmp/output.txt
$ exec 6>&5-                   # Move FD 5 to FD 6 (dup 5->6, then close 5)
$ echo "data" >&6
$ exec 6<&-
```

The `move` operation (`>&M-`) duplicates then closes atomically. This is safer than separate dup + close because if the dup fails, the original FD is not closed.

### Example 14: FD Redirection with Process Substitution

```bash
$ exec 3< <(echo "hello world")
$ cat <&3
hello world
$ exec 3<&-
```

The `<(cmd)` form creates a named pipe (or `/dev/fd/N` entry) and executes the command in a subshell, piping its output into the FD. Your script reads from that FD.

### Example 15: Checking and Setting FD Limits

```bash
$ ulimit -n
1024
$ ulimit -n 4096
$ exec 4>/tmp/test
$ lsof -p $$ | grep 4
bash  1234  user  4w  REG  8,1  0  123456  /tmp/test
$ exec 4<&-
```

## Real-World Use Cases

### FOR the OS
- **Daemon initialization:** During startup, daemons close all inherited FDs (except 0/1/2) to avoid leaking descriptors from the init process
- **Log rotation:** Send output to a new log file without restarting: reopen FDs via exec
- **Lock files:** `/var/run/*.pid` files use `flock` on FDs to prevent duplicate daemon instances

### WITH the OS
- **Network connection tracking:** `/proc/net/tcp` lists all TCP sockets with their inode numbers; match these against `/proc/*/fd/*` to find which process owns which connection
- **Debugging with strace:** `strace -e trace=close,dup2 bash script.sh` reveals all FD management
- **System monitoring:** `/proc/sys/fs/file-nr` shows system-wide FD usage

### AGAINST the OS (defense perspective)
- **FD exhaustion attack:** An attacker can open many FDs to exhaust the process's `ulimit -n`, causing legitimate operations to fail with "Too many open files"
- **FD inheritance exploits:** If a privileged process forgets to close sensitive FDs before execing a user-controlled binary, the child can read/write those FDs
- **procfs leaks:** `/proc/$pid/fd/` is readable by the process owner, which can reveal data in FDs of other processes owned by the same user

### FOR DEFENSE
- **FD audit scripts:** Check all processes for unexpected socket FDs (detect reverse shells)
- **Close-on-exec hardening:** Always use `{var}>file` to prevent FD leaks to child processes
- **Limit FDs:** Set `ulimit -n` appropriately for services to prevent both accidental and malicious FD exhaustion
- **Restrict /proc access:** Mount `/proc` with `hidepid=2` to prevent users from seeing other processes' FDs

## Memory Aids

- **"0-1-2, everything else is you"** — 0=stdin, 1=stdout, 2=stderr; custom FDs start at 3
- **`<` points like an arrow INTO the command** — `3<file`: FD 3 reads FROM file
- **`>` points OUT of the command** — `4>file`: FD 4 writes TO file
- **`<>` is both ways** — think of it as a bidirectional arrow
- **`&-` means "and close"** — the `&` is "FD reference" and `-` is "minus/close"
- **`{var}` as FD number** — Let bash play the "pick a card" game; it chooses the lowest available FD above 9
- **`/dev/fd/N` — the window into your process's soul**: Every FD is visible as a named file

## Trap Vault

1. **Truncation on open:** `exec 4>file` truncates the file immediately. Not when you write. Not when you close. Immediately. This is the #1 FD trap.

2. **FD leaks across exec:** `exec 3>file` without `{var}` form inherits the FD to child processes. If a child writes to FD 3, it writes to your file. Always use `{var}>` for isolation.

3. **File offset sharing with dup:** `exec 4>&1` means FDs 4 and 1 share the same file offset and flags. Writing to FD 4 changes the position for FD 1. If stdout is a file, `echo "a" >&4; echo "b"` — "b" overwrites "a" because the offset moved.

4. **Closing only one direction:** `exec N<&-` closes only the read end of FD N. If the FD was opened bidirectional (`<>`), the write end remains open. Use `exec N<&- N>&-` to close both.

5. **The `{var}` dereference gotcha:** `exec {fd}>file` assigns, `>&$fd` uses the value, `{fd}>&-` closes. Mixing these up causes confusing errors.

6. **`/dev/fd/N` reopens the file:** Opening `/dev/fd/N` is NOT a dup — it reopens the original file. This means you get a new file descriptor with its own offset. If the original FD was a pipe, reopening is impossible and returns an error.

7. **FD limit exhaustion in loops:** `for i in {1..10000}; do exec $i>/dev/null; done` will quickly exhaust the FD limit. You can't open more than `ulimit -n` FDs.

8. **`exec N>&M-` fails silently in older bash:** The "move" syntax requires bash 4.3+. In older versions, `exec N>&M-` closes M but does NOT dup N. Test for support.

9. **FIFO deadlock with write-only:** `exec 3>fifo` blocks waiting for a reader. Use `exec 3<>fifo` (bidirectional) to open without blocking.

10. **`set -e` and failing exec:** `exec 3</nonexistent` with `set -e` exits the script. But `exec 3</nonexistent || true` won't because the `||` suppresses the error.

11. **Symlink race in /tmp:** If you use `exec 3>/tmp/lock`, an attacker could symlink `/tmp/lock` to `/etc/shadow` before you open it. Use a private directory or `mktemp`.

12. **Bash < 4.1 doesn't have `{var}`:** macOS ships bash 3.2. You can't use dynamic FD assignment there. Fall back to manual FD numbers above 9.

13. **`exec` without arguments does nothing:** `exec` with no redirections does nothing. This is NOT the same as `exec command` which replaces the shell process.

14. **`read` and EOF from a custom FD:** `read -r line <&3` reads one line. If FD 3 is at EOF, read returns non-zero. But subsequent reads still return non-zero.

15. **The `0<&-` trap:** Closing stdin with `exec 0<&-` makes subsequent `read` calls fail immediately with EOF. Your script will consume 100% CPU in a read loop.

## See It In The Wild

### Exploring `/proc/self/fd`
```bash
$ ls -la /proc/self/fd/
lrwx------ 1 user user 64 Jul 31 10:00 0 -> /dev/pts/3
lrwx------ 1 user user 64 Jul 31 10:00 1 -> /dev/pts/3
lrwx------ 1 user user 64 Jul 31 10:00 2 -> /dev/pts/3
```

Open a new FD and watch it appear:
```bash
$ exec 4</etc/hostname
$ ls -la /proc/self/fd/
...
lrwx------ 1 user user 64 Jul 31 10:00 4 -> /etc/hostname
$ exec 4<&-
```

### Tracing exec with strace
```bash
$ strace -e trace=openat,close,dup2,fcntl bash -c '
exec 3</etc/hostname
exec 4>/tmp/out.txt
cat <&3 >&4
exec 3<&- 4<&-
'
```

### System-wide FD monitoring
```bash
$ cat /proc/sys/fs/file-nr
3840    0       9223372036854775807
```
Column 1: allocated FDs. Column 2: unused (almost always 0). Column 3: system max.

### Finding processes with open FDs to a specific file
```bash
$ fuser /var/log/syslog
/var/log/syslog:  1234  5678
```

### Using `lsof` to inspect FD activity
```bash
$ lsof -d 0-10 -p $$
COMMAND PID  USER   FD   TYPE DEVICE SIZE/OFF   NODE NAME
bash    1234 user    0u  CHR  136,3      0t0   1234 /dev/pts/3
bash    1234 user    1u  CHR  136,3      0t0   1234 /dev/pts/3
bash    1234 user    2u  CHR  136,3      0t0   1234 /dev/pts/3
```

### Checking `ulimit` for FD limits
```bash
$ ulimit -a | grep -i "open files"
open files                      (-n) 1024
$ ulimit -n 4096
$ ulimit -n
4096
```

The hard limit (set by root) caps this:
```bash
$ ulimit -Hn
1048576
```

## Check Your Understanding

1. What is the difference between `exec 4>file` and `exec 4>>file` in terms of kernel-level open flags?

2. Why does `echo "hello" >&4` followed by `cat <&4` return nothing, even though you just wrote "hello" to the file?

3. What happens to a lock acquired by `flock -n 6` if the script is killed with `kill -9`?

4. How does bash's `{var}>file` handle O_CLOEXEC differently from `exec N>file`? Why does this matter for security?

5. If process A has FD 5 open to a file, and it forks child B, does B inherit FD 5? Does B's FD 5 share the same file offset?

6. What is the output of `exec 5>&1; exec 1>/dev/null; echo "hi"; exec 1>&5; exec 5<&-; echo "there"`? Trace the FD flow.

7. Why can't you do `stat /dev/tcp/example.com/80` but you can do `exec 3<>/dev/tcp/example.com/80`?

8. What is the difference between `exec N<&M` and `exec N<&M-`? When would you use the latter?

9. How would you check if a bash script has any FD leaks? What tools would you use?

10. Why does opening a FIFO with `exec 3>fifo` block, but `exec 3<>fifo` doesn't? What does the kernel do differently?
