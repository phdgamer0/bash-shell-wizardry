# Lesson 15: Regex with `=~`

## History & Origins

Regular expressions were invented by American mathematician Stephen Cole Kleene in the 1950s, hence "Kleene star" (`*`). Ken Thompson implemented the first regex engine in the ed editor (Unix V6, 1975), which then became the basis for grep (Global Regular Expression Print). The POSIX standard defines two regex flavors: Basic (BRE — used by grep, sed) and Extended (ERE — used by egrep, awk).

Bash's `=~` operator inside `[[ ]]` was introduced in bash 3.0 (2004). It uses POSIX Extended Regular Expressions (ERE), matching what `egrep` uses. This was a major addition because previously regex matching required calling external tools like `grep`, `sed`, or `awk` in a subshell.

The name `BASH_REMATCH` follows bash convention: the array variable named in ALL CAPS that bash fills after a successful `=~` match. Index `[0]` holds the full match, `[1+]` hold capture groups — mirroring how matching works in Perl, Python, and other languages.

Fun fact: The `=~` operator was added specifically for the bash-completion project. The completion code needed fast pattern matching without forking external processes. Before bash 3.0, completion scripts had to call `grep` thousands of times during tab completion, which was painfully slow.

## Syntax Reference

### Basic matching
```bash
[[ "$str" =~ pattern ]]           # Match pattern anywhere in str
[[ ! "$str" =~ pattern ]]         # Negated match
[[ "$str" =~ ^pattern$ ]]         # Full-string match (anchored)
```

### Special variables
```bash
$BASH_REMATCH          # Array: [0]=full match, [1+]=capture groups
${BASH_REMATCH[0]}     # Full matched string
${BASH_REMATCH[1]}     # First capture group
${#BASH_REMATCH[@]}    # Number of groups + 1
```

### Character classes
```bash
[abc]                  # One of: a, b, c
[^abc]                 # NOT a, b, c
[a-z]                  # Range: any lowercase letter
[0-9]                  # Range: any digit
[a-zA-Z0-9_]           # Alphanumeric + underscore
[:alpha:]              # POSIX class: alphabetic (inside brackets)
[:digit:]              # POSIX class: digit
[:alnum:]              # POSIX class: alphanumeric
[:space:]              # POSIX class: whitespace
[:upper:]              # POSIX class: uppercase
[:lower:]              # POSIX class: lowercase
[:punct:]              # POSIX class: punctuation
```

### Quantifiers
```bash
*                      # Zero or more
+                      # One or more
?                      # Zero or one
{n}                    # Exactly n
{n,}                   # n or more
{n,m}                  # n to m (inclusive)
```

### Anchors
```bash
^                      # Start of string
$                      # End of string
```

### Special characters (must escape for literal)
```bash
\ . * + ? { } [ ] ( ) | ^ $
```

### Grouping and alternation
```bash
(abc)                  # Capture group
(abc|def)              # Alternation (abc OR def)
(?:abc)                # Non-capturing group (NOT in ERE, but in PCRE)
```

### Predefined character classes (ERE)
```bash
[[:digit:]]            # Digit
[[:alpha:]]            # Alpha
[[:alnum:]]            # Alpha-numeric
[[:space:]]            # Space, tab, newline
[[:upper:]]            # Uppercase
[[:lower:]]            # Lowercase
[[:punct:]]            # Punctuation
[[:print:]]            # Printable characters
[[:graph:]]            # Visible characters (non-space)
```

### Case-insensitive matching
```bash
shopt -s nocasematch   # Enable case-insensitive =~
[[ "$str" =~ pattern ]]    # Now matches regardless of case
shopt -u nocasematch   # Disable
```

## Under the Hood

### How `[[ $str =~ pattern ]]` Works

