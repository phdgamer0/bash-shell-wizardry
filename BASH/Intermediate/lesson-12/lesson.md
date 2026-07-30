# Lesson 12: Process Management

## History & Origins

Process management in Unix dates to the **earliest versions** (1970s). The `fork()` and `exec()` syscalls are the foundation — they've been the Unix process creation model since day one. Job control (`bg`, `fg`, `jobs`) was introduced in the **C shell** (1978) and adopted by the **Korn shell** in the 1980s.

Signal handling comes from **Bell Labs** research (1970s), with the modern `signal(7)` API standardized in **POSIX.1-2001**. The idea of "signals" as software interrupts is one of Unix's most elegant concepts — a lightweight notification system between processes and the kernel.

The `trap` builtin came from the **Bourne shell** (1977), originally for catching signals. The `EXIT` trap (not a signal but a pseudo-signal) was a later innovation — it lets you guarantee cleanup runs regardless of how the script exits.

The naming: "jobs" = tasks the shell is managing. "Background" = running behind the scenes (not in the foreground terminal). "Disown" = dis-associate from ownership. "Nohup" = "no hang up" (ignore SIGHUP).

## Syntax Reference

### Job Control

| Command | Behaviour |
|---------|-----------|
| `cmd &` | Run `cmd` in background (subshell) |
| `jobs` | List background jobs with status |
| `jobs -l` | List jobs with PIDs |
| `jobs -p` | List PIDs only |
| `fg %N` | Bring job N to foreground |
| `bg %N` | Resume suspended job N in background |
| `kill %N` | Send signal to job N |
| `disown %N` | Remove job N from shell's job table |
| `disown -a` | Remove all jobs |
| `disown -h %N` | Mark job N to NOT receive SIGHUP when shell exits |
| `wait %N` | Wait for job N to finish |
| `wait` | Wait for all background jobs |
| `$!` | PID of the most recent background job |
| `$?` | Exit status of last foreground command |
| `$_` | Last argument of previous command |

### Signal Handling

| Signal | Number | Default Action | Can Trap? | Meaning |
|--------|--------|----------------|-----------|---------|
| SIGHUP | 1 | Terminate | Yes | Hangup (terminal disconnected, shell exiting) |
| SIGINT | 2 | Terminate | Yes | Interrupt (Ctrl+C) |
| SIGQUIT | 3 | Core dump | Yes | Quit (Ctrl+\\) |
| SIGKILL | 9 | Terminate | **No** | Kill (cannot be caught/ignored) |
| SIGTERM | 15 | Terminate | Yes | Termination (default `kill`) |
| SIGSTOP | 19 | Stop process | **No** | Stop (cannot be caught/ignored) |
| SIGCONT | 18 | Continue | Yes | Resume stopped process |
| SIGTSTP | 20 | Stop | Yes | Terminal stop (Ctrl+Z) |

### Trap

```
trap 'command' SIGNAL ...
trap 'command' EXIT ERR DEBUG RETURN
trap - SIGNAL ...          # Reset signal to default
trap -p SIGNAL             # Print current trap
```

| Pseudo-signal | Trigger |
|---------------|---------|
| `EXIT` | Script exits (any reason) |
| `ERR` | Any command returns non-zero (with `set -e` caveats) |
| `DEBUG` | Before every command |
| `RETURN` | Returning from a function or sourced script |

### Process Commands

| Command | Behaviour |
|---------|-----------|
| `kill PID` | Send SIGTERM (15) |
| `kill -SIGTERM PID` | Explicit signal name |
| `kill -9 PID` | SIGKILL — immediate death |
| `kill -0 PID` | Check process exists (no signal sent) |
| `pkill pattern` | Kill by process name |
| `pkill -f pattern` | Kill by full command line match |
| `pgrep pattern` | Find PIDs by name |
| `killall name` | Kill by executable name |
| `nohup cmd &` | Immune to SIGHUP |
| `cmd &` | Background |
| `cmd &|` | Background with stderr suppressed (bash 4.4+) |
| `coproc cmd` | Run in coprocess with I/O pipes |

