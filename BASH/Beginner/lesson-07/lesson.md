# Lesson 7: Pipes Deep Dive

## History & Origins

The Unix pipe is arguably the single most important innovation in the history of operating systems. The concept was proposed by Douglas McIlroy at Bell Labs in 1964 — years before Unix existed. He wrote a memo suggesting that programs should be like "a silk thread" that could be connected together:

> "We should have some way of coupling programs like garden hose. Screw in another segment when it becomes necessary to massage the data in another way."

This vision was realized in Unix via the `pipe()` system call, implemented by Ken Thompson in Version 3 Unix (1973). The syntax `cmd1 | cmd2` was introduced in the Bourne shell (1977), but the `|` character was already used for pipes in the Thompson shell.

The original implementation created a temporary file for each pipe. Data was written to a file, then read back. This was slow. The current implementation uses an in-memory buffer managed by the kernel, avoiding disk I/O entirely.

The `|` symbol was chosen because it was on the keyboard (shift-backslash on most terminals) and wasn't used for anything else crucial. On some early keyboards, it appeared as a broken vertical bar (`¦`).

The concept of `PIPESTATUS` — tracking exit codes of all commands in a pipeline — was a late addition to bash (2.0, 1996). Before that, only the exit code of the last command in the pipeline was available.

The `|&` shorthand for piping both stdout and stderr was added in bash 4.0 (2009), inspired by similar features in zsh and ksh.

**Fun anecdote:** The famous Unix philosophy "Write programs that do one thing and do it well" was codified by Peter Salus quoting Doug McIlroy, but pipes are the engine that makes it practical. Without pipes, small reusable tools are useless — you'd have to write custom scripts for every workflow. Pipes make the sum greater than the parts. Also, the "garden hose" metaphor is so old that modern developers might not recognize it — but it's still the best description of pipes ever written.

## Syntax Reference

### Basic Pipe

```bash
cmd1 | cmd2              # Pipe stdout of cmd1 to stdin of cmd2
```

### Pipe Both Streams

```bash
cmd1 |& cmd2             # Pipe both stdout and stderr (bash 4+)
cmd1 2>&1 | cmd2         # Same, portable
```

### Pipeline Exit Codes

```bash
${PIPESTATUS[@]}         # Array of exit codes, one per pipeline command
${PIPESTATUS[0]}         # Exit code of first command in pipeline
${PIPESTATUS[-1]}        # Exit code of last command (same as $?)
$?                       # Exit code of last command in pipeline (same as ${PIPESTATUS[-1]})
```

### Named Pipes (FIFOs)

```bash
mkfifo mypipe            # Create a named pipe
cmd1 > mypipe &          # Write to pipe in background
cmd2 < mypipe            # Read from pipe
rm mypipe                # Remove the named pipe
```

### `pv` — Pipe Viewer (progress bar)

```bash
pv file | cmd            # Show progress of file through pipe
cmd1 | pv | cmd2         # Show rate of data flow
pv -c -N "name" file     # Named progress indicator
```

### `xargs` — Build and Execute Command Lines from Input

```bash
cmd1 | xargs cmd2        # Pass input as arguments to cmd2
cmd1 | xargs -n 1 cmd2   # One argument per invocation
cmd1 | xargs -P 4 cmd2   # Run up to 4 processes in parallel
```

### `parallel` — Run Jobs in Parallel

```bash
cmd1 | parallel cmd2     # GNU parallel, similar to xargs but more powerful
```

## Under the Hood

### What happens when you run `cmd1 | cmd2`

1. **Shell creates a pipe**: The `pipe(int pipefd[2])` syscall creates two file descriptors: `pipefd[0]` (read end) and `pipefd[1]` (write end). The kernel allocates an in-memory buffer (typically 64KB on Linux, can be queried with `fcntl(fd, F_GETPIPE_SZ)`).

2. **Shell forks twice** (or uses `fork()` twice from the same process): creates child1 for `cmd1` and child2 for `cmd2`.

