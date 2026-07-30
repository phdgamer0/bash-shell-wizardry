# Task 18: Capstone — File Organizer CLI Tool

## Objective

Design and implement a complete CLI tool that organizes files by type, date, or size. Must include subcommands, getopts, config file support, colored output, help text, and logging. This task synthesizes argument parsing, library structure, error handling, and professional CLI design patterns.

## Requirements

### Directory Structure

```
fileorg/
├── fileorg.sh           # Main entry point (dispatch)
├── commands/
│   ├── organize.sh      # organize subcommand
│   ├── undo.sh          # undo subcommand
│   ├── status.sh        # status subcommand
│   └── config.sh        # config subcommand
└── lib/
    ├── logging.sh       # Logging functions
    ├── output.sh        # Colored output
    └── config.sh        # Config file loading
```

### Subcommand Specifications

#### 1. `organize` Subcommand

Organizes files in a target directory by type, date, or size.

```
fileorg organize [options]

Options:
  --by type|date|size   Organization strategy (default: type)
  --target DIR          Directory to organize (default: current dir)
  --dry-run             Show what would be done without doing it
  --verbose             Print detailed information
  --no-color            Disable colored output
  --format text|json    Output format (default: text)
```

**By type:** Group files into directories by extension (Documents, Images, Music, Videos, Scripts, Archives, Misc).

**By date:** Group files into directories by modification year/month (e.g., `2026/07/`).

**By size:** Group files into size categories: Small (<1MB), Medium (1-100MB), Large (100MB-1GB), Huge (>1GB).

#### 2. `undo` Subcommand

Reverts the last organization operation using an append-only undo log.

```
fileorg undo [options]

Options:
  --steps N     Undo last N operations (default: 1)
  --list        List recent undoable operations
  --verbose     Print details of restored files
  --force       Skip confirmation prompt
```

#### 3. `status` Subcommand

Shows organization statistics.

```
fileorg status [options]

Options:
  --verbose     Show detailed per-directory breakdown
  --json        Output as JSON
```

Statistics should include: total files organized, total size moved, number of directories created, number of undo steps available, last organization timestamp.

#### 4. `config` Subcommand

View or edit configuration.

```
fileorg config [options]

Options:
  --show        Display current configuration
  --edit        Open config in $EDITOR
  --set KEY=VALUE  Set a configuration value
  --reset       Reset to defaults
```

### Configuration File

Location: `~/.config/fileorg/fileorg.conf` (or `$XDG_CONFIG_HOME/fileorg/fileorg.conf`)

Default contents:

```bash
# fileorg.conf — File Organizer Configuration
ORGANIZE_BY="type"
TARGET_DIR="$HOME/Downloads"
DRY_RUN=false
VERBOSE=false
COLOR=true
LOG_FILE="${XDG_DATA_HOME:-$HOME/.local/share}/fileorg/fileorg.log"
UNDO_LOG="${XDG_DATA_HOME:-$HOME/.local/share}/fileorg/undo.log"
MAX_UNDO_ENTRIES=1000
EXTENSION_MAP="pdf:Documents,doc:Documents,docx:Documents,jpg:Images,jpeg:Images,png:Images,gif:Images,mp3:Music,wav:Music,mp4:Videos,avi:Videos,mkv:Videos,sh:Scripts,py:Scripts,zip:Archives,tar:Archives,gz:Archives"
```

### Logging

- Log to `~/.local/share/fileorg/fileorg.log`
- Format: `[2026-07-31 10:00:00] [INFO] Organized 42 files in /home/user/Downloads`
- Rotate log when it exceeds 5MB
- Log levels: DEBUG, INFO, WARN, ERROR
- Default level: INFO (configurable via `LOG_LEVEL` in config)

### Error Handling

- Validate target directory exists and is readable
- Handle filenames with spaces, newlines, and special characters
- Handle permission errors gracefully (log and continue)
- Exit code 0 on success, 1 on partial failure, 2 on fatal error
- Use `set -euo pipefail` in main.sh
- Custom ERR trap with line number reporting

