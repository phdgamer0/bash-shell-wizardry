# Lesson 7: `find` Deep Dive

## History & Origins

The `find` command dates back to **Version 1 Unix** (1971), making it one of the oldest Unix utilities. It was written by **Ken Thompson** and later standardized by POSIX. The original `find` simply walked directory trees; the expression syntax with `-name`, `-type`, `-size` grew over successive Unix versions.

The `-exec` flag was a revolutionary addition — it let you run commands on found files without piping, avoiding the pitfalls of shell interpretation. The `\+` variant (batching) came from BSD and was adopted by POSIX.

Linux's `find` (from **GNU findutils**) adds many extensions over POSIX: `-maxdepth`, `-mindepth`, `-printf`, `-delete`, `-print0`, and regex matching with `-regex` and `-iregex`.

Why does `find` exist instead of just `ls -R | grep`? Because filenames can contain newlines, spaces, and other special characters. `find` outputs filenames **literally** and with `-print0` outputs them NUL-delimited — the only safe way to handle arbitrary filenames.

## Syntax Reference

### Basic Form

```
find [paths...] [expression] [action]
```

- `paths`: Directories to search (default: current directory `.`)
- `expression`: Tests (filters) combined with boolean operators
- `action`: What to do with matches (default: `-print`)

### Tests (Filters)

**Name-based**:
| Test | Meaning |
|------|---------|
| `-name pattern` | Filename matches glob (case-sensitive) |
| `-iname pattern` | Filename matches glob (case-insensitive) |
| `-path pattern` | Full path matches glob |
| `-ipath pattern` | Full path matches glob (case-insensitive) |
| `-regex pattern` | Full path matches regex (GNU) |
| `-iregex pattern` | Full path matches regex, case-insensitive (GNU) |

**Type-based**:
| Test | Meaning |
|------|---------|
| `-type f` | Regular file |
| `-type d` | Directory |
| `-type l` | Symbolic link |
| `-type p` | Named pipe (FIFO) |
| `-type s` | Socket |
| `-type b` | Block device |
| `-type c` | Character device |

**Size-based**:
| Test | Meaning |
|------|---------|
| `-size 100c` | Exactly 100 bytes |
| `-size +100M` | Larger than 100 MB |
| `-size -1G` | Smaller than 1 GB |
| `-size 1024k` | Exactly 1024 KB |

Suffixes: `c` (bytes), `w` (2-byte words), `b` (512-byte blocks — default), `k` (KB), `M` (MB), `G` (GB).

**Time-based**:
| Test | Meaning |
|------|---------|
| `-atime N` | Last access N days ago |
| `-amin N` | Last access N minutes ago |
| `-mtime N` | Last modification N days ago |
| `-mmin N` | Last modification N minutes ago |
| `-ctime N` | Last status change N days ago |
| `-cmin N` | Last status change N minutes ago |
| `-newer file` | Modified more recently than `file` |
| `-anewer file` | Accessed more recently than `file` |
| `-cnewer file` | Changed more recently than `file` |

For `N`:
- `+N` = greater than N
- `-N` = less than N
- `N` = exactly N (rounded up to next 24-hour period for `*time`)

**Permission-based**:
| Test | Meaning |
|------|---------|
| `-perm 644` | Exact permissions 644 |
| `-perm -644` | ALL of these bits set |
| `-perm /644` | ANY of these bits set |
| `-perm /4000` | SUID bit set |
| `-perm /2000` | SGID bit set |
| `-perm /1000` | Sticky bit set |

**Ownership-based**:
| Test | Meaning |
|------|---------|
| `-user name` | Owned by user |
| `-group name` | Owned by group |
| `-nouser` | No user (orphaned file) |
| `-nogroup` | No group (orphaned) |

**Miscellaneous**:
| Test | Meaning |
|------|---------|
| `-maxdepth N` | Descend at most N levels (GNU) |
| `-mindepth N` | Don't act on files above N levels (GNU) |
| `-prune` | Don't descend into matching directories |
| `-empty` | File is empty (0 bytes, or empty directory) |
| `-readable` | File is readable |
| `-writable` | File is writable |
| `-executable` | File is executable |

### Boolean Operators

| Syntax | Meaning | Example |
|--------|---------|---------|
| `expr1 expr2` | AND (implicit) | `-type f -name "*.txt"` |
| `-a` | AND (explicit) | `-type f -a -name "*.txt"` |
| `-o` | OR | `-name "*.txt" -o -name "*.md"` |
| `! expr` | NOT | `! -name "*.txt"` |
| `\( expr \)` | Grouping | `\( -type f -o -type d \)` |

