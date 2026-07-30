# Task 7: Benchmark & Optimize

## Objective
Benchmark 3 different ways to accomplish the same task, optimize a slow script by identifying and eliminating bottlenecks, use custom `TIMEFORMAT`, and prove that shell builtins are faster than external commands through systematic measurement.

## Requirements

1. **Benchmark 3 methods** for reading a file line by line: `while read`, `mapfile`, and `cat | while read`
2. **Custom TIMEFORMAT** showing real/user/sys with 3 decimal places
3. **Optimize a slow script** (provided below) — target 5x speedup, aim for 10x+
4. **Prove builtins are faster** — compare `echo` vs `/bin/echo`, `[` vs `test` vs `/usr/bin/test`
5. **Statistical significance:** Run each benchmark 5 times and report median, min, max
6. **Warm-up awareness:** Perform a warm-up run before measuring
7. **Report generation:** Produce a formatted table of results

## Sub-tasks (8 cumulative)

### Task 7.1: Benchmark Harness
Create a reusable benchmark function:

```bash
#!/bin/bash
benchmark() {
  local label="$1"
  local command="$2"
  local runs=${3:-5}
  local results=()
  
  # Warm-up
  eval "$command" >/dev/null 2>&1
  
  for ((i=0; i<runs; i++)); do
    TIMEFORMAT='%3R %3U %3S'
    local timing
    timing=$({ time eval "$command" >/dev/null 2>&1; } 2>&1)
    results+=("$timing")
  done
  
  # Calculate stats
  local reals=() users=() syss=()
  for r in "${results[@]}"; do
    read -r ru us sy <<< "$r"
    reals+=("$ru"); users+=("$us"); syss+=("$sy")
  done
  
  echo "$label:"
  echo "  Real: min=$(min "${reals[@]}") median=$(median "${reals[@]}") max=$(max "${reals[@]}")"
  echo "  User: min=$(min "${users[@]}") median=$(median "${users[@]}") max=$(max "${users[@]}")"
}
```

### Task 7.2: File Reading Methods Comparison
Benchmark three approaches to read a file:

```bash
# Generate test file
seq 1 100000 > /tmp/test_100k.txt

# Method 1: while read with redirect
time while IFS= read -r line; do :; done < /tmp/test_100k.txt

# Method 2: mapfile (read into array)
time mapfile -t lines < /tmp/test_100k.txt

# Method 3: cat | while read (subshell!)
time cat /tmp/test_100k.txt | while IFS= read -r line; do :; done
```

**Expected results (relative):**
- `mapfile` fastest (~1x)
- `while read` slower (~3-4x)
- `cat | while read` slowest (~8-10x)

**Why the differences:**
- `mapfile` reads the entire file in one `read()` call, then splits in memory
- `while read` calls `read()` for each line (100,000 system calls)
- `cat | while read` also forks `cat` (extra process) AND reads in a subshell

### Task 7.3: Builtin vs External Comparison

```bash
echo "=== Builtin vs External Benchmark ==="
TIMEFORMAT='%3R'

echo "Builtin echo:"
time for i in {1..10000}; do echo "test" >/dev/null; done

echo "External echo:"
time for i in {1..10000}; do /bin/echo "test" >/dev/null; done

echo "Builtin [:"
time for i in {1..10000}; do [[ "a" == "a" ]]; done

echo "External test:"
time for i in {1..10000}; do /usr/bin/test "a" = "a"; done
```

**Expected ratio:** Builtins should be 10-50x faster.

### Task 7.4: Optimize the Slow Script

Original slow script:
```bash
#!/bin/bash
for i in $(seq 1 1000); do
  echo $(cat /etc/hostname)
done
for i in $(seq 1 500); do
  /bin/echo "Line $i" >> /tmp/out.txt
done
```

**Bottlenecks identified:**
1. `$(seq 1 1000)` — forks `seq` 1000 times (actually 1 time for the seq output, then the for splits it)
   - Fix: `{1..1000}` (brace expansion)
