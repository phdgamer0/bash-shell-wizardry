# Lesson 5: Counting, Sorting & Extracting

## History & Origins

`wc` (word count) is one of the oldest Unix commands, appearing in Version 1 Unix (1971). It counted lines, words, and characters — a "word" being any whitespace-delimited string. The name is simply "word count," and its original purpose was to count words in documents. The `-l` (lines) flag was added later and is now the most common usage.

`sort` also dates to Version 1 Unix. It was written by Ken Thompson and initially could only sort input line-by-line using a merge sort algorithm. The `-n` (numeric sort) was added in Version 7 Unix (1979). The `-k` (key-based sort) came with POSIX. Modern GNU `sort` uses a sophisticated merge sort that handles huge files by writing temporary files to disk.

`uniq` appeared in Version 3 Unix (1973). It was designed to work on pre-sorted data. The requirement for sorted input is intentional — it makes the algorithm O(n) instead of O(n log n) and allows streaming with minimal memory (just keep the previous line). The `-c` (count) flag is the most popular feature.

`cut` was introduced in System III (1982). Before that, extracting columns from text required `awk` or `sed`. `cut` was purpose-built for simple column extraction. It handles delimited fields (`-d` and `-f`) and character positions (`-c`).

`paste` appeared in 4.2BSD (1983), designed to merge files line-by-line. It's the column-wise counterpart to `cat` (which merges vertically).

`join` was part of the original Unix toolkit but was rarely used outside of database-like operations. It implements a relational "join" operation on sorted text files.

`tr` (translate) comes from System III/V. It's the simplest character manipulation tool — replacing, deleting, or squeezing characters. It does NOT handle multi-character strings; for that, use `sed`.

**Fun anecdote:** Bell Labs had a legendary "wc" bet: someone bet that the Unix source code had fewer than 10,000 lines. `wc -l` proved them wrong. Today, the Linux kernel has over 30 million lines. Also, Brian Kernighan (of K&R C fame) wrote a famous one-liner: `sort | uniq -c | sort -rn` which is still the canonical frequency table pipeline.

## Syntax Reference

### `wc` — Word Count

```bash
wc [options] [file...]
wc -l file      # Lines (most common)
wc -w file      # Words (whitespace-delimited)
wc -c file      # Bytes (NOT characters — use -m for characters)
wc -m file      # Characters (multibyte-aware)
wc -L file      # Longest line length (GNU extension)
```

Without flags, `wc` outputs all three: lines, words, bytes.

Reading from stdin: `wc -l < file` or `cmd | wc -l`.

### `sort` — Sort Lines

```bash
sort [options] [file...]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-n` | `--numeric-sort` | Sort numerically (7 < 70 < 100) |
| `-r` | `--reverse` | Reverse sort order |
| `-k` | `--key=KEYDEF` | Sort by a specific field/key |
| `-t` | `--field-separator=SEP` | Set field separator |
| `-u` | `--unique` | Output only first of equal lines (like `sort \| uniq`) |
| `-f` | `--ignore-case` | Case-insensitive sort |
| `-h` | `--human-numeric-sort` | Sort human-readable numbers (2K, 3M, 1G) |
| `-b` | `--ignore-leading-blanks` | Ignore leading whitespace |
| `-M` | `--month-sort` | Sort by month name (JAN < FEB < ... < DEC) |
| `-V` | `--version-sort` | Natural version sort (v1.2 < v1.10) |
| `-c` | `--check` | Check if input is sorted; don't sort |
| `-C` | `--check=quiet` | Like -c but silent |
| `-o` | `--output=FILE` | Write result to FILE (can be same as input) |
| `-s` | `--stable` | Preserve original order of equal lines |
| `-T` | `--temporary-directory=DIR` | Use DIR for temporary files |
| `-S` | `--buffer-size=SIZE` | Use SIZE for main memory |
| `--parallel=N` | | Use N parallel threads |

**Key syntax:** `-k start[,end]` where:
- `start = field[.char]` e.g., `-k2` = field 2, `-k2.3` = field 2, character 3
- `end = field[.char]` e.g., `-k2,3` = fields 2 through 3
- `-k2,2` = field 2 only
- Options can be attached: `-k2,2n` = numeric sort on field 2

