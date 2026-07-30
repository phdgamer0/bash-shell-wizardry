# Task 8: Profile and Optimize a 10,000-Line Processor

## Objective
Profile a script that processes a large file, identify the bottleneck using PS4 and EPOCHREALTIME, optimize it to achieve significant speedup, and document the process with before/after measurements.

## Requirements

1. **Baseline:** Run the provided slow script on a generated 10,000-line file, measure time
2. **Profile:** Use `PS4` with `EPOCHREALTIME` to identify the bottleneck
3. **Optimize:** Achieve at least 10x speedup
4. **Document:** Show before/after times, what was slow, and why the fix works
5. **Memory constraint:** Keep memory under 50MB
6. **Test correctness:** Ensure optimized output matches baseline output
7. **Multiple strategies:** Implement 3 different optimization approaches and compare

## Sub-tasks (8 cumulative)

### Task 8.1: Generate Test Data

```bash
#!/bin/bash
generate_data() {
  local lines=${1:-10000}
  local output=${2:-/tmp/testdata.txt}
  
  > "$output"
  for i in $(seq 1 "$lines"); do
    # Generate a line with random words
    local line=""
    for j in {1..10}; do
      line+="word_$(shuf -i 1-1000 -n 1) "
    done
    echo "$line" >> "$output"
  done
  echo "Generated $lines lines in $output"
}
generate_data "$@"
```

**Better approach using faster tools:**
```bash
# Much faster generation using awk
generate_data_fast() {
  local lines=${1:-10000}
  awk -v n="$lines" 'BEGIN {
    srand();
    for(i=1; i<=n; i++) {
      for(j=1; j<=10; j++) printf "word_%d ", int(rand()*1000)+1;
      printf "\n"
    }
  }' > /tmp/testdata.txt
}
```

### Task 8.2: Baseline Slow Script

```bash
#!/bin/bash
# baseline.sh — deliberately slow
INPUT=$1
OUTPUT=$2

while read line; do
  for word in $line; do
    echo "$word" >> "$OUTPUT.tmp"
  done
done < "$INPUT"

sort "$OUTPUT.tmp" | uniq -c | sort -rn > "$OUTPUT"
rm -f "$OUTPUT.tmp"
```

**Time this as baseline:** `time bash baseline.sh /tmp/testdata.txt /tmp/result.txt`

### Task 8.3: Profile with PS4

```bash
#!/bin/bash
# profile.sh — same as baseline but with profiling
export PS4='+[${EPOCHREALTIME}] [${BASH_SOURCE[0]:-stdin}:$LINENO] '
exec 2>/tmp/trace.log
set -x

INPUT=$1
OUTPUT=$2

while read line; do
  for word in $line; do
    echo "$word" >> "$OUTPUT.tmp"
  done
done < "$INPUT"

sort "$OUTPUT.tmp" | uniq -c | sort -rn > "$OUTPUT"
rm -f "$OUTPUT.tmp"
set +x
```

**Analyze trace:**
```bash
$ ./profile.sh /tmp/testdata.txt /tmp/result2.txt
$ head -20 /tmp/trace.log
$ # Find slow operations by diffing consecutive timestamps:
$ awk 'NR>1 {
  match(prev, /\[([0-9.]+)\]/, pt);
  match($0, /\[([0-9.]+)\]/, ct);
  if(pt[1] && ct[1]) printf "%.6f %s\n", ct[1]-pt[1], $0
} {prev=$0}' /tmp/trace.log | sort -rn | head -10
```

**What the profile reveals:**
- `echo "$word" >> "$OUTPUT.tmp"` is the hottest line — it opens, writes, closes for each word
- `for word in $line` splits the line (word splitting) — correct but slow
- `while read line` reads one line at a time

### Task 8.4: Optimization Approach 1 — Batch I/O

```bash
#!/bin/bash
# optimized1.sh — redirect the loop
INPUT=$1
OUTPUT=$2

while read line; do
  for word in $line; do
    echo "$word"
  done
done < "$INPUT" > "$OUTPUT.tmp"

sort "$OUTPUT.tmp" | uniq -c | sort -rn > "$OUTPUT"
rm -f "$OUTPUT.tmp"
```

