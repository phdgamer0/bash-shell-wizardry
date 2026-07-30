# Task 12: Filesystem Diagnostic Script

Write a script that runs every file test operator on a given path and reports the results. Then extend it to recursively scan directories for issues.

## Steps

### Sub-task 1: Single-File Diagnostics
Create `~/diag.sh` that:
- Takes a path as argument (or prompts if none given)
- Runs ALL file tests: `-e`, `-f`, `-d`, `-r`, `-w`, `-x`, `-s`, `-L`, `-O`, `-G`, `-u`, `-g`, `-k`
- Outputs a formatted table of results

```bash
#!/bin/bash

path="${1:-}"
if [ -z "$path" ]; then
    read -p "Enter path: " -r path
fi

if [ ! -e "$path" ]; then
    echo "Error: $path does not exist." >&2
    exit 1
fi

echo "=== Diagnostics for: $path ==="
echo "  -e (exists):       $([ -e "$path" ] && echo YES || echo NO)"
echo "  -f (regular file): $([ -f "$path" ] && echo YES || echo NO)"
echo "  -d (directory):    $([ -d "$path" ] && echo YES || echo NO)"
echo "  -r (readable):     $([ -r "$path" ] && echo YES || echo NO)"
echo "  -w (writable):     $([ -w "$path" ] && echo YES || echo NO)"
echo "  -x (executable):   $([ -x "$path" ] && echo YES || echo NO)"
echo "  -s (size > 0):     $([ -s "$path" ] && echo YES || echo NO)"
echo "  -L (symlink):      $([ -L "$path" ] && echo YES || echo NO)"
echo "  -O (owned by you): $([ -O "$path" ] && echo YES || echo NO)"
echo "  -G (group owned):  $([ -G "$path" ] && echo YES || echo NO)"
echo "  -u (SUID):         $([ -u "$path" ] && echo YES || echo NO)"
echo "  -g (SGID):         $([ -g "$path" ] && echo YES || echo NO)"
echo "  -k (sticky):       $([ -k "$path" ] && echo YES || echo NO)"
```

**Approach 1:** Inline ternary with `&&`/`||` as shown
**Approach 2:** Function to avoid repetition: `check() { echo "  $1: $([ "$2" ] && echo YES || echo NO)"; }`
**Approach 3:** Array of tests and loop: `for test in e f d r w x s L O G u g k; do ... done`

<details><summary>Hint: -O checks effective UID</summary>
`-O "$path"` returns true if the file's owner matches your EUID (effective user ID). This may differ from your login UID if you're using `sudo` or `su`.
</details>

### Sub-task 2: Comparison Mode
If two paths are given, compare them with `-nt` and `-ot`:

```bash
if [ $# -ge 2 ]; then
    echo
    echo "=== Comparison ==="
    if [ "$1" -nt "$2" ]; then
        echo "  $1 is newer than $2"
    elif [ "$1" -ot "$2" ]; then
        echo "  $1 is older than $2"
    else
        echo "  $1 and $2 are the same age"
    fi

    if [ "$1" -ef "$2" ]; then
        echo "  $1 and $2 are the same file (hard link)"
    fi
fi
```

**Approach 1:** Use positional arguments
**Approach 2:** Use `-ef` for hard link detection
**Approach 3:** Show actual timestamps with `stat -c '%Y'` for verification

<details><summary>Hint: -nt with non-existent files</summary>
If either file in an `-nt` comparison doesn't exist, the result is false. Always check existence first: `[ -e "$1" ] && [ -e "$2" ] && [ "$1" -nt "$2" ]`.
</details>

### Sub-task 3: Detailed Permission Info
Add verbose permission information:

```bash
perms=$(stat -c '%a' "$path" 2>/dev/null)
owner=$(stat -c '%U' "$path" 2>/dev/null)
group=$(stat -c '%G' "$path" 2>/dev/null)
size=$(stat -c '%s' "$path" 2>/dev/null)
links=$(stat -c '%h' "$path" 2>/dev/null)

echo "  Permissions (octal): $perms"
echo "  Owner: $owner"
echo "  Group: $group"
echo "  Size: $(numfmt --to=iec "$size" 2>/dev/null || echo "$size bytes")"
echo "  Hard links: $links"
```

**Approach 1:** `stat -c` format strings
**Approach 2:** `ls -la` and parse (fragile)
**Approach 3:** `find "$path" -printf` for advanced formatting

<details><summary>Hint: stat format specifiers</summary>
`%a` = octal permissions, `%A` = symbolic permissions, `%U` = owner name, `%G` = group name, `%s` = size, `%h` = hard link count, `%Y` = modification epoch, `%N` = quoted name with symlink target.
</details>

### Sub-task 4: Directory Tree Scan
For a directory argument, recursively scan and report:
- Total files, directories, symlinks
- Count of files NOT readable
- Count of empty files
- Warnings for inaccessible items

