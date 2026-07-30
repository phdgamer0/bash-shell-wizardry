# Lesson 17: Large-scale Data Processing

## History & Origins

Processing large datasets has been a Unix strength since the 1970s. The Unix philosophy — small tools, text streams, pipelines — was designed for data processing. Ken Thompson's `grep` (1974), `sort` (Bell Labs, early Unix), and `awk` (Aho, Weinberger, Kernighan, 1977) were all created to manipulate text data efficiently.

The key insight: these tools are written in C and process data in STREAMS. They read a buffer, process it, write a buffer, repeat. Memory usage is bounded by the buffer size, NOT the file size. A 100GB file can be processed with a few KB of memory.

Bash adds the glue: loops, conditionals, and process coordination. But bash's data processing is TEMPTINGLY SLOW — a bash `while read` loop is 100-1000x slower than `awk` for line-by-line field extraction. The bash array, arithmetic, and string operations happen in the shell's interpreter loop, which is orders of magnitude slower than compiled C.

The challenge: know when to use bash (control flow, orchestration) and when to delegate to C tools (data transformation, sorting, filtering, aggregation). The shell script should be the CONDUCTOR, not the musician.

Modern big data tools (Hadoop, Spark, Flink) scale to petabytes across clusters, but for the 100MB-to-100GB range, Unix tools + bash are often simpler and faster. No cluster setup, no JVM overhead, no schema design.

## Syntax Reference

```
# Memory-efficient reading
while IFS= read -r line; do ... done < file     # Line-by-line, constant memory
mapfile -t array < file                           # Entire file into array (memory heavy)
mapfile -t -n 1000 array < file                   # First 1000 lines into array

# Splitting and combining
split -l 10000 bigfile prefix                     # Split into 10k-line chunks
split -b 100M bigfile prefix                      # Split into 100MB chunks
csplit file '/PATTERN/' '{*}'                     # Split based on regex pattern
cat chunk_* > combined                            # Combine chunks
paste file1 file2                                 # Merge columns
join -t, -1 1 -2 1 file1 file2                   # Relational join on field

# Sorting and unique
sort -S 1G -t, -k2 -n bigfile                     # Sort with 1GB buffer
sort -u file                                      # Sort and unique
sort -t, -k1 -k2 -n file                          # Sort by multiple keys
sort -R file                                      # Random sort
uniq -c file                                      # Count consecutive duplicates

# Streaming transformations
sed 's/foo/bar/g' file                            # Replace (streaming)
grep -E 'ERROR|FATAL' file                         # Filter (streaming)
awk '{sum += $1} END {print sum}' file             # Aggregate (streaming)
cut -d, -f1,3 file                                 # Column extraction (streaming)

# Data investigation
head -n 5 file                                     # First 5 lines
tail -n 5 file                                     # Last 5 lines
wc -l file                                         # Line count
shuf -n 100 file                                   # Random 100 lines
wc -c file                                         # Byte count
file file                                          # Detect file type
```

## Under the Hood (GO DEEP)

### Why for+cat is Memory-Oblivious

```bash
# This loads the ENTIRE file into memory:
for line in $(cat hugefile); do
  process "$line"
done
```

**What happens internally:**
1. `$(cat hugefile)` forks a subprocess running `cat`
2. `cat` reads the entire file and writes it to stdout (piped back to bash)
3. bash reads all the output into a string in memory
4. bash then performs WORD SPLITTING on the string (splitting on IFS characters — by default space, tab, newline)
5. Each "word" becomes an iteration of the `for` loop

For a 1GB file, the subprocess memory + bash's internal string use ~2GB. Then word splitting on ALL whitespace means tabs and spaces also split lines. A line like `error count: 42` becomes 3 iterations. This is BOTH memory-heavy AND semantically wrong.

```bash
# This uses CONSTANT (~8KB) memory:
while IFS= read -r line; do
  process "$line"
done < hugefile
```

