# Task 3: Glob Master Challenge

You'll create a messy directory of test files and use every glob feature to select, filter, and manipulate them. Brace expansion, character classes, extglob, nullglob, and dotglob — all in action.

## Setup

```bash
$ mkdir -p /tmp/glob_lab
$ cd /tmp/glob_lab
```

## Sub-task 1: Generate the Mess with Brace Expansion

Use brace expansion to create a set of test files in one command:

```bash
$ touch {report,data,log,backup,temp}-{2023,2024,2025}-{01..03}.{txt,csv,log}
$ touch .hidden .secret .config
$ touch "my file.txt" "important - backup (v2).csv"
```

**Expected result:** How many files did you create?

<details>
<summary>Counting the files</summary>
5 prefixes x 3 years x 3 months x 3 extensions = 135 files. Plus 3 hidden + 2 with spaces = 140 files.
</details>

Verify with: `ls | wc -l` and `ls -la | grep '^\.' | wc -l`.

## Sub-task 2: Basic Globs

Write `ls` commands that match:

1. All `.txt` files
2. All files with `2024` in the name
3. All files where the prefix is exactly 4 characters long
4. All files with `data` or `log` prefix

**Expected output:**
```bash
$ ls *.txt | head -5
data-2023-01.txt
data-2023-02.txt
data-2023-03.txt
data-2024-01.txt
data-2024-02.txt

$ ls *2024* | wc -l
45   # (5 prefixes x 3 months x 3 extensions = 45 files for 2024)

$ ls ????-* | head -3
backup-2023-01.csv
backup-2023-01.log
backup-2023-01.txt

$ ls {data,log}* | wc -l
54   # (2 prefixes x 3 years x 3 months x 3 extensions = 54)
```

<details>
<summary>Hint: counting with wc</summary>
The `wc -l` counts lines. When `ls` outputs one file per line (non-interactive mode), `wc -l` counts the files. But beware: `ls` in a terminal uses columns, not lines. When piped, it uses lines.
</details>

## Sub-task 3: Character Classes

1. List files containing a digit (`[0-9]`) in the name
2. List files whose prefix starts with `r` or `t` (but not `d` or `b`)
3. List files whose extension is `csv` OR `log` (use `*.{csv,log}`)
4. List files whose name does NOT contain a hyphen

**Expected output:**
```bash
$ ls *[0-9]* | head -3
backup-2023-01.csv
backup-2023-01.log
backup-2023-01.txt

$ ls [rt]* | head -3
report-2023-01.csv
report-2023-01.log
report-2023-01.txt

$ ls *.@(csv|log) | head -3
backup-2023-01.csv
backup-2023-01.log
data-2023-01.csv
```

Wait — that last one needs `extglob`!

## Sub-task 4: Extended Globs

Enable `extglob` and explore:

```bash
$ shopt -s extglob
```

Now:

1. Match files ending in `.txt` or `.csv`: `ls *.@(txt|csv)` — how many?
2. Match files NOT ending in `.log`: `ls !(*.log)` — careful, this matches everything else
3. Match files with exactly one digit in the name before the extension
4. Match files that start with a vowel (a, e, i, o, u): `ls [aeiou]*`

**Expected output:**
```bash
$ shopt -s extglob
$ ls *.@(txt|csv) | wc -l
90   # (135 * 2/3 = 90 files that are .txt or .csv)

$ ls !(*.log) | wc -l
92   # 135 - 45 (.log files) = 90 non-log, plus 2 space-named = 92? Wait, check counts
```

<details>
<summary>Understanding extglob negation</summary>
`!(*.log)` matches everything that doesn't match `*.log`. This includes directories, hidden files (if dotglob is set), and the `.` and `..` entries. Be careful with counts!
</details>

## Sub-task 5: Hidden Files and dotglob

```bash
$ ls -a | grep '^\.' | wc -l
3    # . .. and .hidden .secret .config = wait, . and .. are 2, plus .hidden .secret .config = 5

$ echo .*          # What does this show?
. .. .config .hidden .secret

$ shopt -s dotglob
$ echo *           # Now what?
```

**Question:** Why does `echo .*` include `.` and `..`? How would you list ONLY user-created hidden files (not `.` or `..`)?

<details>
<summary>Excluding . and ..</summary>
Use `echo .[!.]*` to match dotfiles with at least 2 characters starting with `..` but NOT `.` alone or `..` alone. Or use `echo .??*` which requires at least 3 characters (dot + two more). Safer: `ls -A` (almost-all).
</details>

## Sub-task 6: nullglob and failglob

Create a scenario where no matches exist:

