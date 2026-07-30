# Lesson 11: `case` & `select` Menus

## History & Origins

The `case` statement came from the **Bourne shell** (1977), which borrowed the concept from **Algol 68** and **C**'s `switch` statement. But unlike C's `switch`, which only matches integers, bash's `case` matches **glob patterns** — a radical departure that makes it vastly more expressive for string-based branching.

The `select` construct is a **bash extension** from the **POSIX 1003.2** draft (early 90s), designed to create interactive menus without the tedium of manually looping over options and parsing input. It was one of those "why didn't anyone think of this sooner" features.

The name `case` is self-explanatory — "in this case, do this." Each branch is a "case" to consider. The name `select` comes from presenting a menu and letting the user "select" an option. The `esac` closing keyword is just `case` spelled backwards — a bash convention (`if`/`fi`, `case`/`esac`).

## Syntax Reference

### case statement

```
case word in
    pattern1)   commands ;;
    pattern2|pattern3) commands ;;
    [a-z])      commands ;;
    *)          default commands ;;
esac
```

**Pattern types (all glob-based):**

| Pattern | Matches |
|---------|---------|
| `start` | Exact word "start" |
| `start\|stop\|restart` | Any of the three words (OR) |
| `[Yy]*` | Anything starting with Y or y |
| `[0-9]` | Single digit |
| `[[:upper:]]` | Any uppercase letter (POSIX class) |
| `*` | Everything (catch-all) |
| `?` | Single character |
| `[!abc]` | Any character NOT a, b, or c |

**Statement terminators:**

| Terminator | Behaviour |
|------------|-----------|
| `;;` | Stop after this match (like C `break`) |
| `;&` | Fall through to **next block's commands** (unconditionally) |
| `;;&` | Continue testing **next pattern** — execute if matches |

**Edge cases:**
- Patterns are **globs**, not regex. `[abc]` matches exactly one character, not a sequence
- `case "$var" in` — quotes prevent glob expansion of `$var`
- An empty pattern list matches nothing
- `case $var in esac` (no patterns) is valid but useless
- The `word` is expanded — `case $((3+2)) in` works
- Pattern matching respects `shopt -s extglob` for extended patterns

### select statement

```
select var in list; do
    commands
done
```

| Component | Behaviour |
|-----------|-----------|
| `select var` | Loop variable gets the **text** of the selected menu item |
| `$REPLY` | Built-in variable holds the **number** the user typed |
| `PS3` | Prompt string for `select` (default `#? `) |
| `break` | Exit the select loop |
| No match | If user enters invalid number, `var` is empty, `$REPLY` holds what they typed |

**Edge cases:**
- `select` loops **forever** — you must `break` or `exit` to leave
- If the list is empty, the prompt doesn't appear and `select` doesn't loop
- `COLUMNS` affects how the menu is displayed (items per row)
- Non-numeric input: `$var` is empty, `$REPLY` is the raw input
- Empty input (just Enter): redisplays the menu without running commands

## Under the Hood

### OS/kernel mechanisms

`case` is a purely **shell-level** construct — no syscalls, no kernel involvement. Bash parses the `case` block into a pattern-matching decision tree, then walks the tree for each match:

1. Bash expands `word` (parameter expansion, arithmetic, command substitution)
2. It tries each pattern in order against the expanded word
3. On the first match, it executes that branch
4. The terminator (`;;`, `;&`, `;;&`) determines what happens next

`select` is similarly internal but involves:
1. Printing the menu items to stderr (output to `/dev/tty`)
2. Reading a line from stdin using `read`
3. Setting `$var` and `$REPLY`
4. Loop until `break`

**strace reveals:**
For `select`, you'd see:
```
write(2, "1) item1\n2) item2\n3) item3\n", ...) = ...
read(0, "2\n", ...) = ...
```

Nothing fancy — just terminal I/O. The magic is all in bash's internal state machine.

### Process implications

- Zero subshells for either `case` or `select`
- No fork/exec overhead
- `case` is typically faster than `if-elif-else` chains because bash can optimize pattern matching internally
- For long `case` statements (50+ branches), bash walks them sequentially — it's not a jump table like C's `switch`