### Color Output

- Green for success/info, Yellow for warnings, Red for errors
- Detect terminal (`-t 1`) and disable colors for piped output
- Respect `--no-color` flag
- Progress bar during organize operation (only in terminal mode)

## Sub-tasks

### 1. Project Skeleton

Create the directory structure and main entry point with subcommand dispatch. Implement `--help` and `--version` flags at the top level.

### 2. Logging Library

Implement `lib/logging.sh` with:
- `log LEVEL "message"` — log with timestamp and level
- Log level filtering (DEBUG < INFO < WARN < ERROR)
- Automatic log rotation at 5MB
- Log file path from config with fallback

### 3. Output Library

Implement `lib/output.sh` with:
- `info "message"` — green, stdout
- `warn "message"` — yellow, stderr
- `error "message"` — red, stderr
- `success "message"` — green checkmark, stdout
- `progress_bar current total` — interactive progress
- Auto TTY detection for colors and progress bar

### 4. Config Library

Implement `lib/config.sh` with:
- Search paths: `./fileorg.conf`, `~/.config/fileorg/fileorg.conf`, `/etc/fileorg/fileorg.conf`
- `load_config` — source the first found config file
- `get_config KEY` — get a config value with default fallback
- `set_config KEY VALUE` — set a value in the config file
- Config file validation (must be valid Bash syntax)
- Log which config file was loaded

### 5. Organize Subcommand

Implement `commands/organize.sh` with:
- Parse `--by`, `--target`, `--dry-run`, `--verbose`, `--no-color`, `--format`
- Validate target directory
- Read extension map from config
- Classify each file by the chosen strategy
- Create target directories as needed
- Move files with dry-run support
- Record each move in the undo log
- Show progress bar during operation
- Summary at end: "Organized N files (SIZE) into M directories"

### 6. Undo Subcommand

Implement `commands/undo.sh` with:
- Parse `--steps`, `--list`, `--verbose`, `--force`
- Read undo log (append-only, most recent last)
- Move files back to original locations
- Validate target still exists before moving
- Remove used entries from undo log
- Handle conflicts (file already exists at original location)
- Show confirmation prompt unless `--force`

### 7. Status Subcommand

Implement `commands/status.sh` with:
- Count organized files from undo log
- Calculate total size moved
- Count directories created
- Show last organization timestamp
- Show undo log entry count
- Include config file path in output
- Support `--json` for machine-readable output

### 8. Config Subcommand

Implement `commands/config.sh` with:
- `--show` — print current config (with comments)
- `--edit` — open in `$EDITOR` (fallback: vi)
- `--set KEY=VALUE` — update single value
- `--reset` — write default config
- Validate KEY against known config keys

### 9. Integration and Testing

- Test with directories containing diverse file types
- Test with filenames containing spaces and special characters
- Test dry-run vs actual run (counts should match)
- Test undo after organize
- Test log rotation by forcing log size
- Test `--no-color` and piped output
- Test missing config file (should use defaults)

## Expected Output

### Organize (by type, dry run)

```
$ ./fileorg.sh organize --by type --target ~/Downloads --dry-run
[2026-07-31 10:00:00] [INFO] Config loaded from /home/user/.config/fileorg/fileorg.conf
[INFO] Organizing /home/user/Downloads by type (DRY RUN)
[DRY-RUN] report.pdf -> Documents/report.pdf
[DRY-RUN] photo.jpg -> Images/photo.jpg
[DRY-RUN] script.sh -> Scripts/script.sh
[DRY-RUN] song.mp3 -> Music/song.mp3
[DRY-RUN] archive.zip -> Archives/archive.zip
[DRY-RUN] movie.mp4 -> Videos/movie.mp4
[OK] DRY RUN complete: 42 files would be organized (156.3 MB) into 6 directories
```

### Organize (by type, actual)