### `uniq` — Unique Lines

```bash
uniq [options] [input [output]]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-c` | `--count` | Prefix lines by count of occurrences |
| `-d` | `--repeated` | Only print duplicate lines (one per group) |
| `-D` | `--all-repeated` | Print ALL duplicate lines |
| `-u` | `--unique` | Only print unique lines (not duplicated) |
| `-i` | `--ignore-case` | Case-insensitive comparison |
| `-f N` | `--skip-fields=N` | Skip first N fields when comparing |
| `-s N` | `--skip-chars=N` | Skip first N characters when comparing |
| `-w N` | `--check-chars=N` | Compare no more than N characters per line |

**IMPORTANT:** `uniq` only removes ADJACENT duplicates. Input must be sorted first.

### `cut` — Cut Out Fields

```bash
cut [options] [file...]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-f` | `--fields=LIST` | Select fields (1-based, comma/range separated) |
| `-d` | `--delimiter=DELIM` | Field delimiter (default TAB) |
| `-c` | `--characters=LIST` | Select character positions |
| `-b` | `--bytes=LIST` | Select byte positions |
| `-s` | `--only-delimited` | Don't print lines without delimiter |
| `--complement` | | Invert selection (all except specified) |
| `--output-delimiter=SEP` | | Output delimiter (default = input delimiter) |

**Field LIST syntax:** `1,3,5` or `1-5` or `1-` or `-5` or `1,3-5,7-`

### `paste` — Merge Lines

```bash
paste [options] file1 file2...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-d` | `--delimiters=LIST` | Use delimiters from LIST (default TAB) |
| `-s` | `--serial` | Paste serially (one file at a time) |

### `join` — Join Lines on Common Field

```bash
join [options] file1 file2
```

| Flag | Long | What it does |
|------|------|-------------|
| `-j` | `--join-field` | Field to join on (default 1) |
| `-1` | | Field in file1 to join on |
| `-2` | | Field in file2 to join on |
| `-t` | `--field-separator=SEP` | Field separator |
| `-o` | `--output=FORMAT` | Output format |
| `-a` | | Include unpairable lines from file N |
| `-v` | | Only show unpairable lines |

### `tr` — Translate Characters

```bash
tr [options] set1 [set2]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-d` | `--delete` | Delete characters in set1 |
| `-s` | `--squeeze-repeats` | Squeeze repeated characters into one |
| `-c` | `--complement` | Complement set1 (everything except) |
| `-t` | `--truncate-set1` | Truncate set1 to length of set2 |

**Set syntax:**
- `'a-z'` — lowercase range
- `'A-Z'` — uppercase range
- `'0-9'` — digits
- `'a-z0-9'` — combined ranges
- `'\n'` — newline, `'\t'` — tab (in some shells, use `$'\n'`)
- `'[:lower:]'`, `'[:upper:]'`, `'[:digit:]'`, `'[:alpha:]'`, `'[:alnum:]'` — character classes

## Under the Hood

### How `sort` works

`sort` uses a **merge sort** algorithm. For files that fit in memory:
1. Read all lines into an array.
2. Sort using `qsort()` (C library quicksort).
3. Write sorted output.

For files larger than available memory:
1. Read as many lines as fit in memory (controlled by `-S`).
2. Sort them, write to a temporary file (`/tmp/` or `-T` directory).
3. Repeat (step 1-2) until all input is in sorted chunks.
4. Merge all sorted chunks using a k-way merge (similar to what `git merge` does).
5. Write final output.

**Performance implications:**
- Sorting a 10MB file is instant (in memory).
- Sorting a 10GB file uses temporary files and is I/O-bound.
- `sort -S 50%` uses 50% of RAM for the sort buffer before writing temp files.
- `sort --parallel=4` uses multiple cores for the merge.

### How `uniq` works

`uniq` is simpler:
1. Read first line, store as `current`, set `count = 1`.
2. Read next line. If it matches `current`: `count++`.
3. If it doesn't match: output `current` (with `-c` count), set `current = new line`, `count = 1`.
4. At EOF, output last `current`.

This O(n) streaming algorithm uses almost no memory (just one line buffer), but REQUIRES sorted input. Without sorting, duplicates scattered across the file won't be detected.

