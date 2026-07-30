# Task 14: Build a Function Library

Create a reusable function library and a script that uses it to perform system tasks. This mirrors how real system administrators organize their tools.

## Steps

### Sub-task 1: Create the Library (~/lib.sh)
Create `~/lib.sh` with these functions:

```bash
#!/bin/bash
# lib.sh - Reusable function library
# Source this file:  source lib.sh  or  . lib.sh

# Print error to stderr and exit
# Usage: die "message" [exit_code]
die() {
    echo "ERROR: $1" >&2
    exit "${2:-1}"
}

# Print usage info and exit with code 2
# Usage: usage
usage() {
    echo "Usage: $0 [options] <file>" >&2
    exit 2
}

# Ask a yes/no question
# Usage: if confirm "Proceed?"; then ...
confirm() {
    read -r -p "$1 [y/N] " response
    case "$response" in
        [yY]|[yY][eE][sS]) return 0 ;;
        *) return 1 ;;
    esac
}

# Check if running as root
# Usage: if is_root; then ... ; fi
is_root() {
    [ "$(id -u)" -eq 0 ]
}

# Back up a file with timestamp
# Usage: backup_file "/path/to/file"
backup_file() {
    local file="$1"
    local backup="${file}.bak.$(date +%Y%m%d-%H%M%S)"

    if [ ! -f "$file" ]; then
        echo "ERROR: File $file does not exist" >&2
        return 1
    fi

    if [ -e "$backup" ]; then
        echo "Backup for this minute already exists. Skipping."
        return 0
    fi

    cp "$file" "$backup"
    echo "Backed up $file to $backup"
}

# Check if a command exists
# Usage: if command_exists "curl"; then ... ; fi
command_exists() {
    command -v "$1" &>/dev/null
}

# Get file size in human-readable format
# Usage: file_size "/path/to/file"
file_size() {
    local file="$1"
    if [ -f "$file" ]; then
        du -h "$file" | cut -f1
    elif [ -d "$file" ]; then
        du -sh "$file" | cut -f1
    else
        echo "0"
    fi
}
```

**Approach 1:** All functions in one file as shown
**Approach 2:** Split into `~/lib/` directory: `lib/log.sh`, `lib/file.sh`, `lib/network.sh`
**Approach 3:** Use `BASH_SOURCE` detection to prevent direct execution

<details><summary>Hint: Prevent direct execution of library</summary>
```bash
# At the top of lib.sh:
if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
    echo "This file should be sourced, not executed." >&2
    exit 1
fi
```
This ensures `source lib.sh` works but `bash lib.sh` or `./lib.sh` fails.
</details>

### Sub-task 2: Create a Consumer Script
Create `~/system_check.sh` that sources the library and uses ALL functions:

```bash
#!/bin/bash
# system_check.sh - System maintenance tool

# Source the library
SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
source "${SCRIPT_DIR}/lib.sh" 2>/dev/null || source ~/lib.sh

# Check arguments
if [ $# -lt 1 ]; then
    echo "Usage: $0 [--force] <file_or_directory>" >&2
    exit 2
fi

# Parse options
force=0
if [ "$1" = "--force" ]; then
    force=1
    shift
fi

target="$1"

# Check if target exists
if [ ! -e "$target" ]; then
    die "Target $target does not exist"
fi

# Check root status
if is_root; then
    echo "Running as root - full access"
else
    echo "Running as non-root - limited access"
    if [ ! -r "$target" ]; then
        die "Cannot read $target"
    fi
fi

# If it's a file, offer backup
if [ -f "$target" ] && [ $force -eq 0 ]; then
    if confirm "Back up $target?"; then
        backup_file "$target"
    else
        echo "Skipping backup."
    fi
fi

# Show file info
echo "=== Target: $target ==="
echo "Type: $([ -f "$target" ] && echo "file" || [ -d "$target" ] && echo "directory" || echo "other")"
echo "Size: $(file_size "$target")"
echo "Readable: $([ -r "$target" ] && echo "yes" || echo "no")"
echo "Writable: $([ -w "$target" ] && echo "yes" || echo "no")"

# Check for required tools
for cmd in stat du date; do
    if command_exists "$cmd"; then
        echo "  $cmd: available"
    else
        echo "  $cmd: MISSING!" >&2
    fi
done

echo "Done."
```

**Approach 1:** Source from same directory
**Approach 2:** Source from `~/.bash_libs/` standard location
**Approach 3:** Add lib directory to `BASH_ENV` for automatic sourcing

<details><summary>Hint: Dynamic sourcing path</summary>
```bash
# Try multiple locations for the library
for libdir in "$(dirname "$0")" ~/lib ~/scripts ~; do
    libfile="$libdir/lib.sh"
    [ -f "$libfile" ] && source "$libfile" && break
done
```
</details>

### Sub-task 3: Test All Functions

