# Task 16: Capstone — 5 Real-World One-liner Puzzles

Solve each puzzle using only bash commands and pipelines — no scripts allowed, one line only. These are real problems that sysadmins face daily.

## Steps

### Puzzle 1: Find Largest Files
Find the 10 largest files (not directories) under `/usr/share/doc` and show their sizes in human-readable format.

**Constraints:** Only one line, no scripts, no `for` loops spanning multiple lines (use `find -exec` or `xargs`).

```bash
$ find /usr/share/doc -type f -exec du -h {} + | sort -rh | head -10
4.2M    /usr/share/doc/bash/bash.pdf
2.1M    /usr/share/doc/perl/perl.pdf
1.8M    /usr/share/doc/git/git.html
...
```

**Approach 1:** `find -exec du -h {} + | sort -rh | head -10` — the standard approach
**Approach 2:** `find -type f -printf '%s %p\n' | sort -rn | head -10 | numfmt --to=iec` — using `-printf` and `numfmt`
**Approach 3:** `du -ah /usr/share/doc | sort -rh | grep -v '/$' | head -10` — using `du` with exclude dirs

**Bonus:** Add a column showing the file's modification date:
```bash
$ find /usr/share/doc -type f -printf '%s %t %p\n' | sort -rn | head -5 | awk '{print $2, $3, $1, $4}'
```

<details><summary>Hint: find + du + sort pipeline</summary>
The key is `find ... -type f -exec du -h {} +` which runs `du` in batch mode (`+` instead of `\;`). This is much faster than spawning `du` for each file. Then `sort -rh` to sort human-readable sizes. `head -10` to take the top 10.
</details>

### Puzzle 2: Extension Census
Count all unique file extensions under `/usr/bin` and show the top 5 most common. Files without an extension count as "no extension."

```bash
$ ls /usr/bin | rev | cut -d. -f1 | rev | sort | uniq -c | sort -rn | head -5
   2012
    342 py
    120 pl
     85 sh
     32 rb
```

The blank line at top represents files without an extension.

**Approach 1:** `rev | cut -d. -f1 | rev` — the classic trick
**Approach 2:** `awk -F. '{print $NF}'` — simpler if you handle files without dots
**Approach 3:** `sed 's/.*\.//' | grep .` — but this drops no-extension files

**Bonus:** Show percentage of total for each extension:
```bash
$ total=$(ls /usr/bin | wc -l)
$ ls /usr/bin | rev | cut -d. -f1 | rev | sort | uniq -c | sort -rn | head -5 \
  | awk -v total=$total '{printf "%5d %6.2f%% %s\n", $1, ($1/total)*100, $2}'
```

<details><summary>Hint: The rev/cut/rev trick explained</summary>
`ls /usr/bin` outputs filenames like `python3.11`, `bash`, `ls.coreutils`.
`rev` reverses them: `11.3.nohtyp`, `hsab`, `slirotuc.sl`.
`cut -d. -f1` takes the first field: `11`, `hsab`, `slirotuc`.
`rev` reverses back: `11`, `bash`, `coturils` — wait, that's not right.

Actually let me think again. For `bash` (no dot):
- `rev` -> `hsab`
- `cut -d. -f1` -> `hsab` (entire string since no dot)
- `rev` -> `bash`
So no-extension files get their FULL NAME, not empty.

For `python3.11`:
- `rev` -> `11.3.nohtyp`
- `cut -d. -f1` -> `11`
- `rev` -> `11`
Wrong! We wanted `.11` or `11` (the extension). So this WORKS for getting the extension.

For `ls.coreutils`:
- `rev` -> `slirotuc.sl`
- `cut -d. -f1` -> `slirotuc`
- `rev` -> `coturils`
Hmm, that gives us the filename reversed, not the extension. Wait, this approach gets the FIRST field after reversal, which is the LAST field before reversal. So it gets the extension!

For `archive.tar.gz`:
- `rev` -> `zg.rat.evihcra`
- `cut -d. -f1` -> `zg` (only `gz`)
- `rev` -> `gz`
This only captures the LAST extension! `.tar.gz` gives just `gz`. This is the limitation.
</details>

### Puzzle 3: Extract All IP Addresses
Extract all unique IPv4 addresses from `/var/log/auth.log` (or `/var/log/syslog`). Sort them numerically.

```bash
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' /var/log/auth.log | sort -u -t. -k1,1n -k2,2n -k3,3n -k4,4n
10.0.0.1
10.0.0.2
192.168.1.100
```

**Approach 1:** `grep -oE` with `\b` word boundaries, `sort -u` with version sort or numeric sort per octet
**Approach 2:** `grep -oP '\d+\.\d+\.\d+\.\d+'` (PCRE) if available
**Approach 3:** `sed -n 's/.*\([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\).*/\1/p' | sort -u`

**Bonus:** Count occurrences:
```bash
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' /var/log/auth.log | sort | uniq -c | sort -rn | head -10
```

<details><summary>Hint: Numeric IP sort</summary>
`sort -u -t. -k1,1n -k2,2n -k3,3n -k4,4n` sorts each octet numerically. `-t.` sets the delimiter to dot. Each `-kN,Nn` sorts that field numerically. Without this, `10.0.0.2` would sort after `10.0.0.10` lexicographically.
</details>

