# Task 1: Multi-Stream Logger with Lock File & FD Deep Dive

## Objective
Build a production-grade logging script that simultaneously reads from one file, writes to another, logs different streams to separate files, implements a lock file mechanism, handles cleanup via traps, and performs FD introspection. This task exercises every `exec` concept from the lesson.

## Requirements

1. **Read from a source file** and write each line to an output file, prepending a line number
2. **Separate logging:** Send INFO-level messages to `/tmp/info.log`, ERROR-level messages to `/tmp/error.log`
3. **Lock file:** Before processing, acquire a lock via `flock` on a custom FD; exit gracefully if locked
4. **Resource cleanup:** Close all custom FDs at exit using a `trap`
5. **Dynamic FD assignment:** Use `exec {var}>file` for at least one FD
6. **FD introspection:** Print the current FD table before and after processing
7. **Error handling:** Detect and report write errors on output FDs

## Sub-tasks (7 cumulative)

### Task 1.1: Manual FD Assignment Skeleton
Write a script that:
- Opens FD 3 for reading `source.txt`
- Opens FD 4 for writing `output.txt`
- Uses FD 5 for lock file `/tmp/script.lock`
- Uses FD 6 for info logging, FD 7 for error logging
- All hardcoded (no dynamic FD assignment yet)

Approach comparison:
```bash
# Approach A: One exec per FD
exec 3<source.txt
exec 4>output.txt

# Approach B: Single exec with multiple redirections
exec 3<source.txt 4>output.txt 5>/tmp/script.lock

# Which is better? Approach B is more atomic — fewer system calls.
```

### Task 1.2: Dynamic FD Assignment
Replace FDs 6 and 7 with `exec {info_fd}>` and `exec {error_fd}>`. Print the assigned FD numbers.

```bash
exec {info_fd}>/tmp/info.log
echo "info_fd=$info_fd"   # Should be >= 10
```

**Note:** Dynamic FDs always start from the first available FD >= 10. They have O_CLOEXEC set.

### Task 1.3: Lock with flock
Implement lock file logic:
```bash
exec 5>/tmp/script.lock
flock -n 5 || { echo "Another instance is running" >&2; exit 1; }
```

What if the lock file doesn't exist? `exec 5>/tmp/script.lock` creates it.
What if the lock is held? `flock -n 5` exits with status 1, the `||` triggers.

**Multiple approaches compared:**

```bash
# Approach A: Non-blocking (preferred for scripts)
flock -n 5 || exit 1

# Approach B: Blocking with timeout
flock -w 5 5 || exit 1

# Approach C: Blocking indefinitely (dangerous for cron jobs)
flock 5
```

### Task 1.4: Trap-Based Cleanup
Register a cleanup function that closes all FDs:
```bash
cleanup() {
  exec 3<&- 4<&- 5<&- 2>/dev/null || true
  exec {info_fd}>&- {error_fd}>&- 2>/dev/null || true
}
trap cleanup EXIT INT TERM
```

**Why trap EXIT and INT and TERM?** EXIT covers normal exit, INT covers Ctrl+C, TERM covers `kill` signals. Without INT/TERM, an interrupted script leaves FDs open.

### Task 1.5: Line-by-Line Processing
Read from FD 3 and write to FD 4 with line numbers:
```bash
while IFS= read -r line <&3; do
  printf "%d: %s\n" "$((++n))" "$line" >&4
done
```

**Key details:**
- `IFS=` prevents stripping leading/trailing whitespace
- `-r` prevents backslash interpretation
- `>&4` writes to output FD
- `$((++n))` increments line counter

**What if the line is empty?** `read -r` still returns success (exit code 0) — it only fails on EOF.

### Task 1.6: FD Table Introspection
Before and after processing, print the current FD state:
```bash
echo "=== FD Table Before ==="
ls -la /proc/$$/fd/
echo "=== Processing ==="
...
echo "=== FD Table After ==="
ls -la /proc/$$/fd/
```

**Compare:** Using `lsof -p $$ -d 0-20` vs `ls -la /proc/$$/fd/` — `lsof` shows more detail (type, device, size), while `/proc` is faster and always available.

### Task 1.7: Error Detection and Reporting
Add error checking after write operations:
```bash
if ! echo "data" >&$info_fd; then
  echo "FATAL: Cannot write to info log" >&2
  exit 1
fi
```

**Alternative:** Redirect entire loop body to a file descriptor, reducing per-write checks.

## Complete Script Blueprint

```bash
#!/bin/bash

# Dynamic FD assignment for logs
exec {info_fd}>/tmp/info.log
exec {error_fd}>/tmp/error.log

# Lock file
exec 5>/tmp/script.lock
flock -n 5 || { echo "Already running" >&2; exit 1; }

# Cleanup
cleanup() {
  exec 3<&- 4<&- 5<&- {info_fd}>&- {error_fd}>&- 2>/dev/null || true
}
trap cleanup EXIT INT TERM

# Open input/output
exec 3<source.txt 4>output.txt

echo "[INFO] Processing source.txt..." >&$info_fd
n=0
while IFS= read -r line <&3; do
  printf "%d: %s\n" "$((++n))" "$line" >&4
  if [[ "$line" == *ERROR* ]]; then
    echo "[ERROR] Line $n: $line" >&$error_fd
  fi
done
echo "[INFO] Wrote $n lines to output.txt" >&$info_fd
```