```bash
# Test 1: Run without arguments (should show usage)
$ chmod +x ~/lib.sh ~/system_check.sh
$ ~/system_check.sh
Usage: system_check.sh [--force] <file_or_directory>
$ echo $?
2

# Test 2: Run on a file, answer "n" to backup
$ ~/system_check.sh /etc/hosts
Running as non-root - limited access
Back up /etc/hosts? [y/N] n
Skipping backup.
=== Target: /etc/hosts ===
Type: file
Size: 320
Readable: yes
Writable: no
Done.

# Test 3: Run on a file, answer "y" to backup
$ ~/system_check.sh /etc/hostname
Running as non-root - limited access
Back up /etc/hostname? [y/N] y
Backed up /etc/hostname to /etc/hostname.bak.20260731-010203
=== Target: /etc/hostname ===
...

# Test 4: Test die with non-existent file
$ ~/system_check.sh /nonexistent
ERROR: Target /nonexistent does not exist
$ echo $?
1
```

### Sub-task 4: Demonstrate `local` Variable Scoping
Add a function to your library that demonstrates the difference between local and global:

```bash
# demo_local.sh
#!/bin/bash

# Function WITHOUT local — modifies global
bad_func() {
    counter=10
    echo "Inside bad_func: counter=$counter"
}

# Function WITH local — doesn't affect global
good_func() {
    local counter=20
    echo "Inside good_func: counter=$counter"
}

counter=1
echo "Before: counter=$counter"

bad_func
echo "After bad_func: counter=$counter"

good_func
echo "After good_func: counter=$counter"
```

Expected output:
```bash
$ source ~/lib.sh && source ~/demo_local.sh
Before: counter=1
Inside bad_func: counter=10
After bad_func: counter=10       # Modified!
Inside good_func: counter=20
After good_func: counter=10      # Still 10 from bad_func, good_func didn't touch it
```

### Sub-task 5: Add Argument Validation
Add input validation to your functions:

```bash
# Validate that arguments are provided
validate_args() {
    local func_name="$1"
    local min_args="$2"
    shift 2

    if [ $# -lt "$min_args" ]; then
        echo "ERROR: $func_name requires at least $min_args arguments (got $#)" >&2
        return 1
    fi
    return 0
}

# Usage inside other functions:
backup_file() {
    validate_args "backup_file" 1 "$@" || return 1
    local file="$1"
    ...
}
```

**Bonus:** Add a `debug_log` function that only prints when `DEBUG=1`:
```bash
debug_log() {
    [ "${DEBUG:-0}" -eq 1 ] && echo "DEBUG: $*" >&2
}
```

### Sub-task 6: Integration Test Script
Run a comprehensive test:

```bash
#!/bin/bash
source ~/lib.sh

echo "=== Testing library functions ==="
echo

# Test is_root
echo -n "is_root: "
is_root && echo "yes" || echo "no"

# Test confirm
echo "Testing confirm (answer y then n):"
echo "y" | confirm "Test?" && echo "  'y' -> yes" || echo "  'y' -> no"
echo "n" | confirm "Test?" && echo "  'n' -> yes" || echo "  'n' -> no"

# Test backup_file
echo
echo "Testing backup_file:"
tmpfile=$(mktemp)
echo "test data" > "$tmpfile"
backup_file "$tmpfile"
backup_file "$tmpfile"  # Second call should say "already exists"

# Test file_size
echo
echo "Testing file_size:"
file_size "$tmpfile"
file_size /

# Test command_exists
echo
echo -n "curl installed: "
command_exists curl && echo "yes" || echo "no"
echo -n "nonexistent_cmd installed: "
command_exists nonexistent_cmd && echo "yes" || echo "no"

# Cleanup
rm -f "$tmpfile" "$tmpfile.bak."*
echo
echo "=== All tests complete ==="
```

## Expected Output Summary

```bash
$ ~/system_check.sh
Usage: system_check.sh [--force] <file_or_directory>
$ echo $?
2

$ ~/system_check.sh /etc/hosts
Running as non-root - limited access
Back up /etc/hosts? [y/N] n
Skipping backup.

$ ~/system_check.sh /etc/hostname
Back up /etc/hostname? [y/N] y
Backed up /etc/hostname to /etc/hostname.bak.20260731-010203

$ source ~/lib.sh
$ backup_file /etc/hostname
Backed up /etc/hostname to /etc/hostname.bak.20260731-010204
$ backup_file /etc/hostname
Backup for this minute already exists. Skipping.
```

## Self-Check Questions

1. What is the difference between `return` and `exit` in a function?

2. Why should you use `local` variables in functions?

3. What does `$#` represent inside a function?

4. What is the maximum value `return` can return?

5. Can you call a function before it's defined in a script? Why or why not?

6. How do you prevent a library script from being executed directly?

7. What's the `command` builtin used for when shadowing commands with functions?
