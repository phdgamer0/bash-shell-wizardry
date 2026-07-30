# Lesson 14: Debugging — Tracing

## History & Origins

The `set -x` (xtrace) feature has been in bash since the early days, derived from the **Bourne shell's** `set -x` option. The "x" stands for "execution trace" — it shows each command as it executes. It's the single most important debugging tool in bash.

`set -v` (verbose) prints shell input lines as they're read — before expansion. It was borrowed from **sh's** `-v` flag. The "v" stands for "verbose."

The `PS4` variable (`PS` = "Prompt String," `4` = fourth prompt) customizes the xtrace prefix. Default is `'+ '`. The convention of `+` for trace output dates back to the original Bourne shell.

`bash -x script.sh` was an early innovation — debugging by running the script in trace mode from the very first line. Before this, you'd have to edit the script to add `set -x`.

The `DEBUG` trap (execute a command before every command) was added in **bash 2.0** (1996). It was originally designed for implementing debuggers — external tools could set `trap ... DEBUG` and intercept every command.

The naming: `PS4` follows `PS1`, `PS2`, `PS3` — the shell has four prompt strings for different contexts. `PS1` = interactive prompt, `PS2` = continuation prompt, `PS3` = select prompt, `PS4` = trace prefix.

## Syntax Reference

### Tracing Commands

| Command | Behaviour |
|---------|-----------|
| `set -x` | Enable command tracing (prints + command to stderr) |
| `set +x` | Disable command tracing |
| `set -v` | Enable verbose mode (prints input lines as read) |
| `set +v` | Disable verbose mode |
| `bash -x script.sh` | Run script with tracing from start |
| `bash -v script.sh` | Run script with verbose mode from start |
| `bash -xv script.sh` | Both trace and verbose |

### PS4 Customization

Default: `PS4='+ '`

| Component | Meaning | Example |
|-----------|---------|---------|
| `$LINENO` | Current line number | `PS4='+[$LINENO] '` |
| `${FUNCNAME[0]}` | Current function name | `PS4='+${FUNCNAME[0]}(): '` |
| `${BASH_SOURCE[0]}` | Current script/file | `PS4='+${BASH_SOURCE[0]}:${LINENO} '` |
| `${BASH_LINENO[0]}` | Line number in caller | `PS4='+${FUNCNAME[0]}:${BASH_LINENO[0]} '` |
| `${EPOCHREALTIME}` | Microsecond timestamp (bash 5.0+) | `PS4='+[${EPOCHREALTIME}] '` |
| `$SECONDS` | Seconds since shell start | `PS4='+[${SECONDS}s] '` |
| `$?` | Last exit code | `PS4='+[rc=$?] '` |
| `$_` | Last argument of previous command | `PS4='+[$_] '` |
| `\e[COLm` | ANSI color codes | `PS4='+\e[31m[$LINENO]\e[0m '` |

### DEBUG Trap

```
trap 'command' DEBUG
```

Fires **before** every command (including builtins, functions, and external commands). Does NOT fire for commands inside the trap handler itself.

| Use | Code |
|-----|------|
| Simple logging | `trap 'echo "# $BASH_COMMAND"' DEBUG` |
| Skip for PROMPT_COMMAND | `trap '[[ "$BASH_COMMAND" != "$PROMPT_COMMAND" ]] && echo "Doing: $BASH_COMMAND"' DEBUG` |
| Conditional break | `trap '[[ "$BASH_COMMAND" == "rm *" ]] && echo "STOP!"' DEBUG` |

### Other Debugging Builtins

| Builtin | Behaviour |
|---------|-----------|
| `caller` | Print line number and function of the current call stack |
| `caller 0` | Current position |
| `caller 1` | Caller's position |
| `caller N` | Nth caller |
| `type -a cmd` | Show all definitions/aliases of a command |
| `declare -f func` | Show function definition |
| `declare -x var` | Show variable attributes |

## Under the Hood

### How bash processes tracing

When `set -x` is enabled:
1. Before executing each simple command, bash prints the expanded command to stderr
2. The prefix is `PS4` (default `'+ '`)
3. Variables are expanded, globs are expanded — you see what bash ACTUALLY runs
4. The trace output goes to the original stderr, even if the command's stderr is redirected
5. Compound commands (loops, case, if) show the top-level command, not every internal token

**strace reveals:**
With `set -x`, bash calls `write(2, ...)` for each trace line. For `set -x; echo hello`:
```
write(2, "+ echo hello\n", 13)      = 13
write(1, "hello\n", 6)              = 6
```

