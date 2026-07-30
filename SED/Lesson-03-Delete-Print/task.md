# Lesson 03: Delete, Print, and Quit

## The `d` Command (Delete)

The `d` command deletes the current line from output:
```bash
sed '/pattern/d' file.txt  # Delete all lines containing "pattern"
```

You can combine `d` with line number addressing:
```bash
sed '5d' file.txt           # Delete line 5
sed '10,20d' file.txt       # Delete lines 10 through 20
sed '10,$d' file.txt        # Delete from line 10 to end
```

## The `p` Command (Print)

With `-n` (quiet mode), `p` prints the current line:
```bash
sed -n '/pattern/p' file.txt  # Print only lines matching "pattern"
```

This is equivalent to `grep pattern file.txt`.

## The `q` Command (Quit)

The `q` command tells sed to stop processing immediately:
```bash
sed '5q' file.txt     # Print first 5 lines, then quit
sed '/pattern/q' file.txt  # Print lines until "pattern" is found, then quit
```

## Combining with `-n`

Using `-n` suppresses normal output, so only lines explicitly printed with `p` show up:
```bash
sed -n '10,20p' file.txt  # Print lines 10-20 only
```

## Multiple commands with `-e` or `;`

Chain multiple commands:
```bash
sed -n -e '/error/p' -e '/fail/p' file.txt  # Print lines with "error" or "fail"
# or
sed -n '/error/p; /fail/p' file.txt
```

But this may print a line twice if it matches both patterns.

## Examples

### Example 1: Delete all "debug" lines
```bash
$ sed '/debug/d' system.log | head -3
# Shows first 3 lines that do NOT contain "debug"
```

### Example 2: Print only lines with "error"
```bash
$ sed -n '/error/p' system.log | head -3
2026-07-29T03:18:03+03:00 archlinux kernel: ACPI BIOS Error (bug): ...
2026-07-29T03:18:03+03:00 archlinux kernel: ACPI Error: AE_NOT_FOUND...
```

### Example 3: Quit after first "fail"
```bash
$ sed '/fail/q' system.log
# Prints lines from the start until the first line containing "fail"
```

## Task

Using **`target.txt`** (system.log, 2500 lines):

1. Delete all lines that contain "systemd" **OR** "NetworkManager"
2. From the remaining lines, **print only** lines that contain "error" or "fail" (case insensitive — match "Error", "ERROR", "Fail", "FAIL", etc.)
3. Save this result
4. Additionally, find the **first** line containing "critical" (case insensitive) and quit immediately, printing only that line

**Hint:** For case insensitive matching, use character classes like `[Ee][Rr][Rr][Oo][Rr]` or the `I` flag if supported. Use `|` (piped via `\|` in sed) for OR between patterns.

**Expected result:** A filtered set of lines free of "systemd" and "NetworkManager", containing only lines with error/fail messages. The quit command should output one specific line containing "critical".