### Memory model

- The entire `case` block is parsed into memory before execution
- Very long `case` statements consume proportional parse-tree memory
- `select` stores the menu list as an array internally

## Core Examples (8-12 minimum)

### Example 1: Yes/No prompt

**Command:**
```bash
read -p "Continue? (y/n): " ans
case "$ans" in
    y|Y|yes|Yes|YES) echo "Proceeding..." ;;
    n|N|no|No|NO)    echo "Aborted." ;;
    *)               echo "Invalid: $ans" ;;
esac
```

**Input:** `yes`

**Output:** `Proceeding...`

**Step-by-step:**
1. `read` captures "yes" into `$ans`
2. `case "$ans"` evaluates the word
3. First pattern `y|Y|yes|Yes|YES` matches "yes"
4. `echo "Proceeding..."` executes
5. `;;` stops matching

**Variations:**
- Single char: `case "${ans:0:1}" in y|Y) ...` — case-insensitive first char
- Use `shopt -s nocasematch` before the case for global case-insensitivity

### Example 2: Character classification

**Command:**
```bash
char="A"
case "$char" in
    [[:upper:]]) echo "Uppercase letter" ;;
    [[:lower:]]) echo "Lowercase letter" ;;
    [0-9])       echo "Digit" ;;
    [[:punct:]]) echo "Punctuation" ;;
    *)           echo "Other character" ;;
esac
```

**Output:** `Uppercase letter`

**Step-by-step:**
1. `$char` is `A`
2. First pattern `[[:upper:]]` matches — `A` is uppercase
3. Executes and stops

**Variations:**
- `[A-Z]` works but is locale-dependent
- POSIX classes `[[:alpha:]]`, `[[:alnum:]]`, `[[:space:]]` are safer

### Example 3: Interactive select menu

**Command:**
```bash
PS3="Choose (1-4): "
select action in "Show date" "Show users" "Show disk" "Exit"; do
    case "$action" in
        "Show date")  date ;;
        "Show users") who ;;
        "Show disk")  df -h ;;
        "Exit")       echo "Bye!"; break ;;
        *)            echo "Invalid: $REPLY" ;;
    esac
done
```

**Input:** `1` then `4`

**Output:**
```
1) Show date
2) Show users
3) Show disk
4) Exit
Choose (1-4): 1
Thu Jul 31 12:00:00 UTC 2026
Choose (1-4): 4
Bye!
```

**Step-by-step:**
1. Menu prints with numbered items
2. User types `1`, `$REPLY=1`, `$action="Show date"`
3. `case` matches `"Show date"`, runs `date`
4. User types `4` next iteration, `$action="Exit"`, `break` exits loop

**Variations:**
- `COLUMNS=1` forces single-column display
- Dynamic list: `select opt in "${options[@]}"`
- Default timeout: wrap with `read -t N` inside the loop

### Example 4: CLI argument parsing

**Command:**
```bash
#!/bin/bash
verbose=0
output=""
file=""

while [[ $# -gt 0 ]]; do
    case "$1" in
        -v|--verbose)    verbose=1; shift ;;
        -o|--output)     output="$2"; shift 2 ;;
        -f|--file)       file="$2"; shift 2 ;;
        -h|--help)       echo "Usage: $0 [-v] [-o dir] [-f file]"; exit 0 ;;
        --)              shift; break ;;
        -*)              echo "Unknown: $1"; exit 1 ;;
        *)               break ;;
    esac
done

echo "verbose=$verbose output=$output file=$file"
```

**Input:** `./script.sh -v --file /etc/hosts --output /tmp`

**Output:** `verbose=1 output=/tmp file=/etc/hosts`

**Step-by-step:**
1. `-v` matches the verbose pattern, sets `verbose=1`, shifts once
2. `--file` matches, takes next arg `/etc/hosts` as `$2`, shifts twice
3. `--output` matches, takes `/tmp` as `$2`, shifts twice
4. No more arguments, loop exits

**Variations:**
- Long options with `=` syntax: `--output=/tmp` — handle with `--output=*) output="${1#*=}"; shift`
- Combined short opts: `-vf file` — requires manual splitting