**What happens internally:**
1. Bash opens `hugefile` on FD 10 (or whatever the redirected stdin becomes)
2. Each iteration of `while`: `read()` reads up to `MAX_INPUT_LINE` (typically 128KB on Linux, but by default bash reads 128 bytes at a time)
3. `read` splits on `$IFS` — with `IFS=` (empty), no splitting occurs
4. `-r` prevents backslash interpretation
5. The `line` variable contains exactly one line (up to the newline character)
6. Memory used: ~the size of one line + the read buffer (~128KB max)

For a 1GB file with 100-byte average lines: constant ~8KB memory. Processing time: slower than awk but memory-safe.

### Awk Internals: Why It's Fast

`awk` is a compiled C program that:
1. Reads a block of data (typically 4096-8192 bytes) from the file
2. Splits into records (by default, newline-separated)
3. For each record, splits into fields (by default, whitespace-separated) using a compiled FSM
4. Applies the pattern-action pairs
5. Writes output to stdout
6. Repeats

The key: fields are NOT stored as separate strings. awk maintains pointers into the record buffer. Field access (`$1`, `$2`) is pointer arithmetic — no allocation, no copying for simple operations.

Numeric operations happen directly in C. `sum += $1` is a C addition. In bash, `sum=$((sum + line))` involves string parsing, numeric conversion, integer addition, and string conversion back.

### Sort Internals: External Merge Sort

`sort` uses a classic external merge sort:
1. Phase 1 (Reading): Read as much data as fits in the buffer (`-S` specifies the buffer size, default ~memory/8). Sort in memory (qsort). Write sorted chunk to temp file (in `/tmp`).
2. Phase 2 (Merging): Open all temp files. Read the first line from each. Find the smallest. Output it. Read the next line from that file. Repeat.
3. Phase 3 (Cleanup): Delete temp files.

For a 10GB file with 1GB buffer: 10 chunks, 10 open file descriptors, merge overhead is one comparison per output line. Total memory during merge: ~1GB buffer.

### Pipe Buffering and tee

When you pipe data `process1 | process2`:
- The kernel allocates a pipe buffer (default 64KB on Linux since 2.6.35, previously 4KB)
- `process1` writes to the pipe. If the pipe is full, `write()` blocks.
- `process2` reads from the pipe. If empty, `read()` blocks.
- The pipe decouples processes — they don't need to run in lockstep.

`tee` writes to BOTH stdout AND a file:
```bash
process1 | tee intermediate.txt | process2
```
`tee` reads from stdin, writes to stdout AND to `intermediate.txt` simultaneously. It's implemented with a simple read/write loop: read from stdin, write to each output. If the file write is slow, it blocks stdin reading, which blocks process1.

### Memory Profiling

```bash
# Track a process's memory usage
$ /usr/bin/time -v command 2>&1 | grep -E 'Maximum resident|Elapsed'
# Or real-time monitoring:
$ while kill -0 $PID 2>/dev/null; do
    cat /proc/$PID/status | grep VmRSS
    sleep 0.1
  done
```

## Core Examples (15 total)

### Example 1: Memory-efficient line processing

```bash
$ while IFS= read -r line; do
    printf '%s\n' "${line,,}"  # lowercase each line
  done < hugefile.txt > output.txt
```

**Anatomy:** `IFS=` prevents word splitting. `-r` prevents backslash interpretation. `< hugefile.txt` feeds the file line-by-line. Output is redirected to a new file.

**Variations:** Add a counter: `lineno=0; while ...; do lineno=$((lineno + 1)); ... done`. Use `head -n 1000 | while ...` for sampling.

**Edge case:** A file without a trailing newline causes `read` to fail (returns non-zero) on the LAST line, which terminates the loop. Fix: `while IFS= read -r line || [ -n "$line" ]; do ... done`.

### Example 2: Batched processing with mapfile

```bash
$ batch_size=1000
$ while mapfile -t -n "$batch_size" chunk && [ "${#chunk[@]}" -gt 0 ]; do
    for line in "${chunk[@]}"; do
      echo "${#line}"  # Process each line
    done
  done < largefile.txt
```

**Anatomy:** `mapfile -t -n 1000` reads 1000 lines into the array `chunk` (removing trailing newlines with `-t`). Process the batch. Repeat. Memory: 1000 lines' worth.

