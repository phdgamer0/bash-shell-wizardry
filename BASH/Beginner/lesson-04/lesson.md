# Lesson 4: Reading Files

## History & Origins

`cat` is one of the oldest Unix commands, appearing in Version 1 Unix (1971). The name stands for "concatenate" — its original purpose was to join multiple files together and print them. Reading a single file was just a special case of concatenation. The tool was written by Ken Thompson and Dennis Ritchie. The phrase "Useless Use of Cat" (UUOC) was coined by the user community later to mock `cat file | cmd` when `cmd < file` works.

`less` (1983) was written by Mark Nudelman as a replacement for `more` (1978), which was the original file pager from Berkeley Unix. `more` could only scroll forward. `less` could scroll both directions, search, and do everything `more` could — hence the name "less is more" (a pun on the architectural slogan "less is more"). `less` is now far more popular than `more` and is included in POSIX.

`head` and `tail` were added to BSD Unix in the late 1970s and standardized in POSIX. `head` shows the beginning of a file; `tail` shows the end. The `-f` (follow) flag for `tail` appeared in 4.4BSD. The `-n` flag for specifying a line count (vs the older `-N` syntax) was standardized later.

`nl` (number lines) was part of the System V "line printer" ecosystem. It's `cat -n` on steroids — you can control numbering style, blank lines, and page breaks. Most people use `cat -n` but `nl` has more options.

`od` (octal dump) appeared in Version 1 Unix. Before ASCII became universal, octal was the preferred way to examine binary files. The name "od" stuck even though modern use includes hex (`od -x`), decimal (`od -d`), and character (`od -c`) output.

`xxd` was written by Juergen Weigert in 1990 as part of the vim editor distribution. It was inspired by the `od` command but provides a more readable hex+ASCII side-by-side view. It can also reverse hex dumps back to binary (`xxd -r`).

`tac` is a GNU coreutils invention (1992) — literally "cat" spelled backwards. It reverses the order of lines in a file. It reads the entire file into memory (or uses lseek for large files), then outputs lines in reverse.

`rev` (reverse characters per line) appeared in the "miscellany" package of early GNU. It's useful for palindrome detection and reversing certain data formats.

**Fun anecdote:** The `cat` command was nearly removed from early Unix by Ken Thompson, who thought it was redundant with `cp` and `pr`. Ritchie convinced him to keep it because `cat` with multiple arguments was genuinely useful. Also, the classic "cat" as a pet name for the command stuck so well that GNU's version is part of "coreutils" rather than having a different name.

## Syntax Reference

### `cat` — Concatenate and Print

```bash
cat [options] [file...]
cat file1 file2 > combined     # Concatenate files
cat > file                     # Type input directly (Ctrl+D to end)
```

| Flag | Long | What it does |
|------|------|-------------|
| `-n` | `--number` | Number all output lines |
| `-b` | `--number-nonblank` | Number non-blank lines |
| `-s` | `--squeeze-blank` | Squeeze multiple blank lines into one |
| `-E` | `--show-ends` | Show `$` at end of each line |
| `-T` | `--show-tabs` | Show tabs as `^I` |
| `-v` | `--show-nonprinting` | Show non-printing characters (except tabs, ends) |
| `-A` | `--show-all` | Equivalent to `-vET` — show everything |

### `less` — Pager (Scrollable Viewer)

```bash
less [options] file...
```

| Key | What it does |
|-----|-------------|
| `Space` / `f` | Forward one page |
| `b` | Backward one page |
| `j` / Down arrow | Forward one line |
| `k` / Up arrow | Backward one line |
| `g` | Go to beginning of file |
| `G` | Go to end of file |
| `N` | Go to line N (e.g., `50g` = line 50) |
| `/pattern` | Search forward for pattern |
| `?pattern` | Search backward for pattern |
| `n` | Next match (same direction) |
| `N` | Previous match (opposite direction) |
| `&pattern` | Display only matching lines |
| `F` | Follow (like `tail -f`), Ctrl+C to stop |
| `h` | Help |
| `q` | Quit |

