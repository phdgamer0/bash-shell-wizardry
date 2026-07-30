# Task 6: Git-Style Backup Tool with Subcommands

## Objective
Build a git-style CLI tool called `backup` with subcommands `create`, `list`, and `restore`, full argument parsing with long options, and comprehensive help text. This tool extends the getopts foundation from Lesson 5 into a real-world multi-command utility.

## Requirements

1. **Subcommands:** `backup create`, `backup list`, `backup restore`
2. **Create:** `--name NAME` (required), `--source DIR` (required), `--compress` (flag), `--verbose`
3. **List:** `--all` (show all backups), `--name PATTERN` (filter)
4. **Restore:** `--name NAME` (required), `--target DIR` (required), `--force`
5. **Global:** `--help` at every level
6. **Backup storage:** Backups stored in `~/.backups/` — create the directory if it doesn't exist
7. **Validation:** Prevent overwriting backups, check source exists, validate directory paths

## Sub-tasks (9 cumulative)

### Task 6.1: Main Dispatch
Build the entry point that routes subcommands:

```bash
#!/bin/bash
set -euo pipefail

main() {
  case ${1:-} in
    create|list|restore)
      local cmd="$1"; shift
      "cmd_$cmd" "$@"
      ;;
    --help|-h)
      global_help
      ;;
    *)
      echo "Error: Unknown command '${1:-}'" >&2
      global_help >&2
      exit 1
      ;;
  esac
}

main "$@"
```

