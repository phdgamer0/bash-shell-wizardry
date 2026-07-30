# Task 9: Shell Customization Project

Customize your shell environment, understand which dotfiles run when, and debug PATH issues. By the end, you'll have a personalized shell that works the way YOU want.

## Steps

### Sub-task 1: Diagnose Your Shell Type
Determine whether your current shell is a login shell or not:
```bash
echo $0
shopt -q login_shell && echo "Login shell" || echo "Not login shell"
```

Your task: Write down BOTH results. The `$0` output shows `-bash` for login shells (notice the leading dash). Then check what's in your startup files:
```bash
ls -la ~/.bashrc ~/.bash_profile ~/.profile ~/.bash_login ~/.bash_logout 2>/dev/null
wc -l ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null
```

**Approach 1:** Use the shell commands listed above
**Approach 2:** Use `bash -xl` to trace which files actually get sourced
**Bonus:** Run `bash --login --norc` then `bash --login --noprofile` and note the differences

<details><summary>Hint: Login shell detection</summary>
`shopt -q login_shell` returns 0 if login shell, 1 if not. `echo $0` shows `-bash` for login shells (with the leading `-`). You can also check `echo $BASH_ENV` or `echo $-` — an interactive shell has `i` in the output.
</details>

### Sub-task 2: Add and Test Aliases
Edit `~/.bashrc` (or the appropriate file) to add these aliases:
```bash
alias ll='ls -lh'
alias la='ls -la'
alias lt='ls -ltrh'
alias rm='rm -i'
alias cp='cp -i'
alias mv='mv -i'
alias ..='cd ..'
alias ...='cd ../..'
alias desk='cd ~/Desktop'
```

Then source it: `. ~/.bashrc`

Now test each one. Which ones work? What happens when you try `cp` and it would overwrite a file?

**Approach 1:** Edit directly with `nano ~/.bashrc` or `vim ~/.bashrc`
**Approach 2:** Append with `echo 'alias ll="ls -lh"' >> ~/.bashrc`
**Approach 3:** Create a separate file `~/.bash_aliases` and source it from `.bashrc`

<details><summary>Hint: alias persistence</summary>
Aliases defined in `.bashrc` only work in interactive shells. For scripts, use functions. To see all your aliases: `alias` (no arguments). To bypass an alias temporarily: `\ls` or `'ls'` or `command ls`.
</details>

### Sub-task 3: Create a Custom bin Directory and Manage PATH
Create `~/bin/`, add a script `~/bin/hello` that says "Hello from ~/bin". Make it executable. Add `export PATH="$HOME/bin:$PATH"` to your `.bashrc`. Source it. Now run `hello` from anywhere.

Your script:
```bash
#!/bin/bash
echo "Hello from ~/bin"
echo "Arguments: $*"
echo "PID: $$"
```

Make it executable with `chmod +x ~/bin/hello`.

Then verify:
```bash
$ type hello
hello is /home/phd/bin/hello
$ hello
Hello from ~/bin
Arguments:
PID: 12345
$ hello world
Hello from ~/bin
Arguments: world
PID: 12346
```

**Approach 1:** Save the script, chmod, add to PATH
**Approach 2:** Create a symlink from `~/bin/hello` -> actual script location
**Approach 3:** Create a function in `.bashrc`: `hello() { echo "Hello from ~/bin"; }` — no PATH change needed

**Bonus challenge:** Add `~/bin` to PATH using `~/.profile` instead of `.bashrc`. When does this work vs not work?

<details><summary>Hint: PATH persistence across sessions</summary>
The PATH change only applies to new shells after sourcing. Your current PATH is session-local. To verify the change is permanent, open a NEW terminal window and run `echo "$PATH" | grep -o 'home/phd/bin'`.
</details>

### Sub-task 4: Trace Startup File Execution
Create a file `/tmp/shelltrace.txt` that records which startup files run by adding `echo "sourcing FILENAME" >> /tmp/shelltrace.txt` to each of `~/.bashrc`, `~/.bash_profile`, `~/.profile`. Then open a new terminal and check `/tmp/shelltrace.txt`.

Add this line to each file:
```bash
echo "sourcing $(realpath "$0" 2>/dev/null || echo $BASH_SOURCE)" >> /tmp/shelltrace.txt
```

Open a new terminal, then:
```bash
$ cat /tmp/shelltrace.txt
sourcing /etc/profile
sourcing /home/phd/.profile
sourcing /home/phd/.bashrc
```

**Approach 1:** Manual echo lines in each file
**Approach 2:** Use `PROMPT_COMMAND` or `DEBUG` trap for more detail
**Approach 3:** Use `bash -x` to trace without modifying files: `bash -xlic '' 2>&1 | grep '^\+ \.' > /tmp/trace.txt`

**Bonus:** Test all four scenarios:
```bash
ssh localhost                         # login shell, interactive
bash -l                              # login shell, interactive
gnome-terminal                       # non-login, interactive
bash -c 'echo "script"'              # non-login, non-interactive
```

