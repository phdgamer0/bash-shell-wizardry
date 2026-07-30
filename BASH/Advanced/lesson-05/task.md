# Task 5: CLI Tool with getopts

## Objective
Build a production-grade CLI tool that uses `getopts` to parse flags and arguments, handle errors gracefully, support combined flags, validate required options, and process positional arguments. This tool will serve as the foundation for a more complex subcommand-based tool in Lesson 6.

## Requirements

1. **Flags:** `-v` (verbose), `-o FILE` (output file, mandatory), `-n COUNT` (count, optional with default), `-f` (force), `-h` (help)
2. **Error handling:** Show custom error messages (use silent mode) with proper exit codes
3. **Combined flags:** `-vf` should work (verbose + force)
4. **Help text:** `-h` prints usage and exits with status 0
5. **Post-processing:** `shift` remaining args, operate on them
6. **Validation:** Check that required options are provided, validate COUNT is a positive integer
7. **Exit codes:** Use meaningful exit codes (0=success, 1=general error, 2=invalid option)

## Sub-tasks (8 cumulative)

### Task 5.1: Basic Option Parsing Skeleton

Build the minimal parsing skeleton:

```bash
#!/bin/bash
verbose=0
force=0
count=10
output=""

OPTIND=1
while getopts ":hvo:n:f" opt; do
  case $opt in
    h) show_help; exit 0 ;;
    v) verbose=1 ;;
    o) output="$OPTARG" ;;
    n) count="$OPTARG" ;;
    f) force=1 ;;
    :) echo "Error: -$OPTARG requires an argument" >&2; exit 1 ;;
    \?) echo "Error: Unknown option -$OPTARG" >&2; exit 1 ;;
  esac
done
shift $((OPTIND - 1))
```

**Multiple approaches compared:**

```bash
# Approach A: Silent mode with leading : (as above)
#   Pro: Custom error messages, full control
#   Con: More code (need to handle : and ? cases)

# Approach B: Default mode (no leading :)
#   Pro: Less code, bash prints errors
#   Con: Can't customize messages, always prints to stderr
while getopts "hvo:n:f" opt; do
  case $opt in
    h) show_help; exit 0 ;;
    v) verbose=1 ;;
    o) output="$OPTARG" ;;
    n) count="$OPTARG" ;;
    f) force=1 ;;
    *) echo "Try -h for help" >&2; exit 1 ;;
  esac
done
```

### Task 5.2: Validate Required Options

After parsing, check that required options were provided:

```bash
validate_required() {
  local errors=0
  if [[ -z "$output" ]]; then
    echo "Error: -o (output file) is required" >&2
    ((errors++))
  fi
  if [[ ! "$count" =~ ^[0-9]+$ ]] || [[ "$count" -lt 1 ]]; then
    echo "Error: -n must be a positive integer (got: $count)" >&2
    ((errors++))
  fi
  return $errors
}

validate_required || exit 1
```

**Edge case considerations:**
- What if `-o ""` is passed? `$output` is empty — caught by `-z "$output"`.
- What if `-n 0`? Caught by `$count -lt 1`.
- What if `-n abc`? Caught by the regex `^[0-9]+$`.

### Task 5.3: Help Text with Here Document

```bash
show_help() {
  cat << EOF
Usage: $(basename "$0") [OPTIONS] <input>...

Process input files with configurable options.

Options:
  -v          Verbose output (show progress messages)
  -o FILE     Output file path (required)
  -n COUNT    Number of iterations (default: 10)
  -f          Force overwrite of existing output
  -h          Show this help message and exit

Arguments:
  input       One or more input files to process

Exit codes:
  0  Success
  1  Validation/processing error
  2  Invalid option

Examples:
  $(basename "$0") -v -o result.txt file1.txt
  $(basename "$0") -o result.txt -n 5 file1.txt file2.txt
  $(basename "$0") -h
EOF
}
```

### Task 5.4: Process Input Files

Implement the processing logic:

