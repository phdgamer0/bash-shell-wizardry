# AWK Lesson 04: Control Flow

AWK supports control flow statements like `if/else`, `while`, `for`, and `do-while`, just like C or JavaScript.

## If / Else

```awk
if (condition) {
    action
} else if (condition2) {
    action2
} else {
    action3
}
```

### Example 1: Categorize HTTP Status Codes

Given Apache log where status is field $9:

```bash
awk '{
    status = $9 + 0
    if (status >= 200 && status < 300) category = "Success"
    else if (status >= 400 && status < 500) category = "ClientError"
    else category = "Other"
    print $9, category
}' access.log
```

## While Loops

```awk
while (condition) {
    action
}
```

### Example 2: Process all fields with a while loop

```bash
awk '{ i=1; while(i <= NF) { print "Field", i, "=", $i; i++ } }' file.txt
```

## For Loops

```awk
for (initialization; condition; increment) {
    action
}
```

### Example 3: Sum all numeric fields

```bash
awk '{ sum=0; for(i=1; i<=NF; i++) sum += $i; print "Sum:", sum }' file.txt
```

## printf for Formatted Output

`printf` gives precise control over formatting:

- `%s` = string
- `%d` = integer
- `%f` = floating point
- `%-10s` = left-aligned, width 10
- `%10d` = right-aligned integer, width 10

```bash
awk '{ printf "%-10s %5d\n", $1, $2 }' file.txt
```

---

## Your Task

File: `target.txt` (access.log - Apache combined log format)

Format: `IP - - [date] "METHOD /path HTTP/1.1" status bytes "referrer" "user-agent"`

Field mapping (default whitespace FS):
- `$6` = METHOD (starts with "GET, "POST, etc.)
- `$9` = HTTP status code (e.g., 200, 404, 500)
- `$10` = bytes transferred

Write an AWK command that:

1. Categorizes each HTTP status code:
   - 2xx (200-299) → "Success"
   - 3xx (300-399) → "Redirect"
   - 4xx (400-499) → "ClientError"
   - 5xx (500-599) → "ServerError"
2. Counts how many of each category using an `if/else` chain
3. In `END`, prints each category and its count
4. Also prints (to stdout) the full lines where method is GET (`$6 ~ /"GET/`) AND status is 200 (limit to first 5 matches in expected output)

Expected output format lines for GET/200 requests, followed by the summary.

```
=== Status Code Summary ===
Success: 9171
Redirect: 609
ClientError: 217
ServerError: 3
```