### How `cut` works

1. Read a line.
2. If `-f`: split line by delimiter (single character, default TAB). Extract specified fields.
3. If `-c`: treat each byte (or character with `-c` in some implementations) as a position.
4. Rejoin extracted parts with output delimiter.
5. Write to stdout.

`cut` reads character-by-character or uses field-separator splitting (like `strtok()`). It's very fast for simple column extraction.

### System calls

```bash
# sort on a 100MB file
strace -e trace=openat,read,write sort bigfile 2>&1 | tail -20
```

You'll see `sort` opening the input, reading in chunks, writing to temporary files in `/tmp`, then merging.

```bash
# wc -l on a file
strace -e trace=read wc -l /etc/passwd 2>&1
```

`wc` just reads through the file counting newlines — no complex processing.

### Memory model

- `wc -l` uses minimal memory (one buffer, ~8KB).
- `uniq` uses minimal memory (two line buffers).
- `cut` uses minimal memory (one line buffer).
- `sort` can use gigabytes (entire file or large chunks in RAM).
- `tr` uses a translation table in memory (256 bytes for byte-based, larger for Unicode).
- `join` holds one file's key column in memory.

## Core Examples

### Example 1: Counting everything in /etc/passwd

```bash
$ wc /etc/passwd
  45   75 2678 /etc/passwd
# 45 lines, 75 words, 2678 bytes
```

**Step by step:**
1. `wc` opens `/etc/passwd`.
2. Reads the entire file into a buffer.
3. Scans buffer: counts newlines (45), whitespace-delimited words (75), bytes (2678).
4. Prints counts and filename.
5. Note: 2678 bytes, but if the file had multi-byte characters, `-m` would give a different count.

**What if:**
```bash
$ wc -l /etc/passwd             # Just lines
45
$ wc -c /etc/passwd             # Just bytes
2678
$ echo -e "hello" | wc -c       # 6 bytes (hello + newline)
6
$ echo -n "hello" | wc -c       # 5 bytes (no newline)
5
$ echo -e "\x61\x62\x63" | wc -c # 4 bytes (abc + newline)
4
```

### Example 2: Cut fields from CSV

```bash
$ echo "name,age,score" | cut -d, -f2
age
$ cut -d: -f1,7 /etc/passwd | head -3
root:/bin/bash
daemon:/usr/sbin/nologin
bin:/usr/sbin/nologin
```

**Step by step:**
1. `cut -d, -f2` reads the line "name,age,score".
2. Splits on `,`: ["name", "age", "score"].
3. Extracts field 2: "age".
4. For `/etc/passwd`: splits on `:`, extracts fields 1 and 7 (username and shell).
5. Output is tab-separated by default (between the two fields).

**What if:**
```bash
$ cut -d: -f1-3 /etc/passwd | head -1   # Fields 1,2,3
root:x:0
$ cut -d: -f3- /etc/passwd | head -1    # Field 3 through end
0:0:root:/root:/bin/bash
$ cut -d: -f1 --complement /etc/passwd | head -1  # All except field 1
x:0:0:root:/root:/bin/bash
```

### Example 3: Numeric sort vs lexical sort

```bash
$ printf "10\n2\n111\n3\n" | sort
10
111
2
3
$ printf "10\n2\n111\n3\n" | sort -n
2
3
10
111
```

**Step by step:**
1. Default sort: compares strings character-by-character. `"10"` < `"111"` (because `'0'` < `'1'` at position 2), and `"111"` < `"2"` (because `'1'` < `'2'` at position 1).
2. `-n`: recognizes numeric values. 2 < 3 < 10 < 111.
3. `sort -n` also handles negative numbers, decimal points, and scientific notation.

**What if:**
```bash
$ printf "1.5\n1.50\n1.05\n" | sort -n
1.05
1.5
1.50        # 1.5 and 1.50 are numerically equal — which comes first is unspecified
$ printf "1.5\n1.50\n1.05\n" | sort -n -s   # -s: stable sort (preserves input order for ties)
```

### Example 4: Uniq with sort for frequency

```bash
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn
     17 /bin/bash
      8 /usr/sbin/nologin
      3 /bin/sh
      2 /usr/bin/fish
```

