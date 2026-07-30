# AWK Lesson 03: Built-in Variables

AWK provides many useful built-in variables that give information about the input and control output formatting.

## Key Variables

| Variable | Meaning |
|----------|---------|
| `FS`     | Field Separator (input) |
| `OFS`    | Output Field Separator |
| `RS`     | Record Separator (input, default newline) |
| `ORS`    | Output Record Separator (default newline) |
| `NR`     | Number of the current Record (line number, cumulative across files) |
| `NF`     | Number of Fields in the current record |
| `FILENAME` | Name of the current input file |
| `FNR`    | File-specific Number of Records (resets per file) |

## Changing Output Separators

You can control how fields are printed.

### Example 1: Using OFS

```bash
awk 'BEGIN { FS="|"; OFS=" | " } { print $1, $2, $3 }' file.csv
```

Now when you `print $1, $2, $3`, they are joined with " | " instead of space.

### Example 2: Printing Line Numbers with NR

```bash
awk '{ print NR": "$0 }' file.txt
```

Numbers each line: `1: content`, `2: content`, etc.

### Example 3: Number of Fields per Line

```bash
awk '{ print "Line", NR, "has", NF, "fields" }' file.txt
```

Useful for checking data consistency.

---

## Your Task

File: `target.txt` (transactions.csv - pipe-delimited)

Write an AWK command that:

1. Sets `FS="|"` and `OFS=" | "` in a `BEGIN` block
2. In `BEGIN`, prints a header: `NR | User | Action | Resource`
3. For each line, prints: NR, username ($2), action ($3), resource ($4)
4. In `END`, prints:
   - A separator line `---`
   - `Lines processed: <NR>`
   - `Filename: <FILENAME>`

Expected format:

```
NR | User | Action | Resource
1 | eve | UPDATE | /etc/nginx/nginx.conf
2 | bob | WRITE | /home/user/.ssh/id_rsa
...
---
Lines processed: 3000
Filename: target.txt
```
