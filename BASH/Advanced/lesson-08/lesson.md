# Lesson 8: Profiling & Optimization

## History & Origins

Shell profiling has always been a neglected art. Unlike compiled languages (gprof, perf) or managed runtimes (JProfiler, Xdebug), bash has no built-in profiler. The tools we have — `PS4`, `set -x`, `EPOCHREALTIME` — are repurposed debugging features.

The `PS4` variable has been part of bash since early versions. Its original purpose was simply to set the prompt for trace output (`+ ` by default). Clever users realized they could embed timestamps in PS4 to create a rudimentary profiler.

`EPOCHREALTIME` was added in bash 5.0 (2019) as a dynamic variable that returns the current time with nanosecond precision. Before that, users relied on `$(date +%s.%N)` which forked a process for every timestamp — ironic for a profiling tool.

`SECONDS` has been in bash since 2.0 — a simple counter that increments every second. It's useful for coarse timing but useless for fine-grained profiling.

## Syntax Reference

### Profiling Tools
```
PS4='+ '                      # Default trace prefix
PS4='+[$EPOCHREALTIME] '     # Timestamped trace (bash 5.0+)
PS4='+[${SECONDS}s] '        # Second-level timestamps
PS4='+[$LINENO] '            # Line number in trace
PS4='+[${BASH_SOURCE[0]}:$LINENO] '  # File:line in trace
set -x                        # Enable execution trace
set +x                        # Disable execution trace
```

### Performance Variables
```
$EPOCHREALTIME               # Current time (seconds.nanoseconds, bash 5.0+)
$EPOCHSECONDS                # Current epoch time (seconds, bash 5.0+)
$SECONDS                     # Seconds since shell start
$BASHPID                     # Current bash process PID
$BASH_SUBSHELL               # Subshell level
```

### Optimization-Relevant Options
```
set -o noglob                 # Disable pathname expansion (speed)
shopt -s extglob              # Extended pattern matching (slower)
shopt -u sourcepath           # Don't search PATH for source (speed)
```

### Logging
```
exec N>file                   # Redirect trace to file
exec 2>trace.log              # Send stderr (and trace) to file
script -q -c "bash -x script.sh" trace_output  # Record terminal output
```

## Under the Hood

### PS4 Profiling Internals

When `set -x` is active, bash prints a trace line to stderr before executing each simple command. The trace prefix is the value of `PS4`. The trace line consists of:
1. `PS4` value (expanded)
2. The command as parsed (after expansion, before execution)

The expansion of `PS4` happens ONCE per command, using the shell's normal parameter expansion. If `PS4` contains `$EPOCHREALTIME`, that variable is read at execution time, giving the current timestamp.

```
bash execution loop:
1. Read next command
2. Expand variables in command
3. Expand PS4
4. Print PS4 + expanded command to stderr
5. Execute command
```

### The Cost of Profiling

Profiling adds overhead. Every trace line requires:
- String expansion (for PS4 content)
- `write()` syscall to stderr
- The tracing itself interrupts normal execution flow

For `PS4='+[$EPOCHREALTIME] '`, each trace line also reads the system clock (a `clock_gettime()` syscall). For 100,000 trace lines, this adds ~0.1-0.5 seconds of overhead.

**Mitigation:** Profile a representative SUBSET, not the entire run. Or redirect trace to a file (`exec 2>trace.log`) to avoid terminal overhead.

### Comparison with C Profiling

