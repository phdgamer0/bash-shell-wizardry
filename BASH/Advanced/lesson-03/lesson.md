# Lesson 3: Security & Injection

## History & Origins

Shell injection has existed as long as shells have. The first known shell injection vulnerability was in the Bourne shell (1977) where environment variables could trigger arbitrary command execution. The fundamental issue is that shells conflate code and data — a string isn't quoted by default, so `echo $user_input` treats `$user_input` as both data (to print) and code (to expand).

The most famous shell injection incident was the **Morris Worm (1988)**, which used a shell injection in the `fingerd` service to gain remote access. The worm sent a 536-byte payload that overflowed a buffer and executed a shell command via `gets()`. This led to the creation of the first CERT and fundamentally changed security awareness.

**Shellshock (CVE-2014-6271)** was the most impactful bash vulnerability. Discovered by Stéphane Chazelas in September 2014, it allowed attackers to execute arbitrary commands by exporting a specially crafted environment variable. The bug existed for 25 years (since bash 1.03 in 1989). Shellshock affected millions of systems — web servers, DHCP clients, SSH servers, and any service that sets environment variables based on user input.

The restricted shell (`rbash`) was introduced in bash 1.14 (1992) as a way to limit what users could do. It's not a security boundary — it's a convenience feature. Numerous escape techniques exist.

## Syntax Reference

### Injection Primitives
```
; command    # Command separator — runs command regardless of previous exit
| command    # Pipe — runs command and feeds it stdin
$(command)   # Command substitution — output replaces the expression
`command`    # Legacy command substitution
&            # Background execution
|| command   # OR — runs if previous command failed
&& command   # AND — runs if previous command succeeded
> file       # Redirect output
< file       # Redirect input
```

### Defense Mechanisms
```
printf '%q' string        # Quote string safely for shell re-entry
read -r                   # Raw read — no backslash escape
set -o noglob             # Disable pathname expansion (globbing)
set -o nounset            # Error on undefined variable
set -o pipefail           # Pipeline fails if any component fails
IFS= read -r var          # Safe read with IFS cleared
```

### Restricted Shell
```
rbash                      # Start restricted bash
exec rbash                 # Replace current shell with restricted
set -r                     # Enable restricted mode
```

### Environment Hardening
```
env -i                     # Clear environment
readonly PATH              # Protect PATH from modification
readonly VARNAME           # Protect any variable
export -n VAR              # Remove variable from environment
```

## Under the Hood

### How Injection Works at the Kernel Level

When bash parses `echo $input`, the following happens:

1. **Expansion phase:** `$input` is replaced with its value (e.g., `"; rm -rf /"`)
2. **Word splitting:** The expanded string is split on `IFS` characters (space, tab, newline)
3. **Pathname expansion:** Wildcards like `*` are expanded to matching files
4. **Quote removal:** Quotes in the original command are removed (but quotes from variable expansion are not special)
5. **Command execution:** Each word is interpreted as a command

The critical insight: **variable expansion happens before command parsing**. So if `$input` contains `; rm -rf /`, by the time bash parses the command line, it sees `echo ; rm -rf /` — two commands.

```
Original:    echo $input
              |
              v (expansion)
After:       echo ; rm -rf /
              |
              v (parsing)
Commands:    [echo] [;] [rm -rf /]
              |
              v
Executed:    execvp("echo", ["echo"])
             execvp("rm", ["rm", "-rf", "/"])
```

### The `eval` Execution Model

`eval` goes through the entire parsing pipeline again:

```bash
$ cmd="echo hello"
$ eval "$cmd"    # Parses "echo hello" as if it were typed directly
```

`eval` effectively does "re-parse the string as a bash command." This is why `eval` with untrusted input is dangerous: the input gets a second pass through the parser, with full access to all shell syntax.

### Shellshock: The Technical Detail

Shellshock (CVE-2014-6271) exploited the way bash handles function definitions in environment variables. When bash starts, it imports environment variables. If a variable looks like a function definition (starts with `() {`), bash evaluates it.

The bug: after parsing the function definition, bash continued executing trailing code:

