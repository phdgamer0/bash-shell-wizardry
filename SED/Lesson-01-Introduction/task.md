# Lesson 01: Introduction to sed

## What is sed?

**sed** (Stream Editor) is a non-interactive text editor that processes text line by line. It reads input line by line, applies editing commands, and outputs the result. It's one of the most powerful tools for text transformation in Unix/Linux.

## Basic Syntax

```bash
sed 'command' file.txt    # Apply command to file, output to stdout
sed 'command' < file.txt  # Same as above using stdin
sed -i 'command' file.txt # Edit file in-place (careful!)
```

The most common sed command is `s` (substitute):

```bash
sed 's/old/new/' file.txt
```

This replaces the **first occurrence** of "old" with "new" on each line.

## The Substitute Command (`s///`)

### Basic substitution
Replace the first occurrence of "old" with "new" on each line:
```bash
$ echo "old old old" | sed 's/old/new/'
new old old
```

### Global flag (`g`)
Replace ALL occurrences on each line:
```bash
$ echo "old old old" | sed 's/old/new/g'
new new new
```

### Case-insensitive flag (`I`)
Match patterns regardless of case:
```bash
$ echo "OLD Old old" | sed 's/old/New/Ig'
New New New
```

### Quiet mode (`-n`) with print (`p`)
Suppress automatic output and only print lines where a substitution was made:
```bash
$ sed -n 's/pattern/replacement/p' file.txt
```

### Write flag (`w`)
Write lines where substitution occurred to a file:
```bash
$ sed -n 's/pattern/replacement/w output.txt' file.txt
```

### The `&` metacharacter
In the replacement string, `&` represents the entire matched pattern:
```bash
$ echo "hello world" | sed 's/world/(&)/'
hello (world)
```

This wraps the matched "world" in parentheses. The `&` is useful when you want to keep the matched text and add something around it.

## Examples

### Example 1: Replace "error" with "ERROR"
```bash
$ echo "found an error in the system" | sed 's/error/ERROR/'
found an ERROR in the system
```

### Example 2: Global replace "warn" with "WARN"
```bash
$ echo "warn: this is a warn level message" | sed 's/warn/WARN/g'
WARN: this is a WARN level message
```

### Example 3: Using `&` to wrap IP addresses in brackets
```bash
$ echo "Connection from 192.168.1.1 denied" | sed 's/[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}/[&]/'
Connection from [192.168.1.1] denied
```

## Task

Using **`target.txt`** (system.log, 2500 lines):

1. Replace **all** occurrences of "systemd" with "SYSTEMD" (case insensitive — match "systemd", "SYSTEMD", "Systemd", etc.)
2. Wrap all PID numbers (like `[1234]`) in curly braces: `{1234}` instead of `[1234]`
3. Show **only** lines that were changed (use `-n` and `p`)

**Hint:** Combine multiple `s///` commands by separating them with semicolons: `sed 's/one/two/; s/three/four/'`

**Expected result:** All lines should show SYSTEMD (with PID numbers in curly braces `{...}` instead of brackets `[...]`). The output should have 2500 lines since all lines contain PIDs.