**Variations:** Process the batch in a subshell to limit variable scope. Use batches with different sizes based on available memory.

**Edge case:** `mapfile` is a bash 4.x feature. On older bash (macOS default), use a counter-based loop instead.

### Example 3: Using awk for fast field extraction

```bash
$ awk '{count[$1]++} END {for (k in count) print k, count[k]}' hugefile.log
```

**Output:**
```
ERROR 1234
WARN 567
INFO 89012
```

**Anatomy:** awk associative arrays (`count[$1]`) are hash maps implemented in C. The entire file is processed in a single pass. Memory: one entry per unique key in the first field.

**Variations:** `awk -F, '{sum+=$3} END {print "Total:", sum}' data.csv`. `awk '$2 > 100 {print $1, $2}'`.

**Edge case:** awk associative arrays use the field as a STRING key. `$1` is a string. For numeric keys, use `count[$1+0]` to force numeric conversion.

### Example 4: Using split for parallel processing

```bash
$ split -l 10000 access_log.csv chunk_
$ for f in chunk_*; do
    awk -F, '{sum+=$3} END {print FILENAME, sum}' "$f" &
  done
$ wait
$ rm chunk_*
```

**Anatomy:** `split` divides the file into 10,000-line chunks. Each chunk is processed in a background job (parallel). Results are printed to stdout (interleaved — use a temp file per chunk for clean output).

**Variations:** Use `split -n 8` to split into 8 equal parts (without knowing file size). Use `split -b 100M` for size-based split.

**Edge case:** Parallel processing is only useful if each chunk's processing is CPU-bound AND you have multiple cores. For I/O-bound tasks, parallelism may not help (disk is the bottleneck).

### Example 5: Sorting large files with memory control

```bash
$ sort -S 1G -t, -k2 -n hugefile.csv > sorted.csv
```

**Anatomy:** `-S 1G` limits sort's memory to 1GB. `-t,` sets comma as delimiter. `-k2` sorts on field 2. `-n` numeric sort. If the file is bigger than 1GB, sort writes temp files to `/tmp`.

**Variations:** `sort -S 50%` uses 50% of total RAM. `sort -T /data/tmp` uses a different directory for temp files (useful if `/tmp` is small).

**Edge case:** Sort uses `/tmp` for temp files by default. If `/tmp` is a tmpfs (memory-backed), you'll run out of memory. Set `-T /var/tmp` for disk-backed sort.

### Example 6: Streaming with named pipes

```bash
$ mkfifo /tmp/log_pipe
$ grep 'ERROR' huge.log > /tmp/log_pipe &
$ sort /tmp/log_pipe > errors_sorted.txt
$ rm /tmp/log_pipe
```

**Anatomy:** The FIFO acts as a buffer. `grep` runs in background, writing to the pipe. `sort` reads from the pipe. Both run simultaneously. No intermediate disk file needed.

**Variations:** Chain multiple filters: `grep ... | grep ... > pipe`. Use multiple pipes for complex DAGs.

**Edge case:** FIFOs block if writer fills the pipe buffer (64KB) and reader is slow. Deadlock possible if two processes are waiting on each other's pipes.

### Example 7: Processing with progress indicator

```bash
$ cat > process_with_progress.sh << 'EOF'
INPUT="$1"
TOTAL=$(wc -l < "$INPUT")
COUNT=0

while IFS= read -r line; do
  COUNT=$((COUNT + 1))
  if [ $((COUNT % 1000)) -eq 0 ]; then
    PERCENT=$((COUNT * 100 / TOTAL))
    printf "\rProgress: %d/%d (%d%%)" "$COUNT" "$TOTAL" "$PERCENT"
  fi
  process "$line"
done < "$INPUT"
echo
echo "Done: processed $COUNT lines"
EOF
```

**Anatomy:** `wc -l` pre-counts lines (reads the file once). Processing loop counts iterations and updates progress every 1000 lines. `\r` (carriage return) overwrites the same line.

**Variations:** Use `pv` for built-in progress: `pv hugefile.txt | while IFS= read -r line; do ... done`.

