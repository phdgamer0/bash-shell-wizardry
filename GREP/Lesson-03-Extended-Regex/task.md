# GREP Lesson 03: Extended Regular Expressions (ERE)

## Introduction

Basic regex (BRE) is powerful, but it has limitations. For example, `|` (alternation) and `+` (one or more) need to be escaped with `\`. With **extended regex (ERE)**, these metacharacters work without escaping.

Use `grep -E` (or the older `egrep` command) to activate ERE.

| Metacharacter | Meaning                     | Example                   | Matches                      |
|---------------|-----------------------------|---------------------------|------------------------------|
| `+`           | One or more of preceding    | `ab+c`                    | abc, abbc, abbbc (not ac)    |
| `?`           | Zero or one of preceding    | `ab?c`                    | ac, abc (not abbc)           |
| `{n,m}`       | Between n and m repetitions | `a{2,4}`                  | aa, aaa, aaaa                |
| `|`           | Alternation (OR)            | `cat\|dog`                 | cat or dog                   |
| `()`          | Grouping                    | `(foo)\|(bar)`             | foo or bar                   |

## Examples (using `access.log` as target)

### Example 1: Match 200 OR 304 status codes
```
$ grep -E " 200 | 304 " target.txt | head -3
83.149.9.216 - - [17/May/2015:10:05:03 +0000] "GET ..." 200 203023 "..." "..."
83.149.9.216 - - [17/May/2015:10:05:43 +0000] "GET ..." 200 171717 "..." "..."
83.149.9.216 - - [17/May/2015:10:05:47 +0000] "GET ..." 200 26185 "..." "..."
```

### Example 2: Find image requests (png, jpg, gif)
```
$ grep -E '\.(png|jpg|gif)' target.txt | head -3
83.149.9.216 - - [17/May/2015:10:05:03 +0000] "GET /presentations/.../kibana-search.png HTTP/1.1" 200 203023 "..." "..."
83.149.9.216 - - [17/May/2015:10:05:43 +0000] "GET /presentations/.../kibana-dashboard3.png HTTP/1.1" 200 171717 "..." "..."
111.199.235.239 - - [17/May/2015:13:05:25 +0000] "GET /presentations/.../office-space-printer-beat-down-gif.gif HTTP/1.1" 404 364 "..." "..."
```

### Example 3: Status codes that are exactly 3 digits
Using `{3}` for repetition:
```
$ grep -E '" [0-9]{3} ' target.txt | head -3
```
This matches lines where the HTTP response code is exactly 3 digits.

## Task

In this task, you'll work with `access.log` (Apache combined format). Your goal is to analyze HTTP status codes.

### Part A: Count successful vs client errors
- **Successful responses**: Count all lines where the status code is between 200-399 (starts with 2 or 3)
- **Client errors**: Count all lines where the status code is between 400-499
- *Hint:* Use `grep -cE` with `[2-3][0-9][0-9]` and `4[0-9][0-9]`

### Part B: Find presentations/projects with 200 responses
Find all lines where the request path contains either `presentations` or `projects` **AND** the response status is exactly `200`.

- *Hint:* Combine `grep -E` with alternation for paths, then pipe to another `grep` for the status
- *Hint:* `grep -E '/presentations|/projects' target.txt | grep ' 200 '`

### Expected:
- Successful count: 9780
- Client error count: 217
- Lines matching path+200: 3160
