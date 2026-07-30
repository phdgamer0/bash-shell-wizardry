# Lesson 2: Parameter Expansion — Defaults

## History & Origins

Parameter expansion is one of the oldest features in shell programming, originating in the **Bourne shell** (1979, Stephen Bourne at Bell Labs). The original Bourne shell introduced `${var-word}` syntax to provide default values and `${var=word}` to assign defaults.

The *colon variants* (`${var:-word}`, `${var:=word}`, `${var:?msg}`, `${var:+word}`) were added later — the colon adds "null-checking" so that an empty string triggers the same behavior as an unset variable. These were present in **System III Unix** (1980) and standardized in **POSIX.2** (1992).

Bash has always supported the full POSIX parameter expansion syntax. Modern bash extends it with `${!var}` (indirect expansion, bash 2.0+), `${!prefix*}` (variable name matching), and case modification operators (bash 4.0+).

## Syntax Reference

### Basic Forms — the five : operators

| Expression | When var is unset | When var is null/empty | When var is set & non-null |
|------------|---------------------|------------------------|----------------------------|
| `${var:-word}` | expands to word | expands to word | expands to $var |
| `${var-word}` | expands to word | expands to "" | expands to $var |
| `${var:=word}` | assigns word to var | assigns word to var | expands to $var |
| `${var=word}` | assigns word to var | expands to "" | expands to $var |
| `${var:?message}` | prints message, exits | prints message, exits | expands to $var |
| `${var?message}` | prints message, exits | expands to "" | expands to $var |
| `${var:+word}` | expands to "" | expands to "" | expands to word |
| `${var+word}` | expands to "" | expands to word | expands to word |

### Edge Cases

- **`word` can be another expansion**: `${var:-${OTHER:-default}}` — nested defaults.
- **`word` can be a command substitution**: `${var:-$(date)}` evaluates command only if needed.
- **Positional parameters**: `${1:-default}`, `${2:?Missing arg 2}` all work.
- **Special variables**: `$-`, `$?`, `$$`, `$!` all work with defaults.
- **Array defaults**: `${arr[0]:-default}` works on individual elements.
- **No spaces allowed**: `${var :- default}` is syntax error.
- **Exit on `:?`**: In a script, terminates immediately with exit code 1. In interactive shell, prints error but doesn't exit.

## Under the Hood

### What bash does

When bash encounters `${var:-word}`:

1. **Tokenization**: Lexer identifies `${` as start of parameter expansion.
2. **Parsing**: Reads parameter name, operator, and word, matching `}`.
3. **Variable lookup**: Searches shell's internal hash table for `var`.
4. **Null/unset check**: If `var` is not found (or null and `:` present), expand `word` instead.
5. **Word expansion**: The `word` undergoes full expansion cycle (brace, tilde, parameter, command substitution, arithmetic, word splitting, pathname expansion).
6. **Result substitution**: The expanded word replaces `${var:-word}` textually.

### System Calls

For parameter expansion defaults: **zero system calls**. Everything in-process — hash table lookup, string comparison, string copy.

### Memory

The shell's variable table is a hash table. When `:=` triggers assignment, a new string is allocated via `xmalloc`, copied, and the pointer stored. Old value is freed first.

### Equivalent C/Python

```c
// C equivalent of ${var:-default}
const char *result = getenv("var") && strlen(getenv("var")) > 0
    ? getenv("var")
    : "default";
```

```python
# Python equivalent
result = os.environ.get("var") or "default"
```

## Core Examples (12 minimum)

### Example 1: Basic fallback with `:-`

```bash
$ unset name
$ echo "Hello, ${name:-World}"
Hello, World
$ name=""
$ echo "Hello, ${name:-World}"
Hello, World
$ name="Alice"
$ echo "Hello, ${name:-World}"
Hello, Alice
```

**Step-by-step**: 1. `name` unset → expansion uses "World". 2. `name=""` → colon says "treat empty as unset" → "World". 3. `name="Alice"` → uses "Alice".

**What if** you use `${name-World}` (no colon)? When `name=""`, it expands to "" (variable *is* set, just empty).

### Example 2: Assign default with `:=`

```bash
$ echo "Count: ${count:=10}"
Count: 10
$ echo "Count is now: $count"
Count is now: 10
```

**Step-by-step**: `count` unset → assigns "10", expands to "10". Now `count` is permanently 10.

**What if** you don't want to modify the variable? Use `:-` instead of `:=`.

### Example 3: Mandatory variable with `:?`

```bash
$ echo "${password:?No password provided}"
bash: password: No password provided
$ echo "This never runs"
```

In a script: `${1:?Usage: $0 filename}` exits with error if no argument.

