# Lesson 5: getopts Basics

## History & Origins

`getopts` entered the Unix world as part of the System V shell in the 1980s. Before that, scripts parsed options manually with while-case-shift patterns — which still work but are error-prone and lack standardized error handling. The POSIX standard formalized `getopts` as the required option parsing mechanism for shell scripts.

`getopts` is a shell BUILTIN (not an external command), which means it doesn't fork. This is in contrast to `getopt` (without the `s`), which is an external command from the util-linux package. The naming is confusing and has tripped up generations of shell programmers.

The design philosophy of `getopts` is minimal and POSIX-focused: single-letter options only, no long options, automatic error reporting, and a simple state-machine interface. The `OPTIND` variable tracks position (like an index in a C argv parser), and `OPTARG` holds the argument for options that require one.

Bash extended `getopts` slightly but maintained full POSIX compatibility. The fundamental interface hasn't changed in 40+ years — a testament to its good design.

## Syntax Reference

### Basic Form
```
getopts optstring name [args]
```

### Optstring Characters
```
ab       # Option letters a and b (both flags)
a:b      # a requires an argument, b is a flag
:a:b     # Leading colon = silent error mode
:ab:c    # Mixed: a flag, b requires arg, c flag
```

### Special Variables
```
$OPTARG  # Argument value for the current option (if it requires one)
$OPTIND  # Index of the next argument to process (starts at 1)
```

### Built-in Error Handling
```
?        # Unknown option (printed as opt char in normal mode)
:        # Missing argument (in silent mode with leading :)
```

### Shift Pattern
```
shift $((OPTIND - 1))   # Remove all parsed options, leave positional args
```

### Function Usage
```
local OPTIND OPTARG opt   # MUST localize these in functions
```

## Under the Hood

### How getopts Works Internally

`getopts` is a state machine implemented in the shell's C code. It maintains an internal pointer to the current position in the current argument. When you call `getopts`:

1. If the internal pointer is at the start of a new argument (or at start), advance `OPTIND` to look at the next argument
2. If the current argument starts with `-` and is not `--`:
   - Examine the character after `-`
   - If it matches a letter in `optstring`: set `OPTARG` if needed, advance pointer
   - If it doesn't match: set `opt` to `?`, print error (unless silent mode)
3. If the current argument is `--`: consume it, set `OPTIND` past it, return false (exit loop)
4. If the current argument doesn't start with `-`: stop, set `OPTIND` at this argument, return false

The key insight: `getopts` modifies `OPTIND` AFTER each call, so the loop naturally advances through arguments.

### OPTIND Reset Issues

`OPTIND` is NOT automatically reset. Once set to, say, 5, it stays 5 until you explicitly set `OPTIND=1`. This is why calling `getopts` twice in the same shell without resetting `OPTIND` produces wrong results:

```bash
$ # First parse:
$ OPTIND=1; while getopts "ab" opt; do case $opt in ... esac; done
$ # OPTIND is now past all options
$ # Second parse (without reset):
$ while getopts "ab" opt; do ... done
$ # getopts sees OPTIND > #args, immediately returns false — skips everything!
```

### Error Mode Implementation

Without leading `:`:
- Unknown option: `getopts` sets `opt` to `?`, prints `script: illegal option -- X` to stderr
- Missing argument: `getopts` sets `opt` to `?`, prints `script: option requires an argument -- X`

With leading `:`:
- Unknown option: sets `opt` to `?`, `OPTARG` to the unknown letter, NO error message
- Missing argument: sets `opt` to `:` (colon), `OPTARG` to the option letter, NO error message

### Comparison with C's getopt

The C function `getopt()` from `<unistd.h>` works almost identically:
- Uses global variables `optarg`, `optind`, `optopt`
- Returns the option character or `?`/`:`
- Uses the same optstring format (`:` for silent mode, trailing `:` for required arg)