```bash
process_file() {
  local file="$1"
  local line_num=0
  
  if [[ ! -f "$file" ]]; then
    echo "Error: File not found: $file" >&2
    return 1
  fi
  
  while IFS= read -r line; do
    ((line_num++))
    if ((verbose)); then
      echo "[$file:$line_num] Processing..." >&2
    fi
    echo "$line_num: $line"
  done < "$file"
}

# Main loop over positional arguments
if [[ $# -eq 0 ]]; then
  echo "Error: No input files specified" >&2
  echo "Usage: $(basename "$0") [OPTIONS] <input>..." >&2
  exit 1
fi

for file in "$@"; do
  process_file "$file" >> "$output"
done
```

### Task 5.5: Verbose Logging Function

```bash
log() {
  local level="$1"
  shift
  if ((verbose)) || [[ "$level" == "ERROR" ]]; then
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $*" >&2
  fi
}

# Usage:
log INFO "Starting processing"
log ERROR "File not found: $file"
log DEBUG "Current count: $count"  # Only if verbose=1
```

### Task 5.6: Force Flag Behavior

Implement the `-f` flag semantics:

```bash
if [[ -f "$output" ]] && ((!force)); then
  echo "Error: Output file exists and -f not specified: $output" >&2
  exit 1
elif [[ -f "$output" ]] && ((force)); then
  log WARN "Overwriting existing output: $output"
fi

# Truncate or create output file
: > "$output"  # Empty the file (or create if not exists)
```

### Task 5.7: Multiple getopts Invocations

Refactor into functions that each use `local OPTIND`:

```bash
parse_global_opts() {
  local OPTIND OPTARG opt
  while getopts ":hvf" opt; do
    case $opt in
      h) show_help; exit 0 ;;
      v) verbose=1 ;;
      f) force=1 ;;
      \?) return 2 ;;
    esac
  done
  return 0
}

parse_io_opts() {
  local OPTIND OPTARG opt
  while getopts ":o:n:" opt; do
    case $opt in
      o) output="$OPTARG" ;;
      n) count="$OPTARG" ;;
      \?) return 2 ;;
    esac
  done
}

# Usage:
parse_global_opts "$@"
shift $((OPTIND - 1))
parse_io_opts "$@"
shift $((OPTIND - 1))
```

**Why separate functions?** Modularity, testability, and reuse in subcommand contexts.

### Task 5.8: Integration and Testing

Create a test script that exercises all flags:

```bash
#!/bin/bash
# test_mytool.sh

echo "=== Test 1: Help ==="
./mytool.sh -h && echo "PASS" || echo "FAIL"

echo "=== Test 2: Basic operation ==="
echo -e "line1\nline2\nline3" > /tmp/test_input.txt
./mytool.sh -v -o /tmp/test_output.txt -n 3 /tmp/test_input.txt
cat /tmp/test_output.txt

echo "=== Test 3: Combined flags ==="
./mytool.sh -vf -o /tmp/test_out2.txt /tmp/test_input.txt

echo "=== Test 4: Missing required option ==="
./mytool.sh -v /tmp/test_input.txt && echo "FAIL" || echo "PASS (expected error)"

echo "=== Test 5: Invalid option ==="
./mytool.sh -Z -o /tmp/out.txt /tmp/test_input.txt && echo "FAIL" || echo "PASS (expected error)"

echo "=== Test 6: Force overwrite ==="
./mytool.sh -f -o /tmp/test_out2.txt /tmp/test_input.txt && echo "PASS" || echo "FAIL"

echo "=== Test 7: No input files ==="
./mytool.sh -o /tmp/out.txt && echo "FAIL" || echo "PASS (expected error)"

echo "=== Test 8: Invalid count ==="
./mytool.sh -o /tmp/out.txt -n abc /tmp/test_input.txt && echo "FAIL" || echo "PASS (expected error)"
```

## Bonus Challenges

1. **Bonus A:** Add a `--` separator handler that treats everything after `--` strictly as positional arguments, even if they look like options.

2. **Bonus B:** Implement a `-q` (quiet) flag that overrides `-v` — if both are specified, quiet wins.

