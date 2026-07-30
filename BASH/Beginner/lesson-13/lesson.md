# Lesson 13: Loops

## History & Origins

Loops in shell scripting date back to the Thompson shell (1971), which had a `goto`-based looping mechanism. The Bourne shell (1977) introduced `for`, `while`, and `until` loops in a form we'd recognize today. The C-style `for ((...))` was added by bash (ksh had it first) — it borrows syntax from C's `for` loop.

The `for var in list` construct evolved from the idea of iterating over command arguments. In early Unix, shell scripts often processed one file at a time using `for f in *.txt` — a pattern that persists unchanged today.

The `while read` pattern for file processing was a breakthrough. Early shells had trouble reading files line by line because they used line-based input internally. The `while IFS= read -r line` incantation evolved to handle edge cases: leading/trailing whitespace (`IFS=` prevents stripping), backslash interpretation (`-r` disables it), and the final line without a newline (POSIX requires `read` to return 0 on EOF before newline, which it does only when `IFS=` and `-r` are used).

The `break N` and `continue N` forms (where N is a number of loop levels) were added in bash to handle nested loops, a feature lacking in the original Bourne shell.

Fun fact: In early Unix V7, the `for` loop didn't support the `in` keyword — it was `for var` and would automatically iterate over `$@` (the arguments). The `in list` form was a later addition for explicit iteration.

## Syntax Reference

### For loop (iterate over list)
```bash
for var in word1 word2 word3; do
    commands
done

for var in "$@"; do        # Iterate over arguments
for var; do                # Implicit: same as "for var in "$@""
for f in *.txt; do         # File globbing
for f in /etc/*.conf; do   # Path globbing
for var in {1..10}; do     # Brace expansion
for var in {a..z}; do      # Letter range
for var in $(cat file); do # DANGEROUS: word splits, glob expands
```

### C-style for loop
```bash
for (( init; condition; increment )); do
    commands
done

for (( i = 0; i < 10; i++ )); do        # Count up
for (( i = 10; i > 0; i-- )); do        # Count down
for (( i = 0, j = 10; i < j; i++, j-- )); do  # Multiple vars
for (( ; ; )); do                        # Infinite loop
```

### While loop
```bash
while condition; do
    commands
done

while [ "$i" -lt 10 ]; do        # Counter-based
while read -r line; do           # Read file line by line
while IFS= read -r line; do      # Read file preserving whitespace
while IFS=, read -r f1 f2 f3; do # Parse CSV
while true; do                   # Infinite loop
while :; do                      # Infinite loop (shorthand, : is no-op)
```

### Until loop
```bash
until condition; do
    commands
done

# until is like "while not" — runs while condition is false
until [ -f /tmp/done ]; do
    sleep 1
done
```

### Loop control
```bash
break         # Exit current loop
break N       # Exit N levels of nested loops
continue      # Skip to next iteration
continue N    # Skip N levels of nested loops
```

### Loop-related
```bash
select var in list; do      # Interactive menu loop
    break
done

# Special variables for loops:
$RANDOM       # Random number 0-32767
$LINENO       # Current line number
$SECONDS      # Seconds since shell start
```

## Under the Hood

### How `for f in *.txt` Works

