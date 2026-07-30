# Task 8: Quoting Gauntlet

Variables and quoting are the #1 source of bash bugs. This task will make you intimately familiar with when and why to quote, how word splitting works, and how to handle arrays, IFS, and command substitution safely.

## Setup

```bash
$ mkdir -p /tmp/quoting_lab
$ cd /tmp/quoting_lab
```

## Sub-task 1: Create Files with Problematic Names

Create files with spaces, brackets, and special characters:

```bash
$ touch "/tmp/quoting_lab/important file.txt"
$ touch "/tmp/quoting_lab/another - file [v2].txt"
$ touch "/tmp/quoting_lab/normal.txt"
$ touch "/tmp/quoting_lab/.hidden"
$ touch "/tmp/quoting_lab/file with (parens).txt"
$ touch "/tmp/quoting_lab/file;with;semicolons.txt"
```

**Question:** What makes these filenames "problematic" for unquoted variable expansion?

<details>
<summary>Problematic characters</summary>
Spaces cause word splitting. `[` triggers glob expansion. `;` is a command separator. `(` is a subshell operator. All of these break unquoted variable expansion because the shell interprets them syntactically.
</details>

## Sub-task 2: The Quoting Showdown Script

Write a script that demonstrates the disaster of unquoted variables:

```bash
$ cat > /tmp/quoting_demo.sh << 'SCRIPT'
#!/bin/bash

DIR="/tmp/quoting_lab"

echo "=== QUOTED loop ==="
for f in "$DIR"/*; do
    echo "Processing: $f"
    wc -l "$f" 2>/dev/null || echo "(empty or error)"
done

echo ""
echo "=== UNQUOTED loop (BROKEN) ==="
for f in $DIR/*; do
    echo "Processing: $f"
    wc -l "$f" 2>/dev/null || echo "(empty or error)"
done
SCRIPT
$ chmod +x /tmp/quoting_demo.sh
```

Run it:

```bash
$ bash /tmp/quoting_demo.sh
```

**Expected output:**
```
=== QUOTED loop ===
Processing: /tmp/quoting_lab/another - file [v2].txt
(empty or error)
Processing: /tmp/quoting_lab/file with (parens).txt
(empty or error)
Processing: /tmp/quoting_lab/file;with;semicolons.txt
(empty or error)
Processing: /tmp/quoting_lab/important file.txt
(empty or error)
Processing: /tmp/quoting_lab/normal.txt
(empty or error)

=== UNQUOTED loop (BROKEN) ===
Processing: /tmp/quoting_lab/another
wc: /tmp/quoting_lab/another: No such file or directory
Processing: -
wc: -: No such file or directory
Processing: file
wc: file: No such file or directory
...
```

**Question:** How many iterations does the QUOTED loop have vs the UNQUOTED loop? Why the difference?

<details>
<summary>Loop counts</summary>
Quoted: 5 iterations (5 files). Unquoted: many more iterations because the filenames with spaces are split into multiple arguments. "important file.txt" becomes "important" and "file.txt", etc.
</details>

## Sub-task 3: Word Splitting Walk

Create a variable and iterate over it quoted vs unquoted:

```bash
$ ITEM="apple  banana   cherry"
$ echo "Unquoted items:"
$ for f in $ITEM; do echo "  - '$f'"; done

$ echo "Quoted items:"
$ for f in "$ITEM"; do echo "  - '$f'"; done
```

**Expected output:**
```
Unquoted items:
  - 'apple'
  - 'banana'
  - 'cherry'

Quoted items:
  - 'apple  banana   cherry'
```

**Step by step:**
1. Unquoted: `$ITEM` expands to `apple  banana   cherry`. Word splitting on spaces (default IFS) produces 3 words.
2. Quoted: `"$ITEM"` expands to `apple  banana   cherry` as ONE word (including double and triple spaces).

**Question:** How many spaces are between "apple" and "banana"? Between "banana" and "cherry"? Does the quoting preserve them?

<details>
<summary>Space preservation</summary>
2 spaces between apple and banana, 3 between banana and cherry. Quoting preserves them exactly. Unquoted, each space becomes a word boundary regardless of count.
</details>

## Sub-task 4: Array Quoting

Create and manipulate arrays with quoting:

