# Lesson 6: Advanced Argument Parsing

## History & Origins

The subcommand pattern was popularized by `git` (2005) and later adopted by `docker`, `kubectl`, and most modern CLI tools. Before git, most Unix tools used flat option sets (`ls -la`, `grep -r pattern file`). Git's `git commit -m "msg"` introduced hierarchical command structures to the masses.

GNU `getopt` (the external utility) was created as an enhancement over POSIX `getopts`. It supports long options (`--verbose`), optional arguments, and alternative parsing modes. The version in util-linux is the most common implementation. It predates bash's `getopts` but is not a builtin — it's an external binary.

The manual parsing approach (while-case-shift) is the oldest and most flexible. It predates both `getopts` and `getopt`. Every shell has it. It's also the most error-prone.

## Syntax Reference

### GNU getopt (external)
```
getopt [options] optstring parameters
  -o shortopts   # Short options (same format as getopts)
  -l longopts    # Long options, comma-separated, colon for required arg
  -n name        # Program name for error messages
  -q             # Quiet mode (suppress errors)
  -u             # Unquoted output (default is quoted)
  -a             # Alternative parsing (allow long options with single dash)
  --             # End of getopt options

Long option format in -l:
  verbose            # Flag (no argument)
  file:              # Required argument
  name::             # Optional argument (GNU extension)
```

### Subcommand Pattern
```
case $1 in
  create) shift; do_create "$@" ;;
  list)   shift; do_list "$@" ;;
  *)      usage; exit 1 ;;
esac
```

### getopt Output Processing
```
args=$(getopt -o h -l help -- "$@") || exit 1
eval set -- "$args"
```

## Under the Hood

### getopt's Parsing Modes

GNU getopt supports three parsing modes controlled by the `$POSIXLY_CORRECT` environment variable:

1. **Default mode (permute):** Arguments can be interleaved with options. `cmd -a file -b` parses both `-a` and `-b`, with `file` as positional. This is the standard for most GNU tools.

2. **POSIX mode (strict):** `getopt` stops at the first non-option argument. `cmd -a file -b` only sees `-a`. This matches `getopts` behavior.

3. **Silent mode:** With `-q` or when error output is suppressed, getopt returns error codes without printing messages.

The permute mode is implemented by rearranging `argv` — getopt physically moves non-option arguments to the end. This is why you need `eval set -- "$(getopt ...)"` — the quoted output becomes the new, rearranged `$@`.

### What strace Shows

```bash
$ strace -f -e execve getopt -o ab: -l verbose,file: -- -a --file foo extra
execve("/usr/bin/getopt", ["getopt", "-o", "ab:", "-l", "verbose,file:", "--", "-a", "--file", "foo", "extra"], ...
```

`getopt` is an external process. Every call forks and execs. For performance-critical parsing (like bash completion), `getopts` (builtin) is faster.

### getopt Output Format

`getopt` with default quoting produces output like:
```
-a --file 'foo' -- 'extra'
```

Each argument is quoted. The `--` separates options from positional arguments. This output is designed for `eval set -- ...`.

Without quoting (`-u` flag):
```
-a --file foo -- extra
```

Unquoted mode breaks with arguments containing spaces or special characters. Always use the default quoted mode.

### Security Considerations

**Command injection via `eval set --`:** If `getopt` fails (returns non-zero), its output may be empty or malformed. Running `eval set -- "$args"` with an empty string sets `$@` to empty — not catastrophic but can cause logic errors. The real danger: if `getopt` is replaced by a malicious version in a writable PATH directory, the `eval` can execute arbitrary code disguised as getopt output.

**Always validate getopt exit:**
```bash
args=$(getopt ...) || { echo "getopt failed" >&2; exit 1; }
eval set -- "$args"
```

## Core Examples

### Example 1: Basic getopt with Long Options

```bash
$ args=$(getopt -o vf: -l verbose,file: -n "$0" -- "$@")
$ eval set -- "$args"
$ while true; do
>   case $1 in
>     -v|--verbose) echo "Verbose"; shift ;;
>     -f|--file) echo "File: $2"; shift 2 ;;
>     --) shift; break ;;
>     *) echo "Internal error!"; exit 1 ;;
>   esac
> done
```

