# Task 11: Conditional File Analyzer

Write a script that analyzes a given path and reports its properties using conditionals. You'll use `[ ]`, `[[ ]]`, file tests, and pattern matching — all the conditional tools you just learned.

## Steps

### Sub-task 1: Argument Handling
Create `~/filecheck.sh` that:
- Takes one argument (a file path)
- If no argument, prompt the user with `read -p`
- If the input (argument or prompt result) is empty, print usage and exit 1
- Uses both `[ ]` and `[[ ]]` in the same script

```bash
#!/bin/bash

if [ $# -eq 0 ]; then
    read -p "Enter a path: " -r path
else
    path="$1"
fi

if [ -z "$path" ]; then
    echo "Usage: filecheck.sh <path>" >&2
    exit 1
fi
```

**Approach 1:** `$# -eq 0` to check argument count
**Approach 2:** `${1:-}` default assignment
**Approach 3:** `[[ -z ${1+set} ]]` to check if set vs empty

<details><summary>Hint: Checking if a variable is set vs empty</summary>
`[ -z "$var" ]` checks if var is empty OR unset. `[ -z "${var+set}" ]` checks if var is UNSET (returns true if unset, false if set to anything including empty). Use the right one!
</details>

### Sub-task 2: Existence and Type Checks
Check and report:
- Does the path exist? (`-e`)
- Is it a file, directory, or something else? (`-f`, `-d`, `-L`)
- If it doesn't exist, print error and exit 1

```bash
# Check existence with [[ ]]
if [[ ! -e "$path" ]]; then
    echo "Error: $path does not exist." >&2
    exit 1
fi

# Determine type
if [[ -d "$path" ]]; then
    type="directory"
elif [[ -f "$path" ]]; then
    type="regular file"
elif [[ -L "$path" ]]; then
    type="symbolic link"
else
    type="special file (socket, device, etc.)"
fi

echo "Path: $path"
echo "  Exists: yes"
echo "  Type: $type"
```

**Approach 1:** `if/elif/elif` chain
**Approach 2:** `case` statement with file tests
**Approach 3:** `stat --format=%F` for the type string

<details><summary>Hint: What -L returns for symlinks</summary>
`-L` returns true only for the symlink itself. After `-L`, you can check the target: `[[ -L "$path" ]] && target=$(readlink "$path")`.
</details>

### Sub-task 3: Permission Checks
Check and report:
- Is it readable? (`-r`)
- Is it writable? (`-w`)
- Is it executable? (`-x`)
- Is it empty or has content? (`-s`)

```bash
echo "  Readable: $([ -r "$path" ] && echo "yes" || echo "no")"
echo "  Writable: $([ -w "$path" ] && echo "yes" || echo "no")"
echo "  Executable: $([ -x "$path" ] && echo "yes" || echo "no")"
echo "  Empty: $([ -s "$path" ] && echo "no (has content)" || echo "yes (empty)")"
```

**Approach 1:** Inline `&&`/`||` as shown
**Approach 2:** Full `if/then/else` for each
**Approach 3:** Use `test` command directly in a function

**Bonus:** Show the actual permission bits from `stat`:
```bash
echo "  Permissions: $(stat -c '%A (%a)' "$path")"
```

<details><summary>Hint: What -x means for directories</summary>
For directories, `-x` means "searchable" — you can enter the directory and access files within it. A directory without `-x` is inaccessible even if `-r` is set.
</details>

### Sub-task 4: Symlink Detection and Resolution
If the path IS a symlink, show where it points:
```bash
if [[ -L "$path" ]]; then
    target=$(readlink "$path")
    echo "  Symlink: yes -> $target"
    # Also show the target's properties
    if [[ -e "$target" ]]; then
        echo "  Target exists: yes"
        echo "  Target type: $([ -d "$target" ] && echo "directory" || [ -f "$target" ] && echo "file" || echo "other")"
    else
        echo "  Target exists: no (broken symlink!)"
    fi
else
    echo "  Symlink: no"
fi
```

**Approach 1:** `readlink "$path"` for the target
**Approach 2:** `stat -c %N "$path"` which shows both name and target
**Approach 3:** `realpath "$path"` for the resolved absolute path

<details><summary>Hint: readlink vs realpath</summary>
`readlink "$path"` shows the link target (may be relative). `realpath "$path"` resolves the full absolute path, following all symlinks. `readlink -f "$path"` also resolves to absolute path.
</details>