**Step by step:**
1. `cut -d: -f7` extracts the login shell field (field 7, colon-delimited).
2. `sort` sorts the shells alphabetically so duplicates are adjacent.
3. `uniq -c` counts adjacent duplicates, prefixing each unique line with count.
4. `sort -rn` sorts the counts numerically in reverse (highest first).

**What if:**
```bash
$ cut -d: -f7 /etc/passwd | sort | uniq -c | sort -rn | head -3   # Top 3 only
$ cut -d: -f7 /etc/passwd | sort | uniq -d   # Only shells used by multiple users
$ cut -d: -f7 /etc/passwd | sort | uniq -u   # Shells used by exactly one user
```

### Example 5: Sorting by a specific column

```bash
$ printf "Alice,85\nBob,72\nCharlie,90\n" > /tmp/grades.txt
$ sort -t, -k2,2 -rn /tmp/grades.txt
Charlie,90
Alice,85
Bob,72
$ sort -t, -k2,2 -n /tmp/grades.txt
Bob,72
Alice,85
Charlie,90
```

**Step by step:**
1. `-t,` sets comma as field separator.
2. `-k2,2` specifies field 2 only.
3. `-n` sorts numerically. Without `-n`, "85" < "90" (string compare) happens to be correct but "100" would sort before "20".
4. `-rn` is numeric + reverse.

**What if:**
```bash
$ sort -t: -k3 -n /etc/passwd          # Sort by UID (field 3)
$ sort -t: -k3 -n -k7 /etc/passwd      # Sort by UID, then by shell for same UIDs
$ sort -t: -k3,3 -n /etc/passwd        # Same (explicit key end)
$ sort -t. -k1,1n -k2,2n -k3,3n -k4,4n IPs.txt  # Sort IP addresses
```

### Example 6: Paste — merging files side by side

```bash
$ printf "a\nb\nc\n" > /tmp/left.txt
$ printf "1\n2\n3\n" > /tmp/right.txt
$ paste /tmp/left.txt /tmp/right.txt
a	1
b	2
c	3
$ paste -d',' /tmp/left.txt /tmp/right.txt
a,1
b,2
c,3
```

**Step by step:**
1. `paste` opens both files, reads one line from each.
2. Writes them separated by tab (default delimiter).
3. Repeats until both files are exhausted.
4. If files have different line counts, paste fills missing lines with empty strings.

**What if:**
```bash
$ paste -d'|' /tmp/left.txt /tmp/right.txt
a|1
b|2
c|3
$ paste -s /tmp/left.txt /tmp/right.txt   # Serial: whole file as one line
a	b	c
1	2	3
```

### Example 7: Tr — character translation

```bash
$ echo "hello world" | tr 'a-z' 'A-Z'
HELLO WORLD
$ echo "UPPERcase" | tr 'A-Z' 'a-z'
uppercase
$ echo "a:b:c:d" | tr ':' '\n'
a
b
c
d
$ echo "   too   many   spaces   " | tr -s ' '
 too many spaces 
```

**Step by step:**
1. `tr 'a-z' 'A-Z'`: for each character in set1 ('a'-'z'), replace with corresponding character in set2 ('A'-'Z').
2. `tr ':' '\n'`: replace colons with newlines.
3. `tr -s ' '`: squeeze consecutive spaces into one.
4. Sets must be the same length (or one char in set2 repeats for overflow).

**What if:**
```bash
$ echo "abc123" | tr -d 'a-z'         # Delete lowercase: 123
$ echo "  hello  " | tr -d ' '        # Delete spaces: hello
$ echo "abc123xyz" | tr -d -c 'a-z'   # Delete non-lowercase: abcxyz
$ echo "hello" | tr '[:lower:]' '[:upper:]'  # Using character classes: HELLO
```

### Example 8: Join — relational merge

```bash
$ printf "1,Alice\n2,Bob\n3,Charlie\n" > /tmp/ids.csv
$ printf "1,85\n2,72\n3,90\n" > /tmp/grades.csv
$ join -t, -1 1 -2 1 /tmp/ids.csv /tmp/grades.csv
1,Alice,85
2,Bob,72
3,Charlie,90
```