**Short-circuit evaluation**: find stops evaluating tests as soon as the result is determined (like `&&`/`||` in bash). This is important for `-prune -o` patterns.

### Actions

| Action | Meaning |
|--------|---------|
| `-print` | Print full path (default) |
| `-ls` | Like `ls -dils` format |
| `-delete` | Delete matched files (implies `-depth`) |
| `-exec cmd {} \;` | Run cmd on each file (one at a time) |
| `-exec cmd {} +` | Run cmd on batches of files |
| `-execdir cmd {} \;` | Run cmd from file's directory |
| `-ok cmd {} \;` | Like `-exec` but prompts before each |
| `-printf format` | Custom output format (GNU) |

### `-exec` Deep Dive

**`-exec cmd {} \;`**: For each matched file, executes `cmd` with `{}` replaced by the filename.
- `;` must be escaped (`\;` or `';'`)
- `{}` can appear multiple times: `-exec cp {} {}.bak \;`
- `{}` must be a separate argument (not embedded in another word) for `-exec ... +`

**`-exec cmd {} +`**: Batches filenames and runs cmd once per batch.
- `{}` must appear at the end of the command (just before `+`)
- Much faster than `\;` for large numbers of files
- The command is called as few times as possible (but at least once if there are matches)

## Under the Hood

### How `find` works

1. **`ftw()` / `nftw()`** — find uses the POSIX `nftw()` (new file tree walk) function internally on most systems. This is a C library function that recursively walks a directory tree.
2. **Directory reading**: At each directory, `find` calls `opendir()`, then repeatedly `readdir()` to get directory entries. It skips `.` and `..`.
3. **Stat caching**: For each entry, `find` calls `lstat()` (or `stat()`) to get metadata. GNU find caches the stat result to avoid re-statting for multiple tests.
4. **Expression evaluation**: Each test is evaluated left-to-right with short-circuiting.
5. **Action**: If all tests pass, the action is executed.

### System calls involved

```
strace find /tmp -name "*.txt" -print
```

Would show:
```
openat(AT_FDCWD, "/tmp", O_RDONLY|O_NONBLOCK|O_DIRECTORY) = 3
newfstatat(3, {st_mode=S_IFDIR|0755})    = 0
getdents64(3, buf, 32768)                = 512
newfstatat(3, "file.txt", {st_mode=S_IFREG}) = 0
write(1, "/tmp/file.txt\n", 14)          = 14
getdents64(3, buf, 32768)                = 0
close(3)                                  = 0
```

For each directory level: `openat` → `getdents64` (repeatedly) → for each matching file: `newfstatat` → `write` (print).

### Performance

- `find` is **fast** — it's written in C and uses buffered I/O
- Bottleneck is usually disk I/O, not CPU
- `-maxdepth` limits directory traversal time
- `-prune` avoids descending into subdirectories
- `-name` with simple globs is faster than `-regex`
- `-exec {} +` is faster than `-exec {} \;` (fewer forks)
- Over NFS, find can be very slow (metadata operations are expensive over network)

### Equivalent Python

```python
# find /tmp -name "*.txt" -type f -size +1k
import os
for dirpath, dirnames, filenames in os.walk('/tmp'):
    for f in filenames:
        path = os.path.join(dirpath, f)
        if not f.endswith('.txt'):
            continue
        if not os.path.isfile(path):
            continue
        if os.path.getsize(path) <= 1024:
            continue
        print(path)
```

## Core Examples (12 minimum)

### Example 1: Find by name

```bash
$ find . -name "*.txt"
./docs/readme.txt
./notes.txt
```

**Step-by-step**: Starts at `.`, walks every directory. For each entry, checks if filename matches `*.txt`. If yes, prints the path.

**What if** you use `-iname`? Case-insensitive matching: `find . -iname "readme.*"` matches `README.TXT`, `Readme.md`, etc.

### Example 2: Find by type and size

```bash
$ find /var/log -type f -size +10M
/var/log/syslog
/var/log/kern.log
```

**What if** you want to see sizes? `-exec ls -lh {} \;` or use `-ls`.

### Example 3: Find recently modified files

```bash
$ find . -type f -mtime -2 -name "*.sh"
./scripts/deploy.sh
```