3. **In child1** (will run `cmd1`):
   - `dup2(pipefd[1], 1)` — makes stdout (fd 1) write to the pipe's write end.
   - `close(pipefd[0])` — closes read end (not needed).
   - `close(pipefd[1])` — closes original write end (fd 1 has a copy).
   - `execve("cmd1", ...)` — runs command.

4. **In child2** (will run `cmd2`):
   - `dup2(pipefd[0], 0)` — makes stdin (fd 0) read from the pipe's read end.
   - `close(pipefd[1])` — closes write end (not needed).
   - `close(pipefd[0])` — closes original read end (fd 0 has a copy).
   - `execve("cmd2", ...)` — runs command.

5. **Parent shell**: Closes both ends of the pipe. Waits for both children.

6. **Data flow**: `cmd1` writes to stdout -> kernel pipe buffer -> `cmd2` reads from stdin.

### Kernel pipe buffer

The pipe buffer is a circular buffer in kernel memory:
- Default size: 65536 bytes (64KB) on most Linux systems.
- Configurable via `fcntl(fd, F_SETPIPE_SZ, size)` up to `/proc/sys/fs/pipe-max-size`.
- Writing to a full pipe blocks the writer until the reader consumes data.
- Reading from an empty pipe blocks the reader until the writer produces data.
- When the writer closes the pipe, the reader gets EOF (read returns 0).
- When the reader closes the pipe, the writer gets SIGPIPE (if it tries to write further).

### SIGPIPE: The silent killer

When a reader closes a pipe before the writer is done:
1. The writer's next `write()` call returns -1 with `errno = EPIPE`.
2. The kernel also sends `SIGPIPE` to the writing process.
3. Default SIGPIPE behavior: terminate the process.
4. This is why `head | cmd` works: after `head` exits, the upstream command gets SIGPIPE and stops.

Example: `yes | head -5`
- `yes` writes "y" repeatedly to stdout (never stops on its own).
- `head -5` reads 5 lines, then exits.
- The pipe closes (reader is gone).
- `yes` writes "y" again -> SIGPIPE -> `yes` terminates.
- Without SIGPIPE, `yes` would run forever producing output that nobody reads.

### Process model

- Each command in a pipeline runs in its own process (except shell builtins).
- All processes in a pipeline are children of the shell.
- The pipeline's processes run concurrently (not one after another).
- The pipe buffer allows them to proceed at different speeds.

### Pipe vs redirect

```bash
cmd1 > file              # cmd1 runs, writes to file, exits
cmd1 | cmd2              # cmd1 and cmd2 run CONCURRENTLY
```

Key difference: redirection is sequential (cmd1 writes, then later you read), while pipes are concurrent (cmd1 produces while cmd2 consumes). This is why `grep pattern hugefile | head -5` is fast — grep stops early (via SIGPIPE) once head has what it needs.

## Core Examples

### Example 1: Classic frequency pipeline

```bash
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
     17 /bin/bash
      8 /usr/sbin/nologin
      3 /bin/sh
      2 /usr/bin/fish
```

**Step by step:**
1. `cut -d: -f7 /etc/passwd` extracts the shell field from each passwd entry.
2. `sort` sorts all shell paths alphabetically (required for uniq).
3. `uniq -c` counts adjacent duplicates, prefixing each unique line with count.
4. `sort -rn` sorts the count lines numerically descending.
5. Result: most common shell at top.

**What if:**
```bash
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn | head -3   # Top 3 only
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -n | tail -3    # Bottom 3
```

### Example 2: Grep in a pipeline

```bash
$ ps aux | grep "systemd" | grep -v grep | head -5
root         1  0.0  0.1 167936 11964 ?  Ss  01:00  0:00 /sbin/init
root       250  0.0  0.0  24508  3980 ?  Ss  01:00  0:00 /lib/systemd/systemd-journald
```