The trace goes to fd 2 (stderr), the command output goes to fd 1 (stdout).

### PS4 expansion

`PS4` undergoes parameter expansion, command substitution, arithmetic expansion, and word splitting **each time** it's printed. This means:
- `PS4='+[$LINENO] '` evaluates `$LINENO` at each trace line (showing the current line)
- `PS4='+[$(date +%H:%M:%S.%N)] '` runs `date` every trace line (expensive!)
- Performance matters: avoid command substitution in `PS4` for tight loops

### Process implications

- Tracing slows execution — each command triggers an extra `write(2)` syscall
- For tight loops (thousands of iterations), tracing can make the script 10-100x slower
- Trace output and command stderr are interleaved — use different PS4 prefixes to distinguish
- `PS4` color codes work because they're just escape sequences written to stderr

## Core Examples (8-12 minimum)

### Example 1: Basic tracing

**Command:**
```bash
set -x
x=5
echo $((x + 3))
set +x
```

**Output (stderr shown with + prefix):**
```
+ x=5
+ echo 8
+ set +x
```
**stdout:**
```
8
```

**Step-by-step:**
1. `set -x` enables tracing
2. Bash prints `+ x=5` to stderr, then executes `x=5`
3. Bash computes `$((5 + 3))` → `8`, then prints `+ echo 8` to stderr
4. Bash executes `echo 8` — writes `8` to stdout
5. `set +x` prints `+ set +x` to stderr, then disables tracing

**Variations:**
- `PS4='+ '` — default, just a plus and space
- Trace output interleaves with command output on terminal (stderr and stdout both visible)

### Example 2: Custom PS4 with line numbers

**Command:**
```bash
PS4='+[\e[33m$LINENO\e[0m] '
set -x
y=10
echo "Line $LINENO: y=$y"
set +x
```

**Output (stderr, with line numbers in yellow):**
```
+[5] y=10
+[6] echo 'Line 6: y=10'
+[7] set +x
```
**stdout:**
```
Line 6: y=10
```

**Step-by-step:**
1. `PS4` is set to include `$LINENO` with ANSI yellow color codes
2. Each trace line shows the line number where the command appears
3. `$LINENO` is the current line number at the time of expansion

**Variations:**
- `PS4='+[$LINENO] '` — plain, no color
- `PS4='+\e[31m[$LINENO]\e[0m '` — red line numbers
- `PS4='+[${BASH_SOURCE##*/}:$LINENO] '` — include script filename

### Example 3: PS4 with function context

**Command:**
```bash
#!/bin/bash
PS4='+ ${FUNCNAME[0]:+${FUNCNAME[0]}():}${LINENO} '
outer() {
    inner "arg"
}
inner() {
    local x="$1"
    echo "x=$x"
}
set -x
outer
set +x
```

**Output (stderr):**
```
+ main:13 outer
+ outer:4 inner arg
+ inner:9 local x=arg
+ inner:10 echo x=arg
+ inner:10 set +x
```
**stdout:**
```
x=arg
```

**Step-by-step:**
1. `FUNCNAME[0]` gives the current function name
2. `${FUNCNAME[0]:+${FUNCNAME[0]}():}` — if FUNCNAME is set, print "function():"
3. At the top level, `FUNCNAME` is unset, so just the line number shows
4. Inside `outer()`, trace shows `outer:4`
5. Inside `inner()`, trace shows `inner:9`

**Variations:**
- `PS4='+[${BASH_SOURCE[0]##*/}:${LINENO}] ${FUNCNAME[0]:+${FUNCNAME[0]}():}'` — full context
- `PS4='+[${BASH_SOURCE##*/}:$LINENO] ${FUNCNAME:-main}(): '` — always show a name

### Example 4: Selective tracing

**Command:**
```bash
echo "Normal execution — no trace"
echo "Still normal"
set -x
echo "Traced execution — see the command"
x=42
echo "x=$x"
set +x
echo "Back to normal — no trace"
```

**Output (stderr lines with +):**
```
+ echo 'Traced execution — see the command'
+ x=42
+ echo x=42
+ set +x
```
**stdout:**
```
Normal execution — no trace
Still normal
Traced execution — see the command
x=42
Back to normal — no trace
```

**Step-by-step:**
1. First two commands run without tracing
2. `set -x` enables trace
3. Commands 3-5 show with `+` prefix on stderr
4. `set +x` disables trace
5. Last command runs without tracing