**Step by step:**
1. `-t,` sets comma as separator for both files.
2. `-1 1` = join on field 1 of file 1. `-2 1` = join on field 1 of file 2.
3. Both files must be sorted on the join field.
4. Output: the join field, then remaining fields from file1, then from file2.

**What if:**
```bash
$ sort -t, -k1,1 /tmp/ids.csv -o /tmp/ids.csv     # Must sort first!
$ sort -t, -k1,1 /tmp/grades.csv -o /tmp/grades.csv
$ join -t, -a 1 ids.csv grades.csv    # Include all from file1, even if no match
$ join -t, -v 1 ids.csv grades.csv    # Only lines in file1 with no match in file2
```

### Example 9: Complex pipeline for data analysis

```bash
$ cat /tmp/grades.csv | tail -n +2 | sort -t, -k3,3 -rn | head -5 | cut -d, -f1,3
Eve,95
Alice,92
Alice,88
Bob,75
Bob,72
```

**Step by step:**
1. `tail -n +2` skips the CSV header.
2. `sort -t, -k3,3 -rn` sorts by grade (field 3) descending.
3. `head -5` keeps top 5.
4. `cut -d, -f1,3` keeps only name and grade.

**What if:**
```bash
$ sort -t, -k3,3 -rn grades.csv | head -5   # What if we didn't skip header?
# "grade" would sort with scores alphabetically!
```

### Example 10: Version sort with -V

```bash
$ printf "v1.1\nv1.10\nv1.2\nv2.0\nv1.3\n" | sort -V
v1.1
v1.2
v1.3
v1.10
v2.0
$ printf "v1.1\nv1.10\nv1.2\nv2.0\nv1.3\n" | sort   # Default
v1.1
v1.10
v1.2
v1.3
v2.0
```

**Step by step:**
1. `sort -V` recognizes version numbers: splits on dot, sorts numerically per segment.
2. Default sort: "v1.10" < "v1.2" because '1' < '2' at character 4.
3. `-V` knows that 1.10 > 1.2.

**What if:**
```bash
$ dpkg -l | grep '^ii' | awk '{print $2, $3}' | sort -V -k2  # Sort Debian packages by version
```

### Example 11: Human-readable sort

```bash
$ printf "2G\n1500M\n300K\n1T\n" | sort -h
300K
1500M
2G
1T
$ printf "2G\n1500M\n300K\n1T\n" | sort -n  # Wrong!
1T
1500M
2G
300K
```

**Step by step:**
1. `-h` understands K, M, G, T suffixes (powers of 1024).
2. Without `-h`, these are sorted lexicographically.

### Example 12: Using cut with character positions

```bash
$ echo "Hello World" | cut -c1-5
Hello
$ echo "Hello World" | cut -c7-11
World
$ echo "abcdef" | cut -c1-3,5-6
abcef
```

**Step by step:**
1. `-c1-5` = characters at positions 1 through 5 (1-indexed, inclusive).
2. `-c7-11` = starting at position 7.
3. `-c1-3,5-6` = characters 1,2,3 and 5,6 (skipping position 4).

**What if:**
```bash
$ ls -l | cut -c1-10        # First 10 characters (permissions + first space)
$ cat /etc/passwd | cut -c-10   # First 10 chars of each line
$ cat /etc/passwd | cut -c10-   # Everything from char 10 onward
```

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`ps aux | wc -l`** — count processes.
- **`ps aux | sort -nrk 4 | head -5`** — top 5 memory-consuming processes.
- **`du -sh /* | sort -rh | head -10`** — top 10 largest directories in root.
- **`last | awk '{print $1}' | sort | uniq -c | sort -rn`** — who logged in most often.
- **`df -h | grep '^/' | sort -k5 -rh`** — partitions sorted by usage percentage.

### 2. WITH the OS — Development

- **`wc -l src/**/*.py`** — count lines of Python code.
- **`git log --oneline | wc -l`** — count commits.
- **`sort -t: -k3 -n /etc/passwd | head -5`** — lowest UIDs (earliest users).
- **`cut -d' ' -f2- CHANGELOG.md | grep -i fix | wc -l`** — count bug fixes.
- **`grep -c 'TODO' src/**/*.js`** — count TODOs across project.

### 3. AGAINST the OS — Exploitation