```
$ ./fileorg.sh organize --by type --target ~/Downloads --verbose
[INFO] Organizing /home/user/Downloads by type
Progress [==================================================] 100%
[INFO] Created directory: /home/user/Downloads/Documents
[INFO] Created directory: /home/user/Downloads/Images
[INFO] Created directory: /home/user/Downloads/Scripts
[INFO] Created directory: /home/user/Downloads/Music
[INFO] Created directory: /home/user/Downloads/Archives
[INFO] Created directory: /home/user/Downloads/Videos
[OK] Organized 42 files (156.3 MB) into 6 directories in 0.34s
```

### Organize (by date)

```
$ ./fileorg.sh organize --by date --target ~/Documents
[INFO] Organizing /home/user/Documents by date
Progress [==================================================] 100%
[OK] Organized 120 files into 8 date-based directories
$ ls ~/Documents
2025/  2026/  2026-01/  2026-07/
```

### Undo

```
$ ./fileorg.sh undo --list
Recent undoable operations:
  1. 2026-07-31 10:00:03 — Organized 42 files (type) in ~/Downloads
  2. 2026-07-30 09:15:22 — Organized 15 files (type) in ~/Desktop

$ ./fileorg.sh undo --steps 1 --verbose
[INFO] Undoing organization from 2026-07-31 10:00:03
[INFO] Restored ~/Downloads/Documents/report.pdf -> ~/Downloads/report.pdf
[INFO] Restored ~/Downloads/Images/photo.jpg -> ~/Downloads/photo.jpg
[INFO] Restored ~/Downloads/Scripts/script.sh -> ~/Downloads/script.sh
[INFO] Restored ~/Downloads/Music/song.mp3 -> ~/Downloads/song.mp3
[INFO] Restored ~/Downloads/Archives/archive.zip -> ~/Downloads/archive.zip
[INFO] Restored ~/Downloads/Videos/movie.mp4 -> ~/Downloads/movie.mp4
[OK] Undo complete: 42 files restored
```

### Status

```
$ ./fileorg.sh status
File Organizer — Status
  Config: /home/user/.config/fileorg/fileorg.conf
  Last organize: 2026-07-31 10:00:03 (42 files, 156.3 MB)
  Total organized: 57 files (201.8 MB)
  Directories created: 8
  Undo steps available: 2
  Log size: 12.4 KB
  Config: OK (all defaults)

$ ./fileorg.sh status --json
{
  "last_organize": "2026-07-31 10:00:03",
  "total_files": 57,
  "total_size": 201800000,
  "directories_created": 8,
  "undo_steps": 2,
  "log_size": 12400,
  "config_status": "ok"
}
```

### Config

```
$ ./fileorg.sh config --show
# fileorg.conf
ORGANIZE_BY="type"
TARGET_DIR="$HOME/Downloads"
DRY_RUN=false
VERBOSE=false
COLOR=true
LOG_FILE="/home/user/.local/share/fileorg/fileorg.log"
UNDO_LOG="/home/user/.local/share/fileorg/undo.log"
MAX_UNDO_ENTRIES=1000

$ ./fileorg.sh config --set ORGANIZE_BY=date
[OK] ORGANIZE_BY set to 'date'

$ ./fileorg.sh config --set MAX_UNDO_ENTRIES=5000
[OK] MAX_UNDO_ENTRIES set to '5000'
```

### Error Handling

```
$ ./fileorg.sh organize --target /nonexistent
[ERROR] Target directory does not exist: /nonexistent

$ ./fileorg.sh unknown
[ERROR] Unknown command: unknown
Usage: fileorg.sh <command> [options]
Try 'fileorg.sh help' for more information.

$ ./fileorg.sh organize --by invalid
[ERROR] Invalid organization strategy: invalid
Valid strategies: type, date, size

$ ./fileorg.sh undo --steps abc
[ERROR] --steps must be a positive integer: abc
```

## Hints

<details>
<summary>Hint 1: Extension Map Parsing</summary>

```bash
# Config stores map as: "pdf:Documents,jpg:Images,mp3:Music"
parse_extension_map() {
  local map_str=${1:-}
  IFS=',' read -ra pairs <<< "$map_str"
  for pair in "${pairs[@]}"; do
    IFS=':' read -r ext dir <<< "$pair"
    EXT_MAP["$ext"]="$dir"
  done
}
```
</details>

