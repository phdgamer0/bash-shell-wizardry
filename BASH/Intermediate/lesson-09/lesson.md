# Lesson 9: String Manipulation — Internals vs Externals

## History & Origins

String manipulation in shell scripting has always been a tug-of-war between built-in operations and external tools.

The **Bourne shell** (1979) had almost no string manipulation — you had to use `sed`, `awk`, `tr`, `cut`, or `expr` for everything. Each of these forked a new process.

The **Korn shell (ksh88)** pioneered built-in string operations: `${#var}`, `${var#pattern}`, `${var%pattern}`, and pattern replacement. These were inspired by the observation that most string operations in scripts were *simple* (extract a field, strip a prefix, change case) and didn't need the full power of `sed`/`awk`.

**Bash** adopted these in 2.0 and added case modification in 4.0. But external tools still excel at:
- Processing very large files (streaming, line-by-line)
- Complex regex operations
- Columnar/structured data
- Multi-file processing

The modern philosophy: **use the right tool for the job**. If you're extracting the filename from a path, `${path##*/}` is perfect. If you're parsing a 10GB CSV file, `awk` is the right choice.

## When to Use What

| Task | Bash Built-in | External Tool |
|------|---------------|---------------|
| Substring removal | `${var#pat}`, `${var%pat}` | `sed`, `cut` |
| Replace all | `${var//old/new}` | `sed 's/old/new/g'` |
| Case conversion | `${var,,}`, `${var^^}` | `tr A-Z a-z` |
| Regex matching | `[[ $var =~ regex ]]` | `grep -E` |
| Field splitting | `IFS` + `read -ra` | `awk '{print $N}'` |
| Complex parsing | — | `awk`, `sed` |
| Multi-file search | — | `grep -r` |
| Large file processing | Slow (per-line read) | `awk` / `sed` (fast C) |
| Statistical operations | — | `awk` |
| JSON/CSV/XML | — | `jq`, `csvkit`, `xmlstarlet` |

## Performance Comparison

### Fork Overhead

Every external command call costs:
1. **fork()**: ~50-200μs (create child process)
2. **execve()**: ~50-200μs (load new program)
3. **pipe()** / **write() / read()**: for passing data
4. **waitpid()**: ~20-50μs (reap child)

**Total overhead per call: ~200-500μs** on modern systems.

For 10,000 operations in a loop:
- Bash built-in: ~0.02s (no forks)
- External command: ~2-5s (10,000 × 200-500μs)
- **Bash is 100-250x faster** for repeated operations

### Scaling with Data Size

| Operation | 1 line | 1000 lines | 1M lines |
|-----------|--------|------------|----------|
| Bash `while read; do ${var%% *}; done` | 1μs | 15ms | 15s |
| `awk '{print $1}'` | 500μs | 600μs | 0.3s |
| `cut -d' ' -f1` | 500μs | 500μs | 0.4s |

**For small data (1-1000 lines)**: Bash internals win.
**For large data (10K+ lines)**: External tools win.

## Under the Hood

### Why external tools are faster for large files

**Bash internal** approach reads a file line-by-line:
```bash
while IFS= read -r line; do
    first="${line%% *}"
done < file
```

This is slow because:
1. **Line-by-line I/O**: `read` is a system call per line (`read() syscall each iteration`)
2. **String handling**: `${%%}` allocates/frees memory per iteration
3. **Interpreted overhead**: Bash parses each statement, resolves variables, etc.

**Awk** approach:
```awk
awk '{print $1}' file
```

This is fast because:
1. **Buffered I/O**: Awk reads in large blocks (typically 4-64KB at a time), not line-by-line
2. **C-level execution**: Awk itself is written in C, compiled
3. **Optimized field splitting**: Uses pointer arithmetic, not substring allocations
4. **No fork overhead**: Awk processes the entire file in one process

### Strace differences

Bash loop on a 1000-line file:
```
read(0, buf, 4096) ...   # read a chunk
# for each line, bash calls read() again
read(0, buf, 4096) ...   # repeat
...
# But also: for each line, bash calls strlen, memcmp, etc. internally
```

Awk on same file:
```
read(0, buf, 8192) ...   # read 8KB at once
write(1, buf, ...)       # write results in large chunks
# Much fewer system calls overall
```

### Memory considerations

