# Quiz 04: User Activity Report (MEDIUM)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `grep <pattern> file | sed 's/delimiter/\t/g' | awk -F'\t' '{...}'`

## Task

Use `target.txt` (pipe-delimited transaction log). Columns: `timestamp|user|action|target|result`

1. **grep**: Find all lines with "ERROR" or "DENY" result codes
2. **sed**: Replace the pipe `|` delimiters with tab characters `\t`
3. **awk**: Count per-user totals, producing a report sorted alphabetically by username

Output format: `USERNAME | ERROR_COUNT | DENY_COUNT`

**Hint**: Use `-F'\t'` in awk to set tab as field separator. Use arrays to accumulate counts per user.