- **`cut -d: -f1,3 /etc/shadow 2>/dev/null`** — extract usernames and password hashes (need root).
- **`sort -u /tmp/stolen_data.txt > unique.txt`** — deduplicate exfiltrated data.
- **`tr -d '[:space:]' payload.bin`** — strip whitespace from payload.
- **`ls -la | tr -s ' ' | cut -d' ' -f9`** — extract filenames from ls output (fragile but commonly used).

### 4. FOR DEFENSE — Detection & Auditing

- **`lastb | wc -l`** — count failed login attempts.
- **`cut -d: -f1 /etc/shadow | sort`** — list all system users.
- **`cat /var/log/auth.log | grep 'Failed password' | cut -d' ' -f9 | sort | uniq -c | sort -rn`** — top brute-force source IPs.
- **`find /etc -type f | wc -l`** — count config files (baseline for integrity monitoring).
- **`sort -u /var/log/auth.log > /tmp/unique_auth.log`** — deduplicate auth logs.

## Memory Aids

### Mnemonics

- **`wc -l`** = "Word Count — Lines." Say it: "wick-el" / "double-you see minus ell."
- **`sort -k`** = "sort — Key." You're sorting by a specific key field.
- **`uniq -c`** = "Unique — Count." Counts how many times each line appears.
- **`cut -d`** = "Cut — Delimiter." The `-d` sets what divides the fields.
- **`tr -s`** = "Translate — Squeeze." Squeezes repeated characters.
- **`join`** = Like SQL JOIN. You're joining two files on a common field.

### Pipeline pattern hooks

The most famous pipeline formula:

```bash
cut -d: -fN file | sort | uniq -c | sort -rn
```

This is the "frequency analysis" pipeline: extract, sort, count, sort by count. Memorize it.

Other patterns:
- `command | wc -l` = "count results"
- `sort -u` = "unique sorted" (same as `sort | uniq`)
- `sort -t: -k3 -n` = "sort by numeric field 3 with colon separator"
- `tr 'A-Z' 'a-z'` = "lowercase everything"

### Common confusions

- **`uniq` requires sorted input**: This is the #1 mistake. `sort | uniq` — always.
- **`cut -d' '` vs `cut -d ' '`**: The argument to `-d` is a single character. `-d' '` sets delimiter to space.
- **`tr` doesn't do words**: `tr 'abc' 'xyz'` replaces a→x, b→y, c→z. It does NOT replace the string "abc."
- **`wc -c` vs `wc -m`**: `-c` = bytes, `-m` = characters (multibyte-aware). For ASCII, same. For UTF-8, different.
- **`sort -g` vs `sort -n`**: `-g` = general numeric (handles scientific notation). `-n` = simple numeric. `-g` is slower.

## Trap Vault

### Trap 1: uniq without sort

**Problem:** Duplicates remain because they're not adjacent.

**Example:**
```bash
$ printf "a\nb\na\n" | uniq
a
b
a
```

**Why:** `uniq` only compares adjacent lines. The two `a` lines are separated by `b`, so they're not seen as duplicates.

**Fix:**
```bash
$ printf "a\nb\na\n" | sort | uniq
a
b
```

### Trap 2: cut default delimiter is TAB

**Problem:** `cut -f2` doesn't work on comma-separated files.

**Example:**
```bash
$ echo "a,b,c" | cut -f2
a,b,c    # No splitting! Output unchanged.
$ echo "a,b,c" | cut -d, -f2
b
```

**Why:** `cut`'s default delimiter is TAB, not space or comma. You must use `-d` explicitly.

**Fix:** Always specify `-d`:
```bash
$ cut -d',' -f2 file.csv
$ cut -d' ' -f2 file.txt   # But spaces can be tricky (multiple spaces)
$ cut -d: -f1 /etc/passwd  # Works for /etc/passwd
```

### Trap 3: sort -n on non-numeric data

**Problem:** `sort -n` on mixed data can produce unexpected results.

**Example:**
```bash
$ printf "abc\n123\n" | sort -n
123
abc
$ printf "abc\n123\n" | sort
123
abc
```

Wait, that's actually the same output! But:
```bash
$ printf "abc\n123\n45\n" | sort -n
45
123
abc
$ printf "abc\n123\n45\n" | sort
123
45
abc
```

