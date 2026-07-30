# Lesson 18: Capstone — CLI Tool

## History & Origins

The command-line interface (CLI) tool tradition in Unix dates to the 1970s when Thompson and Ritchie built the first Unix tools as small, composable programs. Each tool did one thing well and communicated via text streams. This philosophy — encoded in the Unix Philosophy — directly shaped how professional CLI tools are built today.

The `getopts` builtin appeared in the Bourne shell (1979) and was standardized by POSIX. Before getopts, scripts parsed `$1`, `$2`, etc. manually with case statements — a fragile approach. The GNU `getopt` (1990s) added support for long options (--help, --verbose) and option bundling. Modern CLI tools like `git`, `docker`, and `kubectl` popularized the subcommand pattern (e.g., `git commit`, `docker run`) which this capstone implements.

Bash itself evolved from the Bourne shell (sh) created by Stephen Bourne at Bell Labs. Brian Fox wrote Bash in 1987 for the GNU Project. Today, Bash is the default shell on virtually every Linux distribution and macOS, making Bash CLI tools universally portable across Unix-like systems.

## Syntax Reference

### Argument Parsing: getopts (POSIX)

```bash
getopts optstring varname [args]
```

- `optstring`: Letters for valid options. A colon after a letter means it takes an argument (e.g., `:ab:c` means `-a`, `-b ARG`, `-c`).
- Leading colon in optstring suppresses getopts' own error messages.
- `OPTARG`: The argument value for options that take one.
- `OPTIND`: Index of the next argument (reset to 1 to re-parse).

### Argument Parsing: GNU getopt (long options)

```bash
options=$(getopt -o "ho:v" -l "help,output:,verbose" -- "$@") || exit 1
eval set -- "$options"
```

- `-o`: Short options (same format as getopts)
- `-l`: Long options (comma-separated, colon for required arg)
- `--`: Signals end of options
- `eval set -- "$options"` re-positions the arguments

### Subcommand Dispatch Pattern

```bash
case "${1:-help}" in
  init)   shift; source "commands/init.sh" "$@" ;;
  run)    shift; source "commands/run.sh" "$@" ;;
  help)   show_help ;;
  *)      error "Unknown command: $1"; exit 1 ;;
esac
```

### Source vs Execute

- `source file.sh` (or `. file.sh`) — runs in current shell, shares variables
- `bash file.sh` — runs in a subshell, isolated scope
- Subcommands are usually `source`d to access shared functions and state

### Config File Sourcing

```bash
for path in "${CONFIG_PATHS[@]}"; do
  [[ -f "$path" ]] && source "$path" && return 0
done
```

Sourcing config files is idiomatic in Bash — the config file IS Bash syntax.

### Colored Output Detection

```bash
if [[ -t 1 ]]; then
  RED='\033[0;31m'; NC='\033[0m'
else
  RED=''; NC=''
fi
```

`-t 1` checks if file descriptor 1 (stdout) is a terminal.

## Under the Hood

When you run `./mytool.sh init --verbose`, the shell:
1. Forks a new process via `fork()`
2. Executes the script via `execve()`
3. The shebang (`#!/bin/bash`) tells the kernel to use `/bin/bash` as interpreter
4. Bash reads the script, expands variables, parses commands
5. `case "${1:-help}"` matches `init`
6. `shift` drops the subcommand from `$@`, leaving `--verbose`
7. `source "commands/init.sh"` reads and executes init.sh in the same shell
8. Inside init.sh, `getopts` processes `--verbose`

The sourcing mechanism (`source` or `.`) does NOT fork a new process — it reads the file and executes commands in the current shell environment. This is why shared variables, functions, and trap handlers set in `main.sh` are available in subcommand files.

When config files are sourced, arbitrary code execution is possible. This is by design — Bash config files are scripts. The security implication is that config files must be trusted.

The `shift` builtin re-indexes positional parameters. `shift N` discards the first N parameters, shifting remaining ones down. This is how subcommand dispatch removes the command name before passing control to the subcommand implementation.

## Core Examples

### Example 1: Minimal Getopts Skeleton