**Can you catch it?** Not with `if`. Use a subshell to limit damage:
```bash
result=$(
    echo "${var:?is unset}"
    echo "this won't run"
) || echo "Var unset, but script continues"
```

### Example 4: Alternate value with `:+`

```bash
$ debug=1
$ echo "${debug:+Debug mode ON}"

Debug mode ON
$ debug=""
$ echo "${debug:+Debug mode ON}"
# (empty)
```

**Step-by-step**: `:+` is a toggle. If var is set AND non-null, expands to the word. Otherwise, expands to nothing.

**What if** you use `${debug+msg}` (no colon)? Then `debug=""` triggers it too.

### Example 5: Difference with and without colon on empty strings

```bash
$ var=""
$ echo "With colon: '${var:-fallback}'"
With colon: 'fallback'
$ echo "Without colon: '${var-fallback}'"
Without colon: ''
```

### Example 6: Nested defaults

```bash
$ unset FIRST SECOND
$ echo "${FIRST:-${SECOND:-fallback}}"
fallback
$ FIRST="first"
$ echo "${FIRST:-${SECOND:-fallback}}"
first
```

### Example 7: Defaults with positional parameters

```bash
$ greet() {
>   local name="${1:-Stranger}"
>   local greeting="${2:-Hello}"
>   echo "$greeting, $name!"
> }
$ greet
Hello, Stranger!
$ greet Bob
Hello, Bob!
```

### Example 8: Defaults in arithmetic contexts

```bash
$ threads=${JOBS:-4}
$ echo "Using $threads threads"
Using 4 threads
$ (( delay = ${WAIT:-5} ))
$ echo $delay
5
```

### Example 9: Debug/verbose toggle with `:+`

```bash
$ run_cmd() {
>   echo "${VERBOSE:+[DEBUG] Running: $*}"
>   "$@"
> }
$ VERBOSE=1 run_cmd ls
[DEBUG] Running: ls
# then ls output
```

### Example 10: Array element defaults

```bash
$ config=("host1" "" "host3")
$ echo "${config[0]:-default}"
host1
$ echo "${config[1]:-default}"
default
$ echo "${config[5]:-default}"   # unset element
default
```

### Example 11: Indirect expansion with defaults

```bash
$ var_name="PATH"
$ echo "${!var_name:-/usr/bin}"
/usr/local/bin:/usr/bin:/bin:...
$ var_name="NONEXISTENT"
$ echo "${!var_name:-default}"
default
```

### Example 12: The `: "${var:=value}"` idiom

```bash
$ : "${PORT:=8080}"
$ : "${HOST:=localhost}"
$ echo "$PORT | $HOST"
8080 | localhost
```

The `:` (null command) does nothing. The `${:=}` inside does the assignment as a side effect.

## Real-World Use Cases

### FOR the OS

- **Init scripts**: `DAEMON_OPTS=${DAEMON_OPTS:--d}`
- **Docker entrypoints**: `${VAR:?Please set VAR in docker-compose.yml}`
- **Login scripts**: `export PATH=$PATH:${GOPATH:+/usr/local/go/bin}`

### WITH the OS

- **getopt + parameter expansion**: Optional flags with defaults
- **Configuration management**: Source config file, override with env vars
- **Makefiles**: `CC=${CC:-gcc}`, `CFLAGS=${CFLAGS:--O2}`

### AGAINST the OS

- **Hardening**: `${MODE:?Must be dev or prod}`
- **Read-only**: `readonly MAX_RETRIES=${MAX_RETRIES:-3}`

### FOR DEFENSE

- **Reject unset config**: `${API_KEY:?API_KEY not set}`
- **Fail-fast**: Put `:?` guards early to stop execution before damage
- **Defensive defaults**: `${PATH:-/usr/bin:/bin}` to prevent unset PATH

## Memory Aids

- **`:-` = "Colon-minus = minus the default"**: The colon checks emptiness, the minus provides the fallback.
- **`:=` = "Colon-equals = assign the default"**: The equals signs hands the value to the variable.
- **`:?` = "Colon-question = scream if unset"**: Like a guard demanding an answer.
- **`:+` = "Colon-plus = add this extra"**: If the variable exists, add this content.
- **Colon vs no colon**: "Colon checks content" — the colon eyes the actual value.
- **No spaces**: `${var:-word}` is a single token. Spaces break it.

## Trap Vault (12 traps)

### Trap 1: Confusion between `-` and `:-` with empty strings

```bash
config=""
value=${config-fallback}    # → "" (empty!)
# FIX: Use ${config:-fallback}
```

### Trap 2: `:=` modifies the variable (side effect)