```bash
$ env x='() { :;}; echo VULNERABLE' bash -c 'echo hello'
```

The parser saw:
1. `() { :;}` — valid function body
2. `; echo VULNERABLE` — trailing code that was ALSO executed

The fix: bash now detects trailing characters after the function definition and rejects them.

### Security Model Interactions

**SELinux and shell scripts:** SELinux labels processes based on their binary, not the script being executed. A shell script inherits the context of the shell binary (`/usr/bin/bash`). If the shell is unconfined, the script runs unconfined. This means SELinux provides limited protection against shell injection — the injected command runs with the same context as the original script.

**AppArmor:** Similar to SELinux — confinement is per-binary, not per-script. Custom AppArmor profiles can target specific scripts via path-based rules.

**Linux capabilities:** `CAP_NET_RAW` allows creating raw sockets. An injected command that tries `ping` (which needs `CAP_NET_RAW`) may fail if the shell doesn't have the capability. But most injection payloads don't need special capabilities — they use existing tools.

**namespaces:** Containerized scripts have reduced impact from injection. An injected `rm -rf /` in a container only destroys the container's filesystem, not the host's (unless bind mounts are involved).

### Comparison with C/libc

In C, injection vulnerabilities occur when user input is passed to `system()` or `popen()`:

```c
system("echo " + user_input);  // Same vulnerability as bash
```

The `execve()` family does NOT use shell parsing — it directly executes the binary with the given arguments. This makes C programs using `execve()` immune to shell injection:

```c
execve("/bin/echo", ["echo", user_input], envp);  // Safe — no shell parsing
```

The lesson: if you need to run an external command with user-supplied arguments, avoid the shell entirely. In bash, this means using arrays:

```bash
# Dangerous: shell parses the command
eval "ls -l $user_file"

# Safe: exec directly (no shell parsing)
ls -l -- "$user_file"

# Safe with array:
cmd=(ls -l -- "$user_file")
"${cmd[@]}"
```

## Core Examples

### Example 1: The Danger of eval

```bash
$ user_input="; rm -rf ~"
$ eval "echo $user_input"   # NEVER DO THIS
rm: cannot remove '/home/user': Permission denied
```

**Anatomy:**
1. `eval` receives the string `echo ; rm -rf ~`
2. `eval` re-parses it as a command line
3. Two commands: `echo` (with no args) and `rm -rf ~`
4. Both execute in the current shell context

**What if variations:**
- `eval "echo $user_input"` — dangerous because `$user_input` may contain `;`, `` ` ``, `$()`, `|`, etc.
- `eval "echo \"$user_input\""` — slightly safer but still risks command substitution inside the string
- `eval "echo $(printf '%q' "$user_input")"` — safest with eval, but why use eval at all?

### Example 2: Unquoted Variable Expansion

```bash
$ input="*"
$ echo $input
file1 file2 file3  # Oops — glob expanded!
$ echo "$input"
*  # Correct — quoted prevents expansion
```

**Anatomy:**
- `echo $input` — word splitting and pathname expansion happen
- `echo "$input"` — quotes suppress both splitting and globbing
- This is the most common shell scripting vulnerability

**What if variations:**
- `input="/*/*/*/shadow"` and `ls -la $input` — lists multiple shadow files
- `input="--help"` and `ls $input` — reveals ls help (harmless, but surprising)
- `input="-rf /"` and `rm $input` — dangerous if combined with the right command

### Example 3: Command Substitution Injection

```bash
$ filename="$(whoami).txt"
$ echo "$filename"
user.txt  # Seems safe, but:
$ filename="$(rm -rf /).txt"
$ echo "$filename"
# Your system is now deleted
```

**Anatomy:**
- Command substitution `$()` runs its contents as a command
- The output is then assigned to the variable
- But if the command substitution itself contains destructive commands, they run during expansion

### Example 4: The `--` Separator

```bash
$ echo "hello" > -f
$ cat -- -f     # Treats -f as a filename, not an option
hello
$ cat -f        # Treats -f as an option (error or flag)
cat: invalid option -- 'f'
```

**Anatomy:**
- `--` signals the end of option parsing to most POSIX commands
- Without `--`, a filename starting with `-` is interpreted as an option
- Some commands don't support `--` (check the man page)

### Example 5: Restricted Shell (rbash)

```bash
$ rbash
$ cd /tmp
rbash: cd: restricted
$ PATH=/new/path
rbash: PATH: readonly variable
$ ./some_script   # Allowed (in current dir)
$ /bin/ls          # Blocked (absolute path with /)
rbash: /bin/ls: restricted: cannot specify '/' in command names
```

**Anatomy:**
- `rbash` sets `PATH` readonly, prevents `cd`, blocks commands with `/`
- It's a convenience restriction, not a security boundary
- Many escape techniques exist

**What if variations:**
- Run `bash` inside rbash → gets a full shell (unless `bash` is not in PATH)
- Use `env` to get a shell: `env SHELL=/bin/sh bash` — rbash doesn't prevent this
- Use command substitution in a restricted context (some forms work)

### Example 6: Safe eval with printf '%q'

```bash
$ user_input="safe string with 'quotes' and \$vars"
$ eval "echo $(printf '%q' "$user_input")"
safe string with 'quotes' and $vars  # Correctly echoed
$ user_input="; rm -rf /"
$ eval "echo $(printf '%q' "$user_input")"
; rm -rf /  # Correctly echoed as data, NOT executed
```

**Anatomy:**
- `printf '%q'` outputs a shell-quoted version of the string
- Special characters (`;`, `$`, `` ` ``, etc.) are escaped
- When `eval` parses the result, it sees only a quoted string, not commands

