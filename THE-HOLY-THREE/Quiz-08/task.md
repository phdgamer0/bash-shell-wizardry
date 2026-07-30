# Quiz 08: Log Rotation & Summary (HARD)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**:
```
grep -c 'source_tag' file                    # count by source
grep 'source_tag' file | sed 'date/normalize' | awk '{hourly_summary}' 
```

## Task

Use `target.txt` — a collection of logs from multiple services, each using different date formats.

Log sources and their date formats:
- **Apache**: `Mon DD HH:MM:SS` (e.g., `Jan 12 10:30:45`)
- **Syslog**: ISO 8601 (`YYYY-MM-DDTHH:MM:SS+00:00`)
- **Application**: `YYYY/MM/DD HH:MM:SS`
- **Postfix**: `Mon DD HH:MM:SS`
- **Custom**: ISO 8601 with `CUSTOM-LOG:` prefix

1. **grep**: Categorize log entries by source (each has unique format markers — IP at start, ISO date, etc.)
2. **sed**: Normalize ALL date formats to ISO 8601 (`YYYY-MM-DDTHH:MM:SS+00:00`)
3. **awk**: Generate an hourly activity summary, grouping events by the hour they occurred

Output: Source breakdown counts + hourly activity distribution (top 10 hours).

**Hint**: Use regex alternation `|` or multiple `-e` flags in sed to handle different date patterns.
