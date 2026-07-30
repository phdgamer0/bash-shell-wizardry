# Task 9: String Manipulation — Internals vs Externals

## Overview

Write a script `parse_logs.sh` that processes Apache access logs efficiently, using bash internals where possible and external tools where necessary. The goal is to understand the performance tradeoffs and learn when to use each approach.

## Sub-tasks

### 1. Efficient IP Extractor (bash internals)

Read a log file line by line without forking per line. Extract the IP using bash parameter expansion. Count unique IPs with an associative array.

```bash
#!/bin/bash
LOG_FILE="${1:-/dev/stdin}"
declare -A ip_count

while IFS= read -r line; do
    ip="${line%% *}"
    ((ip_count[$ip]++))
done < "$LOG_FILE"

echo "=== Unique IPs ==="
for ip in "${!ip_count[@]}"; do
    printf "%-20s %d\n" "$ip" "${ip_count[$ip]}"
done | sort -k2 -rn
```

**Step-by-step:**
1. `while IFS= read -r line` — safe line-by-line reading (no whitespace trimming, no backslash interpretation).
2. `ip="${line%% *}"` — parameter expansion: remove everything from the first space onward, leaving only the IP (first field).
3. `((ip_count[$ip]++))` — associative array increment for each IP.
4. `sort -k2 -rn` — sort by second field (count) numerically descending.

**What if the log format has leading spaces?** `"${line%% *}"` would return an empty string. Use `"${line## }"` first to strip leading spaces, or use awk.

<details>
<summary>Hint: Why IFS= read -r?</summary>
- `IFS=` prevents trimming leading/trailing whitespace
- `-r` prevents backslash interpretation
- Without these, `read` would strip whitespace and interpret `\n`, `\t`, etc.
</details>

### 2. Status Code Breakdown (awk)

Extract the HTTP status code (9th field in Apache combined log format) using `awk` in a single pass. Report counts per status code.

```bash
echo "=== Status Code Breakdown ==="
awk '{
    status[$9]++
}
END {
    for (s in status) printf "%3d: %d\n", s, status[s]
}' "$LOG_FILE" | sort -k1 -n
```

