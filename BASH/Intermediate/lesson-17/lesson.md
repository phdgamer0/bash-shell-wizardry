# Lesson 17: Date, Time & Random

## History & Origins

**`date`** is one of the oldest Unix commands, present in **Version 1 Unix** (1971). The modern `date +%Y-%m-%d` format strings come from POSIX.2 (1992). The GNU `date` command added the `-d` option for date parsing (1993), which is NOT POSIX — a constant source of portability headaches.

**`$RANDOM`** was added to bash in **bash 1.14** (1993). It provides a simple linear congruential generator (LCG) — fast but not cryptographically secure. The name is self-explanatory: it returns a random integer. The POSIX spec doesn't require it (it's a bash extension), but virtually every modern shell has it.

**`SECONDS`** is a bash special variable (bash 2.0+, 1996) that auto-increments every second. If you set it to 0, it resets and starts counting from there. It's simpler than calling `date +%s` multiple times.

**`shuf`** was introduced in **GNU coreutils 6.0** (2006). It reads lines and shuffles them randomly — useful for sampling, random selection, and `shuf -i` for range-based random numbers. It's a modern replacement for `awk 'BEGIN{srand(); print int(rand()*100)}'`.

**`uuidgen`** comes from the **uuid-ossp** or **util-linux** package on Linux. UUIDs (Universally Unique Identifiers) are standardized by RFC 4122. They're 128-bit identifiers designed to be unique across time and space without a central registry.

The naming: `RANDOM` = random number. `SECONDS` = seconds since reset. `shuf` = "shuffle". `uuidgen` = "UUID generate".

## Syntax Reference

### date

GNU date syntax:
```
date [+format]
date -d "date string" [+format]
date -d "@epoch_seconds" [+format]
date -r filename    # last modified time of file
```

**Common format specifiers:**

| Code | Meaning | Example |
|------|---------|---------|
| `%Y` | Year (4-digit) | 2026 |
| `%y` | Year (2-digit) | 26 |
| `%m` | Month (01-12) | 07 |
| `%d` | Day of month (01-31) | 31 |
| `%H` | Hour (00-23) | 14 |
| `%M` | Minute (00-59) | 30 |
| `%S` | Second (00-60) | 45 |
| `%s` | Unix epoch seconds | 1753920000 |
| `%N` | Nanoseconds (GNU) | 123456789 |
| `%A` | Full weekday name | Thursday |
| `%a` | Abbreviated weekday | Thu |
| `%B` | Full month name | July |
| `%b` | Abbreviated month | Jul |
| `%Z` | Timezone | UTC |
| `%z` | Timezone offset | +0000 |
| `%F` | Equivalent to `%Y-%m-%d` | 2026-07-31 |
| `%T` | Equivalent to `%H:%M:%S` | 14:30:45 |

**Date arithmetic (GNU):**
```
date -d "next Friday"
date -d "last month"
date -d "+1 week"
date -d "-2 years"
date -d "2026-01-01 + 30 days"
```

### BSD/macOS date differences

| Action | GNU | BSD/macOS |
|--------|-----|-----------|
| Parse date | `date -d "2025-01-01"` | `date -j -f "%Y-%m-%d" "2025-01-01"` |
| Epoch to date | `date -d "@1753920000"` | `date -r 1753920000` |
| Date arithmetic | `date -d "+1 week"` | `date -v+1w` |
| Format | `date +%Y-%m-%d` | `date +%Y-%m-%d` (same) |

### Random number sources

| Source | Range | Quality | Use case |
|--------|-------|---------|----------|
| `$RANDOM` | 0-32767 | Low (LCG) | Test data, simple choices |
| `$(( RANDOM % N + MIN ))` | Min-Max | Low | Ranges |
| `shuf -i MIN-MAX -n 1` | Min-Max | Medium | Picking from range |
| `shuf -e a b c -n 1` | Custom list | Medium | Random choice |
| `od -An -N2 -i /dev/urandom` | 0-65535 | High | Security-sensitive |
| `openssl rand -hex N` | Hex string | High | Tokens, keys |
| `/dev/urandom` | Raw bytes | High | Cryptographic |
| `uuidgen` | UUID string | High | Unique identifiers |