**Why:** `sort -n` extracts numeric prefixes. "45" is 45, "123" is 123. Non-numeric lines (like "abc") are treated as 0 or NaN and sorted accordingly.

**Fix:** Be aware of what your data looks like. For strictly numeric, ensure all lines start with numbers.

### Trap 4: cut -f with --complement and output delimiter

**Problem:** `--complement` with `--output-delimiter` can produce confusing results.

**Example:**
```bash
$ echo "a:b:c" | cut -d: -f2 --complement
a:c          # Missing --output-delimiter, so it uses the input delimiter
$ echo "a:b:c" | cut -d: -f2 --complement --output-delimiter=':'
a:c          # Same
```

**Why:** `--complement` inverts the selection. `cut -d: -f2 --complement` selects fields 1 and 3 (everything except field 2).

**Fix:** Use `--output-delimiter` when you want a different separator in the output. When using `--complement`, remember what you're selecting (the complement of your specification).

### Trap 5: tr with multi-character sets

**Problem:** `tr` does NOT replace strings — only individual characters.

**Example:**
```bash
$ echo "abc" | tr 'abc' 'xyz'
xyz     # Replaces a->x, b->y, c->z
$ echo "abc" | tr 'abc' 'x'
xxx     # a->x, b->x, c->x (set2 repeats last char)
```

**Why:** `tr` works on characters, not strings. If set2 is shorter than set1, the last character of set2 repeats.

**Fix:** For string replacement, use `sed`:
```bash
$ echo "abc" | sed 's/abc/xyz/'
xyz
```

### Trap 6: sort no output for large file with -o

**Problem:** `sort file > file` destroys the file.

**Example:**
```bash
$ sort /tmp/data.txt > /tmp/data.txt   # DANGER! File truncated!
$ cat /tmp/data.txt
# Empty!
```

**Why:** The shell's `>` redirect truncates the file before `sort` reads it. `sort` sees an empty file.

**Fix:** Use `-o` for in-place sorting:
```bash
$ sort -o /tmp/data.txt /tmp/data.txt   # Correct! sort manages the I/O
$ sort /tmp/data.txt > /tmp/data.txt.new && mv /tmp/data.txt.new /tmp/data.txt  # Safe
```

### Trap 7: join requires sorted input

**Problem:** `join` silently produces wrong output on unsorted files.

**Example:**
```bash
$ printf "2,Bob\n1,Alice\n" > f1.txt
$ printf "2,85\n1,90\n" > f2.txt
$ join -t, f1.txt f2.txt
# Wrong output or nothing!
```

**Why:** `join` assumes both files are sorted on the join field. If not, it produces incorrect results (or misses matches entirely) because it uses a linear merge algorithm.

**Fix:**
```bash
$ sort -t, -k1,1 -o f1.txt f1.txt
$ sort -t, -k1,1 -o f2.txt f2.txt
$ join -t, f1.txt f2.txt
```

### Trap 8: wc -l counts newlines, not lines

**Problem:** A file without a trailing newline is counted as one fewer line.

**Example:**
```bash
$ printf "hello" > /tmp/nonl.txt    # No newline
$ printf "hello\n" > /tmp/withnl.txt
$ wc -l /tmp/nonl.txt /tmp/withnl.txt
0 /tmp/nonl.txt      # 0 lines!
1 /tmp/withnl.txt
```

**Why:** `wc -l` counts the number of newline characters (`\n`). A file without a final newline has one fewer "line" by this count, even though it has content.

**Fix:** Be aware that standard Unix text files end with a newline. Most tools ensure this. If you generate text without a final newline, `wc -l` will undercount by one.

### Trap 9: sort -k field numbering

**Problem:** `-k` field numbering is 1-based, but people often confuse it with 0-based.

**Example:**
```bash
$ printf "a b c\n" | sort -k1   # Sort by field 1 (a)
$ printf "a b c\n" | sort -k2   # Sort by field 2 (b)
```

**Why:** `-k` is 1-based. Field 1 is the first field. This catches many programmers used to 0-based indexing.