```bash
#!/bin/bash
verbose=false
output_file=""

while getopts ":vo:" opt; do
  case $opt in
    v) verbose=true ;;
    o) output_file=$OPTARG ;;
    \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
    :) echo "Option -$OPTARG requires an argument" >&2; exit 1 ;;
  esac
done
shift $((OPTIND - 1))

$verbose && echo "Verbose mode on"
echo "Output file: ${output_file:-stdout}"
echo "Remaining args: $*"
```

### Example 2: Long Options with GNU getopt

```bash
#!/bin/bash
options=$(getopt -o "o:v" -l "output:,verbose,help" -- "$@") || {
  echo "Usage: $0 [-o file] [-v] [--output file] [--verbose] [--help]" >&2
  exit 1
}
eval set -- "$options"

while [[ $# -gt 0 ]]; do
  case $1 in
    -o|--output) output=$2; shift 2 ;;
    -v|--verbose) verbose=true; shift ;;
    --help) show_help; exit 0 ;;
    --) shift; break ;;
  esac
done
```

⚠️ **TRAP:** Always check the exit code of `getopt` itself. It exits non-zero on invalid options. Use `|| exit 1` after the getopt call.

### Example 3: Help Text with Here-Doc

```bash
show_help() {
  cat <<-EOF
Usage: $(basename "$0") <command> [options]

Commands:
  init    Initialize a new project
  run     Execute the main process
  status  Show current status
  help    Show this help message

Options:
  -h, --help    Show help for any command
  -v, --verbose Verbose output  
  -c, --config  Path to config file

Report bugs to: https://github.com/example/mytool/issues
EOF
}
```

The `<<-EOF` form strips leading tabs, allowing indented here-docs.

### Example 4: Shared Library — Logging

```bash
# lib/logging.sh
LOG_LEVEL=${LOG_LEVEL:-info}
declare -A LEVELS=([debug]=0 [info]=1 [warn]=2 [error]=3)

log() {
  local level=$1 msg=$2
  [[ ${LEVELS[$level]} -lt ${LEVELS[$LOG_LEVEL]} ]] && return
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] [$level] $msg" >&2
}
```

Usage: `log info "Starting process"` outputs to stderr with timestamp.

### Example 5: Shared Library — Colored Output

```bash
# lib/output.sh
if [[ -t 1 ]]; then
  BOLD='\033[1m'; RED='\033[0;31m'
  GREEN='\033[0;32m'; YELLOW='\033[1;33m'
  BLUE='\033[0;34m'; NC='\033[0m'
else
  BOLD=''; RED=''; GREEN=''; YELLOW=''; BLUE=''; NC=''
fi

info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
warn()  { echo -e "${YELLOW}[WARN]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*" >&2; }
success() { echo -e "${GREEN}[OK]${NC} $*"; }
```

### Example 6: Progress Bar

```bash
progress_bar() {
  local current=$1 total=$2 prefix=${3:-Progress}
  local percent=$((current * 100 / total))
  local filled=$((percent / 2))
  local empty=$((50 - filled))
  printf "\r%s [%s%s] %d%%" \
    "$prefix" \
    "$(printf '=%.0s' $(seq 1 $filled))" \
    "$(printf ' %.0s' $(seq 1 $empty))" \
    "$percent"
  [[ $current -eq $total ]] && echo
}
```

### Example 7: Config File with Validation

```bash
# lib/config.sh
CONFIG_PATHS=(
  ./mytool.conf
  "${XDG_CONFIG_HOME:-$HOME/.config}/mytool/mytool.conf"
  /etc/mytool/mytool.conf
)

load_config() {
  for path in "${CONFIG_PATHS[@]}"; do
    if [[ -f "$path" ]]; then
      if grep -qP '^\s*(export\s+)?[A-Z_][A-Z0-9_]*=' "$path" 2>/dev/null; then
        source "$path"
        log info "Loaded config: $path"
        return 0
      else
        log warn "Config file has invalid syntax: $path"
        return 1
      fi
    fi
  done
  log warn "No config file found, using defaults"
  return 1
}
```

### Example 8: Error Handler with Line Number