**Edge case:** `wc -l` on a 10GB file takes ~30 seconds. For very large files, skip the count and show only the line count without percentage.

### Example 8: Extracting specific fields with cut

```bash
$ cut -d, -f1,3,5 huge.csv | sort -t, -k2 -n > extracted.csv
```

**Anatomy:** `cut -d, -f1,3,5` extracts fields 1, 3, and 5 from comma-separated data. The output is piped directly to `sort`.

**Variations:** `cut -c1-80` extracts the first 80 characters (columns). `cut -d' ' -f1` uses space delimiter.

**Edge case:** `cut` doesn't handle quoted fields with embedded delimiters. `echo 'a,"b,c",d' | cut -d, -f2` gives `"b` instead of `b,c`. Use `awk -F, '{print $2}'` or a CSV-aware tool instead.

### Example 9: Aggregation with awk (fast)

```bash
$ awk -F, '{level=$3; val=$5; sum[level]+=val; count[level]++}
     END {for (l in sum) printf "%s: count=%d sum=%d avg=%.2f\n", l, count[l], sum[l], sum[l]/count[l]}' bigdata.csv
```

**Output:**
```
ERROR: count=25000 sum=1234567 avg=49.38
WARN: count=100000 sum=7654321 avg=76.54
INFO: count=500000 sum=25000000 avg=50.00
```

**Anatomy:** awk maintains two associative arrays (sum and count) during a single pass. The END block computes and prints averages.

**Variations:** Add min/max by tracking initial values. Add median by storing values in an array and sorting (memory-heavy for many values).

**Edge case:** awk associative arrays are limited by available memory. For millions of unique keys, awk may slow down or exhaust memory.

### Example 10: Finding the 95th percentile

```bash
$ awk '{vals[NR]=$1} END {n=asort(vals); idx=int(n*0.95); print "P95:", vals[idx]}' data.txt
```

**Anatomy:** Sorts all values in memory (with `asort`), then picks the index at 95% of the sorted array. Memory: one entry per line.

**Variations:** For files too large for in-memory sort, use a two-pass approach: sample, estimate thresholds, then count in range.

**Edge case:** `asort` is GNU awk (gawk) specific. POSIX awk doesn't have it. For portability, pipe to `sort -n | awk '...'` instead.

### Example 11: Two-pass processing

```bash
$ cat > two_pass.sh << 'EOF'
FILE="$1"

# Pass 1: Count lines and compute total
TOTAL_LINES=$(wc -l < "$FILE")
TOTAL_SUM=$(awk '{sum+=$1} END {print sum}' "$FILE")

# Pass 2: Process with normalization
while IFS= read -r value; do
  percent=$(echo "scale=4; $value * 100 / $TOTAL_SUM" | bc)
  echo "$value,$percent"
done < "$FILE" > output_with_pct.csv

echo "Processed $TOTAL_LINES lines, total=$TOTAL_SUM"
EOF
```

**Anatomy:** First pass collects aggregate stats (inexpensive with awk). Second pass uses those stats for per-line computation.

**Variations:** Three-pass: first to get count, second to compute threshold, third to apply. Use temp files for intermediate results.

**Edge case:** Two-pass means reading the file TWICE. For large files on slow I/O, this can be costly. Consider piped approaches if possible.

### Example 12: Large file random sampling

```bash
$ cat > reservoir_sample.sh << 'EOF'
# Reservoir sampling: select K random lines from a stream of unknown length
K="${1:-100}"
SAMPLE=()
SAMPLE_COUNT=0

while IFS= read -r line; do
  SAMPLE_COUNT=$((SAMPLE_COUNT + 1))
  if [ "${#SAMPLE[@]}" -lt "$K" ]; then
    SAMPLE+=("$line")
  else
    R=$((RANDOM % SAMPLE_COUNT))
    [ "$R" -lt "$K" ] && SAMPLE[$R]="$line"
  fi
done

printf '%s\n' "${SAMPLE[@]}"
EOF
$ cat hugefile.txt | bash reservoir_sample.sh 50
```