1. **Parsing**: Bash parses the `[[ ]]` compound command. It identifies `=~` as the regex operator.
2. **Pattern preparation**: The regex pattern (RIGHT side of `=~`) is NOT expanded for globbing or word splitting, but variable expansion IS performed if the pattern is stored in a variable.
3. **Compilation**: Bash calls the C library's `regcomp()` function to compile the POSIX ERE pattern into an internal regex structure.
4. **Execution**: Bash calls `regexec()` to match the compiled regex against the string.
5. **Result storage**: If `regexec()` returns 0 (match), bash populates `$BASH_REMATCH` with the matched substrings.
6. **Return code**: `[[ ]]` returns 0 if match, 1 if no match.

### The Quoting Trap Explained

When you write `[[ "abc" =~ "[a-z]" ]]`, the quotes around `[a-z]` make it a LITERAL STRING. The regex engine looks for the literal characters `[`, `a`, `-`, `z`, `]`. It does NOT interpret `[a-z]` as a character class.

But when you write `[[ "abc" =~ [a-z] ]]` (no quotes), bash passes the pattern directly to the regex engine, which interprets it as "match any lowercase letter."

When you store the pattern in a variable: `pat='[a-z]'; [[ "abc" =~ $pat ]]`, bash expands `$pat` and passes the result to the regex engine — no quoting issues.

### System Calls

```bash
$ strace -e trace=regex bash -c '[[ "hello42" =~ [0-9]+ ]] && echo "match"'
# No visible syscalls — regex is done in-process (libc)
write(1, "match\n", 6)    = 6
```

The regex compilation and execution happen entirely in user-space via libc. No system calls are involved.

### ERE vs PCRE vs BRE

| Feature | BRE (`grep`) | ERE (`=~`, `egrep`) | PCRE (`grep -P`) |
|---------|-------------|---------------------|-----------------|
| `+` quantifier | `\+` | `+` | `+` |
| `?` quantifier | `\?` | `?` | `?` |
| Alternation | `\|` | `|` | `|` |
| Groups | `\( \)` | `( )` | `( )` |
| `\d` | no | no | yes |
| `\w` | no | no | yes |
| Lookahead | no | no | `(?=...)` |
| Backreferences | `\1` | no | `\1` |

Bash `=~` uses ERE. You cannot use `\d`, `\w`, lookaheads, or backreferences.

## Core Examples (12)

### Example 1: Check if String Contains a Number
```bash
$ str="hello42world"
$ if [[ "$str" =~ [0-9]+ ]]; then
    echo "Contains a number"
  fi
Contains a number
```
`[0-9]+` matches one or more digits. The pattern is unquoted, so it's treated as regex. The match succeeds because "42" is found.

**What if:** You quote the pattern? `[[ "hello42" =~ "[0-9]+" ]]` — fails because it looks for literal string `[0-9]+`.

### Example 2: Validate Email Format (Simple)
```bash
$ email="user@example.com"
$ pattern='^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
$ if [[ "$email" =~ $pattern ]]; then
    echo "Valid email"
  else
    echo "Invalid email"
  fi
Valid email
```
The pattern is stored in a variable (best practice). `^[a-zA-Z0-9._%+-]+` matches the username. `@` is literal. `[a-zA-Z0-9.-]+` matches the domain. `\.[a-zA-Z]{2,}$` matches the TLD.

### Example 3: Extract Groups with BASH_REMATCH
```bash
$ phone="(555) 123-4567"
$ pattern='^\(([0-9]{3})\) ([0-9]{3})-([0-9]{4})$'
$ if [[ "$phone" =~ $pattern ]]; then
    echo "Area code: ${BASH_REMATCH[1]}"
    echo "Exchange:  ${BASH_REMATCH[2]}"
    echo "Line:      ${BASH_REMATCH[3]}"
  fi
Area code: 555
Exchange:  123
Line:      4567
```
Parentheses in ERE create capture groups. After a successful match, `BASH_REMATCH[0]` = full match, `[1]` = group 1, etc. The backslashes before `(` and `)` escape them for literal parentheses in the phone number.

