# Task 17: Large Log File Processor

## Objective

Process a simulated large log file (100MB+) to extract metrics and compute statistics while keeping memory consumption under 50MB. Use streaming techniques, batched processing, and parallelization. Compare the performance of pure-bash vs. delegated (awk/sort) approaches.

## Requirements (8 sub-tasks)

### Sub-task 1: Test Data Generator
Write `generate_logs.sh` that:
- Creates a simulated log file at `/tmp/big.log` (tunable size via `--size 100M`)
- Each line format: `TIMESTAMP|LOG_LEVEL|MESSAGE|VALUE`
- TIMESTAMP = Unix timestamp (ascending, 1-second intervals)
- LOG_LEVEL = INFO, WARN, ERROR, DEBUG (weighted: INFO 40%, WARN 25%, ERROR 20%, DEBUG 15%)
- MESSAGE = Random text like "Transaction processed" or "Connection timeout" etc.
- VALUE = Random integer 0-9999
- Generates ~2 million lines for 100MB

### Sub-task 2: Pure-Bash Processor (Baseline)
Write `bash_processor.sh` that:
- Uses `while IFS= read -r line` (streaming, no full file load)
- Parses each line with `IFS='|' read -r ts level msg val`
- Counts lines per log level
- Computes sum, min, max of VALUE per level
- Stores values for percentile calculation (WARN ONLY — to limit memory)
- Reports line count, per-level stats, overall stats
- Reports peak memory usage at end

### Sub-task 3: Awk-Based Processor (Optimized)
Write `awk_processor.sh` that:
- Delegates all counting and aggregation to `awk`
- Uses a single awk pass: `awk -F'|' '{count[$2]++; sum[$2]+=$4; ...} END {...}'`
- Computes min, max, average per level in awk
- Reports the same statistics as the bash version
- Should be 10-50x faster than the pure bash version
- Also reports peak memory

### Sub-task 4: Percentile Calculator
Write `percentile.sh` that:
- Extracts VALUE from each line, writes to per-level temp files
- For WARN level (or all levels), computes: min, max, median (P50), P95, P99
- Uses streaming approach: don't load all values into memory
- For median: use `sort -n | awk '{vals[NR]=$1} END {...}'` — loads ONE level's values
- For very large levels, use sampling instead of full sort
- Reports memory used for percentile calculation

### Sub-task 5: Memory Monitor
Write `memory_monitor.sh` that:
- Takes a command as argument
- Runs it while monitoring `/proc/PID/status` for VmRSS and VmPeak
- Samples every 100ms
- Reports: peak RSS, average RSS, runtime, throughput (lines/sec)
- Outputs a simple memory timeline (sample every N seconds)
- Works with both bash and other commands

### Sub-task 6: Performance Comparison
Write `compare_processors.sh` that:
- Runs the bash processor, awk processor, and (optionally) a Python processor
- Reports wall time, CPU time, peak memory, and throughput for each
- Generates a comparison table (aligned columns)
- Calculates speedup factor (bash time / awk time)
- Reports which approach wins for a given metric

### Sub-task 7: Batch vs. Stream Comparison
Write `batch_test.sh` that:
- Compares memory/performance of different batch sizes: 100, 1000, 10000 lines
- Uses `mapfile` for batching
- Reports memory vs. speed tradeoff for each batch size
- Recommends optimal batch size for the target file
- Tests at least 3 different batch sizes

### Sub-task 8: Integration Report
Write `large_file_processor.sh` (the final tool) that combines everything:
- Detects the input file size and memory available
- Selects the optimal processing strategy automatically
- Supports `--mode bash|awk|auto`
- Supports `--memory-limit 50M`
- Supports `--percentiles` flag
- Generates a comprehensive report with all statistics
- Exits with error if memory limit is exceeded

## Bonus Challenges

1. **Parallel split-merge:** Use `split` to divide the file into chunks, process each chunk with awk in parallel, then merge results.

2. **Incremental processing:** Process a growing log file (like `tail -F` output) — maintain running statistics across executions.

3. **Anomaly detection:** Flag lines where VALUE is > 3 standard deviations from the mean per level. Requires two passes.

4. **MapReduce in bash:** Implement a simple MapReduce framework: split → map (parallel) → shuffle (sort) → reduce.

5. **Compressed data:** Process `.gz` files without decompressing to disk: `zcat big.log.gz | while read line; do ... done`.

## Hints

<details>
<summary>Hint 1: Generate test data efficiently</summary>