**Variations:**
- Wrap in a function: `trace_on() { set -x; }; trace_off() { set +x; }`
- Use `trap ... DEBUG` for even more granular control

### Example 5: bash -x on script

**Command:**
```bash
$ cat test.sh
#!/bin/bash
name="World"
echo "Hello, $name"
echo "Done."

$ bash -x test.sh
```

**Output (stderr):**
```
+ name=World
+ echo 'Hello, World'
+ echo Done.
```
**stdout:**
```
Hello, World
Done.
```

**Step-by-step:**
1. `bash -x test.sh` starts bash with xtrace enabled globally
2. Every command is traced from the first line of the script
3. Variable expansion is shown: `echo 'Hello, World'` (with quotes around the expanded string)
4. Useful for scripts YOU DIDN'T WRITE — no need to edit the file

**Variations:**
- `bash -x script.sh 2>&1 | less` — pipe trace to pager
- `bash -x script.sh 2>trace.log` — save trace to file
- `bash -x -v script.sh` — verbose + trace (extreme detail)

### Example 6: Verbose mode (set -v)

**Command:**
```bash
set -v
x=42
echo $x
y="hello world"
echo $y
set +v
```

**Output (stderr):**
```
x=42
echo $x
y="hello world"
echo $y
```
**stdout:**
```
42
hello world
```

**Step-by-step:**
1. `-v` prints each line AS IT'S READ by the shell (before expansion)
2. You see the raw input, not the expanded form
3. No `+` prefix — it's the actual source lines
4. Useful for seeing what the script looks like before variable substitution

**Variations:**
- Combine with `-x`: `set -xv` gives both raw input AND expanded command
- `bash -v script.sh` — verbose from start

### Example 7: DEBUG trap logging

**Command:**
```bash
#!/bin/bash
trap 'echo "[DEBUG] $(date +%T) - $BASH_COMMAND"' DEBUG
echo "Line 1"
x=10
echo "Line 3: $x"
```

**Output (stderr):**
```
[DEBUG] 12:00:01 - echo "Line 1"
[DEBUG] 12:00:01 - x=10
[DEBUG] 12:00:01 - echo "Line 3: $x"
Line 1
Line 3: 10
```

**Step-by-step:**
1. `trap '...' DEBUG` — runs the echo BEFORE every command
2. `$BASH_COMMAND` contains the exact command about to run
3. Timestamp shows when each command starts
4. Commands inside the trap handler are NOT debuggable (prevents infinite loop)

**Variations:**
- `trap '[[ "$BASH_COMMAND" != "$PROMPT_COMMAND" ]] && echo "Running: $BASH_COMMAND"' DEBUG` — skip PROMPT_COMMAND
- `trap 'read -p "Press enter to continue"' DEBUG` — step through script manually

### Example 8: caller builtin

**Command:**
```bash
#!/bin/bash
func1() {
    func2
}
func2() {
    echo "Call stack:"
    caller 0
    caller 1
    caller 2
}
func1
```

**Output:**
```
Call stack:
5 func2 ./script.sh
4 func1 ./script.sh
10 main ./script.sh
```