**Anatomy:**
1. `getopt -o vf: -l verbose,file:` — short: `-v` (flag), `-f FILE` (required arg). Long: `--verbose`, `--file FILE`
2. `args=$(...) || exit 1` — capture output, exit if getopt fails
3. `eval set -- "$args"` — replace `$@` with getopt's rearranged, quoted version
4. `case $1 in` — process each option
5. `--) shift; break` — `--` signals end of options, break the loop
6. `*)` — should never happen if getopt validates correctly

### Example 2: Subcommand Dispatch

```bash
$ case $1 in
>   create)
>     shift
>     echo "Creating backup: $*"
>     ;;
>   list)
>     echo "Listing backups..."
>     ls /backups/
>     ;;
>   restore)
>     shift
>     echo "Restoring: $1"
>     ;;
>   --help|-h)
>     echo "Usage: backup {create|list|restore}"
>     ;;
>   *)
>     echo "Unknown command: $1" >&2
>     exit 1
>     ;;
> esac
```

**What if variations:**
- `backup` (no args) → falls through to `*)` with empty `$1` — handle this
- `backup --help` → handled explicitly before `*)`
- `backup create --name foo` → shifts once, then `create` function handles remaining args

### Example 3: Combining Short and Long Options

```bash
$ args=$(getopt -o hvo: -l help,verbose,output: -n "tool" -- "$@")
$ eval set -- "$args"
$ while true; do
>   case $1 in
>     -h|--help)    echo "Help!"; shift ;;
>     -v|--verbose) echo "Verbose!"; shift ;;
>     -o|--output)  echo "Output: $2"; shift 2 ;;
>     --) shift; break ;;
>     *) echo "Internal error"; exit 1 ;;
>   esac
> done
$ echo "Positional: $*"
```

**What if you forget `--`?**
```bash
$ bash test.sh -v -- -o foo
Verbose!
Positional: -o foo
# Without --, -o would be parsed as an option
```

### Example 4: Optional Arguments in getopt

GNU `getopt` supports optional arguments with `::`:

```bash
$ args=$(getopt -o l:: -l log:: -n "tool" -- "$@")
$ eval set -- "$args"
$ while true; do
>   case $1 in
>     -l|--log)
>       if [[ -n "$2" && "$2" != "--" ]]; then
>         echo "Log file: $2"; shift 2
>       else
>         echo "Log to default"; shift
>       fi
>       ;;
>     --) shift; break ;;
>   esac
> done
$ bash test.sh --log  # Default log
$ bash test.sh --log=mylog.txt  # Custom log
$ bash test.sh -lmylog.txt  # Short form attached
```

**TRAP:** `getopts` (builtin) does NOT support optional arguments. Only GNU `getopt` (external) does. Check which tool you're using.

### Example 5: Nested Subcommand Parsing

```bash
$ backup_create() {
>   local args
>   args=$(getopt -o n:s:cv -l name:,source:,compress,verbose -n "backup create" -- "$@") || exit 1
>   eval set -- "$args"
>   local name="" source="" compress=0 verbose=0
>   while true; do
>     case $1 in
>       -n|--name)     name="$2"; shift 2 ;;
>       -s|--source)   source="$2"; shift 2 ;;
>       -c|--compress) compress=1; shift ;;
>       -v|--verbose)  verbose=1; shift ;;
>       --) shift; break ;;
>     esac
>   done
>   [[ -z "$name" ]] && { echo "Error: --name required" >&2; exit 1; }
>   [[ -z "$source" ]] && { echo "Error: --source required" >&2; exit 1; }
>   echo "Creating backup '$name' from '$source'"
>   ((compress)) && echo "Compression enabled"
> }
```

### Example 6: Handling `--` in Subcommands

```bash
$ main() {
>   case $1 in
>     create) shift; backup_create "$@" ;;
>     list)   shift; backup_list "$@" ;;
>     --)     shift; echo "Positional: $*" ;;
>     *)      echo "Usage: ..."; exit 1 ;;
>   esac
> }
$ bash backup.sh -- extra args
Positional: extra args
```

### Example 7: Manual Parsing for Simple Cases

```bash
$ parse_simple() {
>   while [[ $# -gt 0 ]]; do
>     case $1 in
>       -v|--verbose) verbose=1; shift ;;
>       -f|--force) force=1; shift ;;
>       -o|--output) output="$2"; shift 2 ;;
>       --) shift; break ;;
>       -*)
>         echo "Unknown: $1" >&2
>         exit 1
>         ;;
>       *) break ;;
>     esac
>   done
>   positional=("$@")
> }
```