| Concept | C (gprof) | Bash |
|---|---|---|
| Function call counting | Automatic (`-pg` flag) | Manual (increment counter in each function) |
| Execution timing | `clock()` or `gettimeofday()` | `$EPOCHREALTIME` |
| Call graph | Automatic | Impossible (bash doesn't track callers) |
| Line-level timing | `perf annotate` | `PS4` + `$EPOCHREALTIME` |
| Profiling overhead | 10-30% | 50-500% (much higher in bash) |
| Memory profiling | `valgrind` | Not available |

### I/O Batching and the Kernel

Each `write()` syscall has overhead (~0.5-2µs). Writing 100,000 lines with 100,000 separate writes takes ~100ms just in syscall overhead.

Batching I/O reduces this dramatically:
```
# 100000 writes (worst)
for i in {1..100000}; do echo "$i" >> file; done

# 1 write (best)
for i in {1..100000}; do echo "$i"; done > file
```

The latter uses shell buffering: bash's `echo` writes to an internal buffer, which is flushed once when the loop completes (one `write()` syscall for the entire output).

## Core Examples

### Example 1: Profiling with PS4 Timestamps

```bash
$ cat > profile.sh << 'EOF'
#!/bin/bash
PS4='+[$(date +%s.%N)] '
set -x
sleep 0.5
echo "hello"
sleep 0.3
set +x
EOF
$ bash profile.sh 2>&1
+[1234567890.123456789] sleep 0.5
+[1234567890.623456789] echo hello
+[1234567890.623456789] hello
+[1234567890.923456789] sleep 0.3
+[1234567890.923456789] echo "world"
+[1234567890.923456789] world
```

**Anatomy:**
1. Each line shows the timestamp and command
2. The difference between consecutive timestamps is the execution time
3. `sleep 0.5` shows ~0.5s difference

**TRAP:** `$(date +%s.%N)` forks `date` for EVERY trace line. This adds significant overhead and its OWN execution time. Use `$EPOCHREALTIME` instead.

### Example 2: Using EPOCHREALTIME (bash 5.0+)

```bash
$ cat > profile_fast.sh << 'EOF'
PS4='+[$(printf "%.6f" $EPOCHREALTIME)] '
set -x
sleep 0.5
set +x
EOF
$ bash profile_fast.sh 2>&1
+[1234567890.123456] sleep 0.5
+[1234567890.623456] set +x
```

**Anatomy:**
- `$EPOCHREALTIME` is a dynamic variable — no fork, no subshell
- `printf "%.6f"` formats to microseconds
- This is the fastest way to get timestamps in bash

**What if `$EPOCHREALTIME` is not set?** (bash < 5.0) — Fall back to `date +%s.%N` or `SECONDS`.

### Example 3: Manual Hot-Spot Identification

```bash
$ cat > find_hot.sh << 'EOF'
start=$EPOCHREALTIME
for i in {1..100000}; do
  : # fast operation
done
end=$EPOCHREALTIME
echo "Loop took: $(echo "$end - $start" | bc -l) seconds"

# Identify which PART of the loop is slow:
for i in {1..10}; do
  a=$EPOCHREALTIME
  # Operation A
  b=$EPOCHREALTIME
  # Operation B  
  c=$EPOCHREALTIME
  echo "$((i)): A=$(echo "$b-$a" | bc -l) B=$(echo "$c-$b" | bc -l)"
done
```

### Example 4: for vs while vs xargs Performance

```bash
$ # for loop (fast for small sets)
$ time for i in {1..1000}; do echo "$i" >/dev/null; done

$ # while read (good for pipes)
$ seq 1000 | time while read i; do echo "$i" >/dev/null; done

$ # xargs (parallelism, limited by fork)
$ seq 1000 | time xargs -P4 -I{} echo {} >/dev/null
```

**What if variations:**
- For 10,000,000 items: `for` loop with `{1..10000000}` uses too much memory (all items expanded at once). Use `for ((i=0; i<10000000; i++))` instead.
- For piped data: `while read` is the only option (data arrives streaming).

### Example 5: Caching Expensive Operations

```bash
$ cache_file=/tmp/.mycache
$ if [[ -f "$cache_file" ]] && [[ $(<"$cache_file") -eq 42 ]]; then
>   result=$(<"$cache_file")
> else
>   result=$(expensive_operation)
>   echo "$result" > "$cache_file"
> fi
```

**Anatomy:**
1. Check if cache file exists AND has expected value
2. If valid cache: read from file (fast, one `read()`)
3. If no cache: run expensive operation and save result

**What if variations:**
- Cache with timeout: `find "$cache_file" -mmin -60` checks freshness
- Cache with size limit: delete cache if >1MB
- Memory cache: `cached_result=$expensive_output` — simple, but resets each run

### Example 6: mapfile vs while read for Large Files

```bash
$ # mapfile (fast — reads entire file at once)
$ time mapfile -t lines < bigfile.txt
real 0m0.015s

$ # while read (slower — one read() per line)
$ time while IFS= read -r line; do :; done < bigfile.txt
real 0m0.045s
```

**Memory tradeoff:** `mapfile` stores the ENTIRE file in memory as an array. For a 1GB file, that's 1GB+ in memory. `while read` processes one line at a time — constant memory regardless of file size.

### Example 7: Split for Parallel Processing

```bash
$ split -l 10000 bigfile.txt part_
$ for f in part_*; do
>   process_file "$f" &
> done
$ wait
$ cat part_* > result.txt
$ rm part_*
```

**Anatomy:**
1. `split` divides the file into chunks (fast, single pass)
2. Each chunk is processed by a background job
3. `wait` waits for all jobs to complete
4. Results are concatenated

**Limitations:** I/O bound tasks don't benefit (disk is the bottleneck). CPU-bound tasks benefit up to the number of CPU cores. Over-parallelizing causes thrashing.

### Example 8: Redirecting the Entire Loop

```bash
$ # BAD — opens/closes file 10000 times
$ for i in {1..10000}; do
>   echo "$i" >> /tmp/out.txt
> done

$ # GOOD — opens/closes file once
$ for i in {1..10000}; do
>   echo "$i"
> done > /tmp/out.txt
```

**Why it matters:** Each `>>` calls `open()` (O_WRONLY|O_CREAT|O_APPEND), then `write()`, then implicitly closes when the command finishes (or with `exec`). The loop redirect style does ONE `open()`, many `write()`, ONE `close()`.

### Example 9: Calling External Commands in Loops

```bash
$ # BAD — forks date every iteration
$ for i in {1..1000}; do
>   echo "$(date) $i"
> done

$ # GOOD — capture once, reuse
$ now=$(date)
$ for i in {1..1000}; do
>   echo "$now $i"
> done

$ # BAD — forks grep every iteration
$ for file in *.log; do
>   result=$(grep "ERROR" "$file" | wc -l)
> done

$ # GOOD — use a single awk or grep
$ awk '/ERROR/{count[FILENAME]++}' *.log
```

### Example 10: AWK vs Shell Loop for Text Processing

```bash
$ # Shell loop (SLOW for large files)
$ while IFS= read -r line; do
>   set -- $line
>   echo "$2"
> done < data.txt > field2.txt

$ # AWK (FAST — compiled C)
$ awk '{print $2}' data.txt > field2.txt
```

**Ratio:** `awk` is typically 10-100x faster than a shell loop for text processing. The shell loop forks nothing (with builtins), but bash's interpreted execution is slow compared to `awk`'s compiled C implementation.

### Example 11: PS4 with Function Call Counting

```bash
$ PS4='+[${FUNCNAME[0]:-main}] '
$ set -x
$ myfunc() { echo "hello"; }
$ myfunc
+[main] myfunc
+[myfunc] echo hello
+[myfunc] hello
```

Useful for seeing which functions are called and how often. Combine with `EPOCHREALTIME` for per-function timing.

### Example 12: Using SECONDS for Coarse Timing

```bash
$ SECONDS=0  # Reset
$ script_operation
$ echo "Took ${SECONDS}s"
```

`SECONDS` is reset by assigning 0 to it. It has 1-second resolution, so it's only useful for operations taking more than 1-2 seconds.

### Example 13: Memory Profiling with /proc

```bash
$ # Check memory usage of a script
$ (echo $$ > /tmp/mypid; exec myscript.sh) &
$ while kill -0 $! 2>/dev/null; do
>   cat /proc/$(cat /tmp/mypid)/status | grep VmRSS
>   sleep 0.1
> done
```

### Example 14: Counting System Calls

```bash
$ strace -c -f bash -c '
  for i in {1..10000}; do
    echo "$i" > /dev/null
  done
' 2>&1 | tail -5
```

### Example 15: Benchmarking with hyperfine

```bash
$ # If hyperfine is installed:
$ hyperfine --warmup 3 --min-runs 10 \
  'bash -c "for i in {1..100000}; do :; done"' \
  'bash -c "for ((i=0; i<100000; i++)); do :; done"'
```

## Real-World Use Cases

### FOR the OS
- **Log rotation timing:** Measure time to rotate and compress multi-GB log files
- **Cron job profiling:** Wrap cron jobs in profiling to detect performance regressions
- **Init script optimization:** Boot-time scripts benefit from subshell elimination

### WITH the OS
- **Data pipeline profiling:** Find bottlenecks in `extract-transform-load` shell scripts
- **CI build optimization:** Reduce CI feedback time by profiling and optimizing build scripts
- **Backup window reduction:** Speed up backup scripts to fit within maintenance windows

### AGAINST the OS (defense perspective)
- **Timing side channels:** An attacker can measure script execution time to infer data values (e.g., password comparison, token validation)
- **Performance degradation:** Inefficient scripts are a DOS vector — attacker feeds inputs that trigger worst-case performance (e.g., ReDoS via shell patterns)

### FOR DEFENSE
- **Profiling for anomalies:** Monitor script execution times — unexpected increases may indicate compromise
- **Resource limits:** Use `ulimit -t` (CPU time), `ulimit -v` (virtual memory) to contain out-of-control scripts
- **strace for forensics:** Trace suspicious scripts to see exactly what syscalls they make

## Memory Aids

- **"PS4 is the probe"** — Set PS4 with timestamps, enable set -x, run the script, analyze the output.
- **"Profile before you optimize"** — Never guess what's slow. Measure first.
- **"Batch your I/O"** — One big write is cheaper than many small writes. Redirect loops, not lines.
- **"Move invariants out of loops"** — If it doesn't change per iteration, compute it before the loop.
- **"External commands in loops = death by a thousand forks"** — Each fork costs microseconds; a million forks costs seconds.
- **"AWK is your friend for field operations on text"** — Shell loops are for orchestration, not heavy data processing.

## Trap Vault

1. **`$(date)` in PS4 for profiling:** This forks a subshell for EVERY traced command. The profiling itself adds huge overhead. Use `$EPOCHREALTIME` instead.

2. **`{1..1000000}` memory explosion:** Brace expansion creates ALL elements before the loop starts. For ranges > 100,000, use C-style `for ((i=1; i<=N; i++))`.

3. **`set -x` output goes to stderr:** If you redirect stderr to a file, trace also goes there. Don't accidentally mix trace with actual error output.

4. **Profiling a pipeline:** `set -x` doesn't trace inside pipeline components started by `|`. Each pipeline stage is a subshell. Add `set -x` to the beginning of the script or use `bash -x`.

5. **`EPOCHREALTIME` resolution:** It provides nanoseconds, but the actual resolution depends on the kernel's clock configuration (typically ~1µs on modern hardware).

6. **Overhead of printf in PS4:** `PS4='+[$(printf "%.6f" $EPOCHREALTIME)] '` calls `printf` (builtin) every time. This is fast but not free. For minimal overhead, use `PS4='+[$EPOCHREALTIME] '` (no formatting).

7. **`strace` significantly slows the script:** `strace` adds 10-100x overhead. Use for selective tracing, not full runs.

8. **Cache warming is essential:** First run of any script/command is significantly slower due to disk and CPU cache. Always do a warm-up run before measuring.

9. **Background jobs and timing:** `time myscript &` doesn't work as expected — `time` measures the background job's launch, not its execution. Use `{ time myscript; } 2> timing.txt &` or wrap in a subshell.

10. **`&>` redirects both stdout and stderr but hides errors:** In profiling, you often want to separate trace (stderr) from output (stdout). `exec 2>trace.log; set -x` keeps them separate.

11. **`read -r` vs `read` performance:** `read` (without `-r`) processes backslash escapes, which adds overhead. For raw data, `read -r` is faster.

12. **Quoting in benchmark runs:** `time for i in {1..1000}; do :; done` — the `time` keyword times the entire `for` compound command. But `time echo "hello"` times just `echo`. The scope of `time` depends on what follows.

13. **`source` inside a loop:** `for file in plugins/*; do source "$file"; done` sources N files. Each source reads the file from disk. If plugins haven't changed, use a cached hash skip.

14. **`wait` granularity:** `wait` waits for ALL background jobs. If one job hangs, `wait` hangs. Use per-job `wait %N` or `timeout` for safety.

15. **`bash -x script.sh` vs `set -x` inside the script:** `bash -x` enables tracing from the first line. `set -x` inside the script starts after it executes. Both work, but `-x` from the CLI traces the entire file including the shebang line.

## See It In The Wild

```bash
# Profile a real system script
$ bash -x /etc/cron.daily/logrotate 2>&1 | head -20

# Use PS4 with timestamps on a long-running script
$ PS4='+[$(printf "%.3f" $EPOCHREALTIME)] ' bash -x ./slow_script.sh 2> /tmp/trace.log

# Analyze trace output
$ awk '{print $1}' /tmp/trace.log | sort | uniq -c | sort -rn | head -10

# Using strace to count system calls
$ strace -c -f bash -c 'for i in {1..100000}; do echo "$i" >/dev/null; done' 2>&1

# Find the hottest lines (most time spent)
$ awk '{
  match($0, /\[([0-9.]+)\]/, t)
  if(prev_time && t[1]) print t[1] - prev_time, $0
  prev_time = t[1]
}' /tmp/trace.log | sort -rn | head -5
```

## Check Your Understanding

1. Why does `PS4='+[$(date +%s.%N)] '` add significant overhead to profiling? How does `$EPOCHREALTIME` solve this?

2. What is the difference between `time for i in {1..10000}; do :; done` and `time (for i in {1..10000}; do :; done)`?

3. Why does redirecting an entire loop (`done > file`) perform better than writing per-iteration (`echo "$i" >> file`)?

4. How would you identify which function in a bash script is the slowest without modifying the script's source?

5. Why does `mapfile` use more memory than `while read`? When would you choose one over the other?

6. How does `strace -c` help identify performance issues? What does it count?

7. What is the overhead of a single `fork()` call? How does this affect scripts that call many external commands in a loop?

8. How does warming the disk cache affect benchmark results? How would you perform a cold-cache benchmark?

9. Why is `awk` often faster than a shell loop for field extraction from text files?

10. How would you profile a script that uses pipeline stages (commands connected with `|`)? How does `set -x` behave with pipelines?
