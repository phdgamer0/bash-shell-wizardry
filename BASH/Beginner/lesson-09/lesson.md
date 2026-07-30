# Lesson 9: Environment & Shell Config

## History & Origins

The concept of a "shell environment" dates back to the Thompson shell (1971) in Unix V1, where the shell maintained a small set of global variables. The Bourne shell (1979, Unix V7) formalized environment variables with `export`, making them inheritable across child processes. The `.profile` startup file appeared in System III (1982). When bash was created by Brian Fox in 1989 for the GNU Project, it introduced the split between login and non-login shells — `.bash_profile` for login, `.bashrc` for interactive non-login. The distinction exists because early Unix terminals were slow: you didn't want to re-run `.profile` every time you spawned a subshell.

The funny truth: the `.bashrc` naming comes from "bash run commands." There's no deep Unix history behind it — it was simply a practical filename. The whole login-vs-interactive confusion has caused more face-palming among sysadmins than almost any other shell feature. Even now, most distros work around it by having `.bash_profile` source `.bashrc`.

## Syntax Reference

### Sourcing files
```bash
source filename [arguments]   # Execute file in current shell
. filename [arguments]        # POSIX equivalent (same thing)
```

### Environment inspection
```bash
env                          # Show ALL environment variables (exported)
env -i command               # Run command with empty environment
env -u VAR command           # Unset VAR before running command
printenv                     # Same as env (more portable)
printenv VAR                 # Show single variable
set                          # Show ALL variables (shell-local + exported)
set -o                       # Show shell option flags
set -x                       # Enable debug tracing
set +x                       # Disable debug tracing
set -u                       # Error on undefined variable reference
set -e                       # Exit on error
set -o pipefail              # Pipe failures propagate
declare -p VAR               # Show variable attributes & value
declare -x VAR=value         # Declare AND export
declare -r VAR=value         # Declare readonly
declare -i VAR=value         # Declare integer attribute
declare -a VAR=(items)       # Declare array
declare -A VAR=([k]=v)       # Declare associative array
typeset                      # Same as declare (ksh compatibility)
```

### Environment manipulation
```bash
export VAR=value             # Set and mark for export
export VAR                   # Mark existing variable for export
export -n VAR                # Remove export attribute (unexport)
unset VAR                    # Remove variable entirely
unset -f FUNCNAME            # Remove a function
```

### Aliases
```bash
alias                        # List all aliases
alias name='command'         # Create alias
alias name='command1; command2'  # Multi-command alias
unalias name                 # Remove alias
unalias -a                   # Remove ALL aliases
```

### PATH and related variables
```bash
PATH=/dir1:/dir2:$PATH       # Prepend directories
PATH=$PATH:/dir1:/dir2       # Append directories
PATH=./bin:$PATH             # Security risk!
CDPATH=/dir1:/dir2           # cd search path
MANPATH=/dir1:/dir2          # man page search path
LD_LIBRARY_PATH=/dir1        # Library search path (Linux)
DYLD_LIBRARY_PATH=/dir1      # Library search path (macOS)
PS1='\u@\h:\w\$ '            # Primary prompt
PS2='> '                     # Continuation prompt
PS3='Select: '               # Select loop prompt
PS4='+ '                     # Debug trace prefix
IFS=$' \t\n'                 # Internal Field Separator
```

### Startup files
```bash
/etc/profile                 # System-wide login shell init
~/.bash_profile              # User's login shell init (checked first)
~/.bash_login                # Fallback if .bash_profile missing
~/.profile                   # Fallback if both missing (also used by POSIX sh)
~/.bashrc                    # Interactive non-login shell init
~/.bash_logout               # Login shell logout
$BASH_ENV                    # Non-interactive shell init file
$BASH_SOURCE                 # Current script's source path
```

### Other environment commands
```bash
shopt -s extglob             # Enable extended pattern matching
shopt -s nullglob            # Globs that match nothing become empty
shopt -s dotglob             # Include dotfiles in globs
shopt -s globstar            # ** for recursive globbing
shopt -s nocasematch         # Case-insensitive matching
shopt -s histappend          # Append history instead of overwriting
shopt                        # Show all shell options
shopt -p                     # Show options as shopt commands
```

## Under the Hood

### What Happens When You Log In

1. **getty/sshd** spawns a login process which authenticates you
2. The login program reads `/etc/passwd` to find your shell (e.g., `/bin/bash`)
3. Login calls `execve("/bin/bash", ["-bash", ...], environ)` — note the leading `-` in argv[0], which signals "login shell"
4. Bash sees argv[0] starts with `-` and reads:
   - `/etc/profile` (if exists)
   - First found of: `~/.bash_profile`, `~/.bash_login`, `~/.profile`
