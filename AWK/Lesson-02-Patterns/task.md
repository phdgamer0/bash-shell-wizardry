# AWK Lesson 02: Patterns

Patterns control WHEN the action runs. There are several types.

## BEGIN / END Blocks

- `BEGIN { }` runs once BEFORE reading the first line (great for headers, setting variables)
- `END { }` runs once AFTER the last line (great for summaries, totals)

### Example 1: Header and Footer

```bash
awk 'BEGIN { print "=== Report ===" } 
     { print $0 } 
     END { print "=== End of Report ===" }' file.txt
```

## Pattern Matching: /pattern/

Use `/pattern/` to run the action only on lines that contain that pattern.

### Example 2: Print only lines containing "ERROR"

```bash
awk '/ERROR/ { print $0 }' logfile.log
```

This prints only lines with "ERROR" somewhere in them.

## Relational Patterns

You can also match on field values using operators like `==`, `!=`, `>`, `<`.

### Example 3: Match a specific field value

```bash
awk '$3 == "LOGIN" { print $0 }' file.csv
```

This prints lines where the third field is exactly "LOGIN".

## Pattern Ranges

You can specify a range of lines: `pattern1, pattern2 { action }`

```bash
awk '/START/, /END/ { print }' file.txt
```

This prints from a line matching "START" through the next line matching "END".

---

## Your Task

File: `target.txt` (transactions.csv - pipe-delimited)

Fields: `datetime|user|action|resource|code`

Write an AWK command that:

1. In `BEGIN`, print a header: `=== Transaction Report ===`
2. Count how many lines match each action type: LOGIN, LOGOUT, READ, WRITE, DELETE, UPDATE, EXEC
   (Hint: Use `/pattern/` to match each action, or increment a counter when `$3 == "ACTION"`)
3. In `END`, print each action and its count, plus the total
4. Format it neatly

Expected output looks like:

```
=== Transaction Report ===
Action      Count
----------------
LOGIN       374
LOGOUT      397
READ        362
WRITE       374
DELETE      395
UPDATE      376
EXEC        368
(empty)     354
----------------
TOTAL      3000
```
