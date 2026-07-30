# Quiz 06: Multi-source Data Merge (MEDIUM)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `grep <source_pattern> file | pipeline1 > tmp1; grep <other_pattern> file | pipeline2 > tmp2; cat tmp1 tmp2`

## Task

Use `target.txt` — a merged stream mixing Apache access log lines and CSV transaction lines, interleaved with marker lines.

1. **grep**: Separate the two formats. Access log lines start with an IP address (`^NUM.NUM.NUM.NUM - - `). CSV lines contain pipe `|` characters. Marker lines start with `===` or `---`.
2. **sed**: Extract timestamps from each format. Access logs use Apache date format (`[dd/Mon/YYYY:HH:MM:SS +0000]`). CSV lines have timestamps in field 1.
3. **awk**: Normalize both timestamp formats to a common sortable format, then produce a chronologically ordered merged timeline

Output: A report showing:
- Count of lines from each source
- Sample timestamps from each source
- The chronological range covered

**Hint**: Access logs are from May 2015. CSV transactions span 2024-2025. Think about how to handle different centuries when sorting.