**Step by step:**
1. `ps aux` lists all processes. Outputs to stdout.
2. `grep "systemd"` filters lines containing "systemd".
3. `grep -v grep` removes the grep process itself (which contains "systemd" in its command line).
4. `head -5` shows first 5 matches.
5. `ps aux` and all greps run concurrently. As soon as `head -5` has 5 lines, it exits, causing SIGPIPE back through the chain.

**What if:**
```bash
$ ps aux | grep "[s]ystemd"     # Alternative to grep -v grep: regex trick
$ pgrep -a systemd               # Better: use pgrep instead of ps | grep
```

### Example 3: PIPESTATUS — pipeline exit codes

```bash
$ ls /etc/passwd | grep "root" | wc -l
1
$ echo ${PIPESTATUS[@]}
0 0 0

$ ls /nonexistent | grep "foo" | wc -l
ls: cannot access '/nonexistent': No such file or directory
0
$ echo ${PIPESTATUS[@]}
2 1 0
```

**Step by step:**
1. First pipeline: all commands succeed (exit 0).
2. Second pipeline: `ls` fails (exit 2 — file not found), `grep` fails (exit 1 — no match because stdin was empty), `wc -l` succeeds (exit 0 — counted 0 lines).
3. `${PIPESTATUS[@]}` captures the exit codes of each command in order.

**What if:**
```bash
$ ls /nonexistent | grep "foo" | wc -l
$ echo ${PIPESTATUS[0]} ${PIPESTATUS[1]} ${PIPESTATUS[2]}  # Individual access
2 1 0
$ echo $?    # Same as ${PIPESTATUS[-1]} or ${PIPESTATUS[2]}
0
```

### Example 4: Pipe both streams

```bash
$ find /root -name "*.conf" |& grep -i permission
find: '/root': Permission denied
```

**Step by step:**
1. `find /root -name "*.conf"` produces both stdout and stderr (search results + permission errors).
2. `|&` pipes BOTH streams to `grep`.
3. `grep -i permission` filters for lines containing "permission" (case-insensitive).
4. Without `|&`, the error would go to the terminal, bypassing the pipe.

**What if:**
```bash
$ find /root -name "*.conf" 2>&1 | grep -i permission  # Portable version
$ find /root -name "*.conf" 2>/dev/null | grep -i permission  # Ignore errors
```

### Example 5: xargs — building commands from input

```bash
$ cut -d: -f1 /etc/passwd | head -5 | xargs echo "Users:"
Users: root daemon bin sys sync
```

**Step by step:**
1. `cut` extracts the first 5 usernames.
2. `head -5` limits to 5 lines.
3. `xargs echo "Users:"` reads lines from stdin and appends them to the command `echo "Users:"`.
4. Result: `echo "Users:" root daemon bin sys sync`.

**What if:**
```bash
$ cut -d: -f1 /etc/passwd | head -5 | xargs -n 1 echo "User:"
User: root
User: daemon
User: bin
User: sys
User: sync

$ cut -d: -f1 /etc/passwd | head -5 | xargs -I{} echo "Hello, {}!"
Hello, root!
Hello, daemon!
Hello, bin!
Hello, sys!
Hello, sync!
```

### Example 6: Named pipes

```bash
$ mkfifo /tmp/mypipe
$ ls /etc > /tmp/mypipe &   # Writer in background
[1] 12345
$ wc -l < /tmp/mypipe       # Reader
245
[1]+  Done
```

**Step by step:**
1. `mkfifo /tmp/mypipe` creates a named pipe (a FIFO — first in, first out).
2. `ls /etc > /tmp/mypipe &`: writer in background. This blocks until a reader also opens the pipe.
3. `wc -l < /tmp/mypipe`: reader opens the pipe. Now both are connected.
4. Data flows through the FIFO from writer to reader.
5. Writer completes (all data sent), closes pipe. Reader sees EOF, finishes.
6. Named pipes persist as files—you must `rm /tmp/mypipe` to remove them.