```bash
$ files=("file one.txt" "file two.txt" "file three.txt")

$ echo "=== Unquoted array expansion ==="
$ count=0
$ for f in ${files[@]}; do count=$((count + 1)); echo "  $count: '$f'"; done

$ echo "=== Quoted array expansion ==="
$ count=0
$ for f in "${files[@]}"; do count=$((count + 1)); echo "  $count: '$f'"; done
```

**Expected output:**
```
=== Unquoted array expansion ===
  1: 'file'
  2: 'one.txt'
  3: 'file'
  4: 'two.txt'
  5: 'file'
  6: 'three.txt'

=== Quoted array expansion ===
  1: 'file one.txt'
  2: 'file two.txt'
  3: 'file three.txt'
```

**Question:** What's the difference between `${files[@]}` and `${files[*]}` (both quoted and unquoted)?

<details>
<summary>@ vs * for arrays</summary>
`"${files[@]}"` — each element as separate word (3 words). `"${files[*]}"` — all elements as ONE word (joined by first IFS char, space by default). Unquoted: `${files[@]}` and `${files[*]}` both split on IFS.
</details>

## Sub-task 5: Export and Subshell Challenge

Demonstrate variable scope:

```bash
$ export PARENT_VAR="I am the parent"
$ CHILD_VAR="Only visible here"

$ echo "In parent: PARENT=$PARENT_VAR, CHILD=$CHILD_VAR"

$ bash -c 'echo "In child: PARENT=$PARENT_VAR, CHILD=$CHILD_VAR"'

$ (CHILD_VAR="Modified in subshell"; echo "In subshell: $CHILD_VAR")
$ echo "After subshell: $CHILD_VAR"
```

**Expected output:**
```
In parent: PARENT=I am the parent, CHILD=Only visible here
In child: PARENT=I am the parent, CHILD=                      # CHILD not exported!
In subshell: Modified in subshell
After subshell: Only visible here                              # Not modified!
```

**Question:** Why can't the child bash process see `CHILD_VAR`? How would you make it visible?

<details>
<summary>Variable visibility</summary>
Only `export`ed variables are passed to child processes via the environment block. `CHILD_VAR` was set but not exported. To fix: add `export CHILD_VAR` or set it with `export CHILD_VAR="Only visible here"`.
</details>

## Sub-task 6: String Manipulation Challenge

Given a set of filenames, extract parts using parameter expansion:

```bash
$ fullpath="/home/phd/projects/myapp/src/utils/helpers.py"

# Extract just the filename
$ echo "${fullpath##*/}"
helpers.py

# Extract the directory
$ echo "${fullpath%/*}"
/home/phd/projects/myapp/src/utils

# Extract the extension
$ echo "${fullpath##*.}"
py

# Strip the extension
$ echo "${fullpath%.*}"
/home/phd/projects/myapp/src/utils/helpers

# Extract just the filename without extension
$ basename="${fullpath##*/}"
$ echo "${basename%.*}"
helpers
```

**Question:** Why does `${fullpath##*.}` give the extension but `${fullpath%.*}` gives everything before the last dot? What's the difference between `#` and `##`? Between `%` and `%%`?

<details>
<summary># vs ## and % vs %%</summary>
`#` removes the SHORTEST matching prefix. `##` removes the LONGEST. `%` removes the SHORTEST matching suffix. `%%` removes the LONGEST. So `${fullpath#*.}` removes `"/home/"` (shortest match of `*.`), while `${fullpath##*.}` removes everything up to the last dot.
</details>

## Sub-task 7: IFS Manipulation

Change IFS and observe the effects:

```bash
$ echo "=== Default IFS ==="
$ data="one:two:three"
$ for item in $data; do echo "  $item"; done   # No splitting on colon by default

$ echo "=== Colon IFS ==="
$ OLDIFS="$IFS"
$ IFS=":"
$ for item in $data; do echo "  $item"; done   # Splits on colon!
$ IFS="$OLDIFS"

$ echo "=== Newline IFS ==="
$ IFS=$'\n'
$ multiline="line1
line2
line3"
$ for item in $multiline; do echo "  $item"; done  # Splits on newlines only
$ IFS="$OLDIFS"
```

**Question:** How would you read a colon-separated file (like `/etc/passwd`) line by line and split on colons?

<details>
<summary>Reading passwd with IFS</summary>
```bash
while IFS=: read -r user pass uid gid name home shell; do
    echo "$user has UID $uid"
done < /etc/passwd
```
Setting `IFS=:` before `read` splits each line on colons into the named variables. This is done inline so it doesn't affect the surrounding code.
</details>