**Fix:** Remember: `-k1` is first field. If you want the last field, use `rev` or `awk`:
```bash
$ printf "a b c\n" | awk '{print $NF, $0}' | sort | cut -d' ' -f2-
```

### Trap 10: tr: Illegal byte sequence

**Problem:** `tr` in non-C locales fails on multi-byte UTF-8 characters.

**Example:**
```bash
$ echo "café" | tr 'a-z' 'A-Z'
tr: Illegal byte sequence
```

**Why:** In UTF-8 locales, `tr` tries to handle multi-byte characters. The range `a-z` in UTF-8 is interpreted differently, and the `é` character (0xC3 0xA9) contains bytes that don't fit in the range.

**Fix:**
```bash
$ LC_ALL=C echo "café" | tr 'a-z' 'A-Z'
CAFé     # But é is not uppercased to É
$ echo "café" | tr '[:lower:]' '[:upper:]'   # Locale-aware
CAFÉ
```

## See It In The Wild

### Where you encounter these commands daily

- **`du -sh * | sort -rh`** — find disk hogs (every sysadmin's daily routine).
- **`history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10`** — your most used commands.
- **`git shortlog -sn`** — count commits per author (uses sort and uniq internally).
- **`grep -c`** — count matches (like `grep | wc -l` in one command).
- **`df -h | sort -k5 -rh`** — filesystems sorted by usage.

### Try this now

```bash
# What's your most used command?
history | awk '{print $2}' | sort | uniq -c | sort -rn | head -10

# Count processes per user
ps aux | awk '{print $1}' | sort | uniq -c | sort -rn

# Largest files in /var/log
ls -lS /var/log | head -10

# Longest man page
man -k . | awk '{print $1}' | xargs -I{} sh -c 'man {} | wc -l | tr -d " "' | sort -rn | head -5

# Find files with the most lines
find /usr/src -name '*.c' -exec wc -l {} + 2>/dev/null | sort -rn | head -10
```

## Check Your Understanding

<details>
<summary>1. You have a file with duplicate lines scattered throughout. You run `uniq` but duplicates remain. Why?</summary>

`uniq` only removes ADJACENT duplicates. If your input isn't sorted, duplicates scattered throughout the file won't be detected. Always run `sort | uniq` (or use `sort -u` which combines both operations).
</details>

<details>
<summary>2. What's the difference between `sort -u` and `sort | uniq`?</summary>

Functionally identical, but `sort -u` only needs to read the file once (it deduplicates during sorting). `sort | uniq` writes the sorted output to the pipe, then `uniq` reads it and removes adjacent duplicates. `sort -u` is slightly more efficient and is the preferred form.
</details>

<details>
<summary>3. Why does `wc -c` sometimes differ from `wc -m`?</summary>

`-c` counts bytes. `-m` counts characters (multibyte-aware). For ASCII text, they're the same. For UTF-8 text with multi-byte characters (like `é` = 2 bytes, `ñ` = 2 bytes, emoji = 4 bytes), `-c` will be higher than `-m`.
</details>

<details>
<summary>4. Your CSV file has fields separated by commas. You want the third field. What command do you use?</summary>

`cut -d',' -f3 file.csv`. Remember that `cut`'s default delimiter is TAB, not comma.
</details>

<details>
<summary>5. How does `sort -h` differ from `sort -n`?</summary>

`-h` (human-numeric) understands size suffixes: K, M, G, T, P, E, Z, Y (powers of 1024). `-n` only understands plain numbers. So `sort -h` correctly sorts: 100K < 2M < 1G. Without `-h`, these would sort lexicographically.
</details>

<details>
<summary>6. You run `sort file1 > file1` and the file is now empty. Why?</summary>

The shell's `>` redirect truncates `file1` to zero bytes BEFORE `sort` reads it. `sort` sees an empty file and writes nothing. Use `sort -o file1 file1` or redirect to a temp file and rename.
</details>

<details>
<summary>7. What does `tr -s '[:space:]'` do? When would you use it?</summary>

It "squeezes" all whitespace characters (space, tab, newline, etc.) into single spaces. Use it to normalize whitespace in text, e.g., `echo "too   many  spaces" | tr -s ' '` becomes "too many spaces" (but `-s '[:space:]'` also squeezes tabs and other whitespace).
</details>