```bash
if [ -d "$path" ]; then
    echo
    echo "=== Directory Tree Scan: $path ==="

    total_files=$(find "$path" -type f 2>/dev/null | wc -l)
    total_dirs=$(find "$path" -type d 2>/dev/null | wc -l)
    total_symlinks=$(find "$path" -type l 2>/dev/null | wc -l)
    unreadable=$(find "$path" -type f ! -readable 2>/dev/null | wc -l)
    unwritable=$(find "$path" -type f ! -writable 2>/dev/null | wc -l)
    empty_files=$(find "$path" -type f -empty 2>/dev/null | wc -l)

    echo "  Files:     $total_files"
    echo "  Dirs:      $total_dirs"
    echo "  Symlinks:  $total_symlinks"
    echo "  Unreadable: $unreadable"
    echo "  Unwritable: $unwritable"
    echo "  Empty:     $empty_files"
    echo

    # Show warnings
    find "$path" -type f ! -readable 2>/dev/null | while IFS= read -r f; do
        echo "  WARNING: $f is not readable"
    done

    find "$path" -type d ! -readable -o -type d ! -executable 2>/dev/null | while IFS= read -r d; do
        echo "  WARNING: $d is inaccessible (missing read/execute)"
    done
fi
```

**Approach 1:** `find` with `! -readable` (GNU find extension)
**Approach 2:** Manual loop with `while IFS= read -r file; do [ -r "$file" ] || ... done`
**Approach 3:** `find -perm` to check mode bits manually

<details><summary>Hint: Safe file iteration</summary>
Use `while IFS= read -r -d '' file; do ... done < <(find "$path" -print0)` for filenames with special characters (spaces, newlines). This uses null-delimited output which is the only safe way.
</details>

### Sub-task 5: SUID/SGID/Sticky Bit Report
Add a security audit section for directories:

```bash
if [ -d "$path" ]; then
    echo "=== Security Audit ==="
    suid_count=$(find "$path" -type f -perm -4000 2>/dev/null | wc -l)
    sgid_count=$(find "$path" -type f -perm -2000 2>/dev/null | wc -l)
    world_writable=$(find "$path" -type f -perm -o=w 2>/dev/null | wc -l)

    echo "  SUID files:     $suid_count"
    echo "  SGID files:     $sgid_count"
    echo "  World-writable: $world_writable"

    [ "$suid_count" -gt 0 ] && echo "  WARNING: SUID binaries found! Potential security risk."
fi
```

### Sub-task 6: Test on Multiple Paths

```bash
# Test 1: Single file
$ ~/diag.sh /etc/passwd
=== Diagnostics for: /etc/passwd ===
  -e (exists):       YES
  -f (regular file): YES
  -d (directory):    NO
  -r (readable):     YES
  -w (writable):     NO
  -x (executable):   NO
  -s (size > 0):     YES
  -L (symlink):      NO
  -O (owned by you): NO
  -G (group owned):  NO
  -u (SUID):         NO
  -g (SGID):         NO
  -k (sticky):       NO

# Test 2: Comparison mode
$ ~/diag.sh /tmp/a /tmp/b
=== Diagnostics for: /tmp/a ===
  ...
=== Comparison ===
  /tmp/b is newer than /tmp/a

# Test 3: Directory scan
$ ~/diag.sh /etc
=== Diagnostics for: /etc ===
  -e (exists):       YES
  -f (regular file): NO
  -d (directory):    YES
  ...
=== Directory Tree Scan: /etc ===
  Files:     2134
  Dirs:      423
  Symlinks:  32
  Unreadable: 3
  Unwritable: 1
  Empty:     17

WARNING: /etc/shadow is not readable
WARNING: /etc/gshadow is not readable
```

## Expected Output

```bash
$ ~/diag.sh /etc/passwd
=== Diagnostics for: /etc/passwd ===
  -e (exists):       YES
  -f (regular file): YES
  -d (directory):    NO
  -r (readable):     YES
  -w (writable):     NO
  -x (executable):   NO
  -s (size > 0):     YES
  -L (symlink):      NO
  -O (owned by you): NO
  -G (group owned):  NO

$ ~/diag.sh /etc
=== Diagnostics for: /etc ===
  ... same table ...

=== Directory Tree Scan: /etc ===
  Files:     2134
  Dirs:      423
  Symlinks:  32
  Unreadable: 3
  Empty:     17

WARNING: /etc/shadow is not readable
WARNING: /etc/gshadow is not readable
WARNING: /etc/ssl/private is not readable (directory)
```

## Self-Check Questions

1. What does `-x` mean for a directory vs a file?

2. If a symlink points to a directory, does `-f` return true or false?

3. What is the difference between `-e` and `-f`?

4. What does `-s` test? What is its negation?

5. Why might `-r` return true even for a file with `chmod 000`?

6. How does `-ef` work and what does it detect?

7. What's the difference between `stat()` and `lstat()` at the kernel level?
