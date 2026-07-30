# Task 8: `xargs` & Parallel

## Overview

Write three scripts demonstrating `xargs` for batch processing and parallel execution. The goal is to understand when `xargs` is needed (ARG_MAX, parallelization, safe filename handling) and how to use it effectively.

## Sub-tasks

### 1. Parallel Image Resizer (resize_parallel.sh)

Find all `.jpg` files and use `convert` (ImageMagick) via `xargs -P` to resize them to 800×600. Compare sequential vs parallel timing.

**Requirements:**
- Count how many .jpg files exist
- Run sequential resize (time it)
- Run parallel resize with -P $(nproc) (time it)
- Print timing comparison

```bash
#!/bin/bash
# Create test images if needed
create_test_images() {
    for i in {1..12}; do
        convert -size 1920x1080 xc:blue "test_${i}.jpg" 2>/dev/null
    done
}

RESIZE="800x600"
COUNT=$(find . -name "*.jpg" | wc -l)

echo "Resizing $COUNT images..."

# Sequential
echo "--- Sequential ---"
TIMEFORMAT='Sequential: %3R seconds'
time {
    find . -name "*.jpg" -print0 | xargs -0 -I {} convert {} -resize "$RESIZE" "{}_small.jpg"
}

# Clean small files
find . -name "*_small.jpg" -delete

# Parallel
CORES=$(nproc)
echo "--- Parallel ($CORES workers) ---"
TIMEFORMAT="Parallel ($CORES workers): %3R seconds"
time {
    find . -name "*.jpg" -print0 | xargs -0 -P "$CORES" -I {} convert {} -resize "$RESIZE" "{}_small.jpg"
}
```

<details>
<summary>Hint: If you don't have ImageMagick</summary>
Simulate with `sleep` instead:
```bash
# Instead of convert, use:
find . -name "*.jpg" -print0 | xargs -0 -I {} sh -c 'sleep 0.1; echo "Resized {}"'
```
</details>

**Step-by-step for parallel:**
1. `find . -name "*.jpg" -print0` outputs filenames NUL-delimited (safe).
2. `xargs -0` reads NUL-delimited input.
3. `-P "$(nproc)"` runs up to N processes simultaneously.
4. `-I {}` replaces `{}` with each filename.
5. `convert {} -resize 800x600 {}_small.jpg` resizes and saves.

### 2. Batch Grep (batchgrep.sh)

Take a pattern and directory, search files in batches of 5 using `find` piped to `xargs -n 5`.

```bash
#!/bin/bash
pattern="$1"
dir="${2:-.}"

if [[ -z "$pattern" ]]; then
    echo "Usage: $0 <pattern> [directory]" >&2
    exit 1
fi

echo "Searching for '$pattern' in $dir..."
find "$dir" -type f -print0 | xargs -0 -n 5 grep -H "$pattern" 2>/dev/null
```

**Alternative — with file count and match counting:**
```bash
#!/bin/bash
pattern="$1"; dir="${2:-.}"
file_count=$(find "$dir" -type f | wc -l)
match_count=$(find "$dir" -type f -print0 | xargs -0 -n 5 grep -l "$pattern" 2>/dev/null | wc -l)
matches=$(find "$dir" -type f -print0 | xargs -0 -n 5 grep -Hn "$pattern" 2>/dev/null)
echo "Searched $file_count files, found matches in $match_count files"
echo "$matches"
```

<details>
<summary>Hint: Why -n 5?</summary>
`-n 5` limits each `grep` invocation to 5 files. Without `-n`, `xargs` would pass as many files as fit in ARG_MAX (~2MB). `-n 5` demonstrates batching and lets you see multiple grep invocations in action. Add `-t` to see them: `xargs -t -0 -n 5 grep ...`
</details>

### 3. Safe File Mover (safe_move.sh)

Find `.log` files older than 30 days and move them to `archive/`. Use `-print0` + `xargs -0` for safety. Support `--dry-run`.

**Requirements:**
- Create `archive/` directory if it doesn't exist
- Use `-print0` and `xargs -0` (safe path handling)
- Support `--dry-run` flag (show what would be moved)
- Print summary of files moved