### Timing

| Tool | Precision | Use |
|------|-----------|-----|
| `SECONDS` | 1 second (integer) | Simple elapsed time |
| `EPOCHREALTIME` | Microseconds (bash 5.0+) | High precision timing |
| `date +%s.%N` | Nanoseconds | Most precise |
| `TIMEFORMAT` + `time` | Milliseconds | Command timing |
| `/usr/bin/time` | Milliseconds | External timing tool |

## Under the Hood

### date: the kernel implementation

When you call `date +%s`, here's what happens:
1. `date` calls `gettimeofday(2)` or `clock_gettime(2)` — a syscall that reads the kernel's wall-clock time
2. The kernel maintains time via the **Timekeeping** subsystem — using a hardware clock (HPET, ACPI PM timer, or TSC)
3. `date` formats the `struct timeval` or `struct timespec` into a string
4. NTP (Network Time Protocol) may adjust the clock incrementally as a background service

**strace for `date`:**
```
clock_gettime(CLOCK_REALTIME, {tv_sec=1753920000, tv_nsec=123456789}) = 0
write(1, "2026-07-31\n", 11)            = 11
```

For `date -d "next friday"`:
```
# GNU date parses the string internally using its own date parser
# Then does the same clock_gettime + formatting
```

### $RANDOM: the linear congruential generator

Bash's `$RANDOM` uses the formula:
```
next = (previous * 1103515245 + 12345) & 0x7fffffff
```

This is the same LCG used by the C standard library `rand()` on glibc systems. Properties:
- Period: 2^31 (about 2 billion values before repeating)
- Lower bits are LESS random than higher bits — `$RANDOM % 2` alternates! Use `$RANDOM >> 15` or `$RANDOM / 32768` for better distribution
- The seed is derived from the PID and the pipe/process creation time
- Predictable if you know the seed — never use for security

