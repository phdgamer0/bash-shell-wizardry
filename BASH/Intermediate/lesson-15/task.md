# Task 15: Debugging — Strict Mode

You'll take a sloppy, error-prone script and harden it with strict mode. Then you'll build a reusable error handling library. This is the difference between scripts that silently corrupt data and scripts that fail loudly and safely.

## Sub-tasks

### 1. The non-strict script — `loose.sh`

Save this buggy script and observe its behavior. It has several issues that strict mode would catch.

**loose.sh:**
```bash
#!/bin/bash
# loose.sh — has several issues strict mode would catch
PROJECT=$1
echo "Setting up project: $PROJECT"
cd /projects/$PROJECT
grep "error" log.txt | wc -l
echo "Line count: $LINES"
rm -rf ./temp
echo "Done."
```

**Bugs to identify (before fixing):**
1. `$1` may be unset → `PROJECT` is empty → everything breaks silently
2. `cd /projects/$PROJECT` may fail → script continues in wrong directory
3. `grep "error" log.txt | wc -l` — if `log.txt` doesn't exist, `grep` fails but pipe hides it
4. `$LINES` is likely an unset variable (terminal `LINES` is set in interactive shells, not scripts)
5. `rm -rf ./temp` — what if this fails?
6. No error reporting anywhere — failures cascade silently

**Expected (broken) behavior:**
```bash
$ ./loose.sh
Setting up project:
cd: /projects/: No such file or directory
grep: log.txt: No such file or directory
0
Line count:        ← LINES is empty or zero
Done.
```
Everything "succeeds" with exit code 0. The user has NO IDEA that most operations failed.

### 2. Add strict mode — `strict.sh`

Create `strict.sh` from `loose.sh` by adding `set -euo pipefail` at the top.

**Requirements:**
- Add `set -euo pipefail` as the first non-comment line
- Run `./strict.sh testproj` and observe ALL the failures
- Note that even with strict mode, the first error (`cd`) kills the script — but that's BETTER than continuing blindly

**Expected:**
```bash
$ ./strict.sh testproj
Setting up project: testproj
./strict.sh: line 7: cd: /projects/testproj: No such file or directory
# Script stops here — that's actually good!
```

### 3. Fix each failure — `fixed.sh`

Create `fixed.sh` that handles every issue properly.

**Requirements for each fix:**

**Fix 1: Missing argument**
```bash
PROJECT=${1:?Usage: $0 <project-name>}
```

**Fix 2: cd failure with context**
```bash
cd "/projects/$PROJECT" || { echo "Error: project directory not found"; exit 1; }
```
Or check first:
```bash
[[ -d "/projects/$PROJECT" ]] || { echo "Error: /projects/$PROJECT not found"; exit 1; }
cd "/projects/$PROJECT"
```

**Fix 3: grep + pipefail**
The grep may legitimately find nothing (exit 1). Handle expected failures:
```bash
count=$(grep -c "error" log.txt 2>/dev/null || echo 0)
echo "Error count: $count"
```
Or:
```bash
grep "error" log.txt 2>/dev/null | wc -l || true
```

**Fix 4: LINES variable**
`$LINES` is not set in non-interactive shells. Use a different name or provide a default:
```bash
term_lines=${LINES:-40}
echo "Terminal lines: $term_lines"
```

**Fix 5: rm failures**
```bash
rm -rf ./temp || echo "Warning: temp directory cleanup failed"
```

**Fix 6: trap ERR for error reporting**
```bash
error_handler() {
    echo "[ERROR] Line $LINENO: Command failed: $BASH_COMMAND (exit $?)"
}
trap 'error_handler' ERR
```

**Expected (fixed) behavior:**
```bash
$ ./fixed.sh testproj
Setting up project: testproj
[ERROR] Line 10: Command failed: cd /projects/testproj (exit 1)
Error: project directory not found

$ ./fixed.sh myapp
Setting up project: myapp
Checking /projects/myapp...
Error count: 3
Terminal lines: 40
temp cleaned up.
Done.
```

### 4. Error handler library — `error_lib.sh`

Build a reusable error handling library that can be sourced by any script.