**What changed:** Moved `>` outside the inner loop. All `echo` output goes to a single file descriptor.

**Expected speedup:** 5-10x (eliminates per-word file open/close).

### Task 8.5: Optimization Approach 2 — Word Splitting Alternative

```bash
#!/bin/bash
# optimized2.sh — use tr to split words
INPUT=$1
OUTPUT=$2

tr ' ' '\n' < "$INPUT" | grep -v '^$' > "$OUTPUT.tmp"

sort "$OUTPUT.tmp" | uniq -c | sort -rn > "$OUTPUT"
rm -f "$OUTPUT.tmp"
```

**What changed:** Replaced the while-read loop + for-word loop with a single `tr` command that splits all words. `tr` is a compiled C program — much faster than bash's interpreted loop.

**Expected speedup:** 20-50x (C-level text processing vs interpreted shell loop).

### Task 8.6: Optimization Approach 3 — AWK Pipeline

```bash
#!/bin/bash
# optimized3.sh — use awk for everything
INPUT=$1
OUTPUT=$2

awk '{
  for(i=1; i<=NF; i++) words[$i]++
} END {
  for(w in words) printf "%d %s\n", words[w], w
}' "$INPUT" | sort -rn > "$OUTPUT"
```

**What changed:** `awk` does the entire thing — splitting, counting, and output — in one pass. No external `sort` until the final ordering (which is harder in awk).

**Expected speedup:** 30-100x (single awk process vs multiple shell constructs).

### Task 8.7: Correctness Verification

```bash
verify_output() {
  local baseline=$1
  local optimized=$2
  
  if diff <(sort "$baseline") <(sort "$optimized") >/dev/null 2>&1; then
    echo "PASS: Output matches"
  else
    echo "FAIL: Output differs"
    diff <(sort "$baseline") <(sort "$optimized") | head -20
    exit 1
  fi
}

# Verify all approaches produce identical results
for approach in baseline optimized1 optimized2 optimized3; do
  "$approach.sh" /tmp/testdata.txt "/tmp/result_$approach.txt"
done
verify_output /tmp/result_baseline.txt /tmp/result_optimized1.txt
verify_output /tmp/result_baseline.txt /tmp/result_optimized2.txt
verify_output /tmp/result_baseline.txt /tmp/result_optimized3.txt
```

### Task 8.8: Report Generation

```bash
generate_report() {
  echo "# Performance Optimization Report"
  echo "## System Info"
  echo "- $(uname -a)"
  echo "- $(bash --version | head -1)"
  echo "- Test file: $(wc -l < /tmp/testdata.txt) lines"
  echo
  echo "## Results"
  echo
  echo "| Approach | Real (s) | User (s) | Sys (s) | Speedup | Memory (KB) |"
  echo "|----------|----------|----------|---------|---------|-------------|"
  
  for approach in baseline optimized1 optimized2 optimized3; do
    TIMEFORMAT='%3R %3U %3S'
    { time bash "${approach}.sh" /tmp/testdata.txt "/tmp/result_${approach}.txt" 2>/dev/null; } 2>&1
    # ... parse and report
  done
  
  echo
  echo "## Bottleneck Analysis"
  echo "The baseline script spent ~80% of time in: \`echo \"\$word\" >> \"\$OUTPUT.tmp\"\`"
  echo "This opens/closes the output file for every word (average 5 words/line * 10000 lines = 50000 opens)"
  echo
  echo "## Key Optimization Principles Applied"
  echo "1. Batch I/O — redirect entire loops instead of per-operation writes"
  echo "2. Replace shell loops with compiled tools (tr, awk)"
  echo "3. Avoid per-iteration file operations"
  echo "4. Minimize subshell invocations"
}
```

## Bonus Challenges

1. **Bonus A:** Parallelize the word counting using `xargs -P` and `sort --merge`. Compare with the awk approach.

2. **Bonus B:** Use `valgrind --tool=callgrind` on the awk version. What does the call graph reveal?

3. **Bonus C:** Implement a version using `python3 -c` with `collections.Counter`. Compare performance.

4. **Bonus D:** Create a visual flame graph from the PS4 trace data (convert timestamps to a flamegraph-compatible format).