3. **Bonus C:** Add colorized output using ANSI escape codes when stdout is a terminal (check with `-t`).

4. **Bonus D:** Implement progress reporting: `-p` shows a progress bar for large file processing. Use `\r` (carriage return) for inline updates.

5. **Bonus E:** Create a configuration file fallback — if `-c config.conf` is given, load default values from the file, which can be overridden by command-line flags.

## Hints

<details>
<summary>Hint 1: Silent mode optstring</summary>

```bash
while getopts ":hvo:n:f" opt; do
# Leading : = silent mode
# h, v, f = flags
# o:, n: = require arguments
```
</details>

<details>
<summary>Hint 2: Default values</summary>

```bash
count=10   # default
output=""  # required — check later
force=0    # default
verbose=0  # default
```
</details>

<details>
<summary>Hint 3: Counting lines in a file</summary>

`wc -l "$file"` for total lines; `wc -l < "$file"` to avoid printing filename.
</details>

<details>
<summary>Hint 4: Checking for file existence</summary>

```bash
[[ -f "$output" ]]  # True if file exists (regular file)
[[ -e "$output" ]]  # True if exists (any type)
```
</details>

<details>
<summary>Hint 5: Integer validation regex</summary>

```bash
[[ "$count" =~ ^[0-9]+$ ]]  # True if count is all digits
```
</details>

## Expected Output

```bash
$ ./mytool.sh -h
Usage: mytool.sh [OPTIONS] <input>...
Process input files with configurable options.

Options:
  -v          Verbose output (show progress messages)
  -o FILE     Output file path (required)
  -n COUNT    Number of iterations (default: 10)
  -f          Force overwrite of existing output
  -h          Show this help message and exit

$ ./mytool.sh -v -o result.txt -n 5 file1.txt
[2026-07-31 10:00:00] [INFO] Starting processing
[2026-07-31 10:00:00] [INFO] Processing file1.txt...
[2026-07-31 10:00:00] [INFO] Wrote 15 lines to result.txt

$ ./mytool.sh -o result.txt file1.txt
[2026-07-31 10:00:00] [INFO] Processing file1.txt...
(no verbose output on stderr)

$ ./mytool.sh
Error: -o (output file) is required
Usage: mytool.sh [OPTIONS] <input>...

$ ./mytool.sh -o result.txt
Error: No input files specified
Usage: mytool.sh [OPTIONS] <input>...

$ ./mytool.sh -vf -o result.txt file1.txt
[2026-07-31 10:00:00] [WARN] Overwriting existing output: result.txt
[2026-07-31 10:00:00] [INFO] Processing file1.txt...
```

## Deep Self-Check

1. **OPTIND scope debugging:** Write a function that calls another function using getopts. What happens if the inner function doesn't localize OPTIND? Trace the values.

2. **`--` handling test:** Run `mytool.sh -v -- -o file.txt`. What does getopts do with `--`? What is `$OPTIND` after the loop?

3. **Empty argument test:** `mytool.sh -o "" -n 5 file.txt` — how does getopts handle an empty string as an argument? Is `$OPTARG` empty or is the next argument consumed?

4. **Exit code verification:** Write a test that checks exit codes for each error condition. Use `$?` explicitly.

5. **Race condition test:** Run `mytool.sh -o result.txt file.txt` while another instance is also writing to result.txt. What happens with and without `-f`?

6. **Strace analysis:** Run `strace -e trace=write ./mytool.sh -v -o /tmp/out.txt /tmp/test.txt 2>&1 | grep -c "write(1"` — count how many times the tool writes to stdout vs stderr.

7. **Memory usage test:** Process a 100MB file with and without verbose mode. Use `/usr/bin/time -v` to compare maximum resident set size.

8. **getopts vs manual parsing performance:** Compare `while getopts` with `while [[ $1 == -* ]]; case $1 in ...` on 100000 arguments. Which is faster?

9. **Symlink output test:** What if `-o` specifies a symlink? Does the tool follow it? Should it?

10. **Signal handling test:** What happens if `mytool.sh` receives SIGINT during processing? Does it leave a partial output file?