## Under the Hood

### OS/kernel mechanisms

When you run `sleep 100 &`:

1. **Bash fork()s**: The shell calls `fork(2)` — a clone of the bash process. The child has PID `N+1`.
2. **Child exec()s**: The child calls `execvp("sleep", ["sleep", "100"])` — the bash code is replaced by `sleep`'s code.
3. **Parent tracks**: The parent (original bash) records the child PID in its internal job table.
4. **Job table**: Bash maintains a mapping: job number → PID → state (Running/Stopped/Done)

**strace reveals:**
```
clone(child_stack=NULL, flags=CLONE_CHILD_CLEARTID|CLONE_CHILD_SETTID|SIGCHLD) = 12345
# In child:
execve("/usr/bin/sleep", ["sleep", "100"], ...) = 0
# In parent:
wait4(12345, 0x7fff..., WNOHANG, NULL) = 0    (checks without blocking)
```

The `WNOHANG` flag is key — it lets bash check on background jobs without blocking.

### signaling

When you run `kill -9 12345`:
1. `kill(1)` sends `SIGKILL` via the `kill(2)` syscall
2. The kernel delivers SIGKILL directly — no userspace handler runs
3. The process is removed from the task list immediately
4. File descriptors are closed, memory freed, children orphaned

With `trap 'cleanup' INT`:
1. Ctrl+C sends SIGINT from the terminal driver to the foreground process group
2. The kernel delivers the signal to bash
3. Bash's signal handler runs — it sets a flag, doesn't interrupt execution immediately
4. At the next "safe point" (before executing the next command), bash runs `cleanup`

This is why `trap` in bash isn't truly preemptive — it checks for pending signals between commands.

### Process implications

- Background jobs share the same terminal — their output still appears on your console
- When the parent shell exits, it sends SIGHUP to all children in its job table
- `disown` removes entries from the job table, so SIGHUP is NOT sent
- Orphaned processes (parent dies) are re-parented to `init` (PID 1) on Linux

### Memory model

