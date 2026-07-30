# Task 3: Security Audit & Safe Wrapper

## Objective
Audit a vulnerable script for injection flaws, write a safe wrapper around eval, implement input sanitization, create a restricted environment, and build a security analysis toolkit. This task simulates a real security audit scenario.

## Requirements

1. **Audit:** Given a vulnerable script, identify at least 6 injection vulnerabilities
2. **Safe eval wrapper:** Write a function `safe_eval` that uses `printf '%q'` to sanitize arguments
3. **Input sanitizer:** Create a function that only allows alphanumeric, `-`, `_`, and `.` characters
4. **Restricted environment:** Create a script that runs a user command in a restricted shell with locked-down PATH and read-only variables
5. **Security scanner:** Build a script that scans a given bash script for common vulnerability patterns
6. **Exploit demonstration (safe):** Show how each vulnerability can be exploited in a safe, contained environment

## Sub-tasks (8 cumulative)

### Task 3.1: Vulnerability Audit

Given this script, identify ALL vulnerabilities (aim for at least 6):

```bash
#!/bin/bash
echo "Welcome to the vulnerable script"

echo "Enter filename:"
read filename
eval "ls -l $filename"

echo "Enter command to run:"
read cmd
$cmd

echo "Enter arithmetic expression:"
read expr
echo $(($expr))

echo "Enter a message:"
read msg
echo "You said: $msg" > /tmp/log.txt

echo "Enter email:"
read email
sendmail "$email" < /tmp/msg.txt

echo "Enter hostname to ping:"
read host
ping -c 1 $host
```

**Vulnerability list:**

1. **Line 5-6:** `eval "ls -l $filename"` — classic command injection via `;`, `|`, `$()`
2. **Line 9-10:** `$cmd` — direct variable execution, any command runs
3. **Line 13-14:** `$(($expr))` — arithmetic evaluation can execute code in older bash
4. **Line 18:** `> /tmp/log.txt` — path traversal if `$msg` contains path components? No, but the redirection truncates the file; also command injection if msg contains special chars
5. **Line 22:** `sendmail "$email" < /tmp/msg.txt` — if `$email` is `; rm -rf /`, the semicolon executes after sendmail (double-quoting prevents this here — actually this is safe! sendmail receives one argument)
6. **Line 26:** `ping -c 1 $host` — unquoted variable, host could be `; rm -rf /` or `-c 5 127.0.0.1` to change behavior

Wait — actually `sendmail "$email"` IS properly quoted. The vulnerabilities are:
1. `eval "ls -l $filename"` — RCE
2. `$cmd` — arbitrary command execution
3. `$(($expr))` — code execution in old bash
4. `$msg` unquoted in redirection target — `msg="file; ls"` expands to two words
5. `ping -c 1 $host` — unquoted $host allows option injection and command injection
6. `read filename` without `-r` — backslash interpretation
7. `read cmd` without `-r`

### Task 3.2: Write a Safe Eval Wrapper

```bash
safe_eval() {
  local cmd=""
  for arg in "$@"; do
    cmd+="$(printf '%q' "$arg") "
  done
  eval "$cmd"
}
```

**Multiple approaches compared:**

```bash
# Approach A: Quote each argument (above) — safest, handles multiple args
# Approach B: Quote the entire string — breaks if args contain spaces
safe_eval_bad() {
  eval "$(printf '%q' "$*")"  # WRONG — $* joins args with IFS
}
# Approach C: Use array expansion — no eval needed!
safe_no_eval() {
  "$@"  # Just run the arguments directly as a command
}
```

When is `safe_eval` actually needed? Almost never. If you can avoid eval entirely, do so. The `safe_eval` function is for cases where you MUST use eval (e.g., dynamically constructing redirects).

### Task 3.3: Input Sanitizer

```bash
sanitize() {
  local input="$1"
  local sanitized="${input//[^a-zA-Z0-9_.-]/}"
  if [[ "$sanitized" != "$input" ]]; then
    echo "Input contained invalid characters. Sanitized: $sanitized" >&2
  fi
  printf '%s' "$sanitized"
}
```

**Comparison with alternatives:**

```bash
# Using case statement (exits on invalid)
case "$input" in
  *[!a-zA-Z0-9_-]*) echo "Invalid!" >&2; exit 1 ;;
esac

# Using grep (forks a process, slower)
if ! echo "$input" | grep -qE '^[a-zA-Z0-9_.-]+$'; then
  echo "Invalid!" >&2; exit 1
fi
```

### Task 3.4: Restricted Environment

```bash
restricted_run() {
  local cmd="$1"
  env -i \
    PATH="/usr/local/bin:/usr/bin:/bin" \
    HOME="$HOME" \
    LC_ALL=C \
    rbash -c "trap '' INT; $cmd"
}
```