| Flag | Long | What it does |
|------|------|-------------|
| `-N` | `--LINE-NUMBERS` | Show line numbers |
| `-i` | `--ignore-case` | Case-insensitive search |
| `-S` | `--chop-long-lines` | Don't wrap long lines (chop) |
| `-R` | `--RAW-CONTROL-CHARS` | Interpret ANSI color codes |
| `-F` | `--quit-if-one-screen` | Exit if content fits one screen |
| `-X` | `--no-init` | Don't clear screen on exit |
| `+F` | | Start in follow mode (like `tail -f`) |

### `head` — Output First Lines

```bash
head [options] [file...]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-n N` | `--lines=N` | Output first N lines (default 10) |
| `-c N` | `--bytes=N` | Output first N bytes |
| `-q` | `--quiet` | Suppress filename headers |
| `-v` | `--verbose` | Always show filename headers |

### `tail` — Output Last Lines

```bash
tail [options] [file...]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-n N` | `--lines=N` | Output last N lines |
| `-c N` | `--bytes=N` | Output last N bytes |
| `-f` | `--follow` | Follow file as it grows (append) |
| `-F` | `--follow=name` | Follow by name (handles log rotation) |
| `-q` | `--quiet` | Suppress headers |
| `-s N` | `--sleep-interval=N` | Sleep N seconds between checks (with -f) |
| `--pid=PID` | | Stop following when process PID dies |

### `nl` — Number Lines

```bash
nl [options] [file...]
```

| Flag | What it does |
|------|-------------|
| `-ba` | Number all lines (body = all) |
| `-bt` | Number only non-empty lines (default — body = non-empty) |
| `-bn` | Don't number body lines |
| `-nln` | Left-justified, no leading zeros |
| `-nrn` | Right-justified, no leading zeros |
| `-nrz` | Right-justified, with leading zeros |
| `-wN` | Width of line number (default 6) |
| `-ssep` | Separator between number and line (default `\t`) |
| `-vN` | Starting number (default 1) |
| `-iN` | Increment by N (default 1) |

### `od` — Octal Dump

```bash
od [options] [file...]
```

| Flag | Long | What it does |
|------|------|-------------|
| `-c` | | Display as characters (backslash escapes for special) |
| `-x` | | Two-byte hex |
| `-d` | | Two-byte decimal unsigned |
| `-o` | | Two-byte octal (default) |
| `-b` | | One-byte octal |
| `-A` | `--address-radix` | Set address format: `d`=decimal, `o`=octal, `x`=hex, `n`=none |
| `-t` | `--format=TYPE` | Specify output type |
| `-j` | `--skip-bytes=N` | Skip N bytes before starting |
| `-N` | `--read-bytes=N` | Read only N bytes |

### `xxd` — Hex Dump

```bash
xxd [options] [file]
xxd -r [options] [file]       # Reverse: hex dump -> binary
```

| Flag | What it does |
|------|-------------|
| `-l N` | Stop after N octets |
| `-s N` | Start at offset N |
| `-c N` | Columns per line (default 16) |
| `-g N` | Group size in bytes (default 2) |
| `-b` | Binary digit dump |
| `-e` | Little-endian dump |
| `-u` | Uppercase hex |
| `-p` | Plain style (no address/ASCII) |
| `-i` | C include file style output |
| `-r` | Reverse: hex -> binary |

### `tac` — Reverse Line Order

```bash
tac [options] [file...]
```

| Flag | What it does |
|------|-------------|
| `-b` | Attach separator before instead of after |
| `-r` | Regex separator |
| `-s` | Use specific separator (default `\n`) |

### `rev` — Reverse Characters Per Line

```bash
rev [options] [file...]
```

No significant flags. Simple tool.

## Under the Hood

### What happens when you `cat file`

1. Shell forks and execs `/usr/bin/cat`.
2. `cat` calls `open("file", O_RDONLY)` → FD 3 (or next available).
3. Loop: `read(3, buf, 65536)` — reads up to 64KB at a time (kernel reads from disk into page cache).
4. `write(1, buf, n)` — writes to stdout (usually the terminal).
5. When `read()` returns 0 → EOF, close file, exit.