**Requirements:**

**Function `die()`:**
- Prints "[FATAL] message" to stderr
- Exits with given code (default 1)

**Function `require_var()`:**
- Takes a variable name and optional message
- If the variable is unset or empty, calls `die()`
- Use indirect expansion `${!var}`

**Function `require_cmd()`:**
- Takes a command name
- Checks if it exists with `command -v`
- Calls `die()` if not found

**Function `safe_cd()`:**
- Takes a directory path
- Attempts `cd`, calls `die()` on failure
- Prints success message

**Function `log_error()`:**
- Takes a message
- Prints "[ERROR] $(date +%T) - message" to stderr
- Does NOT exit (just logs)

**Template:**
```bash
#!/bin/bash
# error_lib.sh — reusable error handling library
# Source this file: source error_lib.sh

set -euo pipefail

die() {
    local msg="${1:-Unknown error}"
    local code="${2:-1}"
    echo "[FATAL] $msg" >&2
    exit "$code"
}

require_var() {
    local var_name="$1"
    local msg="${2:-Required variable $var_name is not set}"
    if [[ -z "${!var_name-}" ]]; then
        die "$msg"
    fi
}

require_cmd() {
    command -v "$1" >/dev/null 2>&1 || die "Required command not found: $1"
}

safe_cd() {
    cd "$1" 2>/dev/null || die "Cannot cd to directory: $1"
}

log_error() {
    echo "[ERROR] $(date +'%T') - $1" >&2
}
```

**Expected:**
```bash
$ source error_lib.sh
$ require_var PATH          # fine, PATH is set
$ require_var NONEXISTENT
[FATAL] Required variable NONEXISTENT is not set

$ require_cmd ls            # fine
$ require_cmd nonexistent_cmd
[FATAL] Required command not found: nonexistent_cmd

$ safe_cd /tmp              # fine
$ safe_cd /nonexistent
[FATAL] Cannot cd to directory: /nonexistent

$ log_error "Something went wrong"
[ERROR] 12:00:00 - Something went wrong
# does NOT exit
```

### 5. Integrate error library — `robust.sh`

Write `robust.sh` that sources `error_lib.sh` and uses its functions.

**Requirements:**
- Sources `error_lib.sh`
- Uses `require_var` for mandatory environment variables
- Uses `require_cmd` for required tools (`grep`, `tar`, `curl`, etc.)
- Uses `safe_cd` for directory changes
- Uses `log_error` for non-fatal warnings
- Uses `die` for fatal failures
- Includes a `trap ERR` that calls `log_error` with context

**Expected:**
```bash
$ ./robust.sh
[ERROR] 12:00:01 - Line 12: require_cmd: Required command not found: unnecessary_tool

$ ./robust.sh
source error_lib.sh
require_var DB_HOST "Database host required"
# If DB_HOST is unset:
[FATAL] Database host required
```

## Solution Approaches

### Approach A: Incremental hardening
1. Run `loose.sh` — see the silent failures
2. Add `set -euo pipefail` one option at a time, observing each effect
3. Fix each failure
4. Build `error_lib.sh` from the patterns that emerged
5. Create `robust.sh` using the library

### Approach B: Library-first
1. Write `error_lib.sh` first — the reusable functions
2. Write `fixed.sh` using the library functions
3. Compare with `loose.sh` to see the improvement
4. Write `robust.sh` as a showcase

### Approach C: Test-driven
1. Define expected behavior for `fixed.sh`
2. Add strict mode, let failures reveal themselves
3. Fix each failure in order
4. Extract common patterns into `error_lib.sh`

<details>
<summary>Hint 1: Loose script bugs revealed</summary>

Run with `bash -x` to see every subtle issue:
```bash
$ bash -x loose.sh testproj 2>&1
+ PROJECT=testproj
+ echo 'Setting up project: testproj'
+ cd /projects/testproj
+ cd: /projects/testproj: No such file or directory
+ grep error log.txt
+ grep: log.txt: No such file or directory
+ wc -l
+ echo 'Line count: '
+ rm -rf ./temp
+ echo Done.
```