**What this protects against:**
- No user-controlled environment variables (env -i clears them)
- PATH is controlled (cannot override)
- `rbash` prevents `cd`, absolute paths with `/`, and some escapes
- SIGINT is trapped (prevents Ctrl+C from escaping)

**What it does NOT protect against:**
- `rbash` escape via `bash -c ''` (if bash is in PATH)
- `rbash` escape via `python -c "import os; os.system('bash')"`
- Memory corruption in bash itself

### Task 3.5: Security Scanner

Build `scan.sh` that analyzes bash scripts for vulnerabilities:

```bash
#!/bin/bash
scan_file() {
  local file="$1"
  local issues=0
  echo "Scanning: $file"
  
  # Check for eval with variable
  if grep -nE 'eval.*\$' "$file" 2>/dev/null; then
    echo "  [HIGH] eval with variable expansion on lines above"
    ((issues++))
  fi
  
  # Check for unquoted variables
  if grep -nE '[^"]\$[A-Za-z_][A-Za-z0-9_]*[^"]' "$file" | grep -v '^\s*#'; then
    echo "  [MEDIUM] Possibly unquoted variables"
    ((issues++))
  fi
  
  # Check for unsafe arith evaluation
  if grep -nE '\$\(\(.*\$' "$file" 2>/dev/null; then
    echo "  [MEDIUM] Arithmetic evaluation with variables (may execute code)"
    ((issues++))
  fi
  
  # Check for source/with variable path
  if grep -nE 'source\s+\$|\.\s+\$' "$file" 2>/dev/null; then
    echo "  [HIGH] Source with variable path"
    ((issues++))
  fi
  
  # Check for read without -r
  if grep -nE '^[^#]*\bread\b(?!.*-r)' "$file" 2>/dev/null; then
    echo "  [LOW] read without -r (backslash interpretation)"
    ((issues++))
  fi
  
  # Check for ssh with variable command
  if grep -nE "ssh.*\\\$" "$file" 2>/dev/null; then
    echo "  [MEDIUM] SSH with variable (possible remote injection)"
    ((issues++))
  fi
  
  echo "Found $issues potential issues"
}
```

**What attack patterns might this miss?**
- Obfuscated eval (e.g., `ev` + `al` concatenation)
- Indirect expansion `${!var}`
- `xargs` injection
- Environment variable inherited from caller
- Dynamic file descriptor manipulation

### Task 3.6: Safe Demonstration (Contained)

Create a contained demonstration of injection that doesn't harm the system:

```bash
#!/bin/bash
# SAFE DEMONSTRATION - uses controlled environment
echo "=== Injection Demo (Safe) ==="
cd /tmp/demo_safe 2>/dev/null || mkdir /tmp/demo_safe && cd /tmp/demo_safe
trap "rm -rf /tmp/demo_safe" EXIT

# Demonstrate unquoted variable
touch "myfile.txt" "myfile.txt;whoami.txt"
echo "Unquoted variable expansion:"
var="myfile.*"
echo "  Without quotes:"
echo $var  # Shows both files
echo "  With quotes:"
echo "$var"  # Shows the literal string
```

### Task 3.7: Fix the Vulnerable Script

Rewrite the vulnerable script from Task 3.1 to be secure:

```bash
#!/bin/bash
echo "Welcome to the secure script"

echo "Enter filename:"
read -r filename
ls -l -- "$filename" 2>/dev/null || echo "File error"

echo "Enter command to run:"
read -r cmd
# Only allow specific commands via case
case "$cmd" in
  date|whoami|uptime)
    "$cmd"
    ;;
  *)
    echo "Command not allowed" >&2
    exit 1
    ;;
esac

echo "Enter a message:"
read -r msg
echo "You said: $msg" | tee -a /tmp/log.txt >/dev/null

echo "Enter hostname to ping:"
read -r host
# Whitelist hostname pattern
case "$host" in
  *[!a-zA-Z0-9.-]*) echo "Invalid hostname" >&2; exit 1 ;;
  *) ping -c 1 -- "$host" 2>&1 || echo "Ping failed" ;;
esac
```

### Task 3.8: Defensive Toolkit

Create a tool `defend.sh` that:
1. Takes a bash script as input
2. Runs shellcheck on it
3. Scans with the pattern scanner from Task 3.5
4. Reports a "security score" (issues per line of code)
5. Suggests fixes for each finding

```bash
#!/bin/bash
defend_script() {
  local script="$1"
  local issues=0
  local lines=0
  
  lines=$(wc -l < "$script")
  echo "=== ShellCheck ==="
  shellcheck "$script" 2>/dev/null || echo "  (shellcheck not installed)"
  
  echo "=== Pattern Scan ==="
  scan_file "$script"
  
  echo "=== Security Score ==="
  echo "  Lines: $lines"
  echo "  Score: $((issues * 100 / (lines > 0 ? lines : 1)))/100 (lower is better)"
}
```

