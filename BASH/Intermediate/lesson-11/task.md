# Task 11: `case` & `select` Menus

You'll build an interactive system administration menu (`sysadmin.sh`), a robust argument parser (`argparser.sh`), and an input validator (`validate_input.sh`). These are the building blocks of every command-line tool you'll ever write.

## Sub-tasks

### 1. System admin menu — `sysadmin.sh`

Create an interactive `select` menu that provides system administration functions. Loop until the user selects Exit.

**Requirements:**
- Use `select` with the options listed below
- Use a `case` inside the `select` block to dispatch each option
- Set a meaningful `PS3` prompt
- Include a `*)` catch-all for invalid selections
- Show the menu title "=== System Admin Menu ===" before the menu
- Display "Goodbye." when exiting
- Options:
  1. Show disk usage (`df -h`)
  2. Show logged-in users (`who`)
  3. Show running processes (`ps aux --sort=-%mem | head -10`)
  4. Show system info (`uname -a`)
  5. Show network connections (`ss -tuln` or `netstat -tuln`)
  6. Backup home directory (`tar czf /tmp/backup-$(date +%F).tar.gz $HOME 2>/dev/null` with feedback)
  7. Exit

**Edge cases:**
- User enters empty line → menu should redisplay without error
- User enters text instead of number → `$action` is empty, `$REPLY` has text — handle gracefully
- `select` loops forever — provide Exit break

**Expected:**
```bash
$ ./sysadmin.sh
=== System Admin Menu ===
1) Show disk usage
2) Show logged-in users
3) Show running processes
4) Show system info
5) Show network connections
6) Backup home directory
7) Exit
Action [1-7]: 1
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   50G   50G  50% /
Action [1-7]: 7
Goodbye.
```

### 2. Argument parser — `argparser.sh`

Write an argument parser that handles short and long options using `case`.

**Requirements:**
- Parse these options:
  - `-f file` or `--file file` — input file
  - `-o dir` or `--output dir` — output directory
  - `-v` or `--verbose` — enable verbose output
  - `-h` or `--help` — show usage and exit
- Handle `--` to stop option parsing (everything after is positional)
- Unknown options → print error and exit
- Print final parsed values
- Accept an additional positional argument (not starting with `-`)

**Edge cases:**
- Missing argument after `-f` or `--file` — detect and error
- Multiple flags combined: `-vf file` — handle this (optionally)
- Help option anywhere in the arg list should show usage and exit
- Trailing `--` should separate options from positional arguments

**Expected:**
```bash
$ ./argparser.sh -f data.txt -v --output /tmp extra_arg
File:      data.txt
Output:    /tmp
Verbose:   ON
Positional: extra_arg

$ ./argparser.sh -h
Usage: argparser.sh [OPTIONS] [positional]
Options:
  -f, --file FILE     Input file
  -o, --output DIR    Output directory
  -v, --verbose       Verbose output
  -h, --help          Show this help

$ ./argparser.sh -x
Error: Unknown option: -x

$ ./argparser.sh --file
Error: --file requires an argument

$ ./argparser.sh -- --file
File:      (not set)
Output:    (not set)
Verbose:   OFF
Positional: --file
```

### 3. Input validator — `validate_input.sh`

Write a function `validate_input()` that uses `case` to classify input strings.

**Requirements:**
- Classify inputs into categories using glob patterns in `case`:
  - Email: contains `@` and a dot after the `@`
  - Phone (US): pattern `XXX-XXX-XXXX` where X is a digit
  - URL: starts with `http://` or `https://`
  - Date (ISO): pattern `YYYY-MM-DD`
  - IP address: four numbers 0-255 separated by dots (basic check)
  - Unknown: everything else
- Read input from a file (one per line) or from command-line arguments
- Print classification for each
- Count totals per category

**Edge cases:**
- Empty input → Unknown
- Input matching multiple patterns — first match wins (case pattern order matters)
- Make phone pattern match with/without area code parens: `(123) 456-7890` (bonus)

**Expected:**
```bash
$ cat inputs.txt
alice@example.com
555-123-4567
https://example.com
2025-12-31
192.168.1.1
just some text

$ ./validate_input.sh inputs.txt
alice@example.com       EMAIL
555-123-4567            PHONE
https://example.com     URL
2025-12-31              DATE
192.168.1.1             IP_ADDRESS
just some text          UNKNOWN
---
Summary:
EMAIL:      1
PHONE:      1
URL:        1
DATE:       1
IP_ADDRESS: 1
UNKNOWN:    1
TOTAL:      6
```

## Solution Approaches

### Approach A: Simple sequential
1. `argparser.sh` first — pure `case` on `$1`, simplest logic
2. `validate_input.sh` — `case` with glob patterns, introduces pattern matching
3. `sysadmin.sh` — combines `select` + `case`, most complex