## Bonus Challenges

1. **Bonus A:** Modify the script so that if FDs 3 or 4 fail to open (e.g., file doesn't exist), the script reports which FD failed and continues without FDs 3/4 if possible.

2. **Bonus B:** Add a `--verbose` flag that, when set, also echoes all INFO and ERROR lines to stderr.

3. **Bonus C:** Implement "FD rotation" — when info.log exceeds 1MB, close FD, rename to info.log.1, open new info.log on the same FD.

4. **Bonus D:** Use `exec {var}>file` for ALL FDs, including input — what happens? (Hint: you can't use `{var}<file` for exec in all cases — test it.)

5. **Bonus E:** Write a companion script `fd_watcher.sh` that runs in the background and outputs a message whenever a monitored process opens or closes an FD. Use `inotifywait` on `/proc/PID/fd/`.

## Hints

<details>
<summary>Hint 1: Combined exec</summary>

```bash
exec 3<source.txt 4>output.txt 5>/tmp/script.lock
```
</details>

<details>
<summary>Hint 2: Dynamic FD close</summary>

```bash
exec {info_fd}>&-  # No $ when closing
```
</details>

<details>
<summary>Hint 3: Checking FD validity</summary>

```bash
if ! true >&$fd 2>/dev/null; then
  echo "FD $fd is not writable"
fi
```
</details>

<details>
<summary>Hint 4: FD direction in /proc</summary>

In `/proc/PID/fd/`, a symlink like `4 -> /tmp/output.txt` doesn't show direction. Use `lsof -p PID -d 4` to see `4w` = write, `4r` = read, `4u` = read-write.
</details>

<details>
<summary>Hint 5: Trap propagation</summary>

Traps are inherited by subshells in some bash versions. Use `trap '' INT` before backgrounding to prevent Ctrl+C from killing children.
</details>

## Expected Output

```bash
$ cat > source.txt << 'EOF'
line one
line two ERROR: something broke
line three
EOF

$ ./multi_logger.sh
=== FD Table Before ===
total 0
lrwx------ ... 0 -> /dev/pts/3
lrwx------ ... 1 -> /dev/pts/3
lrwx------ ... 2 -> /dev/pts/3
lrwx------ ... 3 -> /home/user/source.txt
lrwx------ ... 4 -> /home/user/output.txt
lrwx------ ... 5 -> /tmp/script.lock
lrwx------ ... 10 -> /tmp/info.log
lrwx------ ... 11 -> /tmp/error.log

[INFO] Processing source.txt...
[INFO] Wrote 3 lines to output.txt

$ cat output.txt
1: line one
2: line two ERROR: something broke
3: line three

$ cat /tmp/info.log
[INFO] Processing source.txt...
[INFO] Wrote 3 lines to output.txt

$ cat /tmp/error.log
[ERROR] Line 2: line two ERROR: something broke

$ # Running two instances simultaneously:
$ ./multi_logger.sh &
$ ./multi_logger.sh
Another instance is running
```

## Deep Self-Check

1. **FD exhaustion simulation:** Write a script that intentionally exhausts the FD limit (ulimit -n 50) and runs the logger. What happens? How does the error manifest?

2. **Race condition analysis:** What happens if two instances of the script start at the exact same moment? Can both acquire the lock? (Answer: No — flock is atomic at the kernel level. One will succeed, the other will fail.)

3. **O_CLOEXEC verification:** Modify the script to run a child command (`ls -l /proc/self/fd`) during processing. Do dynamic FDs (10, 11) appear in the child? Do manual FDs (3, 4, 5) appear?

4. **SELinux context:** If SELinux is enforcing, what context would the log files have? Run `ls -Z /tmp/info.log`. Would a confined domain (like a systemd service) be able to write to `/tmp/`?

5. **Signal handling gap:** What happens if the script receives SIGKILL (cannot be caught)? Are the FDs closed? (Yes — kernel closes them, lock is released.)

6. **Performance analysis:** Count the number of `openat`, `write`, and `close` system calls made during a 1000-line run using `strace -c`. Which operation dominates?

7. **Hardcoded vs dynamic comparison:** Run the script with hardcoded FDs (3-7) and with dynamic FDs (10+). Use `strace` to count how many `fcntl(F_SETFD)` calls each makes. Dynamic should have more because it sets CLOEXEC.

8. **Empty source file test:** What does the script do if `source.txt` is empty? Does it still create `output.txt`? Does it still log?

9. **Binary file test:** What if `source.txt` contains null bytes? `read -r` reads up to the first null byte in some versions. Test with `printf '\x00hello\n' > source.txt`.

10. **`/dev/fd/` alternative:** Modify the script to use `/dev/fd/N` paths instead of `&N` redirections. For example, `cat /dev/fd/3` instead of `cat <&3`. Does this work identically? Why or why not?