### Puzzle 4: Summarize Log by Hour
Count how many log entries occurred per hour in `/var/log/syslog`. The log format has a timestamp like `Jul 31 01:00:01` — extract the hour.

```bash
$ grep -oP ' \d{2}:' /var/log/syslog | cut -d: -f1 | sort | uniq -c | sort -rn
    452  01
    389  02
    312  00
    ...
```

**Approach 1:** `grep -oP` for ` HH:` pattern, `cut -d: -f1` extracts the hour number
**Approach 2:** `awk '{print $2}' | cut -d: -f1` — awk splits on whitespace, field 2 is time
**Approach 3:** `sed -n 's/^... \([0-9][0-9]\):.*/\1/p' | sort | uniq -c | sort -rn`

**Bonus:** Show a bar chart:
```bash
$ grep -oP ' \d{2}:' /var/log/syslog | cut -d: -f1 | sort | uniq -c | sort -rn | \
  while read count hour; do
    printf "%02d |%s %d\n" "$hour" "$(printf '#%.0s' $(seq 1 $((count/10))))" "$count"
  done
```

<details><summary>Hint: Timestamp format in syslog</summary>
Syslog format: `Jul 31 01:23:45 hostname service[pid]: message`
The hour is the first two digits of the time field. `grep -oP ' \d{2}:'` matches ` HH:` patterns. The space ensures we match time, not random numbers. `cut -d: -f1` extracts just ` HH` (with leading space), which `sort`/`uniq` handles fine.
</details>

### Puzzle 5: Batch Touch + Permissions
Create files `test_{A..Z}.sh`, make them all executable, then show the ones that are actually executable. All in one line.

```bash
$ for f in {A..Z}; do touch "test_${f}.sh"; done && chmod +x test_*.sh && ls -lh test_*.sh
```

**Approach 1:** `for f in {A..Z}; do touch "test_${f}.sh"; done && chmod +x test_*.sh && ls test_*.sh
**Approach 2:** `touch test_{A..Z}.sh && chmod +x test_*.sh && ls -l test_*.sh`
**Approach 3:** `printf 'test_%s.sh\0' {A..Z} | xargs -0 touch && chmod +x test_*.sh && ls test_*.sh`

**Bonus:** Clean up after: `rm test_{A..Z}.sh` — but ONLY if they exist:
```bash
$ ls test_?.sh &>/dev/null && rm test_{A..Z}.sh && echo "Cleaned up" || echo "No files to clean"
```

<details><summary>Hint: Brace expansion in one-liners</summary>
`{A..Z}` expands to all uppercase letters. You can nest: `{A..C}{1..3}` expands to `A1 A2 A3 B1 B2 B3 C1 C2 C3`. Brace expansion happens BEFORE variable expansion, so you can't use `$var` inside braces. Use C-style for loops instead.
</details>

## Expected Output Summary

```bash
=== Puzzle 1: Largest files ===
$ find /usr/share/doc -type f -exec du -h {} + | sort -rh | head -10
4.2M    /usr/share/doc/bash/bash.pdf
2.1M    /usr/share/doc/perl/perl.pdf
1.8M    /usr/share/doc/git/git.html
...

=== Puzzle 2: Extension census ===
$ ls /usr/bin | rev | cut -d. -f1 | rev | sort | uniq -c | sort -rn | head -5
   2012
    342 py
    120 pl
     85 sh
     32 rb

=== Puzzle 3: IP addresses ===
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' /var/log/auth.log | sort -u -t. -k1,1n -k2,2n -k3,3n -k4,4n
10.0.0.1
10.0.0.2
192.168.1.100

=== Puzzle 4: Log by hour ===
$ grep -oP ' \d{2}:' /var/log/syslog | cut -d: -f1 | sort | uniq -c | sort -rn
    452  01
    389  02
    312  00
    ...

=== Puzzle 5: Batch touch + permissions ===
$ for f in {A..Z}; do touch "test_${f}.sh"; done && chmod +x test_*.sh && ls test_*.sh
test_A.sh  test_B.sh  test_C.sh  ...  test_Z.sh
$ ls -l test_A.sh
-rwxr-xr-x 1 phd phd 0 Jul 31 01:20 test_A.sh
```

## Full Integration: Capstone Challenge

Combine everything! Write a single one-liner that:
1. Creates 10 test files with different extensions
2. Renames all `.sh` files to `.bash`
3. Counts how many files have each extension
4. Removes all remaining test files
5. Reports success

```bash
# One-line to rule them all:
$ for ext in sh py txt md; do for i in {1..3}; do touch "test_${i}.${ext}"; done; done && \
  for f in test_*.sh; do mv "$f" "${f%.sh}.bash"; done && \
  ls -1 | rev | cut -d. -f1 | rev | sort | uniq -c | sort -rn && \
  rm -f test_* && \
  echo "Capstone complete! $(ls test_* 2>/dev/null | wc -l) files remaining."
```

## Self-Check Questions

1. What is the difference between `find -exec` and piping to `xargs`?

2. Why does `ls /usr/bin | rev | cut -d. -f1 | rev` work for extensions? What is the limitation?

3. How would you safely preview a destructive one-liner before running it?

4. What does `-oE` in `grep -oE` do?

5. How would you add error handling to a one-liner that deletes files?

6. What is `PIPESTATUS` and when would you use it?

7. Why does `sudo !!` not work with redirections like `>` or `>>`?