2. `echo $(cat /etc/hostname)` — forks `cat` 1000 times
   - Fix: `hostname=$(</etc/hostname)` or `hostname=$(cat /etc/hostname)` then reuse the variable
3. `/bin/echo "Line $i" >> /tmp/out.txt` — forks external echo + opens/close file each time
   - Fix: use builtin `echo` and redirect the entire loop

**Optimized version:**
```bash
#!/bin/bash
hostname=$(</etc/hostname)  # Read once
for i in {1..1000}; do      # Brace expansion (no fork)
  echo "$hostname"
done > /tmp/out.txt          # Redirect entire loop

for i in {1..500}; do
  echo "Line $i"
done >> /tmp/out.txt         # Redirect entire loop
```

### Task 7.5: Useless cat Elimination

Find and fix all useless `cat` invocations:

```bash
# Before:
cat file | grep pattern
cat file | wc -l
cat file | head -5
cat file | sed 's/foo/bar/'

# After:
grep pattern file
wc -l < file
head -5 file
sed 's/foo/bar/' file
```

Each `cat` removal saves a fork (~10-30µs) and one pipe stage.

### Task 7.6: Loop Optimization Patterns

```bash
# Pattern 1: Move invariant code OUTSIDE the loop
# SLOW:
for file in *.txt; do
  sed -i "s/$(hostname)/$(date)/g" "$file"
done
# FAST:
hname=$(hostname); d=$(date)
for file in *.txt; do
  sed -i "s/$hname/$d/g" "$file"
done

# Pattern 2: Avoid repeated $(()) evaluations
# SLOW:
for ((i=0; i<$(wc -l < file); i++)); do ... done
# FAST:
max=$(wc -l < file)
for ((i=0; i<max; i++)); do ... done

# Pattern 3: Batch file operations
# SLOW:
for i in {1..1000}; do
  touch "$i.txt"
done
# FAST:
touch {1..1000}.txt
```

### Task 7.7: Statistical Reporting

```bash
report_results() {
  local method times_ms
  echo "+---------------------+--------+--------+--------+"
  echo "| Method              | Min    | Median | Max    |"
  echo "+---------------------+--------+--------+--------+"
  
  for method in "while read" "mapfile" "cat|while"; do
    # ... run benchmarks ...
    printf "| %-19s | %6.3f | %6.3f | %6.3f |\n" \
      "$method" "$min" "$median" "$max"
  done
  
  echo "+---------------------+--------+--------+--------+"
}

# Helper functions:
min() {
  local m=$1; for v in "$@"; do (( $(echo "$v < $m" | bc -l) )) && m=$v; done
  echo "$m"
}
median() {
  local arr=("$@")
  IFS=$'\n' arr=($(sort -n <<< "${arr[*]}")); unset IFS
  echo "${arr[${#arr[@]}/2]}"
}
```

### Task 7.8: Comprehensive Optimization Report

```bash
optimization_report() {
  echo "=== Performance Optimization Report ==="
  echo "Date: $(date)"
  echo "System: $(uname -a)"
  echo
  
  echo "--- Baseline ---"
  time bash original.sh
  
  echo "--- Optimized ---"
  time bash optimized.sh
  
  echo "--- Bottlenecks Found ---"
  echo "1. $(cat file) in loop (fork per iteration)"
  echo "2. External echo instead of builtin"
  echo "3. Per-iteration file redirect"
  echo "4. seq instead of brace expansion"
  
  echo "--- Speedup ---"
  # Calculate: baseline / optimized
}
```

## Bonus Challenges

1. **Bonus A:** Use `perf stat -e context-switches,cpu-migrations,page-faults,cycles,instructions` on both versions and compare.

2. **Bonus B:** Benchmark `sed` vs `awk` vs `bash while-read` for a field extraction task on 1M lines.

