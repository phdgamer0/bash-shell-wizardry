# Task 13: Subshells & Grouping

You'll write scripts that demonstrate subshell isolation, command grouping, process substitution, and the infamous pipeline variable-loss bug. These are essential for understanding bash's execution model — and avoiding bugs that have haunted sysadmins for decades.

## Sub-tasks

### 1. Isolation demo — `isolation.sh`

Write a script that clearly demonstrates the difference between subshell `( )` and group `{ }` for variable scope, directory changes, and traps.

**Requirements:**
- Show a variable set inside `( )` is lost after the subshell
- Show a variable set inside `{ }` persists after the group
- Show `cd` inside `( )` doesn't affect the parent
- Show `cd` inside `{ }` does affect the parent (then `cd` back)
- Show `trap` inside `( )` doesn't affect the parent
- Show `exit` inside `( )` doesn't exit the script
- Show `exit` inside `{ }` does exit the script (be careful with this one!)
- Print section headers so the output is self-documenting

**Expected:**
```bash
$ ./isolation.sh
=== Variable Isolation ===
Before subshell: x=before
Inside subshell: x=inside_subshell
After subshell:  x=before        ← LOST

Before group: x=before
Inside group: x=inside_group
After group:  x=inside_group     ← PRESERVED

=== Directory Isolation ===
Original dir: /home/phd/projects
Inside subshell: /tmp
After subshell:  /home/phd/projects ← RESTORED

Inside group: /tmp
After group:  /tmp                   ← CHANGED (then cd back)

=== Exit Behavior ===
Subshell exit doesn't stop script (script continues)
Group exit would stop script (commented out for safety)

=== Trap Isolation ===
Trap set in subshell doesn't affect parent
Parent trap still original
```

### 2. Directory comparison — `cmpdirs.sh`

Write a script that uses process substitution to compare two directories.

**Requirements:**
- Accept two directory paths as arguments
- Show files unique to dir1, dir2, and common to both
- Use `diff` with process substitution to compare `ls` outputs
- Show a side-by-side summary
- Handle non-existent directories gracefully (error message)
- Option: Use `comm` instead of `diff` for cleaner output

**Expected:**
```bash
$ mkdir /tmp/dir1 /tmp/dir2
$ touch /tmp/dir1/a.txt /tmp/dir1/b.txt /tmp/dir1/c.txt
$ touch /tmp/dir2/b.txt /tmp/dir2/c.txt /tmp/dir2/d.txt

$ ./cmpdirs.sh /tmp/dir1 /tmp/dir2
Files only in /tmp/dir1:
  a.txt

Files only in /tmp/dir2:
  d.txt

Files in both:
  b.txt
  c.txt

=== Side-by-side ===
a.txt             | <
b.txt             | b.txt
c.txt             | c.txt
                  | > d.txt

$ ./cmpdirs.sh /tmp/dir1 /nonexistent
Error: /nonexistent is not a directory
```

### 3. Group redirection — `report.sh`

Write a script that generates a formatted system report using `{ }` group redirection.

**Requirements:**
- Use `{ }` to group ALL output into a single file with one redirect
- Include in the report:
  - Header with title and date
  - System information (uname -a)
  - Disk usage (df -h)
  - Memory info (free -h)
  - Logged-in users (who)
  - Uptime (uptime)
  - Running processes count (ps aux | wc -l)
- Accept `-o filename` to specify output file (default: `report.txt`)
- Print "Report written to [file]" at the end
- Set variables inside the group that are used after the group exits

**Edge cases:**
- If output file can't be written, show error
- Ensure variables set inside `{ }` are accessible afterward

**Expected:**
```bash
$ ./report.sh
Report written to report.txt

$ cat report.txt
╔══════════════════════════════════════╗
║   System Report                      ║
║   Generated: Thu Jul 31 12:00:00     ║
╚══════════════════════════════════════╝

--- System ---
Linux kali 6.1.0 x86_64 GNU/Linux

--- Disk ---
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   50G   50G  50% /

--- Memory ---
               total   used   free
Mem:            15G     8G     7G

--- Users ---
user     pts/0    2026-07-31 11:00

--- Uptime ---
 12:00:00 up 3 days, 2:15, 1 user

--- Processes ---
Running: 187
```

### 4. Pipeline vs process substitution — `wordcount.sh`

Write a script that demonstrates the pipeline variable-loss bug and the process substitution fix.

**Requirements:**
- Count lines in a file using BOTH methods and show the difference
- Method 1 (broken): `cat file | while read line; do ((count++)); done`
- Method 2 (fixed): `while read line; do ((count++)); done < <(cat file)`
- Method 3 (best): `while read line; do ((count++)); done < file`
- Show the resulting counts
- Also demonstrate that variables set in a subshell via `|` are inaccessible

**Edge cases:**
- Empty file — count should be 0
- File with no trailing newline — last line counted correctly
- Show the actual mechanisms (print BASHPID inside each loop)

**Expected:**
```bash
$ cat test.txt
line 1
line 2
line 3

$ ./wordcount.sh test.txt
=== Pipeline Method (BROKEN) ===
Inside while (BASHPID=12346): count=1
Inside while (BASHPID=12346): count=2
Inside while (BASHPID=12346): count=3
After pipeline: count=0           ← LOST! (subshell)

=== Process Substitution Method (FIXED) ===
Inside while (BASHPID=12345): count=1
Inside while (BASHPID=12345): count=2
Inside while (BASHPID=12345): count=3
After process sub: count=3        ← PRESERVED (same shell)

=== Direct Redirection Method (BEST) ===
Inside while (BASHPID=12345): count=1
Inside while (BASHPID=12345): count=2
Inside while (BASHPID=12345): count=3
After direct: count=3             ← PRESERVED (same shell)
```