5. **Bonus E:** Profile the memory hierarchy effects by processing a file that's larger than L3 cache but smaller than RAM.

## Hints

<details>
<summary>Hint 1: PS4 with EPOCHREALTIME</summary>

```bash
PS4='+[${EPOCHREALTIME}] '  # bash 5.0+ only
# Or for older bash:
PS4='+[$(date +%s.%N)] '    # forks, but works everywhere
```
</details>

<details>
<summary>Hint 2: Analyzing trace timestamps</summary>

Use awk to compute delta between consecutive lines to find the slowest operations.
</details>

<details>
<summary>Hint 3: tr for whitespace splitting</summary>

`tr -s ' ' '\n'` collapses multiple spaces and converts to newlines — each word becomes one line. Then pipe to `sort | uniq -c`.
</details>

<details>
<summary>Hint 4: Memory measurement</summary>

```bash
/usr/bin/time -v ./script.sh 2>&1 | grep "Maximum resident"
```
</details>

<details>
<summary>Hint 5: awk associative arrays</summary>

Awk's arrays are hashed. `words[$i]++` creates a counter for each word. `END` block prints results. For sorted output, pipe to `sort`.
</details>

## Expected Output

```bash
$ ./benchmark.sh
Generating test data: /tmp/testdata.txt with 10000 lines

=== Performance Results ===

Baseline:    8.452s   user: 4.231s   sys: 4.215s   MaxMem: 12.8MB
Optimized1:  1.234s   user: 0.892s   sys: 0.341s   MaxMem: 8.2MB   (6.8x)
Optimized2:  0.234s   user: 0.198s   sys: 0.035s   MaxMem: 6.1MB   (36.1x)
Optimized3:  0.089s   user: 0.076s   sys: 0.012s   MaxMem: 5.8MB   (95.0x)

=== Bottleneck Identification ===

Top 5 slowest operations (from trace analysis):
1. echo "$word" >> $OUTPUT.tmp  — 4.12s (48.7%)
2. for word in $line           — 2.34s (27.7%)
3. while read line             — 1.02s (12.1%)
4. sort $OUTPUT.tmp            — 0.84s (9.9%)
5. uniq -c                     — 0.12s (1.4%)

=== Optimization Summary ===

Baseline bottleck: The `>>` operation inside the inner loop
opens and closes the output file ~50,000 times.

Optimized1: Moved redirect outside loop — eliminated 50k open/close pairs.
Optimized2: Replaced shell loop with `tr` — compiled C vs interpreted bash.
Optimized3: Single `awk` pass — split, count, output in one process.
```

## Deep Self-Check

1. **strace analysis:** Run `strace -c baseline.sh` and `strace -c optimized3.sh`. Compare the number of system calls. Which syscalls dominate the baseline?

2. **PID counting:** How many processes does each approach create? Use `strace -f -e trace=clone` to count forks.

3. **Scaling analysis:** Test all 4 approaches with 1000, 10000, 100000, and 1M line files. Which approach scales best? Does the ranking change at different sizes?

4. **Warm vs cold cache:** Run each approach once (cold cache), then again immediately (warm cache). What's the ratio? Which approach is most affected by caching?

5. **Kernel page cache:** Before running the baseline, drop caches: `sync && echo 3 > /proc/sys/vm/drop_caches` (requires root). How much slower is the cold-cache baseline?

6. **I/O scheduler effects:** Check the I/O scheduler (`cat /sys/block/sda/queue/scheduler`). Does changing it affect the baseline more than the optimized versions?

7. **CPU pinning:** Run the optimized versions with `taskset -c 0` (single core) vs unrestricted. Is the awk version CPU-bound enough to benefit from multiple cores?

8. **NUMA effects:** On a multi-socket system, run the awk version with `numactl --membind=0` vs `numactl --interleave=all`. Does the memory allocation policy affect performance?

9. **Container overhead:** Run the test inside Docker vs bare metal. What's the overhead for shell scripts vs compiled tools?

10. **Disk vs tmpfs:** Put the test file on tmpfs (`mount -t tmpfs tmpfs /mnt/tmp`). Does the baseline improve more than the awk version? Why?