**`/dev/urandom`**: This is the kernel's **CSPRNG** (Cryptographically Secure Pseudo-Random Number Generator). On Linux:
- Collects entropy from hardware: mouse movements, keystrokes, disk timing, network interrupts
- Uses a pool hashed with SHA-1 (older) or ChaCha20 (Linux 5.4+)
- `/dev/urandom` blocks only until the pool is initialized; after that, it never blocks
- `/dev/random` blocks if entropy estimate is low (deprecated — don't use)

### Process implications

- `date` forks every time — expensive in tight loops
- `SECONDS` is a shell variable — no fork, no overhead
- `$RANDOM` is a shell variable — no fork, very fast
- `shuf` forks — slower than `$RANDOM` but better quality
- `/dev/urandom` reads involve a syscall but no fork

## Core Examples (8-12 minimum)

### Example 1: Date formatting

**Command:**
```bash
date +"%Y-%m-%d %H:%M:%S"
date +"%A, %B %d, %Y"
date +%s
```

**Output:**
```
2026-07-31 14:30:00
Thursday, July 31, 2026
1753920000
```

**Step-by-step:**
1. First format: year-month-day hour:minute:second
2. Second format: full weekday, full month, day, year
3. Third: Unix epoch timestamp (seconds since 1970-01-01 00:00:00 UTC)

**Variations:**
- `date +%F` = `%Y-%m-%d` (ISO 8601 date)
- `date +%T` = `%H:%M:%S` (ISO 8601 time)
- `date +%Y%m%d_%H%M%S` — sortable filename timestamp

### Example 2: Parsing dates with -d (GNU)

**Command:**
```bash
date -d "2025-01-01" +%A
date -d "next Friday"
date -d "@1753920000"
date -d "last month" +%Y-%m
```

**Output:**
```
Wednesday
Fri Aug 1 00:00:00 UTC 2026
Thu Jul 31 14:30:00 UTC 2026
2026-06
```

**Step-by-step:**
1. `-d "2025-01-01"` — parse specific date, format as weekday
2. `-d "next Friday"` — relative date parsing
3. `-d "@1753920000"` — convert epoch to human-readable
4. `-d "last month"` — relative month calculation

**Variations:**
- `date -d "2026-01-01 +30 days"` — date arithmetic
- `date -d "3 months ago" +%F` — three months back
- `date -d "12:00 next Sunday"` — specific time on a future day

### Example 3: $RANDOM

**Command:**
```bash
echo $RANDOM
echo $((RANDOM % 100 + 1))   # 1-100
echo $((RANDOM % 6 + 1))     # dice roll
echo $((RANDOM % 2))          # coin flip (biased toward 0!)
```

**Output:**
```
28471
57
4
0
```

**Step-by-step:**
1. `$RANDOM` returns integer 0-32767
2. `% 100` maps to 0-99, `+1` shifts to 1-100
3. `% 6` gives 0-5, `+1` gives 1-6 (dice roll)
4. `% 2` gives 0 or 1 (coin flip — but flawed, see traps)

**Variations:**
- `echo $((RANDOM >> 15))` — better coin flip (uses high bits)
- `echo $((RANDOM * 1000 / 32768))` — better distribution for 0-999
- `for i in {1..6}; do echo "$((RANDOM % 49 + 1))"; done` — lottery numbers (may repeat!)

### Example 4: shuf for better randomness

**Command:**
```bash
shuf -i 1-100 -n 3          # pick 3 unique numbers
shuf -e apple banana cherry -n 1   # pick one from list
shuf -i 1-49 -n 6 | sort -n  # lottery numbers (sorted)
```

**Output:**
```
42
17
89

banana

5 12 23 34 41 48
```

**Step-by-step:**
1. `-i 1-100` — range 1 to 100
2. `-n 3` — pick 3 unique numbers (no repeats!)
3. `-e` — echo the given arguments as input lines
4. `sort -n` — sorts the lottery numbers numerically

**Variations:**
- `shuf -n 1 file.txt` — random line from file
- `shuf file.txt` — shuffle all lines (randomize order)
- `shuf -r -n 10 -i 1-3` — with replacement (allows repeats)

### Example 5: uuidgen

**Command:**
```bash
uuidgen
uuidgen -t   # time-based UUID
uuidgen -r   # random-based UUID
```

**Output:**
```
550e8400-e29b-41d4-a716-446655440000
```

**Step-by-step:**
1. `uuidgen` generates a UUID v4 (random) by default
2. UUID format: 8-4-4-4-12 hex digits = 36 characters
3. 122 random bits (v4) or time-based + MAC (v1)

**Variations:**
- `uuidgen | tr '[:upper:]' '[:lower:]'` — lowercase UUID
- `echo $(uuidgen | tr -d '-')` — compact UUID (no dashes)
- `od -An -N16 -tx1 /dev/urandom | tr -d ' \n'` — UUID alternative

### Example 6: SECONDS for benchmarking

**Command:**
```bash
SECONDS=0
sleep 2.5
echo "Elapsed: $SECONDS seconds"
```

**Output:** `Elapsed: 2.503 seconds`

**Step-by-step:**
1. `SECONDS=0` resets the counter
2. `SECONDS` is automatically incremented by bash each second (with fractional granularity)
3. After `sleep 2.5`, `SECONDS` shows ~2.5
4. No fork — purely a shell variable

**Variations:**
- `SECONDS=0; for i in {1..1000}; do :; done; echo $SECONDS` — benchmark a loop
- `start=$SECONDS; ...; echo $((SECONDS - start))` — lap timer
- Save to array: `times+=($SECONDS)` — collect multiple samples

### Example 7: TIMEFORMAT

**Command:**
```bash
TIMEFORMAT='Elapsed: %R seconds (user: %U, sys: %S)'
time sleep 1
```

**Output:** `Elapsed: 1.002 seconds (user: 0.001, sys: 0.000)`

**Step-by-step:**
1. `TIMEFORMAT` customizes the `time` builtin's output
2. `%R` = real (wall clock) time
3. `%U` = user-space CPU time
4. `%S` = kernel (system) CPU time
5. `time` measures the entire pipeline, not just individual commands

**Variations:**
- `TIMEFORMAT='%R'` — just the real time
- `TIMEFORMAT=$'\nreal\t%R\nuser\t%U\nsys\t%S'` — tab-separated
- Bash builtin `time` vs `/usr/bin/time` — different output

### Example 8: Timestamp filenames

**Command:**
```bash
backup_file="backup_$(date +%Y%m%d_%H%M%S).tar.gz"
echo "$backup_file"
```

**Output:** `backup_20260731_143000.tar.gz`

**Step-by-step:**
1. `date +%Y%m%d_%H%M%S` produces `20260731_143000`
2. Command substitution embeds it in the filename
3. Sortable alphabetically — `ls` shows chronological order
4. Unique if generated at different times

**Variations:**
- `date +%F_%H-%M-%S` — ISO format with hyphens
- `date +%s` — epoch timestamp (also sortable)
- `date +%Y/%m/%d` — directory structure for logs

### Example 9: Dice rolling simulation

**Command:**
```bash
roll_dice() {
    local sides=${1:-6}
    local count=${2:-1}
    local rolls=()
    for ((i=0; i<count; i++)); do
        rolls+=($((RANDOM % sides + 1)))
    done
    echo "${rolls[@]}"
}

roll_dice 6 5      # roll 5d6
roll_dice 20 1     # roll 1d20
```

**Output:**
```
4 2 6 1 3
17
```

**Step-by-step:**
1. `RANDOM % sides` gives 0 to sides-1
2. `+1` shifts to 1 to sides
3. Loop generates the requested number of rolls

**Variations:**
- Weighted: use `shuf` with weighted input list
- Exploding dice: re-roll on max value
- With advantage (D&D): `max=$(roll_dice 20 2 | tr ' ' '\n' | sort -n | tail -1)`

### Example 10: Countdown timer

**Command:**
```bash
countdown() {
    local total=$1
    local m s
    while (( total > 0 )); do
        m=$((total / 60))
        s=$((total % 60))
        printf '\r%02d:%02d remaining' "$m" "$s"
        sleep 1
        ((total--))
    done
    printf '\r%02d:%02d remaining\n' 0 0
    echo "Time's up!"
}
countdown 10
```

**Output:** (Counts down from 00:10 to 00:00 on the same line)
```
00:10 remaining → 00:09 remaining → ... → 00:00 remaining
Time's up!
```

**Step-by-step:**
1. `total` is the remaining seconds
2. `m = total / 60` (minutes)
3. `s = total % 60` (seconds)
4. `\r` overwrites the same line each second
5. `sleep 1` advances real time

**Variations:**
- Accept `MM:SS` format: split on `:`, calculate `m * 60 + s`
- Add a bell at the end: `echo -e '\a'`
- Allow pause/resume with keypress

### Example 11: Log rotation with dates

**Command:**
```bash
rotate_log() {
    local logfile=$1
    local date_suffix=$(date -r "$logfile" +%Y%m%d)
    local newname="${logfile%.log}-${date_suffix}.log"
    mv "$logfile" "$newname"
    gzip "$newname"
    touch "$logfile"
    echo "Rotated: $logfile → $newname.gz"
}
rotate_log /tmp/app.log
```

**Output:**
```
Rotated: /tmp/app.log → /tmp/app-20260731.log.gz
```

**Step-by-step:**
1. `date -r` gets the file's last modification time
2. Formats it as `%Y%m%d` for the suffix
3. Renames `app.log` to `app-20260731.log`
4. Compresses with `gzip`
5. Creates fresh empty log

**Variations:**
- Find logs older than N days: `find /var/log -name '*.log' -mtime +30 -exec gzip {} \;`
- Keep N compressed logs: count them, delete oldest
- Date-stamp in the filename at creation time: `app_$(date +%F).log`

### Example 12: EPOCHREALTIME for precise timing

**Command:**
```bash
echo "Start: $EPOCHREALTIME"
sleep 0.123
echo "End:   $EPOCHREALTIME"
```

**Output:**
```
Start: 1753920000.123456
End:   1753920000.246789
```

**Step-by-step:**
1. `EPOCHREALTIME` (bash 5.0+) gives epoch seconds with microsecond precision
2. No subshell — it's a shell variable, not a command
3. Subtracting gives elapsed time in microseconds
4. Useful for benchmarks, profiling, and timeout calculations

**Variations:**
- `elapsed=$(bc <<< "$EPOCHREALTIME - $start")` — calculate difference with bc
- Nanoseconds: `date +%s.%N` for nanosecond precision (but forks)

## Real-World Use Cases

### FOR the OS

- **Log rotation**: `log-$(date +%F).log` — daily log files
- **Backup naming**: `backup-$(date +%Y%m%d_%H%M%S).tar.gz` — unique, sortable
- **Session management**: Generate session IDs with `uuidgen` or `/dev/urandom`
- **Rate limiting**: Track timestamps of requests, compare with current time
- **Scheduled tasks**: `at` and `cron` jobs, determining next run time

### WITH the OS

- **Benchmarking**: Measure script performance with `SECONDS` or `EPOCHREALTIME`
- **Password generation**: `tr -dc 'A-Za-z0-9' < /dev/urandom | head -c 20`
- **File sampling**: `shuf -n 100 hugefile.txt` — pick random lines from a file
- **Unique temp files**: `mktemp /tmp/myapp.XXXXXXXXXX` uses random chars
- **Anomaly detection**: Compare timestamps of log entries, detect gaps

### AGAINST THE OS (Security Perspective)

- **`$RANDOM` predictability**: If an attacker knows the seed (often derived from PID), they can predict all "random" values. Never use for tokens, sessions, or passwords.
- **Timing attacks**: Scripts that compare timestamps for authentication (e.g., "code was generated within 30 seconds") can be exploited with timing side-channels.
- **`date` for TOCTOU**: Using `date` to generate temporary filenames is racy — two scripts running at the same microsecond could collide. Use `mktemp` instead.
- **`/dev/urandom` exhaustion**: On embedded systems (routers, IoT), the entropy pool may be low at boot — `/dev/urandom` still produces output but it may be less random.
- **UUID predictability**: UUID v1 (time-based) includes the MAC address and timestamp — an attacker can determine when and where the UUID was generated.
- **`date` determinism**: Date commands return the same value within the same second — don't use `date +%s` as a unique identifier in tight loops.

### FOR DEFENSE

- **Use `/dev/urandom` for any security-sensitive randomness** (passwords, tokens, keys)
- **Avoid `$RANDOM` for security** — use `openssl rand` or `gpg --gen-random`
- **Use `mktemp` over `date`-based temp filenames** — atomic creation prevents races
- **Check return codes of `date -d`** — invalid date strings are silently interpreted differently by GNU vs BSD
- **Use `EPOCHREALTIME` with `bc`** for high-precision timing comparisons
- **Set `TZ=UTC` before date commands** for consistent, timezone-independent timestamps

## Memory Aids

- **`%Y`** = "Year" (4-digit), **`%y`** = "year" (2-digit)
- **`%m`** = "month", **`%d`** = "day"
- **`%H`** = "Hour" (24h), **`%I`** = "I-nternational" (12h) — actually "I" stands for... it's confusing. Just remember `%H` = 24-hour.
- **`%s`** = "seconds since epoch" (lowercase s = "small" = epoch seconds)
- **`$RANDOM`** = "RANDOM number" (all caps because it's a builtin variable)
- **`SECONDS`** = seconds since reset — "how many SECONDS has it been?"
- **`shuf`** = "SHUFfle" (missing the 'f' and an extra 'f')
- **`uuidgen`** = "UUID GENerate"

## Trap Vault (8-12 traps)

### Trap 1: `$RANDOM` lower bits are not random

**Problem:** `$((RANDOM % 2))` always alternates.

**Bad Example:**
```bash
for i in {1..10}; do
    echo $((RANDOM % 2))
done
# May show: 0 1 0 1 0 1 0 1 0 1
```

**Root Cause:** LCGs have poor randomness in the lowest bits. The alternating pattern is a known issue with `rand() % N`.

**Fix:** Use higher bits:
```bash
for i in {1..10}; do
    echo $((RANDOM >> 14))    # 0 or 1 from the high bits
done

# Or use shuf:
shuf -i 0-1 -n 10
```

### Trap 2: `date -d` is NOT portable

**Problem:** Script using `date -d "2025-01-01"` fails on macOS.

**Bad Example:**
```bash
# GNU-specific:
future=$(date -d "+30 days" +%F)
```

**Root Cause:** `-d` is a GNU extension. BSD/macOS uses `-j -f` completely different syntax.

**Fix:** Detect the OS or use a portable approach:
```bash
if date -d "2025-01-01" >/dev/null 2>&1; then
    # GNU date
    future=$(date -d "+30 days" +%F)
else
    # BSD date
    future=$(date -v+30d +%F)
fi
```

### Trap 3: `$RANDOM` in a subshell always returns same seed

**Problem:** Multiple `$RANDOM` calls in subshells give the same values.

**Bad Example:**
```bash
for i in {1..3}; do
    echo $(echo $RANDOM)  # may return same value!
done
```

**Root Cause:** Each subshell re-seeds `$RANDOM` from the same source in rapid succession, potentially getting the same seed.

**Fix:** Use `$RANDOM` in the parent shell, pass as argument:
```bash
for i in {1..3}; do
    r=$RANDOM
    echo $(echo "$r")
done
```

### Trap 4: `uuidgen` UUIDs can collide (rarely)

**Problem:** Two systems generate the same UUID.

**Root Cause:** UUID v4 (random) has 122 random bits — collisions are possible at ~2.7×10^18 UUIDs for 50% collision probability. Theoretical but possible.

**Fix:** For most applications, collisions are astronomically unlikely. For extreme requirements, use a central authority.

### Trap 5: `sleep 1` is not exactly 1 second

**Problem:** Timing loops drift.

**Bad Example:**
```bash
for i in {1..60}; do
    echo "Minute $i elapsed"
    sleep 1   # actually ~1.002s to 1.010s
done
# After 60 iterations, actual time is ~60.3s, not 60s
```

**Root Cause:** `sleep` guarantees MINIMUM time, not exact. Scheduling delays, load, and timer granularity add overhead.

**Fix:** Use `SECONDS` or `EPOCHREALTIME` to compensate:
```bash
target=$((SECONDS + 60))
while (( SECONDS < target )); do
    echo "Still waiting..."
    sleep 0.1
done
```

### Trap 6: Timestamp collisions in loops

**Problem:** Multiple files created in the same second have the same timestamp.

**Bad Example:**
```bash
for i in {1..100}; do
    touch "file_$(date +%Y%m%d_%H%M%S).txt"
done
# All files have the same timestamp!
```

**Root Cause:** `date +%S` has second granularity. Loop iterations within the same second produce identical timestamps.

**Fix:** Add a counter or use nanoseconds:
```bash
for i in {1..100}; do
    touch "file_$(date +%Y%m%d_%H%M%S_%N).txt"  # nanoseconds
done
# Or just add the loop index:
touch "file_$i.txt"
```

### Trap 7: `SECONDS` only has one-second resolution

**Problem:** `SECONDS` shows 0 for operations faster than 0.5 seconds.

**Bad Example:**
```bash
SECONDS=0
echo "hello"
echo $SECONDS   # often 0!
```

**Root Cause:** `SECONDS` resolution is 1 second (though bash uses fractional internally, the variable only updates on command boundaries).

**Fix:** Use `date +%s.%N` or `EPOCHREALTIME` for sub-second precision:
```bash
start=$EPOCHREALTIME
echo "hello"
elapsed=$(echo "$EPOCHREALTIME - $start" | bc)
echo "$elapsed seconds"
```

### Trap 8: `RANDOM` without seed in scripts

**Problem:** Multiple runs of the same script produce the same "random" sequence.

**Bad Example:**
```bash
#!/bin/bash
# Always produces the same sequence at startup!
for i in {1..5}; do
    echo $RANDOM
done
```

**Root Cause:** Bash seeds `$RANDOM` based on PID + startup time. Scripts started in the same second with sequential PIDs can start with the same seed.

**Fix:** Manually re-seed:
```bash
RANDOM=$((SECONDS + $$))   # Re-seed
```

### Trap 9: `shuf` with large files

**Problem:** `shuf hugefile.txt` uses lots of memory.

**Bad Example:**
```bash
shuf -n 1 /var/log/syslog  # reads entire file into memory!
```

**Root Cause:** `shuf` must read all input lines into memory to shuffle them uniformly.

**Fix:** For random line from large file, use reservoir sampling:
```bash
awk 'BEGIN{srand()} rand() < 1/NR {line=$0} END{print line}' /var/log/syslog
```

### Trap 10: Date arithmetic across DST transitions

**Problem:** Adding days across Daylight Saving Time changes gives unexpected results.

**Bad Example:**
```bash
# In a timezone that observes DST:
date -d "2026-03-08 +1 day" +%F
# March 8, 2026 is the spring-forward day in US — 23 hours
# Adding 1 day adds 24 hours, not 1 calendar day
```

**Root Cause:** `+1 day` means "+24 hours", not "next calendar day." With DST transitions, a day is 23 or 25 hours.

**Fix:** Use calendar-aware arithmetic when you need calendar days:
```bash
# Some versions handle this correctly, others don't
# For safety, use: +1 day at noon
date -d "2026-03-08 12:00 +1 day" +%F
```

### Trap 11: `uuidgen` not installed

**Problem:** `uuidgen: command not found` in minimal containers.

**Bad Example:**
```bash
id=$(uuidgen)  # may fail in Docker
```

**Root Cause:** `uuidgen` is part of `util-linux` — not always present in minimal images.

**Fix:** Provide fallback:
```bash
if command -v uuidgen >/dev/null; then
    id=$(uuidgen)
else
    id=$(od -An -N16 -tx1 /dev/urandom | tr -d ' \n')
fi
```

### Trap 12: `TIMEFORMAT` doesn't affect `/usr/bin/time`

**Problem:** Setting `TIMEFORMAT` doesn't change `/usr/bin/time` output.

**Bad Example:**
```bash
TIMEFORMAT='%R'
/usr/bin/time sleep 1  # NOT affected by TIMEFORMAT!
```

**Root Cause:** `TIMEFORMAT` only affects the bash BUILTIN `time`. The external `/usr/bin/time` uses `-f` for its format string.

**Fix:** Use the bash builtin (no path):
```bash
TIMEFORMAT='%R'
time sleep 1   # bash builtin — uses TIMEFORMAT
```

## See It In The Wild

- **`/etc/cron.daily/logrotate`** — date-stamped log files
- **`/usr/bin/mktemp`** — uses random characters from `$RANDOM` or `/dev/urandom`
- **`/var/log/syslog`** — every line has a timestamp from the system
- **Docker container IDs** — 64-character hex strings from `/dev/urandom`
- **Git commit hashes** — SHA-1 from content, not random, but serves similar identification purpose

**Try this now:**

1. `for i in {1..5}; do echo "$((RANDOM % 6 + 1))"; done` — roll virtual dice
2. `shuf -e "yes" "no" -n 1` — random yes/no decision maker
3. `echo "Password: $(tr -dc 'A-Za-z0-9!@#$%^&*' < /dev/urandom | head -c 16)"` — generate a password
4. `SECONDS=0; sleep $(echo "scale=2; $RANDOM/32767" | bc); echo $SECONDS` — random-duration sleep

## Check Your Understanding (5-7 questions)

1. How do you get the current Unix epoch timestamp?
2. Why is `$RANDOM` unsuitable for generating passwords?
3. What does the `SECONDS` variable do, and how is it reset?
4. How does `date -d` differ between GNU and BSD systems?
5. How would you generate a random number between 50 and 100?
6. What's the difference between `uuidgen` and `tr </dev/urandom`?
7. Why might `$((RANDOM % 2))` give non-random results?
8. How do you get sub-second timing in bash without forking?

---
*"Time is what prevents everything from happening at once. `$RANDOM` is what ensures it happens in an interesting order."*