```c
// C version
while ((opt = getopt(argc, argv, "ab:c:")) != -1) {
    switch (opt) {
    case 'a': /* flag */ break;
    case 'b': /* arg in optarg */ break;
    case '?': /* error */ break;
    }
}
```

The bash version is intentionally parallel. This makes it easy for C programmers to write shell scripts.

## Core Examples

### Example 1: Simple Flag Parsing

```bash
$ cat > parse.sh << 'EOF'
while getopts "ab" opt; do
  case $opt in
    a) echo "Flag -a set" ;;
    b) echo "Flag -b set" ;;
    *) echo "Unknown: $opt" ;;
  esac
done
EOF
$ bash parse.sh -a -b
Flag -a set
Flag -b set
```

**Anatomy:**
1. `getopts "ab" opt` — recognizes options `-a` and `-b` (both flags, no arguments)
2. Each call returns the next option letter
3. The `case` block handles each option
4. When all options are consumed, `getopts` returns false, loop exits

**What if variations:**
- `parse.sh -b -a` — order doesn't matter, both get processed
- `parse.sh -ab` — combined flags work, same output
- `parse.sh -c` — unknown option, `opt` gets `?`

### Example 2: Options with Arguments

```bash
$ cat > arg_parse.sh << 'EOF'
while getopts "o:n:" opt; do
  case $opt in
    o) echo "Output file: $OPTARG" ;;
    n) echo "Count: $OPTARG" ;;
  esac
done
EOF
$ bash arg_parse.sh -o output.txt -n 10
Output file: output.txt
Count: 10
```

**Anatomy:**
1. `"o:n:"` — `o` and `n` both require arguments (the `:` after them)
2. When `-o` is found, `$OPTARG` is set to `output.txt`
3. When `-n` is found, `$OPTARG` is set to `10`

**Different argument styles:**
- `-o output.txt` (space-separated)
- `-ooutput.txt` (attached) — both work with getopts

### Example 3: Combined Flags (-vf)

```bash
$ cat > combine.sh << 'EOF'
while getopts "vf" opt; do
  case $opt in
    v) echo "Verbose" ;;
    f) echo "Force" ;;
  esac
done
EOF
$ bash combine.sh -vf
Verbose
Force
```

**Anatomy:**
1. `-vf` is one argument, but `getopts` processes it as `-v` followed by `-f`
2. `getopts` internally tracks the position within the current argument
3. It processes `v`, then advances to `f` within the same `-vf` string

### Example 4: Silent Error Mode with Leading `:`

```bash
$ cat > silent.sh << 'EOF'
while getopts ":a:" opt; do
  case $opt in
    a) echo "Arg: $OPTARG" ;;
    :) echo "Missing argument for -$OPTARG" ;;
    \?) echo "Unknown option: -$OPTARG" ;;
  esac
done
EOF
$ bash silent.sh -a
Missing argument for -a
$ bash silent.sh -b
Unknown option: -b
```

**Anatomy:**
1. Leading `:` enables silent mode — no built-in error messages
2. Missing argument to `-a`: `opt` gets `:`, `OPTARG` gets `a`
3. Unknown option `-b`: `opt` gets `?`, `OPTARG` gets `b`

**Without silent mode:**
```bash
$ bash silent.sh -a  # In normal mode
script: option requires an argument -- a
```
Bash prints the error itself and sets `opt` to `?`.

### Example 5: Required vs Optional Argument Patterns

`getopts` does not support optional arguments. A trailing `::` for optional arguments works in GNU `getopt` but NOT in `getopts`. This is a common point of confusion.

```bash
# getopts does NOT support optional arguments:
# "a::" — NOT valid in getopts (will be parsed as "a:" with extra ":")

# Workaround: require the argument but provide a default
while getopts "a:" opt; do
  case $opt in
    a) arg="${OPTARG:-default}" ;;  # OPTARG is always set if -a is used
  esac
done
```

### Example 6: Shifting After Parsing

