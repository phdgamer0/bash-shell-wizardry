# GREP Lesson 04: Context Lines & Special Flags

## Introduction

Sometimes you need more than just the matching line — you want surrounding context to understand what happened. Grep provides several flags for this, plus other handy options.

## Context Flags

| Flag | Meaning                        | Example           |
|------|--------------------------------|-------------------|
| `-A N` | Show N lines **A**fter match  | `-A 2`            |
| `-B N` | Show N lines **B**efore match | `-B 1`            |
| `-C N` | Show N lines **C**ontext (before + after) | `-C 3` |

## Other Useful Flags

| Flag | Meaning                                |
|------|----------------------------------------|
| `-o` | Show **only** the matching part of the line |
| `-l` | List only filenames with matches       |
| `-L` | List only filenames **without** matches |
| `-r` | Recursive search through directories   |

## Examples

### Example 1: Context around "error" matches
```
$ grep -i -B 1 -A 1 "error" target.txt | head -15
```
This shows 1 line before and 1 line after every match containing "error", with `--` separating each group.

### Example 2: Extract only IPs from access.log
Using `-o` with a regex that matches IPs:
```
$ grep -oP '\d+\.\d+\.\d+\.\d+' access.log | sort -u
```
(This uses Perl regex `\d` for digits — more on that in Lesson 5.)

### Example 3: Combine -v with -c (invert + count)
Count lines that do NOT contain "systemd":
```
$ grep -vc "systemd" target.txt
```

## Task

Working with `warnings.log`, your task has two parts:

### Part A: Context search
Find all lines that match either "systemd" OR "kernel". Show 1 line before and 1 line after each match.

*Hint:* `grep -E 'systemd|kernel' -B 1 -A 1 target.txt`

### Part B: Extract service names
From those matches, extract **only the service name** using `-o`. The service name appears in square brackets like `[907]` or `[613]`.

*Hint:* Use a pattern like `\[[0-9]+\]` (extended regex) to match the bracketed numbers
*Hint:* To extract just the text inside brackets (without the brackets themselves), try: `grep -oP '\[\K[^\]]+'` (Perl regex lookbehind)

### Expected:
- 253 matching lines for systemd/kernel
- Context output will show those lines with 1 line before/after
- Service names extracted include: 1, 613, 907, 893, 895, 615, 612, 1022, etc.