### Example 5: Fallthrough with `;&`

**Command:**
```bash
val=2
case $val in
    1) echo "Case 1" ;&
    2) echo "Case 2 (falls through)" ;&
    3) echo "Case 3 (also runs)" ;;
    4) echo "Case 4 (never reached)" ;;
esac
```

**Output:**
```
Case 2 (falls through)
Case 3 (also runs)
```

**Step-by-step:**
1. `val=2` matches pattern `2`
2. `echo "Case 2"` runs
3. `;&` says: "run the next block's commands too, regardless"
4. `echo "Case 3"` runs
5. `;;` stops

**Variations:**
- Useful for cumulative operations where case 2 should do everything case 3 does
- Risky — easy to forget you have fallthrough enabled

### Example 6: Continue matching with `;;&`

**Command:**
```bash
fruit="apple"
case "$fruit" in
    a*)   echo "Starts with a" ;;&
    *e)   echo "Ends with e" ;;&
    *p*)  echo "Contains p" ;;&
    *le)  echo "Ends with le" ;;
esac
```

**Output:**
```
Starts with a
Ends with e
Contains p
Ends with le
```

**Step-by-step:**
1. `fruit="apple"` matches `a*` → runs, then `;;&` keeps testing
2. Matches `*e` → runs, `;;&` keeps testing
3. Matches `*p*` → runs, `;;&` keeps testing
4. Matches `*le` → runs, `;;` stops

**Variations:**
- Great for multi-category classification
- Like an `if-elif` chain where more than one condition can be true

### Example 7: Init-script style case