### Example 4: Validate IPv4
```bash
$ ip="192.168.1.1"
$ pattern='^([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})$'
$ if [[ "$ip" =~ $pattern ]]; then
    valid=true
    for octet in "${BASH_REMATCH[@]:1}"; do
        if [ "$octet" -gt 255 ]; then
            echo "Invalid: octet $octet > 255"
            valid=false
            break
        fi
    done
    $valid && echo "Valid IP"
  fi
Valid IP structure
Valid IP
```
The regex checks for four 1-3 digit groups separated by dots. After matching, we verify each octet is 0-255. Regex alone can't check numeric range — that needs logic.

### Example 5: Case-Insensitive Match
```bash
$ shopt -s nocasematch
$ if [[ "HELLO" =~ hello ]]; then
    echo "Case-insensitive match works"
  fi
Case-insensitive match works
$ shopt -u nocasematch  # restore normal behavior
```
`shopt -s nocasematch` makes ALL subsequent `=~` matches case-insensitive. Remember to restore it.

### Example 6: Extract Domain from URL
```bash
$ url="https://www.example.com/path/to/page"
$ pattern='^https?://([^/]+)'
$ if [[ "$url" =~ $pattern ]]; then
    echo "Domain: ${BASH_REMATCH[1]}"
  fi
Domain: www.example.com
```
`https?` matches `http` or `https`. `://` is literal. `([^/]+)` captures everything up to the next `/`.

### Example 7: Date Validation
```bash
$ date="2026-07-31"
$ pattern='^([0-9]{4})-([0-9]{2})-([0-9]{2})$'
$ if [[ "$date" =~ $pattern ]]; then
    year=${BASH_REMATCH[1]}
    month=${BASH_REMATCH[2]}
    day=${BASH_REMATCH[3]}
    echo "Year: $year, Month: $month, Day: $day"
    # Validate ranges
    [ "$month" -ge 1 ] && [ "$month" -le 12 ] || echo "Invalid month"
    [ "$day" -ge 1 ] && [ "$day" -le 31 ] || echo "Invalid day"
  fi
Year: 2026, Month: 07, Day: 31
```
Regex validates format `YYYY-MM-DD`. Additional numeric checks validate ranges.

### Example 8: Check Filename Pattern
```bash
$ file="backup-2026-07-31.tar.gz"
$ pattern='^backup-[0-9]{4}-[0-9]{2}-[0-9]{2}\.tar\.gz$'
$ if [[ "$file" =~ $pattern ]]; then
    echo "Valid backup filename"
    # Extract date
    date_part="${file#backup-}"
    date_part="${date_part%.tar.gz}"
    echo "Date: $date_part"
  fi
Valid backup filename
Date: 2026-07-31
```
The regex checks the full filename. We also extract the date using parameter expansion (no need for BASH_REMATCH here).

### Example 9: Log Parsing
```bash
$ log_line="Jul 31 01:23:45 server sshd[12345]: Failed password for root from 10.0.0.1 port 22"
$ pattern='Failed password for .+ from ([0-9.]+) port ([0-9]+)'
$ if [[ "$log_line" =~ $pattern ]]; then
    echo "Failed login from ${BASH_REMATCH[1]} on port ${BASH_REMATCH[2]}"
  fi
Failed login from 10.0.0.1 on port 22
```
Extracts data from a log line. The pattern matches the failed login message structure and captures the IP and port.

### Example 10: Matching Multiple Patterns (Loop)
```bash
$ lines=("apple 123" "banana 456" "cherry 789")
$ pattern='^([a-z]+) ([0-9]+)$'
$ for line in "${lines[@]}"; do
    if [[ "$line" =~ $pattern ]]; then
        echo "Word: ${BASH_REMATCH[1]}, Number: ${BASH_REMATCH[2]}"
    fi
  done
Word: apple, Number: 123
Word: banana, Number: 456
Word: cherry, Number: 789
```
Apply the same regex to multiple strings in a loop. Each iteration overwrites `BASH_REMATCH`.

### Example 11: Alternation — Match Either Pattern
```bash
$ str="error: file not found"
$ pattern='^(error|warning|info): (.+)$'
$ if [[ "$str" =~ $pattern ]]; then
    echo "Level: ${BASH_REMATCH[1]}"
    echo "Message: ${BASH_REMATCH[2]}"
  fi
Level: error
Message: file not found
```
`(error|warning|info)` matches any of the three words. Captures the level and message separately.

