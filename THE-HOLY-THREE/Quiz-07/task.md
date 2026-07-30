# Quiz 07: Security Incident Report (HARD)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: 
```
grep -E 'attack_pattern1|attack_pattern2' file | \
  sed -E 's/normalize/timestamps/' | \
  awk '{build_ip_profile[$1]++} END {generate_report()}'
```

## Task

Use `target.txt` — simulated security logs containing authentication attempts, port scans, and web application attacks.

Stages:
1. **grep**: Identify different attack patterns — failed passwords, UFW block messages, ModSecurity alerts, accepted logins (baseline)
2. **sed**: Normalize the UNIX timestamps at the start of each line into human-readable ISO 8601 format
3. **awk**: Build per-IP threat profiles by counting each type of event per IP. Correlate IPs appearing in multiple attack categories.

Output a comprehensive security report showing:
- TOP 5 most suspicious IPs (highest total threat events)
- Types of attacks detected (with counts)
- Timeline range of the incident

**Hint**: Extract IPs from different field positions depending on the log format. Some IPs appear after "from", others after "SRC=", others right after the timestamp.