```bash
echo "${count:=10}"  # "10"
echo "$count"        # 10 — surprise, count was assigned!
# FIX: Use :- for one-time fallback
```

### Trap 3: `:?` exits the entire script

```bash
echo "${var:?err}"  # script EXITS HERE
# FIX: Only use :? for truly fatal conditions
```

### Trap 4: No spaces allowed around operators

```bash
echo "${var : - default}"  # syntax error
# FIX: No spaces: ${var:-default}
```

### Trap 5: `:?` doesn't show line number in interactive mode

In a script, it shows "script.sh: line 2: var: unset". In interactive, just the error.

### Trap 6: Default word is re-expanded and can have side effects

```bash
max=${MAX_PROCS:-$(count_procs)}  # count_procs RUNS even when MAX_PROCS is set!
# FIX: The word is always expanded first. Use a conditional if needed.
```

### Trap 7: `:=` on positional parameters in functions

```bash
myfunc() {
    : "${1:=default}"  # Error: $1 is read-only
}
# FIX: local first="${1:-default}"
```

### Trap 8: `${@:-default}` behavior with no args

```bash
echo "${@:-nothing}"  # prints "nothing" if no args
# With args, prints all args
```

### Trap 9: Defaults with arrays — only individual elements work

```bash
echo "${arr[@]:-empty}"  # This is SLICING syntax, not defaults!
# Use per-element: "${arr[0]:-default}"
```

### Trap 10: Word splitting of `$@` default

```bash
set -- "file with spaces.txt"
echo "${@:--default-}"  # Works, but the expansion is word-split
```

### Trap 11: Interactive shell doesn't exit on `:?`

```bash
echo "${var:?err}"
echo "Still runs in interactive mode."
# FIX: In a script it exits; in interactive it just prints.
```

### Trap 12: `:+` can't test `$?` directly

```bash
false
echo "${?:+failed}"  # $? is 0 by the time echo runs (echo resets it)
# FIX: Capture: result=$?; then check
```

## See It In The Wild

### Daily Encounters

- Docker entrypoints: `${1:?command required}`
- Homebrew: Heavy use of parameter expansion
- .bashrc: `HISTSIZE=${HISTSIZE:-1000}`
- CI/CD pipelines: defaults for optional inputs

### Exploration

1. `echo ${HOME:-/nonexistent}` — should print your home
2. `echo ${NONEXISTENT_VAR:-fallback}` — prints "fallback"
3. `bash -c 'echo "${A:?unset}"'` — see exit behavior

## Check Your Understanding (7 questions)

1. **What's the output?**: `x=""; echo "${x:-hello} ${x-hello}"`
2. **True or False**: `${var:=value}` modifies `var` even when `var` is already set to non-null.
3. **What happens when a script reaches `${MANDATORY_VAR:?}` without setting it?**
4. **Write a one-liner** that prints "Verbose mode" if `$DEBUG` is set and non-null.
5. **Why does `${1:-default}` work but `${1:=default}` fail in a function?**
6. **Difference between `${var:+yep}` and `[[ -n $var ]] && echo yep`?**
7. **Explain: `: "${var:=default}"`** — what each part does.

## Supplementary Deep Dive: Advanced Default Patterns

### Detecting "set but empty" vs "unset"

Sometimes you need to know if a variable was explicitly set to empty vs never set at all:

```bash
$ detect() {
>   local var="${1}"
>   # ${var+x} expands to "x" ONLY if var is set (even if empty)
>   # ${var:-unset} expands to "unset" if var is unset OR empty
>   if [[ -z "${var}" && "${var+x}" == "x" ]]; then
>     echo "Set but empty"
>   elif [[ -z "${var+x}" ]]; then
>     echo "Unset"
>   else
>     echo "Set: $var"
>   fi
> }
$ unset x; detect "$x"
Unset
$ x=""; detect "$x"
Set but empty
$ x="hello"; detect "$x"
Set: hello
```

### The `+` operator for checking existence (without colon)

```bash
$ unset x
$ echo "${x+EXISTS}"      # empty — x is unset
$ x=""
$ echo "${x+EXISTS}"      # EXISTS — x is set (even though empty)
EXISTS
$ x="hello"
$ echo "${x+EXISTS}"      # EXISTS — x is set and non-empty
EXISTS
```

### The `:+` operator for conditional defaults

```bash
# Print the value if set, otherwise "N/A"
echo "${var:-N/A}"

# Print "PROVIDED" if set and non-empty, otherwise "MISSING"
if [[ -n "${var:+x}" ]]; then
    echo "PROVIDED"
else
    echo "MISSING"
fi
```

### Positional parameter tricks