System calls:
```
openat(AT_FDCWD, "file", O_RDONLY) = 3
read(3, "file content...\n", 65536) = 1024
write(1, "file content...\n", 1024) = 1024
read(3, "", 65536) = 0
close(3) = 0
exit_group(0) = ?
```

**Buffering:** `cat` uses stdio's `fread()`/`fwrite()` which adds a layer of buffering. But `cat` specifically uses `read()`/`write()` syscalls directly (no stdio buffering) for efficiency. It's designed to be a fast data mover.

### What happens when you `tail -f file`

1. `tail` opens the file and seeks to the end (`lseek(fd, 0, SEEK_END)`).
2. It reads backwards to find the last N lines.
3. It outputs those lines.
4. If `-f` is specified, it enters a loop:
   - `sleep(s)` — wait (default 1 second).
   - `check file size` — `fstat(fd)` to check if file has grown.
   - If size changed: `lseek()` to new position, `read()` new data, output it.
   - If `-F` (follow by name): also checks if the inode changed (file was rotated). If so, reopens the file.
5. This continues until you press Ctrl+C, or `--pid` process dies.

System calls for `tail -f`:
```
openat(AT_FDCWD, "/var/log/syslog", O_RDONLY) = 3
newfstatat(3, "", {st_size=12345}, AT_EMPTY_PATH) = 0
lseek(3, 12345, SEEK_SET) = 12345     # Go to end
# Read last N lines by scanning backwards...
lseek(3, 12000, SEEK_SET) = 12000
read(3, "...", 345) = 345
write(1, "...", 345) = 345
# Follow loop:
sleep(1)
newfstatat(3, "", {st_size=12350}, AT_EMPTY_PATH) = 0  # File grew!
lseek(3, 12345, SEEK_SET) = 12345
read(3, "new log\n", 5) = 5
write(1, "new log\n", 5) = 5
...
```

### What happens when you `xxd file`