**Step-by-step**: `-mtime -2` means "modified less than 2 days ago" (within last 2 × 24-hour periods).

**What if** you want minutes? `-mmin -120` = modified within last 120 minutes.

### Example 4: Find SUID/SGID binaries

```bash
$ find /usr -type f -perm /4000 -o -perm /2000
/usr/bin/passwd
/usr/bin/su
$ find /usr -type f \( -perm /4000 -o -perm /2000 \) -ls
```

**Step-by-step**: `-perm /4000` matches if ANY of the SUID bit (4000 octal) is set. `-perm /2000` matches SGID. Grouped with parentheses to avoid operator precedence issues.

### Example 5: Prune .git directories

```bash
$ find . -type d -name ".git" -prune -o -type f -name "*.py" -print
./src/main.py
./tests/test_main.py
```

**Step-by-step**: 1. Try `-type d -name ".git"` — if it's a `.git` directory, `-prune` says "don't descend" and the result is **true** (so `-o` skips the second part). 2. If it's NOT a `.git` directory, the first expression is false, so `-o` tries the second: `-type f -name "*.py" -print`. 3. If it matches, print.

**Memory aid**: Think of it as "prune the .git dirs, OR (otherwise) find .py files and print."

### Example 6: Batch chmod

```bash
$ find . -type f -name "*.sh" -exec chmod +x {} \;
# or faster with +
$ find . -type f -name "*.sh" -exec chmod +x {} +
```

### Example 7: Delete old temp files

```bash
$ find /tmp -type f -atime +30 -delete
```

**Warning**: `-delete` implies `-depth` (process children before parents). Test with `-print` first!

### Example 8: Find empty files

```bash
$ find . -type f -empty
$ find . -type d -empty   # empty directories
```

### Example 9: Custom output with -printf (GNU)

```bash
$ find . -name "*.mp3" -printf "%f\t%s bytes\n"
song1.mp3   5120000 bytes
song2.mp3   3400000 bytes
```

Available `-printf` directives: `%p` (path), `%f` (filename), `%s` (size), `%k` (size in KB), `%m` (permissions octal), `%t` (modification time), `%u` (user), `%g` (group).

### Example 10: Find by multiple conditions with `-o`

```bash
$ find . \( -name "*.jpg" -o -name "*.png" -o -name "*.gif" \) -type f
./image.jpg
./screenshot.png
./animation.gif
```

### Example 11: Find and move with exec

```bash
$ find . -name "*.log" -mtime +30 -exec mv {} /archive/ \;
# Safer with print0 + xargs:
$ find . -name "*.log" -mtime +30 -print0 | xargs -0 -I {} mv {} /archive/
```

### Example 12: Negation with `!` or `-not`

```bash
$ find . -type f ! -name "*.txt"
$ find . -type f -not -name "*.txt"   # GNU equivalent
```

### Example 13: Find files by depth

```bash
$ find . -maxdepth 1 -name "*.txt"      # only current directory
$ find . -maxdepth 2 -name "*.txt"      # current + one level deep
$ find . -mindepth 2 -name "*.txt"      # at least two levels deep
```

### Example 14: Find orphaned files

```bash
$ find / -nouser -o -nogroup 2>/dev/null
# Files owned by deleted users or groups
```

## Real-World Use Cases

### FOR the OS

- **Cron cleanup jobs**: `find /tmp -atime +7 -delete`
- **Disk usage audits**: `find / -type f -size +1G -exec ls -lh {} \;`
- **Permission audits**: `find / -perm /6000 -type f` (all SUID/SGID)
- **Log rotation**: `find /var/log -name "*.log" -mtime +90 -delete`

### WITH the OS

- **Find + grep**: `find . -name "*.py" -exec grep -l "TODO" {} +`
- **Find + xargs**: `find . -name "*.jpg" -print0 | xargs -0 -P 4 convert ...`
- **Find + du**: `find . -type d -exec du -sh {} \; | sort -rh`
- **Find + cp**: `find . -name "*.pdf" -exec cp {} /backup/ +`

### AGAINST the OS (security forensics)

- **Rootkit detection**: Find files with suspicious permissions, SUID binaries not in package manager database
- **Modified system files**: `find /etc -mtime -1 -ls` after a potential breach
- **Hidden backdoors**: `find / -name ".*" -type f` — hidden files
- **World-writable files**: `find / -perm -0002 -type f` — dangerous permissions

### FOR DEFENSE