- **Bash loops**: Store the entire output (if captured with `$()`) or process line-by-line. For large files, line-by-line is memory-efficient but slow.
- **Awk/sed**: Process streams — no need to store everything in memory (usually). Memory-efficient and fast.
- **grep**: Stops at first match with `-q` or `-m1` — can be very fast for "does match exist" checks.

## Syntax & Key Differences

### Field extraction

**Bash**: `IFS` + `read -ra` + array access
```bash
while IFS=':' read -ra fields; do
    echo "${fields[0]}"  # first field
done < /etc/passwd
```

**Awk**:
```bash
awk -F: '{print $1}' /etc/passwd
```

**Cut**:
```bash
cut -d: -f1 /etc/passwd
```

### Regex matching

**Bash** (`[[ =~ ]]`):
```bash
if [[ "hello123" =~ ^[a-z]+[0-9]+$ ]]; then
    echo "matched"
fi
# Capture groups: BASH_REMATCH[1], BASH_REMATCH[2]
```

**grep -E**:
```bash
if echo "hello123" | grep -qE '^[a-z]+[0-9]+$'; then
    echo "matched"
fi
```

### Substring operations

| Operation | Bash | External |
|-----------|------|----------|
| Length | `${#str}` | `echo "$str" | wc -c` |
| Prefix | `${str#prefix}` | `echo "$str" | sed 's/^prefix//'` |
| Suffix | `${str%suffix}` | `echo "$str" | sed 's/suffix\$//'` |
| Replace | `${str//old/new}` | `echo "$str" | sed 's/old/new/g'` |
| Substring | `${str:1:3}` | `echo "$str" | cut -c2-4` |

## Core Examples (12 minimum)

### Example 1: Extract domain from email

```bash
# Bash internal (fast — no fork)
$ email="user@example.com"
$ domain="${email#*@}"
$ echo "$domain"
example.com

# External (slower — forks sed)
$ echo "$email" | sed 's/.*@//'
example.com

# Performance: internal ~10x faster
```

**Step-by-step**: `${email#*@}` removes the shortest prefix matching `*@` (up to and including the first `@`). Result: "example.com".

### Example 2: Regex matching with `[[ ]]`

```bash
$ phone="+1-555-123-4567"
$ if [[ $phone =~ ^\+[0-9]-[0-9]{3}-[0-9]{3}-[0-9]{4}$ ]]; then
>   echo "Valid phone format"
> fi
Valid phone format

# Capture groups with BASH_REMATCH
$ if [[ $phone =~ ^\+([0-9])-([0-9]{3})-(.+) ]]; then
>   echo "Country: ${BASH_REMATCH[1]}"
>   echo "Area: ${BASH_REMATCH[2]}"
>   echo "Rest: ${BASH_REMATCH[3]}"
> fi
Country: 1
Area: 555
Rest: 123-4567
```

**Step-by-step**: `[[ $var =~ regex ]]` uses ERE (Extended Regular Expressions) without forking. Capture groups are stored in `BASH_REMATCH` array (index 0 = full match, 1+ = groups).

### Example 3: grep -oP for extraction

```bash
$ log="2025-07-30 12:34:56 ERROR: Connection failed to 10.0.0.5"
$ echo "$log" | grep -oP '\d+\.\d+\.\d+\.\d+'
10.0.0.5
```

**Step-by-step**: `-o` prints only the matched part. `-P` enables Perl-compatible regex (PCRE). `\d+` matches digits. Escaped dots match literal dots.

**What if** `-P` isn't available (BSD/macOS)? Use `grep -E` with `[0-9]+` instead of `\d+`. Or use `grep -oE '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+'`.

### Example 4: awk for column extraction

```bash
$ df -h | awk 'NR>1 {print $5, $6}'
15% /
62% /home
8% /var
```

**Step-by-step**: `NR>1` skips the header. `{print $5, $6}` prints field 5 (usage percentage) and field 6 (mount point). Awk splits each line into `$1`, `$2`, etc. automatically on whitespace.

### Example 5: Performance comparison — loop vs tools

```bash
$ for i in {1..10000}; do echo "line $i data here"; done > /tmp/test.txt

# Bash internal (slow for large files)
$ time while read -r line; do first="${line%% *}"; done < /tmp/test.txt
real    0m0.45s

# awk (fast for large files)
$ time awk '{print $1}' /tmp/test.txt > /dev/null
real    0m0.02s
```