```bash
generate_logs() {
  local lines="${1:-2000000}" file="${2:-/tmp/big.log}"
  local levels=(INFO INFO INFO INFO WARN WARN ERROR ERROR DEBUG)
  local messages=(
    "Transaction processed"
    "Connection timeout"
    "Rate limit exceeded"
    "Invalid request format"
    "Resource allocation failed"
    "Cache hit"
    "Cache miss"
    "Authentication successful"
    "Authentication failed"
    "Session expired"
  )

  > "$file"  # Truncate

  for ((i=1; i<=lines; i++)); do
    local ts=$((1700000000 + i))
    local lvl=${levels[$RANDOM % ${#levels[@]}]}
    local msg=${messages[$RANDOM % ${#messages[@]}]}
    local val=$((RANDOM % 10000))
    echo "$ts|$lvl|$msg|$val"
  done > "$file"
}
```

For faster generation, use `awk` instead:
```bash
awk 'BEGIN {for(i=1;i<=2000000;i++) printf "%d|%s|%s|%d\n", 1700000000+i, levels[int(rand()*4)], msgs[int(rand()*10)], int(rand()*10000)}' 
```
</details>

<details>
<summary>Hint 2: Pure bash streaming processor</summary>

```bash
#!/bin/bash
INPUT="$1"
declare -A COUNTS SUMS MINS MAXES VALUES
TOTAL=0

while IFS='|' read -r ts level msg val; do
  LEVELS[$level]=1
  ((COUNTS[$level]++))
  ((SUMS[$level] += val))
  TOTAL=$((TOTAL + 1))

  if [ -z "${MINS[$level]}" ] || [ "$val" -lt "${MINS[$level]}" ]; then
    MINS[$level]=$val
  fi
  if [ -z "${MAXES[$level]}" ] || [ "$val" -gt "${MAXES[$level]}" ]; then
    MAXES[$level]=$val
  fi

  # Store values for percentile (WARN only)
  if [ "$level" = "WARN" ]; then
    echo "$val" >> /tmp/warn_values.$$
  fi
done < "$INPUT"

echo "=== Per-Level Statistics ==="
printf "%-10s %10s %10s %10s %10s %10s\n" "Level" "Count" "Sum" "Avg" "Min" "Max"
for level in "${!COUNTS[@]}"; do
  avg=$((SUMS[$level] / COUNTS[$level]))
  printf "%-10s %10d %10d %10d %10d %10d\n" \
    "$level" "${COUNTS[$level]}" "${SUMS[$level]}" "$avg" "${MINS[$level]}" "${MAXES[$level]}"
done

# Clean up temp
rm -f /tmp/warn_values.$$
```
</details>

<details>
<summary>Hint 3: Awk processor (fast)</summary>

```bash
awk_processor() {
  awk -F'|' '
  {
    count[$2]++
    sum[$2] += $4
    total_sum += $4
    total_count++
    if ($4 < min[$2] || min[$2] == "") min[$2] = $4
    if ($4 > max[$2] || max[$2] == "") max[$2] = $4

    # Store for percentile (sample 1% for large data)
    if ($2 == "WARN" && int(rand()*100) == 0) {
      warn_vals[++warn_n] = $4
    }
  }
  END {
    printf "%-10s %10s %10s %10s %10s %10s\n", "Level", "Count", "Sum", "Avg", "Min", "Max"
    for (l in count) {
      printf "%-10s %10d %10d %10d %10d %10d\n", l, count[l], sum[l], sum[l]/count[l], min[l], max[l]
    }
  }' "$1"
}
```
</details>

<details>
<summary>Hint 4: Memory monitoring function</summary>

```bash
monitor_memory() {
  local cmd="$1"
  local pid
  local peak=0
  local samples=()

  # Run command in background
  eval "$cmd" &
  pid=$!

  # Monitor memory
  while kill -0 "$pid" 2>/dev/null; do
    local rss=$(grep VmRSS /proc/$pid/status 2>/dev/null | awk '{print $2}')
    if [ -n "$rss" ]; then
      samples+=("$rss")
      [ "$rss" -gt "$peak" ] && peak=$rss
    fi
    sleep 0.1
  done

  wait "$pid"
  local exit_code=$?

  # Calculate average
  local total=0
  for s in "${samples[@]}"; do
    total=$((total + s))
  done
  local avg=$((total / ${#samples[@]}))

  echo "Peak RSS: ${peak}KB"
  echo "Avg RSS: ${avg}KB"
  echo "Samples: ${#samples[@]}"
  return $exit_code
}
```
</details>

<details>
<summary>Hint 5: Percentile from sorted values</summary>

```bash
compute_percentiles() {
  local file="$1"
  local total=$(wc -l < "$file")
  [ "$total" -eq 0 ] && return

  sort -n "$file" | awk -v total="$total" '
  {
    vals[NR] = $1
  }
  END {
    p50_idx = int(total * 0.5)
    p95_idx = int(total * 0.95)
    p99_idx = int(total * 0.99)
    if (p50_idx < 1) p50_idx = 1
    if (p95_idx < 1) p95_idx = 1
    if (p99_idx < 1) p99_idx = 1
    printf "Min: %d\n", vals[1]
    printf "Max: %d\n", vals[total]
    printf "P50 (Median): %d\n", vals[p50_idx]
    printf "P95: %d\n", vals[p95_idx]
    printf "P99: %d\n", vals[p99_idx]
  }'
}
```
</details>