### Approach B: Menu-first
1. `sysadmin.sh` — most fun, immediate feedback with interactive menu
2. `validate_input.sh` — builds pattern matching skill
3. `argparser.sh` — most methodical, requires `shift` logic

### Approach C: All-at-once pattern practice
1. Write all three `case` patterns first (shell, validation, dispatch)
2. Add `select` wrapper to sysadmin menu
3. Add `shift` loop to argparser

<details>
<summary>Hint 1: Select menu loop structure</summary>

```bash
PS3="Action [1-7]: "
echo "=== System Admin Menu ==="
select action in "Show disk" "Show users" "Exit"; do
    case "$action" in
        "Show disk")  df -h ;;
        "Show users") who ;;
        "Exit")       echo "Goodbye."; break ;;
        *)            echo "Invalid selection: $REPLY" ;;
    esac
done
```
</details>

<details>
<summary>Hint 2: Arg parser with shift</summary>

```bash
file=""
output=""
verbose=0
positional=()

while [[ $# -gt 0 ]]; do
    case "$1" in
        -f|--file)
            [[ -z "$2" || "$2" =~ ^- ]] && { echo "Error: $1 requires an argument"; exit 1; }
            file="$2"; shift 2 ;;
        -o|--output)
            [[ -z "$2" || "$2" =~ ^- ]] && { echo "Error: $1 requires an argument"; exit 1; }
            output="$2"; shift 2 ;;
        -v|--verbose) verbose=1; shift ;;
        -h|--help)    usage; exit 0 ;;
        --)           shift; positional+=("$@"); break ;;
        -*)           echo "Error: Unknown option: $1"; exit 1 ;;
        *)            positional+=("$1"); shift ;;
    esac
done
```
</details>

<details>
<summary>Hint 3: Input validation patterns</summary>

```bash
validate_input() {
    case "$1" in
        *@*.*)                        echo "EMAIL" ;;
        [0-9][0-9][0-9]-[0-9][0-9][0-9]-[0-9][0-9][0-9][0-9]) echo "PHONE" ;;
        http://*|https://*)           echo "URL" ;;
        [0-9][0-9][0-9][0-9]-[0-9][0-9]-[0-9][0-9]) echo "DATE" ;;
        *.*.*.*)                      echo "IP_ADDRESS" ;;
        *)                            echo "UNKNOWN" ;;
    esac
}
```

Order matters — `*@*.*` before `*.*.*.*` because an email also matches `*.*.*.*`!
</details>

<details>
<summary>Hint 4: Read input from file or args</summary>

```bash
if [[ $# -eq 1 && -f "$1" ]]; then
    mapfile -t inputs < "$1"
else
    inputs=("$@")
fi
```
</details>

<details>
<summary>Hint 5: Handling combined short options (-vf)</summary>

```bash
case "$1" in
    -v*) 
        # Check each character after -v
        flags="${1#-}"
        for ((i=0; i<${#flags}; i++)); do
            case "${flags:$i:1}" in
                v) verbose=1 ;;
                f) file="$2"; shift; break 2 ;;  # break out of both loops
                *) echo "Unknown flag: -${flags:$i:1}" ;;
            esac
        done
        shift ;;
esac
```
This is complex — save it for bonus.
</details>

## Bonus Challenges

1. **Nested menus**: Add sub-menus to sysadmin.sh (e.g., "Show disk" → submenu: "Show all", "Show / only", "Show tmpfs")
2. **Combined short options**: `-vf file.txt` should set verbose AND parse `-f file.txt`
3. **Auto-completion hints**: Use `read -e` or `compgen` to give tab completion in the select menu
4. **Persistent history**: Log all menu selections to `~/.sysadmin_history` with timestamps
5. **Color-coded validation**: Output EMAIL in green, PHONE in yellow, UNKNOWN in red
6. **Redacted input**: For argparser, add `--password` that shows `****` on entry (use `read -s`)

## Expected Output Summary

```
sysadmin.sh:
  Interactive select menu
  7 options including Exit
  Loops until Exit selected
  Handles invalid input gracefully

argparser.sh:
  Parses -f, -o, -v, -h with short and long forms
  Handles -- separator
  Validates required arguments
  Reports unknown options

validate_input.sh:
  Classifies input into 6 categories
  Uses case with glob patterns
  Pattern order matters!
  Summary statistics
```

## Self-Check

- How do you match multiple patterns (OR) in a single `case` branch?
- What happens if you omit `;;` in a `case` statement?
- Why does `select` need a `break` to exit?
- What is `PS3` for and what happens if you don't set it?
- How do you pattern-match a range of characters in a `case` pattern?
- What's the difference between `case` patterns and regex patterns?
- Why does the catch-all `*` branch need to be last in a `case`?
- How does `select` behave when the user enters a non-numeric value?
