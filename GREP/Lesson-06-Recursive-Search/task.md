# GREP Lesson 06: Recursive Search & File Filtering

## Introduction

When you need to search across many files (not just one), grep's recursive flags come to the rescue. Combined with file filtering, you can target specific file types.

## Key Flags

| Flag | Meaning                                |
|------|----------------------------------------|
| `-r` or `-R` | Recursive search through directories |
| `--include=GLOB` | Only search files matching the glob |
| `--exclude=GLOB` | Skip files matching the glob       |
| `-f FILE` | Read patterns from a file, one per line |

## Examples

### Example 1: Simple recursive search
Search all files in a directory for "ERROR":
```
$ grep -r "ERROR" testdata/
testdata/file1.log:2026-07-29T10:15:34+00:00 appserver1 webapp[1234]: ERROR: Failed to initialize cache module...
testdata/file2.txt:ERROR is mentioned here but this is not a log file.
testdata/file3.conf:    error_log /var/log/nginx/error.log ERROR;
testdata/file4.json:      "status": "ERROR",
```

### Example 2: Search only .log files
```
$ grep -r --include="*.log" "ERROR" testdata/
testdata/file1.log:2026-07-29T10:15:34+00:00 appserver1 webapp[1234]: ERROR: Failed to initialize cache module...
```

### Example 3: Using a pattern file
Create a file with patterns, then use `-f`:
```
$ cat patterns.txt
ERROR
WARN
CRITICAL
failure
timeout

$ grep -r -f patterns.txt testdata/
```

### Example 4: Combining find with grep
```
$ find testdata/ -name "*.conf" -exec grep "ERROR" {} \;
```

## Task

Use the `testdata/` directory under this lesson folder.

### Step 1: Create a pattern file
A file called `patterns.txt` already exists in this lesson directory. It contains:
```
ERROR
WARN
CRITICAL
failure
timeout
```

### Step 2: Recursive search with patterns
Run `grep -r -f patterns.txt testdata/` to find all lines matching any pattern across all files.

### Step 3: Filter by file type
Now use `--include` to search only `.log` files within `testdata/`.

*Hint:* `grep -r -f patterns.txt --include='*.log' testdata/`

Compare the two outputs. How many lines come from files other than .log?

### Expected:
- Full search: 40 lines across all 4 files (file1.log, file2.txt, file3.conf, file4.json)
- Log-only search: 14 lines, all from file1.log (the only .log file)