3. **Bonus C:** Create a graphical benchmark report using ASCII bar charts based on execution times.

4. **Bonus D:** Benchmark different I/O strategies: mmap (via `dd`), buffered read, direct I/O.

5. **Bonus E:** Write a script that automatically detects slow patterns in other bash scripts and suggests faster alternatives.

## Hints

<details>
<summary>Hint 1: TIMEFORMAT for machine parsing</summary>

```bash
TIMEFORMAT='%3R'  # Only real time in seconds — easy to parse
```
</details>

<details>
<summary>Hint 2: Redirecting entire loops</summary>

```bash
# The entire loop's stdout goes to file once
for i in {1..1000}; do
  echo "$i"
done > output.txt
```
</details>

<details>
<summary>Hint 3: Reading file into variable</summary>

```bash
content=$(<file)       # Faster than $(cat file)
mapfile -t arr < file  # Fastest for arrays
```
</details>

<details>
<summary>Hint 4: bc for floating point</summary>

```bash
ratio=$(echo "$baseline / $optimized" | bc -l)
```
</details>

<details>
<summary>Hint 5: uuidgen for temporary files</summary>

Use `mktemp` for temp files to avoid conflicts in benchmarks.
</details>

## Expected Output

```bash
$ ./benchmark.sh
=== File Reading Benchmarks (5 runs each) ===
+---------------------+--------+--------+--------+
| Method              | Min    | Median | Max    |
+---------------------+--------+--------+--------+
| while read          |  0.042 |  0.044 |  0.047 |
| mapfile             |  0.015 |  0.016 |  0.018 |
| cat | while read    |  0.089 |  0.092 |  0.098 |
+---------------------+--------+--------+--------+
Speedup: mapfile is 2.8x faster than while read

=== Builtin vs External (10000 iterations) ===
Builtin echo:    0.045s
External echo:   1.234s (27.4x slower)
Builtin [:       0.008s
External test:   0.421s (52.6x slower)

=== Optimization Results ===
Baseline:  3.452 seconds
Optimized: 0.089 seconds
Speedup:   38.8x

Bottlenecks:
1. $(cat /etc/hostname) forked cat 1000 times
2. /bin/echo forked 500 times
3. >> in loop opened/closes file 500 times
4. $(seq ...) forked seq
```

## Deep Self-Check

1. **Cache warming measurement:** Measure the difference between cold-cache (first run) and hot-cache (subsequent runs) for a script that reads a 100MB file. What's the ratio?

2. **Fork counter:** Use `strace -c` to count fork/clone/vfork calls in both the original and optimized scripts. How many fewer system calls does the optimized version make?

3. **Memory vs speed tradeoff:** `mapfile` is faster but uses more memory. Find the file size where `mapfile` exceeds available RAM on your system (and fails). Where is the crossover point?

4. **Hyperfine comparison:** If `hyperfine` is available, compare the builtin `time` results with `hyperfine` results. Are they consistent? Why might they differ?

5. **CPU frequency scaling:** Modern CPUs downclock when idle. Run a benchmark while `stress -c 1` is running (simulating loaded system). How do results differ?

6. **Pipe vs temp file:** Benchmark `cmd1 | cmd2` vs `cmd1 > /tmp/tmpfile; cmd2 < /tmp/tmpfile`. Which is faster for large data? Why?

7. **Parallel processing:** Implement `xargs -P` parallel processing for a CPU-bound task. Measure speedup vs thread count. At what point does overhead outweigh gain?

8. **Container overhead:** Run the benchmark inside a Docker container and natively. What's the overhead of containerization on shell script performance?

9. **Shebang overhead:** Time `bash script.sh` vs `./script.sh` (with shebang). Does the shebang add overhead? Why?

10. **Benchmarking `exec` vs `source`:** Compare `bash script.sh` (fork + exec) vs `source script.sh` (no fork). For scripts that define functions and exit, which is faster?
