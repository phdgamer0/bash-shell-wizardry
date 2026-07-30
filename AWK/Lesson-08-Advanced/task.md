# AWK Lesson 08: Advanced Topics

This lesson covers more advanced AWK features: `getline`, `system()`, multi-file processing, and user-defined functions.

## User-Defined Functions

Define your own functions with the `function` keyword:

```awk
function function_name(param1, param2, ...) {
    # body
    return value
}
```

### Example 1: Function to categorize HTTP status

```awk
function status_category(code) {
    if (code >= 200 && code < 300) return "Success"
    else if (code >= 300 && code < 400) return "Redirect"
    else if (code >= 400 && code < 500) return "ClientError"
    else if (code >= 500 && code < 600) return "ServerError"
    else return "Other"
}
{
    cat = status_category($9)
    print $9, cat
}
```

## getline

`getline` reads the next line from a file or pipe into a variable.

```awk
# Read from a file
while ((getline line < "other.txt") > 0) {
    print line
}
close("other.txt")

# Read from a pipe
"wc -l target.txt" | getline linecount
print linecount
```

## system()

Run shell commands:

```awk
BEGIN {
    system("date")
    system("ls -la")
}
```

## Multi-file Processing

Use `FILENAME` and `FNR` to track which file you're processing.

```awk
{
    if (FILENAME == "file1.txt") {
        # process file1
    } else if (FILENAME == "file2.txt") {
        # process file2
    }
}
```

---

## Your Task

File: `target.txt` (access.log)

Write an AWK script (save as `report.awk`) that:

1. **Defines a function** `cat_status(s)` that returns the category string for an HTTP status code
2. **Uses `getline`** to capture the total line count of `target.txt` via a shell command (`wc -l target.txt | getline total_lines`)
3. In `BEGIN`, prints a report header with the generated date using `system("date ...")`
4. For each line, collects:
   - Unique IPs
   - Path request counts (field $7)
   - Status code categories
   - Bytes transferred (field $10) - track min, max, sum
5. In `END`:
   - Print total lines processed and unique IP count
   - Print top 5 most requested paths
   - Print bytes summary (total, min, max, average with printf)
   - Print status code distribution with percentages

Expected output format:

```
=== Access Log Summary Report ===
Generated: 2026-07-31 01:28:48

Total lines processed: 10000
Unique IP addresses: 1753

--- Top 5 Paths ---
1. /favicon.ico (807)
2. /style2.css (546)
3. /reset.css (538)

--- Bytes Transferred ---
Total: 2747282740 bytes
Min: 0 bytes
Max: 69192717 bytes
Average: 274728 bytes

--- Status Code Distribution ---
Category           Count Percent
---------------  ------- -------
Success             9171   91.7%
Redirect             609    6.1%
ClientError          217    2.2%
ServerError            3    0.0%
```
