# Lesson 8: Variables & Quoting

## History & Origins

Variables in Unix shells started simple. The original Bourne shell (1977) by Stephen Bourne at Bell Labs introduced `$VAR` for variable expansion, `${VAR}` for braced expansion, and the simple assignment `VAR=value`. The only requirement: no spaces around `=`.

The C shell (csh, 1978) by Bill Joy at UC Berkeley introduced a different syntax: `set var = value` (with spaces allowed) and `$var` for access. The C shell also introduced arrays and the `$?var` test for existence.

When the Free Software Foundation created bash (1989) as a free replacement for the Bourne shell, it incorporated features from both the Bourne shell and the Korn shell (ksh, 1983 by David Korn at Bell Labs). The `${var:-default}` syntax came from ksh, as did `${var/pattern/replacement}` and other parameter expansions.

Quoting has been a shell feature since the beginning. Single quotes (`'`) prevent ALL expansion. Double quotes (`"`) prevent word splitting and globbing but allow variable expansion, command substitution, and arithmetic expansion. The escape character (`\`) was borrowed from C.

The rule "always quote your variables" is the single most important shell scripting lesson. It was codified in the early days of Unix when filenames with spaces became more common (thanks to macOS and Windows users creating files on Unix systems). Before that, many shell scripts were written without quotes because filenames rarely contained spaces.

**Fun anecdote:** The `$` sign for variable expansion came from the ed editor's regex syntax. In ed, `$` meant "end of line." The Bourne shell used `$` as a prefix for variables, and it stuck. The `${}` syntax was added because `$var_text` was ambiguous — does it mean `$var` + `_text` or `${var_text}`? The braces disambiguate. Also, the famous "Bobby Tables" XKCD comic (`Robert'); DROP TABLE Students;--`) is a SQL injection joke, but the same principle applies to shell quoting — unquoted user input can execute arbitrary commands.

## Syntax Reference

### Variable Assignment

```bash
VAR=value                # Assignment (NO spaces around =)
VAR=                     # Set to empty string (not unset)
VAR="value with spaces"  # Quoted value
VAR='literal $value'     # Single-quoted value
let "x = 2 + 3"         # Arithmetic (old style)
(( x = 2 + 3 ))          # Arithmetic (bash preferred)
declare -i x=5           # Declare integer
declare -r x=5           # Declare readonly (constant)
declare -a arr=(a b c)   # Array
declare -A map=([k]=v)   # Associative array
local VAR=value          # Local variable in function
export VAR=value         # Export to environment
readonly VAR=value       # Make readonly
```

### Variable Expansion

```bash
$VAR                     # Simple expansion
${VAR}                   # Braced expansion
${VAR:-default}          # Use default if VAR is unset or null
${VAR:=default}          # Assign default if VAR is unset or null
${VAR:?error}            # Error if VAR is unset or null
${VAR:+alternate}        # Use alternate if VAR is set
${#VAR}                  # Length of VAR
${VAR:offset}            # Substring starting at offset
${VAR:offset:length}     # Substring of given length
${VAR#pattern}           # Remove shortest prefix matching pattern
${VAR##pattern}          # Remove longest prefix matching pattern
${VAR%pattern}           # Remove shortest suffix matching pattern
${VAR%%pattern}          # Remove longest suffix matching pattern
${VAR/old/new}           # Replace first match of old with new
${VAR//old/new}          # Replace ALL matches of old with new
${VAR/#old/new}          # Replace match at start
${VAR/%old/new}          # Replace match at end
${VAR^}                  # Uppercase first character
${VAR^^}                 # Uppercase ALL characters
${VAR,}                  # Lowercase first character
${VAR,,}                 # Lowercase ALL characters
```

### Special Variables

```bash
$0                       # Script name/shell name
$1, $2, ...              # Positional parameters
$#                       # Number of positional parameters
$@                       # All positional parameters (separate words)
$*                       # All positional parameters (single word)
$?                       # Exit code of last command
$$                       # PID of current shell
$!                       # PID of last background process
$-                       # Current shell options
$_                       # Last argument of last command
$IFS                     # Internal Field Separator (default: space, tab, newline)
$PATH                    # Command search path
$HOME                    # User's home directory
$PWD                     # Current working directory
$OLDPWD                  # Previous working directory
$RANDOM                  # Random number (0-32767)
$LINENO                  # Current line number in script
$SECONDS                 # Seconds since shell started
$BASH_VERSION            # Bash version string
$BASH_SOURCE[0]          # Source file of current script/function
$FUNCNAME                # Current function name
```

### Quoting

```bash
'literal'                # Single quotes: ALL characters literal
"$VAR"                   # Double quotes: $, `, \ preserved; others literal
$'string'                # ANSI-C quoting: \n, \t, \x00, etc.
"$(cmd)"                 # Command substitution inside double quotes
\char                    # Escape next character
```

### Arrays

```bash
arr=(a b c)              # Indexed array
arr[0]=a                 # Set element
echo ${arr[0]}           # Get element
echo ${arr[@]}           # All elements (separate words)
echo ${arr[*]}           # All elements (single word)
echo ${#arr[@]}          # Array length
echo ${!arr[@]}          # List of indices
arr+=(d e)               # Append elements
declare -A map           # Associative array (declare -A required)
map[key]=value           # Set associative element
```

### Command Substitution

```bash
$(command)               # Preferred form
`command`                # Legacy backtick form (avoid)
```

### Arithmetic Expansion

```bash
$(( 2 + 3 ))             # Arithmetic expansion
$(( x++ ))               # Post-increment
$(( ++x ))               # Pre-increment
$(( RANDOM % 6 + 1 ))   # Random die roll
```

## Under the Hood

### Shell expansion order

The shell processes a command line through a specific sequence of expansions. Understanding this order explains ALL quoting behavior:

1. **Brace expansion** `{a,b}` -> `a b`
2. **Tilde expansion** `~` -> `/home/user`
3. **Parameter expansion** `$VAR`, `${VAR}`
4. **Command substitution** `$(cmd)`, `` `cmd` ``
5. **Arithmetic expansion** `$(( expr ))`
6. **Word splitting** — splits results on `$IFS` (space, tab, newline by default)
7. **Pathname expansion (globbing)** — `*`, `?`, `[abc]` are expanded
8. **Quote removal** — all unquoted backslash, single-quote, double-quote are removed

### What happens during `echo "$HOME"`

1. Shell tokenizes: command=`echo`, argument=`"$HOME"`.
2. Tilde expansion: no tilde found.
3. Parameter expansion: `$HOME` is expanded to `/home/phd` (still inside double quotes).
4. Command substitution: none.
5. Arithmetic expansion: none.
6. Word splitting: **SKIPPED because of double quotes** — `"/home/phd"` stays as one word (no splitting on spaces in the value, even though there are none).
7. Pathname expansion: **SKIPPED because of double quotes** — `/home/phd` has no glob characters anyway.
8. Quote removal: the double quotes are removed.
9. `echo` receives one argument: `/home/phd`.

### What happens during `echo $HOME` (unquoted)

1-5: Same as above.
6. Word splitting: the result `/home/phd` is split on `$IFS`. No spaces here, so it stays as one word.
7. Pathname expansion: `/home/phd` contains no `*`, `?`, or `[`, so no expansion.
8. Quote removal: nothing to remove.
9. Result is the same — for THIS case. But `$HOME` with spaces would differ.

### What happens during `echo $HOME/*`

1-5: `$HOME` expands to `/home/phd`, giving the token `/home/phd/*`.
6. Word splitting: no splitting (no IFS chars in `/home/phd/*`).
7. Pathname expansion: the shell calls `glob("/home/phd/*")` which matches all files.
8. Quote removal: none.
9. `echo` receives the list of files in `/home/phd`.

But `echo "$HOME/*"`:
- Steps 1-5: same expansion of `$HOME`.
- Step 6-7: **Skipped because of double quotes**.
- Token stays as literal `/home/phd/*`.
- `echo` prints `/home/phd/*`.

### Word splitting mechanics

Word splitting happens on the characters in `$IFS` (Internal Field Separator). Default: space, tab, newline.

```bash
$ IFS=:
$ x="a:b:c"
$ echo $x
a b c    # Split on colons!
```

When you change `IFS`, you change how unquoted variables split. This is both powerful and dangerous.

### The subshell environment

When a script sets a variable:
```bash
VAR="hello"
```
This sets it in the current shell. Child shells (processes spawned by the script) do NOT see it unless it's exported.

```bash
export VAR="hello"      # Seen by child processes
VAR="hello"             # NOT seen by children
export VAR              # Export an existing variable
```

`export` marks the variable for inclusion in the environment of child processes. The kernel's `execve()` syscall takes an array of environment strings — `export` ensures the variable is in that array.

## Core Examples

### Example 1: Basic assignment and expansion

```bash
$ NAME="John Doe"
$ echo "Hello, $NAME"
Hello, John Doe
$ echo 'Hello, $NAME'
Hello, $NAME
```

**Step by step:**
1. `NAME="John Doe"` assigns the string `John Doe` to variable `NAME`.
2. `echo "Hello, $NAME"`: inside double quotes, `$NAME` expands → `echo "Hello, John Doe"` → prints.
3. `echo 'Hello, $NAME'`: inside single quotes, `$NAME` is literal → prints `$NAME`.

**What if:**
```bash
$ NAME=John Doe      # Error: tries to run "Doe" as a command
bash: Doe: command not found
$ NAME="John Doe"    # Correct: quote the value
```

### Example 2: Braces disambiguate

```bash
$ FRUIT="apple"
$ echo "I like $FRUITs"        # $FRUITs — not a defined variable! Empty.
I like
$ echo "I like ${FRUIT}s"      # ${FRUIT} + "s"
I like apples
```

**Step by step:**
1. `$FRUITs` is parsed as variable name `FRUITs` (not `FRUIT` + `s`). `FRUITs` has no value.
2. `${FRUIT}s` parses as `FRUIT` with a literal `s` suffix.

**What if:**
```bash
$ echo "There are ${#FRUIT} letters in $FRUIT"
There are 5 letters in apple
```

### Example 3: Word splitting danger

```bash
$ FILES="file1.txt  file2.txt  important file.txt"
$ ls $FILES
ls: cannot access 'file1.txt': No such file or directory
ls: cannot access 'file2.txt': No such file or directory
ls: cannot access 'important': No such file or directory
ls: cannot access 'file.txt': No such file or directory
$ ls "$FILES"
ls: cannot access 'file1.txt  file2.txt  important file.txt': No such file or directory
```

**Step by step:**
1. Unquoted `$FILES`: word splitting splits on spaces into 4 words: `file1.txt`, `file2.txt`, `important`, `file.txt`.
2. Also: pathname expansion checks each word for globs (none found).
3. `ls` tries to open all 4 separate files.
4. Quoted `"$FILES"`: one argument: `"file1.txt  file2.txt  important file.txt"` (with double spaces preserved).

**What if:**
```bash
$ FILES="*.txt"       # Unquoted: will glob!
$ ls $FILES           # Lists all .txt files (if any)
$ ls "$FILES"         # Tries to open literal file "*.txt"
```

### Example 4: Default value with :-

```bash
$ echo "Hello, ${NAME:-World}"
Hello, World
$ NAME="Alice"
$ echo "Hello, ${NAME:-World}"
Hello, Alice
```

**Step by step:**
1. `${VAR:-default}`: if `$VAR` is unset or empty, use `default`. Otherwise use `$VAR`.
2. First call: `NAME` is unset → prints "World".
3. Second call: `NAME` is "Alice" → prints "Alice".

**What if:**
```bash
$ NAME=""             # Empty
$ echo "${NAME:-World}"
World                  # :- triggers on empty too!
$ echo "${NAME-World}"
                       # Without colon: only triggers on unset, not empty
```

### Example 5: String manipulation

```bash
$ FILENAME="data-2024-01-15.tar.gz"
$ echo "${FILENAME%.tar.gz}"    # Remove suffix
data-2024-01-15
$ echo "${FILENAME##*.}"        # Extension (remove longest prefix up to dot)
gz
$ echo "${FILENAME#*.}"         # Remove shortest prefix up to dot
tar.gz
$ echo "${FILENAME%%.*}"        # Remove longest suffix from dot
data-2024-01-15
```

**Step by step:**
1. `%` removes the SHORTEST matching suffix. `*.tar.gz` matches exactly — removed.
2. `##` removes the LONGEST matching prefix. `*.` matches "data-2024-01-15.tar." → leaves "gz".
3. `#` removes the SHORTEST matching prefix. `*.` matches "data-2024-01-15." → leaves "tar.gz".
4. `%%` removes the LONGEST matching suffix. `.*` matches ".tar.gz" → leaves "data-2024-01-15".

**What if:**
```bash
$ echo "${FILENAME/2024/2025}"              # Replace first 2024 with 2025
data-2025-01-15.tar.gz
$ echo "${FILENAME//[0-9]/X}"               # Replace all digits with X
data-XXXX-XX-XX.tar.gz
$ echo "${FILENAME^^}"                      # Uppercase all
DATA-2024-01-15.TAR.GZ
```

### Example 6: Arrays

```bash
$ files=("report.pdf" "data.csv" "notes.txt")
$ echo "First: ${files[0]}"
First: report.pdf
$ echo "All: ${files[@]}"
All: report.pdf data.csv notes.txt
$ echo "Count: ${#files[@]}"
Count: 3
$ files+=("summary.doc")                    # Append
$ echo "Now: ${#files[@]}"
Now: 4
```

**Step by step:**
1. Array created with 3 elements.
2. `${files[0]}` accesses first element (0-indexed).
3. `${files[@]}` expands to all elements as separate words.
4. `${#files[@]}` gives the array length.
5. `files+=("...")` appends a new element.

**What if:**
```bash
$ for f in "${files[@]}"; do echo "$f"; done    # Correct: preserves spaces
$ for f in ${files[@]}; do echo "$f"; done       # Wrong: word splitting!
```

### Example 7: Command substitution

```bash
$ DATE=$(date +%Y-%m-%d)
$ echo "Today: $DATE"
Today: 2026-07-31
$ OLD=$(pwd)
$ cd /tmp && ls && cd "$OLD"
```

**Step by step:**
1. `$(date +%Y-%m-%d)` runs `date +%Y-%m-%d` in a subshell.
2. The output ("2026-07-31") is assigned to `DATE`.
3. `$OLD` captures the current directory path.
4. `cd "$OLD"` returns to the captured path (quoted for safety).

**What if:**
```bash
$ FILES=$(ls *.txt)         # DON'T DO THIS — breaks with spaces
$ FILES=(*.txt)               # RIGHT: glob expands to array
$ echo "$(pwd)"               # Unnecessary — just use pwd
$ echo "$(echo "$(echo test)")" # Nesting works!
```

### Example 8: Export and subshells

```bash
$ export DB_URL="postgres://localhost/mydb"
$ python3 -c "import os; print(os.environ['DB_URL'])"
postgres://localhost/mydb
$ (unset DB_URL; echo "Child: $DB_URL")
Child: 
$ echo "Parent: $DB_URL"
Parent: postgres://localhost/mydb
```

**Step by step:**
1. `export DB_URL` adds it to the environment of child processes.
2. `python3` (a child) can access it via `os.environ`.
3. In a subshell `( ... )`, `unset DB_URL` only affects the subshell.
4. The parent's `DB_URL` is unchanged.

**What if:**
```bash
$ DB_URL="temporary" python3 -c "import os; print(os.environ['DB_URL'])"
temporary
# One-off environment: set for this command only
```

### Example 9: ANSI-C quoting

```bash
$ echo $'Tab:\tseparated\tcolumns'
Tab:	separated	columns
$ echo $'Line 1\nLine 2\nLine 3'
Line 1
Line 2
Line 3
$ echo $'\x48\x65\x6c\x6c\x6f'
Hello
```

**Step by step:**
1. `$'...'` enables escape sequences: `\t` = tab, `\n` = newline, `\xNN` = hex byte.
2. Useful for setting variables with special characters.
3. Without `$'`, `\t` is literal backslash-t.

**What if:**
```bash
$ echo 'Tab:\tseparated'                 # Literal \t
Tab:\tseparated
$ echo "Tab:\tseparated"                 # Also literal \t in bash (except with printf)
Tab:\tseparated
$ printf "Tab:\tseparated\n"             # printf interprets escapes
Tab:	separated
```

### Example 10: Indirect expansion

```bash
$ COLOR="red"
$ VARNAME="COLOR"
$ echo ${!VARNAME}
red
```

**Step by step:**
1. `VARNAME` contains the string `"COLOR"`.
2. `${!VARNAME}` expands `$VARNAME` to `COLOR`, then expands `$COLOR` to `red`.
3. This is "indirect" or "variable indirection."

**What if:**
```bash
$ # Before bash 2.0, you'd use:
$ eval "echo \$$VARNAME"    # Danger: eval can execute anything!
red
$ # Prefer ${!VARNAME} for safety
```

### Example 11: $@ vs $*

```bash
$ set -- "file one.txt" "file two.txt" "file three.txt"
$ for f in "$@"; do echo "File: $f"; done
File: file one.txt
File: file two.txt
File: file three.txt

$ for f in $*; do echo "File: $f"; done
File: file
File: one.txt
File: file
File: two.txt
File: file
File: three.txt
```

**Step by step:**
1. `"$@"` preserves each argument as a separate word, even with spaces.
2. `$*` (unquoted) concatenates all args with first IFS char (space by default), then word-splits.
3. `"$*"` (quoted) concatenates all args into ONE string.

**What if:**
```bash
$ echo "Number of args: $#"
Number of args: 3
$ echo "All args: $*"
All args: file one.txt file two.txt file three.txt
```

### Example 12: readonly and declare

```bash
$ readonly PI=3.14159
$ PI=3
bash: PI: readonly variable
$ declare -l DOWN=  # Declare lowercase
$ DOWN="HELLO"
$ echo "$DOWN"
hello
$ declare -u UP=
$ UP="hello"
$ echo "$UP"
HELLO
$ declare -i NUM=42
$ NUM="20+22"
$ echo "$NUM"
42      # Automatically evaluated as arithmetic!
```

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`export PATH=$PATH:/custom/bin`** — extend PATH for session.
- **`BACKUP_DIR="/backup/$(date +%Y%m%d)" && mkdir -p "$BACKUP_DIR"`** — timestamped backup directory.
- **`if [ -z "$HOSTNAME" ]; then HOSTNAME=$(hostname); fi`** — default hostname.
- **`readonly TMPDIR=/tmp`** — protect critical variables.
- **`for user in "${users[@]}"; do userdel "$user"; done`** — iterate user list.

### 2. WITH the OS — Development

- **`export NODE_ENV=production`** — set environment for Node.js apps.
- **`DB_NAME=${DB_NAME:-"myapp_development"}`** — default database name.
- **`echo "${PWD##*/}"`** — current directory basename (without full path).
- **`filename="${1%.*}"`** — strip extension from a filename argument.
- **`IFS=$'\n'`** — set IFS to newline only for iterating file lists.

### 3. AGAINST the OS — Exploitation

- **Shell injection via unquoted vars**: `user_input="; rm -rf /"; cmd="ls $user_input"` — if unquoted, `ls` runs and then `rm -rf /` runs.
- **`export LD_PRELOAD=/tmp/evil.so`** — library injection (if running a setuid/root binary).
- **`IFS=/; read -ra parts <<< "/etc/passwd"; echo "${parts[@]}"`** — path splitting.
- **`PATH=/tmp:$PATH; sudo ./vulnerable_program`** — hijack shared library loading.

### 4. FOR DEFENSE — Detection & Auditing

- **`readonly PATH=/usr/bin:/bin`** in security-critical scripts — prevent PATH hijacking.
- **`set -u`** (nounset) — script fails on undefined variables, catching typos.
- **`grep -r '\${\w*:-' scripts/`** — find variables with defaults (may hide config issues).
- **`auditctl -w /etc/profile -p wa -k env_changes`** — monitor environment modifications.
- **`env -i bash -c 'echo $PATH'`** — run with empty environment to detect hardcoded paths.

## Memory Aids

### Mnemonics

- **`$`** = "expand this." Think of it as "give me the value of..."
- **`"`** = "partial protection" — let variables through, but stop splitting/glob.
- **`'`** = "full protection" — everything literal, nothing gets through.
- **`\`** = "escape the next character" — make it literal.
- **`${VAR:-default}`** = "use VAR, or if VAR is empty, use default."
- **`${VAR##*/}`** = "strip everything up to the last /" (like `basename`).
- **`${VAR%.*}`** = "strip everything from the last dot" (strip extension).

### Quoting decision flowchart

```
Does the value contain:
  - Spaces/tabs/newlines? → MUST quote
  - Wildcards (* ? [])? → MUST quote if literal
  - Variable references? → Use double quotes
  - Nothing special? → Quoting still recommended

Always double-quote variable expansions unless you EXPLICITLY want word splitting or globbing.
```

### Common confusions

- **`$@` vs `$*`**: `"$@"` preserves argument boundaries (the golden rule). `"$*"` makes one string.
- **`${HOME}` vs `$HOME`**: Same in simple cases. Use `{}` when followed by alphanumeric or underscore.
- **`$(cmd)` vs `` `cmd` ``**: Same result. `$()` nests, is easier to read, and is preferred.
- **`eval` is dangerous**: `eval "echo $FOO"` where `FOO` contains `; rm -rf /` runs the `rm`.
- **`export` vs `setenv`**: In bash, use `export`. `setenv` is csh syntax.
- **`declare -a` vs `declare -A`**: `-a` = indexed array (like a list), `-A` = associative array (like a hash/dict).

## Trap Vault

### Trap 1: Spaces around = in assignment

**Problem:** `VAR = value` doesn't assign — it runs a command.

**Example:**
```bash
$ VAR = "hello"
bash: VAR: command not found
```

**Why:** The shell parses `VAR = "hello"` as command `VAR` with arguments `=` and `"hello"`. No spaces are allowed around `=` in assignments.

**Fix:**
```bash
$ VAR="hello"    # Correct: no spaces
$ VAR= "hello"   # Wrong: VAR set to empty, then runs "hello" as command
```

### Trap 2: Unquoted variable in test

**Problem:** `[ $VAR == "x" ]` fails when `$VAR` is empty or has spaces.

**Example:**
```bash
$ VAR=""
$ [ $VAR == "x" ] && echo "match"
bash: [: ==: unary operator expected
```

**Why:** When `$VAR` is empty, the command becomes `[ == "x" ]` — three tokens instead of four. `[` sees `==` as the first argument and expects a unary operator.

**Fix:**
```bash
$ [ "$VAR" == "x" ] && echo "match"  # Always quote in tests
$ [[ $VAR == "x" ]] && echo "match"  # [[ ]] handles empty safely
```

### Trap 3: Forgetting to export

**Problem:** Child processes can't see unexported variables.

**Example:**
```bash
$ MYVAR="secret"
$ python3 -c "import os; print(os.environ.get('MYVAR'))"
None
```

**Why:** Without `export`, `MYVAR` is only in the shell's internal namespace, not in the environment block passed to `execve()` for child processes.

**Fix:**
```bash
$ export MYVAR="secret"
$ python3 -c "import os; print(os.environ.get('MYVAR'))"
secret

# Or set for one command:
$ MYVAR="secret" python3 -c "import os; print(os.environ['MYVAR'])"
secret
```

### Trap 4: `${var}` vs `$var` with adjacent characters

**Problem:** `$var_text` looks for variable `var_text`, not `var` + `_text`.

**Example:**
```bash
$ var="hello"
$ echo "$var_text"       # $var_text — no such variable
$ echo "${var}_text"     # hello_text
```

**Why:** Variable names can contain underscores. `$var_text` is the variable `var_text`. Braces `${var}` delimit the name explicitly.

**Fix:** Always use `${var}` when followed by alphanumeric or `_`:
```bash
$ echo "${var}_text"
$ echo "${var}text"
```

### Trap 5: Word splitting on unquoted $@

**Problem:** `$@` without quotes can cause word splitting in arguments.

**Example:**
```bash
$ set -- "file with spaces.txt" "another.txt"
$ for f in $@; do echo "$f"; done
file
with
spaces.txt
another.txt
```

**Why:** `$@` expands to the arguments, but without quotes, word splitting re-splits them on spaces.

**Fix:**
```bash
$ for f in "$@"; do echo "$f"; done    # CORRECT: preserves arguments
file with spaces.txt
another.txt
```

### Trap 6: Using backticks instead of $()

**Problem:** Backticks don't nest well and are harder to read.

**Example:**
```bash
$ echo `echo \`echo "nested"\``    # A nightmare
$ echo $(echo $(echo "nested"))    # Clean
```
Also, backticks have different escaping behavior:
```bash
$ echo `echo \$HOME`     # $HOME literal? Or expanded?
$ echo $(echo \$HOME)    # Always literal \$
```

**Fix:** Always use `$(...)`:
```bash
$ DATE=$(date +%Y-%m-%d)
$ FILES=$(ls -la)
```

### Trap 7: `IFS` changes break everything

**Problem:** Changing `IFS` can break all subsequent unquoted expansions.

**Example:**
```bash
$ IFS=":"
$ PATH="/usr/bin:/bin:/usr/local/bin"
$ echo $PATH
/usr/bin /bin /usr/local/bin    # Split on colons! Looks OK?
$ ls $PATH    # Tries to list three separate directories — works here
$ IFS="x"     # Now "x" is a separator
$ echo "exit"    # Still works — IFS doesn't affect literal strings
$ VAR="abc"    # But "${VAR}" and other expansions might behave weirdly
```

**Fix:** Save and restore IFS:
```bash
$ OLDIFS="$IFS"
$ IFS=":"
# ... do colon-splitting work ...
$ IFS="$OLDIFS"
```

### Trap 8: Command substitution trailing newlines stripped

**Problem:** `$(cmd)` strips trailing newlines.

**Example:**
```bash
$ echo "Hello" > /tmp/data.txt
$ content=$(cat /tmp/data.txt)
$ echo "$content" | wc -l
1    # BUT the original file had 1 line + 1 newline
# The trailing newline from echo is back, but the file's trailing newline was stripped!
```

**Why:** Bash strips trailing newlines from command substitution output. This is by design — many commands output a final newline that isn't "content."

**Fix:** Add a suffix and strip it:
```bash
$ content=$(cat /tmp/data.txt; echo x)
$ content="${content%x}"
```

### Trap 9: Exported variables in scripts called by cron

**Problem:** Cron scripts don't have the same environment as interactive shells.

**Example:**
```bash
# In ~/.bashrc:
export MYAPP_HOME=/opt/myapp

# In crontab:
* * * * * /home/user/script.sh  # script.sh can't see MYAPP_HOME!

# In script.sh:
echo "MYAPP_HOME=$MYAPP_HOME"   # Empty!
```

**Why:** Cron runs with a minimal environment. It does NOT source `.bashrc`, `.bash_profile`, or any interactive shell startup files.

**Fix:** Source the environment explicitly in the script:
```bash
#!/bin/bash
source /home/user/.bashrc
# or set directly:
MYAPP_HOME=/opt/myapp
export MYAPP_HOME
```

### Trap 10: `local` in functions at global scope

**Problem:** Using `local` outside a function causes an error in older bash.

**Example:**
```bash
$ local x=5
bash: local: can only be used in a function
```

**Why:** `local` is only valid inside function definitions. Outside, it's a syntax error.

**Fix:** Only use `local` in functions:
```bash
$ myfunc() {
    local x=5
    echo "$x"
}
```

## See It In The Wild

### Where you encounter variables and quoting daily

- **`~/.bashrc`** and **`~/.bash_profile`**: Full of `export` statements and variable assignments.
- **`/etc/environment`**: System-wide environment variables.
- **`Dockerfile`**: `ENV` and `ARG` instructions.
- **`docker-compose.yml`**: `environment:` section.
- **`.env` files** (used by many frameworks): `DATABASE_URL=postgres://...`.
- **Makefiles**: `VAR = value` (Make has its own variable syntax with `=` and `:=`).

### How to observe variable behavior

```bash
# See what's in your environment
env | head -20
declare -p | head -20   # All variables

# Trace expansion
set -x
x="hello"; echo "$x"
set +x

# See the environment of a running process
cat /proc/$$/environ | tr '\0' '\n' | head -10
```

### Try this now

```bash
# Test quoting with set -x
set -x
var="a  b  c"
echo $var
echo "$var"
set +x

# Explore parameter expansion
file="/usr/local/bin/script.sh"
echo "Base: ${file##*/}"
echo "Dir: ${file%/*}"
echo "No ext: ${file%.*}"

# See how IFS affects splitting
echo "Default IFS"
x="one two three"
for word in $x; do echo "  '$word'"; done

IFS=","
echo "Comma IFS"
x="one,two,three"
for word in $x; do echo "  '$word'"; done
```

## Check Your Understanding

<details>
<summary>1. What's the difference between `"$@"` and `$@`? When would each be correct?</summary>

`"$@"` preserves each argument as a separate word — even if they contain spaces, tabs, or newlines. `$@` (unquoted) word-splits each argument on IFS characters and glob-expands each word. `"$@"` is correct 99% of the time. `$@` is correct only when you explicitly want word splitting and globbing of argument values.
</details>

<details>
<summary>2. Why does `[ $var == "x" ]` fail when `$var` is empty or contains spaces?</summary>

When `$var` is empty, the test becomes `[ == "x" ]` — three arguments. The `[` command expects either 2, 3, or 4 arguments depending on operator. With spaces, `[ file with spaces == "x" ]` becomes 6 arguments. Always quote: `[ "$var" == "x" ]`.
</details>

<details>
<summary>3. What does `echo "${PWD##*/}"` output and why?</summary>

It outputs the basename of the current directory (e.g., `lesson-08` if in that directory). `${PWD##*/}` removes the longest prefix matching `*/` — everything up to and including the last slash. What remains is the final path component (the current directory name).
</details>

<details>
<summary>4. What's the difference between `export VAR=value` and `VAR=value; export VAR`?</summary>

Functionally identical in modern bash. The first form is shorter but historically not POSIX-compliant (some older shells separated the command incorrectly). The two-line form is more portable. In practice, `export VAR=value` works everywhere.
</details>

<details>
<summary>5. What does word splitting split on? How can you change it?</summary>

Word splitting splits on characters in `$IFS` (Internal Field Separator). Default: space, tab, newline. Change it with `IFS=":"` or `IFS=$'\n'` (newline only). Always save and restore `IFS` if you change it.
</details>

<details>
<summary>6. You have `f="*.txt"` and run `echo $f` (unquoted). What happens if there are .txt files? What if there aren't?</summary>

If there are .txt files: `echo $f` expands to `echo *.txt`, then pathname expansion expands `*.txt` to matching files. You see the file list. If there are no .txt files: `*.txt` has no matches, and with nullglob off (default), it stays as `*.txt` — which `echo` prints literally. With `echo "$f"`, it always prints `*.txt` literally.
</details>

<details>
<summary>7. What does `${VAR:-default}` vs `${VAR-default}` do? What's the difference?</summary>

`${VAR:-default}`: uses `default` if `$VAR` is unset OR null (empty string). `${VAR-default}`: uses `default` only if `$VAR` is UNSET (not empty). If `VAR=""`, the first gives "default", the second gives "".
</details>