**Command:**
```bash
#!/bin/bash
case "$1" in
    start)
        echo "Starting $NAME..."
        /usr/sbin/$NAME
        ;;
    stop)
        echo "Stopping $NAME..."
        pkill $NAME
        ;;
    restart)
        $0 stop
        $0 start
        ;;
    status)
        pgrep -x $NAME && echo "Running" || echo "Stopped"
        ;;
    *)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

**Input:** `./service.sh status`

**Output:** (depends on whether process is running)

**Step-by-step:**
1. `$1` is `status`
2. Matches the `status)` branch
3. `pgrep -x $NAME` checks for the process
4. Returns appropriate message

**Variations:**
- Add `reload|force-reload` patterns pointing to `restart`
- Use `getopt` or `getopts` for more complex option handling

### Example 8: select with dynamic options

**Command:**
```bash
options=(/var/log/*.log)
PS3="Which log to view? "
select log in "${options[@]}" "Quit"; do
    [[ "$log" == "Quit" ]] && break
    [ -z "$log" ] && { echo "Invalid"; continue; }
    head -20 "$log"
    echo "--- (end of $log) ---"
done
```

**Input:** User picks `1`

**Output:** (first 20 lines of the selected log)

**Step-by-step:**
1. Globs /var/log/*.log into an array at script start
2. `select` presents the list with appended "Quit"
3. User picks a number → `$log` is the full path
4. `head -20` displays the start of the file

**Variations:**
- File selector: `select f in *; do ... done`
- Kill picker: `select pid in $(pgrep sshd); do kill $pid; done`

### Example 9: System info script with nested case

**Command:**
```bash
#!/bin/bash
echo "1. CPU info"
echo "2. Memory info"
echo "3. Disk info"
read -p "Choice: " choice

case "$choice" in
    1) case "$(uname -s)" in
           Linux)  cat /proc/cpuinfo | head -5 ;;
           Darwin) sysctl -a | grep machdep.cpu | head -5 ;;
           *)      echo "Unsupported OS" ;;
       esac ;;
    2) free -h ;;
    3) df -h ;;
    *) echo "Invalid" ;;
esac
```

**Output:** Varies by system

**Step-by-step:**
1. Outer case dispatches by top-level choice
2. CPU info (choice 1) has a nested inner case for OS-specific commands
3. Memory and disk are cross-platform

**Variations:**
- Can nest `case` inside `select` inside `while` — it's turtles all the way down
- Each level adds a new dimension of branching

### Example 10: extglob patterns in case

**Command:**
```bash
shopt -s extglob
input="hello42"
case "$input" in
    +([[:digit:]]))          echo "All digits" ;;
    +([[:alpha:]]))          echo "All letters" ;;
    *([[:alnum:]]))          echo "Alphanumeric" ;;
    *)                       echo "Contains special chars" ;;
esac
```

**Output:** `Alphanumeric`

**Step-by-step:**
1. `shopt -s extglob` enables extended glob patterns
2. `+([[:digit:]])` means "one or more digits" — doesn't match (has letters)
3. `+([[:alpha:]])` means "one or more letters" — doesn't match (has digits)
4. `*([[:alnum:]])` means "zero or more alnum" — matches!

**Variations:**
- `?(pattern)` — zero or one match
- `*(pattern)` — zero or more
- `+(pattern)` — one or more
- `@(p1|p2)` — exactly one of the patterns
- `!(pattern)` — anything except

### Example 11: Color-coded log level

**Command:**
```bash
log_level="ERROR"
case "$log_level" in
    INFO)    color='\e[32m' ;;   # green
    WARN)    color='\e[33m' ;;   # yellow
    ERROR)   color='\e[31m' ;;   # red
    DEBUG)   color='\e[36m' ;;   # cyan
    *)       color='\e[0m'  ;;
esac
printf "${color}[%s]%s\e[0m\n" "$log_level" "Something happened"
```

**Output:** `[ERROR]Something happened` (in red)

**Step-by-step:**
1. Maps each log level to an ANSI color
2. The choice is used later for colored output
3. `*` catch-all provides default (no color)

**Variations:**
- Can map to icons: `INFO) icon="ℹ️"`, `ERROR) icon="❌"`
- Can map to syslog priorities

### Example 12: select with timeout

**Command:**
```bash
PS3="Select in 5s: "
select opt in "Option A" "Option B" "Exit"; do
    [ -z "$opt" ] && { echo "Invalid/Timed out"; continue; }
    [ "$opt" = "Exit" ] && break
    echo "You chose: $opt"
done
```

**Input:** Wait 5 seconds, then type `2`

**Output:**
```
1) Option A
2) Option B
3) Exit
Select in 5s: 2
You chose: Option B
Select in 5s:
```

**Step-by-step:**
1. Menu displays normally
2. `select` blocks on `read`
3. User types `2` within no timeout (but you can externally wrap with `read -t`)
4. `$opt="Option B"`, executes branch

**Variations:**
- Add `read -t 5` wrapper: `if ! read -t 5; then echo "Timed out"; break; fi`

## Real-World Use Cases

### FOR the OS

- **Service management scripts**: `/etc/init.d/*` scripts are practically all `case "$1" in start|stop|restart...)`
- **System installation wizards**: Arch Linux installer, debconf prompts
- **Log rotation and parsing**: Classifying log entries by severity
- **User account management**: `useradd`, `usermod` wrappers with `case` for options
- **Package manager frontends**: `apt-get` subcommands
- **Configuration tools**: `raspi-config`, `nmtui`

### WITH the OS

- **`case` for filetype handling**: `case "$file" in *.jpg|*.png) ...` — dispatches by extension
- **`select` for device selection**: `select dev in /dev/sd*; do dd if=$dev of=image.img; done`
- **Signal handling with `kill`**: `case "$sig" in HUP|TERM|KILL) ...`
- **`select` for kernel selection**: Choosing which kernel to boot (though GRUB handles this)

### AGAINST the OS (Security Perspective)

- **Case pattern injection**: If an attacker controls the `word` in `case "$word" in`, they can't inject patterns (the patterns are fixed at parse time), but they can trigger different branches. This is less of an injection vector than unquoted eval.
- **`select` menu mask**: An interactive menu can hide what's really happening — a malicious script could present a fake selection menu that actually runs `sudo rm -rf /` disguised as "System Update."
- **Brute force via `select`**: A `select` loop that doesn't limit attempts can be abused to try many inputs rapidly.
- **Hidden fallthrough**: `;&` fallthrough can accidentally execute privileged blocks if a developer misconfigures pattern ordering.

### FOR DEFENSE

- **Always include `*)` default** in every `case` — unhandled inputs should be caught
- **Limit `select` attempts**: Wrap in a counter: `if ((++attempts > 3)); then echo "Too many attempts"; break`
- **Quote `$var` in `case "$var"`** — prevents glob expansion of the variable (though uncommon)
- **Use `shopt -s nocasematch`** for explicit case-insensitivity rather than enumerating `[Yy][Ee][Ss]`
- **Validate before switching**: Don't rely solely on `case` for security-critical input validation

## Memory Aids

- **`case`...`esac`** = `case` spelled backwards (like `if`/`fi`, `while`/`elihw`... OK, maybe not `elihw`)
- **`;;`** looks like two terminal posts — "stop here"
- **`;&`** looks like it's reaching into the next block — "fall through"
- **`;;&`** looks like it's poking the NEXT pattern — "continue matching"
- **`select`** = "select" from a list, like a restaurant menu
- **`PS3`** = "Prompt String 3" — the shell has PS1, PS2, PS3, PS4
- **`$REPLY`** = the user's "reply" — what they typed, not what it matched

## Trap Vault (8-12 traps)

### Trap 1: Missing `;;` causes fallthrough

**Problem:** Two branches execute when you expected only one.

**Bad Example:**
```bash
case "$opt" in
    start) echo "Starting..."
    stop)  echo "Stopping..." ;;
esac
```

**Root Cause:** Missing `;;` after start — the echo in `start` runs, then bash falls through to `stop`'s echo too (in versions before bash 4.0, this was a syntax error; modern bash treats a missing `;;` as fallthrough up to the next pattern).

**Fix:**
```bash
case "$opt" in
    start) echo "Starting..." ;;
    stop)  echo "Stopping..." ;;
esac
```

### Trap 2: Patterns are globs, not regex

**Problem:** Trying to match a regex pattern in `case`.

**Bad Example:**
```bash
case "$email" in
    .+@.+\..+) echo "Valid email" ;;    # doesn't work!
esac
```

**Root Cause:** `case` uses glob patterns, not regex. `.+` means "a dot followed by any character" in glob, not "one or more of anything."

**Fix:** Use `[[ $email =~ regex ]]` for regex, or globs for globs:
```bash
case "$email" in
    *@*.*) echo "Looks like email" ;;
esac
```

### Trap 3: `select` loops forever

**Problem:** Your `select` loop runs forever with no exit.

**Bad Example:**
```bash
select opt in "A" "B" "C"; do
    echo "You picked: $opt"
done
# Never exits!
```

**Root Cause:** `select` loops indefinitely. It requires `break` or `exit` to terminate.

**Fix:**
```bash
select opt in "A" "B" "C" "Exit"; do
    case "$opt" in
        "Exit") break ;;
        *)      echo "You picked: $opt" ;;
    esac
done
```

### Trap 4: Unquoted `$var` in case word

**Problem:** A variable with wildcards causes unexpected matching.

**Bad Example:**
```bash
var="*"
case $var in
    start) echo "Starting" ;;
esac
# $var expands to *, which expands to file list!
```

**Root Cause:** Without quotes, `$var` undergoes glob expansion before `case` evaluates it. If there are files in the CWD, `*` becomes the file list.

**Fix:** Always quote: `case "$var" in`

### Trap 5: `PS3` not set — unhelpful prompt

**Problem:** User sees `#? ` instead of a meaningful prompt.

**Bad Example:**
```bash
select opt in "A" "B"; do ... done
# Shows: #? 
```

**Root Cause:** Default `PS3` is `#? ` — cryptic.

**Fix:** `PS3="Choose an option: "`

### Trap 6: `select` with empty array

**Problem:** Menu doesn't show, loop exits immediately.

**Bad Example:**
```bash
files=()
select f in "${files[@]}"; do
    echo "You picked: $f"
done
echo "Loop ended immediately"
# Never enters loop body
```

**Root Cause:** If the list in `select` is empty, the loop body never executes.

**Fix:** Check for empty list:
```bash
files=()
if [[ ${#files[@]} -eq 0 ]]; then
    echo "No files found"
else
    select f in "${files[@]}"; do ... done
fi
```

### Trap 7: `COLUMNS` affects menu layout

**Problem:** Menu items are crammed on one line or spread weirdly.

**Root Cause:** `select` respects the `COLUMNS` terminal width setting to lay out items horizontally.

**Fix:** `COLUMNS=1` forces one item per line regardless of terminal width.

### Trap 8: Variables in patterns are expanded

**Problem:** A pattern containing a variable doesn't match as expected.

**Bad Example:**
```bash
ext="jpg"
case "photo.jpg" in
    *.$ext) echo "JPEG file" ;;
esac
# This actually works, but the issue is if $ext contains glob chars
```

**Root Cause:** Patterns ARE expanded — `$ext` becomes `jpg`, so the pattern `*.jpg` matches. But if `$ext` were `???`, it'd match any 3-char extension. This is usually what you want, but be aware of it.

**Fix:** Know your data — if `$ext` comes from user input, validate it first.

### Trap 9: `--` option parsing without break

**Problem:** Options after `--` are not treated as positional.

**Bad Example:**
```bash
while [[ $# -gt 0 ]]; do
    case "$1" in
        -f|--file) file="$2"; shift 2 ;;
        # forgot -- handler
    esac
done
```

**Root Cause:** No `--) shift; break` pattern — `--` is treated as an unknown option.

**Fix:**
```bash
case "$1" in
    --) shift; break ;;
esac
```

### Trap 10: Nested case with same patterns

**Problem:** Inner `case` matches outer's pattern instead of intended one.

**Bad Example:**
```bash
case "$1" in
    start)
        case "$2" in
            now)  echo "Starting now" ;;
            soon) echo "Starting soon" ;;
        esac
        ;;
    stop) echo "Stopping" ;;
esac
```
(This is actually fine — but gets confusing with many levels)

**Root Cause:** No actual problem, just readability. Deep nesting in case = deep confusion.

**Fix:** Extract inner cases into functions.

### Trap 11: `select` + `read` interaction

**Problem:** Mixing `select` with manual `read` inside the loop behaves oddly.

**Bad Example:**
```bash
select opt in "A" "B"; do
    read -p "Extra: " extra   # this steals input meant for select
done
```

**Root Cause:** `select` already uses `read`. An inner `read` grabs the next line of input intended for the next `select` iteration.

**Fix:** Handle all input in the select loop. Don't mix manual `read` inside `select`.

### Trap 12: Forgetting `shopt -s extglob` for complex patterns

**Problem:** Extended glob patterns fail silently.

**Bad Example:**
```bash
case "hello123" in
    +([[:alpha:]]) ) echo "Pure alpha" ;;
esac
# Empty output — no match
```

**Root Cause:** `+([...])` is an extglob pattern, disabled by default.

**Fix:** `shopt -s extglob` before the case statement.

## See It In The Wild

- **`/etc/init.d/functions`** — the canonical example of case-based service management
- **Most `git` aliases** use `case` for argument dispatch in wrapper scripts
- **Docker entrypoint scripts** — classic `case "$1" in` to handle different commands
- **Arch Linux's `pacman`** uses `case` extensively for option parsing in helper scripts
- **Your own `.bashrc`** probably has `case "$TERM" in` somewhere

**Try this now:**

1. `PS3='Pick: '; select x in {A,B,C}; do echo "Got $x ($REPLY)"; break; done` — instant menu
2. Write a one-liner that classifies a character: `c='x'; case "$c" in [[:upper:]]) echo up;; [[:lower:]]) echo low;; esac`
3. Run `select pid in $(pgrep bash); do kill -0 $pid && echo "Alive" || echo "Dead"; break; done`

## Check Your Understanding (5-7 questions)

1. What's the default terminator behavior in `case` — fallthrough or break?
2. How do you make `case` match multiple patterns in one branch?
3. What shell variable holds the user's typed number in a `select` loop?
4. Why would `select` immediately exit without showing a menu?
5. What's the difference between `;&` and `;;&` in fallthrough behavior?
6. What happens if you forget `PS3` in a `select` loop?
7. How would you match a `case` pattern that checks for exactly 5 digits?

---
*"`case` is the Swiss Army knife of control flow — it cuts, it slices, it makes decisions, and it never complains about your variable naming."*
