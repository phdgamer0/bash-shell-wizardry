# Lesson 06: Multiline Operations

By default, sed works on a single line at a time. The multiline commands extend sed's power to handle patterns that span multiple lines.

## The `N` Command (Next — Append to Pattern Space)

`N` reads the **next** line and **appends** it to the current pattern space, separated by a newline `\n`:

```bash
sed 'N; s/\n/ /' file.txt
```

This joins every pair of lines into a single line.

## The `D` Command (Delete First Line)

`D` deletes the first line of the pattern space (up to the first newline) and then restarts the sed cycle on the remaining text:
```bash
sed 'N; /pattern/D' file.txt
```

## The `P` Command (Print First Line)

`P` prints only the first line of the pattern space (up to the first newline):
```bash
sed 'N; /pattern/P' file.txt
```

## Common Patterns

Join pairs of lines:
```bash
sed 'N; s/\n/ /' file.txt
```

Process multi-line blocks:
```bash
sed -n '/^START$/{:loop;N;/^END$/!bloop;/CONTENT/p}' file.txt
```

This finds blocks from `START` to `END` that contain `CONTENT` and prints them.

## Examples

### Example 1: Use N to join lines
```bash
$ cat pairs.txt
line one
line two
line three
line four
$ sed 'N; s/\n/ /' pairs.txt
line one line two
line three line four
```

### Example 2: Using N;P;D to process line pairs
```bash
$ sed 'N; /error/p; D' system.log
```

### Example 3: Printing multi-line blocks containing "FAIL"
```bash
sed -n '/^BEGIN EVENT$/{:loop;N;/\nEND EVENT$/!bloop;/Status: FAIL/p}' events.txt
```

This reads an entire event block, checks if it has "Status: FAIL", and only prints those blocks.

## Task

Using **`target.txt`** (multi-line event log, 2000+ lines):

The file contains structured event blocks in this format:
```
BEGIN EVENT
Timestamp: 2024-03-20 12:36:32
User: bob
Action: READ
Resource: /tmp/cache.dat
Status: FAIL
Message: Some message
Detail-1: Additional info
END EVENT
```

Find all "EVENT" blocks where the **Status** is "FAIL" and print the **entire block** (from BEGIN to END). Use the `N` command to read the full block into pattern space.

**Hint:** Use `:label` to create a loop with `N` to accumulate lines. Check for `/\nEND EVENT$/` to know when the block ends. Then check if "Status: FAIL" exists somewhere in the pattern space.

**Expected result:** Only blocks where `Status: FAIL` should appear. Each block should be complete from `BEGIN EVENT` to `END EVENT`. Non-FAIL blocks should be suppressed.
