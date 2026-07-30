# Quiz 03: Clean the Log (EASY)

**Remember**: You need all THREE tools (grep, sed, awk) working together in a pipeline.

**Pipeline example**: `grep -v <pattern> file | sed -E 's/regex/replacement/g' | awk '{action}'`

## Task

Use `target.txt` (system journal log).

1. **grep -v**: Remove all lines containing "systemd"
2. **sed**: Replace all timestamps (pattern: `YYYY-MM-DDTHH:MM:SS+0000`) with just `[TIMESTAMP]` (hint: use extended regex `-E`)
3. **awk**: Count how many lines remain for each service name. The service name is the word after the hostname (usually the 3rd field, e.g., "kernel", "NetworkManager[617]")

**Hint**: Strip trailing colons and PID brackets from the service field before counting.
