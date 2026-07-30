# Lesson 7: Performance & Benchmarking

## History & Origins

Shell performance was not a primary concern in the 1970s — Unix shells ran single-user on PDP-11s with 256KB of RAM. The shell's job was to launch other programs, and the overhead of forking was negligible compared to program execution time.

As scripts grew to process millions of lines (log analysis, data processing, build systems), performance became critical. The key insight: **every external command forks a process**, and **forking is expensive** (~5-50 microseconds per fork on modern hardware). A loop that forks 100,000 times adds seconds to runtime.

The `time` keyword (bash builtin) has been part of the Bourne shell since the beginning. The `TIMEFORMAT` variable (bash 2.0+) allows customization. The `hash` table (also from the Bourne shell) optimizes command lookup by caching full paths.

## Syntax Reference

### Timing Commands
```
time command [args]          # Time a command (bash builtin)
time { command; }            # Time a compound command (braces required)
```

### TIMEFORMAT Variable
```
TIMEFORMAT='%R'              # Real (wall clock) time in seconds
TIMEFORMAT='%U'              # User CPU time
TIMEFORMAT='%S'              # System CPU time
TIMEFORMAT='%P'              # CPU percentage (user+sys)/real
TIMEFORMAT='%3R'             # Real time with 3 decimal places
TIMEFORMAT=$'\nreal\t%3R\nuser\t%3U\nsys\t%3S'  # Multi-line format
```

### Command Lookup Cache
```
hash                         # Display hash table
hash -l                      # List in reusable format
hash -r                      # Clear hash table
hash -d command              # Remove specific entry
hash command                 # Look up and hash a command
```

### Builtin vs External Detection
```
type -a command              # Show all locations
type -t command              # Show type only (builtin, file, function, etc.)
command -v command           # Show path (or "builtin" if builtin)
enable -n command            # Disable builtin (forces external)
enable command               # Re-enable builtin
```

## Under the Hood

### The Cost of Forking

When bash runs an external command, the sequence is:

