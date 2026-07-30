# Quiz 01: Find and Count (EASY)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `grep <pattern> file | sed 's/old/new/g' | awk '{action}'`

## Task

Use `target.txt` (warning log entries).

1. **grep**: Find all lines containing "error" or "fail" (case insensitive)
2. **sed**: Replace every occurrence of "error" with "ERROR" and "fail" with "FAIL" (hint: use global flag `/g`)
3. **awk**: Count how many total "ERROR" and "FAIL" tokens appear across all filtered lines

Output format: `ERRORS: N, FAILS: M`

**Hint**: `gawk`'s `gsub()` function returns the number of substitutions made - useful for counting!