## Bonus Challenges

1. **Bonus A:** Create a honeypot script that intentionally looks vulnerable but logs attacker attempts to syslog.

2. **Bonus B:** Build a command allowlist system that uses cryptographic signatures to verify allowed commands.

3. **Bonus C:** Implement a "safe eval" that runs in a subprocess with `timeout`, resource limits, and namespace isolation.

4. **Bonus D:** Write a script that detects shellshock-like environment variable injection attempts in real-time using auditd.

5. **Bonus E:** Create a fuzzer that generates random injection payloads and tests them against a target script in a container.

## Hints

<details>
<summary>Hint 1: Finding all vulnerabilities</summary>

Look at every place user input is used. Each `read`, each `eval`, each unquoted `$var` is a potential vector. Count them systematically.
</details>

<details>
<summary>Hint 2: Safe eval limitations</summary>

`printf '%q'` quotes for re-entry into bash. It works by escaping special characters. But it can be fooled by `$'\xHH'` ANSI-C quoting in some versions. Combine with whitelisting.
</details>

<details>
<summary>Hint 3: Restricted shell escapes</summary>

Common rbash escapes: `bash` (if in PATH), `python -c`, `perl -e`, `awk 'BEGIN{system("/bin/sh")}'`. Test each.
</details>

<details>
<summary>Hint 4: Blacklist bypasses</summary>

Attackers use: `${IFS}` instead of spaces, tabs (`\t`), `$'\n'`, `$()`, backticks, `{ls,-la}`, wildcards.
</details>

<details>
<summary>Hint 5: Testing without damage</summary>

Use `docker run --rm -it bash:latest` for a safe test environment. Or `podman` with `--userns=keep-id`.
</details>

## Expected Output

```bash
$ ./audit.sh vulnerable.sh
=== Vulnerability Report ===
HIGH:   Line 5 — eval with unquoted user input (RCE)
HIGH:   Line 9 — direct variable execution (arbitrary command)
MEDIUM: Line 13 — arithmetic evaluation may execute code
MEDIUM: Line 18 — unquoted variable in redirection
HIGH:   Line 22 — possible sendmail injection
MEDIUM: Line 26 — unquoted host variable with option injection
LOW:    Line 4, 8, 12, 16 — read without -r (backslash interpretation)

$ ./fixed.sh
Enter filename: test.txt
-rw-r--r-- 1 user user 12 Jul 31 10:00 test.txt
Enter command: whoami
user
Enter command: rm -rf /
Command not allowed
Enter hostname: 127.0.0.1
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.023 ms

$ ./scan.sh -d /usr/local/bin/*.sh
===== Security Audit =====
Scanning: /usr/local/bin/backup.sh
  [HIGH] eval with variable on line 17
  [MEDIUM] Possibly unquoted variable on line 23, 31
Found 3 potential issues

$ ./safe_eval_demo.sh
=== Safe Eval Demo ===
  Input: ; ls /
  Quoted: \ \; ls /
  Eval result: ; ls /
  (no files listed — it's a literal string now)
```

## Deep Self-Check

1. **Shellshock test:** Test your system's current bash against Shellshock. What does the warning message look like? What CVE is associated with the incomplete original fix?

2. **rbash escape attempt:** In a rbash session, try each of these escapes: `bash`, `python -c "import os; os.system('bash')"`, `env`, `export PATH=/usr/bin:$PATH`. Which work and which fail?

3. **`printf '%q'` edge cases:** Test `printf '%q'` with inputs containing newlines, tabs, null bytes, and Unicode characters. Which cause issues?

4. **`eval` with `set --`:** Combine `eval set -- "$(getopt ...)"` with user input. Can you inject through getopt output? (Hint: getopt quotes, but eval re-parses.)

5. **auditd injection detection:** Configure auditd to log all calls to `execve` by bash. What rule would you use? Test it.

6. **CVE-2019-9893:** This was a `printf '%q'` vulnerability. What was it? How did it bypass quoting?

7. **`$*` vs `$@` injection:** Write a function that uses `"$*"` (wrong) vs `"$@"` (right). Can you craft arguments that cause the wrong version to execute arbitrary code?

8. **Blacklist regex bypass:** Given `grep -E '([;&|`$()]|\\$)'`, craft an input that bypasses this filter but still executes a command.

9. **Container escape via script:** If your script runs in a Docker container with `--privileged`, which injection payloads could escape the container?

10. **Timing side channel:** Can you exploit `read -t` timeout differences to leak information from a script? How would you prevent this?