5. After startup files, bash presents `$PS1` and waits for commands

### What Happens When You Open a Terminal (GUI)

1. Terminal emulator (gnome-terminal, xterm, etc.) forks and execs bash **without** the leading `-`
2. Bash sees it's an interactive shell but NOT a login shell
3. Bash reads and executes `~/.bashrc` (not `.profile`)

### What Happens With `source file` vs `./file`

**`source file` (or `. file`):**
- The shell reads the file line by line in the **current** process
- No fork occurs
- Variable changes persist after the source completes
- `exit` in the sourced file exits the ENTIRE shell
- System call: `read()`, `open()` — that's it. The kernel doesn't even know a "script" ran.

**`./file` (or `bash file`):**
- The shell calls `fork()` to create a child process
- The child calls `execve()` to run the script
- The kernel reads the shebang line to find `/bin/bash`
- A new bash process interprets the script
- Variable changes are lost when the child exits
- System calls: `fork()`, `execve()`, `waitpid()`, `read()`, `open()`

### Environment Inheritance

The environment is stored as a flat array of `KEY=VALUE` strings in the process's memory area (the `environ` array). When a process calls `execve()`, the kernel copies the entire environment to the new process. This is how environment variables propagate:

```
Shell (PID 1234)            # Has environ = ["PATH=...", "HOME=...", ...]
  +-- fork() -> execve("script.sh")
  |   Child (PID 1235)      # Inherits same environ
  |     +-- export NEWVAR
  |     +-- exit()
  +-- $NEWVAR is empty      # Child couldn't modify parent's environ
```

The key insight: **environment flows downward only**. A child can never modify its parent's environment. This is why `source` exists — it runs in the same process, avoiding the fork barrier.

### Memory Layout

Each process's memory includes:
- **Text segment**: Program code (read-only)
- **Data segment**: Global variables
- **Heap**: Dynamic allocation (malloc)
- **Stack**: Local variables, function calls
- **Environment area**: Below the stack, holds `KEY=VALUE\0KEY=VALUE\0...`
- **Argument area**: Below environment, holds argv strings

The environment size is limited by `ARG_MAX` (usually 2MB on Linux). Check with `getconf ARG_MAX`.

### strace View

```bash
$ strace -e trace=process bash -c 'echo $HOME'
execve("/usr/bin/bash", ["bash", "-c", "echo $HOME"], 0x7fff...) = 0
arch_prctl(ARCH_SET_FS, ...)           = 0
...
write(1, "/home/phd\n", 11)            = 11
exit_group(0)                          = 0
```

The `execve` call passes the environment from the parent. All bash does internally is read `environ["HOME"]` and write it to stdout.

## Core Examples (12)

### Example 1: See Your PATH
```bash
$ echo "$PATH"
/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
```
The PATH variable is a colon-separated list. When you type a command, bash checks each directory in order. The first match wins.

**What if:** `PATH` is empty or unset? Then bash uses a default (compiled-in): usually `/usr/local/bin:/usr/bin:/bin`. Modern bash won't let you run `ls` without PATH — it uses a hash table of command locations.

### Example 2: Add to PATH Temporarily
```bash
$ export PATH="$HOME/bin:$PATH"
$ echo "$PATH"
/home/phd/bin:/usr/local/bin:/usr/bin:/bin:/usr/local/games:/usr/games
$ which mycommand  # now searches ~/bin first
```
This prepends `~/bin` so your personal scripts override system commands. Without `export`, the change is shell-local and won't reach child processes.

**What if:** You omit `$PATH` and write `export PATH="$HOME/bin"`? You just nuked every other directory. Only `~/bin` will be searched. Typing `ls` would fail. Always include `$PATH` when modifying.

### Example 3: Create and Use Aliases
```bash
$ alias ll='ls -lh'
$ alias lt='ls -ltrh'
$ alias
alias ll='ls -lh'
alias lt='ls -ltrh'
$ ll
total 48K
drwxr-xr-x 15 phd phd 4.0K Jul 31 01:00 Desktop
drwxr-xr-x  2 phd phd 4.0K Jul 31 01:00 Documents
```
Aliases are simple text substitutions. When bash sees `ll`, it replaces it with `ls -lh`. Aliases are expanded during command parsing, NOT at execution time.

**What if:** You alias `ls='ls --color=auto'` then later decide to use `\ls` (backslash prefix)? The backslash bypasses alias expansion. `\ls` runs the real `ls` with no aliasing.