```bash
error_handler() {
  local line=$1 code=$2
  error "Error at line $line (exit code $code)"
  error "Last command: $BASH_COMMAND"
  exit "$code"
}
trap 'error_handler $LINENO $?' ERR
```

### Example 9: Subcommand with Getopts

```bash
# commands/run.sh — sourced by main.sh
verbose=${verbose:-false}
count=${count:-1}
threshold=${threshold:-50}

while getopts ":vc:t:" opt; do
  case $opt in
    v) verbose=true ;;
    c) count=$OPTARG ;;
    t) threshold=$OPTARG ;;
    \?) error "Invalid option: -$OPTARG"; exit 1 ;;
  esac
done
shift $((OPTIND - 1))

$verbose && info "Running with count=$count, threshold=$threshold"
for ((i=0; i<count; i++)); do
  result=$((RANDOM % 100))
  if [[ $result -gt $threshold ]]; then
    info "Iteration $((i+1)): Result=$result (ABOVE threshold)"
  else
    info "Iteration $((i+1)): Result=$result (below threshold)"
  fi
done
```

⚠️ **TRAP:** In a sourced subcommand, global variables from main.sh are visible. Never redeclare them with `local` unless you intend to shadow them.

### Example 10: Dry-Run Mode

```bash
DRY_RUN=${DRY_RUN:-false}

run_or_dry() {
  local desc=$1
  shift
  if $DRY_RUN; then
    info "[DRY-RUN] $desc: $*"
  else
    "$@"
    log info "$desc: $*"
  fi
}

# Usage
run_or_dry "Moving file" mv "$src" "$dst"
run_or_dry "Creating directory" mkdir -p "$new_dir"
```

### Example 11: Undo Log System

```bash
UNDO_LOG="${XDG_DATA_HOME:-$HOME/.local/share}/mytool/undo.log"

record_action() {
  local action=$1 source=$2 dest=$3
  mkdir -p "$(dirname "$UNDO_LOG")"
  printf '%s|%s|%s|%s\n' "$(date +%s)" "$action" "$source" "$dest" >> "$UNDO_LOG"
}

undo_last() {
  [[ ! -f "$UNDO_LOG" ]] && { error "Nothing to undo"; return 1; }
  local last_action
  last_action=$(tail -1 "$UNDO_LOG")
  local ts action src dst
  IFS='|' read -r ts action src dst <<< "$last_action"
  info "Undoing: $action $src -> $dst"
  mv "$dst" "$src" && {
    sed -i '$d' "$UNDO_LOG"
    success "Undo complete"
  }
}
```

### Example 12: JSON Output Mode

```bash
OUTPUT_FORMAT=${OUTPUT_FORMAT:-text}

emit_result() {
  local status=$1 message=$2 data=${3:-}
  if [[ $OUTPUT_FORMAT == "json" ]]; then
    printf '{"status":"%s","message":"%s","data":%s}\n' \
      "$status" "$message" "${data:-null}"
  else
    echo "[$status] $message"
  fi
}
```

### Example 13: Main Entry Point with All Features

```bash
#!/bin/bash
set -euo pipefail

SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd)
source "$SCRIPT_DIR/lib/logging.sh"
source "$SCRIPT_DIR/lib/output.sh"
source "$SCRIPT_DIR/lib/config.sh"

load_config

trap 'error_handler $LINENO $?' ERR
trap 'log info "Interrupted by user"; exit 1' INT TERM

show_help() {
  cat <<-HELP
Usage: $(basename "$0") <command> [options]

Commands:
  init    Initialize project structure
  run     Execute main process
  status  Show current state
  undo    Revert last operation
  help    Show this message

Options:
  -v, --verbose  Enable verbose logging
  -c, --config   Config file path
  -f, --format   Output format (text|json)
HELP
}

case "${1:-help}" in
  init|run|status|undo)
    cmd=$1; shift
    source "$SCRIPT_DIR/commands/${cmd}.sh" "$@"
    ;;
  help|--help|-h) show_help ;;
  *) error "Unknown command: $1"; exit 1 ;;
esac
```