**What if:**
```bash
$ # Using named pipes for tricky setups
$ mkfifo /tmp/pipe1 /tmp/pipe2
$ cmd1 < /tmp/pipe1 > /tmp/pipe2 &   # Bidirectional
$ cmd2 < /tmp/pipe2 > /tmp/pipe1 &   # Cross-connected
$ # This creates a bidirectional byte stream between cmd1 and cmd2
```

### Example 7: tee inside a pipeline

```bash
$ cat /etc/hosts | tee /tmp/hosts_capture.txt | wc -l
3
$ cat /tmp/hosts_capture.txt
127.0.0.1  localhost
127.0.1.1  desktop
::1        localhost ip6-localhost ip6-loopback
```

**Step by step:**
1. `cat /etc/hosts` outputs the file content.
2. Pipe to `tee`: writes content to `/tmp/hosts_capture.txt`, also outputs to stdout.
3. Pipe from `tee` to `wc -l` counts the lines.

**What if:**
```bash
$ cat /etc/hosts | tee /tmp/a.txt /tmp/b.txt | wc -l  # Multiple sink files
$ echo "data" | tee >(gzip > /tmp/data.gz) | wc -c    # Process substitution with tee
```

### Example 8: Pipeline variable scope issue

```bash
$ echo "hello" | read var; echo "$var"
# (empty line — var wasn't set in the parent shell!)
```

**Step by step:**
1. `echo "hello"` runs in a subshell, writes to pipe.
2. `read var` runs in a subshell (each pipe segment is a subshell in bash).
3. `var` is set in the subshell, but that subshell exits.
4. The parent shell's `$var` is unchanged.
5. `echo "$var"` shows nothing.

**Fix:**
```bash
$ read -r var <<< "hello"; echo "$var"
hello
$ # Or use process substitution:
$ read -r var < <(echo "hello"); echo "$var"
hello
```

### Example 9: Pipes with while loops

```bash
$ cat /etc/passwd | head -5 | while IFS=: read user shell; do
    echo "$user uses $shell"
done
root uses /bin/bash
daemon uses /usr/sbin/nologin
bin uses /usr/sbin/nologin
sys uses /dev/null
sync uses /bin/sync
```

**Step by step:**
1. `cat /etc/passwd | head -5` provides 5 lines.
2. `while IFS=: read user shell` reads each line, splitting on `:` into two variables.
3. `echo "$user uses $shell"` outputs the reformatted data.
4. Note: the while loop runs in a subshell because it's part of the pipeline. Variables set inside it are lost.

**What if:**
```bash
$ while IFS=: read user shell; do
    echo "$user uses $shell"
    count=$((count + 1))
done < <(cat /etc/passwd | head -5)
echo "Count: $count"    # Works! Process substitution doesn't use a subshell.

# vs pipeline version:
$ cat /etc/passwd | head -5 | while IFS=: read user shell; do
    count=$((count + 1))
done
echo "Count: $count"    # 0! The count was lost in the subshell.
```

### Example 10: Pipeline performance — head stops early

```bash
$ time (yes | head -1000000 | wc -c)
real    0m0.042s
```

**Step by step:**
1. `yes` writes "y" repeatedly. It would run forever.
2. `head -1000000` reads 1,000,000 lines, then exits.
3. After `head` exits, the pipe reader is gone. Next `write` from `yes` gets SIGPIPE.
4. `yes` dies. `wc -c` counts the bytes from the 1M lines.
5. Total time: 42ms. If `yes` ran until `wc -c` finished, it would be slower.

**What if:**
```bash
$ # Without head, this would be slower because yes keeps producing
$ yes | wc -c
^C     # Interrupt — it never stops!
```

### Example 11: Multiple pipes for complex filtering

```bash
$ cat /var/log/auth.log | grep "Failed password" | awk '{print $9}' | sort | uniq -c | sort -rn | head -5
    127 192.168.1.100
     45 10.0.0.50
     12 172.16.0.1
      3 203.0.113.5
      1 198.51.100.2
```