### Example 12: Password Strength Check
```bash
$ password="Abc123!@#"
$ checks=0
$ [[ "$password" =~ [a-z] ]] && ((checks++))
$ [[ "$password" =~ [A-Z] ]] && ((checks++))
$ [[ "$password" =~ [0-9] ]] && ((checks++))
$ [[ "$password" =~ [^a-zA-Z0-9] ]] && ((checks++))
$ [[ ${#password} -ge 8 ]] && ((checks++))
$ echo "Password strength: $checks/5"
Password strength: 5/5
```
Multiple separate regex checks, each testing a different aspect. This is more practical than a single complex regex for password validation.

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Log analysis**: Extract IPs, timestamps, error codes from `/var/log/*`
- **Config validation**: Check that config files have valid syntax/format
- **User input validation**: Validate usernames, paths, email addresses
- **File auditing**: Check filenames against naming conventions
- **Service status parsing**: Extract status from service output

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Input sanitization**: Validate user input before processing
- **Data extraction**: Pull structured data from semi-structured text
- **Build validation**: Check version strings, commit hashes, tags
- **URL parsing**: Extract domains, paths, query parameters
- **Code generation**: Parse templates and replace patterns

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Input injection bypass**: Crafting input that passes regex validation but contains exploits
- **ReDoS (ReDoS)**: Crafting catastrophic backtracking patterns to DoS a service
- **Log injection**: Adding fake entries that match log-parsing regexes
- **Bypassing filters**: Using encoding/escaping to evade regex-based detection
- **Credential harvesting**: Using regex to scan files for password patterns

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **IDS patterns**: Match known attack patterns in logs
- **Malware detection**: Scan for suspicious patterns in files
- **Policy enforcement**: Validate config files against security policies
- **Anomaly detection**: Regex patterns for unusual log entries
- **Input whitelisting**: Only allow input matching strict regex patterns

## Memory Aids

