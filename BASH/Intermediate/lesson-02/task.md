# Task 2: Parameter Expansion — Defaults

## Overview

Write a script `setup_env.sh` that uses **only parameter expansion** (no `if`, no `[[ ]]`, no `test`) to handle all configuration. The goal is to handle every possible variable state — unset, null, set, and required — declaratively.

## Sub-tasks

### 1. Mandatory Argument

Require the first argument (project name). If missing, exit with `:?` and a clear error message.

```bash
project=${1:?Usage: $0 project_name}
```

<details>
<summary>Hint: Why :? exits</summary>
The `:?` operator writes the message to stderr and calls `exit 1` if the variable is unset or null. This is fail-fast behavior — the script never runs with missing required input.
</details>

**Multiple solution paths:**
- Path A (standard): `${1:?Usage: $0 project_name}`
- Path B (custom error): `${1:?Error: Project name required. Usage: $0 <name>}`
- Path C (no colon — only unset, not empty): `${1?Usage: $0 project_name}` — allows empty string arg

### 2. Environment Defaults

Set the following with defaults using `:-` for fallbacks:
- `PORT=${PORT:-8080}`
- `DB_HOST=${DB_HOST:-localhost}`
- `DB_NAME=${DB_NAME:-${project}_db}` (uses the project name)

```bash
PORT=${PORT:-8080}
DB_HOST=${DB_HOST:-localhost}
DB_NAME=${DB_NAME:-${project}_db}
```

<details>
<summary>Hint: Nested defaults</summary>
`DB_NAME=${DB_NAME:-${project}_db}` uses the already-assigned `$project` variable inside the default value. This is evaluated only if `DB_NAME` is unset/null — so if the user sets `DB_NAME` externally, `$project` is never expanded.
</details>

### 3. Verbose Mode Check

Check if `VERBOSE` is set and non-null, and if so print "Verbose mode enabled" using alternate value syntax `:+`.

```bash
echo "${VERBOSE:+Verbose mode enabled}"
```

<details>
<summary>Hint: :+ vs checking in if</summary>
`${VERBOSE:+Verbose mode enabled}` expands to nothing if `VERBOSE` is unset or empty. It's equivalent to: `if [[ -n $VERBOSE ]]; then echo "Verbose mode enabled"; fi` but in a single expression.
</details>

### 4. Config File Assignment

Set `CONFIG_FILE` to the first argument's value if provided, otherwise `config.ini`, and use `:=` so it persists.

```bash
CONFIG_FILE=${1:-config.ini}
# OR with :=
: "${CONFIG_FILE:=${1:-config.ini}}"
```

<details>
<summary>Hint: := modifies the variable</summary>
`${var:=value}` not only expands to `value` but also assigns `value` to `var`. This is useful when you want the variable to retain the default for later use. The `: "${var:=value}"` idiom uses the null command `:` to safely perform the side-effect assignment.
</details>

### 5. Formatted Output

Print all resolved values in a formatted block.

```bash
echo "Configuration:"
echo "  Project:       $project"
echo "  Port:          $PORT"
echo "  DB Host:       $DB_HOST"
echo "  DB Name:       $DB_NAME"
echo "  Config File:   ${CONFIG_FILE:-config.ini}"
```

### 6. Bonus: Override Detection

Add a section that detects whether each value was user-provided or defaulted:

```bash
echo "Override status:"
echo "  PORT:      ${PORT:+explicit}${PORT:-default}"
# This prints "explicit" if PORT is set, "default" if unset
```

### 7. Bonus: Further Defense

Add a mandatory check for the environment (dev/staging/prod) using another `:?`:

```bash
env=${ENV:?ENV must be set to dev, staging, or prod}
case $env in
    dev|staging|prod) ;;
    *) echo "Invalid ENV: $env" >&2; exit 1 ;;
esac
```

## Alternative Implementation: All in One Function

```bash
#!/bin/bash
setup_env() {
    project=${1:?Usage: $0 project_name}
    PORT=${PORT:-8080}
    DB_HOST=${DB_HOST:-localhost}
    DB_NAME=${DB_NAME:-${project}_db}
    CONFIG_FILE=${CONFIG_FILE:-config.ini}

    echo "${VERBOSE:+Verbose mode enabled}"

    cat <<EOF
Configuration:
  Project:       $project
  Port:          $PORT
  DB Host:       $DB_HOST
  DB Name:       $DB_NAME
  Config File:   $CONFIG_FILE
EOF
}

setup_env "$@"
```

## Expected Output

```
$ ./setup_env.sh
bash: 1: Usage: ./setup_env.sh project_name

$ ./setup_env.sh myapp
Configuration:
  Project:       myapp
  Port:          8080
  DB Host:       localhost
  DB Name:       myapp_db
  Config File:   config.ini

$ PORT=3000 DB_HOST=10.0.0.1 VERBOSE=1 ./setup_env.sh myapp
Verbose mode enabled
Configuration:
  Project:       myapp
  Port:          3000
  DB Host:       10.0.0.1
  DB Name:       myapp_db
  Config File:   config.ini

$ DB_NAME=custom_db ./setup_env.sh myapp
Configuration:
  ...
  DB Name:       custom_db
  ...

$ ./setup_env.sh ""    # empty first arg
bash: 1: Usage: ./setup_env.sh project_name  # because :? checks empty too!
```

## Self-Check

- What's the difference between `${var:-x}` and `${var-x}`?
- Does `${var:=value}` modify `var` in the parent shell or just in a subshell?
- What happens when `${var:?err}` triggers in a sourced script?
- Why is there no space between the variable name and `:-`?
- How would you provide a default of "unset" when a variable is empty?
- How would you detect if a variable was explicitly set (even to empty) vs unset?
- When would you choose `:=` over `:-`?

### Sub-task 8: Robust Configuration Loader

Write a function `load_config` that reads a `.env` file and exports each variable, but uses `${var:-default}` to provide sensible defaults for any variable that wasn't in the file.

Expected patterns:
```bash
$ cat > .env <<< "DB_HOST=localhost"
$ load_config ./.env
$ echo "${DB_HOST:-not set}"
localhost
$ echo "${DB_PORT:-5432}"
5432
$ echo "${DB_NAME:-myapp}"
myapp
```

### Sub-task 9: Default Expansion in Function Arguments (Bonus)

Rewrite `/etc/init.d/functions`-style `daemon()` replacement:

```bash
daemon() {
    local name="${1:-unknown}"
    local pidfile="${2:-/var/run/${name}.pid}"
    local user="${3:-root}"
    local cmd="${4:-false}"
    # start the daemon with these parameters
}
```

<details>
<summary>Hints</summary>

- Use `local` with defaults inside functions to avoid polluting global scope
- The colon in `:-` is your friend — it catches both unset AND empty
- For the .env parser, use `source` or `set -a +a` to auto-export
</details>
