# Task 17: Date, Time & Random

You'll build a countdown timer, a log rotation tool, a random password generator, and a benchmarking script. These are the utilities you'll reach for daily — timestamps for filenames, random data for testing, and timing for performance measurement.

## Sub-tasks

### 1. Countdown timer — `timer.sh`

Write a countdown timer with visual display.

**Requirements:**
- Accept time in seconds (numeric) or `MM:SS` format
- Display countdown on a single line using `\r`
- Format: `MM:SS remaining` (colorful)
- Show the last 5 seconds in RED (`\e[31m`), others in default
- Play a terminal bell with `echo -e '\a'` at the end
- Show "TIME'S UP!" when finished

**Edge cases:**
- Zero or negative input → error message
- `MM:SS` with invalid minutes/seconds → error
- Maximum: 99:59 (5999 seconds)
- Interrupt with Ctrl+C → show how much time was remaining

**Expected:**
```bash
$ ./timer.sh 10
00:10 remaining
00:09 remaining
...
00:05 remaining   ← turns red for last 5
00:04 remaining
...
00:00 remaining
TIME'S UP! (bell rings)

$ ./timer.sh 5:30
05:30 remaining
05:29 remaining
...

$ ./timer.sh -5
Error: Time must be positive

$ ./timer.sh 99:99
Error: Invalid time format (use MM:SS or seconds)

$ ./timer.sh 60
# Press Ctrl+C after 10 seconds
Interrupted at 00:50 remaining!
```

### 2. Log rotation — `logrotate.sh`

Write a script that rotates, compresses, and prunes log files.

