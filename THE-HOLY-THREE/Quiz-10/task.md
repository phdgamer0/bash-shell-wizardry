# Quiz 10: Multi-stage Pipeline (VERY HARD)

**Remember**: You need all THREE tools (grep, sed, awk) working together in creative ways.

**Core pipeline pattern**: `extract | transform | aggregate | report`

This is a capstone challenge combining ALL skills. The data contains deliberate patterns and correlations to discover.

## Task

Use `target.txt` — 3200 lines combining ACCESS, SYSLOG, TRANSACTION, and FIREWALL log entries, each line prefixed by source type.

### Stage 1: Extract by Severity
Use grep to extract events by severity. The data has multiple severity indicators:
- `CRITICAL`, `[ERROR]`, `WARNING` in SYSLOG entries
- `DENY`, `ERROR` in TRANSACTION entries (last pipe-delimited field)
- `DROP` in FIREWALL entries
- 4xx/5xx HTTP status codes in ACCESS entries

Separate each severity class and count them.

### Stage 2: Clean and Normalize
Use sed to:
- Normalize all timestamps (UNIX epoch seconds in ACCESS, SYSLOG, FIREWALL; ISO format in TRANSACTION)
- Remove noise: strip leading source tags (`ACCESS `, `SYSLOG `, `TRANSACTION `, `FIREWALL `)
- Standardize IP address formatting

### Stage 3: Cross-reference and Correlate
Use awk to find related entries across different sources. Look for:
- IP addresses that appear in BOTH SYSLOG failed authentications AND ACCESS log error requests
- Users from TRANSACTION entries whose usernames match suspicious patterns
- Firewall DROP events targeting the same IPs that appear in other logs

### Stage 4: Comprehensive Report
Generate a report showing:
- Total events by source and severity
- Top suspicious IPs (appearing across multiple sources)
- Correlation findings (IPs appearing in 2+ different log types)
- Security summary with recommendations

**Hint**: Check if certain IPs like `10.0.0.99` appear in ALL log types — that's the correlation pattern.
