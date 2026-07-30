# Task 10: Build a Backup Script

Write a robust backup script that takes a directory path and creates a timestamped tar archive. Along the way, you'll learn argument parsing, error handling, and how to make scripts "cron-ready."

## Steps

### Sub-task 1: Skeleton Script
Create `~/backup.sh` with the basics:
- Shebang line
- Prompt the user: "Enter directory to backup:"
- Read input with `read -r`
- Check if the directory exists with `[ -d ]`
- Exit with error code 1 if directory doesn't exist
- Create a filename: `backup_DIRNAME_YYYY-MM-DD.tar.gz`
- Create the tar archive with `tar -czf`
- Print success message with the archive name and size

```bash
#!/bin/bash
read -p "Enter directory to backup: " -r dir
if [ ! -d "$dir" ]; then
    echo "Error: Directory $dir does not exist." >&2
    exit 1
fi

dirname=$(basename "$dir")
archive="backup_${dirname}_$(date +%Y-%m-%d).tar.gz"

tar -czf "$archive" "$dir" 2>/dev/null
if [ $? -eq 0 ]; then
    size=$(du -h "$archive" | cut -f1)
    echo "Backup created: $archive ($size)"
else
    echo "Error: tar command failed." >&2
    exit 2
fi
```

**Approach 1:** Use `basename` to extract the directory name
**Approach 2:** Use parameter expansion: `${dir%/}` then `${dir##*/}`
**Approach 3:** Use `realpath` for absolute paths

<details><summary>Hint: Getting just the directory name</summary>
`basename /home/phd/projects` outputs `projects`. `basename /var/log/` outputs `log` (trailing slash is handled). For the date, `date +%Y-%m-%d` gives ISO format.
</details>

### Sub-task 2: Add Error Handling
Improve the script with:
- If the directory doesn't exist, print error to stderr and exit with code 1
- If the tar command fails, exit with code 2
- If the disk is full or permission denied for writing, show the actual error

```bash
tar -czf "$archive" "$dir" 2>&1
tar_exit=$?
if [ $tar_exit -ne 0 ]; then
    echo "Error: tar failed with exit code $tar_exit" >&2
    exit 2
fi
```

**Approach 1:** Check exit code with `$?`
**Approach 2:** Use `set -e` and `set -E` with trap
**Approach 3:** Use `tar -czf "$archive" "$dir" || exit 2`

<details><summary>Hint: Why to check exit codes</summary>
Without checking, if `tar` fails (disk full, permission denied), the script cheerfully says "Success!" even though no valid archive was created. Always verify.
</details>

### Sub-task 3: Add Command-Line Argument Support
Modify the script so the directory can be given as a command-line argument (`$1`). If no argument, then prompt.

```bash
if [ $# -ge 1 ]; then
    dir="$1"
else
    read -p "Enter directory to backup: " -r dir
fi
```

**Approach 1:** Check `$#` count
**Approach 2:** Use `${1:-}` default — but that doesn't interactively prompt
**Approach 3:** Use `getopts` for proper option parsing

**Bonus:** Add support for `-o filename` to specify output name:
```bash
$ ./backup.sh -o mybackup.tar.gz /var/log
```

<details><summary>Hint: $# vs $1</summary>
`$#` is the NUMBER of arguments. `$1` is the FIRST argument. `if [ $# -eq 0 ]` checks if no arguments given. `if [ -n "${1:-}" ]` checks if first arg is non-empty.
</details>

### Sub-task 4: Add a --quiet Flag
Add `--quiet` flag that suppresses all output except errors. This makes the script suitable for cron jobs.

```bash
quiet=0
if [ "$1" = "--quiet" ]; then
    quiet=1
    shift
fi

# Then in the success case:
if [ $quiet -eq 0 ]; then
    echo "Backup created: $archive ($size)"
fi

# Errors should ALWAYS print, regardless of quiet mode.
```

**Approach 1:** Simple `--quiet` check as shown
**Approach 2:** Use `getopts` for `-q`
**Approach 3:** Log to syslog instead of stdout in quiet mode: `logger "Backup created: $archive"`

