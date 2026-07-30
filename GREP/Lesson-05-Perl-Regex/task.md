# GREP Lesson 05: Perl-Compatible Regular Expressions (PCRE)

## Introduction

Perl-compatible regex (PCRE) adds powerful features not available in BRE or ERE. Use `grep -P` to enable PCRE.

## New Shorthands

| Pattern | Meaning            | Equivalent           |
|---------|--------------------|-----------------------|
| `\d`    | Any digit          | `[0-9]`              |
| `\s`    | Any whitespace     | `[ \t\n\r\f]`        |
| `\w`    | Word character     | `[a-zA-Z0-9_]`       |
| `\b`    | Word boundary      | Position between `\w` and `\W` |

## Lookaround Assertions

| Pattern   | Name             | Example                          | Matches                  |
|-----------|------------------|----------------------------------|--------------------------|
| `(?=...)` | Positive lookahead  | `foo(?=bar)`                   | foo only if followed by bar |
| `(?!...)` | Negative lookahead  | `foo(?!bar)`                   | foo only if NOT followed by bar |
| `(?<=...)`| Positive lookbehind | `(?<=foo)bar`                  | bar only if preceded by foo |
| `(?<!...)`| Negative lookbehind | `(?<!foo)bar`                  | bar only if NOT preceded by foo |

## Examples

### Example 1: Extract IP addresses
```
$ grep -oP '\d+\.\d+\.\d+\.\d+' target.txt | sort -u | head -5
0.255.255.2
0.9832.10273.0
100.2.4.116
100.43.83.137
10.0.648.127
```

### Example 2: Find word boundaries with \b
Search for the word "GET" as a whole word (not "TARGET" or "BUDGET"):
```
$ grep -P '\bGET\b' target.txt | wc -l
9952
```

### Example 3: Lookahead — find "200" only when followed by a digit
```
$ grep -P '200(?=\s)' target.txt | head -3
```

### Example 4: Lookbehind — match the HTTP method after the opening quote
```
$ grep -oP '(?<=")GET ' target.txt | head -3
GET
GET
GET
```

## Task

Working with `access.log`, perform the following analysis:

### Part A: Extract all unique IP addresses
Use `grep -oP` with `\d+\.\d+\.\d+\.\d+` to extract all IPs, then pipe through `sort -u`.

### Part B: Count GET vs POST unique IPs
- Find all lines where the HTTP method is GET (use lookahead/lookbehind or `\b`)
- Extract the unique IPs from those GET lines and count them
- Do the same for POST
- *Hint:* `grep -P '(?<=")GET ' target.txt` matches GET right after an opening quote

### Expected:
- Total unique IPs: 1863
- GET lines: 9952
- POST lines: 5
- Unique IPs that used GET: 1846
- Unique IPs that used POST: 4
