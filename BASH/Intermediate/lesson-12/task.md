# Task 12: Process Management

You'll build a process monitor, worker scripts, and graceful shutdown handlers. This is exactly how real-world daemons, CI runners, and parallel processing frameworks work.

## Sub-tasks

### 1. Worker script — `worker.sh`

Write a simulated long-running worker process.

**Requirements:**
- Accept two arguments: worker ID (number) and total iterations (default 10)
- Loop N times, each iteration:
  - Print: `Worker [ID] PID [$$]: iteration [N]`
  - Sleep 1 second
- Trap SIGTERM and SIGINT to print a graceful shutdown message: `Worker [ID]: shutting down gracefully`
- Exit with code 0 if completed, 1 if interrupted
- Simulate occasional "work" by generating a random sleep between 0.5-1.5s

**Expected:**
```bash
$ ./worker.sh 1 3
Worker [1] PID [12345]: iteration 1
Worker [1] PID [12345]: iteration 2
Worker [1] PID [12345]: iteration 3
Worker [1]: completed

$ ./worker.sh 2 100 &
[1] 12350
$ kill -TERM 12350
Worker [2]: shutting down gracefully
```

### 2. Process monitor — `procmon.sh`

Write a monitor that launches, tracks, and manages multiple workers.

**Requirements:**
- Start 3 instances of `worker.sh` in the background with different IDs
- Store their PIDs in an array
- Every 2 seconds, display:
  - A timestamp
  - The PID list
  - Their status (use `kill -0` for each)
- Trap SIGINT and SIGTERM to trigger graceful shutdown
- Trap EXIT to print a final summary
- Accept `-n NUM` option to launch N workers instead of 3
- Accept `--iterations N` to pass to workers

**Graceful shutdown (required):**
1. Print "Shutting down workers..."
2. Send SIGTERM to all workers
3. Wait 3 seconds
4. Check each with `kill -0`
5. Print which workers are still alive
6. Send SIGKILL to survivors
7. Print "All workers stopped."

**Expected:**
```bash
$ ./procmon.sh
[12:00:00] Workers: 12345 12346 12347
Worker [1] PID [12345]: iteration 1
Worker [2] PID [12346]: iteration 1
Worker [3] PID [12347]: iteration 1
[12:00:02] Workers: 12345 12346 12347 — 3 alive
Worker [1] PID [12345]: iteration 2
Worker [2] PID [12346]: iteration 2
Worker [3] PID [12347]: iteration 2
^C
Shutting down workers...
Waiting 3s for graceful exit...
Worker [1]: shutting down gracefully
Worker [3]: shutting down gracefully
Worker [2] still running — sending SIGKILL
All workers stopped.
=== Summary: 3 workers started, 0 still running ===

$ ./procmon.sh -n 2 --iterations 5
Worker [1] PID [12400]: iteration 1
Worker [2] PID [12401]: iteration 1
...
```

### 3. Detach mode — `--detach`

Add a `--detach` flag to `procmon.sh` that uses `disown` so workers persist after the monitor exits.

**Requirements:**
- When `--detach` is used, call `disown -a` after launching workers
- Print "Workers detached — they will survive shell exit."
- Do NOT trap signals in detach mode (the monitor exits immediately)
- Workers should keep running after the monitor script exits

**Expected:**
```bash
$ ./procmon.sh --detach
[1] 12345
[2] 12346
[3] 12347
Workers detached — they will survive shell exit.
$ exit  # shell closes, workers still running
```

### 4. PID file management — `pidfile.sh`

Write a wrapper that uses PID files to track workers.

**Requirements:**
- Accept `start`, `stop`, `status`, `restart` commands (like an init script)
- `start`: Launch worker.sh, write PID to `/tmp/procmon.pid`
- `stop`: Read PID from file, send SIGTERM, wait, SIGKILL if needed, remove file
- `status`: Check if PID from file is running with `kill -0`
- `restart`: Stop then start
- Handle stale PID files (file exists but process is dead)
- Prevent multiple starts (check if already running)

**Expected:**
```bash
$ ./pidfile.sh start
Worker started (PID: 12345)

$ ./pidfile.sh status
Worker running (PID: 12345)

$ ./pidfile.sh stop
Worker stopped.

$ ./pidfile.sh status
Worker not running.

$ ./pidfile.sh start
Worker started (PID: 12350)

$ ./pidfile.sh start
Error: Worker already running (PID: 12350)
```

## Solution Approaches

### Approach A: Incremental build
1. Write `worker.sh` first — test it standalone
2. Write `procmon.sh` — background 3 workers, monitor loop
3. Add signal handling — the graceful shutdown sequence
4. Add `--detach` and `pidfile.sh` as extensions

### Approach B: Signal-first
1. Test `trap` handlers in minimal scripts first
2. Build the graceful shutdown flow
3. Wrap in `procmon.sh` with background launch
4. Add PID file management

### Approach C: Production-grade
1. Write PID file management first (reusable component)
2. Write `worker.sh` with proper signal handling
3. Write `procmon.sh` using the PID manager
4. Add detach mode