**Limitations:** `printf '%q'` can be tricked with certain Unicode/encoding edge cases in older bash versions.

### Example 7: Input Whitelisting

```bash
$ input="ValidName_123"
$ case "$input" in
  *[!a-zA-Z0-9_-]*) echo "Invalid characters" >&2; exit 1 ;;
esac
$ echo "Processing $input"
Processing ValidName_123
```

**Anatomy:**
- The pattern `*[!a-zA-Z0-9_-]*` matches any string containing characters NOT in the allowed set
- If matched, the input is rejected
- Whitelisting is fundamentally safer than blacklisting

**Comparison with blacklist:**
```bash
# Blacklist (ALWAYS MISSES SOMETHING)
[[ "$input" == *";"* ]] && echo "Blocked semicolons"
[[ "$input" == *"|"* ]] && echo "Blocked pipes"
# What about: $IFS, tabs, newlines, or encoded variants?
```

### Example 8: Environment Injection Prevention

```bash
$ env -i PATH=/usr/bin:/bin HOME="$HOME" bash --norc
$ # Or sanitize:
$ unset IFS
$ export LC_ALL=C
```

`env -i` starts with an empty environment, then adds only explicitly listed variables. This prevents attackers from passing malicious environment variables.

### Example 9: Using Arrays to Avoid Injection

```bash
$ # Dangerous:
$ cmd="ls -la $user_file"
$ eval "$cmd"
$ # Safe with arrays:
$ cmd=(ls -la -- "$user_file")
$ "${cmd[@]}"
```

Arrays preserve word boundaries. Even if `$user_file` contains spaces, `"${cmd[@]}"` passes it as a single argument. No shell parsing occurs.

### Example 10: SSH Command Injection

```bash
$ # Dangerous:
$ ssh user@host "$command"  # If $command contains user input, remote injection
$ # Safer:
$ ssh user@host -- "$command"  # The -- helps, but shell still parses on remote
$ # Safest:
$ printf '%q ' "$command" | ssh user@host "bash -s"
```

When you run `ssh host "echo $input"`, the remote shell parses the expanded command. If `$input` contains special characters, they execute on the remote side.

### Example 11: Source/Injection

```bash
$ # Simulating what happens when you source untrusted input:
$ user_file="/tmp/evil.sh"
$ cat "$user_file"  # Check contents first? Race condition — TOCTOU
$ source "$user_file"  # Equivalent to eval on the entire file
$ # What file might contain:
$ #    rm -rf /  # Executes immediately
$ #    :(){ :|:& };:  # Fork bomb
```