1. `fork()` — create a child process (copies parent's address space)
2. In child: `execve()` — replace child's memory with new program
3. In parent: `waitpid()` — wait for child to finish

Each step has costs:
- `fork()`: ~5-10 microseconds (copy page tables, duplicate file descriptors)
- `execve()`: ~5-20 microseconds (load binary, link libraries, initialize)
- `waitpid()`: ~1-2 microseconds (context switch, reaping)

Total: ~10-30 microseconds per external command. In a loop of 100,000 iterations, that's 1-3 seconds just for process overhead.

Builtins bypass ALL of this — zero fork, zero exec, zero wait.

```
External command:              Builtin:
┌─────────────────────┐       ┌─────────────────────┐
│ fork()              │       │ call internal        │
│   ┌─────────────┐   │       │ function directly    │
│   │ execve()    │   │       │                     │
│   │ (run cmd)  │   │       │ No fork, no exec    │
│   │ _exit()    │   │       │ No new process      │
│   └─────────────┘   │       └─────────────────────┘
│ waitpid()           │
└─────────────────────┘
```

### The Hash Table

Bash maintains a hash table mapping command names to full paths. When you type `ls`, bash:

1. Checks the hash table for `ls`
2. If found: uses cached path (`/usr/bin/ls`)
3. If not found: searches `$PATH` directories, caches result

The hash lookup is O(1). PATH search is O(n) where n = number of PATH components × number of directories × number of files per directory.

```
hash -l output:
builtin echo
/usr/bin/ls
/usr/bin/cat
/usr/bin/grep
```

### Memory Layout Implications

**Fork and COW (Copy on Write):** Modern Linux uses copy-on-write for fork. The child's address space points to the parent's physical pages until either writes to them. This makes fork fast (~5µs) regardless of parent memory size. However, `execve()` must then tear down the address space and load a new program.

**Page cache effects:** First run of a command reads its binary from disk into the page cache. Subsequent runs find it already cached. This is why benchmarks should include a warm-up run.

### TIMEFORMAT Internals

When `time` is used, bash:
1. Records the start time (`clock_gettime(CLOCK_MONOTONIC)` for real time, `getrusage(RUSAGE_CHILDREN)` for CPU times)
2. Runs the command
3. After command completes, records end times
4. Formats the output according to `TIMEFORMAT`

The resolution depends on the kernel's timer frequency (typically 1ms = 1000Hz on modern kernels, but `clock_gettime` provides nanosecond resolution).

## Core Examples

### Example 1: Using time (Builtin vs External)

```bash
$ # Builtin time (bash keyword):
$ time sleep 1

real    0m1.002s
user    0m0.000s
sys     0m0.001s
```

**Anatomy:**
- `real` = wall clock time (actual elapsed time)
- `user` = CPU time spent in user mode in the COMMAND (not in bash)
- `sys` = CPU time spent in kernel mode for the command

**What if variations:**
- `/usr/bin/time sleep 1` — external `time` from GNU coreutils, has `-v` for verbose output
- `time (sleep 1; sleep 1)` — times the entire subshell, not each command

### Example 2: Custom TIMEFORMAT

```bash
$ TIMEFORMAT='Real: %3R  User: %3U  Sys: %3S'
$ time sleep 1
Real: 1.002  User: 0.000  Sys: 0.001
```

**What if variations:**
- `TIMEFORMAT='%P%%'` — shows CPU percentage (useful for detecting I/O-bound processes)
- `TIMEFORMAT=$'\nreal\t%3R'` — tabs and newlines for readable output

### Example 3: Subshell Overhead

```bash
$ # Without subshell (FAST)
$ time for i in {1..1000}; do :; done
real 0m0.008s

$ # With subshell (SLOW — 5-10x)
$ time for i in {1..1000}; do (:); done
real 0m0.042s
```

**Anatomy:**
- `(:)` creates a subshell — bash forks, runs `:`, exits, parent waits
- 1000 subshells = 1000 forks/wait cycles
- `:` (no-op builtin) in a subshell is still costly due to fork

**Why subshells are expensive:** Each `()` requires `fork()` + `waitpid()`. That's the same overhead as running an external command, even if the subshell does nothing.

### Example 4: Builtin vs External

```bash
$ # Builtin echo (FAST)
$ time for i in {1..10000}; do echo "$i" >/dev/null; done
real 0m0.045s

$ # External echo (SLOW — 20-30x slower)
$ time for i in {1..10000}; do /bin/echo "$i" >/dev/null; done
real 1m1.234s
```

**Anatomy:**
- `/bin/echo` forks 10,000 times = 10,000 processes created and destroyed
- builtin `echo` calls bash's internal `echo` function — zero overhead
- More than just fork: also exec, dynamic linking, initialization

### Example 5: printf vs echo

```bash
$ time for i in {1..10000}; do printf '%s\n' "$i" >/dev/null; done
real 0m0.038s
$ time for i in {1..10000}; do echo "$i" >/dev/null; done
real 0m0.045s
```

`printf` is slightly faster and more portable. Both are builtins.

### Example 6: The Hash Table Effect

```bash
$ hash -r  # Clear cache
$ time ls >/dev/null   # First run — hash miss
real 0m0.003s
$ time ls >/dev/null   # Second run — hash hit
real 0m0.001s
$ time ls >/dev/null   # Third run — disk cache + hash hit
real 0m0.001s
```

**Anatomy:**
- First run: bash searches PATH (hash miss), kernel reads `ls` from disk (page cache miss)
- Second run: hash hit, page cache hit
- The difference is mainly page cache, not hash lookup (hash is microseconds, disk is milliseconds)

### Example 7: type -a to Understand Priority

```bash
$ type -a echo
echo is a shell builtin
echo is /usr/bin/echo
$ type -a ls
ls is aliased to `ls --color=auto'
ls is /usr/bin/ls
```

**Resolution order:** aliases → functions → builtins → hash table → PATH search

### Example 8: Redirecting Loop Output vs Per-Command Redirection

```bash
$ # SLOW: Redirect in each iteration
$ for i in {1..10000}; do echo "$i" >> /tmp/out.txt; done

$ # FAST: Redirect the entire loop
$ for i in {1..10000}; do echo "$i"; done > /tmp/out.txt
```

**Why:** Each `>>` opens, seeks to end, writes, closes. Loop redirection opens once, writes, closes once.

### Example 9: seq vs Brace Expansion

```bash
$ # SLOW (external command):
$ time for i in $(seq 1 1000); do :; done
$ # FAST (brace expansion — builtin):
$ time for i in {1..1000}; do :; done
```

`$(seq 1 1000)` forks `seq` and captures its output. `{1..1000}` is expanded by bash internally.

### Example 10: cat vs Redirect

```bash
$ # SLOW (useless cat):
$ n=$(cat file | wc -l)
$ # FAST (redirect):
$ n=$(wc -l < file)
```

`cat file | wc -l` forks TWO processes and a pipe. `wc -l < file` is a simple redirect.

### Example 11: mapfile vs while read

```bash
$ # mapfile (fast for small-to-medium files):
$ time mapfile -t lines < /tmp/testfile.txt

$ # while read (slower but constant memory):
$ time while IFS= read -r line; do :; done < /tmp/testfile.txt
```

`mapfile` reads the entire file into an array in one operation. `while read` reads one line at a time with 10,000+ read() calls.

### Example 12: Checking Command Type

```bash
$ for cmd in echo printf find grep sed awk; do
>   type -t "$cmd"
> done
builtin
builtin
file
file
file
file
```

Only `echo` and `printf` are builtins. `find`, `grep`, `sed`, `awk` are external — every call forks.

### Example 13: Function Call Overhead

```bash
$ myfunc() { :; }
$ time for i in {1..10000}; do myfunc; done
real 0m0.012s
$ time for i in {1..10000}; do :; done
real 0m0.008s
```

Function calls add very little overhead (a few microseconds per call). They're much cheaper than subshells or external commands.

### Example 14: Pipeline Component Cost

```bash
$ # Single process:
$ time grep pattern bigfile.txt > /dev/null
real 0m0.100s

$ # Three-process pipeline:
$ time cat bigfile.txt | grep pattern | wc -l > /dev/null
real 0m0.130s
```

Each pipeline stage adds process creation overhead. But for I/O-bound tasks, the overhead is negligible compared to I/O time.

### Example 15: HERE String vs Echo Pipe

```bash
$ # HERE string (no fork):
$ grep foo <<< "foobar"
$ # Echo pipe (fork + pipe):
$ echo "foobar" | grep foo
```

`<<<` is a bash builtin — no fork. `echo | grep` forks two processes.

## Real-World Use Cases

### FOR the OS
- **Benchmarking tools:** `time` for command timing, `TIMEFORMAT` for custom output
- **Build systems:** Make uses shell timestamps for incremental builds
- **Cron job monitoring:** Wrap cron commands in `time` to log execution duration

### WITH the OS
- **Log processing:** Millions of lines → subshell avoidance matters
- **Data transform scripts:** `for i in $(cat file)` vs `while read` — the right choice saves hours
- **CI/CD pipelines:** Faster scripts = faster builds = faster feedback

### AGAINST the OS (defense perspective)
- **Timing attacks:** Measuring execution time can leak information (e.g., password comparison timing)
- **Resource exhaustion:** Inefficient scripts can be exploited for denial of service (fork bombs)
- **Hash table poisoning:** If an attacker controls PATH, they can replace hashed commands

### FOR DEFENSE
- **Hash table monitoring:** Changes to the hash table can indicate PATH manipulation
- **Benchmark as monitoring:** Track script execution times — sudden increases may indicate resource issues or compromise
- **Fork bomb detection:** Monitor process creation rate with `watch -n 1 'ps -eLf | wc -l'`

## Memory Aids

- **"Builtins are free, externals cost a fork"** — Builtins are zero-overhead; external commands spawn processes.
- **"Redirect the loop, not the line"** — Move redirection outside the loop for massive speedups.
- **"Hash caches paths, but not speed"** — The hash table saves path lookup (microseconds), but not execution time (milliseconds+).
- **"seq forks, braces don't"** — `$(seq N)` forks; `{1..N}` is built-in.
- **"cat | is always wrong"** — `cat file | cmd` forks cat needlessly. Use `cmd < file`.
- **"Read the man page for your tools"** — Many tools have performance flags (`grep -F` for fixed strings, `sort -S` for buffer size).

## Trap Vault

1. **`time` in a pipeline times the whole pipeline:** `time ls | wc -l` times both `ls` AND `wc`, not just `ls`. Use `time ls >/dev/null` or `{ time ls; } | wc -l` for per-component timing.

2. **Hash table is not persistent:** `hash -r` in a script clears it. Each new shell starts with an empty hash table. The first command in any script always has a cache miss.

3. **`TIMEFORMAT` only affects the builtin `time`:** The external `/usr/bin/time` ignores `TIMEFORMAT`. It has its own format string via `-f` or `TIME` environment variable.

4. **`/usr/bin/time` vs `time`:** The external `time` can measure memory (`%M`), I/O (`%I`, `%O`), and signals (`%k`). The builtin only measures real/user/sys CPU time.

5. **First-run benchmarks are misleading:** Disk cache, CPU cache, and branch predictor warm-up make first runs significantly slower. Always do a warm-up run.

6. **`for i in $(cat file)` is a double mistake:** It forks `cat` AND splits the output on IFS (whitespace). Use `while read -r line; do ... done < file`.

7. **`$()` in a loop forks every iteration:** `for i in ...; do echo $(cmd); done` forks for each `$()`. Use `result=$(cmd); for i in ...; do echo "$result"; done`.

8. **Background processes don't appear in time's measurement:** `time sleep 1 &` — the `&` makes it a background job; time measures only until the bg job starts. Use `wait` inside a timed block.

9. **`type -a` shows resolution order but can be misleading:** A function may mask a builtin which masks an external. `type -a` shows all, but only the first one is used.

10. **Redirect timing matters:** `time cat file > /dev/null` measures cat. `time cat file > /dev/null 2>&1` is the same. But `time { cat file > /dev/null; } 2>&1` redirects TIME's stderr too.

11. **`{1..1000000}` allocates memory:** Brace expansion creates ALL elements in memory before the loop starts. For 1M items, this uses ~50MB of memory. Use `for ((i=1; i<=1000000; i++))` for large ranges.

12. **`printf` vs `echo` portability:** `echo` behavior varies across shells (`-n`, `-e` flags). `printf` is consistent. Performance is similar.

13. **`[[ ]]` vs `[ ]` performance:** `[[ ]]` is a bash builtin (keyword) with no fork. `[ ]` (test) is either a builtin or `/usr/bin/[` depending on version. In modern bash, both are builtins. `[[ ]]` has fewer parsing steps.

14. **`$(( ))` vs `expr`:** `$(( ))` is a builtin (no fork). `expr` is an external command (fork). For arithmetic, always use `$(( ))`.

15. **`let` vs `(( ))`:** Both are builtins. `let i++` and `((i++))` are equivalent performance-wise. `(( ))` is more readable for complex expressions.

## See It In The Wild

```bash
# Check your bash version's builtins
$ enable -p | head -20

# See which commands are hashed
$ hash
hits    command
   3    /usr/bin/ls
   1    /usr/bin/cat

# Use /usr/bin/time for detailed stats
$ /usr/bin/time -v sleep 1
    Command being timed: "sleep 1"
    User time (seconds): 0.00
    System time (seconds): 0.00
    Percent of CPU this job got: 0%
    Elapsed (wall clock) time (h:mm:ss or m:ss): 0:01.00
    Major (requiring I/O) page faults: 0
    Minor (reclaiming a frame) page faults: 0
    Voluntary context switches: 2
    Involuntary context switches: 0
    Swaps: 0
    File system inputs: 0
    File system outputs: 0
    Socket messages sent: 0
    Socket messages received: 0
    Signals delivered: 0
    Page size (bytes): 4096
    Exit status: 0

# Benchmark with hyperfine (if installed)
$ hyperfine 'sleep 1' 'sleep 2'
```

## Check Your Understanding

1. Why is `echo "$var"` (builtin) faster than `/bin/echo "$var"`? What are the kernel-level operations involved?

2. What is the difference between `time` as a bash builtin and `/usr/bin/time`?

3. Why does `cat file | command` perform worse than `command < file`?

4. What does the `hash` table store? How does it speed up command execution?

5. Why does the first run of a command in a script often take longer than subsequent runs?

6. How does `for i in $(seq 1000)` differ from `for i in {1..1000}` in terms of process creation?

7. What is the overhead of a subshell `()` in terms of system calls?

8. Why does `printf` make a better choice than `echo` for consistent performance across systems?

9. What information does `/usr/bin/time -v` provide that the bash `time` builtin doesn't?

10. How would you measure the real, user, and system time of a pipeline of three commands?
