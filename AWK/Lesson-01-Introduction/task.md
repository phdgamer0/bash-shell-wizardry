# AWK Lesson 01: Introduction

AWK is a powerful text-processing language. It reads input line by line and performs actions on each line.

## Basic Syntax

```
awk 'pattern { action }' filename
```

- `pattern` determines when the action runs (e.g., before processing, after processing, on matching lines)
- `action` is what to do (usually `print`)
- If no pattern is given, the action runs on EVERY line
- If no action is given, the default action is `print` (prints the entire line)

## Fields: $0, $1, $2, ...

AWK automatically splits each line into fields. By default, fields are separated by whitespace (spaces or tabs).

- `$0` = the entire line
- `$1` = first field
- `$2` = second field
- `$3` = third field
- ... and so on

### Example 1: Print entire line

```bash
awk '{ print $0 }' file.txt
```

This prints every line in the file. `{ print $0 }` is the action applied to every line.

### Example 2: Print first field only

```bash
awk '{ print $1 }' file.txt
```

If the line is `hello world from awk`, this prints: `hello`

### Example 3: Print multiple fields

```bash
awk '{ print $1, $3 }' file.txt
```

This prints the first and third fields, separated by the output field separator (default: space).

## The BEGIN Block

Use a `BEGIN` block to run code BEFORE the first line is read. This is useful for setting the field separator.

```bash
awk 'BEGIN { FS="|" } { print $1 }' file.csv
```

`FS` (Field Separator) tells AWK how to split columns. Setting it in `BEGIN` ensures it's set before reading begins.

---

## Your Task

File: `target.txt` (transactions.csv - pipe-delimited)

Format: `datetime|user|action|resource|code`

Write an AWK command that:

1. Sets `FS="|"` in a `BEGIN` block
2. For each line, prints the username (field 2) and action (field 3)
3. Output should be in the format: `user: action`

Hint: `awk 'BEGIN { FS="|" } { print $2": "$3 }' target.txt`

Expected output shows lines like:
```
eve: UPDATE
bob: WRITE
peggy: LOGIN
```