- Background tasks have their own address space (they're separate processes after fork/exec)
- The parent's job table is small — just PIDs and states
- `nohup` doesn't prevent termination — it just ignores SIGHUP
- `trap` handlers are stored in bash's internal signal table — minimal overhead

## Core Examples (8-12 minimum)

### Example 1: Background jobs and job control

**Command:**
```bash
sleep 30 &
sleep 40 &
sleep 50 &
jobs -l
```

**Output:**
```
[1]   12345 Running                 sleep 30 &
[2]-  12346 Running                 sleep 40 &
[3]+  12347 Running                 sleep 50 &
```

**Step-by-step:**
1. Three `sleep` commands launched in background with `&`
2. Each gets a job number: `[1]`, `[2]`, `[3]`
3. `+` marks the most recent job, `-` marks the second most recent
4. Jobs persist in the table until they finish or are disowned

**Variations:**
- `fg %2` brings job 2 to foreground — Ctrl+C kills it
- `bg %2` resumes a stopped job (one you Ctrl+Z'd) in background
- `kill %1` sends SIGTERM to job 1

### Example 2: disown — survive shell exit

**Command:**
```bash
sleep 120 &
disown %1
echo "Background task disowned. Exiting..."
exit
```

**Output:** Shell exits, but `sleep 120` keeps running (visible in `ps aux`)

**Step-by-step:**
1. `sleep 120 &` starts in background, registered in job table
2. `disown %1` removes it from the job table
3. When the shell exits, it doesn't send SIGHUP to job 1 (it's no longer tracked)
4. The sleep process continues running, re-parented to PID 1

**Variations:**
- `disown -h %1` — keep in job table but mark as "no SIGHUP"
- `disown -a` — disown all jobs
- Useful for GUI apps launched from terminal: `firefox & disown`

### Example 3: nohup — immunity from the start

**Command:**
```bash
nohup ./long_script.sh &
```

**Output:**
```
nohup: ignoring input and appending output to 'nohup.out'
```

**Step-by-step:**
1. `nohup` modifies the child process to ignore SIGHUP
2. If the parent shell exits, SIGHUP is sent but ignored
3. stdout is redirected to `nohup.out` if it was a terminal
4. stdin is redirected from `/dev/null`

**Variations:**
- `nohup cmd & disown` — belt and suspenders
- `nohup cmd > my.log 2>&1 &` — control output location
- `nohup` is POSIX, `disown` is bash-specific

### Example 4: kill with different signals

**Command:**
```bash
sleep 500 &
pid=$!
echo "PID: $pid"
kill -15 $pid          # SIGTERM — graceful
echo "Sent SIGTERM"
sleep 500 &
pid2=$!
echo "PID2: $pid2"
kill -9 $pid2          # SIGKILL — immediate
echo "Sent SIGKILL"
```

**Output:**
```
PID: 12345
Sent SIGTERM
PID2: 12346
Sent SIGKILL
```

**Step-by-step:**
1. First sleep gets SIGTERM — it gets to clean up (though `sleep` doesn't trap it)
2. Second sleep gets SIGKILL — kernel terminates it immediately
3. No cleanup handlers run for SIGKILL. Ever.

**Variations:**
- `kill -2 PID` = SIGINT (like Ctrl+C)
- `kill -HUP PID` = SIGHUP (often used to reload configs)
- `kill -0 PID` — check existence without killing

### Example 5: trap for cleanup

**Command:**
```bash
#!/bin/bash
cleanup() {
    echo "[$(date)] Cleaning up temporary files..."
    rm -f /tmp/myapp_*.tmp
    echo "Done."
    exit 0
}
trap cleanup SIGINT SIGTERM SIGHUP

echo "Working... (PID: $$)"
touch /tmp/myapp_$$.tmp
while true; do
    echo "Processing..."
    sleep 2
done
```

**Input:** Ctrl+C after a few seconds

**Output:**
```
Working... (PID: 54321)
Processing...
Processing...
^C[Thu Jul 31 12:00:05 UTC 2026] Cleaning up temporary files...
Done.
```

**Step-by-step:**
1. Script sets `trap cleanup SIGINT SIGTERM SIGHUP` — these three signals run `cleanup`
2. `touch` creates a temp file
3. Loop runs until interrupted
4. Ctrl+C sends SIGINT → `cleanup()` runs → temp file removed

**Variations:**
- `trap 'rm -f /tmp/*.tmp; exit' INT` — inline cleanup (no function)
- Multiple traps on same signal: last one wins (unless you use `trap -p`)

### Example 6: trap EXIT

**Command:**
```bash
#!/bin/bash
cleanup() {
    echo "Script ended at $(date)"
    echo "Temp files cleaned."
}
trap cleanup EXIT
echo "Doing work..."
sleep 1
echo "More work..."
exit 42
```

**Output:**
```
Doing work...
More work...
Script ended at Thu Jul 31 12:00:02 UTC 2026
Temp files cleaned.
```

**Step-by-step:**
1. `trap cleanup EXIT` — `cleanup()` runs on ANY exit (normal, error, signal)
2. Script runs normally
3. `exit 42` triggers the EXIT trap
4. `cleanup()` runs with the exit code still set (you can access `$?` inside it)

**Variations:**
- EXIT trap runs even after SIGINT if you don't trap INT separately
- EXIT trap exit code: `trap 'rc=$?; cleanup; exit $rc' EXIT` preserves exit code

### Example 7: pkill by pattern

**Command:**
```bash
# Terminal 1
sleep 300 &

# Terminal 2
pkill -f "sleep 300"
```

**Output:** The sleep in terminal 1 is terminated

**Step-by-step:**
1. `pkill -f` matches the full command line against the pattern
2. Without `-f`, it matches only process name (not arguments)
3. `pkill` sends SIGTERM by default
4. All matching processes are killed

**Variations:**
- `pkill -9 -f pattern` — force kill
- `pgrep -f pattern` — preview matches before pkill
- `pkill -x pattern` — exact match (name must equal pattern)

### Example 8: wait for background jobs

**Command:**
```bash
#!/bin/bash
worker() {
    local id=$1
    echo "Worker $id starting..."
    sleep $((id * 2))
    echo "Worker $id done."
    return $((id * 10))
}

worker 1 &
p1=$!
worker 2 &
p2=$!
worker 3 &
p3=$!

echo "Waiting for workers..."
wait $p1; echo "Worker 1 exited: $?"
wait $p2; echo "Worker 2 exited: $?"
wait $p3; echo "Worker 3 exited: $?"
echo "All done."
```

**Output:**
```
Worker 1 starting...
Worker 2 starting...
Worker 3 starting...
Waiting for workers...
Worker 1 done.
Worker 1 exited: 10
Worker 2 done.
Worker 2 exited: 20
Worker 3 done.
Worker 3 exited: 30
All done.
```

**Step-by-step:**
1. Three workers start in background, PIDs stored in `$p1`, `$p2`, `$p3`
2. `wait $p1` blocks until worker 1 finishes
3. `$?` captures the worker's exit code (set by `return`)
4. Sequential waiting — but all workers run in parallel
5. Without `wait`, the script would exit before workers finish

**Variations:**
- `wait` (no argument) waits for ALL background jobs
- `wait -n` (bash 4.3+) waits for ANY job to finish
- `wait -f` (bash 5.1+) waits for all, including `disown`ed jobs

### Example 9: trap DEBUG for step-through

**Command:**
```bash
#!/bin/bash
trap 'echo "[DEBUG] Line $LINENO: $BASH_COMMAND"' DEBUG
x=5
y=10
echo $((x + y))
```

**Output:**
```
[DEBUG] Line 5: x=5
[DEBUG] Line 6: y=10
[DEBUG] Line 7: echo 15
15
```

**Step-by-step:**
1. `trap ... DEBUG` runs the command BEFORE every subsequent command
2. `$LINENO` gives the current line number
3. `$BASH_COMMAND` holds the exact command about to execute
4. After the last command, the DEBUG trap stops firing

**Variations:**
- DEBUG trap skips commands inside `trap` handlers
- Can be used for a primitive step debugger

### Example 10: Coprocess (coproc)

**Command:**
```bash
coproc MYPROC { tr 'a-z' 'A-Z'; }
echo "hello world" >&${MYPROC[1]}
read -u ${MYPROC[0]} line
echo "Got: $line"
```

**Output:** `Got: HELLO WORLD`

**Step-by-step:**
1. `coproc` starts `tr 'a-z' 'A-Z'` in background with a two-way pipe
2. `MYPROC[0]` is the read fd, `MYPROC[1]` is the write fd
3. We write `"hello world\n"` to the coprocess's stdin
4. We read the transformed output from its stdout

**Variations:**
- Coprocesses are rare but powerful for client-server patterns in bash
- Name can be omitted: `coproc { cmd; }`

### Example 11: Graceful multi-worker shutdown

**Command:**
```bash
#!/bin/bash
pids=()
cleanup() {
    echo "Shutting down workers..."
    kill "${pids[@]}" 2>/dev/null
    sleep 2
    for pid in "${pids[@]}"; do
        kill -0 "$pid" 2>/dev/null && { echo "Killing $pid with SIGKILL"; kill -9 "$pid"; }
    done
    echo "All workers stopped."
}
trap cleanup EXIT SIGINT SIGTERM

for i in {1..3}; do
    ( while true; do echo "Worker $i alive"; sleep 3; done ) &
    pids+=($!)
done

echo "Workers: ${pids[*]}"
wait
```

**Output:** (varies, but shows graceful then force-kill)

**Step-by-step:**
1. Creates 3 workers in background, each looping forever
2. Stores PIDs in array
3. On Ctrl+C or exit: SIGTERM all workers first
4. Wait 2 seconds for graceful shutdown
5. Check each with `kill -0` — still alive? Send SIGKILL

**Variations:**
- Add `trap '' SIGINT` in workers so they ignore Ctrl+C directly
- Use a PID file for each worker
- Send different signals for different purposes

### Example 12: kill -0 for health check

**Command:**
```bash
#!/bin/bash
pid=$1
if kill -0 "$pid" 2>/dev/null; then
    echo "Process $pid is running"
    ps -p "$pid" -o pid,state,comm
else
    echo "Process $pid is not running"
fi
```

**Input:** `./check_pid.sh $$`

**Output:**
```
Process 12345 is running
  PID STAT COMMAND
12345 S    bash
```

**Step-by-step:**
1. `kill -0 PID` sends signal 0 — no signal is actually sent
2. The syscall checks if the process exists and we have permission to signal it
3. Returns 0 (success) if process exists, 1 if not
4. 2>/dev/null suppresses "No such process" error messages

**Variations:**
- `kill -0 $pid` inside loops for health monitoring
- Works on zombie processes too — they still exist until reaped

## Real-World Use Cases

### FOR the OS

- **Daemon management scripts**: Start/stop/restart with PID files and graceful shutdown
- **Parallel processing**: Fork workers for batch jobs, image processing, data pipeline stages
- **Log rotation**: Send SIGHUP to syslog/rsyslog after rotating logs
- **Service supervision**: Monitor and restart crashed services (simple init replacement)
- **Timeout enforcement**: Run a command with a timer and kill it if it exceeds time

### WITH the OS

- **`pkill -STOP -u username`** — freeze all processes for a user
- **`nohup make &`** — long compilation survives terminal close
- **`trap 'echo "$(date): $USER logged out"' EXIT`** — log user sessions
- **`coproc` for interactive CLI tools**: Embed `python` or `bc` as a persistent REPL

### AGAINST the OS (Security Perspective)

- **Signal injection**: If an attacker can control a command run by a privileged script, they can send arbitrary signals. `pkill -f` with a loose pattern can kill unintended processes.
- **PID races**: The PID returned by `$!` could theoretically be reused if the process exits quickly — classic TOCTOU (Time of Check, Time of Use) race.
- **`disown` and orphaned tasks**: Attackers can use disown to make malicious processes survive the user's logout, persisting in the process table.
- **`trap` for stealth**: A malicious script can trap SIGINT and SIGTERM to hide its behavior — when an admin tries to kill it, the trap runs and the process keeps going (though `kill -9` still works).
- **`nohup` for persistence**: `nohup ./backdoor & disown` — malware uses this to survive terminal disconnection.
- **`kill -0` for discovery**: An attacker can probe for running processes they might not otherwise see by scanning PID ranges with `kill -0`.

### FOR DEFENSE

- **Use PID files with locking**: Write PID to a file with `flock` to prevent race conditions
- **Always trap EXIT** for cleanup — it runs even if the admin kills your script
- **Use `pgrep -f` before `pkill`** to preview matches
- **Validate PIDs before signaling**: Check `/proc/$pid` exists
- **Limit SIGKILL usage**: Try SIGTERM first, wait, then SIGKILL
- **Run workers with different user accounts**: Limit blast radius of a compromised process

## Memory Aids

- **`$!`** = "the last one!" (the most recent background PID)
- **`$?`** = "how was that?" (exit status)
- **`disown`** = "dis-own" = "you're not my child anymore"
- **`nohup`** = "no hangup" = "don't hang up on me"
- **SIGHUP** = historically: terminal line hang-up (modem disconnected)
- **SIGKILL (9)** = "kill with 9 lives — you can't escape"
- **`trap`** = like an animal trap — when the signal triggers, the handler snaps

## Trap Vault (8-12 traps)

### Trap 1: kill -9 as first resort

**Problem:** Process doesn't clean up, leaves corrupted state.

**Bad Example:**
```bash
kill -9 $pid  # immediately
```

**Root Cause:** SIGKILL cannot be caught. The process gets no chance to close files, release locks, or remove temp files.

**Fix:**
```bash
kill $pid            # SIGTERM
sleep 3
kill -0 $pid 2>/dev/null && kill -9 $pid  # only if still running
```

### Trap 2: Background jobs die when shell exits

**Problem:** Launched a long task in background, closed terminal, found it dead.

**Bad Example:**
```bash
# In SSH session:
./big_data_process.sh &
exit  # process dies!
```

**Root Cause:** Shell sends SIGHUP to background jobs on exit. They die.

**Fix:**
```bash
nohup ./big_data_process.sh &
# or
./big_data_process.sh & disown
```

### Trap 3: Forgetting wait in scripts

**Problem:** Script finishes before background jobs complete.

**Bad Example:**
```bash
#!/bin/bash
sleep 5 &
echo "Done"  # prints immediately, script exits before sleep finishes
```

**Root Cause:** `wait` is not called. The script exits immediately after launching background jobs. Background jobs are terminated when the script exits (by default).

**Fix:**
```bash
sleep 5 &
wait
echo "Done"  # prints after sleep finishes
```

### Trap 4: Trap inside a function

**Problem:** Trap set inside a function doesn't work as expected.

**Bad Example:**
```bash
start_worker() {
    trap 'echo "Caught signal"' INT
    while true; do sleep 1; done
}
start_worker &
kill -INT $!  # trap handler doesn't run!
```

**Root Cause:** The trap was set in the parent shell, then the function was backgrounded. The trap setting doesn't propagate through fork/exec properly in all cases.

**Fix:** Set the trap inside the subshell or before backgrounding:
```bash
trap 'echo "Caught"' INT
( while true; do sleep 1; done ) &
```

### Trap 5: pkill -f too broad

**Problem:** `pkill` kills unintended processes.

**Bad Example:**
```bash
pkill -f "test"  # kills everything with "test" in the command line
```

**Root Cause:** `-f` matches against the full command line. A process like `/usr/bin/python3 /var/www/test_server.py` matches. Even `firefox --new-tab about:testing` matches.

**Fix:** Use `pgrep -f` first to preview, or use specific patterns:
```bash
pgrep -f "test"  # preview
# or
pkill -x test     # exact process name match (no -f)
```

### Trap 6: disown after the shell is already exiting

**Problem:** You background a task, but by the time you disown, it's too late.

**Bad Example:**
```bash
sleep 30 &
# Lots of work...
exit  # triggers SIGHUP before you get to disown
```

**Root Cause:** The shell sends SIGHUP on exit to all jobs in the job table. If you haven't disowned them yet, they die.

**Fix:** Use `nohup` from the start (it ignores SIGHUP at the process level) or disown immediately:
```bash
nohup sleep 30 &
# or
sleep 30 & disown
```

### Trap 7: Race condition with $!

**Problem:** Using `$!` after starting a background job, but the job finishes before you use it.

**Bad Example:**
```bash
sleep 0.01 &   # finishes almost instantly
# ... some slow code ...
kill $! 2>/dev/null  # PID might already be reused!
```

**Root Cause:** PIDs are a finite resource. If the background job finishes quickly, its PID can be recycled for a new process. `kill $!` might kill the wrong process.

**Fix:** Minimize the window between backgrounding and using `$!`. Or use `wait`:
```bash
sleep 0.01 & pid=$!
wait $pid 2>/dev/null  # reaps the process, PID is no longer valid
# Now $pid is safe (can't be reused yet — the wait tracked it)
```

### Trap 8: trap EXIT with exit code overwrite

**Problem:** The EXIT trap runs but changes the exit code.

**Bad Example:**
```bash
trap 'echo "Cleanup"; exit 0' EXIT
false  # sets exit code to 1
exit   # EXIT trap runs, sets exit code to 0
```

**Root Cause:** If your EXIT trap calls `exit` again, the exit code can change. The script exits with 0 instead of 1.

**Fix:** Preserve exit code:
```bash
trap 'rc=$?; echo "Cleanup (exit $rc)"; exit $rc' EXIT
```

### Trap 9: nohup doesn't redirect stderr

**Problem:** `nohup` redirects stdout to `nohup.out`, but stderr goes to terminal or `nohup.out` depending on version.

**Bad Example:**
```bash
nohup ./script_that_errors.sh &
# stderr still shows on terminal
```

**Root Cause:** `nohup` only redirects stdout if it's a terminal. Stderr behavior varies by implementation.

**Fix:** Explicit redirection:
```bash
nohup ./script.sh > output.log 2>&1 &
```

### Trap 10: Job control disabled in non-interactive shells

**Problem:** `bg` and `fg` don't work in scripts.

**Bad Example:**
```bash
#!/bin/bash
some_task &
bg %1  # fails: "bg: no job control"
```

**Root Cause:** Job control is disabled by default in non-interactive shells. Scripts don't have terminal access for job control.

**Fix:** Use process management without `bg`/`fg` in scripts:
```bash
some_task &
pid=$!
# manage with kill/wait, not bg/fg
```

### Trap 11: Trapping SIGKILL

**Problem:** Attempting to trap SIGKILL (signal 9).

**Bad Example:**
```bash
trap 'echo "You cant kill me!"' KILL
```

**Root Cause:** SIGKILL cannot be caught, blocked, or ignored. It's the kernel's ultimate termination signal.

**Fix:** You can't. Don't try. Use SIGTERM for trappable termination.

### Trap 12: Zombie processes from unwaited children

**Problem:** Background jobs that finish leave zombie entries until reaped.

**Bad Example:**
```bash
for i in {1..100}; do
    ( sleep 1; exit $i ) &
done
# 100 zombies temporarily exist
```

**Root Cause:** When a child process finishes, it becomes a zombie until the parent calls `wait()` to reap it. Bash does reap children automatically between commands, but many zombies can pile up in a tight loop.

**Fix:** Wait periodically or use `wait`:
```bash
for i in {1..100}; do
    ( sleep 1; exit $i ) &
    ((i % 10 == 0)) && wait  # reap every 10
done
wait  # reap remaining
```

## See It In The Wild

- **`/etc/init.d/*`** — Every init script uses `start`, `stop`, `status` with PIDs and signals
- **`/usr/bin/screen`** and **`tmux`** — Terminal multiplexers that manage process sessions
- **Docker's `docker stop`** sends SIGTERM, waits 10s, then SIGKILL — the classic 2-phase shutdown
- **Apache's `apachectl`** — sends SIGHUP to reload config, SIGTERM to stop
- **Systemd's `systemctl kill`** — sophisticated signal management for units
- **`/etc/security/limits.conf`** — `nofile`, `nproc` limits on process creation

**Try this now:**

1. `trap 'echo "Ouch!"' INT; while true; do sleep 1; done` — Ctrl+C yourself
2. `sleep 30 & kill -0 $! && echo "Alive" || echo "Dead"` — check existence
3. `time (sleep 2 & sleep 3 & wait)` — parallel vs sequential timing
4. `strace -e trace=kill kill -9 $$ 2>&1 | tail -5` — watch the suicide syscall (don't worry, the strace finishes)

## Check Your Understanding (5-7 questions)

1. What's the default signal sent by `kill`?
2. What signal can't be caught by `trap` and why?
3. What does `trap ... EXIT` do that's different from `trap ... SIGINT`?
4. Why should you avoid `kill -9` as a first resort?
5. How does `disown` differ from `nohup`?
6. What does `kill -0 PID` do and why is it useful?
7. What happens to background jobs when the parent shell exits (without disown/nohup)?
8. Why do zombie processes exist and how do you prevent them?

---
*"SIGTERM is a polite request. SIGKILL is a SWAT team. Always send the polite request first."*