**Why the difference?** Awk processes in C with buffered I/O. Bash reads and processes each line with interpreter overhead.

### Example 6: Using `read -r` for safe line reading

```bash
$ cat > /tmp/test.txt <<< $'line1\nline2\nline with\\backslash'

# WRONG: for loop
$ for line in $(cat /tmp/test.txt); do echo "$line"; done
line1
line2
line
with\backslash  # word-split on spaces!

# CORRECT: while read
$ while IFS= read -r line; do echo "$line"; done < /tmp/test.txt
line1
line2
line with\backslash
```

**The `IFS= read -r` incantation**:
- `IFS=` prevents trimming leading/trailing whitespace
- `read -r` prevents backslash interpretation
- This is the ONLY safe way to read lines in bash

### Example 7: grep -q for existence check

```bash
# Fast: grep stops at first match
$ if grep -q "ERROR" /var/log/syslog; then
>   echo "Errors found"
> fi

# Even with a million-line file, grep -q stops after finding the first "ERROR"
```

### Example 8: tr for single-character translation

```bash
# Uppercase to lowercase
$ echo "HELLO" | tr 'A-Z' 'a-z'
hello

# ROT13
$ echo "hello" | tr 'a-zA-Z' 'n-za-mN-ZA-M'
uryyb

# Delete characters
$ echo "Hello 123 World!" | tr -d '0-9'
Hello  World!
```

### Example 9: cut for simple field extraction

```bash
$ echo "user:pass:uid:gid:desc:home:shell" | cut -d: -f1,3,7
user:uid:shell

# Character positions
$ echo "abcdef" | cut -c2-4
bcd
```

### Example 10: sed for streaming edits

```bash
# Replace all
$ echo "foo foo foo" | sed 's/foo/bar/g'
bar bar bar

# Delete matching lines
$ sed '/^#/d' config.conf    # remove comments

# Print specific lines
$ sed -n '10,20p' file.txt   # lines 10-20
```

### Example 11: Combined tools in a pipeline

```bash
# Find top 10 most common IPs in access log
$ awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10
  452 192.168.1.1
  312 10.0.0.1
  198 192.168.1.5

# Step-by-step:
# awk '{print $1}' → extract IPs
# sort → group identical IPs together
# uniq -c → count occurrences
# sort -rn → sort by count descending
# head -10 → top 10
```

### Example 12: Bash internal vs external for large files

```bash
$ cat /tmp/large.csv
col1,col2,col3,...
...
# 1 million lines

# BASH (slow — 10+ seconds)
$ while IFS=',' read -r -a fields; do
>   echo "${fields[2]}"
> done < /tmp/large.csv

# AWK (fast — < 1 second)
$ awk -F',' '{print $3}' /tmp/large.csv

# CUT (fast — < 1 second)
$ cut -d',' -f3 /tmp/large.csv
```

### Example 13: When to hybrid — bash + grep

```bash
# Extract IPv4 addresses from output
$ ifconfig | grep -oP 'inet \K[\d.]+'
192.168.1.5
127.0.0.1

# The \K in PCRE means "keep out of match" — matches "inet " but only outputs the IP
```

### Example 14: eval is dangerous — but parameter expansion is safe

```bash
# BAD: eval re-evaluates everything (command injection risk)
eval "result=\${$var_name}"   # dangerous if var_name contains "; rm -rf /"

# GOOD: indirect expansion (safe)
result="${!var_name}"         # bash handles this without eval
```

## Real-World Use Cases

### FOR the OS

- **Log analysis**: `awk '{print $1, $9}' access.log | sort | uniq -c` for IP+status summary
- **Config validation**: `grep -vE '^\s*(#|$)' config.conf` to strip comments
- **Process monitoring**: `ps aux | awk '$3 > 50 {print $2}'` — find CPU-hungry PIDs
- **Disk usage**: `df -h | awk 'NR>1 {print $5, $6}' | sort -rn` — full filesystems

### WITH the OS

- **sed in-place editing**: `sed -i 's/old/new/g' file.txt` — edit files without temp files
- **tr for pipeline normalization**: `tr '[:upper:]' '[:lower:]'` for case-insensitive pipelines
- **cut for restricted output**: `cut -c1-80` — truncate lines
- **grep context lines**: `grep -C 3 "ERROR" logfile` — show error with context