## Sub-task 8: Default Values

Create a script that uses parameter defaults:

```bash
$ cat > /tmp/default_demo.sh << 'SCRIPT'
#!/bin/bash
# Simulate a script with configurable variables
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-myapp_db}"
DB_USER="${DB_USER:-${USER:-admin}}"

echo "Connecting to $DB_HOST:$DB_PORT/$DB_NAME as $DB_USER"
SCRIPT
$ chmod +x /tmp/default_demo.sh

$ bash /tmp/default_demo.sh
Connecting to localhost:5432/myapp_db as phd

$ DB_HOST="db.example.com" DB_NAME="production" bash /tmp/default_demo.sh
Connecting to db.example.com:5432/production as phd
```

**Question:** What's the difference between `${VAR:-default}` and `${VAR:=default}`? When would you use each?

<details>
<summary>:- vs :=</summary>
`${VAR:-default}`: expands to `default` if VAR is unset/null, but does NOT assign `default` to VAR. `${VAR:=default}`: expands to `default` AND assigns `default` to VAR. Use `:-` for one-time defaults, `:=` when you want to set the variable for later use.
</details>

## Sub-task 9: Here-Doc with and without Expansion

Create two config files and observe the difference:

```bash
$ APP_HOME="/opt/myapp"
$ cat << EOF > /tmp/config_expanded.conf
APP_HOME=$APP_HOME
LOG_DIR=$APP_HOME/logs
EOF

$ cat << 'EOF' > /tmp/config_literal.conf
APP_HOME=$APP_HOME
LOG_DIR=$APP_HOME/logs
EOF

$ cat /tmp/config_expanded.conf
APP_HOME=/opt/myapp
LOG_DIR=/opt/myapp/logs

$ cat /tmp/config_literal.conf
APP_HOME=$APP_HOME
LOG_DIR=$APP_HOME/logs
```

**Question:** Which version would you use as a TEMPLATE for deployment? Which for a specific server?

## Bonus Challenge: The Unquote Zone

Create a function that safely handles any filename:

```bash
$ cat > /tmp/safe_iter.sh << 'SCRIPT'
#!/bin/bash
# A safe file iterator that handles ANY filename
# Including files with spaces, newlines, quotes, and binary characters

safe_iterate() {
    local dir="$1"
    local pattern="${2:-*}"
    
    echo "Directory: $dir"
    echo "Pattern: $pattern"
    echo "---"
    
    # Use null-delimited processing
    while IFS= read -r -d '' file; do
        # Get file info safely
        size=$(stat -c%s "$file" 2>/dev/null || echo "?")
        echo "  [$size bytes] $file"
    done < <(find "$dir" -maxdepth 1 -name "$pattern" -print0 2>/dev/null)
}

# Test with normal directory
safe_iterate "/tmp/quoting_lab" "*.txt"

# Compare with broken version:
echo ""
echo "=== Broken version ==="
for f in /tmp/quoting_lab/*.txt; do
    stat -c"%s" $f     # BROKEN — unquoted!
done
SCRIPT
$ chmod +x /tmp/safe_iter.sh
```

**Question:** Why does `-print0` with `read -d ''` handle any filename? What would break with newlines in filenames using the standard `for f in *` approach?

<details>
<summary>Null-delimited safety</summary>
`-print0` delimits filenames with the NUL character (`\0`), which is the only character that can NEVER appear in a filename. `read -d ''` reads until a NUL. This handles spaces, tabs, newlines, quotes, and any other special character in filenames. The standard `for f in *` approach breaks on newlines in filenames because `for` splits on IFS (which includes newline by default).
</details>

## Self-Check

1. What's the difference between `"$@"` and `$@`? When would you use each?
2. What does word splitting split on? What is `IFS` and how do you change it?
3. Why does `echo $var` sometimes print multiple lines instead of one?
4. What's the difference between `$(cmd)` and `"$(cmd)"`?
5. When would you WANT to omit quotes around a variable expansion? (There are legitimate cases.)
6. What's the difference between `${var:-default}`, `${var:=default}`, and `${var:?error}`?
7. Why does `export PATH=$PATH:/new/dir` work but `export PATH= $PATH:/new/dir` fail?