Note: `$LINES` expands to empty because it's not set in non-interactive shells!
</details>

<details>
<summary>Hint 2: Strict mode addition order</summary>

Test each option individually:
```bash
set -e      # stop on first error
set -u      # error on unset variables
set -o pipefail  # pipeline fails if any stage fails
```

Then combine. Each reveals different bugs:
- `-u` catches `$LINES` being unset
- `-e` catches `cd` failure
- `pipefail` catches `grep` pipe failure
</details>

<details>
<summary>Hint 3: Fixed script structure</summary>

```bash
#!/bin/bash
set -euo pipefail

PROJECT=${1:?Usage: $0 <project-name>}
echo "Setting up project: $PROJECT"

PROJECT_DIR="/projects/$PROJECT"
if [[ ! -d "$PROJECT_DIR" ]]; then
    echo "Error: Project directory $PROJECT_DIR not found" >&2
    exit 1
fi
cd "$PROJECT_DIR"

log_file="${PROJECT_DIR}/log.txt"
if [[ -f "$log_file" ]]; then
    count=$(grep -c "error" "$log_file" 2>/dev/null || echo 0)
    echo "Error count: $count"
else
    echo "No log file found"
fi

echo "Terminal lines: ${LINES:-40}"
rm -rf ./temp 2>/dev/null && echo "temp cleaned up." || echo "Warning: temp cleanup skipped"
echo "Done."
```
</details>

<details>
<summary>Hint 4: Error library die function patterns</summary>

```bash
die() {
    local msg="${1:-Unknown error}"
    local code="${2:-${?:-1}}"   # preserve last exit code if available
    echo "[FATAL] $msg" >&2
    exit "$code"
}

# Usage:
cd /somewhere || die "Can't access /somewhere" $?
```
</details>

<details>
<summary>Hint 5: Indirect expansion for require_var</summary>

```bash
require_var() {
    local var_name="$1"
    local msg="${2:-Required variable '$var_name' is not set}"
    # Use indirect expansion: ${!var_name} expands to the value of the variable named by var_name
    if [[ -z "${!var_name:-}" ]]; then
        die "$msg"
    fi
}

# Test:
MY_VAR="hello"
require_var MY_VAR   # passes
require_var UNDEFINED  # fails
```
</details>

<details>
<summary>Hint 6: Trap ERR with library integration</summary>

```bash
# In robust.sh
source error_lib.sh

error_handler() {
    local rc=$?
    log_error "Line $LINENO: Command '$BASH_COMMAND' failed (exit $rc)"
    # Don't exit — let -e handle that
}
trap 'error_handler' ERR

set -euo pipefail
# ... script continues, ERR trap fires on failures
```
</details>

## Bonus Challenges

1. **Stack trace in die()**: Modify `die()` to print the call stack using `caller` in a loop before exiting
2. **Error suppression regions**: Implement `pushd_err` / `popd_err` that temporarily disable strict mode for sections where failure is expected
3. **Email alerts**: Add a `notify_admin()` function to the error library that sends an email when a fatal error occurs
4. **Recovery actions**: Implement `try()` / `catch()` functions that mimic exception handling — `try cmd || catch "handle failure"`
5. **Verbose mode**: Add `VERBOSE` environment variable support — when set, `log_error` also includes system info (hostname, timestamp, script version)

## Expected Output Summary

```
loose.sh:        Silent failures, continues on errors, wrong output
strict.sh:       Added strict mode → reveals all bugs immediately
fixed.sh:        All bugs fixed, proper error handling, informative messages
error_lib.sh:    Reusable library with die, require_var, require_cmd, safe_cd, log_error
robust.sh:       Integration example using error_lib.sh
```

## Self-Check

- What does `set -o pipefail` do?
- Why does `set -e` not trigger inside an `if` condition?
- How do you provide a default value for a variable in strict mode?
- What's the difference between `cmd || true` and just `cmd`?
- Why might `source` or `.` break with `-u` enabled?
- What's the difference between `${var:-default}` and `${var-default}`?
- How does the `die()` function differ from just calling `exit 1`?
- Why should you always use `trap EXIT` even with strict mode?