<details>
<summary>Hint 6: Comparing bash vs awk speed</summary>

```bash
compare_speed() {
  echo "=== Performance Comparison ==="
  echo "File: $1 ($(wc -c < "$1") bytes, $(wc -l < "$1") lines)"
  echo ""

  for processor in bash_processor.sh awk_processor.sh; do
    echo "--- $processor ---"
    /usr/bin/time -f "Real: %e s\nUser: %U s\nSys: %S s\nMem: %M KB" \
      bash "$processor" "$1" > /tmp/out_$$.txt 2>/tmp/time_$$.txt
    cat /tmp/time_$$.txt
    echo ""
  done

  rm -f /tmp/out_$$.txt /tmp/time_$$.txt
}
```
</details>

<details>
<summary>Hint 7: Reading compressed files</summary>

```bash
# Process gzipped file without decompressing to disk
process_gz() {
  local file="$1"
  zcat "$file" | while IFS= read -r line; do
    echo "$line"
  done
}

# Or with awk (faster):
process_gz_awk() {
  zcat "$1" | awk -F'|' '{...}'
}
```
</details>

## Expected Output

```bash
$ ./generate_logs.sh --size 100M --output /tmp/big.log
Generating 100MB log file...
Wrote 2,000,000 lines to /tmp/big.log
File size: 104,857,600 bytes (100M)

$ ./bash_processor.sh /tmp/big.log
=== Large Log Processor (Bash) ===
Input: /tmp/big.log (100MB, 2,000,000 lines)
Processing... (this may take a while)

=== Per-Level Statistics ===
Level         Count        Sum        Avg        Min        Max
INFO         800,234  3,998,234       4995          0       9999
WARN         500,123  2,501,234       5000          0       9999
ERROR        200,621    998,765       4978          0       9999
DEBUG         99,022    499,876       5048          0       9999

Peak memory: 8,456 KB (8.3 MB)
Processing time: 142.3 seconds
Throughput: 14,050 lines/sec

$ ./awk_processor.sh /tmp/big.log
=== Large Log Processor (Awk) ===
Input: /tmp/big.log (100MB, 2,000,000 lines)

=== Per-Level Statistics ===
Level         Count        Sum        Avg        Min        Max
INFO         800,234  3,998,234       4995          0       9999
WARN         500,123  2,501,234       5000          0       9999
ERROR        200,621    998,765       4978          0       9999
DEBUG         99,022    499,876       5048          0       9999

Peak memory: 3,456 KB (3.4 MB)
Processing time: 2.8 seconds
Throughput: 714,285 lines/sec

$ ./compare_processors.sh /tmp/big.log
=== Performance Comparison ===

Processor      Time(s)    Mem(KB)   Lines/sec   Speedup
Bash           142.3      8,456     14,050      1.0x
Awk             2.8       3,456     714,285     50.8x
Python         12.1       12,345    165,289     11.8x

Winner by metric:
  Fastest: Awk (2.8s)
  Lowest Mem: Awk (3.4MB)
  Best Throughput: Awk (714K lines/sec)

$ ./large_file_processor.sh /tmp/big.log --mode auto --percentiles
=== Large File Processor (auto-selected: awk) ===

=== Per-Level Statistics ===
Level         Count        Sum        Avg        Min        Max
INFO         800,234  3,998,234       4995          0       9999
WARN         500,123  2,501,234       5000          0       9999
ERROR        200,621    998,765       4978          0       9999
DEBUG         99,022    499,876       5048          0       9999

=== Percentiles (WARN) ===
Min:     0
P50:     4998
P95:     9499
P99:     9899
Max:     9999

=== Verification ===
Memory constraint (50MB): ✅ PASS
Lines processed:  2,000,000
Total sum:        7,998,109
Runtime:          2.8s
Throughput:       714,285 lines/sec
```

## Self-Check Questions

1. Why does `for line in $(cat bigfile)` crash on large files? What's the memory impact?

2. How does `while IFS= read -r line` keep memory constant regardless of file size?

3. What is the advantage of writing values to per-level temp files for median/percentile calculation?

4. Why is `awk` often 10-100x faster than bash for field extraction and aggregation?

5. How would you calculate the 95th percentile without loading all values into memory?

6. What does the `-S` flag do in `sort`, and why is it important for large file processing?

7. How does `split` combined with background jobs enable parallel processing of a single file?

8. What is reservoir sampling and when is it useful for large file processing?
