# Task 3: Parameter Expansion — Slicing

## Overview

Write a script `parse_path.sh` that extracts components from file paths and URLs using **only parameter expansion** — no `cut`, `sed`, `awk`, or `basename`/`dirname`.

## Sub-tasks

### 1. Path Parser

Given a full path like `/home/user/docs/report.pdf`, extract:
- **Directory path**: everything before the last `/`
- **Filename**: everything after the last `/`
- **Extension**: everything after the last `.`
- **Basename**: filename without extension

```bash
fullpath="$1"
dir="${fullpath%/*}"
file="${fullpath##*/}"
ext="${file##*.}"
base="${file%.*}"

echo "Directory:   $dir"
echo "Filename:    $file"
echo "Extension:   $ext"
echo "Basename:    $base"
```

**Step-by-step for `/home/user/docs/report.pdf`:**
1. `dir="${fullpath%/*}"`: `%` removes shortest suffix matching `/*`. The last `/` is at position 15. Removes `/report.pdf`. Result: `/home/user/docs`.
2. `file="${fullpath##*/}"`: `##` removes longest prefix matching `*/`. Greedy match: everything up to the last `/`. Removes `/home/user/docs/`. Result: `report.pdf`.
3. `ext="${file##*.}"`: Greedy prefix removal of `*.`. Removes everything up to the last `.`. Result: `pdf`.
4. `base="${file%.*}"`: Shortest suffix removal of `.*`. Removes `.pdf`. Result: `report`.

<details>
<summary>Hint: Edge case — no extension</summary>
If `file` has no extension, `${file##*.}` returns the filename itself (no dot match). And `${file%.*}` returns the whole filename (no "." at the end to match). Test with `report` (no dot) to see.
</details>