**TOCTOU (Time of Check, Time of Use):** The file content can change between your check `cat` and your use `source`.

### Example 12: Piggybacking on Sudo

```bash
$ # If sudo allows a command like:
$ sudo /usr/bin/less /var/log/syslog
$ # From inside less:
$ :!bash  # Spawns a root shell (sudo runs commands, less runs commands)
```

**Defense:** Avoid `NOPASSWD` on commands that can spawn shells (less, vim, vi, more, man, etc.)

### Example 13: Arithmetic Evaluation Injection

```bash
$ expr="1+1"
$ echo $(($expr))
2
$ expr="1+1; ls"  # Older bash versions evaluate this!
$ # In bash 4.2 and earlier, $(()) could execute code
$ # Fixed: arithmetic evaluation is now sandboxed
```

### Example 14: IFS Manipulation

```bash
$ # Attacker sets IFS to ';' in environment
$ export IFS=';'
$ # Now this happens:
$ echo "Hello;World"
Hello  # IFS changed word splitting behavior
```

**Defense:** Always use `IFS= read -r` and quote all variable expansions.

### Example 15: Function Injection via Environment

```bash
$ # An attacker can pre-define a shell function that runs when your script calls a command
$ # This is the mechanism behind Shellshock
$ export evil='() { malicious_code; }'
$ bash -c 'echo hello'  # If the bug is present, evil executes
```

## Real-World Use Cases

### FOR the OS
- **System integrity:** Proper quoting prevents accidental data loss and unexpected behavior
- **SELinux/AppArmor integration:** Confined shells limit damage from injection
- **Secure by default:** Modern distros default to `-fstack-clash-protection`, ASLR, and non-executable stacks

### WITH the OS
- **Auditd monitoring:** `auditctl -w /bin/bash -p x -k shell_injection` monitors shell execution
- **SELinux sandbox:** `sandbox -X -t sandbox_x_t bash script.sh` runs scripts in confined context
- **capabilities dropping:** Run scripts with minimal capabilities using `capsh --drop=all`

### AGAINST the OS (defense education)
- **Understanding the technique:** Attackers chain injection primitives — a single unquoted variable can lead to complete compromise
- **Common vectors:** CGI scripts, `eval` on web form input, unsafe `ssh` command passing, git hooks with unsanitized ref names
- **Bypass techniques:** Using `${IFS}` instead of spaces, tab characters, null bytes, Unicode normalization attacks
- **Defense in depth:** No single measure prevents injection — combine quoting, whitelisting, restricted environments, and monitoring

### FOR DEFENSE
- **Static analysis:** `shellcheck` (shellcheck.net) catches 90% of injection vulnerabilities
- **Linting in CI:** Add `shellcheck` to pre-commit hooks or CI pipelines
- **Runtime detection:** Monitor for unexpected `exec` calls from shell scripts using auditd
- **Principle of least privilege:** Run scripts with minimal necessary permissions
- **Immutable infrastructure:** If a script is injected, the container is replaced, not repaired

## Memory Aids

- **"Quote or Croak"** — Always quote variable expansions. Unquoted variables are the #1 shell vulnerability.
- **"eval is evil"** — If you can avoid eval, do. If you can't, `printf '%q'` everything.
- **"Whitelist not Blacklist"** — Allow only what's safe, don't block what's dangerous.
- **"Arrays > Strings"** — Use `"${array[@]}"` for safe argument passing. Arrays preserve boundaries.
- **"The Shell Parses Twice"** — `eval` re-parses the string. If you wouldn't type it, don't eval it.
- **"-- saves the day"** — Use `--` before filenames to prevent option injection.

## Trap Vault

1. **`eval "echo $input"` is NEVER safe:** Even with `printf '%q'`, eval has edge cases. The only safe eval is no eval.

2. **Quoting doesn't protect inside `[[ ]]`:** `[[ $var == "$pattern" ]]` — the right side of `==` is a pattern, not a literal string, unless quoted. Quote the pattern for literal matching.

3. **`read` without `-r` eats backslashes:** `read line` interprets `\n` as newline escape. Always use `read -r`.