⚠️ **TRAP:** `set -euo pipefail` in main.sh applies to sourced subcommands too. If a subcommand contains a command that returns non-zero (like `grep` with no match), the entire script exits. Use `|| true` or `grep -q ... || [[ $? -eq 1 ]]` to handle expected failures.

### Example 14: File Size Calculation

```bash
format_size() {
  local bytes=$1
  if ((bytes >= 1073741824)); then
    printf "%.2f GB" "$(echo "scale=2; $bytes / 1073741824" | bc)"
  elif ((bytes >= 1048576)); then
    printf "%.2f MB" "$(echo "scale=2; $bytes / 1048576" | bc)"
  elif ((bytes >= 1024)); then
    printf "%.2f KB" "$(echo "scale=2; $bytes / 1024" | bc)"
  else
    printf "%d B" "$bytes"
  fi
}
```

### Example 15: File Organization by Type

```bash
ORGANIZE_DIR=${1:-.}
declare -A EXT_MAP=(
  [pdf]=Documents [doc]=Documents [docx]=Documents
  [jpg]=Images [jpeg]=Images [png]=Images [gif]=Images
  [mp3]=Music [wav]=Music [flac]=Music
  [mp4]=Videos [avi]=Videos [mkv]=Videos
  [sh]=Scripts [py]=Scripts [pl]=Scripts
)

organize_file() {
  local file=$1
  local ext="${file##*.}"
  local target_dir="${EXT_MAP[$ext]}"
  [[ -z $target_dir ]] && target_dir="Misc"
  
  local dest="${ORGANIZE_DIR}/${target_dir}/$(basename "$file")"
  [[ "$file" == "$dest" ]] && return
  
  mkdir -p "${ORGANIZE_DIR}/${target_dir}"
  if $DRY_RUN; then
    info "[DRY-RUN] $file -> $dest"
  else
    mv "$file" "$dest"
    record_action "move" "$file" "$dest"
  fi
}
```

## Real-World Use Cases

1. **Git** — The most famous subcommand-based CLI tool. `git commit`, `git push`, `git log`. Each subcommand is a separate Perl/shell script in `libexec/git-core/`.

2. **Docker CLI** — Uses the same subcommand pattern: `docker run`, `docker build`, `docker ps`. Config files in `~/.docker/config.json`.

3. **Kubernetes kubectl** — `kubectl apply`, `kubectl get pods`, `kubectl describe`. Supports JSON and YAML output with `-o json`.

4. **Vagrant** — `vagrant up`, `vagrant ssh`, `vagrant destroy`. Uses Ruby but the CLI architecture mirrors what we build here.

5. **AWS CLI v1** — Originally a Bash/Python hybrid with subcommands, config file in `~/.aws/config`, and colored output.

6. **Homebrew (Linuxbrew)** — `brew install`, `brew update`, `brew doctor`. Written in Ruby but follows the same subcommand dispatch pattern.

7. **System administration tools** — `systemctl`, `journalctl`, `firewall-cmd` all use subcommands and colored output.

## Memory Aids

- **GOSUB**: Getopts, Options, Shift, Use, Break — flow of argument parsing.
- **SLICE**: Source, Library, Include, Config, Execute — order of initialization in main.sh.
- **U-D-R**: Undo-Dry-Run pattern — every destructive action should support all three modes.
- **4 P's of CLI**: Parsing, Processing, Presentation, Persistence — the four stages of any CLI tool.
- **STDERR vs STDOUT**: "Errors go to 2, data goes to 1" — `echo "data"` (stdout), `error "msg" >&2` (stderr).

## Trap Vault

1. **⚠️ TRAP:** `source` doesn't fork. Variables set in main.sh are visible in subcommands, but `exit` in a sourced script exits the entire process, not just the sourced file.

2. **⚠️ TRAP:** `getopts` resets `OPTIND` to 1 automatically. If you parse options in a function, `OPTIND` must be declared `local` to avoid interfering with the caller's option parsing.

3. **⚠️ TRAP:** GNU `getopt` returns 0 even with invalid options if you don't use the `-o` flag correctly. Always check `$?` after getopt and always use `eval set -- "$options"`.