### Example 4: Source vs Execute a Script
```bash
$ cat > /tmp/setvar.sh << 'EOF'
VAR="hello from script"
EOF
$ chmod +x /tmp/setvar.sh
$ ./setvar.sh
$ echo "${VAR:-empty}"
empty
$ source /tmp/setvar.sh
$ echo "$VAR"
hello from script
```
The fundamental fork semantics: `./script` creates a child, child sets VAR, child exits, parent never saw it. `source` reads the file into the current shell.

**What if:** The sourced script contains `exit`? Your terminal closes. Always use `return` in files you intend to source.

### Example 5: declare -p Shows Type
```bash
$ declare -p PATH
declare -x PATH="/usr/local/bin:/usr/bin:/bin"
$ myvar="hello"
$ declare -p myvar
declare -- myvar="hello"
$ readonly PI=3.14
$ declare -p PI
declare -r PI="3.14"
$ declare -x EXPORTED_VAR="visible"
$ declare -p EXPORTED_VAR
declare -x EXPORTED_VAR="visible"
```
`declare -p` is your x-ray vision. The flags tell you everything: `-x` means exported, `-r` means readonly, `--` means plain local variable.

### Example 6: env vs set — The Size Difference
```bash
$ env | wc -l
28
$ set | wc -l
165
```
`env` shows only what was in the `environ` array at process start plus anything explicitly `export`ed. `set` shows everything — every local variable, every shell function definition, every internal shell parameter.

**What if:** You run `env -i bash`? That starts bash with an empty environment. No PATH, no HOME, no USER. Try it — your prompt changes because PS1 isn't set.

### Example 7: Startup File Tracing
```bash
$ bash -l -c 'echo done'  # -l forces login shell
$ bash -c 'echo done'     # Non-login, non-interactive
$ bash -i -c 'echo done'  # Interactive non-login
```
Each invocation reads different startup files. To see what's actually sourced:
```bash
$ bash -xlic '' 2>&1 | grep '^\. ' | head -5
. /etc/profile
. ~/.bashrc
```

### Example 8: Unset vs Empty
```bash
$ EMPTY=""
$ unset UNSET
$ echo "EMPTY: [${EMPTY:-fallback}]"
EMPTY: [fallback]
$ echo "UNSET: [${UNSET:-fallback}]"
UNSET: [fallback]
$ echo "EMPTY: [${EMPTY-empty}]"
EMPTY: []
$ echo "UNSET: [${UNSET-empty}]"
UNSET: [empty]
```
`unset` removes the variable entirely. Setting to `""` keeps it but empty. They differ in `${var-default}` vs `${var:-default}` — the colon checks existence+non-empty.

### Example 9: PS1 — Your Prompt
```bash
$ PS1='\u@\h:\w\$ '
phd@hostname:~/projects$
$ PS1='[ \d \t ] \w > '
[ Fri Jul 31 01:23:45 ] ~ >
$ PS1='\n\$ '

$
```
PS1 escape sequences: `\u`=user, `\h`=hostname, `\w`=working directory, `\d`=date, `\t`=time, `\n`=newline.

### Example 10: Alias Expansion and Functions
```bash
$ alias up='cd ..'
$ up
$ cd /var/log/apache2
$ up
$ up; up

# Functions handle arguments — aliases can't
$ mkcd() { mkdir -p "$1" && cd "$1"; }
$ mkcd /tmp/newproject && pwd
/tmp/newproject
```
Aliases can't handle arguments. Functions can. If you find yourself wanting `alias mkcd='mkdir -p $1 && cd $1'`, that's a function.

### Example 11: Shell Options with shopt
```bash
$ shopt -s globstar
$ echo /etc/**/*.conf 2>/dev/null | wc -w
127
$ shopt -u globstar
$ echo /etc/**/*.conf
/etc/**/*.conf
$ shopt -s nullglob
$ echo /tmp/*.nonexistent  # prints nothing
```
`shopt` toggles bash shell options. `globstar` enables `**` recursive matching. `nullglob` makes unmatched globs vanish.

### Example 12: Source a Library
```bash
$ cat > ~/colors.sh << 'EOF'
export RED='\033[0;31m'
export GREEN='\033[0;32m'
export NC='\033[0m'
info()  { echo -e "${GREEN}[INFO]${NC} $*"; }
error() { echo -e "${RED}[ERROR]${NC} $*" >&2; }
EOF
$ source ~/colors.sh
$ info "System check passed"
[INFO] System check passed
$ declare -p GREEN
declare -x GREEN="\\033[0;32m"
```
This shows the library pattern: source a file that defines functions and exports variables.

## Real-World Use Cases