<details>
<summary>Hint 1: Worker with trap</summary>

```bash
#!/bin/bash
id=$1
iterations=${2:-10}
shutdown=0

trap 'shutdown=1; echo "Worker [$id]: shutting down gracefully"' SIGTERM SIGINT

for ((i=1; i<=iterations; i++)); do
    ((shutdown)) && { echo "Worker [$id]: interrupted at iteration $i"; exit 1; }
    echo "Worker [$id] PID [$$]: iteration $i"
    sleep 0.$((RANDOM % 10 + 5))  # 0.5-1.4s
done
echo "Worker [$id]: completed"
exit 0
```
</details>

<details>
<summary>Hint 2: Store PIDs from background jobs</summary>

```bash
pids=()
for i in $(seq 1 "$count"); do
    ./worker.sh "$i" "$iterations" &
    pids+=($!)
done
```
</details>

<details>
<summary>Hint 3: Graceful shutdown function</summary>

```bash
cleanup() {
    echo "Shutting down workers..."
    kill "${pids[@]}" 2>/dev/null     # SIGTERM all
    sleep 3
    for pid in "${pids[@]}"; do
        if kill -0 "$pid" 2>/dev/null; then
            echo "Worker $pid still running — sending SIGKILL"
            kill -9 "$pid"
        fi
    done
    echo "All workers stopped."
}
trap cleanup EXIT SIGINT SIGTERM
```
</details>

<details>
<summary>Hint 4: kill -0 for status checks</summary>

```bash
alive=0
for pid in "${pids[@]}"; do
    if kill -0 "$pid" 2>/dev/null; then
        ((alive++))
    fi
done
echo "[$(date +%T)] Workers: ${pids[*]} — $alive alive"
```
</details>

<details>
<summary>Hint 5: PID file management</summary>

```bash
PIDFILE="/tmp/procmon.pid"

start() {
    if [[ -f "$PIDFILE" ]]; then
        pid=$(<"$PIDFILE")
        if kill -0 "$pid" 2>/dev/null; then
            echo "Error: Worker already running (PID: $pid)"
            exit 1
        fi
        echo "Removing stale PID file"
        rm -f "$PIDFILE"
    fi
    ./worker.sh "$1" &
    echo $! > "$PIDFILE"
    echo "Worker started (PID: $!)"
}

stop() {
    [[ ! -f "$PIDFILE" ]] && { echo "Not running"; return; }
    pid=$(<"$PIDFILE")
    kill "$pid" 2>/dev/null
    sleep 2
    kill -0 "$pid" 2>/dev/null && kill -9 "$pid"
    rm -f "$PIDFILE"
    echo "Worker stopped."
}

status() {
    [[ ! -f "$PIDFILE" ]] && { echo "Not running"; return 1; }
    pid=$(<"$PIDFILE")
    if kill -0 "$pid" 2>/dev/null; then
        echo "Worker running (PID: $pid)"
    else
        echo "Stale PID file (PID $pid not running)"
        rm -f "$PIDFILE"
    fi
}
```
</details>

<details>
<summary>Hint 6: Detach mode logic</summary>

```bash
detach_mode=0

case "$1" in
    --detach) detach=1; shift ;;
esac

# After launching workers:
if ((detach)); then
    disown -a
    echo "Workers detached — they will survive shell exit."
    exit 0
fi

# Only set up traps if not in detach mode
trap cleanup EXIT SIGINT SIGTERM
# ... monitor loop ...
```
</details>

## Bonus Challenges

1. **Restart individual workers**: `procmon.sh --restart 2` restarts only worker 2
2. **CPU/memory monitoring**: Use `ps -p $pid -o %cpu,%mem --no-headers` to track resource usage
3. **Coprocess communication**: Use `coproc` to send commands TO workers and GET results back
4. **Grace period config**: Add `--grace 5` to customize wait time before SIGKILL
5. **Logging**: Redirect each worker's output to a separate log file: `worker_N.log`
6. **Heartbeat**: Workers write timestamps to a file; monitor checks heartbeats to detect stalled workers

## Expected Output Summary

```
worker.sh:
  Simulated worker with loop iterations
  Handles SIGTERM/SIGINT gracefully
  Accepts ID and iteration count

procmon.sh:
  Launches N workers
  Monitors with timestamp and health checks
  Graceful 2-phase shutdown (SIGTERM → wait → SIGKILL)
  EXIT trap with summary
  --detach mode with disown

pidfile.sh:
  Init-script style (start/stop/status/restart)
  PID file with stale detection
  Prevents multiple starts
```

## Self-Check

- What's the difference between sending SIGTERM and SIGKILL?
- What does `trap ... EXIT` guarantee that a simple end-of-script doesn't?
- Why would a background job die when the parent shell exits?
- What does `kill -0 PID` actually do at the kernel level?
- How does `disown` change a process's relationship with the shell?
- Why should you prefer `wait` over manual `sleep`-based waiting for background jobs?
- What's the race condition risk with `$!`?
- How do you prevent a script from exiting before its background jobs finish?