**Requirements:**
- Accept a log directory and retention days
- Default retention: 30 days, default directory: `./logs`
- Also accept `-c` to compress files older than N days (default: 7)
- For each `.log` file:
  - Rename `app.log` → `app-YYYY-MM-DD.log` (using file's mtime)
  - Compress with `gzip` if older than compress days
  - Delete if older than retention days
- Show what it WOULD do before doing it (`--dry-run`)
- Print a summary of actions taken

**Edge cases:**
- No log files found → "No log files to process"
- Directory doesn't exist → error
- Already rotated files (with date in name) → skip them
- File with no `.log` extension → ignore

**Expected:**
```bash
$ ./logrotate.sh /var/log/myapp 30
Processing /var/log/myapp...
  app.log → app-2026-07-31.log
  Compressing app-2026-07-24.log.gz (7+ days old)
  Deleting app-2026-06-01.log (older than 30 days)
Summary: 1 rotated, 1 compressed, 1 deleted

$ ./logrotate.sh /var/log/myapp 30 --dry-run
DRY RUN: Would rotate: app.log
DRY RUN: Would compress: app-2026-07-24.log
DRY RUN: Would delete: app-2026-06-01.log

$ ./logrotate.sh /tmp/nonexistent
Error: Directory /tmp/nonexistent not found
```

### 3. Random password generator — `genpass.sh`

Write a password generator with multiple options.

**Requirements:**
- Default length: 16 characters
- Default count: 5 passwords
- Options:
  - `-l N` or `--length N` — password length
  - `-c N` or `--count N` — number of passwords
  - `--no-special` — omit special characters
  - `--no-upper` — omit uppercase letters
  - `--no-lower` — omit lowercase letters
  - `--no-digits` — omit digits
  - `--source METHOD` — `urandom` (default), `openssl`, `shuf`
- Each on its own line
- Print summary at end: "Generated 5 passwords of length 16"

**Edge cases:**
- Length < 4 → error "Minimum length is 4"
- All character types excluded → error "No character types selected"
- Source method not found → gracefully fall back to `urandom`

**Character sets:**
- Upper: `A-Z`
- Lower: `a-z`
- Digits: `0-9`
- Special: `!@#$%^&*()-=_+{}[]|;:,.<>?/~`

**Expected:**
```bash
$ ./genpass.sh
aB3$xK9#mQ2pR7&v
W1*nY4@zH8)jL0-c
F5^sG7=tR2%wK9*y
X4#vB1+aM8&qN0-z
J3$kL9-pV5^rT7!w
Generated 5 passwords of length 16

$ ./genpass.sh -l 8 -c 3 --no-special
aB3xK9mQ
W1nY4zH8
F5sG7tR2
Generated 3 passwords of length 8

$ ./genpass.sh -l 32 --source openssl
f8a2c3d4e5b6f7a8c9d0e1f2a3b4c5d6
...
Generated 5 passwords of length 32

$ ./genpass.sh -l 2
Error: Minimum length is 4

$ ./genpass.sh --no-upper --no-lower --no-digits --no-special
Error: No character types selected
```

### 4. Benchmark script — `benchmark.sh`

Write a script that benchmarks command execution time.

**Requirements:**
- Accept a command to run (all remaining arguments)
- Run the command N times (default: 5, option: `-n N`)
- Use `EPOCHREALTIME` or `date +%s.%N` for timing
- Show each run's time
- Calculate and display:
  - Min (fastest)
  - Max (slowest)
  - Average (mean)
  - Median (middle value when sorted)
- Accept `--quiet` to show only the stats, not individual runs
- Accept `--csv` for machine-readable output

**Edge cases:**
- Command not found → error (use `command -v`)
- Command returns non-zero → note it in results (but still include timing)
- Very fast commands (<0.001s) → ensure meaningful timing with enough iterations
- No arguments → error

**Expected:**
```bash
$ ./benchmark.sh sleep 1
Benchmarking: sleep 1 (5 runs)
Run 1: 1.003s
Run 2: 1.001s
Run 3: 1.002s
Run 4: 1.002s
Run 5: 1.001s
---
Min:    1.001s
Max:    1.003s
Avg:    1.002s
Median: 1.002s

$ ./benchmark.sh -n 3 ls /tmp
Benchmarking: ls /tmp (3 runs)
Run 1: 0.002s
Run 2: 0.001s
Run 3: 0.002s
---
Min:    0.001s
Max:    0.002s
Avg:    0.002s
Median: 0.002s

$ ./benchmark.sh --quiet -- csv ls
min,max,avg,median
0.001,0.003,0.002,0.002

$ ./benchmark.sh nonexistent_cmd
Error: Command not found: nonexistent_cmd
```

## Solution Approaches

### Approach A: Simple to complex
1. `timer.sh` — simplest, uses `date` and `sleep`
2. `genpass.sh` — `/dev/urandom` reading
3. `logrotate.sh` — `date -r` and file operations
4. `benchmark.sh` — `EPOCHREALTIME` and statistics

### Approach B: Utility-focused
1. `genpass.sh` — most likely to be used daily
2. `timer.sh` — practical tool
3. `logrotate.sh` — sysadmin tool
4. `benchmark.sh` — performance measurement

### Approach C: Algorithm-heavy
1. `benchmark.sh` — statistics calculation
2. `logrotate.sh` — date-based file management
3. `genpass.sh` — character set manipulation
4. `timer.sh` — display formatting

<details>
<summary>Hint 1: Timer with MM:SS parsing</summary>

```bash
parse_time() {
    local input="$1"
    if [[ "$input" =~ ^[0-9]+$ ]]; then
        total=$input
    elif [[ "$input" =~ ^([0-9]{1,2}):([0-9]{2})$ ]]; then
        total=$((10#${BASH_REMATCH[1]} * 60 + 10#${BASH_REMATCH[2]}))
    else
        echo "Invalid format"; exit 1
    fi
    echo $total
}

countdown() {
    local total=$1
    while (( total > 0 )); do
        local m=$((total / 60))
        local s=$((total % 60))
        if (( total <= 5 )); then
            printf '\r\e[31m%02d:%02d remaining\e[0m' "$m" "$s"
        else
            printf '\r%02d:%02d remaining' "$m" "$s"
        fi
        sleep 1
        ((total--))
    done
    printf '\r\e[32m%02d:%02d remaining\e[0m\n' 0 0
    echo -e '\aTIME'\''S UP!'
}
```
</details>

<details>
<summary>Hint 2: Log rotation with date -r</summary>

```bash
rotate_logs() {
    local dir="$1"
    local retention=$2
    local compress_days=$3
    local now=$(date +%s)

    find "$dir" -name '*.log' ! -name '*-*-*.log' | while read log; do
        local mtime=$(date -r "$log" +%s)
        local age=$(( (now - mtime) / 86400 ))
        local date_suffix=$(date -r "$log" +%F)
        local newname="${log%.log}-${date_suffix}.log"

        mv "$log" "$newname"
        echo "  $log → $newname"

        if (( age > compress_days )); then
            gzip "$newname"
            echo "  Compressed $newname"
        fi
    done

    find "$dir" -name '*.log.gz' -mtime +$retention -delete
}
```
</details>

<details>
<summary>Hint 3: Password generator with /dev/urandom</summary>

```bash
genpass_urandom() {
    local chars="$1"
    local len="$2"
    tr -dc "$chars" < /dev/urandom | head -c "$len"
    echo
}

# Build character set dynamically:
upper='A-Z'
lower='a-z'
digits='0-9'
special='!@#$%^&*()-=_+{}[]|;:,.<>?/~'

chars=""
use_upper=1; use_lower=1; use_digits=1; use_special=1

[[ "$use_upper" == 1 ]] && chars+="$upper"
[[ "$use_lower" == 1 ]] && chars+="$lower"
[[ "$use_digits" == 1 ]] && chars+="$digits"
[[ "$use_special" == 1 ]] && chars+="$special"

# Escaping for tr:
chars_escaped=$(printf '%s\n' "$chars" | sed 's/[][!@#$%^&*()_+{}|:<>?,.\/;=-]/\\&/g')
# Or use LC_ALL=C tr:
LC_ALL=C tr -dc "$chars" < /dev/urandom | head -c "$len"
```
</details>

<details>
<summary>Hint 4: Benchmark with EPOCHREALTIME</summary>

```bash
benchmark() {
    local cmd=("$@")
    local runs=5
    local times=()

    for ((i=0; i<runs; i++)); do
        local start=$EPOCHREALTIME
        "${cmd[@]}" >/dev/null 2>&1
        local end=$EPOCHREALTIME
        times+=($(echo "$end - $start" | bc -l))
    done

    # Sort times
    IFS=$'\n' sorted=($(sort -n <<<"${times[*]}"))
    unset IFS

    local min=${sorted[0]}
    local max=${sorted[-1]}

    # Average
    local sum=0; for t in "${times[@]}"; do sum=$(echo "$sum + $t" | bc -l); done
    local avg=$(echo "$sum / $runs" | bc -l)

    # Median
    local mid=$((runs / 2))
    local median
    if (( runs % 2 == 0 )); then
        median=$(echo "(${sorted[mid-1]} + ${sorted[mid]}) / 2" | bc -l)
    else
        median=${sorted[mid]}
    fi

    printf "Min:    %.3fs\n" "$min"
    printf "Max:    %.3fs\n" "$max"
    printf "Avg:    %.3fs\n" "$avg"
    printf "Median: %.3fs\n" "$median"
}
```
</details>

<details>
<summary>Hint 5: Sort with bc for floating point</summary>

```bash
# bc comparison doesn't exist directly — use sort -n instead:
sorted=($(printf '%s\n' "${times[@]}" | sort -n))
```
</details>

<details>
<summary>Hint 6: CSV output formatting</summary>

```bash
if [[ "$output_csv" == 1 ]]; then
    printf 'min,max,avg,median\n'
    printf '%.3f,%.3f,%.3f,%.3f\n' "$min" "$max" "$avg" "$median"
else
    # Pretty output
fi
```
</details>

## Bonus Challenges

1. **Stopwatch mode**: `./timer.sh --stopwatch` that counts UP instead of down
2. **Time zones**: `logrotate.sh --tz UTC` that uses a specific timezone for date stamps
3. **Password strength meter**: In `genpass.sh`, calculate entropy and display a strength rating
4. **Benchmark comparison**: Save results and compare with previous runs
5. **Parallel timer**: `./timer.sh --pomodoro 25:5` — 25 min work, 5 min break, repeating
6. **Random file selector**: `./genpass.sh --files /path` — random file picker using `shuf`

## Expected Output Summary

```
timer.sh:
  Countdown from seconds or MM:SS
  Last 5 seconds in red
  Terminal bell at finish
  Interrupt shows remaining time

logrotate.sh:
  Renames logs with date from mtime
  Compresses old logs (>7 days)
  Deletes very old logs (>30 days)
  Dry-run mode
  Summary of actions

genpass.sh:
  Configurable length, count, character types
  Multiple random sources
  Error handling for invalid options
  Summary line

benchmark.sh:
  Runs command N times
  Reports min, max, avg, median
  Quiet mode for stats-only
  CSV output option
```

## Self-Check

- How do you get the current Unix epoch timestamp?
- Why is `$RANDOM` unsuitable for generating passwords?
- What does the `SECONDS` variable do, and how is it reset?
- How does `date -d` differ between GNU and BSD systems?
- How would you generate a random number between 50 and 100?
- What's the advantage of `/dev/urandom` over `$RANDOM`?
- How do you get nanosecond-precision timestamps in bash?
- What's the difference between `date +%s` and `date +%N`?