- **Honeypot file monitoring**: `find /important -newer /var/log/auth.log -exec alert {} \;`
- **Integrity checking**: `find /bin -type f -printf "%p %s %m\n" | sort > baseline.txt` then diff later
- **Malware quarantine**: Find and isolate suspicious files by extension or size patterns

## Memory Aids

- **"find is a command, not a function"**: No `|` needed — it has `-exec` built in
- **`-name` vs `-iname`**: The `i` = "ignore case" (like `grep -i`)
- **`+N` vs `-N` vs `N`**: Think of a number line: `+N` (to the right, greater), `-N` (to the left, less), `N` (exact point)
- **`-exec {} \;` vs `-exec {} +`**: The `;` is like a period — one sentence per file. The `+` is like a paragraph — batch the whole thought.
- **`-prune` vs `-delete`**: Prune = "don't go in", delete = "remove it"
- **`-perm /4000`**: The `/` = "any of these bits" (like OR), `-` = "all of these bits" (like AND)

## Trap Vault (12 traps)

### Trap 1: `-exec {} \;` vs `-exec {} +`

```bash
# BAD: slow for 10,000 files
find . -exec ls -l {} \;   # runs ls 10,000 times

# GOOD: fast
find . -exec ls -l {} +    # runs ls in batches

# BUT: with +, {} must be at the END of the command
find . -exec cp {} /backup/ {} \;    # works with ;
find . -exec cp {} /backup/ {} +     # ERROR with +
```

### Trap 2: `-delete` implies `-depth`

```bash
# BAD: unexpected behavior
find . -name ".git" -delete   # -depth is implied, so it deletes contents first

# This can cause issues if combined with -prune or other path-dependent options
```

### Trap 3: Forgetting `-print0` when piping to xargs

```bash
# BAD: breaks on filenames with spaces
find . -name "*.txt" | xargs rm

# GOOD: safe
find . -name "*.txt" -print0 | xargs -0 rm
```

### Trap 4: `-prune` needs `-o` to combine with other conditions

```bash
# BAD: prunes AND doesn't print anything
find . -type d -name ".git" -prune -name "*.py"

# GOOD: prune OR find
find . -type d -name ".git" -prune -o -type f -name "*.py" -print
```

### Trap 5: Quoting `{}` in -exec

```bash
# These are equivalent — {} doesn't need quoting in most shells
find . -exec ls -l {} \;
find . -exec ls -l '{}' ';'
```

### Trap 6: `-exec ... +` doesn't guarantee all files in one invocation

```bash
find . -exec grep -l "pattern" {} +   # grep is called multiple times
# This is fine, but `-exec +` has a max argument size limit (~2MB on Linux)
```

### Trap 7: `-name` vs `-path` confusion

```bash
find . -name "*.txt"       # matches basename
find . -path "*/src/*.txt" # matches full path
find . -name "src"         # matches any directory named "src"
find . -path "*/src/*"     # matches anything inside a "src" directory
```

### Trap 8: Time-based tests use 24-hour rounding

```bash
find . -mtime 1   # exactly 1 day ago (24-48 hours ago)
find . -mtime 0   # less than 24 hours ago
find . -mtime +0  # more than 24 hours ago
```

### Trap 9: `-perm` with `+` vs `-` vs no prefix

```bash
find . -perm 644    # EXACTLY 644
find . -perm -644   # owner read/write AND group read AND other read (at least)
find . -perm /644   # ANY of those bits set
```

### Trap 10: Symbolic link behavior

```bash
find . -type l    # finds symlinks
find . -type f    # does NOT follow symlinks by default
find -L . -type f # follows symlinks (use -L or -follow)
```

### Trap 11: `-regex` uses GNU regex, not glob

```bash
find . -regex ".*\.txt"    # regex, NOT glob! .* matches anything, \. is literal dot
find . -name "*.txt"       # glob — simpler for simple patterns
```

### Trap 12: Large directory trees and performance

```bash
# BAD: searches entire / without excluding
find / -name "core"  # could take hours

# GOOD: limit depth, exclude, or use specific paths
find / -maxdepth 3 -name "core"
find / \( -path /proc -o -path /sys \) -prune -o -name "core" -print
```

## See It In The Wild

- **Git hooks**: `find . -name "*.orig" -delete` — cleanup after merge conflicts
- **Docker layers**: `find /var/lib/docker -size +100M` — large container layers
- **Mac Homebrew**: `find /usr/local -name "*.brewname"` — package tracking
- **Debian package**: `find /usr/share/doc -name "copyright"` — license files

