# Lesson 11: Conditionals

## History & Origins

The `test` command (also known as `[`) traces back to Unix V7 (1979) where it was an external binary at `/bin/test`. The `[` form was a hard link to `test` — the kernel didn't know about brackets. When you wrote `[ "$x" = "y" ]`, the shell called `/bin/[` with arguments `"$x"`, `=`, `"y"`, `]`. The `]` was just a required last argument that `[` checked for and ignored. This is why spaces around brackets are MANDATORY — `[` is literally a command name.

The `test` command was made a shell builtin in the SVR2 shell (1984) for performance, but the external binary still exists. Try `which [ /usr/bin/[` — yes, `/usr/bin/[` is a real file. You can even run it directly: `$ /usr/bin/[ "hello" = "hello" ] && echo "yes"`.

Bash's `[[ ]]` was introduced as a bash keyword (not a command) in bash 2.0 (1996). Unlike `[` which is a command, `[[ ]]` is shell syntax. This means the shell parses it differently — no word splitting, no pathname expansion inside, and `&&`/`||` work naturally. The `[[ ]]` keyword was inspired by the Korn shell `[[ ]]` from ksh88.

The short-circuit operators `&&` and `||` come from the shell's logical operators for chaining commands. They're not part of `test` — they're shell-level constructs. `cmd1 && cmd2` runs `cmd2` only if `cmd1` succeeds (exit 0). `cmd1 || cmd2` runs `cmd2` only if `cmd1` fails (exit non-zero).

## Syntax Reference

### if/then/elif/else/fi
```bash
if command; then
    ...
elif command; then
    ...
else
    ...
fi
```

### test / [ ] (POSIX)
```bash
test "$var" = "value"      # test command
[ "$var" = "value" ]       # bracket form (same thing)
[ ! "$var" = "value" ]     # negation
[ "$var1" = "$var2" -a "$var3" = "$var4" ]  # AND
[ "$var1" = "$var2" -o "$var3" = "$var4" ]  # OR
[ "$var1" = "$var2" ] && [ "$var3" = "$var4" ]  # AND (recommended)
[ "$var1" = "$var2" ] || [ "$var3" = "$var4" ]  # OR (recommended)
```

### [[ ]] (bash keyword)
```bash
[[ "$str" == "value" ]]      # Equal (pattern matching allowed)
[[ "$str" = "value" ]]       # Equal (same as ==)
[[ "$str" != "value" ]]      # Not equal
[[ "$str" =~ regex ]]        # Regex match
[[ ! "$str" =~ regex ]]      # Negated regex
[[ -z "$str" ]]              # String is empty (zero length)
[[ -n "$str" ]]              # String is non-empty
[[ "$str" == *.txt ]]        # Pattern match (globbing)
[[ "$num" -eq 5 ]]           # Numeric equal
[[ "$num" -ne 5 ]]           # Numeric not equal
[[ "$num" -lt 5 ]]           # Less than
[[ "$num" -le 5 ]]           # Less or equal
[[ "$num" -gt 5 ]]           # Greater than
[[ "$num" -ge 5 ]]           # Greater or equal
[[ -f "$file" && -r "$file" ]]   # AND inside [[ ]]
[[ -d "$dir" || -L "$dir" ]]     # OR inside [[ ]]
[[ -v VAR ]]                 # Variable is set (even if empty)
[[ -R VAR ]]                 # Variable is set and is a name reference
```

### Short-circuit operators
```bash
command1 && command2         # Run command2 ONLY if command1 succeeds
command1 || command2         # Run command2 ONLY if command1 fails
command1 && command2 || command3   # Short-circuit ternary (careful!)
```

### case statement
```bash
case "$var" in
    pattern1) commands ;;
    pattern2) commands ;;
    *) default ;;
esac
```

### Arithmetic conditionals
```bash
(( a == b ))     # Arithmetic equal (C-style)
(( a < b ))      # Less than
(( a >= b ))     # Greater or equal
(( a + b > c ))  # Arithmetic expression
(( a++ ))        # Increment
(( var = 42 ))   # Assignment in arithmetic context
```

