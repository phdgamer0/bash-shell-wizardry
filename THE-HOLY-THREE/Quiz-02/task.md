# Quiz 02: IP Report (EASY)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `grep -o <pattern> file | sort | uniq -c | awk '{print ...}'`

## Task

Use `target.txt` (Apache access log).

1. **grep -o**: Extract all IP addresses from the log (hint: `\b` word boundary helps with regex)
2. **sort & uniq -c**: Count occurrences of each IP
3. **awk**: Format output as `IP_ADDRESS: COUNT`, sorted by count descending

**Hint**: Pipe through `sort -rn` after `uniq -c` before awk.