**Step by step:**
1. Read auth log.
2. Filter for "Failed password" lines.
3. Extract field 9 (IP address, typically).
4. Sort IPs alphabetically.
5. Count occurrences.
6. Sort by count descending.
7. Show top 5 attackers.

### Example 12: pv for progress monitoring

```bash
$ pv /var/log/syslog | grep "error"
# Shows progress bar: 12.5MiB 0:00:02 [5.21MiB/s] [=========> ] 45%
```

**Step by step:**
1. `pv` reads the file, measures throughput.
2. Writes data to stdout through the pipe.
3. On stderr (or terminal), displays progress: amount read, elapsed time, rate, ETA.
4. `grep` processes the data as it arrives.

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`dmesg | grep -i error | tail -20`** — recent kernel errors.
- **`journalctl -u sshd | grep 'Failed password' | wc -l`** — count SSH failures.
- **`ps aux --sort=-%mem | head -10`** — top 10 memory processes.
- **`du -sh /* | sort -rh | head -10`** — top 10 disk consumers.
- **`find / -type f -perm -4000 -ls 2>/dev/null | awk '{print $NF}'`** — list setuid binaries.

### 2. WITH the OS — Development

- **`git log --oneline | head -5`** — recent commits.
- **`npm test 2>&1 | tee test.log | grep -E 'FAIL|ERROR'`** — watch only errors during test.
- **`cat src/*.js | wc -l`** — total lines of code (rough).
- **`find . -name '*.py' | xargs wc -l | tail -1`** — total Python lines in project.
- **`docker logs mycontainer 2>&1 | tail -f`** — follow container logs.

### 3. AGAINST the OS — Exploitation

- **`nc -l -p 4444 | /bin/bash`** — simple reverse shell (reads commands from network).
- **`cat /etc/passwd | cut -d: -f1 | xargs -I{} echo "User: {}"`** — extract usernames.
- **`find / -perm -4000 2>/dev/null | xargs ls -la`** — enumerate setuid binaries for privilege escalation.
- **`tcpdump -i eth0 -w - | strings | grep password`** — sniff passwords from network.

### 4. FOR DEFENSE — Detection & Auditing

- **`last | awk '{print $1}' | sort | uniq -c | sort -rn`** — login frequency per user.
- **`cat /var/log/auth.log* | grep -E 'Accepted|Failed' | awk '{print $1, $2, $9}'`** — auth timeline.
- **`lsof -i :22 | grep ESTABLISHED | awk '{print $9}'`** — active SSH connections.
- **`find /etc -newer /etc/shadow -type f`** — files modified since shadow (potential backdoor).

## Memory Aids

### Mnemonics

- **`|`** = pipe. Looks like a vertical conduit connecting commands.
- **`|&`** = pipe AND stderr. The `&` means "and" (both streams).
- **`${PIPESTATUS[@]}`** = "pipe status array." The `@` means all elements.
- **`xargs`** = "execute arguments." Takes stdin and turns it into command-line arguments.
- **`tee`** = T-pipe. Splits flow in two directions.
- **`pv`** = "pipe viewer." Shows the data flowing through.

### "Unix Philosophy" in one line

```
cmd1 | cmd2 | cmd3 | cmd4 | cmd5
```

Each command does ONE thing. The pipe composes them. This is the Unix philosophy in action.

### Pipeline patterns to memorize

```
# Frequency analysis
cut -d: -fN file | sort | uniq -c | sort -rn

# Top N
command | sort -rn | head -N

# Filter and transform
grep pattern | cut -d, -fN | sort -u

# Watch for errors
tail -f log | grep -i error

# Extract and reformat
cat file | awk '{print $1, $3}' | tr '[:upper:]' '[:lower:]'
```

### Common confusions