<details>
<summary>Hint: Edge case — hidden file</summary>
For a file like `.bashrc`:
- `${file##*.}` = `bashrc` (the `.*` match removes the dot and everything before? No — `.bashrc` has `##*.` = everything up to the last `.*` = the whole string "bashrc" after the dot? Actually `.bashrc` — the greedy prefix match `*.` matches `.bashrc` (the `*` matches empty, then `.` matches the leading dot). So `${file##*.}` = `bashrc`. Correct!
- `${file%.*}` = `.bashrc` (no `.` after the leading one to match as suffix — actually `.bashrc` does have `.` at position 0, so suffix `.*` matches the whole string. Wait, `.*` matches a dot followed by anything. In `.bashrc`, the suffix match would be `.bashrc` itself. So `${file%.*}` = `` (empty). That's wrong!
- **Better approach for hidden files**: Don't use extension extraction on hidden files, or check if the filename starts with `.`.
</details>

### 2. URL Parser

Given a URL like `https://www.example.com:8080/path/to/page?q=search#section`, extract:
- **Protocol**: before `://`
- **Domain+Port**: after `://` up to next `/`
- **Path**: between domain and `?` or end
- **Query**: after `?`, before `#`
- **Fragment**: after `#`

```bash
url="$1"
proto="${url%%://*}"
rest="${url#*://}"
domain="${rest%%/*}"
path="${rest#*/}"
query="${path#*\?}"     # or path="${path%%\#*}"; query="${path#*\?}"
fragment="${path#*\#}"  # path was modified — careful!

# Better approach:
base="${rest%%\?*}"
domain="${base%%/*}"
pathpart="${base#*/}"
# Query and fragment from rest:
querystr="${rest#*\?}"
fragment="${querystr#*\#}"
queryonly="${querystr%%\#*}"
```

**Step-by-step:**
1. `proto="${url%%://*}"`: Longest suffix removal of `://*` — removes everything from `://` onward. Result: `https`.
2. `rest="${url#*://}"`: Shortest prefix removal of `*://` — removes up to and including `://`. Result: `www.example.com:8080/path/to/page?q=search#section`.
3. `domain="${rest%%/*}"`: Longest suffix removal of `/*` — wait, `%%` is greedy suffix removal. `rest` is `www.example.com:8080/path/...`. The longest suffix matching `/*` is everything from the first `/` onward? Actually `/*` matches a `/` followed by anything. The longest suffix starting from `/*` would be `/path/to/page?q=search#section`. So `%%/*` removes that. Result: `www.example.com:8080`.
4. `path="${rest#*/}"`: Shortest prefix removal of `*/` — removes up to first `/`. Result: `path/to/page?q=search#section`.

<details>
<summary>Hint: Escaping ? and # in patterns</summary>
The `?` and `#` are glob special characters. To match them literally, escape with backslash: `\?`, `\#`. OR rely on the fact that they only have special meaning in certain positions — `?` as a pattern matches any single char, so `*\?` explicitly matches a literal `?` prefix.
</details>

### 3. Batch Rename Plan

Write a loop that renames all `.jpg` files to `_backup.jpg` (before the extension). Use only parameter expansion.

```bash
#!/bin/bash
for f in *.jpg; do
    [ -f "$f" ] || continue   # skip if no .jpg files
    base="${f%.*}"
    mv -v "$f" "${base}_backup.jpg"
done
```

**What if** filenames have spaces? The loop works correctly because `for f in *.jpg` expands with proper globbing, and all variable expansions are quoted.

<details>
<summary>Hint: Making it safer with dry-run</summary>
```bash
for f in *.jpg; do
    base="${f%.*}"
    echo "mv \"$f\" \"${base}_backup.jpg\""
done
```
Replace `echo` with `mv` when you're confident.
</details>

### 4. Bonus: Batch Image Rename by Date

```bash
#!/bin/bash
# Rename all .jpg files to YYYY-MM-DD_originalname.jpg
for f in *.jpg; do
    base="${f%.*}"
    date_part=$(date -r "$f" +%F)   # modification date
    echo "mv \"$f\" \"${date_part}_${base}.jpg\""
done
```

### 5. Bonus: Extract Nth Character

```bash
str="$1"
n=$2
echo "${str:$n-1:1}"   # this is wrong — $n-1 is a string
# FIX: Use arithmetic
echo "${str:$((n-1)):1}"

# Test: ./getchar.sh "hello" 2 → "e"
```

## Expected Output

```
$ ./parse_path.sh /home/user/docs/report.pdf
Directory:   /home/user/docs
Filename:    report.pdf
Extension:   pdf
Basename:    report

$ ./parse_path.sh /home/user/docs/
Directory:   /home/user/docs
Filename:
Extension:
Basename:

$ ./parse_path.sh "https://www.example.com:8080/path/to/page?q=test#sec1"
Protocol:    https
Domain+Port: www.example.com:8080
Path:        path/to/page
Query:       q=test
Fragment:    sec1

$ ./parse_path.sh "/etc/hostname"
Directory:   /etc
Filename:    hostname
Extension:   hostname    # no dot — extension = full filename
Basename:    hostname

$ ls
photo.jpg  vacation.jpg  selfie.jpg
$ ./rename_backup.sh
photo.jpg → photo_backup.jpg
vacation.jpg → vacation_backup.jpg
selfie.jpg → selfie_backup.jpg
```

## Self-Check

- What does `${path##*/}` do? Could you use `${path#*/}` instead? Why or why not?
- How do you extract the last character of a string?
- What's the difference between `${var%.*}` and `${var%%.*}` when `var="a.b.c"`?
- Why does `${str: -3}` need a space?
- How would you get the first 5 characters of a string using parameter expansion?
- What happens when you run `${path##*/}` on a path that ends with `/`?
- How would you extract the filename without extension from `/path/to/.hidden`?

### Sub-task 7: Log Parser with Slicing

Write a script that reads a log file with lines like `[2025-07-30 14:30:22] ERROR: something broke` and uses parameter expansion slicing (not external tools) to extract:
- The date portion
- The time portion  
- The log level
- The message body

Test with:
```bash
$ echo "[2025-07-30 14:30:22] ERROR: something broke" | ./parse_log.sh
Date: 2025-07-30
Time: 14:30:22
Level: ERROR
Message: something broke
```

<details>
<summary>Hints</summary>

- Use `##` and `%%` to strip brackets and prefixes
- For fixed-width fields, `${var:0:10}` works cleanly
- Combine multiple expansions in a pipeline of parameter substitutions
</details>