**Apache log format reference:**
```
192.168.1.1 - - [30/Jul/2025:12:34:56 +0000] "GET /index.html HTTP/1.1" 200 1234
```
- `$1` = IP
- `$2` = identd
- `$3` = userid
- `$4` `$5` = timestamp (bracketed, two fields)
- `$6` = request method (e.g., "GET)
- `$7` = URL path
- `$8` = protocol (HTTP/1.1")
- `$9` = status code
- `$10` = bytes sent

Wait — the request "GET /index.html HTTP/1.1" is quoted as one field? Actually, in Apache combined format, the request is `"GET /index.html HTTP/1.1"` which awk sees as three fields because it's quoted. So the status code is `$9`.

<details>
<summary>Hint: Verifying field numbers</summary>
Run this to see the field numbering:
```bash
awk '{for (i=1; i<=NF; i++) print i": "$i}' access.log | head -20
```
</details>

### 3. URL Path Histogram (awk)

Extract the request path (7th field, e.g. `/index.html`) using `awk` and produce a frequency-sorted list of the top 15 most-requested paths.

```bash
echo "=== Top 15 URLs ==="
awk '{
    path=$7
    count[path]++
}
END {
    for (p in count) print count[p], p
}' "$LOG_FILE" | sort -rn | head -15
```

**Alternative — extract relative path only:**
```bash
awk '{
    path=$7
    gsub(/^.*\/\/[^\/]+/, "", path)  # remove domain if present
    count[path]++
}
...
```

<details>
<summary>Hint: Handling query strings in paths</summary>
Apache logs show only the path, not the full URL with query string for the request line. But sometimes you'll see `/path?query`. To strip query strings in awk:
```bash
awk '{
    path=$7
    sub(/\?.*/, "", path)
    count[path]++
}'
```
</details>

### 4. Performance Comparison

Write a version that uses only bash internals for everything and compare runtime with the awk-based version.

```bash
#!/bin/bash
# bash_internal_version.sh
LOG_FILE="$1"
declare -A ip_count status_count url_count

while IFS= read -r line; do
    # IP is first field
    ip="${line%% *}"
    ((ip_count[$ip]++))

    # Request is in quotes — extract it
    rest="${line#*\"}"
    request="${rest%%\"*}"
    # Path is second word in request
    read -r _ path _ <<< "$request"
    ((url_count[$path]++))

    # Status code — find 3-digit code after request
    after_request="${rest#*\" }"
    status="${after_request%% *}"
    ((status_count[$status]++))
done < "$LOG_FILE"

echo "=== Unique IPs ==="
for ip in "${!ip_count[@]}"; do echo "${ip_count[$ip]} $ip"; done | sort -rn

echo "=== Status Codes ==="
for s in "${!status_count[@]}"; do echo "${status_count[$s]} $s"; done | sort -rn

echo "=== Top URLs ==="
for u in "${!url_count[@]}"; do echo "${url_count[$u]} $u"; done | sort -rn | head -15
```

**Comparison runner:**
```bash
#!/bin/bash
LOG_FILE="$1"
echo "=== Performance ==="
TIMEFORMAT='Bash internal version: %3R seconds'
time {
    bash bash_internal_version.sh "$LOG_FILE" > /dev/null
}

TIMEFORMAT='Awk version: %3R seconds'
time {
    awk '{
        ip=$1; status=$9; path=$7
        ipc[ip]++; sc[status]++; pc[path]++
    } END {
        for (i in ipc) print i, ipc[i]
        for (s in sc) print s, sc[s]
        for (p in pc) print p, pc[p]
    }' "$LOG_FILE" > /dev/null
}
```

<details>
<summary>Hint: Generating a test log file</summary>
```bash
generate_logs() {
    local count=${1:-10000}
    local ips=(192.168.1.{1..10} 10.0.0.{1..5})
    local paths=(/index.html /about.html /contact.html /blog/ /products/ /services/ /login /admin /api/v1/ /api/v2/)
    local statuses=(200 200 200 200 200 200 200 200 304 404 404 500 301)
    for ((i=0; i<count; i++)); do
        local ip=${ips[$RANDOM % ${#ips[@]}]}
        local path=${paths[$RANDOM % ${#paths[@]}]}
        local status=${statuses[$RANDOM % ${#statuses[@]}]}
        local bytes=$((RANDOM % 10000 + 100))
        printf '%s - - [30/Jul/2025:12:%02d:%02d +0000] "GET %s HTTP/1.1" %s %d\n' \
            "$ip" $((RANDOM % 60)) $((RANDOM % 60)) "$path" "$status" "$bytes"
    done
}
generate_logs 10000 > test_access.log
```
</details>

### 5. Bonus: Time-based Histogram

```bash
#!/bin/bash
echo "=== Requests by Hour ==="
awk '{
    # Extract hour from [30/Jul/2025:12:34:56
    match($4, /:([0-9]{2}):/, arr)
    hour = arr[1] + 0  # remove leading zero
    hour_count[hour]++
}
END {
    for (h=0; h<24; h++) {
        bar = ""
        for (i=0; i<hour_count[h]/5; i++) bar = bar "#"
        printf "%02d: %4d %s\n", h, hour_count[h], bar
    }
}' "$LOG_FILE"
```

### 6. Bonus: Combined Analysis with Process Substitution

```bash
#!/bin/bash
LOG_FILE="$1"

echo "=== Parallel Analysis ==="

# Run three analyses in parallel using process substitution
# This is faster than running them sequentially for large files
echo "Total requests: $(wc -l < "$LOG_FILE")"

echo "Most active IPs:"
awk '{print $1}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -5

echo "Busiest hour:"
awk 'match($4, /:([0-9]{2}):/, a) {print a[1]}' "$LOG_FILE" | sort | uniq -c | sort -rn | head -1

echo "Error rate (4xx/5xx):"
awk '$9 ~ /^[45]/ {errors++} END {print errors, "errors"}' "$LOG_FILE"
```

## Expected Output

```
$ ./parse_logs.sh access.log

=== Unique IPs ===
192.168.1.1             42
192.168.1.2             15
10.0.0.1                 8

=== Status Code Breakdown ===
200: 55
404: 7
500: 3
301: 2
304: 1

=== Top 15 URLs ===
15 /index.html
12 /about.html
8 /contact.html
5 /products/
3 /services/
2 /login

=== Performance ===
Bash internal version: 0.45s
Awk version: 0.08s
```

## Self-Check

- Why is calling `echo $line | cut -d' ' -f1` inside a loop slow?
- When would you choose `awk` over bash parameter expansion?
- What does `grep -P` offer that `grep -E` doesn't?
- Why is `while IFS= read -r line` the correct way to read lines?
- What's the fastest way to extract the Nth field from every line of a 1GB file?
- How does the performance of bash `while read` compare to `awk` on large files?
- When would you use `grep -q` instead of parsing the full file?