## Under the Hood

### [ ] is a Command, [[ ]] is Syntax

This is the most important distinction in this lesson:

**`[ "$a" = "$b" ]`** — After variable expansion, bash treats `[` as a command. It performs word splitting and glob expansion on the arguments. So `[ $var = x ]` (unquoted) becomes `[ = x ]` if `$var` is empty — that's a syntax error because `[` sees 3 arguments (`=`, `x`, `]`) instead of 4.

**`[[ "$a" == "$b" ]]`** — Bash parses `[[` specially. No word splitting. No pathname expansion. The `&&` and `||` inside are shell operators, not arguments. This is why `[[ -n $var && $var -gt 5 ]]` works but `[ -n $var -a $var -gt 5 ]` is fragile.

### How the Shell Evaluates `if`

The `if` statement doesn't know about "true" and "false" in the programming sense. It only checks exit codes:
- Exit code 0 = "true" (run the then branch)
- Exit code non-zero = "false" (skip the then branch)

```bash
if ls /nonexistent 2>/dev/null; then
    echo "Directory exists"
fi
```
This runs `ls` and checks its exit code. `ls /nonexistent` returns 2, so the `then` branch is skipped.

### Execution Flow

```
if [ "$x" -eq 5 ]; then
    echo "x is 5"
elif [ "$x" -gt 5 ]; then
    echo "x is greater"
else
    echo "x is less"
fi
```

1. Bash forks (or uses builtin) to run `[ "$x" -eq 5 ]`
2. `[` returns 0 or 1
3. If 0: run `echo "x is 5"`, skip elif/else
4. If non-zero: go to `elif`, run `[ "$x" -gt 5 ]`
5. If 0: run `echo "x is greater"`, skip else
6. If non-zero: run `else` block

### strace View

```bash
$ strace -e trace=process bash -c '[ "hello" = "world" ] && echo "match"'
```
For `[` as external (if not builtin), you'd see `execve("/usr/bin/[", ...)`. As a builtin, no process creation occurs.

### Truth Table
```
Condition       [ ] exit code    [[ ]] result
true            0                true (runs then)
false           1                false (skips then)
command fails   non-zero         false
```

## Core Examples (12)

### Example 1: Basic if/else with Numeric Comparison
```bash
$ cat > ~/check_age.sh << 'EOF'
#!/bin/bash
read -p "Enter your age: " -r age
if [ "$age" -ge 18 ]; then
    echo "You are an adult."
else
    echo "You are a minor."
fi
EOF
$ chmod +x ~/check_age.sh && ./check_age.sh
Enter your age: 25
You are an adult.
```
`-ge` = "greater or equal." `[ "$age" -ge 18 ]` returns 0 if age >= 18.

**What if:** You use `>` instead of `-ge`? `[ "$age" > 18 ]` works but does STRING comparison, not numeric. `[ 9 > 18 ]` is true because "9" > "18" alphabetically. Always use `-gt`, `-lt`, `-ge`, `-le`, `-eq`, `-ne` for numbers.

### Example 2: elif Chain
```bash
$ cat > ~/grade.sh << 'EOF'
#!/bin/bash
read -p "Score: " -r score
if [ "$score" -ge 90 ]; then
    echo "A"
elif [ "$score" -ge 80 ]; then
    echo "B"
elif [ "$score" -ge 70 ]; then
    echo "C"
elif [ "$score" -ge 60 ]; then
    echo "D"
else
    echo "F"
fi
EOF
$ ./grade.sh
Score: 85
B
```
The chain evaluates top to bottom. First match wins. The order matters — if you put `-ge 80` before `-ge 90`, nobody gets A's.

### Example 3: Short-circuit && and ||
```bash
$ mkdir /tmp/testdir && echo "Created" || echo "Failed"
Created
$ mkdir /tmp/testdir && echo "Created" || echo "Failed"
mkdir: cannot create directory '/tmp/testdir': File exists
Failed
```
`&&` chains: first command succeeds -> run next. `||` chains: first command fails -> run next. This is the shell's "if-then-else" in operator form.