**Anatomy:** Reservoir sampling maintains a fixed-size sample (K items) while streaming through the data. Each new item has a K/N chance of replacing a current sample item. Memory: K items (constant).

**Variations:** For truly random sampling, use `shuf -n K file` (reads the whole file). Reservoir sampling works with pipes (streaming).

**Edge case:** `RANDOM` in bash only gives 15 bits (0-32767). For very large files (> 32767 lines), the RANDOM distribution is biased. Use `$RANDOM << 15 | $RANDOM` or `od -An -N2 -i /dev/urandom` for larger ranges.

### Example 13: Processing CSV with embedded newlines

```bash
$ cat > robust_csv_parser.sh << 'EOF'
# Handle CSV files with embedded newlines in quoted fields
in_record=0
while IFS= read -r line; do
  if [ "$in_record" -eq 0 ]; then
    record="$line"
  else
    record="$record
$line"
  fi

  # Count quotes — if odd, we're in a multi-line field
  quote_count=$(echo "$record" | tr -cd '"' | wc -c)
  if [ $((quote_count % 2)) -eq 0 ]; then
    # Complete record
    echo "$record" | process_csv_line
    in_record=0
  else
    in_record=1
  fi
done < "$1"
EOF
```

**Anatomy:** Counts quotes to determine if we're inside a quoted field (which may span multiple lines). Accumulates lines until quotes are balanced.

**Variations:** Use a real CSV parser (like `csvkit`'s `csvjson` or Python's `csv` module) for complex cases.

**Edge case:** This doesn't handle escaped quotes (`""`) correctly in all cases. Real CSV parsing is surprisingly complex.

### Example 14: Parallel processing with GNU parallel

```bash
$ # Process each file with a command, in parallel
$ find /var/log -name '*.log' | parallel -j4 gzip
$ # Process a large file line by line in parallel (split + parallel)
$ split -l 100000 bigfile chunk_ && parallel -j4 awk '{sum+=\$3} END {print FILENAME, sum}' ::: chunk_* > results.tmp && rm chunk_*

# Aggregate results
$ awk '{sum+=$2} END {print "Total:", sum}' results.tmp
```

**Anatomy:** `parallel -j4` runs up to 4 jobs concurrently. `::: chunk_*` passes file arguments. `\$3` escapes $ from parallel's shell.

**Variations:** `parallel -pipe` processes stdin in chunks. `parallel --block 10M` uses 10MB blocks.

**Edge case:** GNU parallel is not always installed. Check with `command -v parallel`. `xargs -P4` is POSIX and works similarly: `xargs -P4 -I{} sh -c 'process "$1"' -- {}`.

### Example 15: Monitoring memory during processing

```bash
$ cat > mem_safe_processor.sh << 'EOF'
INPUT="$1"
PEAK=0

# Background memory monitor
monitor_mem() {
  local pid=$1
  while kill -0 "$pid" 2>/dev/null; do
    rss=$(grep VmRSS /proc/$pid/status 2>/dev/null | awk '{print $2}')
    [ -n "$rss" ] && [ "$rss" -gt "$PEAK" ] && PEAK=$rss
    sleep 0.1
  done
  echo "$PEAK"
}

# Start processing in a monitored subshell
(
  monitor_pid=$!
  monitor_mem $$ &
  monitor_pid=$!

  while IFS= read -r line; do
    # process line
    :
  done < "$INPUT"
  kill "$monitor_pid" 2>/dev/null
)

echo "Peak RSS: ${PEAK}KB"
EOF
```

**Anatomy:** Background process reads `/proc/PID/status` every 100ms to track RSS. At the end, reports peak memory.

**Variations:** Use `/usr/bin/time -v` instead: `env TIME="%M" /usr/bin/time -v ./script.sh`. This gives max RSS at exit.

**Edge case:** Memory monitoring adds overhead. For production, only use when debugging memory issues.

## Real-World Use Cases

### FOR the OS

- **Log processing:** syslog aggregation, error counting, access log analysis
- **Data transformation:** ETL pipelines (extract, transform, load) with clean text formats
- **Report generation:** Daily summaries from transaction logs
- **Database exports:** Processing CSV dumps from SQL databases
- **File deduplication:** Finding duplicate files by checksum

