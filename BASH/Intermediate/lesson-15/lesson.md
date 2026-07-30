# Lesson 15: Debugging — Strict Mode

## History & Origins

"Strict mode" isn't a single bash feature — it's a **best-practice combination** of three shell options that emerged in the early 2010s. The phrase was popularized by blogs like "Unofficial Bash Strict Mode" (2011) by Aaron Toponce and David Pashley's "Better Bash Scripting" articles.

The three components:

- **`set -e`** (errexit): Existed since the **Bourne shell** (1977). It tells the shell to exit if a command returns non-zero. The name "errexit" = "error exit."
- **`set -u`** (nounset): Also from the Bourne shell. It treats references to unset variables as an error. "nounset" = "no unset."
- **`set -o pipefail`**: Added in **bash 3.0** (2004). Without this, a pipeline's exit code is ONLY the exit code of the LAST command. `pipefail` makes it fail if ANY command in the pipe fails.

The combination `set -euo pipefail` became the "holy trinity" of bash reliability. The `-o pipefail` is the odd one out (it's a `set -o` option, not a `-o` letter), so it can't be combined as `set -euop pipefail` — it must be separate.

The philosophy is called **"fail fast"** or **"fail early"** — catch errors as close to their source as possible, rather than letting them cascade into harder-to-diagnose problems.

## Syntax Reference

### The Holy Trinity

```bash
set -euo pipefail
```

| Option | Short | Effect | Exceptions |
|--------|-------|--------|------------|
| `errexit` | `-e` | Exit on command failure (non-zero return) | Disabled inside `if`, `while`, `until`, `||`, `&&`, `!` |
| `nounset` | `-u` | Error on unset variable reference | `${var:-default}` works; using `$@` when no args is OK |
| `pipefail` | `-o pipefail` | Pipeline fails if any stage fails | Only the exit code of the last command is affected without this |

### Individual flags

| Command | Behaviour |
|---------|-----------|
| `set -e` | Enable exit on error |
| `set +e` | Disable exit on error |
| `set -u` | Enable error on unset variables |
| `set +u` | Allow unset variables (default) |
| `set -o pipefail` | Enable pipeline failure detection |
| `set +o pipefail` | Disable pipefail (default) |
| `set -E` | ERR traps propagate to functions/subsets (bash 4.1+) |
| `set -T` | DEBUG/RETURN traps inherit in functions |
| `set -o errtrace` | Same as `-E` |
| `set -o functrace` | Same as `-T` |

### Companion features

| Feature | Purpose |
|---------|---------|
| `trap 'cmd' ERR` | Run `cmd` when any command fails |
| `trap 'cmd' EXIT` | Run `cmd` on script exit (always) |
| `${var:?error msg}` | Fail with message if `var` is unset or null |
| `${var:-default}` | Use default if `var` is unset or null |
| `${var-default}` | Use default if `var` is unset ONLY (not null) |
| `cmd || true` | Suppress `-e` failure for expected errors |
| `if cmd; then ... fi` | Allow "failure" in condition context |
| `! cmd` | Invert exit code (allowed under `-e`) |

## Under the Hood

### How set -e works at the kernel level

`set -e` is purely a **shell-level feature** — there's no kernel involvement. Bash tracks the exit status of every command it executes. When a command returns non-zero, bash checks:

1. Is `set -e` enabled?
2. Is the command in a "protected" context (inside `if`, `while`, `until`, `||`, `&&`, `!`)?
3. If not protected → call `exit(1)` (after running any EXIT trap)

The tricky part: what counts as "protected"? Bash has specific rules:

- `if cmd; then` — `cmd` is protected (failure expected)
- `cmd ||` — everything before `||` is protected
- `cmd &&` — everything before `&&` is protected
- `while cmd; do` — `cmd` is protected
- `! cmd` — `cmd` is protected
- Functions: `-e` applies INSIDE functions unless the function is called in a protected context
- Subshells: `-e` applies INSIDE subshells

### How set -u works

When bash encounters an unset variable:
1. Does the expansion use `${var:-}` or `${var:+}` syntax? → OK, use default/null
2. Does the expansion use `${var:?}`? → Print message and exit
3. Is `-u` set? → Print error and exit
4. Otherwise → Expand to empty string

Edge cases with `-u`:
- `$@` and `$*` are exempt — they don't trigger `-u` when no arguments exist
- `$#` (argument count) is always set
- `$?` is always set
- `$$` is always set
- Array expansion `${arr[@]}` with unset `arr` DOES trigger `-u`

### How pipefail works

Without `pipefail`, a pipeline's exit status is the exit status of the LAST command:
```bash
false | true   # exit code: 0 (true's exit code)
```

With `pipefail`, the pipeline fails if ANY command fails:
```bash
set -o pipefail
false | true   # exit code: 1 (false's exit code)
```

Bash implements this by tracking each process in the pipeline via `waitpid(2)`. After all processes finish, it checks the exit status of each one in reverse order (last to first), returning the first non-zero status found.

### Memory/performance implications

- Strict mode has ZERO performance overhead — it's just conditional logic in bash's command loop
- `trap ERR` adds minimal overhead (triggered only on failures)
- `trap EXIT` adds no per-command overhead
- `set -u` has no performance impact (variable lookup is unchanged, just adds a check)

## Core Examples (8-12 minimum)

### Example 1: Without strict mode (silent failures)

**Command:**
```bash
#!/bin/bash
echo "Before"
cd /nonexistent
echo "After"        # This runs despite cd failing!
echo "Exit: $?"
```

**Output:**
```
Before
./script.sh: line 3: cd: /nonexistent: No such file or directory
After
Exit: 1
```

**Step-by-step:**
1. `cd /nonexistent` fails (exit code 1)
2. The error message goes to stderr
3. The script CONTINUES running — it doesn't care about the error
4. "After" prints, then "Exit: 1"
5. The exit code is 1, but no one checked it

**Variations:**
- This is the DEFAULT bash behavior — errors are non-fatal
- Every command's exit code must be checked manually
- This is why `&&` chaining exists: `cmd1 && cmd2 && cmd3`

### Example 2: With strict mode (failure stops)

**Command:**
```bash
#!/bin/bash
set -euo pipefail
echo "Before"
cd /nonexistent
echo "After"        # This never runs
```

**Output:**
```
Before
./script.sh: line 5: cd: /nonexistent: No such file or directory
```

**Step-by-step:**
1. `set -euo pipefail` enables all three protections
2. `cd /nonexistent` fails (exit code 1)
3. `-e` triggers — bash exits immediately
4. "After" never executes
5. The script exits with code 1

**Variations:**
- `cd /nonexistent || exit 1` — manual equivalent (without `-e`)
- `cd /nonexistent || { echo "Failed"; exit 1; }` — with custom message

### Example 3: pipefail in action

**Command:**
```bash
#!/bin/bash
echo "Without pipefail:"
grep "nothing" /etc/passwd | wc -l
echo "Exit code: $?"

set -o pipefail
echo "With pipefail:"
grep "nothing" /etc/passwd | wc -l
echo "Exit code: $?"
```

**Output:**
```
Without pipefail:
0
Exit code: 0
With pipefail:
0
Exit code: 1
```

**Step-by-step:**
1. First run: without pipefail. `grep` returns 1 (no match), but `wc -l` returns 0. Pipeline exit code is `wc -l`'s 0.
2. Second run: with pipefail. `grep` returns 1, so pipeline exit code is 1.
3. The `0` output (from `wc -l`) is correct — no matches found
4. But without pipefail, you don't know `grep` failed!

**Variations:**
- `grep -c "pattern" file || true` — handle expected grep failures
- `grep "pattern" file | head -n 5` — `head` terminating early causes SIGPIPE in `grep`

### Example 4: trap ERR in strict mode

**Command:**
```bash
#!/bin/bash
set -euo pipefail

error_handler() {
    local rc=$?
    echo "[ERROR] Line $1: '$BASH_COMMAND' failed with code $rc"
    exit $rc
}
trap 'error_handler $LINENO' ERR

echo "Starting..."
cd /nonexistent
echo "This never runs"
```

**Output:**
```
Starting...
[ERROR] Line 14: 'cd /nonexistent' failed with code 1
```

**Step-by-step:**
1. `trap ... ERR` registers the error handler
2. `cd /nonexistent` fails
3. `-e` would normally exit — but first, the ERR trap fires
4. `error_handler` runs: prints the command, line number, and exit code
5. Then it calls `exit $rc` to terminate

**Variations:**
- ERR trap fires BEFORE `-e` exits
- ERR trap does NOT fire inside `if`, `||`, `&&` conditions (same exceptions as `-e`)
- Combine with `caller` for full stack trace

### Example 5: EXIT trap in strict mode

**Command:**
```bash
#!/bin/bash
set -euo pipefail

cleanup() {
    echo "Cleaning up before exit..."
    rm -f /tmp/tempfile
}
trap cleanup EXIT

echo "Working..."
touch /tmp/tempfile
false   # triggers exit
echo "This never runs"
```

**Output:**
```
Working...
Cleaning up before exit...
```

**Step-by-step:**
1. `trap cleanup EXIT` — `cleanup()` runs on ANY exit
2. `false` returns 1, `-e` triggers exit
3. BEFORE bash exits, the EXIT trap fires
4. `cleanup()` removes the temp file
5. Script exits

**Variations:**
- EXIT trap runs even if `-e` triggers it
- EXIT trap runs on normal `exit 0` too
- EXIT trap runs on `SIGINT` if `-e` is set (since `-e` doesn't catch signals)

### Example 6: Exception — `-e` disabled in conditions

**Command:**
```bash
#!/bin/bash
set -euo pipefail

echo "Before false in if"
if false; then
    echo "This won't print"
fi
echo "Still alive!"    # This DOES print!

echo "Before false NOT in if"
false                  # This triggers -e
echo "This never runs"
```

**Output:**
```
Before false in if
Still alive!
Before false NOT in if
```

**Step-by-step:**
1. First `false` is inside `if` — `-e` is suspended for the condition
2. Script continues
3. Second `false` is NOT protected — `-e` triggers, script exits

**Variations:**
- `while false; do ... done` — `-e` suspended
- `until false; do ... done` — `-e` suspended
- `! false` — `-e` suspended (the `!` inverts, but context is protected)

### Example 7: Handling expected failures with `|| true`

**Command:**
```bash
#!/bin/bash
set -euo pipefail

echo "Before grep"
grep "nonexistent_pattern" /etc/passwd || true
echo "After grep — still alive!"

echo "Before grep (no ||)"
grep "nonexistent_pattern" /etc/passwd
echo "After grep — never reached"
```

**Output:**
```
Before grep
After grep — still alive!
Before grep (no ||)
```

**Step-by-step:**
1. First grep: `|| true` means "if grep fails, run `true`." The overall command succeeds.
2. Second grep: no fallback — `-e` catches the failure and exits.

**Variations:**
- `cmd || :` — using the null command (`:` is a builtin that always succeeds)
- `cmd | true` — doesn't work the same way! `true` ignores input, but the pipe may hide the failure
- `cmd || handler_func` — run a custom handler on failure

### Example 8: `${var:?}` for required variables

**Command:**
```bash
#!/bin/bash
set -euo pipefail

username=${1:?Usage: $0 <username>}
echo "Hello, $username"
```

**Input:** `./script.sh`

**Output:**
```
./script.sh: line 4: 1: Usage: ./script.sh <username>
```

**Step-by-step:**
1. `$1` is unset (no argument provided)
2. `${1:?Usage: $0 <username>}` — the `:?` triggers because `$1` is unset
3. The message is printed to stderr
4. Script exits with code 1
5. No need for manual `if [ -z "$1" ]; then ... fi`

**Variations:**
- `${var:?}` — no custom message (bash prints default error)
- Without the colon: `${var?}` — only fails if var is UNSET, not null
- `${USER:?USER must be set}` — ensure environment variable exists

### Example 9: Default values with ${var:-}

**Command:**
```bash
#!/bin/bash
set -euo pipefail

port=${PORT:-8080}
host=${HOST:-localhost}
echo "Starting on $host:$port"
```

**Input:** `PORT=3000 ./script.sh`

**Output:**
```
Starting on localhost:3000
```

**Step-by-step:**
1. `${PORT:-8080}` — if `$PORT` is set and not null, use its value; otherwise use `8080`
2. `$HOST` is not set, so `localhost` is used
3. This is the SAFE way to handle optional variables under `-u`

**Variations:**
- Without colon: `${PORT-8080}` — only uses default if UNSET (allows empty PORT)
- `${var:+alt}` — use `alt` if `var` is set, otherwise empty
- Can nest: `${port:-${DEFAULT_PORT:-80}}`

### Example 10: Strict mode with subshell exceptions

**Command:**
```bash
#!/bin/bash
set -euo pipefail

echo "Before"
(
    echo "In subshell"
    false
    echo "In subshell after false — never runs"
)
echo "After subshell — never runs either!"
```

**Output:**
```
Before
In subshell
```

**Step-by-step:**
1. `set -e` is inherited by the subshell
2. Inside `( )`, `false` returns 1
3. `-e` triggers — the subshell exits
4. The parent shell sees the subshell's exit code (1)
5. `-e` in the parent ALSO triggers — parent exits
6. "After subshell" never prints

**Variations:**
- `( false ) || echo "ok"` — `||` protects the parent
- `( set +e; false; )` — disable inside subshell only

### Example 11: Strict mode with functions

**Command:**
```bash
#!/bin/bash
set -euo pipefail

fail_if_negative() {
    local n=$1
    if (( n < 0 )); then
        return 1
    fi
    echo "n=$n is OK"
}

echo "Testing 5"
fail_if_negative 5
echo "Still alive"

echo "Testing -1"
fail_if_negative -1
echo "Will this run?"   # Nope
```

**Output:**
```
Testing 5
n=5 is OK
Still alive
Testing -1
```

**Step-by-step:**
1. `fail_if_negative 5` — returns 0, script continues
2. `fail_if_negative -1` — returns 1
3. `-e` catches it — script exits
4. Note: the function's `return 1` is treated as a failure even inside the function body

**Variations:**
- Functions inherit `-e` from caller
- `caller || true` — protect a function call that may fail
- `set -e` in function body is independent of caller's `-e` setting

### Example 12: Not checking `$?` (the original sin)

**Command:**
```bash
#!/bin/bash
# The old way — checking $? manually
cd /somewhere
if [[ $? -ne 0 ]]; then
    echo "Failed to cd"
    exit 1
fi
```

**Strict mode way:**
```bash
#!/bin/bash
set -euo pipefail
cd /somewhere    # just do it — -e handles failure
# Or with a message:
cd /somewhere || { echo "Failed to cd"; exit 1; }
```

**Step-by-step:**
1. Without strict mode, you'd litter your code with `if [[ $? -ne 0 ]]; then ...`
2. With strict mode, failures are caught automatically
3. Only handle the ones you EXPECT to fail

**Variations:**
- `|| exit 1` — handles failure with exit
- `|| echo "failed" >&2 && exit 1` — log and exit
- `|| return 1` — in a function, propagate failure

## Real-World Use Cases

### FOR the OS

- **System scripts**: `/etc/init.d/`, `/etc/cron.d/`, `/usr/local/bin/` scripts should ALL use strict mode
- **CI/CD pipelines**: GitHub Actions, GitLab CI, Jenkins scripts — `set -euo pipefail` prevents silent failures
- **Docker entrypoints**: `set -euo pipefail` at the top prevents container startup with partial failures
- **Provisioning scripts**: Ansible shell modules, Terraform provisioners, cloud-init scripts
- **Backup scripts**: `set -euo pipefail` ensures partial backups are caught immediately

### WITH the OS

- **`set -euo pipefail` in sourced profiles**: Set it in `~/.bashrc` for interactive safety? (Not recommended — `set -u` breaks many completion scripts)
- **`set -u` with `BASH_ENV`**: If you set strict mode in `~/.bashrc`, sourced scripts may break on unset variables
- **`pipefail` with `grep`**: Common pattern: `grep -q pattern file || true` to handle "not found" as expected

### AGAINST THE OS (Security Perspective)

- **`set -e` hides where errors occur**: When a script exits with `set -e`, you know something failed but not what. An attacker can trigger a failure to bypass later security checks.
- **Inconsistent `-e` behavior across bash versions**: Different bash versions handle edge cases differently. An attacker might exploit a version-specific `-e` quirk.
- **`set -u` + injection**: If an attacker can set environment variables, they might cause a script to exit early via `-u` by unsetting expected variables (denial of service).
- **`pipefail` cmd masking**: `cmd1 | cmd2 | cmd3` — an attacker who controls any stage can make the pipeline fail, masking errors in other stages.
- **`trap ERR` handler hijacking**: If an attacker can write to a script after trap but before the failing command, they can change the error handler.

### FOR DEFENSE

- **Always `set -euo pipefail`** at the top of production scripts
- **`trap ERR`** with detailed logging — know EXACTLY what failed where
- **`trap EXIT`** for cleanup — ensure resources are released even on strict-mode exit
- **Use `${var:-}`** for optional variables to avoid `-u` failures
- **Use `${var:?required}`** for mandatory variables with descriptive messages
- **Don't rely on `-e` alone** for security-critical operations — always validate explicitly
- **Test scripts under `-u`** to find all unset variable references before production

## Memory Aids

- **`set -e`** = "e" for "exit on error" — or "EXTERMINATE"
- **`set -u`** = "u" for "unset is unacceptable"
- **`pipefail`** = "if any pipe in the pipeline fails, the WHOLE thing fails"
- **`set -euo pipefail`** = "error, unset, pipe" = "EUP" = "everything under protection"
- **`${var:?}`** = looks like a question mark — "QUESTION this variable's existence"
- **`${var:-}`** = looks like a dash — "DASH away to the default"
- **`|| true`** = "OR else do nothing that fails"

## Trap Vault (8-12 traps)

### Trap 1: `set -e` doesn't work inside `if`

**Problem:** Command inside `if` condition fails but script continues (unexpectedly).

**Bad Example:**
```bash
set -e
if grep "pattern" file; then
    echo "Found"
fi
echo "grep failed but I still run!"
```

**Root Cause:** `-e` is DISABLED inside `if` conditions by design. This is intentional — `if` is meant to handle failure.

**Fix:** This is NOT a bug — `-e` is supposed to do this. If you want the failure to exit even in `if`, add `|| exit 1`:
```bash
if grep "pattern" file || exit 1; then
```

### Trap 2: `set -u` with array expansion

**Problem:** Error on `${arr[@]}` when array is unset.

**Bad Example:**
```bash
set -u
echo "${arr[@]}"  # error: arr: unbound variable
```

**Root Cause:** Unlike scalar variables, array expansion triggers `-u` even when you'd expect it to be empty.

**Fix:**
```bash
echo "${arr[@]-}"  # use default empty string
# or
arr=()  # initialize as empty
```

### Trap 3: pipefail with `head`/`tail` — SIGPIPE

**Problem:** Pipeline fails unexpectedly with `head`.

**Bad Example:**
```bash
set -eo pipefail
grep "pattern" hugefile.txt | head -n 5
# Exit code 141 (128 + 13 = SIGPIPE)
```

**Root Cause:** `head` terminates after 5 lines. `grep` tries to write more and gets SIGPIPE (signal 13). With pipefail, this becomes a pipeline failure.

**Fix:**
```bash
grep "pattern" hugefile.txt | head -n 5 || true
# Or use grep -m 5 to limit matches
grep -m 5 "pattern" hugefile.txt
```

### Trap 4: `${var:?}` with positional parameters in functions

**Problem:** `${1:?error}` inside a function shows wrong context.

**Bad Example:**
```bash
greet() {
    local name=${1:?Name required}
    echo "Hello, $name"
}
greet  # works, but ...
greet ""  # also fails — :? rejects null too
```

**Root Cause:** `${var:?}` fails on null too (the `:` version). Use `${var?}` (without colon) to only fail on unset.

**Fix:**
```bash
local name=${1?Name required}  # only fails if no arg
# Or check manually:
[[ -z "${1-}" ]] && { echo "Name required"; return 1; }
```

### Trap 5: Source/`.` breaks with `-u`

**Problem:** Sourcing a script that references parent variables fails.

**Bad Example:**
```bash
# parent.sh
set -u
# ...
source child.sh

# child.sh
echo "Path: $PATH"   # fine, PATH is set
echo "Custom: $CUSTOM_VAR"  # error if CUSTOM_VAR not set!
```

**Root Cause:** The sourced script inherits `-u` and fails on any unset variable it references.

**Fix:** Either set defaults in the sourced script or check before referencing:
```bash
echo "Custom: ${CUSTOM_VAR:-unset}"
```

### Trap 6: `set -e` with compound commands

**Problem:** The LAST command in a compound command determines failure.

**Bad Example:**
```bash
set -e
false && true
echo "This runs!"  # Because false && true exits 1? Actually no...
```

Wait: `false && true` exits 1 (false is 1, && short-circuits). With `-e`, this should exit.

Let me reconsider:
```bash
set -e
false && true   # exits 1 due to false
# -e should trigger here because the compound command failed
echo "will this run?"
```

Actually, `-e` has a specific exception: it does NOT trigger on commands in a `&&` or `||` list that are NOT the final command. But `false && true` — the WHOLE command is `false && true`, and it fails. So `-e` DOES trigger.

The tricky case:
```bash
set -e
true && false   # whole command fails (false is last)
true || false   # whole command succeeds (true succeeds, short-circuit)
false || true   # whole command succeeds (true runs after false)
```

**Root Cause:** The rules for `-e` with `&&`/`||` are subtle and version-dependent.

**Fix:** Make it explicit:
```bash
true && false || exit 1
```

### Trap 7: Forgetting `set +e` for expected failures

**Problem:** Script exits when you expected a command to fail.

**Bad Example:**
```bash
set -euo pipefail
# Check if a package is installed
dpkg -s apache2 2>/dev/null  # exits 1 if not installed — trigger -e!
echo "Package check complete"
```

**Root Cause:** `dpkg -s` returns 1 if the package isn't installed. `-e` catches it.

**Fix:**
```bash
if dpkg -s apache2 2>/dev/null; then
    echo "apache2 is installed"
else
    echo "apache2 is NOT installed"
fi
# Script continues because -e is disabled inside if
```

### Trap 8: `set -e` ignored in `grep -q`

**Problem:** `grep -q` inside a pipeline crashes the script.

**Bad Example:**
```bash
set -eo pipefail
grep -q "pattern" hugefile.txt | wc -l
```

**Root Cause:** `grep -q` exits as soon as it finds the first match — it closes stdout. If it's in a pipeline, the next command gets a broken pipe.

**Fix:** Don't use `-q` in the middle of pipelines:
```bash
grep "pattern" hugefile.txt | wc -l
# Or use grep -c instead of wc -l
grep -c "pattern" hugefile.txt
```

### Trap 9: `set -u` with indirect expansion `${!var}`

**Problem:** `${!var_name}` fails if the variable named by `var_name` doesn't exist.

**Bad Example:**
```bash
set -u
var_name="NONEXISTENT_VAR"
echo "${!var_name}"  # error: NONEXISTENT_VAR: unbound variable
```

**Root Cause:** Indirect expansion dereferences the variable; if it doesn't exist, `-u` triggers.

**Fix:**
```bash
# Use default
echo "${!var_name:-}"
# Or check first
if [[ -n "${!var_name-}" ]]; then
    echo "${!var_name}"
fi
```

### Trap 10: `set -e` in `for` loop when command fails

**Problem:** `-e` inside a `for` loop exits the entire loop.

**Bad Example:**
```bash
set -e
for file in /nonexistent/*.txt; do
    echo "Processing $file"
done
echo "Loop done"  # never runs if /nonexistent doesn't exist
```

**Root Cause:** If the glob `/nonexistent/*.txt` matches nothing (and `nullglob` is not set), bash passes the literal string `/nonexistent/*.txt` as the loop value. The loop still runs once — but if you reference `$file` in a command that fails, `-e` exits.

Actually, with `nullglob` off, the glob pattern is passed literally. The loop runs once with `file=/nonexistent/*.txt`. No failure from the glob itself.

But if the glob matches files and one fails... the loop exits.

**Fix:**
```bash
shopt -s nullglob
for file in /nonexistent/*.txt; do
    echo "Processing $file" || true
done
```

### Trap 11: `trap ERR` doesn't fire in all contexts

**Problem:** ERR trap doesn't fire for some commands.

**Bad Example:**
```bash
set -e
trap 'echo "ERROR"' ERR
false | true   # ERR trap doesn't fire!
```

**Root Cause:** Without pipefail, the pipeline exit code is 0 (from `true`). `-e` doesn't trigger and neither does ERR trap.

**Fix:**
```bash
set -eo pipefail
trap 'echo "ERROR"' ERR
false | true   # Now: pipeline fails → ERR trap fires → -e exits
```

## See It In The Wild

- **Most production bash scripts** start with `set -euo pipefail` followed by `trap ... EXIT` and `trap ... ERR`
- **Docker's official images** entrypoint scripts: `#!/bin/bash\nset -euo pipefail`
- **Kubernetes helm charts** often generate scripts with strict mode
- **GitLab CI `before_script`**: `set -euo pipefail` is commonly the first command
- **Arch Linux PKGBUILDs**: often use strict mode in `build()` and `package()` functions

**Try this now:**

1. `bash -c 'set -u; echo $NONEXISTENT'` — feel the `-u` sting
2. `bash -c 'set -e; false; echo "alive"'` vs `bash -c 'set +e; false; echo "alive"'` — see `-e` in action
3. `bash -c 'set -o pipefail; false | true; echo $?'` vs `bash -c 'set +o pipefail; false | true; echo $?'` — pipefail difference
4. `bash -c 'set -u; : ${1:?Need an arg}'` — `${var:?}` test

## Check Your Understanding (5-7 questions)

1. What does `set -o pipefail` do, and why is it important?
2. Why does `set -e` not trigger inside an `if` condition?
3. How do you provide a default value for a variable in strict mode?
4. What's the difference between `cmd || true` and just `cmd`?
5. Why might `source` or `.` break with `-u` enabled?
6. What's the difference between `${var:-default}` and `${var-default}`?
7. Why does `set -e` behave differently in different bash versions?
8. What happens if you forget `set +e` before running a command that's expected to fail?