**What if variations:**
- `backup` (no args) → shows error with help
- `backup create --help` → should show create-specific help
- `backup --help create` → alternative style (doesn't need to support both)

### Task 6.2: Create Subcommand

```bash
cmd_create() {
  local name="" source="" compress=0 verbose=0
  local args
  args=$(getopt -o n:s:cv -l name:,source:,compress,verbose -n "backup create" -- "$@") || exit 1
  eval set -- "$args"
  
  while true; do
    case $1 in
      -n|--name)     name="$2"; shift 2 ;;
      -s|--source)   source="$2"; shift 2 ;;
      -c|--compress) compress=1; shift ;;
      -v|--verbose)  verbose=1; shift ;;
      --help)        echo "Usage: backup create [options]"; return 0 ;;
      --) shift; break ;;
      *) echo "Internal error" >&2; exit 1 ;;
    esac
  done
  
  # Validate
  [[ -z "$name" ]] && { echo "Error: --name is required" >&2; exit 1; }
  [[ -z "$source" ]] && { echo "Error: --source is required" >&2; exit 1; }
  [[ ! -d "$source" ]] && { echo "Error: Source not found: $source" >&2; exit 1; }
  
  local backup_dir="$HOME/.backups/$name"
  [[ -d "$backup_dir" ]] && { echo "Error: Backup '$name' already exists" >&2; exit 1; }
  
  # Create backup
  mkdir -p "$backup_dir"
  local archive="$backup_dir/data.tar"
  echo "[INFO] Creating backup '$name' from $source"
  
  if ((compress)); then
    tar -czf "$archive.gz" -C "$(dirname "$source")" "$(basename "$source")"
    echo "[INFO] Backup created: $archive.gz"
  else
    tar -cf "$archive" -C "$(dirname "$source")" "$(basename "$source")"
    echo "[INFO] Backup created: $archive"
  fi
}
```

### Task 6.3: List Subcommand

```bash
cmd_list() {
  local all=0 pattern=""
  local args
  args=$(getopt -o a -l all,name: -n "backup list" -- "$@") || exit 1
  eval set -- "$args"
  
  while true; do
    case $1 in
      -a|--all)    all=1; shift ;;
      --name)      pattern="$2"; shift 2 ;;
      --help)      echo "Usage: backup list [options]"; return 0 ;;
      --) shift; break ;;
      *) echo "Internal error" >&2; exit 1 ;;
    esac
  done
  
  local backup_dir="$HOME/.backups"
  if [[ ! -d "$backup_dir" ]]; then
    echo "No backups found."
    return 0
  fi
  
  echo "Available backups:"
  for dir in "$backup_dir"/*/; do
    local name=$(basename "$dir")
    [[ -n "$pattern" && "$name" != *"$pattern"* ]] && continue
    
    local size=$(du -sh "$dir" 2>/dev/null | cut -f1)
    local date=$(stat -c '%y' "$dir" 2>/dev/null | cut -d. -f1)
    echo "  $name  $date  $size"
  done
}
```

### Task 6.4: Restore Subcommand

```bash
cmd_restore() {
  local name="" target="" force=0
  local args
  args=$(getopt -o n:t:f -l name:,target:,force -n "backup restore" -- "$@") || exit 1
  eval set -- "$args"
  
  while true; do
    case $1 in
      -n|--name)   name="$2"; shift 2 ;;
      -t|--target) target="$2"; shift 2 ;;
      -f|--force)  force=1; shift ;;
      --help)      echo "Usage: backup restore [options]"; return 0 ;;
      --) shift; break ;;
      *) echo "Internal error" >&2; exit 1 ;;
    esac
  done
  
  [[ -z "$name" ]] && { echo "Error: --name is required" >&2; exit 1; }
  [[ -z "$target" ]] && { echo "Error: --target is required" >&2; exit 1; }
  
  local backup_dir="$HOME/.backups/$name"
  [[ ! -d "$backup_dir" ]] && { echo "Error: Backup '$name' not found" >&2; exit 1; }
  
  if [[ -d "$target" ]] && ((!force)); then
    echo "Error: Target exists. Use --force to overwrite" >&2
    exit 1
  fi
  
  mkdir -p "$target"
  if [[ -f "$backup_dir/data.tar.gz" ]]; then
    tar -xzf "$backup_dir/data.tar.gz" -C "$target"
  elif [[ -f "$backup_dir/data.tar" ]]; then
    tar -xf "$backup_dir/data.tar" -C "$target"
  else
    echo "Error: No backup archive found" >&2
    exit 1
  fi
  echo "[INFO] Restored '$name' to $target"
}
```

### Task 6.5: Global and Subcommand Help

```bash
global_help() {
  cat <<'EOF'
Usage: backup <command> [options]

Backup management tool

Commands:
  create   Create a new backup
  list     List existing backups
  restore  Restore from a backup

Options:
  --help   Show help for any command

Run 'backup <command> --help' for command-specific help.
EOF
}

# Add help to each subcommand:
# backup create --help  → show_create_help()
# backup list --help    → show_list_help()
# backup restore --help → show_restore_help()
```

### Task 6.6: Backup Naming Convention

```bash
# Validate backup name (allow only safe characters)
validate_name() {
  local name="$1"
  case "$name" in
    *[!a-zA-Z0-9_-]*) 
      echo "Error: Backup name can only contain letters, numbers, hyphens, underscores" >&2
      exit 1
      ;;
  esac
}
```

### Task 6.7: Dry-Run Mode

Add `--dry-run` to all subcommands:

```bash
cmd_create() {
  local dry_run=0
  # ... parse options, add:
  --dry-run) dry_run=1; shift ;;
  
  if ((dry_run)); then
    echo "[DRY-RUN] Would create backup '$name' from $source"
    ((compress)) && echo "[DRY-RUN] Compression enabled"
    return 0
  fi
  # ... actual backup logic
}
```

### Task 6.8: Backup Metadata

Save metadata alongside the backup:

```bash
save_metadata() {
  local backup_dir="$1"
  local source="$2"
  cat > "$backup_dir/META" << EOF
name=$name
source=$source
created=$(date -Iseconds)
host=$(hostname)
user=$USER
version=1.0
EOF
}
```

### Task 6.9: Integration Test Suite

```bash
test_backup() {
  local test_dir=$(mktemp -d)
  local restore_dir=$(mktemp -d)
  trap "rm -rf $test_dir $restore_dir" EXIT
  
  # Create test data
  echo "test data" > "$test_dir/file1.txt"
  mkdir "$test_dir/subdir"
  echo "nested" > "$test_dir/subdir/file2.txt"
  
  echo "=== Test: Create backup ==="
  ./backup.sh create --name test1 --source "$test_dir" --compress
  [[ -f "$HOME/.backups/test1/data.tar.gz" ]] && echo "PASS" || echo "FAIL"
  
  echo "=== Test: List backups ==="
  ./backup.sh list --all | grep -q test1 && echo "PASS" || echo "FAIL"
  
  echo "=== Test: Restore backup ==="
  ./backup.sh restore --name test1 --target "$restore_dir"
  [[ -f "$restore_dir/file1.txt" ]] && echo "PASS" || echo "FAIL"
  
  echo "=== Test: Duplicate backup ==="
  ./backup.sh create --name test1 --source "$test_dir" 2>&1 | grep -q "already exists" && echo "PASS" || echo "FAIL"
  
  echo "=== Test: Missing name ==="
  ./backup.sh create --source "$test_dir" 2>&1 | grep -q "required" && echo "PASS" || echo "FAIL"
}
```

## Bonus Challenges

1. **Bonus A:** Add incremental backups — only tar files changed since last backup (use `tar --newer`).

2. **Bonus B:** Implement a `--remote` option that pushes backups to a remote server via `rsync`.

3. **Bonus C:** Add encryption with GPG: `--encrypt KEYID` encrypts the archive before storing.

4. **Bonus D:** Implement backup pruning: `backup prune --keep 7 — daily, weekly, monthly rotation.

5. **Bonus E:** Add a `--format` option to `list`: `--format json` outputs machine-readable JSON.

## Hints

<details>
<summary>Hint 1: getopt short options</summary>

```bash
args=$(getopt -o n:s:cv -l name:,source:,compress,verbose -n "backup create" -- "$@")
# -n requires arg, -s requires arg, -c and -v are flags
```
</details>

<details>
<summary>Hint 2: eval set -- must be after getopt</summary>

Always check getopt exit code before eval:
```bash
args=$(getopt ...) || exit 1
eval set -- "$args"
```
</details>

<details>
<summary>Hint 3: Subcommand function naming</summary>

Use `cmd_create`, `cmd_list`, `cmd_restore` pattern. The dispatch is simply:
```bash
"cmd_$cmd" "$@"
```
</details>

<details>
<summary>Hint 4: Home directory expansion</summary>

`~/.backups` doesn't expand in all contexts. Use `$HOME/.backups` instead.
</details>

<details>
<summary>Hint 5: getopt --help is not automatic</summary>

getopt does NOT add `--help` to your options. You must add `--help` to the long option list and handle it in your case statement.
</details>

## Expected Output

```bash
$ ./backup.sh --help
Usage: backup <command> [options]
Commands:
  create   Create a new backup
  list     List existing backups
  restore  Restore from a backup

$ ./backup.sh create --help
Usage: backup create [options]
  --name NAME     Backup name (required)
  --source DIR    Source directory (required)
  --compress      Enable compression
  --verbose       Verbose output
  --dry-run       Show what would be done

$ ./backup.sh create --name mydata --source /home/user/docs --compress
[INFO] Creating backup 'mydata' from /home/user/docs
[INFO] Backup created: /home/user/.backups/mydata/data.tar.gz

$ ./backup.sh list --all
Available backups:
  mydata    2026-07-31 10:00:00  1.2M
  oldstuff  2026-07-30 09:00:00  89K

$ ./backup.sh list --name mydata
Available backups:
  mydata    2026-07-31 10:00:00  1.2M

$ ./backup.sh restore --name mydata --target /tmp/restore
[INFO] Restored 'mydata' to /tmp/restore

$ ./backup.sh restore --name nonexistent --target /tmp/x
Error: Backup 'nonexistent' not found

$ ./backup.sh create --name mydata --source /home/user/docs
Error: Backup 'mydata' already exists
```

## Deep Self-Check

1. **getopt quoting test:** Create a file with spaces in the source path. Does the backup tool handle it? Test with `--source "/home/user/My Documents"`.

2. **Subcommand argument bleeding:** What happens if you run `backup create --name foo` without `--source`? Trace the argument flow through the dispatch and into `cmd_create`.

3. **Symlink handling:** If the source directory contains symlinks, does `tar` follow them by default? What flags control this?

4. **`set -e` and getopt:** If `set -e` is active and `getopt` fails, does the script exit before checking the exit code? Test with `set -e` and an invalid option.

5. **Backup storage race condition:** What happens if two instances of `backup create --name same` run simultaneously? Can both proceed?

6. **Large backup performance:** Time how long `backup create --source /usr --compress` takes. Where is the bottleneck (tar, gzip, disk)?

7. **PATH security:** If a malicious `getopt` is placed in a writable PATH directory, can it compromise the backup tool? How does the `eval set -- "$args"` pattern make this worse?

8. **Metadata file format:** The META file uses `key=value` format. What would happen if a key or value contains `=`? How does `git config` handle this?

9. **`--dry-run` consistency:** Does your dry-run mode use the same validation code as the real run? If not, dry-run may report success when the real run would fail.

10. **Exit code convention:** Check the exit codes for each error condition. Do they follow the sysexits convention (EX_USAGE=64, EX_DATAERR=65, etc.)?