### 1. FOR the OS — Administration, Automation, System Maintenance
- **System-wide PATH management**: Add custom directories to PATH via `/etc/profile.d/` scripts
- **Default editor**: `export EDITOR=vim` in `/etc/profile`
- **Motd replacement**: Customize `/etc/profile.d/motd.sh` to show system stats on login
- **History sharing**: `export PROMPT_COMMAND='history -a'` to immediately append to history
- **UMASK setting**: `umask 022` in profile to ensure new files are world-readable

### 2. WITH the OS — Development, Data Processing, Daily Workflow
- **Language-specific config**: `export GOPATH=$HOME/go`, `export JAVA_HOME=/usr/lib/jvm/default`
- **PS1 with git branch**: Embed `$(git branch 2>/dev/null | sed 's/* //')` in PS1
- **Alias for common typos**: `alias sl='ls'`, `alias gerp='grep'`
- **Quick navigation**: `alias desk='cd ~/Desktop'`, `alias ..='cd ..'`
- **Command shortcuts**: `alias n='nano'`

### 3. AGAINST the OS — Exploitation, Bypasses, Attacks
- **PATH hijacking**: If `/tmp` is early in root's PATH, place a malicious `ls`
- **LD_PRELOAD injection**: Set `LD_PRELOAD` to a malicious .so
- **IFS manipulation**: Setting IFS to `/` changes how bash splits paths
- **PS1 injection**: A crafted PS1 that executes commands on prompt display
- **BASH_ENV trick**: If you control BASH_ENV, force arbitrary script execution

### 4. FOR DEFENSE — Detection, Prevention, Auditing
- **Readonly PATH**: `declare -r PATH` in system scripts prevents modification
- **Immutable dotfiles**: `chattr +i ~/.bashrc ~/.bash_profile` prevents tampering
- **Env whitelist**: Check for unexpected environment variables in startup files
- **PATH auditing**: Periodically check PATH entries with `find $PATH -type d -perm -o+w`
- **Login monitoring**: Log all logins via `/etc/profile` hooks

## Memory Aids

- **"source dot"**: `. file` is the POSIX source. Think "dot means 'do over here.'"
- **PS1, PS2, PS3, PS4**: "P" stands for "Prompt String." 1=main, 2=continuation, 3=select, 4=debug.
- **IFS**: "I Frickin' Split" — because it decides where words split.
- **shopt**: "Shell OPTions" — the 'sh' of shell + 'opt' for options.
- **Startup file order**: "Personal Before BASH" — `.profile` (Personal/POSIX) before `.bashrc` (bash).
- **How to remember `declare -p`**: "declare = describe" — it describes the variable.

## Trap Vault (12 traps)

### Trap 1: Sourcing Instead of Running
**Problem:** You run a configuration script expecting it to set up variables, but nothing changes.
**Example:**
```bash
$ cat > setup.sh << 'EOF'
export PROJECT_DIR=/opt/myproject
cd "$PROJECT_DIR"
EOF
$ chmod +x setup.sh
$ ./setup.sh
$ echo "$PROJECT_DIR"

$ pwd
/home/phd
```
**Why:** `./setup.sh` runs in a subshell. The `export` and `cd` happen in the child.
**Fix:**
```bash
$ source setup.sh
$ echo "$PROJECT_DIR"
/opt/myproject
```

### Trap 2: .bashrc vs .bash_profile Confusion
**Problem:** You add aliases to `.bash_profile`, open a terminal, aliases aren't there.
**Example:**
```bash
$ echo 'alias ll="ls -lh"' >> ~/.bash_profile
$ gnome-terminal
$ ll
-bash: ll: command not found
```
**Why:** Terminal emulators start non-login interactive shells, which only read `.bashrc`.
**Fix:**
```bash
$ echo '[[ -f ~/.bashrc ]] && . ~/.bashrc' >> ~/.bash_profile
$ echo 'alias ll="ls -lh"' >> ~/.bashrc
```

### Trap 3: PATH Nuking
**Problem:** You try to add a directory but replace the whole PATH.
**Example:**
```bash
$ export PATH="/my/custom/dir"
$ ls
-bash: ls: command not found
```
**Why:** You omitted `:$PATH`. Only `/my/custom/dir` is in PATH now.
**Fix:**
```bash
$ export PATH=/usr/local/bin:/usr/bin:/bin
$ export PATH="/my/custom/dir:$PATH"
```

### Trap 4: Alias Not Expanding in Scripts
**Problem:** Your script doesn't recognize aliases from `.bashrc`.
**Example:**
```bash
$ cat ~/myscript.sh
#!/bin/bash
ll /tmp
$ ./myscript.sh
./myscript.sh: line 2: ll: command not found
```
**Why:** Aliases are disabled in non-interactive shells (scripts).
**Fix:** Define functions instead of aliases for anything used in scripts.