- **Variables in pipelines**: They set in subshells and are lost. Use process substitution or here-strings instead.
- **`$?` vs `${PIPESTATUS[0]}`**: `$?` is only the LAST command's exit code. `${PIPESTATUS[@]}` gives all.
- **`|&` vs `2>&1 |`**: Same thing. `|&` is cleaner but bash-specific.
- **SIGPIPE is normal**: Don't treat "Broken pipe" errors as bugs — they're by design.
- **Pipes vs redirects**: Pipes are concurrent; redirects are sequential.

## Trap Vault

### Trap 1: Variables lost in pipeline subshells

**Problem:** Variables set inside pipelines disappear.

**Example:**
```bash
$ count=0
$ cat /etc/passwd | while IFS=: read user; do
    count=$((count + 1))
done
$ echo "Count: $count"
Count: 0    # Not incremented!
```

**Why:** Each segment of a pipeline (including the `while` loop) runs in a subshell. Variables set in a subshell don't propagate to the parent.

**Fix:**
```bash
$ count=0
$ while IFS=: read user; do
    count=$((count + 1))
done < /etc/passwd
$ echo "Count: $count"
Count: 45    # Works!

# Or use process substitution:
$ while IFS=: read user; do
    count=$((count + 1))
done < <(cat /etc/passwd)
```

### Trap 2: PIPESTATUS resets immediately

**Problem:** `PIPESTATUS` only holds the LAST pipeline's exit codes.

**Example:**
```bash
$ ls /etc | grep root | wc -l
1
$ echo "PIPESTATUS: ${PIPESTATUS[@]}"
PIPESTATUS: 0 0 0
$ echo "Status: $?"
Status: 0
$ echo "PIPESTATUS: ${PIPESTATUS[@]}"   # Lost! Now it's from the echo pipeline (which succeeded)
PIPESTATUS: 0
```

**Why:** Every command updates `$?`. Every pipeline updates `PIPESTATUS`. The next command overwrites it.

**Fix:** Capture immediately:
```bash
$ ls /etc | grep root | wc -l
1
$ result=${PIPESTATUS[@]}
$ echo "Pipeline exits: $result"
Pipeline exits: 0 0 0
```

### Trap 3: `grep -q` causes SIGPIPE

**Problem:** `grep -q` exits after first match, killing the upstream.

**Example:**
```bash
$ cat hugefile | grep -q "needle"
# Works fine, but upstream (cat) is killed by SIGPIPE.
$ cat hugefile | grep -q "needle" 2>&1
cat: write error: Broken pipe
```

**Why:** `grep -q` exits immediately on first match. The pipe breaks. `cat` gets SIGPIPE when it tries to write more. This isn't a real error — `cat` was just told to stop.

**Fix:** Ignore the Broken pipe message:
```bash
$ cat hugefile 2>/dev/null | grep -q "needle"   # Suppress stderr from cat
$ grep -q "needle" hugefile                      # Even better: no pipe at all!
```

### Trap 4: `xargs` with special characters

**Problem:** `xargs` splits on spaces/tabs/newlines, breaking with quoted filenames.

**Example:**
```bash
$ printf "file1.txt\nfile with spaces.txt\nfile2.txt\n" | xargs cat
cat: file: No such file or directory
cat: with: No such file or directory
cat: spaces.txt: No such file or directory
```

**Why:** `xargs` by default splits on whitespace, treating each word as a separate argument. "file with spaces.txt" becomes three arguments.

**Fix:**
```bash
$ printf "file1.txt\nfile with spaces.txt\nfile2.txt\n" | xargs -0 cat
# But this requires NUL-delimited input!
$ find . -name '*.txt' -print0 | xargs -0 cat   # Use find -print0
```

### Trap 5: `head` in a pipeline truncates early

**Problem:** `head` exits after N lines, which can cause "broken pipe" or incomplete processing.

**Example:**
```bash
$ seq 1 1000000 | head -5
1
2
3
4
5
$ # The seq process is killed by SIGPIPE. No error shown.
$ seq 1 1000000 2>/dev/null | head -5   # Silences write error if any
```

