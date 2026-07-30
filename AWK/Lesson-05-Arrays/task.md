# AWK Lesson 05: Associative Arrays

AWK's arrays are **associative** (like dictionaries/maps in other languages). They use strings as keys, not just numbers.

## Creating and Using Arrays

```awk
arr[key] = value
```

### Example 1: Count Frequency

Count how many times each IP appears in a web log:

```bash
awk '{ ip_count[$1]++ } END { for(ip in ip_count) print ip, ip_count[ip] }' access.log
```

- `ip_count[$1]++` creates an entry for each IP and increments it
- `for(ip in ip_count)` iterates over all keys

### Example 2: Multiple Arrays per IP

Track multiple values per key:

```bash
awk '{
    ip = $1
    total[ip]++
    if ($9 == 200) ok[ip]++
    if ($9 == 404) nf[ip]++
}
END {
    for (ip in total) {
        print ip, total[ip], ok[ip]+0, nf[ip]+0
    }
}' access.log
```

Use `+0` to ensure unset values print as 0 instead of blank.

### Example 3: Check if Key Exists

```bash
if ("192.168.1.1" in ip_count) {
    print "Found it!"
}
```

## for (key in array) Loop

The loop `for (key in array)` iterates over all keys. The order is NOT guaranteed.

```awk
for (key in myarray) {
    print key, myarray[key]
}
```

---

## Your Task

File: `target.txt` (access.log)

Write an AWK command that:

1. Build an array `count[ip]` counting how many requests each IP made
2. In `END`, find and print the **top 5** most active IPs (by request count)
3. Then, build a 2D-style summary: for each IP, track total requests, 200 count, 404 count, and 500 count
4. Print a formatted table (use printf):

```
=== Top 5 Most Active IPs ===
1. 66.249.73.135   482 requests
2. 46.105.14.53    364 requests
3. 130.237.218.86  357 requests
4. 75.97.9.59      273 requests
5. 50.16.19.13     113 requests

=== Per-IP Status Summary ===
IP              Total    200      404      500
--------------- -------  -------  -------  -------
66.249.73.135   482      464      1        0
46.105.14.53    364      298      46       0
```

Hint: To find top 5, maintain an array of the 5 highest counts seen so far in the END block.