```bash
# Default values for multiple arguments in one go
first=${1:-default1}
second=${2:-default2}
third=${3:-default3}

# Require at least 2 of 3
: "${1:?Missing arg1}" "${2:?Missing arg2}"

# Shift + default pattern
arg="${1:?}"; shift
next="${1:-fallback}"
```

### Using :? with functions for parameter validation

```bash
myfunc() {
    local required="${1:?myfunc: arg1 required}"
    local optional="${2:-default}"
    echo "$required $optional"
}
```

### The empty default trick

```bash
# Reset to empty if unset/null
var="${var:-}"

# Set to empty string if unset (useful for arrays)
arr=(${arr[@]:-})
```

### Defaults with indirect expansion

```bash
$ set_default() {
>   local varname="$1"
>   local default="$2"
>   printf -v "$varname" "${!varname:-$default}"
> }
$ set_default MY_VAR "default_value"
$ echo "$MY_VAR"
default_value
```

### Combining defaults with command substitution

```bash
# Only run the expensive command if var is unset
val=$(command)
result="${var:-$val}"
# But $val is always computed! Better:
result="${var:-$(expensive_command)}"
```

### Using defaults in arithmetic contexts

```bash
# In (( )), defaults work differently
(( count = ${COUNT:-0} + 1 ))

# With let
let count=${COUNT:-0}+1
```

### Multi-level fallback chain

```bash
# Chain: VAR > CONFIG > DEFAULT
value="${VAR:-${CONFIG:-default}}"

# Or: try three sources
value="${VAR:-${CONFIG:-${FILE_VAL:-default}}}"
```

## Supplementary Deep Dive: Edge Cases and Advanced Defaults

### Combining default with assignment for idempotent scripts

```bash
$ # Only set if not already set — safe for sourced scripts
$ : "${MY_VAR:=default_value}"
$ # If sourced twice, won't overwrite user's custom value
```

### Using `${var:+alt}` for conditional logic

```bash
$ # Output "alt" only if var is set AND non-empty
$ name="Bob"
$ echo "${name:+Hello, $name!}"
Hello, Bob!
$ name=""
$ echo "${name:+Hello, $name!}"

$ # Useful for optional flags
$ verbose="--verbose"
$ echo "${verbose:+Verbose mode enabled}"
Verbose mode enabled
```

### `${var=default}` vs `${var:=default}` — the subtle difference

```bash
$ # With = (no colon): only assigns if var is UNSET (not if empty)
$ unset var
$ echo "${var=fallback}"  # sets because unset
fallback
$ echo "$var"
fallback

$ var=""
$ echo "${var=fallback}"  # does NOT set because var exists
                         # (empty, but exists)
$ echo "$var"
                         # still empty
```

### Chaining defaults for hierarchical configs

```bash
$ # User override → project config → global default
$ color="${USER_COLOR:-${PROJECT_COLOR:-${DEFAULT_COLOR:-blue}}}"
```

### Using default expansion in arithmetic context

```bash
$ # Ensure a number has a default in arithmetic
$ count=${COUNT:-10}
$ for ((i=0; i<count; i++)); do
>   echo "$i"
> done
```

### Default expansion with positional parameters

```bash
$ # Function with default arguments
$ greet() {
>   local name="${1:-World}"
>   local greeting="${2:-Hello}"
>   echo "$greeting, $name!"
> }
$ greet
Hello, World!
$ greet "Alice"
Hello, Alice!
$ greet "Bob" "Howdy"
Howdy, Bob!
```

### Default expansion for arrays

```bash
$ arr=()
$ echo "${arr[@]:-nothing}"
nothing
$ arr=("a" "b")
$ echo "${arr[@]:-nothing}"
a b
```

### Default expansion with command substitution

```bash
$ # Provide default if command fails
$ result=$(some_command) || result="default"
$ # Or inline:
$ result="${$(some_command):-default}"  # does NOT work — syntax error
$ # Correct:
$ result="$(some_command || echo "default")"
```

### Using ${!var} indirection with defaults

```bash
$ x="hello"
$ varname="x"
$ echo "${!varname:-not set}"
hello
$ varname="nonexistent"
$ echo "${!varname:-not set}"
not set
```

### Parameter expansion in heredocs

```bash
$ cat << EOF
> Your path is ${HOME:-/home/default}
> EOF
Your path is /home/phd
```

### Debugging with :+ for verbose logging

```bash
$ debug_mode=true
$ echo "${debug_mode:+[DEBUG] Starting process}"
[DEBUG] Starting process
$ # Toggle off:
$ debug_mode=false
$ echo "${debug_mode:+[DEBUG] Starting process}"
                      # nothing printed
```