**What if:** You chain three: `cmd1 && cmd2 || cmd3`? If cmd1 succeeds AND cmd2 succeeds: runs cmd1, cmd2, then cmd3 (because `&&` returns success, then `||` sees success and runs nothing... wait, no). Actually `cmd1 && cmd2 || cmd3`: if cmd1 succeeds, run cmd2. If cmd2 fails, run cmd3. If cmd1 fails, run cmd3. This is NOT a ternary! It's two separate operators.

### Example 4: [[ ]] Pattern Matching
```bash
$ file="script.sh"
$ if [[ "$file" == *.sh ]]; then echo "Shell script"; fi
Shell script
$ if [[ "$file" == script.* ]]; then echo "Script file"; fi
Script file
$ if [[ "$file" == "*.sh" ]]; then echo "Literal match"; fi
# Nothing — quoted makes it literal
```
`[[ ]]` with `==` does pattern matching (globbing), not string equality. The `*.sh` pattern matches any string ending in `.sh`. Quotes make it literal.

**What if:** You try `[ "$file" = *.sh ]`? Without `[[ ]]`, `*.sh` is glob-expanded. If there's a file named `script.sh` in the current directory, it expands to that filename. Otherwise, it stays literal `*.sh` — which doesn't match `script.sh`.

### Example 5: [[ ]] Handles Empty Strings Safely
```bash
$ unset var
$ if [[ "$var" == "hello" ]]; then echo "match"; else echo "no match"; fi
no match
$ if [[ $var == "hello" ]]; then echo "match"; else echo "no match"; fi
no match
# Both work — [[ ]] doesn't need quotes!

$ if [ $var = "hello" ]; then echo "match"; fi
-bash: [: =: unary operator expected
# FAILS — unquoted $var expands to nothing, [ sees [ = "hello" ]
```
`[[ ]]` internally handles empty and multi-word variables. `[ ]` does NOT — it passes the expanded words as arguments. If `$var` is empty, `[ $var = "hello" ]` becomes `[ = "hello" ]` — three arguments, but `[` expects at least one more. Syntax error.

### Example 6: Negation with !
```bash
$ if [ ! -f "/nonexistent" ]; then echo "File does not exist"; fi
File does not exist
$ if [[ ! -d "/tmp" ]]; then echo "Not a directory"; else echo "Is a directory"; fi
Is a directory
```
`!` flips the exit code. `[ ! -f file ]` is true when the file does NOT exist.

### Example 7: Combining Multiple Conditions
```bash
$ file="/etc/passwd"
$ if [ -f "$file" ] && [ -r "$file" ]; then
    echo "File exists and is readable"
fi
File exists and is readable

# Inside [[ ]], you can use && directly:
$ if [[ -f "$file" && -r "$file" ]]; then
    echo "File exists and is readable"
fi

# With [ ] AND operator (-a) — works but confusing:
$ if [ -f "$file" -a -r "$file" ]; then
    echo "File exists and is readable"
fi
```
`&&` between separate `[ ]` is clearer than `-a` inside one `[ ]`. Using `[[ ]]` with `&&` is the most readable.

### Example 8: String Tests — Empty vs Non-Empty
```bash
$ empty=""
$ nonempty="hello"

$ [ -z "$empty" ] && echo "empty string" || echo "non-empty"
empty string
$ [ -n "$nonempty" ] && echo "non-empty string" || echo "empty"
non-empty string

$ [[ -z $empty ]] && echo "empty"   # works without quotes in [[ ]]
empty
```
`-z` = "zero length." `-n` = "non-zero length."

### Example 9: Case Statement for Multi-Way Branching
```bash
$ cat > ~/fruit.sh << 'EOF'
#!/bin/bash
read -p "Enter a fruit: " -r fruit
case "$fruit" in
    apple|pear)
        echo "Pomaceous fruit"
        ;;
    banana)
        echo "Tropical fruit"
        ;;
    orange|lemon|lime)
        echo "Citrus fruit"
        ;;
    *)
        echo "Unknown fruit"
        ;;
esac
EOF
$ ./fruit.sh
Enter a fruit: apple
Pomaceous fruit
```
`case` is cleaner than long `if/elif/elif/elif` chains. Patterns use glob syntax (`|` for OR, `*` for default).