### WITH the OS

- **DNS log analysis:** Count queries per domain, sort by frequency
- **Web server logs:** Extract unique IPs, count 404s, identify top referrers
- **CSV data merging:** Join multiple CSV files on common keys
- **System metrics aggregation:** Average CPU, disk, memory across all servers

### AGAINST THE OS

- **Log tampering:** Big file processing can hide modifications in massive logs
- **Resource exhaustion:** Intentionally crafting huge files to crash poorly-written processors
- **CSV injection:** Embedding formulas in CSV fields that execute when opened in Excel

### FOR DEFENSE

- **Log analysis for intrusion detection:** Grep auth logs for suspicious patterns
- **File integrity:** SHA256 hashing of large file sets (find -exec sha256sum)
- **Forensic analysis:** Processing packet captures (tcpdump) or audit logs
- **Anomaly detection:** Statistical analysis of system metrics over time

## Memory Aids

**"NEVER FOR-IN-CAT, ALWAYS WHILE-READ"** — The golden rule of large file processing. `for line in $(cat ...)` loads everything into memory. `while read` streams one line at a time.

**"DELEGATE TO C"** — Bash for control flow, C tools (awk, sed, grep, sort) for data processing. Bash string operations are ~100x slower than awk.

**"STREAM, DON'T SUCK"** — Keep data moving in pipelines. Write temp files to disk only when necessary. Named pipes connect processes without intermediate files.

**"SPLIT TO PARALLELIZE"** — Use `split` to chunk large files, then process chunks in parallel. Aggregate results at the end.

**"S FOR SORT BUFFER"** — `sort -S 1G` controls memory use. Default is ~12% of RAM. Without `-S`, sort grabs as much memory as it can.

## Trap Vault (15 traps)

**Trap 1:** `for line in $(cat bigfile)` loads the ENTIRE file into memory AND performs word splitting on IFS (space, tab, newline). A 1GB file can consume 2GB+ RAM and split on every space.

**Trap 2:** `while IFS= read -r line` is MEMORY-SAFE but SLOW for field processing. Each `${line:0:5}` or `IFS=, read -r f1 f2` adds overhead. Use awk for field extraction.

**Trap 3:** Without `IFS=` in `while read`, leading/trailing whitespace is trimmed. Without `-r`, backslashes are interpreted. `while read line` (without IFS=, -r) is almost always wrong for data processing.

**Trap 4:** `sort` defaults to using `/tmp` for temporary files. If `/tmp` is a tmpfs (RAM-backed), sorting a file larger than RAM will fill memory and swap. Use `-T /var/tmp` for disk-backed sort.

**Trap 5:** `tee` writes to file and stdout simultaneously. If the pipe blocks (reader is slow), `tee` buffers in memory. For very large streams, this can use significant memory.

**Trap 6:** `$(<file)` is a bash optimization that reads the file into a string WITHOUT forking `cat`. But it still loads the entire file into memory. Same memory issue as `$(cat file)`.

**Trap 7:** `mapfile` reads lines into an array. It uses memory proportional to the number of lines read. `mapfile -t -n 1000` reads 1000 lines — 1000 entries in an array. For large batches, monitor memory.

**Trap 8:** `awk` associative arrays use memory per unique key. `awk '{count[$1]++}' hugefile` with 10 million unique first-field values = ~1GB RAM. Use `sort | uniq -c` when you have many unique keys.

**Trap 9:** `cut` does NOT handle quoted CSV fields. `a,"b,c",d | cut -d, -f2` gives `"b` instead of `b,c`. Use `awk -F,` with proper quoting logic or a CSV-parsing tool.

**Trap 10:** Pipes buffer 64KB by default. If you have many pipelined commands and the total pipeline stalls, the pipe buffer fills up and the producer blocks. Add `stdbuf -oL` to force line-buffered output.

**Trap 11:** Using `sort -u` on a 10GB file uses the sort buffer + temp files. The `-u` (unique) pass happens AT THE END, after sorting. Memory for unique doesn't add much, but the sort must complete first.