### AGAINST the OS (troubleshooting)

- **Extract timestamps**: `awk '{print $1, $2}' /var/log/syslog | tail -20` — last 20 log entries
- **Find most frequent systemd unit failures**: `journalctl -u nginx | awk '{print $5}' | sort | uniq -c | sort -rn`
- **Parse /proc/meminfo**: `grep -E '^(MemTotal|MemFree|MemAvailable)' /proc/meminfo | awk '{sum+=$2} END {print sum/1024 " MB"}'`

### FOR DEFENSE

- **Input sanitization**: Remove dangerous chars before processing: `safe=$(echo "$input" | tr -dc '[:print:]')`
- **Log monitoring**: `tail -f /var/log/auth.log | grep -E 'Failed password|Invalid user'` — real-time auth monitoring
- **IP whitelisting**: `grep -Ff whitelist.txt access.log` — extract lines matching whitelisted IPs
- **Size limits**: `find / -type f | xargs ls -la | awk '$5 > 1000000 {print $NF}'` — files > 1MB

## Memory Aids

- **"Bash for brevity, awk for bulk"**: Small strings → bash internals. Large files → awk.
- **"If you fork in a loop, you'll get the boot"**: Avoid forking external commands in loops.
- **"`IFS= read -r` — the holy trinity"**: `IFS=` (trim no), `read` (read a line), `-r` (raw mode).
- **"Awk is a programming language, not a command"**: It has variables, loops, arrays, and more.
- **"grep -q stops at 'quite enough'"**: It exits at the first match — very fast for existence checks.
- **"`${var##pattern}` doesn't need sed"**: 90% of sed usage can be replaced with parameter expansion.

## Trap Vault (12 traps)

### Trap 1: Calling external tools in a loop is slow

```bash
# BAD: forks grep 10,000 times
for file in *.txt; do
    grep "TODO" "$file"   # each grep forks!
done

# GOOD: let grep handle the loop
grep -l "TODO" *.txt

# OR: xargs
find . -name "*.txt" -print0 | xargs -0 grep -l "TODO"
```

### Trap 2: `grep -P` is not POSIX

```bash
# BAD: -P only works with GNU grep
grep -P '\d+\.\d+\.\d+\.\d+' file

# FIX on macOS/BSD: use -E
grep -E '[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+' file
```

### Trap 3: `read` without `-r` interprets backslashes

```bash
# BAD: read -r interprets \n, \t, etc.
echo "hello\nworld" | while read line; do echo "$line"; done
# Prints "hellonworld" (treats \n as literal n!)

# FIX: always use -r
echo "hello\nworld" | while read -r line; do echo "$line"; done
# Prints "hello\nworld" literally
```

### Trap 4: `for word in $(cat file)` splits on whitespace

```bash
# BAD: splits on every space, newline, tab
for ip in $(cat ip_list.txt); do ping -c 1 "$ip"; done
# If file contains "192.168.1.1 10.0.0.1", it works (no spaces).
# But "192.168.1.1\n10.0.0.1" → word-split into two? Actually that works.
# The real problem: spaces in data break it.

# FIX: use while read
while IFS= read -r ip; do ping -c 1 "$ip"; done < ip_list.txt
```

### Trap 5: `sed` regex syntax differs from bash `=~`

```bash
# bash =~ uses ERE: \d doesn't work, use [0-9]
# sed uses BRE by default: + needs \+, ? needs \?, | needs \|
# sed -E uses ERE (like bash =~)

# bash =~:
[[ "123" =~ ^[0-9]+$ ]]     # OK

# sed BRE:
echo "123" | sed 's/^[0-9]\+$/MATCH/'  # needs \+ for "one or more"

# sed ERE:
echo "123" | sed -E 's/^[0-9]+$/MATCH/'  # + works
```

### Trap 6: Word splitting on unquoted variable expansions

```bash
$ file="my document.txt"
$ cat $file           # tries to cat "my" and "document.txt"
$ cat "$file"         # proper — quotes preserve spaces
```

### Trap 7: awk field numbering vs bash array indexing

```bash
# awk: fields are 1-indexed
awk '{print $1, $2, $3}'   # first, second, third field

# bash: arrays are 0-indexed
arr=(one two three)
echo "${arr[0]}"             # first element
```