```bash
$ cat > shift_demo.sh << 'EOF'
while getopts "a:b" opt; do :; done
shift $((OPTIND - 1))
echo "Remaining args: $*"
EOF
$ bash shift_demo.sh -a foo bar baz
Remaining args: bar baz
```

**Anatomy:**
1. `getopts` parses `-a foo`, sets `OPTIND` to 3 (pointing at `bar`)
2. `shift $((OPTIND - 1))` = `shift 2` — removes `-a` and `foo`
3. `$*` now contains `bar baz`

### Example 7: Using getopts in a Function

```bash
$ parse_opts() {
>   local OPTIND OPTARG opt
>   while getopts "ab:" opt; do
>     case $opt in
>       a) echo "a!" ;;
>       b) echo "b: $OPTARG" ;;
>     esac
>   done
> }
$ parse_opts -b hello
b: hello
```

**Critical:** Without `local OPTIND`, calling `parse_opts` twice produces wrong results. The first call sets `OPTIND` to some value, and the second call starts from that value instead of 1.

**What happens without local:**
```bash
$ parse_opts -a  # OPTIND becomes 2 (past all args)
$ parse_opts -a  # OPTIND starts at 2 — skips arg 1, nothing to parse!
```

### Example 8: Parsing from an Array (Non-standard args)

```bash
$ args=(-a -b foo -c)
$ while getopts "ab:c" opt "${args[@]}"; do
>   case $opt in
>     a) echo "Flag a" ;;
>     b) echo "b=$OPTARG" ;;
>     c) echo "Flag c" ;;
>   esac
> done
Flag a
b=foo
Flag c
```

By passing `"${args[@]}"` as the final argument to `getopts`, you can parse any array, not just `$@`. This is useful for parsing command strings or configuration arrays.

### Example 9: Nested Option Parsing (Subcommand)

```bash
$ subcommand_a() {
>   local OPTIND OPTARG opt
>   while getopts "x:" opt; do
>     case $opt in
>       x) echo "sub-a -x $OPTARG" ;;
>     esac
>   done
> }
$ case ${1:-} in
>   a) shift; subcommand_a "$@" ;;
>   *) echo "Unknown" ;;
> esac
$ bash script.sh a -x hello
sub-a -x hello
```

Without `local OPTIND` in `subcommand_a`, the outer `OPTIND` value would corrupt the inner parsing.

### Example 10: getopts with Default Values

```bash
$ args=("$@")
$ verbose=0
$ count=10  # default
$ output=""  # required
$ while getopts ":hvo:n:" opt; do
>   case $opt in
>     v) verbose=1 ;;
>     o) output="$OPTARG" ;;
>     n) count="$OPTARG" ;;
>     h) echo "Usage: ..."; exit 0 ;;
>     :) echo "Missing arg for -$OPTARG" >&2; exit 1 ;;
>     \?) echo "Unknown: -$OPTARG" >&2; exit 1 ;;
>   esac
> done
$ shift $((OPTIND - 1))
$ [[ -z "$output" ]] && { echo "Error: -o is required" >&2; exit 1; }
$ echo "verbose=$verbose, count=$count, output=$output, positional=$*"
```

### Example 11: The `--` Separator with getopts

```bash
$ cat > sep_test.sh << 'EOF'
while getopts "ab:" opt; do
  case $opt in
    a) echo "Flag a" ;;
    b) echo "b=$OPTARG" ;;
  esac
done
shift $((OPTIND - 1))
echo "Remaining: $*"
EOF
$ bash sep_test.sh -a -- -b
Flag a
Remaining: -b
```

`--` signals the end of options. Everything after `--` is treated as positional, even if it starts with `-`.

### Example 12: getopts with No Options at All

```bash
$ while getopts "a" opt; do
>   echo "Got: $opt"
> done
$ echo "Exit code: $?"  # 1 (false) — no options to parse
```

If there are no arguments at all, `getopts` immediately returns false (1). The loop body never executes.

### Example 13: Error Handling Without Silent Mode