**Why:** This is by design. `head` reads what it needs and exits. The upstream gets SIGPIPE. This is efficient.

**Fix:** None needed — this is correct behavior. Just be aware that `seq` or whatever upstream process may not complete, which is fine if you don't need it to.

### Trap 6: `xargs` argument limits

**Problem:** `xargs` can pass too many arguments to a command.

**Example:**
```bash
$ find /usr -name '*.h' | xargs grep "ERROR"
# This works because xargs splits into batches (limited by ARG_MAX)
$ find /usr -name '*.h' | xargs -n 1 grep "ERROR"
# This runs grep once per file — slower but more predictable
```

**Why:** `xargs` defaults to running the command with as many arguments as fit within ARG_MAX. It splits the input into batches. Each batch runs a separate command invocation.

**Fix:** Use `-n N` to control batch size. Use `-P N` to run batches in parallel.

### Trap 7: Pipes and built-in commands

**Problem:** Some shell builtins behave differently in pipelines.

**Example:**
```bash
$ echo "hello" | read var; echo "$var"   # Empty!
$ echo "hello" | echo "pipe"
pipe    # echo ignores stdin!
```

**Why:** Some builtins don't read stdin (like `echo`). Others, like `read`, run in a subshell in the pipeline and their variable assignment is lost.

**Fix:** Know which commands read stdin. `read`, `grep`, `sed`, `awk`, `sort`, `uniq`, `head`, `tail`, `tr`, `cut` all read stdin. `echo`, `printf`, `rm`, `cp`, `mv`, `cd`, `pwd` don't.

### Trap 8: SIGPIPE vs real errors

**Problem:** Distinguishing between a real crash and SIGPIPE.

**Example:**
```bash
$ cmd1 | cmd2
# If cmd1 exits with code 141 (128 + 13 = SIGPIPE), it was killed by pipe close.
# This is usually NOT an error — cmd2 just didn't need more input.
```

**Why:** When a process receives SIGPIPE (signal 13) and doesn't handle it, it terminates with exit code `128 + 13 = 141`. This is different from a crash (signal 6 = 134, signal 11 = 139).

**Fix:** Check `PIPESTATUS`. 141 is expected and normal in pipelines with `head`, `grep -q`, etc.
```bash
$ seq 1 1000 | head -5
$ echo ${PIPESTATUS[0]}    # May be 141
141
$ echo ${PIPESTATUS[1]}    # head: 0
0
```

### Trap 9: Named pipe deadlock

**Problem:** Writing and reading the same named pipe from the same process deadlocks.

**Example:**
```bash
$ mkfifo /tmp/deadlock
$ cat /tmp/deadlock > /tmp/out.txt    # Blocks waiting for writer
$ cat /tmp/data.txt > /tmp/deadlock   # Never reached! First cat is running.
# DEADLOCK — both processes are waiting.
```

**Why:** Opening a FIFO for reading blocks until a writer opens it (and vice versa). If you open for reading first (write is waiting to open), you can't get to the write command.

**Fix:** Open both ends simultaneously:
```bash
$ cat /tmp/data.txt > /tmp/deadlock &   # Writer in background
$ cat /tmp/deadlock                     # Reader (will see data)
# Or use the "both ends open" trick: exec 3<>/tmp/deadlock in a single process.
```

### Trap 10: `|&` compatibility

**Problem:** `|&` doesn't work in POSIX sh or older bash.

**Example:**
```bash
$ sh -c 'find /root |& grep denied'
sh: 1: Syntax error: "|&" unexpected
```

**Why:** `|&` is a bash 4.0+ extension. POSIX sh doesn't support it.

**Fix:**
```bash
$ sh -c 'find /root 2>&1 | grep denied'    # Portable
```

## See It In The Wild

### Where you encounter pipes daily