### Trap 5: exit in Sourced Script
**Problem:** You source a utility file and your terminal closes.
**Example:**
```bash
$ cat > utils.sh << 'EOF'
check_root() {
    if [ "$(id -u)" -eq 0 ]; then
        echo "OK"
    else
        echo "Not root"
        exit 1
    fi
}
EOF
$ source utils.sh
$ check_root
Not root
# Terminal closes!
```
**Why:** `exit` in a sourced script exits the current shell.
**Fix:** Use `return` in functions meant to be sourced.

### Trap 6: Trailing Spaces in PATH
**Problem:** Adding a path with a trailing space breaks things.
**Example:**
```bash
$ export PATH="/my/dir /other:$PATH"
```
**Why:** The space makes bash see `/my/dir` and `/other` as separate entries.
**Fix:** `export PATH="/my/dir:/other:$PATH"`

### Trap 7: Forgetting to Export
**Problem:** Variable set but not visible to child processes.
**Example:**
```bash
$ MYAPP_CONFIG=/etc/myapp/config
$ ./myapp.sh
# can't see $MYAPP_CONFIG
```
**Fix:** `export MYAPP_CONFIG=/etc/myapp/config`

### Trap 8: env -i Without PATH
**Problem:** Clean environment but can't find anything.
**Example:**
```bash
$ env -i bash -c 'ls'
bash: line 1: ls: command not found
```
**Fix:** `env -i PATH="$PATH" bash -c 'ls'`

### Trap 9: declare -r Can't Be Unset
**Problem:** Readonly variable blocks your script.
**Example:**
```bash
$ declare -r SHELL=/bin/bash
$ unset SHELL
-bash: unset: SHELL: cannot unset: readonly variable
```
**Fix:** Start a new shell with `bash --norc` to bypass problematic profiles.

### Trap 10: ~ Expansion in Quotes
**Problem:** `PATH="~/bin:$PATH"` doesn't work.
**Example:**
```bash
$ export PATH="~/bin:$PATH"
$ echo "$PATH"
~/bin:/usr/local/bin:/usr/bin:/bin
$ which mytool
mytool not found
```
**Why:** Tilde expansion only happens outside quotes.
**Fix:** `export PATH="$HOME/bin:$PATH"`

### Trap 11: alias Expansion Order
**Problem:** Your alias is overridden by a later one.
**Example:**
```bash
$ alias cd='cd /tmp'
$ alias cd='cd /var'
# which one wins? The LAST one loaded.
```
**Fix:** Check with `alias`. Order matters in startup files.

### Trap 12: shopt Options Persist in Sourced Scripts
**Problem:** A sourced script's `shopt` changes affect your shell.
**Example:**
```bash
$ source ~/myscript.sh  # has shopt -s nullglob
$ rm *.nonexistent
rm: missing operand
```
**Fix:** Save and restore or wrap in a subshell: `(shopt -s nullglob; ...)`.

## See It In The Wild

### Your dotfiles
Every time you log in or open a terminal, your environment is configured by these files:
```bash
$ ls -la ~/.bash* ~/.profile ~/.bashrc 2>/dev/null
$ wc -l ~/.bashrc ~/.bash_profile ~/.profile 2>/dev/null
```

### Observe sourcing in action:
```bash
$ bash -xlic '' 2>&1 | head -30
```

### Try this now:
```bash
# Watch environment variable flow
$ export TEST_VAR="hello from $$"
$ bash -c 'echo "Child $$ sees: $TEST_VAR"'
$ bash -c 'unset TEST_VAR; echo "Child unset it"'
$ echo "Parent still has: $TEST_VAR"

# Watch PATH resolution
$ type -a ls
$ hash -r
$ command -V ls

# See your environment as a child sees it
$ env | sort | head -20
$ printenv | wc -l
```

## Check Your Understanding (7 questions)

1. You add `alias ll='ls -lh'` to `~/.bash_profile`. You open a new terminal in GNOME. `ll` doesn't work. Why?

2. What is the actual difference between `env` and `set`? When would each be useful?

3. A script has `export FOO=bar`. You run it with `./script.sh`. After it finishes, `echo $FOO` shows nothing. Why?

4. You want a variable to be visible to every child process but NOT modifiable by them. What combination of `declare` flags do you use?

5. What's wrong with `export PATH="$PATH:~/opt/bin"` and how do you fix it?

6. If you run `env -i bash`, why can't you run `ls`? How would you run a command under a clean environment but still have access to tools?

7. You have `exit` in a file you intend to `source`. What happens? What should you use instead?