```bash
#!/bin/bash
DRY_RUN=false
[[ "$1" == "--dry-run" ]] && DRY_RUN=true

mkdir -p archive

move_cmd() {
    if $DRY_RUN; then
        echo "mv" "$@"    # show what would run
    else
        mv "$@"
    fi
}

if $DRY_RUN; then
    echo "=== DRY RUN ==="
    find . -maxdepth 1 -name "*.log" -mtime +30 -print0 | xargs -0 -I {} echo mv {} archive/
else
    count=$(find . -maxdepth 1 -name "*.log" -mtime +30 -print0 | xargs -0 -I {} mv {} archive/ 2>&1 | wc -l)
    # Alternative with count via find first
    files=$(find . -maxdepth 1 -name "*.log" -mtime +30)
    file_count=$(echo "$files" | grep -c .)
    if (( file_count > 0 )); then
        find . -maxdepth 1 -name "*.log" -mtime +30 -print0 | xargs -0 -I {} mv {} archive/
        echo "Moved $file_count files to archive/"
    else
        echo "No files to move"
    fi
fi
```

**Better implementation:**
```bash
#!/bin/bash
DRY_RUN=false
SRC_DIR="${1:-.}"
ARCHIVE="${SRC_DIR}/archive"

[[ "$*" == *--dry-run* ]] && DRY_RUN=true
mkdir -p "$ARCHIVE"

# Count files
mapfile -t files < <(find "$SRC_DIR" -maxdepth 1 -name "*.log" -mtime +30 -print0 2>/dev/null | xargs -0 -n 1 echo)
count=${#files[@]}

if (( count == 0 )); then
    echo "No .log files older than 30 days found"
    exit 0
fi

echo "Found $count files to move"

if $DRY_RUN; then
    echo "Would move:"
    find "$SRC_DIR" -maxdepth 1 -name "*.log" -mtime +30 -print0 | xargs -0 -I {} echo "  {} → $ARCHIVE/"
else
    find "$SRC_DIR" -maxdepth 1 -name "*.log" -mtime +30 -print0 | xargs -0 -I {} mv {} "$ARCHIVE/"
    echo "Moved $count files to $ARCHIVE/"
fi
```

<details>
<summary>Hint: Testing with mtime</summary>
Create test files:
```bash
touch -t 202001010000 old_file.log    # 6+ years old — definitely older than 30 days
touch -t $(date +%Y%m%d%H%M.%S) new_file.log  # today
```
Or use `-mmin`:
```bash
find . -name "*.log" -mmin +43200   # 43200 min = 30 days
```
</details>

### 4. Bonus: Parallel Downloader

```bash
#!/bin/bash
URLS_FILE="$1"
[[ -z "$URLS_FILE" ]] && { echo "Usage: $0 urls.txt"; exit 1; }

cat "$URLS_FILE" | xargs -P 5 -I {} curl -s -O {} 2>/dev/null
echo "Downloads initiated (parallel: 5)"
```

### 5. Bonus: Split Large File and Process with xargs

```bash
#!/bin/bash
# Split a large file into chunks, process each with xargs
seq 1 1000 > /tmp/data.txt

# Process in parallel batches of 10 numbers each
# Sum each batch
cat /tmp/data.txt | xargs -n 10 -P 4 sh -c 'echo "$@" | tr " " "+" | bc'
```

## Expected Output

```
$ ./resize_parallel.sh
Resizing 12 images...
Sequential: 12.34s
Parallel (4 workers): 3.45s

$ ./batchgrep.sh "error" ~/logs/
./logs/app.log:23: error: connection refused
./logs/system.log:5: error: disk full
./logs/debug.log:112: error: timeout exceeded

$ ./safe_move.sh --dry-run
mv ./logs/debug.log archive/
mv ./logs/access.log.1 archive/
Would move 2 files. Run without --dry-run to execute.

$ ./safe_move.sh
Moved 2 files to archive/

$ ./safe_move.sh
No .log files older than 30 days found
```

## Self-Check

- Why does `xargs -0` pair with `find -print0`?
- What does the `-P` flag do, and when is it beneficial?
- What problem does `-I` solve that plain `xargs` doesn't?
- Why might parallel output from `xargs -P` look garbled?
- What does `-r` prevent?
- What's the difference between `-n 1` and `-L 1`?
- How does `xargs` handle filenames with spaces by default (without `-0`)?