1. `xxd` opens the file, reads it into memory (or mmap's it).
2. For each byte, it formats the hex representation and the ASCII representation.
3. The left column shows the offset (address).
4. The middle columns show hex bytes, grouped.
5. The right column shows printable ASCII characters (non-printable as `.`).
6. Output is written to stdout.

### Virtual memory implications

Reading a huge file with `cat` doesn't read the whole file into memory — it reads in 64KB chunks. But `tac` with large files may need significant memory (it reads backwards using `lseek` or loads the entire file). `less` pages through files efficiently, using `mmap` or reading chunks on demand.

### The page cache

When you `cat` a file, the kernel stores the data in the **page cache** (unless you use `O_DIRECT`). A second `cat` of the same file is much faster because it reads from RAM, not disk. This is transparent — the `read()` syscall doesn't distinguish between "read from disk" and "read from cache."

## Core Examples

### Example 1: Numbered lines with cat

```bash
$ cat -n /etc/hosts
     1  127.0.0.1	localhost
     2  127.0.1.1	desktop
     3  ::1		localhost ip6-localhost ip6-loopback
```

**Step by step:**
1. `cat` opens `/etc/hosts`.
2. Reads content line by line.
3. `-n` prepends line number (right-justified in 6-character field, tab, then the line).
4. Same as `nl -ba /etc/hosts` but with different default formatting.

**What if:**
```bash
$ nl -ba -w3 -s': ' /etc/hosts   # 3-wide numbers, separator ': '
  1: 127.0.0.1	localhost
  2: 127.0.1.1	desktop
  3: ::1		localhost ip6-localhost ip6-loopback
```

### Example 2: head and tail working together

```bash
$ head -20 /etc/passwd | tail -5
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
phd:x:1000:1000:phd,,,:/home/phd:/bin/bash
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
```

**Step by step:**
1. `head -20` reads lines 1-20, writes to stdout.
2. Pipe sends lines 1-20 to `tail -5`.
3. `tail -5` reads all 20 lines but outputs only the last 5 (lines 16-20).

**What if:**
```bash
$ head -n -5 /etc/passwd     # GNU head: all EXCEPT last 5 lines
$ tail -n +5 /etc/passwd     # All lines starting from line 5 (skip first 4)
$ tail -n 10 /etc/passwd      # Same as tail -10 (POSIX prefers -n 10)
```

### Example 3: Monitor a log live

```bash
$ sudo tail -f /var/log/syslog
Jul 31 01:00:01 desktop systemd[1]: Started User Manager for UID 1000
Jul 31 01:00:01 desktop systemd[1]: Started Session 1 of User phd
...
# (Run this in one terminal, then in another:)
$ logger "testing tail -f"
# Watch it appear in the first terminal!
```

**Step by step:**
1. `tail -f` reads the last 10 lines and outputs them.
2. It then sleeps 1 second and checks if the file has grown.
3. When `logger` writes to syslog, the file size increases.
4. `tail` detects the change, reads the new data, and outputs it.
5. This loops forever until Ctrl+C.

**What if:**
```bash
$ tail -F /var/log/syslog      # Same but handles log rotation
$ tail -f --pid=12345 /var/log/build.log   # Auto-stop when process 12345 exits
$ tail -f -s 0.1 /var/log/fast.log         # Check every 0.1 seconds
```

### Example 4: Exploring binary files with xxd

```bash
$ echo -n "Hello" > /tmp/hello
$ xxd /tmp/hello
00000000: 4865 6c6c 6f                             Hello
```

**Step by step:**
1. `echo -n` writes "Hello" (5 bytes, no newline) to `/tmp/hello`.
2. `xxd` reads the file.
3. Offsets: `00000000` = byte 0.
4. Hex: `48` = 'H', `65` = 'e', `6c` = 'l', `6c` = 'l', `6f` = 'o'.
5. ASCII: `Hello` in the right column.

**What if:**
```bash
$ xxd -b /tmp/hello          # Binary: 01001000 01100101 01101100 01101100 01101111
$ xxd -p /tmp/hello          # Plain: 48656c6c6f
$ echo '48656c6c6f' | xxd -r -p   # Reverse: converts hex back to "Hello"
$ xxd -i /tmp/hello          # C array:
unsigned char hello[] = {
  0x48, 0x65, 0x6c, 0x6c, 0x6f
};
unsigned int hello_len = 5;
```

### Example 5: tac — reversing file lines

```bash
$ printf "line1\nline2\nline3\n" | tac
line3
line2
line1
```

**Step by step:**
1. `printf` sends three lines to stdout.
2. Pipe to `tac`.
3. `tac` reads all lines (or seeks to end and reads backwards).
4. Outputs lines in reverse order.

**What if:**
```bash
$ tac file1 file2            # Reverses lines of both files, output concatenated
$ tac -s '.' file            # Use '.' as "line" separator instead of newline
```

### Example 6: rev — reversing characters per line

```bash
$ echo "desserts" | rev
stressed
$ echo "hello world" | rev
dlrow olleh
```

**Step by step:**
1. `rev` reads a line.
2. Reverses the character order of that line.
3. Outputs the reversed line.
4. Does this for every line.

**What if:**
```bash
$ echo -e "abc\ndef" | rev
cba
fed
$ # Palindrome checker: if rev output == original, it's a palindrome
$ echo "racecar" | rev
racecar
```

### Example 7: less interactive exploration

```bash
$ less /var/log/syslog
# You see the file content, scrollable.
# Keys: j (down), k (up), /error (search), n (next match), q (quit)
```

**Step by step:**
1. `less` opens the file and reads the first screenful.
2. It maps the file (or reads chunks) for efficient navigation.
3. Key presses trigger actions: scrolling, searching, etc.
4. Search (`/error`) compiles a regex, finds first match, positions display there.
5. `n`/`N` navigate between matches.
6. `q` calls `exit(0)`.

**What if:**
```bash
$ less -N /etc/hosts              # Show line numbers
$ less -S /etc/hosts              # Don't wrap long lines (chop)
$ less -R output.log              # Interpret ANSI color codes
$ less +F /var/log/syslog         # Start in follow mode (tail -f)
$ grep -r "error" /var/log/ | less  # Pipe search results to less
```

### Example 8: od for octal/hex inspection

```bash
$ printf "Hello\n" | od -c
0000000   H   e   l   l   o  \n
0000006
$ printf "Hello\n" | od -x
0000000  6548 6c6c 0a6f
0000006
```

**Step by step:**
1. `od -c` shows each byte as a character. Non-printable shown as escape sequences (`\n` = newline).
2. `od -x` shows two-byte groups in hex. But note: `6548` is `He` in little-endian (x86).
3. On little-endian systems, byte order within each 2-byte word is reversed.

**What if:**
```bash
$ od -A d -c file              # Decimal addresses instead of octal
$ od -t x1 file                # One-byte hex (preferred for clarity)
$ od -j 100 -N 50 file         # Skip 100 bytes, read 50
```

### Example 9: head -c for binary header inspection

```bash
$ head -c 16 /bin/ls | xxd
00000000: 7f45 4c46 0201 0103 0000 0000 0000 0000  .ELF............
```

**Step by step:**
1. `head -c 16` reads only the first 16 bytes of `/bin/ls`.
2. Pipe to `xxd` displays the hex dump.
3. The magic bytes `7f 45 4c 46` spell `.ELF` — all ELF binaries start with this.
4. `02` = 64-bit format, `01` = little-endian, `01` = ELF version 1, `03` = System V ABI.

**What if:**
```bash
$ head -c 4 /usr/bin/python3 | xxd   # Python is also an ELF
$ head -c 8 image.png | xxd          # PNG magic: 89 50 4E 47 0D 0A 1A 0A
$ head -c 8 file.pdf | xxd           # PDF magic: 25 50 44 46 (%PDF)
```

### Example 10: Combining nl with grep

```bash
$ grep -n "root" /etc/passwd
1:root:x:0:0:root:/root:/bin/bash
$ nl /etc/passwd | grep root
     1  root:x:0:0:root:/root:/bin/bash
$ grep -n "nologin" /etc/passwd | wc -l
8
```

**Step by step:**
1. `grep -n` outputs matching lines with their line numbers (starting from 1).
2. `nl` numbers ALL lines (not just matching ones), then `grep` filters the numbered output.
3. Results are similar but formatting differs (tab vs colon separator).

**What if:**
```bash
$ nl -ba -s': ' /etc/passwd | grep root   # Match nl output formatting
```

### Example 11: Reading the middle of a file

```bash
$ wc -l /etc/passwd
45 /etc/passwd
$ head -n 30 /etc/passwd | tail -n 10   # Lines 21-30
```

**Step by step:**
1. `wc -l` tells us the file has 45 lines.
2. `head -n 30` gives lines 1-30.
3. `tail -n 10` gives the last 10 of those: lines 21-30.
4. Alternative: `sed -n '21,30p' /etc/passwd` or `awk 'NR>=21&&NR<=30' /etc/passwd`.

**What if:**
```bash
$ tail -n +21 /etc/passwd | head -n 10   # Same result: lines 21-30
$ sed -n '21,30p' /etc/passwd            # Simpler for arbitrary ranges
```

### Example 12: watch a file for changes

```bash
$ while true; do head -3 /tmp/status.txt 2>/dev/null; sleep 2; done
# This polls every 2 seconds
```

But better:
```bash
$ tail -f /tmp/status.txt   # Real-time monitor
$ less +F /tmp/status.txt   # Same thing, with less
```

**What if:**
```bash
$ inotifywait -m /tmp/status.txt   # Linux inotify: blocks until file changes
$ entr -c echo "file changed"      # Run command when file changes: echo "content" | entr script
```

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`tail -f /var/log/syslog`** — monitor system messages in real-time.
- **`head -1 /etc/shadow`** — check root's password hash format.
- **`cat /proc/cpuinfo | grep 'model name' | head -1`** — check CPU model.
- **`less +F /var/log/nginx/access.log`** — watch web server access log.
- **`od -c /dev/urandom | head -5`** — dump random bytes.
- **`xxd /usr/bin/ssh | head -20`** — inspect SSH binary header.

### 2. WITH the OS — Development

- **`head -c 32 /dev/random | xxd -p`** — generate a random hex token.
- **`nl -ba -w2 -s'. ' source.py`** — print numbered source code.
- **`tac CHANGELOG.md`** — see latest changes at the top.
- **`rev input.txt`** — reverse text for encoding puzzles.
- **`tail -f -n 50 app.log`** — watch application log (last 50 lines first).

### 3. AGAINST the OS — Exploitation

- **`cat /etc/shadow`** — dump password hashes (needs root).
- **`head -c 1024 /dev/sda | xxd`** — read raw disk data (needs root).
- **`tail -f /var/log/auth.log`** — monitor authentication logs in real-time.
- **`cat /proc/1/environ`** — extract environment variables from PID 1 (systemd leaks secrets).
- **`xxd -r payload.hex > payload.bin`** — convert hex-encoded payload to binary.

### 4. FOR DEFENSE — Detection & Auditing

- **`less /var/log/auth.log`** — review login attempts.
- **`grep 'Failed' /var/log/auth.log* | wc -l`** — count brute force attempts.
- **`head -c 20 suspicious.exe | xxd`** — check file magic to verify claims.
- **`tail -f /var/log/syslog | grep -i 'error\|fail\|attack'`** — real-time alerting.
- **`cat /proc/net/tcp | head`** — inspect active TCP connections.

## Memory Aids

### Mnemonics

- **`cat`** = "conCATenate" — it joins files together. Single file display is a special case.
- **`tac`** = "cat" spelled backwards = reverses lines.
- **`nl`** = "number lines" — simple.
- **`head`** = the head (top) of the file.
- **`tail`** = the tail (end) of the file.
- **`less`** = "less is more" — better than `more`.
- **`od`** = "octal dump" — the default format is octal, but you can change it.
- **`xxd`** = originally from xx- (hex) dump. The `xx` represents hex digits.

### Pattern hooks

- Want to see a file? `cat` for short, `less` for long.
- Want the start? `head -n N`.
- Want the end? `tail -n N`.
- Want to watch? `tail -f`.
- Want the line count? `wc -l` (not `cat -n`).
- Want to see non-printable? `cat -A` or `xxd`.

### Common confusions

- **`head -20` is deprecated**: Use `head -n 20` (POSIX-compliant).
- **`tail -f` never returns**: It's interactive. Use Ctrl+C, or `--pid` for automatic exit.
- **`cat file | cmd` vs `cmd < file`**: The latter avoids a useless fork. Neither is "wrong" for one-off use, but UUOC is a style concern.
- **`tac` and `rev` are different**: `tac` reverses line ORDER; `rev` reverses each line's CHARACTERS.
- **`less` vs `more`**: `more` can only scroll forward. `less` can scroll both ways, search, and more. Always use `less`.

## Trap Vault

### Trap 1: Useless Use of Cat (UUOC)

**Problem:** `cat file | command` forks an extra process.

**Example:**
```bash
$ cat /etc/passwd | grep root    # Works but wasteful
$ grep root /etc/passwd          # Same result, no extra process
```

**Why:** `cat` reads the file and writes to stdout. The pipe sends stdout to `grep`. But `grep` can open the file itself. The `cat` adds an unnecessary process and pipe.

**Fix:**
```bash
$ grep root /etc/passwd
$ < /etc/passwd grep root        # Redirect input
```

### Trap 2: cat a binary file to terminal

**Problem:** `cat` on a binary file garbles the terminal.

**Example:**
```bash
$ cat /bin/ls
# Terminal goes haywire! Random characters, beeps, maybe locks up.
# You might need 'reset' to fix.
```

**Why:** Binary files contain bytes that terminals interpret as control characters. Escape sequences, bell characters (`\a = 0x07`), and `Ctrl+Z` (0x1A suspends processes in some shells). Modern terminals handle this better but it's still risky.

**Fix:**
```bash
$ file /bin/ls                # Check what it is first
$ xxd /bin/ls | head -20      # Safe hex view
$ head -c 64 /bin/ls | xxd    # Just the header
```

### Trap 3: `tail -f` blocks forever

**Problem:** `tail -f` never exits on its own.

**Example:**
```bash
$ tail -f /var/log/syslog
# It just sits there. Forever.
```

**Why:** That's the feature — "follow" means watch for new data. It's meant to run until you stop it.

**Fix:**
```bash
$ tail -f --pid=$PID /var/log/build.log   # Auto-exit when PID dies
$ timeout 10 tail -f /var/log/syslog      # Exit after 10 seconds
$ Ctrl+C                                   # Manual stop
```

### Trap 4: `head` and `tail` with stdin blocking

**Problem:** `head` may cause SIGPIPE, but `tail` in a pipe may wait for more data.

**Example:**
```bash
$ command-that-runs-forever | head -5
# head exits after 5 lines. The writing process gets SIGPIPE.
# This is normal but can cause "Broken pipe" messages.
```

**Why:** `head` reads 5 lines and exits, closing the pipe. The writing process continues writing to a pipe with no readers — the kernel sends SIGPIPE. By default this kills the writer.

**Fix:** That's the expected behavior. Silence SIGPIPE errors with `2>/dev/null`.

### Trap 5: `nl` default only numbers non-blank lines

**Problem:** `nl` without flags skips blank lines, surprising users.

**Example:**
```bash
$ printf "a\n\n\nb\n" | nl
     1	a

     2	b
# Blank lines aren't numbered! Wait, they sort of are but the number doesn't increment.
```

**Why:** `nl` by default uses `-bt` (number non-empty body lines only). Blank lines get a blank line number field.

**Fix:**
```bash
$ nl -ba file    # Number ALL lines
$ cat -n file    # Simpler: always numbers all lines
```

### Trap 6: `xxd -r` requires clean hex input

**Problem:** Reverse hex dump fails on irregular input.

**Example:**
```bash
$ echo "48656c6c6f" | xxd -r -p
Hello    # Works!
$ echo "48 65 6c 6c 6f" | xxd -r -p
xxd: stdin: bad character (No: at byte 2)  # Spaces break it
```

**Why:** `xxd -r -p` expects plain hex (no spaces, no addresses, no ASCII). Spaces, newlines, or non-hex characters cause parsing errors.

**Fix:**
```bash
$ echo "48 65 6c 6c 6f" | tr -d ' \n' | xxd -r -p  # Strip spaces first
$ xxd -r /tmp/input.hex                               # Use file with proper format
```

### Trap 7: `less -R` displays raw control characters

**Problem:** ANSI escape codes are shown as raw text without `-R`.

**Example:**
```bash
$ echo -e '\e[31mRED\e[0m' > /tmp/color.txt
$ less /tmp/color.txt
# Shows: ESC[31mREDESC[0m  (ugly)
$ less -R /tmp/color.txt
# Shows: RED (in red!)  (correct)
```

**Why:** Without `-R`, `less` treats escape characters as non-printing and shows them as `ESC`. With `-R`, it passes them through to the terminal, which interprets the color codes.

**Fix:** Use `less -R` for colored output. Or `less -r` for raw control chars (stronger but may mess up display).

### Trap 8: `cat -A` surprises

**Problem:** `cat -A` (show-all) reveals hidden characters users didn't know existed.

**Example:**
```bash
$ printf "hello\tworld\n" > /tmp/test.txt
$ cat -A /tmp/test.txt
hello^Iworld$
```

**Why:** `cat -A` = `-vET`: `^I` for tabs, `$` for line endings. This is invaluable for debugging but confusing if you don't know the notation.

**Fix:** Get used to it — `cat -A` is your friend for debugging invisible characters. `^I` = tab, `^M` = Windows carriage return.

### Trap 9: No such file with cat and multiple args

**Problem:** `cat file1 file2` silently continues if `file2` doesn't exist... wait, it doesn't. It errors but continues with remaining files.

**Example:**
```bash
$ cat exists.txt missing.txt exists2.txt
# Outputs exists.txt, then error, then exists2.txt
cat: missing.txt: No such file or directory
# But output from exists.txt and exists2.txt is still written!
```

**Why:** `cat` processes arguments in order. If a file is missing, it prints an error to stderr but continues with the next file.

**Fix:** Check files first, or use shell check:
```bash
$ for f in exists.txt missing.txt exists2.txt; do [ -f "$f" ] && cat "$f"; done
```

### Trap 10: `tac` with huge files uses memory

**Problem:** `tac` on a 10GB log file may exhaust memory.

**Example:**
```bash
$ tac /var/log/massive.log > reversed.log
# Eats huge amounts of RAM or swap
```

**Why:** `tac` must read the entire file (or at least track all line positions) to output in reverse. For large files, this means holding many offsets in memory, or reading the whole file.

**Fix:**
```bash
$ tail -r /var/log/massive.log   # BSD tail -r reverses lines, uses less memory
$ sort -rn /var/log/massive.log  # If you have numeric timestamps
```

## See It In The Wild

### Where you encounter file reading daily

- **`journalctl -xe`** — reads systemd logs with a pager (usually `less` is the default pager).
- **`git log`** — pipes through `$PAGER` (usually `less`).
- **`man ls`** — man pages are displayed through `less` (or `more`).
- **`systemctl status`** — shows service status through a pager.
- **`dmesg`** — kernel ring buffer, usually piped through `less`.

### How to observe file reading with strace

```bash
$ strace -e trace=read,write cat /etc/hosts 2>&1 | head -20
$ strace -e trace=read,write head -3 /etc/passwd 2>&1
$ strace -e trace=read,write,lseek tail -5 /etc/passwd 2>&1
```

### Try this now

```bash
# Watch how cat reads work
strace -e read cat /etc/hosts 2>&1 | grep -E 'read\(3|write\(' | head -10

# Create a file with Windows line endings and find them:
printf "line1\r\nline2\r\n" > /tmp/windows.txt
cat -A /tmp/windows.txt    # See the ^M$ end-of-line markers

# Monitor your own log
logger "hello from my shell"
tail -f /var/log/syslog &
logger "another message"
kill %1

# Magic byte identification
head -c 4 /usr/bin/python3 /bin/ls /etc/passwd image.png 2>/dev/null | xxd
```

## Check Your Understanding

<details>
<summary>1. What does `head -n -5 file` do vs `tail -n -5 file`?</summary>

`head -n -5 file` shows all lines EXCEPT the last 5 (GNU extension). `tail -n -5 file` shows all lines EXCEPT the first 5. They're complementary: `tail -n +5` starts from line 5 (same as skip first 4). GNU `head -n -5` outputs all but last 5. This is NOT portable to POSIX-only systems.
</details>

<details>
<summary>2. Why might `cat largefile` fill your terminal with garbage, but `less largefile` is safe?</summary>

`cat` dumps the entire file to stdout as fast as possible. If the file is binary, control characters reach the terminal directly. `less` checks if the file is binary (via `isatty()` on stdin and examining content) and warns: "WARNING: /path may be binary, use --binary to read it anyway."
</details>

<details>
<summary>3. What's the difference between `tail -f` and `tail -F`?</summary>

`-f` (follow) watches the open file descriptor for new data. `-F` (follow by name) watches the filename — if the file is rotated (replaced with a new file with the same name, as with logrotate), `-F` reopens the new file. `-f` would still follow the old, now-renamed file.
</details>

<details>
<summary>4. In `xxd` output, what does the right column (ASCII representation) show for non-printable bytes?</summary>

Non-printable bytes (below 0x20 and above 0x7E) are shown as `.` in the ASCII column. Only printable ASCII characters (0x20-0x7E) are displayed as their character.
</details>

<details>
<summary>5. How does `less` handle searching? What is the difference between `/pattern` and `?pattern`?</summary>

`/pattern` searches forward from the current position. `?pattern` searches backward. After a search, `n` repeats in the same direction, `N` repeats in the opposite direction. The pattern is a basic regex (or extended with less -E).
</details>

<details>
<summary>6. What does `cat -s` do? When might it be useful?</summary>

`cat -s` (squeeze-blank) collapses multiple consecutive blank lines into a single blank line. Useful for cleaning up log files with many blank lines, or making config files easier to read.
</details>

<details>
<summary>7. `od -c` shows `\n` as `\n`. `od -x` shows it as `0a`. Why do these differ?</summary>

`od -c` displays bytes as characters — control characters are shown as escape sequences (`\n` = newline, `\t` = tab, `\0` = null). `od -x` displays 2-byte groups as hexadecimal numbers. The newline character (`\n` = 0x0A) shows as `0a` in hex. Both represent the same byte, just formatted differently.
</details>
