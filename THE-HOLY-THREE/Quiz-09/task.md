# Quiz 09: Performance Analyzer (HARD)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `awk '{aggregate_by_path}' file | sort -t'=' -k6 -rn | head -20`

## Task

Use `target.txt` — full Apache access log (10000 lines).

Calculate response size statistics per URL path:

1. **grep**: No direct filtering needed here — but consider separating static content (images, css, js) from dynamic content (pages, api endpoints)
2. **sed**: Not essential for the core task, but consider extracting just the path from the request line
3. **awk**: For each unique URL path, compute: count, min bytes, max bytes, average bytes, total bytes

Output: One line per path, sorted by total bytes descending, formatted as:
```
/path | COUNT=N | MIN=N | MAX=N | AVG=N | TOTAL=N
```

**Hint**: Field 7 is the path, field 10 is the byte count. Use arrays keyed by path name. Handle zero-byte responses (e.g., `robots.txt` is requested 180 times with 0 bytes).