<details>
<summary>Hint 2: Safe File Move with Filename Handling</summary>

```bash
safe_move() {
  local src=$1 dst=$2
  if [[ ! -f "$src" ]]; then
    warn "Source not found: $src"
    return 1
  fi
  if [[ -e "$dst" ]]; then
    warn "Destination exists: $dst (skipping)"
    return 1
  fi
  mkdir -p "$(dirname "$dst")"
  mv "$src" "$dst" || {
    error "Failed to move $src to $dst"
    return 1
  }
  return 0
}
```
</details>

<details>
<summary>Hint 3: Date-Based Organization</summary>

```bash
organize_by_date() {
  local file=$1
  local mod_time
  mod_time=$(stat -c '%Y' "$file")
  local year month
  year=$(date -d "@$mod_time" '+%Y')
  month=$(date -d "@$mod_time" '+%m')
  echo "${year}/${month}"
}
```
</details>

<details>
<summary>Hint 4: Size-Based Organization</summary>

```bash
size_category() {
  local file=$1
  local size
  size=$(stat -c '%s' "$file")
  if ((size < 1048576)); then
    echo "Small"
  elif ((size < 104857600)); then
    echo "Medium"
  elif ((size < 1073741824)); then
    echo "Large"
  else
    echo "Huge"
  fi
}
```
</details>

<details>
<summary>Hint 5: Log Rotation</summary>

```bash
rotate_log() {
  local logfile=$1 max_size=${2:-5242880}
  [[ ! -f "$logfile" ]] && return
  local size
  size=$(stat -c '%s' "$logfile")
  if ((size > max_size)); then
    mv "$logfile" "${logfile}.old"
    gzip "${logfile}.old" 2>/dev/null || true
    log info "Log rotated (was ${size} bytes)"
  fi
}
```
</details>

<details>
<summary>Hint 6: Undo Log Format and Parsing</summary>

```bash
# Format: timestamp|strategy|source|destination
# Write:
echo "$(date +%s)|$strategy|$file|$dest" >> "$UNDO_LOG"

# Read:
parse_undo_entry() {
  local line=$1
  IFS='|' read -r ts strategy src dst <<< "$line"
  echo "Strategy: $strategy, From: $src, To: $dst"
}
```
</details>

<details>
<summary>Hint 7: JSON Output Construction</summary>

```bash
json_escape() {
  local str=$1
  str="${str//\\/\\\\}"
  str="${str//\"/\\\"}"
  str="${str//$'\n'/\\n}"
  str="${str//$'\t'/\\t}"
  echo "$str"
}

json_output() {
  local key=$1 value=$2
  printf '  "%s": "%s"' "$(json_escape "$key")" "$(json_escape "$value")"
}
```
</details>

<details>
<summary>Hint 8: TTY Detection for Progress</summary>

```bash
# Only show progress bar if stdout is a terminal
if [[ -t 1 ]]; then
  progress_bar "$current" "$total"
fi

# Alternative: force progress even in pipes with --progress flag
show_progress=${SHOW_PROGRESS:-false}
if $show_progress || [[ -t 1 ]]; then
  progress_bar "$current" "$total"
fi
```
</details>

## Self-Check

1. Why should color output be disabled when stdout is not a terminal?

2. How does `set -euo pipefail` prevent silent failures, and what are its downsides?

3. What is the advantage of an append-only undo log over a single state file?

4. Why should config files be sourced rather than parsed line-by-line?

5. How would you handle filenames with spaces, newlines, or special characters in the organize command?

6. What happens if the user runs organize twice on the same directory without running undo in between?

7. Why is the dry-run mode critical for user trust in a CLI tool?

8. How would you add support for custom extension mappings without editing the config file directly?

9. What security considerations apply when sourcing a user-editable config file?

10. How would you implement a `--watch` mode that continuously monitors a directory and organizes new files automatically?