### Example 10: Arithmetic Conditionals with (())
```bash
$ a=5
$ b=10
$ if (( a < b )); then echo "$a is less than $b"; fi
5 is less than 10
$ (( a + b > 20 )) && echo "Sum > 20" || echo "Sum <= 20"
Sum <= 20
$ (( result = a * b ))
$ echo "$result"
50
```
`(( ))` evaluates C-style arithmetic expressions. Returns 0 (true) if result is non-zero, 1 (false) if result is 0. Variables don't need `$` inside `(( ))`.

### Example 11: [[ ]] vs [ ] — The Word Splitting Difference
```bash
$ words="hello world"
$ [ $words = "hello world" ] 2>&1
-bash: [: too many arguments
$ [ "$words" = "hello world" ] && echo "match"
match
$ [[ $words == "hello world" ]] && echo "match"
match
$ [[ "$words" == "hello world" ]] && echo "match"
match
```
Without quotes in `[ ]`, `$words` expands to two words: `hello` and `world`. `[ hello world = "hello world" ]` has 4 arguments — too many for `[` which expects 3 or fewer. In `[[ ]]`, word splitting doesn't happen, so `[[ $words == "hello world" ]]` works fine.

### Example 12: Checking Command Success Directly
```bash
$ if grep -q "root" /etc/passwd; then
    echo "Found root user"
fi
Found root user

# The ANTI-PATTERN (don't do this):
$ grep -q "root" /etc/passwd
$ if [ $? -eq 0 ]; then
    echo "Found root user"
fi
```
The anti-pattern stores the exit code in `$?` and then checks it. The direct way just uses the command as the condition. It's shorter, clearer, and avoids `$?` being overwritten by another command before the `if`.

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Service health checks**: `if systemctl is-active --quiet apache2; then echo "Running"; fi`
- **Disk space monitoring**: `if [ "$(df / | awk 'NR==2 {print $5}' | tr -d '%')" -gt 90 ]; then alert; fi`
- **User existence**: `if id "$user" &>/dev/null; then echo "User exists"; fi`
- **Package installed**: `if dpkg -l "$pkg" &>/dev/null; then echo "Installed"; fi`
- **Mount point check**: `if mountpoint -q /mnt/backup; then ... ; fi`

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Build validation**: `if ./tests; then echo "Tests passed"; else echo "Tests failed" >&2; exit 1; fi`
- **Input validation**: `if [[ "$email" =~ .+@.+ ]]; then echo "Valid email"; fi`
- **Empty result handling**: `if [ -z "$(ls -A dir 2>/dev/null)" ]; then echo "Empty dir"; fi`
- **Version checks**: `if (( $(echo "$version >= 3.0" | bc -l) )); then ...; fi`

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Readable shadow file**: `if [ -r /etc/shadow ]; then echo "Vulnerable"; fi`
- **SUID binary check**: `if [ -u /usr/bin/sudo ]; then echo "SUID set"; fi`
- **Writable cron directories**: `if [ -w /etc/cron.d ]; then echo "Can plant cron job"; fi`
- **Bypassing `set -e`**: Using `if` around a command that might fail prevents `set -e` from triggering
- **Injection via unquoted test**: `[ $USER = "admin" ]` can be bypassed if attacker controls USER variable

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **Audit system files**: `if [ "$(stat -c '%a' /etc/shadow)" -ne 640 ]; then alert; fi`
- **Rootkit detection**: `if [ ! -x /usr/bin/ls ]; then echo "Possible rootkit"; fi`
- **Permission monitoring**: `if find / -perm -4000 -o -perm -2000 2>/dev/null | wc -l` changed — alert
- **Login monitoring**: `if [ "$(last -n 1 | wc -l)" -gt 0 ]; then ...; fi`

## Memory Aids

