# GREP Lesson 02: Basic Regular Expressions

## Introduction

In Lesson 1, we used simple fixed strings like `"error"` and `"fail"`. But grep's real power comes from **regular expressions** (regex) — patterns that describe sets of strings.

By default, grep uses **basic regular expressions (BRE)**. Key metacharacters:

| Metacharacter | Meaning                     | Example                | Matches                     |
|---------------|-----------------------------|------------------------|-----------------------------|
| `.`           | Any single character        | `f.ll`                 | fall, fill, f@ll, f3ll      |
| `*`           | Zero or more of preceding   | `ab*c`                 | ac, abc, abbc, abbbc        |
| `^`           | Start of line               | `^test`                | Lines starting with "test"  |
| `$`           | End of line                 | `end$`                 | Lines ending with "end"     |
| `[abc]`       | Any one of a, b, or c       | `gr[ae]y`              | gray, grey                  |
| `[^abc]`      | Any character NOT a,b,c     | `[^0-9]`               | Any non-digit               |
| `\`           | Escape next character       | `\.`                   | Literal period              |

**Important:** In BRE, metacharacters `?`, `+`, `{`, `}`, `(`, `)`, `|` lose their special meaning — you must escape them with `\` to make them special (covered in Lesson 3).

## Examples

### Example 1: Lines starting with a timestamp
Every log line starts with "2026-". Match lines beginning with the year:
```
$ grep -n "^2026" target.txt | head -3
1:2026-07-29T02:24:34+03:00 archlinux systemd-journald[24]: Journal started
2:2026-07-29T02:24:34+03:00 archlinux systemd-journald[24]: Runtime Journal ...
3:2026-07-29T02:24:34+03:00 archlinux systemd[1]: Starting Flush Journal to Persistent Storage...
```

### Example 2: Lines ending with "failed"
The `$` anchor matches the end of a line:
```
$ grep -n "failed$" target.txt
1653:2026-07-29T03:19:17+03:00 archlinux plasmalogin[671]: Authentication for user  ""  failed
21570:2026-07-29T14:09:05+03:00 archlinux NetworkManager[619]: assertion '<dropped>' failed
...
```

### Example 3: Character classes — words with "err" or "warn"
Find lines containing "kernel" using a class for the first letter:
```
$ grep -n "[kw]ernel" target.txt
```
This would match "kernel" but not "Kernel" (but in our log they're all lowercase, so this is just a demonstration).

A more practical example — find lines with a 3-digit number by matching a space then three digits:
```
$ grep -n " [0-9][0-9][0-9] " target.txt
```

### Example 4: Combining anchors
Find lines that start with a timestamp AND contain "failed":
```
$ grep -n "^2026.*failed" target.txt
```

## Task

Find all lines in `target.txt` that **start with "202"** AND **end with "failed"**. Show line numbers.

- *Hint:* Anchor the start with `^` and the end with `$`
- *Hint:* Use `.*` to match anything in between
- *Hint:* Add `-n` for line numbers

### Expected:
16 lines should match. The first one should be line 1653, and the last one should be line 35095.

### Command:
```bash
grep -n "^202.*failed$" target.txt
```