```bash
$ cat > nosilent.sh << 'EOF'
while getopts "a:" opt; do
  case $opt in
    a) echo "OK: $OPTARG" ;;
    ?) echo "Error occurred — exiting" >&2; exit 1 ;;
  esac
done
EOF
$ bash nosilent.sh -a
script: option requires an argument -- a
Error occurred — exiting
```

Both the built-in error (printed by bash) and the custom error (from the script) appear.

### Example 14: getopts with Dash-only Arguments

```bash
$ while getopts ":a" opt; do
>   case $opt in
>     a) echo "a" ;;
>     \?) echo "Unknown: -$OPTARG" ;;
>   esac
> done
$ bash test.sh -a - -c
a
Unknown: -
```

A single `-` is NOT treated as an option (it's not `--`). It's treated as a positional argument, so `getopts` stops at it. Then `--` is the separator, and `-c` after it is also positional.

### Example 15: Multiple getopts Calls (Resetting OPTIND)

```bash
$ parse_section() {
>   local OPTIND=1  # Reset every time!
>   while getopts "ab" opt; do
>     case $opt in
>       a) echo "Section saw -a" ;;
>       b) echo "Section saw -b" ;;
>     esac
>   done
> }
$ args1=(-a -b)
$ args2=(-b -a)
$ parse_section "${args1[@]}"
Section saw -a
Section saw -b
$ parse_section "${args2[@]}"
Section saw -b
Section saw -a
```

Each call resets `OPTIND=1` locally, so each parse starts fresh.

## Real-World Use Cases

### FOR the OS
- **init scripts:** `/etc/init.d/functions` uses getopts for start/stop/restart parsing
- **System administration tools:** Almost all CLI utilities use getopts or getopt
- **Package managers:** `apt-get` (though written in C) uses the same option model

### WITH the OS
- **Build scripts:** Makefile helpers that parse flags (verbose, debug, output dir)
- **Log analysis tools:** Scripts that take date ranges, format flags, file patterns
- **Deployment scripts:** Environment selection, dry-run mode, rollback flags

### AGAINST the OS (defense perspective)
- **Argument injection via `$@`:** If user input flows into `$@` and getopts doesn't handle `--`, options can be injected
- **`OPTIND` manipulation:** An attacker who can set variables before your script runs can corrupt `OPTIND` and bypass option parsing

### FOR DEFENSE
- **Always handle `--`:** Process the `--` separator to separate options from positional args
- **Always validate OPTARG:** An empty `$OPTARG` for a required argument should be caught
- **Use silent mode:** Custom error messages are more informative and don't reveal script internals
- **Localize OPTIND in functions:** Prevents cross-contamination between parsers

## Memory Aids

- **"Colon = consumption"** — `:` after a letter means consume the next argument. `:` at the start means consume errors silently.
- **"getopts is the builtin, getopt is the external"** — The `s` stands for "shell" (builtin).
- **"OPTIND is the needle, OPTARG is the thread"** — `OPTIND` tracks position (where we are), `OPTARG` carries the data (the argument).
- **"One letter, one colon, one way"** — getopts is single-letter, colon for argument, no alternatives.
- **"Shift after parse, or live to regret it"** — Always `shift $((OPTIND - 1))` to separate options from positional args.

## Trap Vault

1. **`getopts` does not support long options:** `--verbose` will be parsed as `-v` `-e` `-r` `-b` `-o` `-s` `-e` (each letter as a separate option). Use `getopt` (external) for long options.

2. **`OPTIND` is global and persistent:** Calling getopts twice in the same function without resetting `OPTIND=1` silently produces wrong results. Always use `local OPTIND` in functions.

3. **`OPTARG` is not cleared between calls:** If option `a` doesn't take an argument but option `b` does, and `-a` is processed after `-b`, `OPTARG` still contains the value from `-b`. Always treat `OPTARG` as undefined unless the current option requires it.

4. **`getopts` vs `getopt` name confusion:** `getopts` (builtin) and `getopt` (external) are different tools. `getopts` is POSIX. `getopt` supports long options but is not POSIX. They have different interfaces.

5. **Leading `:` kills all error output:** In silent mode, bash prints NOTHING to stderr about bad options. If you don't handle `?` and `:` in your case, errors are silently ignored.

6. **`-` as an option argument:** `-` by itself is NOT treated as getopts option terminator. Only `--` stops option parsing. If a positional arg starts with `-`, it's parsed as options after options are exhausted.

7. **`$OPTIND` starts at 1, not 0:** The first argument is `$1` (index 1). After parsing the first option, `OPTIND` becomes 2. This is consistent with bash's 1-indexed argument model but trips up C programmers.

8. **No optional arguments in getopts:** GNU `getopt` supports `a::` for optional arguments. `getopts` does not. If you need optional arguments, you must use `getopt` or manual parsing.

9. **`set -e` interaction:** If `getopts` encounters an error in normal mode, it sets `opt` to `?` but does NOT cause the script to exit (even with `set -e`). But if your case falls through to `*)`, you might exit yourself.

10. **getopts modifies `$@` indirectly:** `getopts` does NOT modify `$@`. You must `shift $((OPTIND - 1))` to remove options from `$@`. Forgetting this leaves options in `$@`.

11. **`OPTIND` reset with `shift`:** If you `shift` inside the getopts loop, `OPTIND` becomes incorrect because it tracks positions in the original `$@`, not the shifted one. Don't shift inside the loop.

12. **Interleaving options and positional args:** `getopts` stops at the first non-option argument. `command -a file -b` — `-b` after `file` is NOT parsed as an option. GNU `getopt` handles this with permute mode; `getopts` does not.

13. **`$OPTARG` contains leading whitespace:** If the argument has leading spaces, they're preserved (because `$OPTARG` is the raw next argument, not trimmed).

14. **getopts with `-` option letter:** You can't have `-` as an option letter because getopts uses it as the prefix. The optstring `"-a"` doesn't mean "dash" option — it's an error.

15. **Nested getopts without local variables in outer scope:** If `main` calls `parse_sub` which uses getopts, and `main` also uses getopts after `parse_sub` returns, the `OPTIND` from `parse_sub` corrupts `main`'s parsing.

## See It In The Wild

### Checking getopts usage on your system
```bash
$ grep -r 'getopts' /usr/bin/ 2>/dev/null | head -10
# Many system scripts use getopts
```

### Tracing getopts execution
```bash
$ strace -f -e trace=write bash -c '
while getopts "ab:" opt; do :; done
' 2>&1 | head -20
# getopts is a builtin — no external calls. strace shows nothing for getopts itself.
```

### Debugging getopts with xtrace
```bash
$ bash -x -c '
while getopts "ab:" opt; do
  echo "opt=$opt OPTARG=$OPTARG OPTIND=$OPTIND"
done
' -- -a -b hello
+ getopts ab: opt
+ echo 'opt=a OPTARG= OPTIND=2'
+ getopts ab: opt
+ echo 'opt=b OPTARG=hello OPTIND=4'
+ getopts ab: opt
```

## Check Your Understanding

1. Why must `OPTIND` be localized inside functions? What happens if you forget?

2. What does the leading `:` in the optstring `":ab:"` do? How does it change error handling?

3. How does `getopts` handle `-vf` as a single argument? What does it do internally?

4. What happens when an option requiring an argument (like `-o` in `"o:"`) has no argument?

5. Why is `shift $((OPTIND - 1))` needed? What would happen without it?

6. How does `getopts` differ from `getopt` (the external utility)? What can `getopt` do that `getopts` can't?

7. What happens to `$OPTARG` when processing a flag option (no argument required)? Is it always empty?

8. How would you parse both `-o file` and `-ofile` (space-separated and attached argument)?

9. What does `getopts` do when it encounters `--` in the argument list?

10. How would you implement a nested subcommand parser where both the main command and subcommand have options?
