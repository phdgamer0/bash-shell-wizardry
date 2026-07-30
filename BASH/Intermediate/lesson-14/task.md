# Task 14: Debugging — Tracing

You'll debug a deliberately broken script, create a custom tracing helper, and learn selective debugging techniques. These are the skills that separate "it works on my machine" from "I know exactly what's happening."

## Sub-tasks

### 1. Debug the broken script — `broken.sh` / `fixed.sh`

Here's a broken script. Save it as `broken.sh`, use `bash -x` to find the bugs, and write the fixed version as `fixed.sh`.

**broken.sh:**
```bash
#!/bin/bash
# Counts from 1 to N and computes sum — has bugs
n=$1
sum=0
for i in {1..$n}; do
    sum=$((sum + i))
done
echo "Sum from 1 to $n = $sum"
```

**Bugs to find and fix:**
1. Brace expansion happens BEFORE variable expansion — `{1..$n}` doesn't work
2. No argument validation — if no arg, `$n` is empty, `{1..}` is invalid
3. Missing `-u` strict mode — unset `$n` would silently break
4. No error if `$1` is not a number

**Requirements for fixed.sh:**
- Use `for ((i=1; i<=n; i++))` or `seq` instead of brace expansion
- Validate that argument exists and is a positive integer
- Handle the case `n=0` gracefully (sum is 0)
- Use `bash -x` during development to verify each fix
- Include proper exit codes (0 for success, 1 for error)

**Expected:**
```bash
$ bash -x broken.sh 5
+ n=5
+ sum=0
+ for i in '{1..$n}'
+ sum=0    ← Bug! Brace expansion didn't happen, $n treated as literal
+ for i in '{1..$n}'
...
Sum from 1 to 5 = 0

$ ./fixed.sh 5
Sum from 1 to 5 = 15

$ ./fixed.sh
Usage: fixed.sh <positive-integer>
Exit code: 1

$ ./fixed.sh 0
Sum from 1 to 0 = 0

$ ./fixed.sh -5
Usage: fixed.sh <positive-integer>
Exit code: 1
```

### 2. Custom PS4 trace function — `trace_helper.sh`

Write a script that demonstrates a rich, customizable trace environment.

**Requirements:**
- Set `PS4` to include ALL of the following:
  - Timestamp (HH:MM:SS) or epoch with milliseconds
  - Line number
  - Current function name (or "main" at top level)
  - Source filename
  - Exit code of the previous command
  - Color: timestamps in cyan, line numbers in yellow, commands in default