## Solution Approaches

### Approach A: Subshell-first
1. `isolation.sh` — demonstrates core concepts
2. `wordcount.sh` — shows practical impact of subshells
3. `cmpdirs.sh` — applies process substitution
4. `report.sh` — applies grouping

### Approach B: Tool-first
1. `cmpdirs.sh` — practical file comparison
2. `report.sh` — generates useful output
3. `wordcount.sh` — debugging pipeline bug
4. `isolation.sh` — theoretical understanding

### Approach C: Bug-hunting
1. `wordcount.sh` — encounter the subshell bug first
2. `isolation.sh` — understand WHY it happens
3. `report.sh` — use grouping to solve redirection
4. `cmpdirs.sh` — use process substitution for comparison

<details>
<summary>Hint 1: BASHPID to prove subshell</summary>

```bash
echo "Before: BASHPID=$BASHPID"
(
    echo "Inside subshell: BASHPID=$BASHPID"
)
echo "After: BASHPID=$BASHPID"
```
`$$` does NOT change in subshells — use `$BASHPID`.
</details>

<details>
<summary>Hint 2: cmddirs with comm</summary>

```bash
comm <(ls -1 "$1" 2>/dev/null) <(ls -1 "$2" 2>/dev/null)
```
`comm` needs sorted input. `ls -1` output is sorted.
- Column 1: unique to file 1
- Column 2: unique to file 2
- Column 3: common to both

Use `-1`, `-2`, `-3` flags to suppress columns.
</details>

<details>
<summary>Hint 3: Group redirection pattern</summary>

```bash
output="${1:-report.txt}"
{
    echo "╔════════════════╗"
    echo "║  System Report ║"
    echo "║  $(date)       ║"
    echo "╚════════════════╝"
    echo ""
    echo "--- System ---"
    uname -a
    echo ""
    echo "--- Disk ---"
    df -h
} > "$output"
echo "Report written to $output"
```
</details>

<details>
<summary>Hint 4: Pipeline BASHPID demo</summary>

```bash
echo "Parent BASHPID: $BASHPID"
count=0
cat "$file" | while read line; do
    ((count++))
    echo "In pipeline (BASHPID=$BASHPID): count=$count"
done
echo "After pipeline: count=$count (BASHPID=$BASHPID)"
```
</details>

<details>
<summary>Hint 5: Trapping exit in groups vs subshells</summary>

```bash
# Subshell exit doesn't stop script
( echo "About to exit subshell"; exit 42; echo "This won't print" )
echo "But this DOES print — subshell exit doesn't kill parent"
echo "Exit code was: $?"   # shows 42

# Group exit STOPS the script (commented for safety)
# { echo "About to exit group"; exit 42; echo "This won't print"; }
# echo "This won't print either"
```
</details>

<details>
<summary>Hint 6: One-liners for testing</summary>

Quick tests:
```bash
# Subshell isolation
x=1; (x=2); echo $x    # 1

# Group persistence
x=1; { x=2; }; echo $x  # 2

# Process substitution diff
diff <(echo a; echo b) <(echo b; echo c)

# Pipeline vs redirect
echo "hello" | read var; echo $var     # empty (subshell)
read var <<< "hello"; echo $var        # hello (no subshell)
```
</details>

## Bonus Challenges

1. **FIFO explorer**: `ls -la <(echo test)` — see what process substitution creates. Write a script that shows the file type.
2. **Nested command substitution**: Write a single line that extracts the third field of a CSV, capitalizes it, and wraps it in JSON — all with nested `$( )`.
3. **Group-within-subshell**: `( { cmd1; cmd2; } | cmd3 )` — predict and test the behavior. Write a script demonstrating four levels of nesting.
4. **Performance benchmark**: Write a script that times `$(< file)` vs `$(cat file)` vs `while read` for reading a 10MB file. Show the time differences.
5. **File descriptor leak detector**: Write a script that uses process substitution in a loop and tracks open FDs with `ls /proc/self/fd` to detect leaks.

## Expected Output Summary

```
isolation.sh:
  Demonstrates ( ) vs { } for variables, cd, exit, traps
  BASHPID shows different PIDs in subshells
  Self-documenting output with section headers

cmpdirs.sh:
  Process substitution with diff or comm
  Two-directory comparison
  Files unique to each + files in common
  Error handling for invalid directories

report.sh:
  Group redirection with { } > file
  System report with multiple sections
  Variables set inside group accessible after

wordcount.sh:
  Pipeline: count=0 (broken)
  Process substitution: count=N (fixed)
  Direct redirection: count=N (best)
  BASHPID verification
```

## Self-Check

- What's the difference between `( )` and `{ }` regarding variable scope?
- Why does `echo "data" | read var` lose `$var` but `read var <<< "data"` doesn't?
- What does process substitution `<()` actually create at the filesystem level?
- What's the advantage of `$( )` over backticks?
- What happens if you forget the space before `}` in a group command?
- How do `$$` and `$BASHPID` differ in behavior inside subshells?
- Why is `$(< file)` faster than `$(cat file)`?
- How would you capture both stdout and stderr from a command substitution?