- **`[ ]` is like a question**: Spaces are essential — `[` is a command! Think of it as asking the system "is this true?"
- **`[[ ]]` is like a safe room**: Inside, no word splitting, no glob expansion — everything is safe.
- **`-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`**: "Equal," "Not Equal," "Less Than," "Less or Equal," "Greater Than," "Greater or Equal." The `-` prefix distinguishes them from redirection operators like `>`.
- **`-z` and `-n`**: "Zero length" and "Non-zero length." `-z` checks for EMPTY. `-n` checks for NON-empty.
- **`&&` = AND, `||` = OR**: The number of characters matches the number of conditions: `&&` has two characters (needs both true), `||` has two (needs at least one true). Actually that's a terrible mnemonic. Just remember: `&&` = "and" (both), `||` = "or" (either).
- **`if command; then`**: The semicolon replaces a newline. Read it as "if command, THEN do..."
- **`case` patterns**: Use `|` like "OR" in the pattern. `apple|pear` matches either.

## Trap Vault (12 traps)

### Trap 1: Unquoted Variable in [ ]
**Problem:** Variable with spaces or empty variable causes syntax error.
**Example:**
```bash
$ var=""
$ [ $var = "hello" ]
-bash: [: =: unary operator expected
```
**Why:** `$var` expands to nothing, so `[` sees `[ = "hello" ]` — missing an argument.
**Fix:**
```bash
$ [ "$var" = "hello" ]  # Quotes! Always quote in [ ]
```

### Trap 2: Using > Instead of -gt
**Problem:** Numeric comparison works but gives wrong results.
**Example:**
```bash
$ [ 9 > 18 ] && echo "true" || echo "false"
true
```
**Why:** `>` is string redirection inside `[ ]`. `9 > 18` creates a file named `18` and writes `9` to it. The comparison never happens.
**Fix:** `[ 9 -gt 18 ]` or use `(( 9 > 18 ))`.

### Trap 3: == vs = in [ ]
**Problem:** Script works in bash but fails in sh.
**Example:**
```bash
$ cat > test.sh << 'EOF'
#!/bin/sh
[ "$1" == "help" ] && echo "Help"
EOF
$ dash test.sh          # dash is /bin/sh on Debian
dash: 3: [: help: unexpected operator
```
**Why:** `==` is not POSIX. `[` in POSIX shell only supports `=`.
**Fix:** Use `[ "$1" = "help" ]` for POSIX compatibility. In `[[ ]]`, both work.

### Trap 4: [[ ]] is a Bashism
**Problem:** Script with `[[ ]]` fails under dash or POSIX sh.
**Example:**
```bash
$ cat > script.sh << 'EOF'
#!/bin/sh
[[ -f "/etc/passwd" ]] && echo "Exists"
EOF
$ sh script.sh
script.sh: 2: [[: not found
```
**Fix:** Use `#!/bin/bash` shebang when using `[[ ]]`, or use `[ ]` for POSIX sh.

### Trap 5: Spaces in [ ] Are Mandatory
**Problem:** Missing spaces around brackets.
**Example:**
```bash
$ ["$var" = "x"]   # WRONG
$ [ "$var" = "x" ] # RIGHT
$ [ "$var" = "x"]  # WRONG — space before ] needed
```
**Why:** `[` is a command name. `["$var"` is a different command (like `[-x`). The `]` is the last argument to `[`.

### Trap 6: -a and -o Are Unreliable
**Problem:** Complex conditions with `-a` and `-o` break with certain values.
**Example:**
```bash
$ if [ -n "$var" -a "$var" -gt 5 ]; then echo "ok"; fi
```
**Why:** `-a` and `-o` within `[ ]` are ambiguous and deprecated by POSIX. If `$var` contains `!` or `(`, the test parser gets confused.
**Fix:** `if [ -n "$var" ] && [ "$var" -gt 5 ]; then echo "ok"; fi`

### Trap 7: if grep Without -q
**Problem:** `grep` output appears on screen.
**Example:**
```bash
$ if grep "root" /etc/passwd; then
    echo "Found root in passwd"
  fi
root:x:0:0:root:/root:/bin/bash  # ← grep output leaks!
Found root in passwd
```
**Fix:** `if grep -q "root" /etc/passwd; then echo "Found"; fi` — `-q` makes grep quiet.