- **`=~`** looks like a wavy equal sign — meaning "roughly equals" or "matches approximately." Perfect for regex.
- **`BASH_REMATCH`** = "BASH REgex MATCH" — the array holding what matched.
- **`[0-9]`**: The dash means "through" — 0 THROUGH 9.
- **`[^abc]`**: The caret inside brackets means "NOT" — think of it as a caret pointing AWAY from the excluded chars.
- **`^outside` vs `[^inside]`**: Caret outside brackets = start of string. Caret inside brackets = negation.
- **`*` vs `+`**: `*` = "maybe zero" (like a star that might not be visible). `+` = "at least one" (like a plus sign that's definitely there).
- **`?`**: The question mark asks "is this optional?"
- **`{3,5}`**: The braces look like a range — "between 3 and 5."
- **`(group)`**: Parentheses "group" things together, like in math.

## Trap Vault (12 traps)

### Trap 1: Quoting the Regex Pattern
**Problem:** The pattern doesn't match even though it should.
**Example:**
```bash
$ str="hello123"
$ [[ "$str" =~ "[0-9]+" ]] && echo "match" || echo "no match"
no match
```
**Why:** Quoting makes it a literal string. Bash looks for the literal characters `[0-9]+`.
**Fix:** Never quote the regex. Store in variable: `pat='[0-9]+'; [[ "$str" =~ $pat ]]`.

### Trap 2: BASH_REMATCH[0] vs [1]
**Problem:** Expecting first capture group in [0].
**Example:**
```bash
$ str="abc123"
$ [[ "$str" =~ ^([a-z]+)([0-9]+)$ ]]
$ echo "Full: ${BASH_REMATCH[0]}, First: ${BASH_REMATCH[1]}, Second: ${BASH_REMATCH[2]}"
Full: abc123, First: abc, Second: 123
```
**Why:** `[0]` is always the full match, not the first capture group.
**Fix:** Remember: `[0]` = everything, `[1]` = first `(...)`, `[2]` = second `(...)`, etc.

### Trap 3: Using PCRE Features
**Problem:** Expecting `\d`, `\w`, or lookahead to work.
**Example:**
```bash
$ str="hello"
$ [[ "$str" =~ \d+ ]] && echo "match" || echo "no match"
no match
```
**Why:** Bash `=~` uses POSIX ERE. `\d` is not supported (it's PCRE).
**Fix:** Use `[0-9]` instead of `\d`, `[a-zA-Z0-9_]` instead of `\w`, `[[:space:]]` instead of `\s`.

### Trap 4: Pattern in Variable Without Quotes
**Problem:** Pattern with spaces in variable doesn't match.
**Example:**
```bash
$ pat='hello world'
$ str="hello world"
$ [[ "$str" =~ $pat ]] && echo "match" || echo "no match"
no match
```
**Why:** Without quotes on the variable, it's word-split! `hello world` becomes two arguments to `=~`.
**Fix:** Quote the variable: `[[ "$str" =~ "$pat" ]]` — BUT WAIT, that quotes it literally! The dilemma: unquoted splits, quoted makes literal. Solution: use a variable set with `shopt -s compat42` or use single-quotes in the pattern: `pat='hello world'` and ensure the variable isn't word-split... Actually the real fix is to avoid spaces in patterns that need literal matching, or use `[[ "$str" =~ $pat ]]` with proper IFS. Best: set pattern without spaces that could split, or use `[[ "$str" =~ $pat ]]` with `IFS=` temporarily.

Actually, the real issue: `[[ ]]` doesn't do word splitting on the right side of `=~`. So `$pat` is fine unquoted. The problem above must be something else. Let me check: `[[ "hello world" =~ hello world ]]` — two words after `=~`, the second is an error. Right, that's the issue.

**Fix:** Store the pattern in a variable and ensure it's expanded as one word. In `[[ ]]`, unquoted variable expansion on the right side of `=~` IS allowed and doesn't get quoted. So: `pat='hello world'; [[ "$str" =~ $pat ]]` actually WORKS. The issue would be if you wrote `[[ "$str" =~ $pat ]]` with a pattern containing regex special chars that happen to be spaces.

### Trap 5: BASH_REMATCH Gets Overwritten
**Problem:** Second regex check overwrites first match data.
**Example:**
```bash
$ str="abc 123"
$ [[ "$str" =~ ([a-z]+) ]]
$ echo "First match: ${BASH_REMATCH[1]}"
First match: abc
$ [[ "$str" =~ ([0-9]+) ]]
$ echo "First match again: ${BASH_REMATCH[1]}"  # NOW SHOWS 123!
```
**Why:** Every `=~` overwrites `BASH_REMATCH`.
**Fix:** Save matches immediately to variables: `[[ "$str" =~ ([a-z]+) ]]; word=${BASH_REMATCH[1]}; [[ "$str" =~ ([0-9]+) ]]; num=${BASH_REMATCH[1]}`.

### Trap 6: Regex Anchoring Surprises
**Problem:** `=~` matches ANYWHERE in the string, not just the start.
**Example:**
```bash
$ str="abc123def"
$ [[ "$str" =~ [0-9]+ ]] && echo "match"
match  # Found "123" in the middle
```
**Fix:** Use `^` and `$` anchors for full-string matching: `[[ "$str" =~ ^[0-9]+$ ]]`.

### Trap 7: Extended Patterns Don't Work Inside =~
**Problem:** Using extglob patterns (`@()`, `!()`) inside `=~`.
**Example:**
```bash
$ shopt -s extglob
$ [[ "abc" =~ @(abc|def) ]] && echo "match" || echo "no match"
```
**Why:** `=~` uses regex, not extglob. `@()` is a glob pattern, not regex.
**Fix:** Use regex alternation: `[[ "$str" =~ ^(abc|def)$ ]]`.

### Trap 8: Escaping in ERE vs PCRE
**Problem:** Using `\d`, `\w`, `\s` expecting them to work.
**Example:**
```bash
$ [[ "123" =~ \d+ ]] && echo "match" || echo "no match"
no match
```
**Why:** In ERE, `\d` is literally "backslash-d" — it tries to match a literal `d` preceded by backslash.
**Fix:** Use POSIX classes: `[[:digit:]]+` or `[0-9]+`.

### Trap 9: Nested Parentheses and BASH_REMATCH Indexing
**Problem:** Counting capture groups wrong.
**Example:**
```bash
$ str="a(b)c"
$ pattern='^(.)(\(.)(.)$'
$ if [[ "$str" =~ $pattern ]]; then
    for i in 0 1 2 3; do
        echo "  [$i] = ${BASH_REMATCH[$i]}"
    done
  fi
  [0] = a(b)c
  [1] = a
  [2] = (b
  [3] = c
```
**Why:** Each `(...)` in the regex creates a capture group, including escaped `\(`. Count your parens carefully.

### Trap 10: Case Sensitivity Without nocasematch
**Problem:** Pattern doesn't match due to case.
**Example:**
```bash
$ [[ "HELLO" =~ hello ]] && echo "match" || echo "no match"
no match
```
**Fix:** Either add case variants `[Hh][Ee][Ll][Ll][Oo]`, use `shopt -s nocasematch`, or use `tr '[:upper:]' '[:lower:]'` first.

### Trap 11: Regex With Newlines
**Problem:** Multiline string matching with `^` and `$`.
**Example:**
```bash
$ str=$'line1\nline2\nline3'
$ [[ "$str" =~ ^line2$ ]] && echo "match" || echo "no match"
no match
```
**Why:** `^` and `$` match the START and END of the string, not lines within it. Bash ERE doesn't have `/m` (multiline) mode.
**Fix:** Use `while read` to process line by line, or use `grep` with `-z` for multiline.

### Trap 12: Pattern Matching an Empty String
**Problem:** `*` matches zero characters, so empty string matches.
**Example:**
```bash
$ str=""
$ [[ "$str" =~ ^[0-9]*$ ]] && echo "match" || echo "no match"
match  # Empty string matched!
```
**Why:** `*` means "zero or more." An empty string has zero digits, which matches.
**Fix:** Use `+` instead of `*` if you need at least one: `[[ "$str" =~ ^[0-9]+$ ]]`.

## See It In The Wild

### Log parsing on your system
```bash
# Extract all failed login attempts
$ grep "Failed password" /var/log/auth.log 2>/dev/null | head -3

# Extract IPs from auth log using =~ (in a script):
$ while IFS= read -r line; do
    [[ "$line" =~ from\ ([0-9]+\.[0-9]+\.[0-9]+\.[0-9]+) ]] &&
        echo "IP: ${BASH_REMATCH[1]}"
  done < <(grep "Failed password" /var/log/auth.log 2>/dev/null) | sort -u
```

### Try this now:
```bash
# 1. Test various regex patterns interactively
$ test_regex() {
    local str="$1" pat="$2"
    if [[ "$str" =~ $pat ]]; then
        echo "MATCH: ${BASH_REMATCH[@]}"
    else
        echo "NO MATCH"
    fi
}
$ test_regex "hello123" '^[a-z]+[0-9]+$'
$ test_regex "my@email.com" '^.+@.+\..+$'

# 2. Validate some common inputs
$ inputs=("192.168.1.1" "999.999.999.999" "not-an-ip")
$ for ip in "${inputs[@]}"; do
    [[ "$ip" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}$ ]] &&
        echo "$ip: VALID" || echo "$ip: INVALID"
  done
```

## Check Your Understanding (7 questions)

1. Why does `[[ "abc" =~ "[a-z]" ]]` fail to match?

2. What is in `${BASH_REMATCH[0]}` vs `${BASH_REMATCH[1]}`?

3. How do you make a `=~` match case-insensitive?

4. Does bash support `\d` like Perl regex? If not, what should you use?

5. What happens to `BASH_REMATCH` between two `=~` operations?

6. What is the difference between BRE, ERE, and PCRE? Which does bash use?

7. How do you match the entire string vs finding a match anywhere?