4. **⚠️ TRAP:** Colored output codes in scripts piped to a file will include raw escape sequences like `\033[0;31m`. Use `[[ -t 1 ]]` to detect terminals and disable colors for non-TTY output.

5. **⚠️ TRAP:** Config file sourcing is `eval` in disguise. A config file with `$(rm -rf /)` inside a variable value will execute that. Validate config files or restrict permissions to `600`.

6. **⚠️ TRAP:** `set -e` (errexit) causes the script to exit on ANY command that returns non-zero, including `grep` that finds no matches, `mv` of a non-existent file, or `rm` with non-existent paths. Use `|| true` or `command || [[ $? -eq 1 ]]` patterns.

7. **⚠️ TRAP:** `shift` without checking `$#` first can cause "shift count must be <= $#" error. Always validate argument count before shifting.

8. **⚠️ TRAP:** Dry-run mode must be perfectly implemented — if the dry-run path differs from the real path in any way, it's not a valid dry run. Always test both paths with identical logic.

9. **⚠️ TRAP:** The `-o` option collision — `getopts` short options like `-h` and long options like `--help` are different systems. Mixing getopts and GNU getopt in the same script causes confusion. Pick one.

10. **⚠️ TRAP:** Subcommand scripts that use `#!/bin/bash` (standalone shebang) but are meant to be sourced will cause confusion. Sourced scripts should NOT have a shebang — they're not executed directly.

11. **⚠️ TRAP:** Log rotation in the middle of a script — if you `mv logfile logfile.old` while the script has an open file handle to logfile, writes still go to the old file. Use `copytruncate` or close/reopen log handles.

12. **⚠️ TRAP:** `trap ... ERR` fires for every command that returns non-zero, including commands inside `if` conditions and `while` loops. This can cause unexpected exits. Test your trap handler carefully.

13. **⚠️ TRAP:** Hard-coded paths (like `~/config/mytool.conf`) break when run as root (root's `~` is `/root`, not `/home/user`). Use `$HOME` or `$XDG_CONFIG_HOME` instead.

14. **⚠️ TRAP:** Progress bars using `\r` overwrite the line but fail when output is piped — the `\r` literal appears in the file. Detect TTY before using progress bars.

15. **⚠️ TRAP:** Undo logs accumulate. Without log rotation or trimming, a frequently-used tool can generate gigabytes of undo data. Implement a max-entries policy or TTL.

## See It In The Wild

Git's subcommand dispatch is in `git.c` (the C source), but many Git subcommands are actually shell scripts in `git-core/`. For example, `git stash` used to be a shell script before being rewritten in C. The pattern is identical to what we've built.

The `systemctl` command uses a similar dispatch pattern. Running `systemctl status sshd` dispatches to the `status` subcommand handler. The help text, colored output, and option parsing follow the same architecture.

A typical production CLI tool like `gh` (GitHub CLI) shows:
- Top-level help with subcommand list: `gh help`
- Subcommand-specific help: `gh pr help`
- Config file: `~/.config/gh/config.yml`
- Colored output in terminal, suppressed in pipes
- JSON output mode with `--json` flag

## Check Your Understanding

1. What is the difference between `getopts` and GNU `getopt`? When would you choose one over the other?

2. Why must shared libraries (`lib/*.sh`) be sourced in `main.sh` before subcommands are dispatched?

3. What happens to `OPTIND` if you call `getopts` inside a function that was called from another function that also uses `getopts`?

4. How would you add a `--quiet` flag that suppresses all output except errors?

5. Why does `source` make config files a security concern? How would you mitigate this?

6. What does `shift $((OPTIND - 1))` accomplish after a `getopts` loop?

7. When using `set -euo pipefail`, why might `grep -q pattern file` cause your script to exit unexpectedly?

8. How does the subcommand dispatch pattern in `case "${1:-help}"` handle the case where no arguments are given?

9. What is the advantage of logging to stderr (`>&2`) while outputting data to stdout?

10. How would you implement a `--version` flag that prints the tool version and exits?