### Trap 8: The Dreaded [ $? -eq 0 ]
**Problem:** Using $? explicitly instead of the command.
**Example:**
```bash
$ grep -q "root" /etc/passwd
$ echo "After grep"
$ if [ $? -eq 0 ]; then echo "Found root"; fi
After grep
Found root
```
Wait, that worked! But only because `echo` succeeded. If `echo` failed, `$?` would be wrong. More importantly:
```bash
$ grep -q "root" /etc/passwd
$ if [ $? -eq 0 ]; then echo "Found"; fi  # This works, but is fragile
```
**Better:** `if grep -q "root" /etc/passwd; then echo "Found"; fi`

### Trap 9: Integer Comparison vs String Comparison
**Problem:** Numbers compared as strings give wrong results.
**Example:**
```bash
$ [ "9" -lt "10" ] && echo "numeric: 9 < 10"
numeric: 9 < 10
$ [[ "9" < "10" ]] && echo "string: 9 < 10" || echo "string: 9 > 10"
string: 9 > 10
```
**Why:** `<` inside `[[ ]]` does STRING comparison (lexicographic). "9" > "10" because '9' > '1'. `-lt` does numeric comparison.

### Trap 10: elif Without Condition
**Problem:** Using `else if` instead of `elif`.
**Example:**
```bash
$ cat > wrong.sh << 'EOF'
if [ "$x" -eq 1 ]; then
    echo "one"
else if [ "$x" -eq 2 ]; then
    echo "two"
fi
EOF
$ bash wrong.sh
wrong.sh: line 5: syntax error near unexpected token `fi'
```
**Fix:** `elif` is ONE word. `else if` starts a new `if` inside the `else` block, needing a second `fi`.

### Trap 11: Case Pattern With Variables
**Problem:** Variables inside case patterns aren't expanded.
**Example:**
```bash
$ ext="txt"
$ case "file.txt" in
    *.$ext) echo "match" ;;
  esac
match  # Actually this DOES work in bash
```
OK, this one works. But if you need regex in case — case doesn't support regex. Use `[[ $var =~ pattern ]]`.

### Trap 12: Confusing exit Codes in if
**Problem:** Command succeeds but if condition fails.
**Example:**
```bash
$ if ! grep -q "root" /etc/passwd; then
    echo "root not found"
  else
    echo "root found"
  fi
root found
```
The `!` flips the exit code. `grep -q "root"` returns 0 (found). `!` flips to 1, which means "false" for the `if`. But then the else branch runs saying "root found" — which is correct but confusing to read. The logic: `!` makes `if` treat success as failure.

## See It In The Wild

### System init scripts
Look at any system service file:
```bash
$ head -30 /etc/init.d/ssh
$ head -30 /usr/lib/systemd/system/*.service 2>/dev/null
$ head -30 /etc/rc.local 2>/dev/null
```

### Try this now:
```bash
# 1. Test your understanding of exit codes
$ true  && echo "true -> success"
$ false && echo "false -> success" || echo "false -> failure"
$ ! true && echo "!true -> failure" || echo "!true -> success"

# 2. Watch [ ] vs [[ ]] differences
$ var=""
$ [ $var = "" ] 2>&1; echo "Exit: $?"
$ [[ $var == "" ]]; echo "Exit: $?"

# 3. Test pattern matching
$ [[ "hello_world" == *world ]] && echo "glob match"
$ [[ "hello_world" =~ ^hello.*world$ ]] && echo "regex match"
```

## Check Your Understanding (7 questions)

1. Why must variables be quoted inside `[ ]` but not necessarily in `[[ ]]`?

2. What is the difference between `=` and `==` in bash? Where is each valid?

3. What does `-z` check? What does `-n` check?

4. How do you combine two conditions in `[ ]` vs `[[ ]]`? Which is preferred?

5. Why is `if command` better than `command; if [ $? -eq 0 ]`?

6. What's the difference between `[[ 9 < 10 ]]` and `[[ 9 -lt 10 ]]`?

7. Why does `[ $var = "x" ]` fail when `$var` is empty?