```bash
$ ./backup.sh --quiet /tmp
# No output, but file exists
$ ls backup_tmp_2026-07-31.tar.gz
backup_tmp_2026-07-31.tar.gz
```

<details><summary>Hint: getopts for option parsing</summary>
```bash
quiet=0
while getopts ":qo:" opt; do
    case "$opt" in
        q) quiet=1 ;;
        o) outfile="$OPTARG" ;;
        \?) echo "Usage: $0 [-q] [-o file] [directory]" >&2; exit 1 ;;
    esac
done
shift $((OPTIND-1))
```
</details>

### Sub-task 5: Verify the Archive
After creating the archive, verify its integrity:

```bash
# Check that the archive exists and has content
if [ ! -f "$archive" ] || [ ! -s "$archive" ]; then
    echo "Error: Archive file is missing or empty!" >&2
    exit 3
fi

# Verify tar integrity
tar -tzf "$archive" > /dev/null 2>&1
if [ $? -ne 0 ]; then
    echo "Error: Archive verification failed!" >&2
    exit 4
fi

echo "Archive verified successfully."
```

**Approach 1:** Use `tar -tzf` to list contents (t=test/list)
**Approach 2:** Use `gzip -t` on the gzip layer, then `tar -tf` on the tar layer
**Approach 3:** Compute checksum: `sha256sum "$archive" > "$archive.sha256"`

<details><summary>Hint: tar -t test</summary>
`tar -tzf file.tar.gz` reads the compressed archive and lists its contents without extracting. It returns 0 if the archive is valid, non-zero if corrupt. This is the standard "archive integrity check."
</details>

### Sub-task 6: Directory Empty Check
Add detection for empty directories (backing up an empty dir is valid but worth noting):

```bash
file_count=$(find "$dir" -type f 2>/dev/null | wc -l)
if [ "$file_count" -eq 0 ]; then
    echo "Warning: Directory $dir contains no files."
fi
```

**Bonus:** Add `--exclude` pattern support using `tar --exclude`:
```bash
tar -czf "$archive" --exclude="*.tmp" --exclude="*.log" "$dir"
```

### Sub-task 7: Full Integration Test
Test the complete script:

```bash
# 1. Create a test directory
mkdir -p /tmp/backup_test/subdir
echo "test file" > /tmp/backup_test/file1.txt
echo "another" > /tmp/backup_test/subdir/file2.txt

# 2. Run backup
$ ./backup.sh /tmp/backup_test
Backup created: backup_backup_test_2026-07-31.tar.gz (1.2K)

# 3. Verify contents
$ tar -tzf backup_backup_test_2026-07-31.tar.gz
backup_test/
backup_test/file1.txt
backup_test/subdir/
backup_test/subdir/file2.txt

# 4. Clean up
rm -f backup_backup_test_2026-07-31.tar.gz
rm -rf /tmp/backup_test
```

## Expected Output

```bash
$ ./backup.sh
Enter directory to backup: /tmp/quoting_lab
Backup created: backup_quoting_lab_2026-07-31.tar.gz (1.2K)

$ ./backup.sh /var/log
Backup created: backup_var_log_2026-07-31.tar.gz (4.5M)

$ ./backup.sh /nonexistent
Error: Directory /nonexistent does not exist.
$ echo $?
1

$ ./backup.sh --quiet /tmp
# No output, but file exists
$ ls backup_tmp_2026-07-31.tar.gz
backup_tmp_2026-07-31.tar.gz

$ ./backup.sh /empty_test
Warning: Directory /empty_test contains no files.
Backup created: backup_empty_test_2026-07-31.tar.gz (0.1K)
```

## Self-Check Questions

1. What does `#!/bin/bash` do?

2. Why is `chmod +x` needed?

3. What is the difference between `echo` and `printf`?

4. What is `$?` and when should you check it?

5. Why would you use `read -r` instead of `read`?

6. What does `>&2` do and why is it important for error messages?

7. Why is the `--quiet` flag important for cron jobs?