**Step-by-step:**
1. `caller 0` prints the caller of the current function: line 5 in func1
2. `caller 1` prints the caller of func1: line 10 in the main script
3. `caller 2` prints the caller of the main script: nothing (it's the top level)
4. Format: `line_number function_name source_file`

**Variations:**
- `caller` (no arg) = `caller 0`
- `caller | read line func file; echo "Called from $file:$line in $func()"` — parse caller output
- Useful in error handlers to show the call chain

### Example 9: Tracing with colored output

**Command:**
```bash
#!/bin/bash
PS4='+\e[1;31m[${BASH_SOURCE##*/}:${LINENO}]\e[0m \e[33m${FUNCNAME[0]:+${FUNCNAME[0]}(): }\e[0m'
set -x
hello() {
    echo "Hello, world!"
}
hello
set +x
```

**Output (stderr, with colors):**
```
+[script.sh:6] hello():
+[script.sh:4] hello(): echo 'Hello, world!'
+[script.sh:8] set +x
```
**stdout:**
```
Hello, world!
```

**Step-by-step:**
1. Red color for filename:line
2. Yellow color for function name
3. Reset at end so subsequent terminal output isn't colored
4. Makes trace output visually scannable

**Variations:**
- `PS4='+\e[36m[${EPOCHREALTIME}]\e[0m '` — cyan timestamp
- `PS4='+\e[31m[rc=$?]\e[0m '` — red exit code display

### Example 10: Tracing only failed commands

**Command:**
```bash
#!/bin/bash
trap 'echo "FAILED: $BASH_COMMAND (line $LINENO, exit $?)"' ERR
set -e
echo "This works"
false
echo "This never runs"
```

**Output:**
```
This works
FAILED: false (line 6, exit 1)
```

**Step-by-step:**
1. `trap ... ERR` — runs the handler when any command returns non-zero
2. `set -e` — exit on error (after the trap runs, the script exits)
3. `$BASH_COMMAND` shows the failed command
4. `$?` shows the exit code
5. The third `echo` never runs because `set -e` triggers after the trap

**Variations:**
- Combine with `set -x` for full trace + error highlight
- `trap 'echo "ERROR: $BASH_COMMAND at $LINENO"' ERR` — simpler

### Example 11: Selective tracing with functions

**Command:**
```bash
#!/bin/bash
debug_on() { set -x; }
debug_off() { set +x; }

compute() {
    local a=$1 b=$2
    echo $((a + b))
}

echo "Normal mode"
result=$(compute 5 3)
echo "Result: $result"

debug_on
result=$(compute 10 20)
debug_off

echo "Normal again"
```

**Output (stderr):**
```
++ compute 10 20
++ local a=10 b=20
++ echo 30
+ result=30
+ debug_off
```
**stdout:**
```
Normal mode
Result: 8
Result: 30
Normal again
```

**Step-by-step:**
1. `debug_on` / `debug_off` functions wrapping the traced section
2. Only the middle section is traced
3. Command substitution `$(compute ...)` shows a `+` for each nesting level (`++`)
4. Reduces trace noise — only debug what you need

**Variations:**
- `DEBUG_FUNCS=1; [[ "$DEBUG_FUNCS" ]] && set -x` — conditional debug from environment variable
- `[[ -o xtrace ]] && echo "Tracing is ON"` — check if tracing is active

### Example 12: PS4 with exit code tracking

**Command:**
```bash
#!/bin/bash
PS4='+[rc=$?] [${LINENO}] '
set -x
true
false
echo "After false"
set +x
```

**Output (stderr):**
```
+[rc=0] [5] true
+[rc=0] [6] false
+[rc=1] [7] echo After false
+[rc=0] [8] set +x
```
**stdout:**
```
After false
```

**Step-by-step:**
1. `rc=$?` in PS4 captures the exit code of the PREVIOUS command
2. After `true`, `rc=0`
3. After `false`, `rc=1` — you see the failure
4. After `echo`, `rc=0`
5. Instantly see which commands succeeded/failed

**Variations:**
- `PS4='+[${?}] '` — minimal rc display
- `PS4='+\e[1;3${?}m[$?]\e[0m '` — color-coded: green for 0, red for 1

## Real-World Use Cases

### FOR the OS

- **Debugging init scripts**: `bash -x /etc/init.d/nginx start` — trace the init system
- **Finding configuration errors**: `bash -x /etc/profile` — see what your profile does
- **Debugging cron jobs**: Redirect trace to log: `bash -x script.sh 2>/tmp/cron_debug.log`
- **Understanding completion scripts**: `bash -x /usr/share/bash-completion/bash_completion` (add `set -x`)

### WITH the OS

- **`bash -x script.sh 2>&1 | less`** — interactively page through trace output
- **`PS4='+ $EPOCHREALTIME '`** — microsecond timestamps for performance analysis
- **`trap 'logger -p user.debug "CMD: $BASH_COMMAND"' DEBUG`** — log every command to syslog
- **Custom PS4 with caller info**: `PS4='+[${BASH_SOURCE[0]##*/}:${LINENO}] '` — trace with file:line

### AGAINST THE OS (Security Perspective)

- **Tracing sensitive data**: If `set -x` is on when processing passwords, API keys, or tokens, they appear in clear text in stderr. This can leak to log files, terminal logs, or CI output.
- **Exposing internal logic**: `set -x` in a script reveals the full command flow — filenames, variables, directory structures. An attacker with log access can reverse-engineer your infrastructure.
- **`PS4` injection**: If an attacker can control `PS4` (via environment), they can inject ANSI escape sequences or commands via `$(...)` in PS4.
- **DEBUG trap for surveillance**: A malicious sourced script can set `trap 'send_to_attacker "$BASH_COMMAND"' DEBUG` to log every command you run.
- **Bash history vs set -x**: `set -x` shows commands in real-time, can't be turned off by the user, and goes to stderr which might be piped to an attacker-controlled log.

### FOR DEFENSE

- **Never run `set -x` in production with sensitive data** — use `set +x` around password/token handling
- **Redirect trace to a secure log file**: `bash -x script.sh 2>/var/log/debug/secure.log`
- **Use `set -x` selectively**: wrap only the problematic section, not the whole script
- **Sanitize PS4**: Don't include `$@` or other user-controlled variables in PS4 — they could contain escapes
- **Check for DEBUG trap**: `trap -p DEBUG` to see if something's watching
- **For security audits**: Use `bash -x` to verify a script isn't doing something unexpected

## Memory Aids

- **`set -x`** = "x" for "execution trace" (or "x-ray" — you see through the code)
- **`set +x`** = "cross it out" (disable)
- **`set -v`** = "v" for "verbose" (but really "view source")
- **`PS4`** = "Prompt String #4" — the fourth of the shell's prompt variables (PS1=prompt, PS2=continuation, PS3=select, PS4=trace)
- **`$LINENO`** = self-explanatory — the line number
- **`$FUNCNAME`** = "Function NAME" — an array of the call stack
- **`$BASH_COMMAND`** = "the BASH COMMAND about to execute"

## Trap Vault (8-12 traps)

### Trap 1: Trace output goes to stderr, not stdout

**Problem:** Piping script output to a file misses all trace info.

**Bad Example:**
```bash
bash -x script.sh > output.log
# trace info goes to terminal, not the file!
```

**Root Cause:** `set -x` writes to stderr (fd 2). Redirecting stdout (fd 1) doesn't capture it.

**Fix:**
```bash
bash -x script.sh 2>&1 | tee output.log   # trace + stdout
bash -x script.sh 2>trace.log             # trace to separate file
```

### Trap 2: PS4 with `$LINENO` gives wrong line in some contexts

**Problem:** `$LINENO` inside PS4 shows the PS4 assignment line, not the executed command line.

**Bad Example:**
```bash
PS4='+[$LINENO] '   # This line is line 1
set -x
echo "hello"        # This is line 3 — shows [1] instead of [3]
```

**Root Cause:** `$LINENO` in PS4 is evaluated... wait, actually this works correctly. Let me check.

Actually, `$LINENO` in `PS4` IS evaluated at each trace point and shows the line of the executed command. The above is NOT a trap — it works correctly.

Real trap: `$LINENO` evaluated ONCE at PS4 assignment (in some older bash versions).

**Fix:** Always test PS4 carefully. Modern bash (4.x+) evaluates PS4 at each trace line.

### Trap 3: Leaving set -x on in production

**Problem:** Script leaks all internal operations to logs.

**Bad Example:**
```bash
#!/bin/bash
set -x
# ... 100 lines of production code ...
# No set +x anywhere!
```

**Root Cause:** Developer turned it on for debugging and forgot to remove it.

**Fix:** Remove `set -x` from production scripts. Use `bash -x` from the command line instead.

### Trap 4: Nested command substitution shows extra +

**Problem:** Tracing shows confusing `++` prefixes.

**Bad Example:**
```bash
PS4='+[$LINENO] '
set -x
result=$(echo "hello")
set +x
```

**Output:**
```
+[5] result=$(echo "hello")
++[5] echo hello
+[5] result=hello
```

**Root Cause:** Each level of command substitution adds a `+`. The first `+` is the main command, `++` is the subshell inside `$( )`.

**Fix:** It's not a bug — it's a feature! The number of `+` indicates nesting depth. Use it to understand execution flow.

### Trap 5: PS4 colors persist after script ends

**Problem:** Terminal prompt is colored strangely after a traced script.

**Bad Example:**
```bash
PS4='+\e[31m'   # red on, never reset
set -x
echo "test"
set +x
# Terminal is now in red!
```

**Root Cause:** ANSI color codes are not automatically reset. If PS4 sets a color without `\e[0m`, the color leaks.

**Fix:**
```bash
PS4='+\e[31m[$LINENO]\e[0m '   # always reset at the end
```

### Trap 6: DEBUG trap with sleep/read causes delays

**Problem:** Script runs slowly with DEBUG trap active.

**Bad Example:**
```bash
trap 'echo "[DEBUG] Running: $BASH_COMMAND"' DEBUG
for i in {1..1000}; do
    : $((i * 2))
done
# Extra slow!
```

**Root Cause:** The DEBUG trap runs BEFORE EVERY COMMAND. In a loop with 1000 iterations, that's 1000 extra echo commands.

**Fix:** Use `set -x` instead of DEBUG trap for loop-heavy code:
```bash
set -x
for i in {1..1000}; do
    : $((i * 2))
done
set +x
```

### Trap 7: -v leaks sensitive data verbatim

**Problem:** Password variable in the script appears in verbose output.

**Bad Example:**
```bash
set -v
password="s3cr3t!"   # This line appears verbatim in verbose output
echo "Password set"
set +v
```

**Root Cause:** `-v` prints the raw source line, including any literal values.

**Fix:** Don't use `-v` with sensitive data. Use `-x` carefully (it shows expansions, not raw text).

### Trap 8: set -x in sourced scripts affects parent shell

**Problem:** Sourcing a script with `set -x` turns on tracing for your interactive shell.

**Bad Example:**
```bash
$ source ./debug_script.sh  # has set -x
+ echo "Running setup"
+ ...
# Now your interactive shell is tracing everything!
```

**Root Cause:** Source executes in the current shell. `set -x` remains enabled after the script finishes.

**Fix:** Debug script should always have `set +x` as the last line, or run with `bash -x` instead of sourcing.

### Trap 9: Redirecting stderr to capture trace also redirects command errors

**Problem:** You want trace, but error messages get mixed.

**Bad Example:**
```bash
bash -x script.sh 2>&1 | grep -v '^\+' > clean_output.txt
# This ALSO captures actual errors mixed with trace
```

**Root Cause:** Both trace and errors go to stderr. Separating them is hard.

**Fix:** Use different file descriptors or exec trickery:
```bash
exec 3>&2 2>trace.log    # original stderr goes to fd 3, stderr becomes trace.log
set -x
# ... commands ...
set +x
exec 2>&3 3>&-           # restore stderr
```

### Trap 10: Forgetting that `+x` has no effect in a subshell

**Problem:** `set +x` inside `$( )` doesn't disable tracing in the parent.

**Bad Example:**
```bash
set -x
result=$(set +x; echo "hello")  # This subshell has tracing OFF
echo "still traced?"            # YES — the set +x only affected the subshell
```

**Root Cause:** `$( )` runs in a subshell. `set +x` only affects that subshell.

**Fix:** Use `set +x` OUTSIDE the subshell:
```bash
set -x
result=$(echo "hello")  # subshell still traced, that's fine
set +x                 # now disable for parent
```

### Trap 11: Not quoting expanded values in trace output

**Problem:** Trace shows ambiguous values.

**Bad Example:**
```bash
x="hello   world"   # multiple spaces
set -x
echo $x            # trace shows: + echo hello world (spaces collapsed)
```

**Root Cause:** Without quoting, `$x` is word-split for the trace display. The actual command `echo` receives `hello   world`, but the trace shows the split version.

**Fix:** The trace shows bash's expanded view — quote your variables:
```bash
echo "$x"  # trace: + echo 'hello   world' (quoted in trace too)
```

## See It In The Wild

- **CI/CD pipelines**: Many CI scripts add `set -x` as the second line (after shebang) for full visibility
- **Docker build debugging**: `docker build --progress=plain` shows all build steps — bash -x equivalent
- **Initramfs/busybox scripts**: Heavy use of `set -x` since interactive debugging is impossible
- **Vagrant provisioning scripts**: `config.vm.provision "shell", path: "script.sh"` — often traced
- **Your own `.bashrc`**: Try `bash -x -l` (login shell) to debug profile issues

**Try this now:**

1. `bash -x -c 'echo $((2 + 2))'` — trace a one-liner
2. `PS4='+[${SECONDS}s] '; set -x; sleep 1; echo "done"; set +x` — see timing in trace
3. `PS4='+\e[31m[$LINENO]\e[0m '; set -x; for i in 1 2 3; do echo $i; done; set +x` — colored trace
4. `bash -x <<< 'echo hello; echo world'` — trace a heredoc script

## Check Your Understanding (5-7 questions)

1. How do you enable and disable tracing for only a specific section of code?
2. What's the difference between `-x` and `-v`?
3. How do you customize the trace prefix to show line numbers?
4. Where does trace output go (stdout or stderr)?
5. Why might brace expansion not work inside a `for` loop — and how would `-x` reveal this?
6. What does `$BASH_COMMAND` contain in a DEBUG trap?
7. How can you tell the difference between a command's own error output and the trace output?

---
*"`set -x` is the bash equivalent of an X-ray machine: it shows you the bones beneath the skin."*
