# Lesson 04: Insert, Append, and Change

sed can not only modify line content but also add, remove, or replace entire lines.

## The `i` Command (Insert)

Insert a line **before** the current line:
```bash
sed '/pattern/i\NEW LINE' file.txt
```

The backslash after `i` starts the text to insert.

## The `a` Command (Append)

Add a line **after** the current line:
```bash
sed '/pattern/a\NEW LINE' file.txt
```

## The `c` Command (Change)

Replace the entire current line with new text:
```bash
sed '/pattern/c\REPLACEMENT LINE' file.txt
```

## The `r` Command (Read)

Read content from a file and insert it after the current line:
```bash
sed '/pattern/r otherfile.txt' file.txt
```

## The `w` Command (Write)

Write the current line to a file:
```bash
sed '/pattern/w output.txt' file.txt
```

## Examples

### Example 1: Insert a header before each line with 404
```bash
$ sed '/ 404 /i\# 404 NOT FOUND' access.log
```

### Example 2: Append a warning after each 500 response
```bash
$ sed '/ 500 /a\# WARNING: Server Error' access.log
```

### Example 3: Change all lines with 200 status
```bash
$ sed '/ 200 /c\# 200 OK' access.log
```

## Task

Using **`target.txt`** (access.log, 2500 lines, Apache Combined Log Format):

1. **Before** every line where the HTTP status is **404**, insert a comment line: `# 404 NOT FOUND`
2. **After** every line where the status is **500**, append: `# 500 SERVER ERROR`
3. On lines with status **200**, **change** the entire line to: `# 200 OK - <original_ip>` (extract the IP address from the beginning of the line)

**Hint:** The status code appears after the HTTP request string. Look for patterns like `" 404 ` (quote-space-404-space). For the IP extraction on 200 lines, use substitution: `s/^\([0-9.]*\).*/# 200 OK - \1/`

**Expected result:** 404 lines preceded by `# 404 NOT FOUND`, 500 lines followed by `# 500 SERVER ERROR`, and all 200 lines replaced with `# 200 OK - <ip>`. Other lines (301, 304, etc.) remain unchanged. Total lines should be 2500 + 49 inserted + 1 appended = 2550.