1. **Pathname expansion**: Bash encounters `*.txt`. It calls the internal globber which reads the current directory and matches filenames against the pattern.
2. **Word splitting**: The resulting list is split (though glob results don't need splitting).
3. **Assignment**: Bash assigns each word to `$f` in sequence.
4. **No glob match**: If no files match and `nullglob` is off, `*.txt` is treated literally. If `nullglob` is on, the loop body never executes.

### How `while IFS= read -r line` Works

1. **Redirection**: `< file` opens the file for reading on fd 0 (stdin).
2. **read builtin**: `read` reads bytes from stdin until it hits a newline (or EOF with `-d ''`).
3. **IFS=**: Sets IFS to empty for the `read` command only, preventing leading/trailing whitespace stripping.
4. **-r**: Raw mode — backslashes are treated literally, not as escape characters.
5. **Exit code**: `read` returns 0 if it read any data (including the final line even without newline). Returns 1 on EOF with no data.

### How C-style `for ((i=0; i<10; i++))` Works

1. **Init**: `i=0` is evaluated as an arithmetic expression.
2. **Condition**: `i<10` is evaluated. If 0 (false), loop exits.
3. **Body**: Commands execute.
4. **Increment**: `i++` is evaluated as an arithmetic expression.
5. **Repeat**: Go to step 2.

All expressions are evaluated in arithmetic context — no `$` needed for variables.

### Memory and Process Model

```bash
for f in *.txt; do
    wc -l "$f"
done
```

This does NOT fork a process for the loop itself. The loop runs within the current shell. Each `wc -l "$f"` forks a new process, but the loop control is entirely in-process.

For large lists:
```bash
for f in {1..1000000}; do :; done  # Expands entire list in memory!
```
Brace expansion `{1..N}` creates ALL N words in memory before the loop starts. For N=1M, that's ~7MB of strings.

```bash
for (( i=1; i<=1000000; i++ )); do :; done  # No expansion — counter in memory
```
C-style loop uses a single integer variable — memory efficient.

### strace of a While-Read Loop

```bash
$ strace -e trace=read,write bash -c 'while IFS= read -r line; do echo "X$line"; done < /etc/hostname'
read(3, "myhostname\n", 8192)    = 11
write(1, "Xmyhostname\n", 12)    = 12
read(3, "", 8192)                = 0
```

The `read` builtin reads from fd 3 (the opened file) in 8K chunks. It splits on newlines internally. Each iteration calls `read()` to refill the internal buffer.

## Core Examples (12)

### Example 1: Iterate Over Files — The Classic
```bash
$ for f in *.txt; do
    echo "Processing: $f"
    wc -l "$f"
  done
Processing: file.txt
  12 file.txt
Processing: notes.txt
  8 notes.txt
```
The glob expands to matching filenames. Each is assigned to `$f` in turn. The `wc -l` counts lines.

**What if:** No `.txt` files exist? Without `nullglob`, the loop runs once with `f=*.txt` (literal). With `nullglob`, the loop body never runs.

### Example 2: Iterate Over an Explicit List
```bash
$ for fruit in apple banana cherry; do
    echo "I like $fruit"
  done
I like apple
I like banana
I like cherry
```
Simple list iteration. The list is hardcoded — no globbing needed.

### Example 3: C-Style For Loop — Counting
```bash
$ for (( i = 1; i <= 5; i++ )); do
    echo "Count: $i"
  done
Count: 1
Count: 2
Count: 3
Count: 4
Count: 5
```
C-style `for` uses three expressions: initialization, condition, increment. Note no `$` on `i` inside `(( ))`.

### Example 4: While-Read — The Correct Way to Read a File
```bash
$ while IFS= read -r line; do
    echo "Line: $line"
  done < /etc/hostname
Line: desktop
```
The `IFS=` prevents trimming whitespace. `-r` prevents backslash interpretation. `< file` feeds the file to the loop's stdin.

**What if:** You omit `IFS=`? Leading/trailing whitespace is stripped. Omit `-r`? Backslashes are interpreted — `\n` becomes a newline, `\t` becomes a tab.

### Example 5: Until Loop — Waiting for a Condition
```bash
$ count=5
$ until [ "$count" -eq 0 ]; do
    echo "Countdown: $count"
    ((count--))
  done
Countdown: 5
Countdown: 4
Countdown: 3
Countdown: 2
Countdown: 1
```
`until` is the inverse of `while`. The loop runs AS LONG AS the condition is FALSE. When condition becomes true, the loop stops.

### Example 6: break and continue
```bash
$ for i in {1..10}; do
    if [ "$i" -eq 3 ]; then
        continue    # skip 3
    fi
    if [ "$i" -eq 7 ]; then
        break       # stop at 7
    fi
    echo "i=$i"
  done
i=1
i=2
i=4
i=5
i=6
```
`continue` jumps to the next iteration. `break` exits the loop entirely.

### Example 7: Nested Loops — Multiplication Table
```bash
$ for i in {1..5}; do
    for j in {1..5}; do
        printf "%3d " $((i * j))
    done
    echo
  done
  1   2   3   4   5
  2   4   6   8  10
  3   6   9  12  15
  4   8  12  16  20
  5  10  15  20  25
```
Nested loops: outer loop controls rows, inner loop controls columns. `printf "%3d"` formats the output with padding.

### Example 8: Select Menu Loop
```bash
$ PS3="Choose a fruit: "
$ select fruit in apple banana cherry quit; do
    case "$fruit" in
        apple) echo "You chose apple" ;;
        banana) echo "You chose banana" ;;
        cherry) echo "You chose cherry" ;;
        quit) break ;;
        *) echo "Invalid choice" ;;
    esac
  done
1) apple
2) banana
3) cherry
4) quit
Choose a fruit: 2
You chose banana
Choose a fruit: 4
```
`select` creates an interactive numbered menu. The user's choice goes into `$fruit`. The `REPLY` variable contains the raw input number.

### Example 9: Process Substitution with While-Read
```bash
$ while IFS= read -r line; do
    echo "Got: $line"
  done < <(ls -la /tmp)
Got: total 24
Got: drwxrwxrwt 10 root root 4096 Jul 31 01:00 .
Got: drwxr-xr-x 18 root root 4096 Jul 31 00:00 ..
...
```
`< <(command)` feeds the output of a command into the loop without creating a subshell (unlike pipes). Variables set in the loop persist after the loop.

### Example 10: Iterating Over Lines in a Variable
```bash
$ data="line1
line2
line3"
$ while IFS= read -r line; do
    echo "Read: $line"
  done <<< "$data"
Read: line1
Read: line2
Read: line3
```
`<<<` (here-string) feeds a string into the loop's stdin. Useful for processing multi-line variables.

### Example 11: Rename Batch Files
```bash
$ for f in *.html; do
    mv "$f" "${f%.html}.htm"
  done
```
Parameter expansion `${f%.html}` removes the `.html` suffix. We then add `.htm`. Result: `page.html` becomes `page.htm`.

**What if:** A filename has spaces? `"$f"` is quoted, so spaces are preserved. The `${f%.html}` expansion is safe inside double quotes.

### Example 12: Infinite Loop with Break Condition
```bash
$ count=0
$ while true; do
    ((count++))
    echo "Iteration $count"
    [ "$count" -ge 5 ] && break
  done
Iteration 1
Iteration 2
Iteration 3
Iteration 4
Iteration 5
```
`while true` (or `while :`) creates an infinite loop. The `break` exits when a condition is met. Useful for "loop until valid input" patterns.

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Log rotation**: `for f in /var/log/*.log; do gzip "$f"; done`
- **Service restart**: `for svc in nginx php-fpm mysql; do systemctl restart "$svc"; done`
- **User processing**: `while IFS=: read -r user _ uid gid; do ... done < /etc/passwd`
- **Cron task**: Iterate over backup directories, remove old ones
- **Monitor loop**: `while true; do df -h /; sleep 60; done`

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Batch renaming**: `for f in *; do mv "$f" "${f//old/new}"; done`
- **File conversion**: `for f in *.png; do convert "$f" "${f%.png}.jpg"; done`
- **CSV processing**: `while IFS=, read -r name email; do sendmail "$email"; done < users.csv`
- **Bulk download**: `for url in $(cat urls.txt); do curl -O "$url"; done`
- **Test execution**: `for test in test_*.py; do python "$test"; done`

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Brute force**: Nested loops to generate password combinations
- **File enumeration**: `for f in /*; do [ -r "$f" ] && [ "$(stat -c%s "$f")" -gt 0 ] && echo "$f"; done`
- **Persistence scanning**: `for dir in /etc/cron* /var/spool/cron; do [ -w "$dir" ] && echo "writable: $dir"; done`
- **Log flooding**: `while true; do logger "Fake entry $RANDOM"; done` — DoS via log filling
- **Process killing**: `for pid in $(pgrep -u victim); do kill -9 "$pid"; done`

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **File integrity scan**: `for f in /bin/* /usr/bin/*; do sha256sum "$f"; done`
- **Permission audit**: `for dir in ${PATH//:/ }; do [ -w "$dir" ] && echo "WARNING: $dir writable"; done`
- **Log monitoring**: `tail -f /var/log/auth.log | while IFS= read -r line; do echo "$line" | grep -q "Failed" && alert; done`
- **Connection tracking**: `while read -r conn; do [ "$(echo "$conn" | cut -d' ' -f5)" -gt 100 ] && ban_ip; done < <(ss -tun)"
- **Backup verification**: `for backup in /backups/*.tar.gz; do tar -tzf "$backup" > /dev/null || echo "CORRUPT: $backup"; done`

## Memory Aids

- **`for var in list`**: Read it as English: "for each variable IN the list, do..."
- **`for (( ; ; ))`**: The three slots are "Start; Keep-going; Next-step."
- **`while` vs `until`**: "WHILE condition is TRUE, run" vs "run UNTIL condition is TRUE" (which means run WHILE condition is FALSE).
- **`break N`**: N breaks OUT N levels. Picture a stack of loops — break pops N levels off the stack.
- **`continue`**: Like skipping a song in a playlist — move to the next track.
- **`select`**: Think "let me SELECT from a menu." The variable name stores the SELECTion.
- **`IFS= read -r line`**: The three-part incantation: "I Frickin' Set nothing, Read raw, tell me the Line."

## Trap Vault (12 traps)

### Trap 1: `for f in $(ls *.txt)` — Parsing ls
**Problem:** Using `ls` in a for loop.
**Example:**
```bash
$ for f in $(ls *.txt); do echo "$f"; done
file1.txt
file2.txt  # Works for simple names, but:
$ touch "file with spaces.txt"
$ for f in $(ls *.txt); do echo "$f"; done
file
with
spaces.txt
```
**Why:** `ls` output isn't designed for parsing. Word splitting separates on spaces.
**Fix:** `for f in *.txt; do echo "$f"; done` — let the shell glob.

### Trap 2: Memory Explosion With Brace Expansion
**Problem:** `{1..1000000}` creates a million-element list.
**Example:**
```bash
$ for i in {1..1000000}; do :; done
# Uses ~7MB of memory just for the list!
```
**Fix:** Use C-style: `for ((i=1; i<=1000000; i++)); do :; done`

### Trap 3: While-Read Without IFS= and -r
**Problem:** Whitespace stripped, backslashes eaten.
**Example:**
```bash
$ echo "  hello  world" > /tmp/test
$ while read line; do echo ">$line<"; done < /tmp/test
>hello  world<  # Leading spaces stripped!
```
**Fix:** `while IFS= read -r line; do echo ">$line<"; done < /tmp/test`

### Trap 4: For Loop With Command Substitution
**Problem:** `for f in $(cat file)` splits on whitespace, expands globs.
**Example:**
```bash
$ cat > file << 'EOF'
foo
bar *
baz
EOF
$ for f in $(cat file); do echo "$f"; done
foo
bar
file       # * expanded to filenames!
baz
```
**Fix:** `while IFS= read -r f; do echo "$f"; done < file`

### Trap 5: Infinite Loop With while : and Missing Break
**Problem:** Loop never ends because condition never changes.
**Example:**
```bash
$ i=1
$ while [ "$i" -le 10 ]; do
    echo "$i"
    # Forgot: ((i++))
  done
# Infinite loop!
```
**Fix:** Always ensure the loop condition can change.

### Trap 6: Reading CSV With Leading Spaces
**Problem:** CSV with spaces after commas parsed wrong.
**Example:**
```bash
$ echo "name, email, age" | while IFS=, read -r name email age; do
    echo ">$name<"
  done
>name<  # fine, but:
> email<  # leading space!
```
**Fix:** `while IFS=', ' read -r name email age` — add space to IFS to strip it.

### Trap 7: For Loop Over Arguments Without Quotes
**Problem:** Arguments with spaces are split.
**Example:**
```bash
$ cat > args.sh << 'EOF'
for arg in $@; do
    echo "Arg: $arg"
done
EOF
$ ./args.sh "hello world" "foo bar"
Arg: hello
Arg: world
Arg: foo
Arg: bar
```
**Fix:** `for arg in "$@"; do echo "Arg: $arg"; done`

### Trap 8: Modify IFS Globally
**Problem:** Changing IFS affects everything after.
**Example:**
```bash
$ IFS=,
$ for word in a b c; do echo "$word"; done
a b c  # Only one iteration because IFS is comma!
```
**Fix:** Set IFS only for the command: `while IFS= read -r line; do ... done` or scope with `(IFS=,; ...)`.

### Trap 9: empty Globbing
**Problem:** Loop runs once with literal pattern.
**Example:**
```bash
$ for f in /tmp/*.nonexistent; do
    echo "Processing: $f"
  done
Processing: /tmp/*.nonexistent
```
**Why:** Without `nullglob`, unmatched globs stay literal.
**Fix:** `shopt -s nullglob` before the loop, or check `[ -f "$f" ]` inside.

### Trap 10: `break` in Sourced Script
**Problem:** `break` outside a loop is a syntax error.
**Example:**
```bash
$ source script.sh
script.sh: line 5: break: only meaningful in a `for', `while', or `until' loop
```
**Fix:** Don't use `break` in scripts that might be sourced at top level.

### Trap 11: `seq` is Not Portable
**Problem:** `seq` doesn't exist on all systems.
**Example:**
```bash
$ for i in $(seq 1 10); do echo "$i"; done
```
**Why:** `seq` is a GNU utility, not POSIX. BSD systems may not have it.
**Fix:** Use `{1..10}` (bash) or C-style `for ((i=1; i<=10; i++))`.

### Trap 12: While-Read in a Pipe Loses Variables
**Problem:** Variables set in a piped while loop are lost.
**Example:**
```bash
$ count=0
$ echo -e "a\nb\nc" | while IFS= read -r line; do
    ((count++))
  done
$ echo "$count"
0  # count is still 0!
```
**Why:** Pipe segments run in subshells. The `while` loop's `count` variable is in a subshell.
**Fix:** Use process substitution: `while ... done < <(echo -e "a\nb\nc")`

## See It In The Wild

### System init scripts
```bash
$ head -50 /etc/init.d/ssh | grep -E '(for|while|until)'
```

### Try this now:
```bash
# 1. Count files per directory
$ for dir in /etc /usr /var; do
    count=$(find "$dir" -type f 2>/dev/null | wc -l)
    echo "$dir: $count files"
  done

# 2. Show top 10 processes by memory (one-liner loop)
$ for pid in $(ps -eo pid --sort=-%mem | head -11 | tail -10); do
    ps -p "$pid" -o pid,comm,%mem --no-headers
  done

# 3. Generate numbered backups
$ for f in /etc/*.conf; do
    cp "$f" "$f.bak.$(date +%Y%m%d)"
  done

# 4. Interactive file selector
$ select f in *.txt; do
    echo "You chose: $f"
    wc -l "$f"
    break
  done
```

## Check Your Understanding (7 questions)

1. Why is `for f in $(ls *.txt)` bad practice? What should you use instead?

2. What does `IFS=` in `while IFS= read -r line` do? Why is it necessary?

3. What is the difference between `break` and `continue`?

4. How would you loop over numbers 1 to 100 without using `{1..100}` or `seq`?

5. Why does `for f in *.txt` pass `*.txt` literally when no .txt files exist?

6. What happens to variables set inside a `while` loop that's part of a pipeline?

7. What's the difference between `while` and `until`?