**Trap 12:** `split -l 10000 bigfile prefix` creates files named `prefixaa`, `prefixab`, etc. On systems with case-insensitive filesystems (macOS), `prefixAA` and `prefixaa` are the same file.

**Trap 13:** `comm` (compare sorted files) requires sorted input. If you forget to sort: `comm: file 1 is not in sorted order`. Use `sort -u` on each file before `comm`.

**Trap 14:** `join` requires sorted input on the join field AND the input files must have the same delimiter. `join -t, -1 1 -2 1` specifies comma delimiter and field 1 for both files.

**Trap 15:** `wc -l` counts newline characters. A file without a trailing newline reports one less line than `grep -c '.*'` would find. Always ensure your data files end with a newline.

## See It In The Wild

- **Log aggregation at scale:** Elasticsearch uses Unix pipelines for log processing. Logstash (the data processing pipeline) uses patterns similar to awk/grep for field extraction.

- **Wikipedia database dumps:** Multi-gigabyte XML files processed with `awk` and `sed` for extracting article titles, revision counts, and cross-references.

- **Financial transaction processing:** Banks use Unix pipelines to reconcile millions of daily transactions. `sort` + `join` + `awk` for matching records across systems.

- **DNS analysis (DNSmeet):** Processing multi-GB DNS query logs with `cut | sort | uniq -c | sort -rn` to find top queried domains and detect data exfiltration.

- **NASA Earth data (NetCDF to CSV):** Climate data processing pipelines convert binary formats to CSV, then use Unix tools for aggregation and analysis.

- **Genome sequencing data:** Processing FASTA/Q files (often hundreds of GB) with `awk` for sequence count, GC content, and quality scores.

## Check Your Understanding (10 questions)

1. **Q:** Why does `for line in $(cat bigfile)` crash on large files? **A:** It loads the entire file into memory AND performs word splitting. A 1GB file becomes a ~2GB in-memory string that then gets split on all whitespace characters.

2. **Q:** How does `while IFS= read -r line` keep memory constant? **A:** It reads ONE line at a time via the `read` built-in. The line variable is overwritten each iteration. The file is opened as a stream (FD), not loaded into memory. Maximum memory is the longest single line.

3. **Q:** What is the advantage of writing values to per-level temp files for median/percentile calculation? **A:** You can sort each temp file independently (or use streaming sort), keeping memory proportional to one level's data rather than the entire dataset.

4. **Q:** Why is `awk` often faster than bash for field extraction? **A:** awk is compiled C, processes data in blocks (not line-by-line), uses pointer arithmetic for field access (no string allocation), and has optimized hash tables.

5. **Q:** How would you calculate the 95th percentile without loading all values into memory? **A:** Use a two-pass approach: (1) sample the data to estimate the distribution, (2) in a second pass, count how many values fall below each estimated threshold, refining until you find the true P95. Or use the `sort` utility with a memory limit.

6. **Q:** What does `sort -S 1G` do, and why is it important? **A:** Limits sort's in-memory buffer to 1GB. Data beyond that is written to temp files and merged via external sort. Prevents sort from consuming all system RAM.

7. **Q:** How does `split` enable parallel processing? **A:** Split divides a file into N chunks. Each chunk can be processed by a separate CPU core simultaneously (via background jobs or GNU parallel). Results are aggregated after all chunks complete.

8. **Q:** What's the difference between `tee file | command` and just `command` in terms of memory/I/O? **A:** `tee` writes a copy of the data to disk WHILE piping it through. This doubles I/O bandwidth but provides an intermediate file for debugging or later reuse.

9. **Q:** Why is `LC_ALL=C` often used with `sort` and `grep`? **A:** The C locale uses byte-by-byte comparison (fast) instead of locale-aware comparison (slow, considers character ordering rules). This gives 2-10x speedup for ASCII data.

10. **Q:** What is reservoir sampling and when is it useful? **A:** It selects K random items from a stream of unknown total size. It uses exactly K slots of memory (constant). Useful when you can't fit the data in memory but need a random sample.