**Comparison with getopt:** Manual parsing doesn't fork, supports everything, but is more code and error-prone (forgetting `shift`, missing `--` handling, broken permutation support).

### Example 8: Mixing getopts and getopt

Don't do this. Pick one. But if you must:

```bash
# BAD — confusing:
while getopts "ab:" opt; do  # getopts for short only
  case $opt in ... esac
done
shift $((OPTIND - 1))
# Then... wait, how do we parse long options now?
# This is a mess. Just use getopt for everything.
```

### Example 9: getopt with Subcommand Completion Detection

```bash
$ # Detect if we're in a completion context
$ if [[ -n "$COMP_LINE" ]]; then
>   # Generate completion words
>   compgen -W "create list restore" -- "$2"
> fi
```

### Example 10: Help Text for Subcommands

```bash
$ global_help() {
>   cat <<'EOF'
> Usage: backup <command> [options]
>
> Commands:
>   create   Create a new backup
>     --name NAME     Backup name (required)
>     --source DIR    Source directory (required)
>     --compress      Enable compression
>
>   list     List existing backups
>     --all           Show all backups
>     --name PATTERN  Filter by name
>
>   restore  Restore from backup
>     --name NAME     Backup name (required)
>     --target DIR    Restore target (required)
>     --force         Overwrite existing files
> EOF
> }
```

### Example 11: getopt with Short Option Aggregation

```bash
$ args=$(getopt -o ab:c -l all,block:,count -n "tool" -- "$@")
$ eval set -- "$args"
$ # -abc is parsed as -a -b c (if b requires arg)
$ # -ac is parsed as -a -c
```

### Example 12: Using getopt in Strict POSIX Mode

```bash
$ POSIXLY_CORRECT=1
$ args=$(getopt -o ab: -l all,block: -- "$@")
$ # Now getopt stops at first non-option argument
```

### Example 13: Error Recovery with getopt

```bash
$ if ! args=$(getopt -o h -l help -- "$@" 2>/dev/null); then
>   echo "Failed to parse options"
>   # Fall back to manual parsing or exit
>   exit 2
> fi
$ eval set -- "$args"
```

### Example 14: Dynamic Option Specification

```bash
$ build_optstring() {
>   local opts="h"
>   local longopts="help"
>   for cmd in "${AVAILABLE_COMMANDS[@]}"; do
>     opts+=""
>     longopts+=",$cmd"
>   done
>   echo "-o $opts -l $longopts"
> }
$ args=$(getopt $(build_optstring) -n "tool" -- "$@")
```

### Example 15: getopt with Multiple Long Synonyms

```bash
$ # getopt doesn't support multiple long names for one option
$ # Workaround: just handle both in case:
$ case $1 in
>   -v|--verbose|--talkative|--chatty)
>     verbose=1; shift ;;
> esac
```

## Real-World Use Cases

### FOR the OS
- **System management tools:** `systemctl start/stop/restart`, `mount -t type device dir`
- **Network configuration:** `ip addr add/del/show`, `nmcli connection up/down`
- **Package management:** `apt-get install/remove/update`, `dpkg -i/-r/-l`

### WITH the OS
- **Build tools:** `make target variable=value`, custom build scripts with subcommands
- **Deployment pipelines:** `deploy staging/production --rollback --version X.Y.Z`
- **Container tools:** Docker-style CLI wrappers

### AGAINST the OS (defense perspective)
- **Option injection through $@:** If user input reaches `$@` and `--` is not handled, options can be injected. `eval set --` with getopt output is safer because getopt rearranges.
- **Subcommand collision:** An attacker could create a file named `create` in `$PATH` that runs their code when `backup create` is invoked (if backup uses PATH lookup for subcommands).

### FOR DEFENSE
- **Always validate getopt exit code:** Never `eval set --` on failed getopt output
- **Use `--` in all internal wrappers:** `grep -- "$pattern" "$file"` to prevent pattern starting with `-`
- **Principle of least surprise:** Subcommands should follow established patterns (git-style) to reduce misconfiguration
- **Input validation:** Even with getopt, validate OPTARG values against expected patterns

## Memory Aids