4. **`set -e` doesn't prevent injection:** If you eval an injection payload that succeeds (exit 0), `set -e` doesn't save you. The damage is already done.

5. **`printf '%q'` is not a security guarantee:** In some bash versions, `printf '%q'` can be bypassed with carefully crafted Unicode input. Always combine with whitelisting.

6. **Globbing in `[[ ]]` vs `[ ]]`:** `[[ $var = foo* ]]` is a pattern match (foo*). `[ "$var" = foo* ]` is a literal string comparison. Mixing these up causes logic errors.

7. **`export PATH` vs `PATH=...; export PATH`:** Same result. But if `PATH` is marked readonly, neither works. Attackers check for writable PATH components.

8. **`source` with variable path:** `source "$user_file"` is equivalent to eval. Source only files you fully control.

9. **`system()` in C vs shell:** `system("echo " + input)` in C is the same vulnerability. But `execv()` is safe — the difference is shell invocation.

10. **`readonly` can be bypassed in a new shell:** `readonly x=5; bash -c 'x=10'` — the new shell has its own scope. Readonly only affects the current shell.

11. **`trap` can be overwritten:** If an attacker injects `trap '' EXIT`, your cleanup doesn't run. Validate trap handlers at startup.

12. **`set -o nounset` causes script to abort on undefined vars.** This can DOS your own script if a variable is legitimately undefined. Use `${var:-default}` for optional variables.

13. **`${!var}` indirect reference:** `name="RM"; ${!name} -rf /` — indirect expansion can reference any command. Sanitize the control variable.

14. **The `$*` vs `$@` difference:** `"$*"` concatenates args into one string (with IFS); `"$@"` preserves individual arguments. Always use `"$@"`.

15. **Null bytes in input:** Shell strings can't contain null bytes. An attacker can't inject null to truncate strings (unlike C), but they can use other techniques.

## See It In The Wild

### Shellcheck in Action
```bash
$ cat > vuln.sh << 'EOF'
echo "Enter filename:"
read f
cat $f
EOF
$ shellcheck vuln.sh
Line 3:
cat $f
    ^-- SC2086: Double quote to prevent globbing and word splitting.
```

### Auditing /tmp for Injection Vectors
```bash
$ find /tmp -type f -perm -0002 2>/dev/null
$ find /tmp -type d -perm -0002 2>/dev/null
```

### Checking for Stale Restricted Shell Sessions
```bash
$ ps aux | grep rbash
```

### Monitoring Shellshock Attempts in Logs
```bash
$ grep -r '() {' /var/log/ 2>/dev/null | head -5
# Shellshock scans often leave () { in logs
```

### Verifying Shellshock Patch
```bash
$ env x='() { :;}; echo vulnerable' bash -c 'echo test'
bash: warning: x: ignoring function definition attempt
bash: error importing function definition for 'x'
test
# No "vulnerable" output = patched
```

### System Call Tracing of Injection
```bash
$ strace -f -e execve bash -c '
  user_input="; whoami"
  eval "echo $user_input"
' 2>&1 | grep execve
```

Watch how `execve` is called for each command in the injected chain.

## Check Your Understanding

1. Why is `eval "echo $input"` more dangerous than `echo "$input"`? What does `eval` do differently?

2. How does `printf '%q'` prevent injection? What are its limitations?

3. What is the difference between `''` (single quotes) and `""` (double quotes) for security?

4. Why does `ssh host "$command"` risk remote injection even if `$command` is safe locally?

5. List 5 ways an attacker can bypass a blacklist that blocks `;`, `|`, and `\`` characters.

6. How does a restricted shell (rbash) differ from a regular shell with readonly PATH?

7. What was the Shellshock vulnerability (CVE-2014-6271)? How did it work at the parser level?

8. Why does using an array (`cmd=(ls "$file"); "${cmd[@]}"`) prevent injection while `eval "ls $file"` doesn't?

9. How would you safely pass user input to a command that needs to run in a subshell?

10. What is the TOCTOU vulnerability when sourcing a file? How would you mitigate it?