- Define functions `trace_on()` and `trace_off()` that enable/disable tracing with the custom PS4
- Demonstrate with a function that does a calculation and a loop
- Show the colored output (use `printf` for preview if terminal doesn't support it)

**Expected output (conceptually):**
```bash
$ ./trace_helper.sh
Normal execution (no trace)...
+ [12:30:01] [trace_helper.sh:18 main()] rc=0: echo 'Starting traced section'
+ [12:30:01] [trace_helper.sh:19 main()] rc=0: x=10
+ [12:30:01] [trace_helper.sh:20 main()] rc=0: y=20
+ [12:30:01] [trace_helper.sh:21 main()] rc=0: result=30
+ [12:30:01] [trace_helper.sh:22 main()] rc=0: echo 'Result: 30'
Back to normal execution...
```

### 3. Selective tracing — `selective_trace.sh`

Write a script that demonstrates selective tracing, a DEBUG trap with logging, and error highlighting.

**Requirements:**
- Run some commands WITHOUT tracing (normal mode)
- Enable tracing for a critical section (a loop with computation)
- Disable tracing after the critical section
- Use a DEBUG trap that logs each command with a timestamp to a log file
- Use a `trap ERR` handler that prints the failed command and line number
- Show the difference between the three modes

**Expected:**
```bash
$ ./selective_trace.sh
=== Phase 1: Normal (no trace) ===
Setting up...

=== Phase 2: Critical section (traced) ===
+ loop_compute 5
+ local n=5
+ for ((i=1; i<=n; i++))
+ result=1
+ for ((i=1; i<=n; i++))
+ result=3
+ for ((i=1; i<=n; i++))
+ result=6
+ for ((i=1; i<=n; i++))
+ result=10
+ for ((i=1; i<=n; i++))
+ result=15
+ echo 'Result: 15'

=== Phase 3: Normal again ===
Cleaning up...

=== DEBUG Log (written to debug.log) ===
[12:30:01] DEBUG: echo "Setting up..."
[12:30:01] DEBUG: echo "Starting critical section"
[12:30:01] DEBUG: loop_compute 5
[12:30:01] DEBUG: result=15
...

$ cat debug.log
[12:30:01] DEBUG: echo "Setting up..."
[12:30:01] DEBUG: echo "Starting critical section"
[12:30:01] DEBUG: loop_compute 5
[12:30:01] DEBUG: local n=5
[12:30:01] DEBUG: for ((i=1; i<=n; i++))
...
```

### 4. Error locator — `error_trace.sh`

Write a script with a `trap ERR` handler that reports the exact location of any error.

**Requirements:**
- Set a `trap ERR` that calls a function `error_handler()`
- `error_handler()` prints:
  - The command that failed (`$BASH_COMMAND`)
  - The line number (`$LINENO`)
  - The function name and caller (using `caller`)
  - The exit code (`$?`)
- Then script exits with the same exit code
- Include a function that may fail, and a buggy command at the top level
- Don't use `set -e` — let ERR trap catch everything

**Expected:**
```bash
$ ./error_trace.sh
Processing...
ERROR: Command 'false' at line 25 in function 'process_data()'
  Called from line 32 in main script
  Exit code: 1
Script aborted.

$ ./error_trace.sh  # another path
ERROR: Command 'ls /nonexistent' at line 40 in main script
  Exit code: 2
Script aborted.
```

## Solution Approaches

### Approach A: Bug-hunter
1. `broken.sh` first — run `bash -x`, identify the brace expansion bug
2. `error_trace.sh` — build the error handler
3. `trace_helper.sh` — customize PS4
4. `selective_trace.sh` — put it all together

### Approach B: Tool-focused
1. `trace_helper.sh` — create your debugging toolbox first
2. `error_trace.sh` — add error detection
3. `selective_trace.sh` — selective tracing
4. `broken.sh` — apply tools to find bugs

### Approach C: Education sequence
1. `broken.sh` — see the problem
2. `selective_trace.sh` — learn the tools
3. `trace_helper.sh` — customize the tools
4. `error_trace.sh` — automate error detection

<details>
<summary>Hint 1: Fixing brace expansion bug</summary>

The bug: `{1..$n}` — brace expansion occurs BEFORE parameter expansion. By the time bash expands `$n`, it's too late.

Fix option 1 (C-style for loop):
```bash
for ((i=1; i<=n; i++)); do
    sum=$((sum + i))
done
```

Fix option 2 (seq command):
```bash
for i in $(seq 1 "$n"); do
    sum=$((sum + i))
done
```

Fix option 3 (eval — dangerous, not recommended):
```bash
for i in $(eval "echo {1..$n}"); do
```
But `eval` is evil for this.
</details>

<details>
<summary>Hint 2: Custom PS4 with all features</summary>

```bash
PS4='+\e[36m[$(date +%T)]\e[0m '
PS4+='\e[33m[${BASH_SOURCE##*/}:${LINENO}]\e[0m '
PS4+='\e[35m${FUNCNAME[0]:+${FUNCNAME[0]}(): }\e[0m'
PS4+='\e[31m[rc=$?]\e[0m '

# Just having $? in PS4 is tricky — it shows the rc of the previous trace, not the command.
# Better approach:
PS4='+\e[36m[$(date +%T)]\e[0m '
PS4+='\e[33m[${BASH_SOURCE##*/}:${LINENO}]\e[0m '
PS4+='\e[35m${FUNCNAME[0]:+${FUNCNAME[0]}(): }\e[0m'
```

Note: `$?` in PS4 gives the exit code of the command that wrote the trace output, not the command being traced. Use with care.
</details>

<details>
<summary>Hint 3: Selective tracing pattern</summary>

```bash
#!/bin/bash
PS4='+[$LINENO] '

trace_on()  { set -x; }
trace_off() { set +x; }

echo "Normal section"
trace_on
# Critical section
for i in {1..3}; do
    echo "Traced: $i"
done
trace_off

echo "Normal again"
```
</details>

<details>
<summary>Hint 4: trap ERR with caller info</summary>

```bash
error_handler() {
    local rc=$?
    local cmd="$BASH_COMMAND"
    local line=$1
    local func="${FUNCNAME[1]:+in function ${FUNCNAME[1]}()}"
    local caller_info=$(caller 0)
    echo "ERROR: Command '$cmd' at line $line $func" >&2
    echo "  Caller: $caller_info" >&2
    echo "  Exit code: $rc" >&2
    exit $rc
}

trap 'error_handler $LINENO' ERR

# Test
false   # triggers ERR
echo "This doesn't run"
```
</details>

<details>
<summary>Hint 5: DEBUG trap logging to file</summary>

```bash
DEBUG_LOG="debug.log"
: > "$DEBUG_LOG"   # clear log

trap 'echo "[$(date +%T)] DEBUG: $BASH_COMMAND" >> "$DEBUG_LOG"' DEBUG

echo "This gets logged"
x=42
echo "x=$x"
```
</details>

<details>
<summary>Hint 6: bash -x to verify brace expansion bug</summary>

Run this to see the bug in action:
```bash
n=5
echo {1..$n}
# Output: {1..5}   ← literal, not expanded!

# vs with eval:
eval echo {1..$n}
# Output: 1 2 3 4 5

# vs the right way:
for ((i=1; i<=n; i++)); do echo $i; done
```
</details>

## Bonus Challenges

1. **Trace-with-pager**: Write a function that runs a script with `bash -x`, pipes to `less -R` (to preserve colors), and highlights error lines in red.
2. **Conditional breakpoints**: In the DEBUG trap, check if `$BASH_COMMAND` matches a pattern before logging. Implement `breakpoint "pattern"` and `continue`.
3. **Trace-level control**: Implement `TRACE_LEVEL={0,1,2}` environment variable that controls detail level — 0=no trace, 1=normal, 2=with variable values.
4. **Stack trace exporter**: Write a function `dump_stack()` that prints the full call stack using `caller` in a loop — useful for error handlers.
5. **Animated trace**: Use `\r` and short sleep in PS4 to create an "executing..." animation (impractical but educational).

## Expected Output Summary

```
fixed.sh:
  Fixes broken.sh's brace expansion bug
  Proper input validation
  Correct sum calculation

trace_helper.sh:
  Custom PS4 with timestamp, line, function, file, color
  trace_on() / trace_off() functions
  Demonstrated with calculation

selective_trace.sh:
  Three phases: normal → traced → normal
  DEBUG trap writing to debug.log
  ERR trap for error detection

error_trace.sh:
  trap ERR with BASH_COMMAND, LINENO, caller
  Function context in error messages
  Proper exit code preservation
```

## Self-Check

- How do you enable and disable tracing for only a specific section of code?
- What's the difference between `-x` and `-v`?
- How do you customize the trace prefix to show line numbers?
- Where does trace output go (stdout or stderr)?
- Why does `{1..$n}` not work in a for loop?
- What information does `$BASH_COMMAND` contain in a DEBUG trap?
- How would you make PS4 show different colors for different levels of trace?
- What's the risk of leaving `set -x` enabled in a production script?