### Trap 8: `cut` splits on single character delimiter only

```bash
# cut doesn't handle multi-character delimiters
echo "a||b||c" | cut -d'||' -f2   # ERROR: delimiter must be one char
# FIX: Use awk
echo "a||b||c" | awk -F'||' '{print $2}'
```

### Trap 9: `echo` vs `printf` for strings with special chars

```bash
# BAD: echo may interpret escape sequences
echo "hello\nworld"   # some shells print "hello\nworld", some "hello<newline>world"

# GOOD: printf is consistent
printf '%s\n' "hello\nworld"   # always prints "hello\nworld" literally
```

### Trap 10: `tr` can't handle multi-character replacements

```bash
# BAD: expects tr to replace "ab" with "cd"
echo "ab" | tr 'ab' 'cd'   # "cd" — works by coincidence (a→c, b→d)
echo "aba" | tr 'ab' 'cd'  # "cdc" — not "cdcd"!
# tr is a character-by-character translator, not string replacement
# FIX: sed 's/ab/cd/g'
```

### Trap 11: `head` and `tail` in pipes can cause SIGPIPE

```bash
# BAD: if head exits early, the upstream command gets SIGPIPE
find / -type f 2>/dev/null | head -5
# find keeps running (usually) but might get SIGPIPE, which is fine but noisy

# GREP with -m is cleaner:
grep -m 5 "error" hugefile.log   # stops after 5 matches
```

### Trap 12: Mixing `-n` and `-i` in sed can cause confusion

```bash
# BAD: -n suppresses output, -i edits in-place — they conflict
sed -ni 's/foo/bar/p' file   # probably not what you want

# Usually you want one or the other
sed -i 's/foo/bar/g' file     # in-place replacement
sed -n 's/foo/bar/p' file     # print only changed lines
```

## See It In The Wild

- **Cron jobs**: `awk '{print $6}' /var/log/apt/history.log | grep install` — what was installed?
- **Docker**: `docker ps --format '{{.Names}}' | xargs -I {} docker logs {} | grep ERROR`
- **Git hooks**: `git diff --cached --name-only | grep '\.py$' | xargs pylint`
- **Journald**: `journalctl --since "1 hour ago" | grep -oP 'Failed password for \K\S+' | sort -u`

### Exploration

1. Time comparison: `time awk '{print $1}' /var/log/syslog > /dev/null` vs `time while read -r line; do echo "${line%% *}"; done < /var/log/syslog > /dev/null`
2. `grep -c` vs `wc -l | grep` — which is faster for counting specific patterns?
3. Write a bash-only version of `wc -l` — compare its speed to the real `wc -l`
4. How many forks does `echo "$var" | tr 'a-z' 'A-Z'` involve?

## Check Your Understanding (7 questions)

1. **Why is calling `echo $line | cut -d' ' -f1` inside a loop slow?**
2. **When would you choose `awk` over bash parameter expansion?**
3. **What does `grep -P` offer that `grep -E` doesn't?**
4. **Why is `while IFS= read -r line` the correct way to read lines?**
5. **What's the fastest way to extract the Nth field from every line of a 1GB file?**
6. **How does `${BASH_REMATCH[1]}` work with `[[ $var =~ regex ]]`?**
7. **Why does `for word in $(cat file)` break on most text files?**

## Supplementary Deep Dive: Advanced String Processing Patterns

### Building a CSV parser with bash internals

```bash
$ parse_csv() {
>   local line
>   while IFS= read -r line; do
>     # Handle quoted fields: "hello, world","foo"
>     local fields=()
>     local rest="$line"
>     while [[ -n "$rest" ]]; do
>       if [[ "${rest:0:1}" == '"' ]]; then
>         # Quoted field — find closing quote
>         rest="${rest:1}"
>         local field="${rest%%\"*}"
>         fields+=("$field")
>         rest="${rest#*\"}"
>         rest="${rest#,}"  # remove comma separator
>       else
>         # Unquoted field
>         local field="${rest%%,*}"
>         [[ "$field" == "$rest" ]] && rest="" || rest="${rest#*,}"
>         fields+=("$field")
>       fi
>     done
>     # Now fields array has the CSV columns
>     echo "Row: ${fields[*]}"
>   done
> }
```

