# GREP Lesson 07: Real-World Pattern Analysis

## Introduction

This lesson brings together everything you've learned: basic grep, regex (BRE/ERE/PCRE), context flags, counting, and pipelines. Real-world log analysis rarely uses just one command — you'll build **pipelines** that combine grep with `sort`, `uniq`, `head`, `wc`, and other tools.

## Common Pipeline Patterns

### Pattern 1: Count occurrences per value
```
$ grep "pattern" file.txt | grep -oP 'extract_this' | sort | uniq -c | sort -rn
```

### Pattern 2: Top N results
```
$ [pipeline] | head -N
```

### Pattern 3: Two-section report
```
$ echo "=== Section 1 ===" && [command1] && echo "=== Section 2 ===" && [command2]
```

## Examples

### Example 1: Count requests per IP
```
$ grep -oP '\d+\.\d+\.\d+\.\d+' access.log | sort | uniq -c | sort -rn | head -5
   34 83.149.9.216
   28 66.249.73.135
   24 66.249.84.55
   22 173.252.110.119
   20 180.76.5.114
```

### Example 2: Extract specific data fields
Use `-oP` to extract the request path from HTTP requests:
```
$ grep -oP '"[A-Z]+ \K[^" ]+' access.log | sort | uniq -c | sort -rn | head -5
```

### Example 3: Generate a mini report
```
$ echo "Total requests: $(wc -l < access.log)"
$ echo "404 errors: $(grep -c ' 404 ' access.log)"
$ echo "Unique IPs: $(grep -oP '\d+\.\d+\.\d+\.\d+' access.log | sort -u | wc -l)"
```

## Task

Working with `access.log`, generate a **404 Error Analysis Report** with two sections.

### Part A: Top 5 most frequent 404 paths
1. Find all lines with status 404 (`' 404 '`)
2. Extract the request path (the part after GET/POST and before HTTP/)
   - *Hint:* `grep -oP '"[A-Z]+ \K[^" ]+'`
3. Sort, count unique occurrences, and sort by frequency descending
4. Show the top 5

### Part B: Unique user agents for 404 requests
1. From the same 404 lines, extract the user-agent string
   - *Hint:* User-agent is the last quoted field in the line: `grep -oP '"([^"]*)"$'`
2. Count how many unique user agents generated 404 errors

### Part C: Summary
Calculate total 404 requests and unique 404 paths.

### Expected:
```
---[ Top 5 404 Paths ]---
     61 /files/logstash/logstash-1.3.2-monolithic.jar
     32 /presentations/logstash-puppetconf-2012/images/office-space-printer-beat-down-gif.gif
      6 /wp/wp-admin/
      6 /wp-login.php?action=register
      6 /wp-login.php

---[ 404 User Agents ]---
48 unique user agents

---[ Summary ]---
Total 404 requests: 213
Unique 404 paths: 72
```