- **"getopt = get options"** — The external utility. Think: "I need to get opt ions, I'll call an external program."
- **"getopts = get options, shell version"** — The builtin. The 's' stands for 'shell'.
- **"eval set -- quotes protect"** — Always use `eval set -- "$(getopt ...)"` with the default quoting.
- **"Permute = Permutation"** — GNU getopt reorders arguments. POSIX mode preserves order.
- **"Subcommand = function dispatch"** — `case $1 in` maps command names to functions. That's it.
- **"Colon = consumption, double-colon = optional"** — `:` for required, `::` for optional (GNU only).

## Trap Vault

1. **`getopt` vs `getopts` name confusion:** These are different tools. `getopts` is a bash builtin. `getopt` is an external binary. They have different syntax, different features, and different output formats.

2. **getopt forks a process:** Every `getopt` call spawns a subprocess. In a subcommand parser called thousands of times, this matters. `getopts` is builtin and has zero fork overhead.

3. **`eval set -- "$args"` is mandatory:** If you forget `eval set --`, the arguments remain as a single quoted string. `set -- "$args"` sets `$1` to the entire getopt output with quoting preserved but not applied.

4. **getopt quoting breaks with some inputs:** In edge cases (like arguments with embedded newlines), getopt's quoting may not round-trip correctly through `eval`. Testing is essential.

5. **`--` handling in subcommands:** Each subcommand needs its own `--` handler. A `--` meant for the outer command must not interfere with inner parsing.

6. **No subcommand alias support:** `git co` for `git checkout` requires manual alias support. Add a case `co|checkout)` in your dispatch.

7. **getopt exit code in conditionals:** `args=$(getopt ... || exit 1)` works but `set -e` may exit before you can handle the error. Use explicit `||` with error message.

8. **Long option abbreviation:** GNU getopt supports unambiguous abbreviations (`--ver` for `--verbose`). This can lead to surprising behavior if new options are added later.

9. **`$@` vs `$*` in subcommand dispatch:** Always use `"$@"` to preserve quotes. `"$*"` joins arguments with spaces, corrupting multi-word arguments.

10. **getopt's unquoted output is dangerous:** `getopt -u` produces output without quotes. If an argument contains spaces, it splits. Never use `-u` with `eval set --`.

11. **`set -e` + `eval set --`:** If `set -e` is active and `getopt` succeeds, `eval set -- "$args"` could still fail if the eval produces a non-zero exit (unlikely but possible with bad quoting).

12. **Subcommand argument bleeding:** If a subcommand doesn't `shift` properly, the subcommand name remains in `$@`. Subsequent processing sees it as a positional argument.

13. **getopt and environment variables:** `GETOPT_COMPATIBLE` forces `getopt` to behave like the old Unix `getopt` (no long options). If set in the environment, your long options fail silently.

14. **Multiple `--` in argument list:** `getopt` only treats the first `--` as the separator. All subsequent `--` are passed through as positional arguments.

15. **Nested subcommands with getopt:** `backup create --name foo` — after `create` shifts, `$@` is `--name foo`. The inner getopt must parse this correctly. Ensure `OPTIND` is reset (via new getopt call, not shared state).

## See It In The Wild

```bash
# Check if getopt supports long options
$ getopt --version
getopt from util-linux 2.38

# See how system tools use subcommands
$ cat /usr/bin/docker 2>/dev/null | head -20
# Docker is a binary, but many tools are shell scripts:
$ file /usr/bin/apt-get
# Binary, but /usr/sbin/update-grub is a script using getopts

# Count getopt usage in system scripts
$ grep -r 'eval set --.*getopt' /usr/bin/ /usr/sbin/ 2>/dev/null | wc -l

# Examine a real subcommand script
$ cat /usr/sbin/update-grub 2>/dev/null | head -50
```

## Check Your Understanding

1. What is the difference between `getopt` and `getopts`? When would you use each?

2. Why is `eval set -- "$(getopt ...)"` necessary? What happens if you just use `set -- "$(getopt ...)"`?

3. How does GNU getopt's "permute" mode differ from POSIX mode? How do you control this behavior?

4. Why should each subcommand function call `getopt` separately instead of sharing parsed options?

5. How do you handle `--` (end of options) in a subcommand parser?

6. What is the output format of `getopt` when given `-a --file "hello world" extra`?

7. How would you implement optional arguments with `getopt`? Can `getopts` do the same?

8. What security risk does `eval set -- "$args"` pose? How do you mitigate it?

9. How would you implement a `--help` flag that works at both the global and subcommand level?

10. What is the difference between `getopt -o ab:c` and `getopt -o a:b:c` in terms of which options require arguments?
