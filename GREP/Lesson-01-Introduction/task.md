# GREP Lesson 01: Introduction to grep

## What is grep?

`grep` (Global Regular Expression Print) is a command-line tool for searching plain-text data for lines matching a pattern. It is one of the most essential tools in a Linux user's toolkit.

## Basic Usage

The basic syntax is:
```
grep [options] pattern [file...]
```

### Example 1: Basic pattern search
Search for the word "error" in `target.txt` (case-sensitive):
```
$ grep "error" target.txt
2026-07-29T03:18:04+03:00 archlinux kernel: i8042: PNP: PS/2 appears to have AUX port disabled, if this is incorrect please boot with i8042.nopnp
2026-07-29T03:19:56+03:00 archlinux kernel: warning: `kdeconnectd' uses wireless extensions which will stop working for Wi-Fi 7 hardware; use nl80211
...
```

### Example 2: Case-insensitive search with -i
The `-i` flag makes the search case-insensitive, matching "Error", "ERROR", "error", etc.:
```
$ grep -i "error" target.txt
```
This finds many more matches than the case-sensitive version.

### Example 3: Showing line numbers with -n
The `-n` flag shows the line number of each match:
```
$ grep -n "error" target.txt
2036:2026-07-29T03:20:10+03:00 archlinux dbus-broker-launch[613]: Activation request for...
```

### Example 4: Counting matches with -c
The `-c` flag prints only the count of matching lines:
```
$ grep -ci "error" target.txt
42
```

## Other Useful Flags

- `-v` — Invert match (show lines that do NOT match)
- `-c` — Count matching lines (show only the number)
- `-n` — Prefix each match with its line number
- `-i` — Case-insensitive matching

## Task

Your task has two parts:

1. **Find all lines** in `target.txt` that contain the word "fail" (case-insensitive). Show the line numbers alongside each match.
   - *Hint:* Use `grep -in`

2. **Count** how many lines contain "fail" (case-insensitive).
   - *Hint:* Use `grep -ci`

### Expected output:
When you run the correct commands, you should see output similar to the file `expected-result.txt`. The first command will output hundreds of lines (each prefixed with a line number), and the second command will show just a single number.

### Commands to run:
```bash
grep -in "fail" target.txt
grep -ci "fail" target.txt
```