### Using printf for complex formatting

```bash
$ # Column alignment
$ printf "%-20s %8s\n" "Name" "Score"
$ printf "%-20s %8d\n" "Alice" 95
$ printf "%-20s %8d\n" "Bob" 87
Name                    Score
Alice                    95
Bob                      87

$ # Dynamic width
$ width=30
$ printf "%*s\n" $width "right-aligned"
$ printf "%-*s\n" $width "left-aligned"
```

### String length and byte vs character count

```bash
$ # In C locale, ${#var} counts bytes
$ LC_ALL=C; str="café"; echo "${#str}"
5   # 5 bytes (é is 2 bytes in UTF-8)

$ # In UTF-8 locale, counts characters
$ LC_ALL=en_US.UTF-8; str="café"; echo "${#str}"
4   # 4 characters
```

### Using eval carefully (and avoiding it)

```bash
$ # BAD — eval is dangerous
$ cmd="echo $USER"
$ eval "$cmd"   # users can inject commands via $USER

$ # GOOD — use arrays or expansions
$ cmd=(echo "$USER")
$ "${cmd[@]}"
```

### Building a progress indicator with tr

```bash
$ # Show a spinning bar
$ spinner() {
>   local chars='-\|/'
>   local i=0
>   while true; do
>     printf "\r%c" "${chars:$((i++ % 4)):1}"
>     sleep 0.1
>   done
> }

$ # Use it
$ spinner &
$ pid=$!
$ sleep 5
$ kill $pid
```

### Using fold and fmt for text wrapping

```bash
$ # Wrap long lines at 40 characters
$ long="This is a very long line that needs to be wrapped for display purposes"
$ echo "$long" | fold -w 40
This is a very long line that needs
to be wrapped for display purposes

$ # Smart wrapping (doesn't break words)
$ echo "$long" | fmt -w 40
This is a very long line that
needs to be wrapped for display
purposes
```

### Extracting and validating IP addresses

```bash
$ # Pure bash IP validation
$ validate_ip() {
>   local ip="$1"
>   local IFS='.'
>   local parts=($ip)
>   (( ${#parts[@]} != 4 )) && return 1
>   for part in "${parts[@]}"; do
>     [[ "$part" =~ ^[0-9]+$ ]] || return 1
>     (( part >= 0 && part <= 255 )) || return 1
>   done
>   return 0
> }
$ validate_ip "192.168.1.1" && echo "valid" || echo "invalid"
valid
$ validate_ip "256.1.1.1" && echo "valid" || echo "invalid"
invalid
```

### Parsing JSON without jq (limited)

```bash
$ # Extract simple values with grep/sed
$ json='{"name":"Alice","age":30,"city":"NYC"}'
$ echo "$json" | grep -oP '"name":"\K[^"]+'
Alice
$ echo "$json" | grep -oP '"age":\K[0-9]+'
30
$ # For complex JSON, use jq
$ echo "$json" | jq -r '.name'
Alice
```

### Using mlr (Miller) for CSV/JSON processing

```bash
$ # Miller is a powerful tool for structured data
$ mlr --csv cut -f name,email data.csv
$ mlr --json put '$age = $age + 1' data.json
```

### String hashing with md5sum/sha256sum

```bash
$ echo -n "hello" | md5sum | cut -d' ' -f1
5d41402abc4b2a76b9719d911017c592
$ echo -n "password" | sha256sum | cut -d' ' -f1
5e884898da28047151d0e56f8dc6292773603d0d6aabbdd62a11ef721d1542d8
```

### Using sort -t for field-based sorting

```bash
$ cat data.txt
Alice:30:NYC
Bob:25:London
Charlie:35:Paris
$ sort -t: -k2 -n data.txt   # sort by age (numeric)
Bob:25:London
Alice:30:NYC
Charlie:35:Paris
$ sort -t: -k3 data.txt       # sort by city (alphabetical)
Bob:25:London
Alice:30:NYC
Charlie:35:Paris
```

### The comm command for file comparison

```bash
$ sort file1.txt > f1.sorted
$ sort file2.txt > f2.sorted
$ comm -12 f1.sorted f2.sorted   # lines in both files (intersection)
$ comm -23 f1.sorted f2.sorted   # lines only in file1
$ comm -13 f1.sorted f2.sorted   # lines only in file2
```