### Exploration

1. `find /etc -type f | wc -l` — how many files in /etc?
2. `find /tmp -atime +30 -print` (before -delete) — check what would be deleted
3. `find /usr/bin -type f -exec file {} \; | grep "shell script"` — find all shell scripts
4. `find . -type f -printf "%s\t%p\n" | sort -rn | head -10` — largest 10 files

## Check Your Understanding (7 questions)

1. **Difference between `-exec {} \;` and `-exec {} +`?**
2. **Why does `-delete` imply `-depth`? What problem does this solve?**
3. **How to safely pipe `find` output to `xargs`?**
4. **What does `-prune` do, and how is it combined with `-o`?**
5. **How to find all empty files in a directory tree?**
6. **What's the difference between `-perm 644`, `-perm -644`, and `-perm /644`?**
7. **How to find all files modified in the last 24 hours?**

## Supplementary Deep Dive: Advanced find Patterns

### Combining -exec with multiple commands

```bash
$ # Run multiple commands on each found file
$ find . -name "*.sh" -exec chmod +x {} \; -exec echo "Made executable: {}" \;
Made executable: ./script.sh
Made executable: ./test.sh
# Note: the second -exec only runs if the first succeeds (due to implicit AND)
```

### Using -ok for interactive confirmation

```bash
$ find . -name "*.bak" -ok rm {} \;
< rm ... ./file.bak > ? y
```

### Finding files by inode number

```bash
$ find . -inum 123456 -print
# Useful for finding hard links to the same file
$ # Find all hard links to a specific file
$ stat -c '%i' /path/to/file | xargs -I{} find / -inum {} 2>/dev/null
```

### Find and report disk usage by type

```bash
$ find . -type f -name "*.jpg" -printf "%s\n" | awk '{sum+=$1} END {printf "Total JPG size: %.2f MB\n", sum/1048576}'
Total JPG size: 45.23 MB
```

### Excluding multiple patterns with -not

```bash
$ find . -type f \
>   -not -name "*.txt" \
>   -not -name "*.md" \
>   -not -path "*/.git/*" \
>   -print
```

### Find files with specific permissions recursively

```bash
$ find /home -type d -perm 777   # world-writable directories
$ find / -type f -perm -0002      # world-writable files
$ find / -type d -perm -1000      # sticky bit directories
```

### Using -newer for differential backups

```bash
$ touch /tmp/timestamp
$ # ... later ...
$ find /home -type f -newer /tmp/timestamp -print
# Files modified since the timestamp was created
```

### Find and count by extension

```bash
$ find . -type f -name "*.?" | awk -F. '{count[$NF]++} END {for (e in count) print count[e], e}' | sort -rn
15 pdf
12 txt
8 jpg
5 py
```

### Find files with specific content

```bash
$ find . -type f -name "*.py" -exec grep -l "TODO" {} +
# Or with xargs for performance:
$ find . -type f -name "*.py" -print0 | xargs -0 grep -l "TODO"
```

### Find and replace in files (sed + find)

```bash
$ find . -type f -name "*.txt" -exec sed -i 's/oldtext/newtext/g' {} +
# Warning: -i (in-place) behavior differs between GNU and BSD sed
```

### Finding symbolic links and their targets

```bash
$ find . -type l -ls        # list all symlinks with details
$ find . -type l -exec readlink -f {} \;  # show actual target paths
```

### Find files accessed exactly N days ago

```bash
$ # Files accessed exactly 7 days ago (between 7 and 8 days)
$ find . -atime 7
$ # Within 24-48 hours ago (1 day ago):
$ find . -atime 1
```

### Using -regex with GLOB vs regex

```bash
$ # GLOB: -name "*.txt" matches .txt extension
$ # REGEX: -regex ".*\.txt$" matches .txt extension
$ find . -regex ".*\.\(txt\|md\)$"   # .txt or .md
$ find . -iregex ".*\.pdf"           # case-insensitive .pdf
```

### Finding large directories

```bash
$ find . -type d -exec sh -c 'du -sh "$1" | cut -f1' _ {} \; | sort -rh | head -10
```

### Time-bounded batch operations

```bash
$ # Archive files from last week
$ find . -type f -mtime -7 -print0 | tar -czf /tmp/weekly_backup.tar.gz --null -T -
```