```bash
$ echo *.nonexistent
*.nonexistent    # Literal! No error?

$ shopt -s nullglob
$ echo *.nonexistent
                 # Empty — pattern vanished

$ shopt -s failglob
$ echo *.nonexistent
bash: no match: *.nonexistent
```

Now write a for loop that iterates over `*.nonexistent` with each nullglob setting and observe what happens:

```bash
$ shopt -u nullglob
$ for f in *.nonexistent; do echo "File: $f"; done
File: *.nonexistent   # Bug! Iterates over the literal string!

$ shopt -s nullglob
$ for f in *.nonexistent; do echo "File: $f"; done
# No output — correct
```

**Question:** Which setting should you use in scripts that iterate over files?

## Sub-task 7: Globbing Directories

```bash
$ mkdir -p /tmp/glob_lab/sub/{dir_a,dir_b,dir_c}
$ touch /tmp/glob_lab/sub/{dir_a,dir_b,dir_c}/file.txt

# List only directories
$ ls -d */           # */ matches only directories
$ ls -d sub/*/
sub/dir_a/ sub/dir_b/ sub/dir_c/

# List files within specific subdirectories
$ ls sub/{dir_a,dir_c}/*.txt
sub/dir_a/file.txt  sub/dir_c/file.txt
```

**Question:** Why does `ls */` only show directories? What happens if you have a file named with a trailing asterisk?

<details>
<summary>The */ trick</summary>
`*/` is a glob that matches "any name with a trailing slash." Only directories have a trailing slash in glob expansion. Regular files don't. It's the simplest way to select only directories.
</details>

## Sub-task 8: Globstar Recursive Walk

```bash
$ shopt -s globstar

# Count all files recursively
$ ls **/*.txt | wc -l

# Find specific files deep in the tree
$ ls sub/**/*.txt
sub/dir_a/file.txt sub/dir_b/file.txt sub/dir_c/file.txt

$ shopt -u globstar   # Disable when done
```

**Question:** How is `**/*.txt` different from `find . -name '*.txt'` in terms of performance and output?

<details>
<summary>** vs find</summary>
`**` is a bash feature that recursively expands globs. `find` is an external command that walks the directory tree. `find` is usually faster on large trees, doesn't have ARG_MAX issues, and can filter by size, time, permissions, etc. `**` is convenient for simple cases but can be slow and memory-heavy on big directories.
</details>

## Sub-task 9: The GLOBIGNORE Blacklist

```bash
$ export GLOBIGNORE="*.log:temp-*"

$ ls *
# No .log files, no temp-* files
$ ls *.log
# Empty — even explicit globs are suppressed
```

**Question:** What happens if you try to `rm *.log` with `GLOBIGNORE` set? Is this safe?

## Bonus Challenge: The Glob Puzzle

You have this directory:

```
files/
  a1.txt  a2.txt  a3.txt  b1.txt  b2.txt
  a1.csv  a2.csv  a3.csv  b1.csv
  .hidden_data.txt
  .config.yml
  my config.txt
  readme.md
```

Write globs that match:

1. Only `.txt` files starting with `a`: `_____`
2. All files except `.txt`: `_____`
3. Files with a digit but NOT `b1`: `_____`
4. Only the hidden files (not `.` or `..`): `_____`
5. The file with a space in its name: `_____`
6. Files ending in `.txt` or `.md`: `_____`
7. Every file in the directory (including hidden): `_____`
8. Files where the extension starts with `c`: `_____`

<details>
<summary>Answers</summary>
1. `a*.txt`
2. With extglob: `!(*.txt)`
3. `*[0-9]*` then exclude b1... tricky. `[!b]*[0-9]*` or `!(b1)*[0-9]*`
4. `.[!.]*` or `.[!.]?*`
5. `my\ config.txt` or `"my config.txt"`
6. `*.@(txt|md)`
7. `{.,}*` or `shopt -s dotglob; *`
8. `*.c*` or `*.@(csv|conf|yml)`
</details>

## Self-Check

1. Does glob expansion happen before or after variable expansion? Why does this distinction matter for quoting?
2. You have `shopt -u nullglob` (default). What does `rm *.log` do when there are no `.log` files?
3. What's the difference between `[!a]` and `[^a]` in glob patterns?
4. How would you match all `.jpg`, `.jpeg`, and `.png` files with a single glob pattern?
5. Why does `mkdir -p {2023,2024}/{01..12}` create 24 directories? Walk through the expansion order.
6. What's the practical difference between `find . -name '*.py'` and `**/*.py` (with globstar)?
7. If `GLOBIGNORE="*.o"` is set and you need to delete `.o` files anyway, what do you do?