- **`git log --oneline | head`** — most common pipeline for developers.
- **`ps aux | grep myprocess`** — finding processes.
- **`dmesg | tail -20`** — recent kernel messages.
- **`history | grep docker`** — find past docker commands.
- **`echo $PATH | tr ':' '\n' | sort`** — sorted PATH entries.
- **`cat /proc/cpuinfo | grep 'model name' | uniq`** — CPU info.

### How to observe pipes with strace

```bash
# Watch pipe creation
$ strace -e trace=pipe2,pipe,clone,dup2 bash -c 'ls | wc -l' 2>&1

# See SIGPIPE in action
$ strace -e trace=write,kill bash -c 'yes | head -3' 2>&1
```

### Try this now

```bash
# Count your most-used commands
$ history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Check pipe buffer size
$ mkfifo /tmp/pipesize; echo "test" > /tmp/pipesize & cat /tmp/pipesize
$ fcntl -s /tmp/pipesize 2>/dev/null || echo "fcntl not available"
$ cat /proc/sys/fs/pipe-max-size
1048576

# Observe concurrent execution
$ (echo "start1"; sleep 2; echo "end1") | (echo "start2"; sleep 1; echo "end2")
start1
start2
end2
end1    # Note: output is interleaved, not sequential!
```

## Check Your Understanding

<details>
<summary>1. What is the difference between `cmd1 | cmd2` and `cmd1 2>&1 | cmd2`?</summary>

`cmd1 | cmd2` pipes only stdout. `cmd1 2>&1 | cmd2` redirects stderr to stdout before the pipe, so both stdout and stderr reach `cmd2`. In bash 4+, `cmd1 |& cmd2` does the same.
</details>

<details>
<summary>2. Why does `head -5 bigfile | wc -l` complete quickly, even though `bigfile` is huge?</summary>

`head -5` reads only 5 lines of `bigfile` and exits. The pipe reader (head) closes the pipe. When `bigfile` tries to write more data, it gets SIGPIPE and stops. `wc -l` only processes the 5 lines that made it through. This is the "early termination" optimization.
</details>

<details>
<summary>3. How would you capture the exit code of the second command in a 3-command pipeline?</summary>

Run the pipeline, then immediately access `${PIPESTATUS[1]}` (0-indexed, second command). For a 3-command pipeline `a | b | c`, `$?` is `c`'s exit, `PIPESTATUS[0]` is `a`'s, `PIPESTATUS[1]` is `b`'s.
</details>

<details>
<summary>4. Why does `cat file | grep foo` earn the "Useless Use of Cat" award?</summary>

Because `grep foo file` does the same thing without forking an extra `cat` process and creating an unnecessary pipe. The pipe variant forks, creates a pipe, schedules two processes — all for no benefit.
</details>

<details>
<summary>5. What happens when a named pipe (FIFO) has no reader?</summary>

The writer blocks in `open()` (if using blocking mode) or in `write()` (if the buffer fills up). By default, `open()` for writing on a FIFO blocks until a reader opens the read end. This is how FIFOs synchronize — the writer waits for the consumer.
</details>

<details>
<summary>6. What is the maximum size of the kernel pipe buffer on your system? How can you increase it?</summary>

Default is 64KB (65536 bytes). Check with `fcntl(fd, F_GETPIPE_SZ)` or `cat /proc/sys/fs/pipe-max-size`. Increase per-pipe with `fcntl(fd, F_SETPIPE_SZ, size)` up to `pipe-max-size`. The system limit can be changed with `sysctl -w fs.pipe-max-size=1048576`.
</details>

<details>
<summary>7. What's the difference between a pipeline and process substitution?</summary>

Pipeline: `cmd1 | cmd2` — connects stdout of cmd1 to stdin of cmd2. Both run concurrently. Process substitution: `diff <(cmd1) <(cmd2)` — runs commands and presents their output as file-like paths. Process substitution is for when a command expects file arguments rather than stdin.
</details>