<details><summary>Hint: Clean up after tracing</summary>
Remove the `echo` lines from your dotfiles after the task to avoid cluttering `trace.txt` forever. Check with `grep shelltrace ~/.bashrc` to find and remove them. Or better: create a backup first and restore after.
</details>

### Sub-task 5: Variable Export Deep Dive
Create a variable `MYVAR="test"`, then `export MYVAR`. Use `declare -p MYVAR` to show it's exported. Then run `bash -c 'echo $MYVAR'` to prove it's inherited.

```bash
$ MYVAR="test"
$ declare -p MYVAR
declare -- MYVAR="test"
$ export MYVAR
$ declare -p MYVAR
declare -x MYVAR="test"          # -x means exported!
$ bash -c 'echo "Child: $MYVAR"'
Child: test
$ bash -c 'MYVAR="modified"; echo "Child: $MYVAR"'
Child: modified
$ echo "Parent: $MYVAR"
Parent: test                      # Parent unchanged!
```

**Approach 1:** Sequential export
**Approach 2:** One-liner: `export MYVAR="test"` directly
**Approach 3:** Test with `declare -x` directly

Now test the same with a function:
```bash
$ myfunc() { echo "Original"; }
$ export -f myfunc
$ bash -c 'myfunc'
Original
```

**Bonus:** What happens with `declare -xr` (export + readonly)? Test it.

<details><summary>Hint: Function exporting</summary>
`export -f FUNCNAME` exports a function definition to child shells. This is how `.bashrc` functions appear in child bash processes.
</details>

### Sub-task 6: PATH Security Audit
Audit your current PATH for security issues:

```bash
# Find world-writable directories in PATH
$ IFS=:; for dir in $PATH; do
    if [ -d "$dir" ] && [ -w "$dir" ] && [ "$(stat -c '%U' "$dir")" != "root" ]; then
        echo "WARNING: $dir is writable by you"
    fi
    if [ -d "$dir" ] && [ -w "$dir" ] && [ "$(stat -c '%a' "$dir" | tail -c 2)" -ge 2 ]; then
        echo "WARNING: $dir is world-writable"
    fi
done
```

Check for `.` in PATH:
```bash
$ echo "$PATH" | grep -E '(^|:)(\.|:)' && echo "DANGER: dot in PATH"
```

**Approach 1:** Manual loop as shown
**Approach 2:** Use `find` to check all PATH directories: `find ${PATH//:/ } -maxdepth 0 -perm -o+w 2>/dev/null`

<details><summary>Hint: Why dot in PATH is dangerous</summary>
If you're in `/tmp` and someone has placed a malicious `ls` or `sudo` there, running `ls` would execute their version instead of the system one. Modern practice: NEVER put `.` in PATH.
</details>

### Sub-task 7: Customize Your Prompt
Build a custom PS1 prompt that shows:
- Username
- Hostname
- Current directory (abbreviated)
- Git branch (if in a git repo)
- Exit code of last command (if non-zero)
- Timestamp

Example:
```bash
$ PS1='\[\e[32m\]\u@\h\[\e[00m\]:\[\e[34m\]\w\[\e[00m\]$(__git_ps1 " (%s)" 2>/dev/null)\$ '
phd@kali:~/projects (main)$
```

**Bonus:** Create a function that toggles between a short and long prompt:
```bash
short_prompt() { PS1='\$ '; }
long_prompt()  { PS1='\u@\h:\w\$ '; }
```

<details><summary>Hint: PS1 escapes</summary>
`\u`=user, `\h`=hostname, `\w`=full path, `\W`=basename, `\d`=date, `\t`=24h time, `\T`=12h time, `\@`=12h am/pm, `\n`=newline, `\$`=`$` or `#` if root, `\\`=backslash, `\#`=command number, `\!`=history number.
</details>

## Expected Output Summary

```bash
$ echo $0
-bash                            # leading - means login shell

$ shopt -q login_shell && echo "Login" || echo "Not login"
Login

$ type hello
hello is /home/phd/bin/hello

$ hello
Hello from ~/bin

$ declare -p MYVAR
declare -x MYVAR="test"           # -x means exported

$ bash -c 'echo $MYVAR'
test                              # child shell inherited it

$ cat /tmp/shelltrace.txt
sourcing /etc/profile
sourcing /home/phd/.bashrc
```

## Self-Check Questions

1. What is the difference between `source file` and `./file`?
2. Why do login shells read `.profile` but interactive shells read `.bashrc`?
3. What is the security risk of having `.` in your PATH?
4. How do you make an alias permanent?
5. What command shows all exported environment variables?
6. Why does `export PATH="~/bin:$PATH"` not work but `export PATH="$HOME/bin:$PATH"` does?
7. If you add an alias to `.bash_profile` and it doesn't appear in your terminal, what's the most likely fix?