### Sub-task 5: Pattern Matching with [[ ]]
Use `[[ ]]` to match filename patterns:
- If the filename matches `*.sh`, print "This is a shell script."
- If it matches `*.conf` or `*.cfg`, print "This is a config file."
- If it matches `*.txt`, print "This is a text file."
- If it matches `*.log`, print "This is a log file."

```bash
filename=$(basename "$path")
echo "  Filename: $filename"
echo -n "  Pattern match: "
if [[ "$filename" == *.sh ]]; then
    echo "Shell script"
elif [[ "$filename" == *.conf || "$filename" == *.cfg ]]; then
    echo "Config file"
elif [[ "$filename" == *.txt ]]; then
    echo "Text file"
elif [[ "$filename" == *.log ]]; then
    echo "Log file"
else
    echo "No known pattern"
fi
```

**Approach 1:** `elif` chain with `[[ ]]` pattern matching
**Approach 2:** `case` statement with glob patterns
**Approach 3:** Regex with `=~`

<details><summary>Hint: Pattern matching vs regex in [[ ]]</summary>
`[[ "$f" == *.sh ]]` uses glob-style pattern matching. `[[ "$f" =~ \.sh$ ]]` uses regex. Both work inside `[[ ]]`, but `==` with glob is more readable for simple cases.
</details>

### Sub-task 6: Demonstrate the Unquoting Trap
Add a comment or optional demo mode showing what happens when variables are unquoted:
```bash
# DEMO: Uncomment these lines to see the unquoting trap in action
# echo "=== DEMO: Unquoted variable trap ==="
# bad_var=""
# echo "Unquoted in [ ]:"
# [ $bad_var = "x" ] 2>&1 && echo "OK" || echo "BROKEN"
# echo "Quoted in [ ]:"
# [ "$bad_var" = "x" ] 2>&1 && echo "OK" || echo "BROKEN"
# echo "Unquoted in [[ ]]:"
# [[ $bad_var == "x" ]] 2>&1 && echo "OK" || echo "BROKEN"
```

### Sub-task 7: Test Your Script Thoroughly
Run your script on these test cases and verify output:

```bash
$ ~/filecheck.sh /etc/passwd
Path: /etc/passwd
  Exists: yes
  Type: regular file
  Readable: yes
  Writable: no
  Executable: no
  Symlink: no
  Empty: no (has content)
  Filename: passwd
  Pattern match: No known pattern

$ ~/filecheck.sh /dev/null
Path: /dev/null
  Exists: yes
  Type: special file (socket, device, etc.)
  Readable: yes
  Writable: yes
  Executable: no
  Symlink: no
  Empty: no (has content)
  Filename: null
  Pattern match: No known pattern

$ ~/filecheck.sh /nonexistent
Path: /nonexistent
Error: does not exist.
$ echo $?
1

$ ~/filecheck.sh /usr/bin/python3
Path: /usr/bin/python3
  Exists: yes
  Type: symbolic link
  Readable: yes
  Writable: no
  Executable: yes
  Symlink: yes -> python3.11
  Target exists: yes
  Target type: file
  Filename: python3
  Pattern match: No known pattern

$ ~/filecheck.sh /etc/hostname
Path: /etc/hostname
  Exists: yes
  Type: regular file
  Readable: yes
  Writable: yes (as root or in group)
  Executable: no
  Symlink: no
  Empty: no (has content)
  Filename: hostname
  Pattern match: No known pattern
```

## Expected Output

```bash
$ ~/filecheck.sh /etc/passwd
Path: /etc/passwd
  Exists: yes
  Type: regular file
  Readable: yes
  Writable: no
  Executable: no
  Symlink: no
  Empty: no (has content)
  Filename: passwd
  Pattern match: No known pattern

$ ~/filecheck.sh /nonexistent
Path: /nonexistent
Error: does not exist.
$ echo $?
1
```

## Self-Check Questions

1. Why must variables be quoted inside `[ ]` but not necessarily in `[[ ]]`?

2. What is the difference between `=` and `==` in bash?

3. What does `-s` check? What is the opposite (empty check)?

4. How do you combine two conditions in `[ ]` vs `[[ ]]`?

5. What is `[ $? -eq 0 ]` an anti-pattern for?

6. Why does `[ -L "$path" ]` matter when `-f` and `-d` follow symlinks?

7. What pattern would match both `.conf` and `.cfg` files in a `[[ ]]` comparison?
