# Quiz 05: HTTP Analysis (MEDIUM)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `awk '{extract fields}' file | grep <pattern> | sed 's/anonymize/' | awk '{aggregate}'`

## Task

Use `target.txt` (Apache access log with mixed GET and POST requests).

1. **awk**: Extract fields: IP (field 1), method (field 6, strip quotes), path (field 7), status (field 9), bytes (field 10)
2. **grep**: Filter to only POST requests with 4xx status codes (400-499)
3. **sed**: Anonymize IPs by replacing the last octet with XXX (e.g., `192.168.1.50` -> `192.168.1.XXX`)
4. **awk**: Count unique paths among the filtered results

Output: List each unique path, then "UNIQUE PATHS: N"

**Hint**: Use `substr($6, 2)` to strip the leading quote from the method field.
