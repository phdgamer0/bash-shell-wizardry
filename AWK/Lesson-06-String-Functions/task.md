# AWK Lesson 06: String Functions

AWK has many built-in string functions for text manipulation.

## Common String Functions

| Function    | Description |
|-------------|-------------|
| `length(s)` | Returns the number of characters in string `s` |
| `substr(s, p, n)` | Returns substring of `s` starting at position `p`, length `n` |
| `index(s, t)` | Returns position of substring `t` in `s`, or 0 if not found |
| `split(s, arr, sep)` | Splits string `s` into array `arr` using separator `sep` |
| `gsub(r, t, s)` | Globally substitutes regex `r` with `t` in string `s` |
| `match(s, r)` | Returns position where regex `r` matches in `s`, or 0 |
| `tolower(s)` | Converts string to lowercase |
| `toupper(s)` | Converts string to uppercase |

### Example 1: Extract parts of a date

Given a date like `2024-07-14 00:00:00`:

```bash
awk '{
    split($1, dt_parts, " ")
    split(dt_parts[1], date_parts, "-")
    year = date_parts[1]
    month = date_parts[2]
    day = date_parts[3]
    print "Year:", year, "Month:", month, "Day:", day
}' file.csv
```

### Example 2: Convert to uppercase

```bash
awk '{ print toupper($2) }' file.csv
```

### Example 3: Find length of each field

```bash
awk '{ for(i=1; i<=NF; i++) print "Field", i, "length:", length($i) }' file.csv
```

## split() Details

`split(string, array, separator)` splits the string based on the separator.

```bash
str = "2024-07-14 00:00:00"
split(str, parts, " ")
split(parts[1], date, "-")
# date[1] = "2024", date[2] = "07", date[3] = "14"
```

---

## Your Task

File: `target.txt` (transactions.csv - pipe-delimited)

Fields: `datetime|user|action|resource|code`

Write an AWK command that:

1. From the `datetime` field (field 1, format `YYYY-MM-DD HH:MM:SS`), extract the **month** using `split()` or `substr()`
2. Count how many transactions occurred in each month
3. Convert all usernames to uppercase using `toupper()`
4. In `END`, print each month number and its count, and all uppercase usernames

Expected output format:

```
=== Transactions per Month ===
Month 01: 353
Month 02: 308
Month 03: 323
Month 04: 318
Month 05: 315
Month 06: 322
Month 07: 190
Month 08: 167
Month 09: 161
Month 10: 172
Month 11: 202
Month 12: 169

=== Usernames (Uppercase) ===
ALICE
BOB
CHARLIE
DAVE
EVE
FRANK
GRACE
HEIDI
IVAN
JUDY
MALLORY
OSCAR
PEGGY
TRENT
```
