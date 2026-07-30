# Lesson 02: Addressing in sed

By default, `s///` applies to **every** line in the file. Addressing lets you restrict commands to specific lines.

## Line Number Addressing

### Single line
Apply a command only to a specific line number:
```bash
sed '5s/foo/bar/' file.txt  # Only line 5
```

### Line range
Apply a command to a range of lines:
```bash
sed '10,20s/foo/bar/' file.txt  # Lines 10 through 20
```

### Last line
Use `$` to refer to the last line:
```bash
sed '$s/foo/bar/' file.txt  # Only the last line
```

## Regex Addressing

Match lines containing a pattern:
```bash
sed '/pattern/s/foo/bar/' file.txt  # Lines matching "pattern"
```

Combine with line numbers:
```bash
sed '/error/,/end/s/foo/bar/' file.txt  # From line matching "error" to line matching "end"
```

## Negation (`!`)

Invert the match — apply the command to lines that do NOT match:
```bash
sed '/pattern/!s/foo/bar/' file.txt  # Lines NOT matching "pattern"
```

## Examples

### Example 1: Substitute on line 5 only
```bash
$ cat example.txt
line one
line two
line three
line four
line five
$ sed '5s/two/CHANGED/' example.txt
line one
line two
line three
line four
line five
```

### Example 2: Substitute on lines 10-20
```bash
$ sed '10,20s/old/new/' data.txt
```

### Example 3: Substitute on lines matching "DENY"
```bash
$ sed '/DENY/s/ALLOW/BLOCK/' access.log
```

### Example 4: Delete lines NOT containing "error"
```bash
sed -n '/error/p' file.txt   # print only lines with "error"
sed '/error/!d' file.txt     # delete lines without "error"
```

## Task

Using **`target.txt`** (transactions.csv, 3000 lines, pipe-delimited format: `datetime|user|action|resource|code`):

1. On lines where the **user** is "alice" **or** "bob", replace their username with "REDACTED"
2. Delete all lines where the **action** is "DENY"

**Hint:** The pipe `|` is the delimiter. To match the user field exactly, use patterns like `|alice|` (with surrounding pipes). To delete lines with DENY, use the `d` command with address: `/DENY/d`

**Expected result:** Usernames "alice" and "bob" should be replaced with "REDACTED". All lines where the action column contains "DENY" should be removed.
