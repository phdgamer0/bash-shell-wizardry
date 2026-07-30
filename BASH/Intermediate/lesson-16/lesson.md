# Lesson 16: Terminal Interaction

## History & Origins

Terminal interaction in Unix dates back to the **Teletype ASR-33** (1960s) and **VT100** (1978). The escape sequences we still use (`\e[31m` for red) come from the **ANSI X3.64** standard (1979), later adopted by ECMA-48 and ISO 6429.

The `tput` command (part of **terminfo**, 1982) was a breakthrough — instead of hardcoding escape codes, you query the terminal database. `tput` reads `/etc/terminfo` or `/usr/share/terminfo` to find the right code for your terminal type (`$TERM`).

`stty` dates back to Version 7 Unix (1979), managing terminal line discipline settings. It controls how the terminal driver processes input — echo, line buffering, special characters.

`read -s` (silent) was added in **bash 2.0** (1996) for password input. Before that, you'd use `stty -echo` to turn off echo manually.

`read -n N` (read N characters) and `read -t N` (timeout) came in **bash 2.04** (1999), enabling single-key input without waiting for Enter — crucial for interactive prompts.

The naming: `tput` = "Terminal PUT" (outputs terminal capabilities). `stty` = "Set TTY" (set teletype parameters). `terminfo` = "terminal info" database. The ANSI escape `\e` represents the ESC character (0x1B), which signals the terminal "an escape sequence follows."

## Syntax Reference

### Terminal Output

| Construct | Effect |
|-----------|--------|
| `\e[31m` | Red foreground |
| `\e[41m` | Red background |
| `\e[1;31m` | Bold red |
| `\e[0m` | Reset all attributes |
| `\e[2J` | Clear screen |
| `\e[H` | Move cursor to home (1,1) |
| `\e[5B` | Move cursor down 5 lines |
| `\e[10C` | Move cursor right 10 columns |
| `\e[s` | Save cursor position |
| `\e[u` | Restore cursor position |
| `\e[?25l` | Hide cursor |
| `\e[?25h` | Show cursor |
| `\r` | Carriage return (go to column 0) |
| `\a` | Bell (beep) |

### tput equivalents

| `tput command` | ANSI equivalent | Effect |
|----------------|-----------------|--------|
| `tput setaf 1` | `\e[31m` | Red foreground |
| `tput setab 4` | `\e[44m` | Blue background |
| `tput bold` | `\e[1m` | Bold on |
| `tput sgr0` | `\e[0m` | Reset all |
| `tput clear` | `\e[2J\e[H` | Clear screen |
| `tput cols` | — | Number of columns |
| `tput lines` | — | Number of lines |
| `tput cup 5 10` | `\e[6;11H` | Cursor to row 5, col 10 |
| `tput civis` | `\e[?25l` | Hide cursor |
| `tput cnorm` | `\e[?25h` | Show cursor |

### Input with read

| Flag | Behaviour |
|------|-----------|
| `-s` | Silent mode — no echo (for passwords) |
| `-n N` | Read exactly N characters (no Enter needed) |
| `-N N` | Read exactly N characters (delimiter ignored) |
| `-t N` | Timeout after N seconds |
| `-p prompt` | Display prompt on stderr |
| `-r` | Raw mode — backslashes aren't escapes |
| `-d delim` | Read until `delim`, not newline |
| `-a array` | Store in array (split on IFS) |
| `-e` | Use Readline for input editing |

### Terminal Settings with stty

| Command | Effect |
|---------|--------|
| `stty -echo` | Disable echo of typed characters |
| `stty echo` | Enable echo |
| `stty raw` | Raw mode (no processing at all) |
| `stty -raw` | Cooked mode (line buffering) |
| `stty sane` | Reset to sane defaults |
| `stty -a` | Show all settings |
| `stty rows 50` | Set rows to 50 |
| `stty columns 120` | Set columns to 120 |

### Color Reference (ANSI 256-color)

```
\e[38;5;Nm  — foreground color N (0-255)
\e[48;5;Nm  — background color N (0-255)
```

Color ranges:
- 0-7: Standard (black, red, green, yellow, blue, magenta, cyan, white)
- 8-15: Bright versions
- 16-231: 216-color cube
- 232-255: Grayscale

## Under the Hood

### OS/kernel mechanisms

Terminal I/O goes through the **TTY subsystem** in the Linux kernel:

1. **Line discipline** (`n_tty`): Processes input — echo, line buffering, signal generation (Ctrl+C → SIGINT)
2. **Terminal parameters**: Stored in the `struct termios` for each TTY device
3. **`stty`** calls `tcsetattr(3)` / `tcgetattr(3)` which invoke `ioctl(2)` on the TTY device

**strace reveals:**
For `read -s -p "Password: " pass`:
```
write(2, "Password: ", 10)              = 10    # prompt to stderr
ioctl(0, TCGETS, {c_iflag=...})          = 0    # get terminal settings
ioctl(0, TCSETS, {c_lflag &= ~ECHO})     = 0    # turn off echo
read(0, "secret\n", 128)                 = 7    # read password (no echo!)
ioctl(0, TCSETS, {c_lflag |= ECHO})      = 0    # restore echo
```

For `tput setaf 1`:
```
open("/usr/share/terminfo/x/xterm-256color", O_RDONLY) = 3
read(3, ...)                                         = ...  # read terminfo
close(3)                                              = 0
write(1, "\33[31m", 5)                               = 5    # output ANSI code
```

### The terminal is a file

- `/dev/tty` — the controlling terminal for the current process
- `/dev/pts/N` — pseudo-terminal slave (SSH, terminal emulators)
- `/dev/console` — system console
- stdin/stdout/stderr may or may not be a terminal — check with `[ -t 1 ]`

### Process implications

- Terminal escape codes are handled by the **terminal emulator**, not the kernel
- `tput` forks and execs — it's slow in tight loops (cache values instead!)
- `stty` changes affect the **entire TTY**, not just your script — always restore!
- `read -s` modifies terminal settings temporarily — it restores them on read completion
- If your script crashes with `stty -echo` active, the terminal stays in silent mode

## Core Examples (8-12 minimum)

### Example 1: Colored output with printf

**Command:**
```bash
printf '\e[31mRed text\e[0m\n'
printf '\e[1;32mBold Green\e[0m\n'
printf '\e[44;37mBlue bg, white text\e[0m\n'
```

**Output:** (colors rendered by terminal)

```
Red text               (in red)
Bold Green             (in bold green)
Blue bg, white text    (blue background, white foreground)
```

**Step-by-step:**
1. `\e[31m` — set foreground to red
2. "Red text" — printed in red
3. `\e[0m` — reset all attributes
4. `\n` — newline

**Variations:**
- `printf '\e[3;31m'` — italic red
- `printf '\e[4;31m'` — underlined red
- `printf '\e[9;31m'` — strikethrough red

### Example 2: Read password silently

**Command:**
```bash
read -s -p "Enter password: " password
echo
echo "Password length: ${#password}"
```

**Input:** User types `secret123` (not visible)

**Output:**
```
Enter password:
Password length: 10
```

**Step-by-step:**
1. `-s` suppresses echo — characters typed are not displayed
2. `-p "Enter password: "` displays the prompt
3. Input is stored in `$password`
4. `echo` prints a newline (since `-s` suppresses the automatic newline after Enter)
5. Length is displayed, not the password itself

**Variations:**
- `read -s -p "PIN: " -n 4 pin` — read exactly 4 chars, no Enter needed
- `read -s -t 10 -p "Enter (10s): " var` — with timeout
- Password confirmation: `read -s -p "Again: " pass2`

### Example 3: Read with timeout

**Command:**
```bash
if read -t 5 -p "You have 5 seconds: " input; then
    echo "You said: $input"
else
    echo
    echo "Timed out!"
fi
```

**Input:** User types `hello` within 5 seconds

**Output:**
```
You have 5 seconds: hello
You said: hello
```

**Step-by-step:**
1. `-t 5` sets a 5-second timer
2. `-p` shows the prompt
3. `read` blocks until input or timeout
4. Exit code: 0 if input received, >0 if timeout/EOL
5. On timeout, `$input` is whatever was typed before timeout

**Variations:**
- `read -t 0` — poll for available input (returns immediately)
- `read -t 0.5` — bash 4+ supports fractional seconds
- Combine with `-n 1` for timed key press

### Example 4: Read single key (no Enter)

**Command:**
```bash
read -n 1 -p "Press Y to continue: " key
echo
case "$key" in
    [Yy]) echo "Continuing..." ;;
    *)    echo "Aborted." ;;
esac
```

**Input:** User presses `Y` (no Enter needed)

**Output:**
```
Press Y to continue: Y
Continuing...
```

**Step-by-step:**
1. `-n 1` — read exactly one character, return immediately
2. No Enter required — the moment a key is pressed, `read` returns
3. `$key` contains the character
4. `case` dispatches based on input

**Variations:**
- `read -n 1 -s key` — read key without showing it
- `read -n 3 -p "Enter 3 chars: " code` — read exactly 3 characters
- Arrow keys send escape sequences (multi-byte) — need `read -n 3` to capture

### Example 5: tput for terminal dimensions

**Command:**
```bash
rows=$(tput lines)
cols=$(tput cols)
printf 'Terminal size: %d rows x %d cols\n' "$rows" "$cols"

# Draw a horizontal line across the terminal
printf '%*s\n' "$cols" '' | tr ' ' '-'
```

**Output:**
```
Terminal size: 24 rows x 80 cols
--------------------------------------------------------------------------------
```

**Step-by-step:**
1. `tput lines` queries the terminal for row count
2. `tput cols` queries for column count
3. These involve a terminfo database lookup and an `ioctl()` call
4. The line uses `printf` with `%*s` (dynamic width) and `tr` to create dashes

**Variations:**
- Cache values: `COLS=$(tput cols); ROWS=$(tput lines)` — avoid repeated lookups
- `$COLUMNS` and `$LINES` environment variables (bash sets these automatically)
- `resize` command updates terminal size

### Example 6: printf formatting

**Command:**
```bash
printf '| %-10s | %8s | %5s |\n' "Name" "Score" "Grade"
printf '|%s|\n' "$(printf '=%.0s' {1..29})"
printf '| %-10s | %8d | %5s |\n' "Alice" 95 "A"
printf '| %-10s | %8d | %5s |\n' "Bob" 87 "B+"
printf '| %-10s | %8d | %5s |\n' "Charlie" 62 "F"
```

**Output:**
```
| Name       |    Score | Grade |
|=============================|
| Alice      |       95 |     A |
| Bob        |       87 |    B+ |
| Charlie    |       62 |     F |
```

**Step-by-step:**
1. `%-10s` — left-aligned string, minimum width 10
2. `%8d` — right-aligned integer, minimum width 8
3. `%5s` — right-aligned string, minimum width 5
4. Separator line uses `printf '=%.0s'` with brace expansion
5. Column data aligns perfectly

**Variations:**
- `%10.10s` — max width 10, truncates
- `%010d` — zero-padded integer
- `%8.2f` — floating point with 2 decimal places

### Example 7: Progress bar

**Command:**
```bash
progress() {
    local total=$1
    local i pct filled
    for i in $(seq 1 "$total"); do
        pct=$((i * 100 / total))
        filled=$pct
        printf '\r['
        for ((j=0; j<50; j++)); do
            if (( j * 2 < filled )); then printf '#'
            elif (( j * 2 == filled )); then printf '>'
            else printf ' '
            fi
        done
        printf '] %3d%%' "$pct"
        sleep 0.1
    done
    echo
}
progress 20
```

**Output:**
```
[##################################################] 100%
```
(Progressively fills from left to right over 2 seconds)

**Step-by-step:**
1. `\r` returns cursor to column 0 (start of line)
2. Each iteration overwrites the previous progress bar
3. `#` characters accumulate, `>` is the leading edge
4. Percentage updates next to the bar
5. Final `echo` moves to the next line

**Variations:**
- Color the bar: `printf '\r\e[32m%s\e[0m' "$bar"`
- Show ETA: compute elapsed time and project remaining
- Two-line: bar on top, file being processed below

### Example 8: Spinner

**Command:**
```bash
spinner() {
    local pid=$1
    local spin='|/-\'
    local i=0
    while kill -0 "$pid" 2>/dev/null; do
        printf '\r%c' "${spin:i++%4:1}"
        sleep 0.1
    done
    printf '\r '
}

echo -n "Working... "
sleep 3 &
spinner $!
echo "Done!"
```

**Output:**
```
Working... |
Working... /
Working... -
Working... \
Working... Done!
```
(The `|/-\` characters rotate in place)

**Step-by-step:**
1. `spinner` takes a PID to watch
2. Loop runs while the process exists (`kill -0`)
3. Cycles through `|`, `/`, `-`, `\` characters
4. `\r` returns to start of line, overwriting previous character
5. When process finishes, clears the spinner and exits

**Variations:**
- Use `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` Unicode braille dots for a smoother spinner
- Add elapsed time next to the spinner
- Change color: `printf '\r\e[36m%c\e[0m' ...`

### Example 9: Cursor positioning

**Command:**
```bash
#!/bin/bash
# Draw a box on the terminal
width=40
height=10
left=5
top=3

# Hide cursor
tput civis

# Draw top border
tput cup $top $left
printf '+%*s+' $((width - 2)) '' | tr ' ' '-'

# Draw sides
for ((row=1; row<height-1; row++)); do
    tput cup $((top + row)) $left
    printf '|%*s|' $((width - 2)) ''
done

# Draw bottom border
tput cup $((top + height - 1)) $left
printf '+%*s+' $((width - 2)) '' | tr ' ' '-'

# Write text inside box
tput cup $((top + height / 2)) $((left + 2))
printf 'Hello in a box!'
tput cup $((top + height + 2)) 0

# Show cursor
tput cnorm
```

**Output:** (A box drawn on the terminal with "Hello in a box!" inside)

**Step-by-step:**
1. `tput civis` — hide cursor for clean drawing
2. `tput cup row col` — position cursor
3. Box is drawn line by line
4. Text placed in center
5. `tput cnorm` — restore cursor

**Variations:**
- `\e[5B` — move down 5 lines relative
- `\e[s` / `\e[u` — save/restore cursor position
- Use for menus, dashboards, game UIs

### Example 10: Terminal size change detection with SIGWINCH

**Command:**
```bash
#!/bin/bash
draw_header() {
    local cols=$(tput cols)
    printf '\e[44;37m%*s\e[0m\n' "$cols" '' | tr ' ' '='
    printf '\e[44;37m%*s\e[0m\n' "$((cols - 1))" "  Terminal: ${cols}x$(tput lines)"
    printf '\e[44;37m%*s\e[0m\n' "$cols" '' | tr ' ' '='
}

trap 'draw_header' SIGWINCH

draw_header
echo "Resize your terminal to see the header update..."
echo "Press Ctrl+C to exit."
while true; do sleep 1; done
```

**Output:** (A header bar that spans the full terminal width, resizing when terminal changes)

**Step-by-step:**
1. `SIGWINCH` is sent when the terminal window is resized
2. `trap ... SIGWINCH` catches it and redraws
3. `tput cols` and `tput lines` query the new dimensions
4. Header fills the full width

**Variations:**
- Update dashboards on resize
- Redraw complex TUI layouts

### Example 11: Color palette display

**Command:**
```bash
for color in {0..15}; do
    printf '\e[48;5;%dm  %3d  \e[0m' "$color" "$color"
    (((color + 1) % 8 == 0)) && echo
done
```

**Output:** (16 colored blocks with their numbers)

```
  0    1    2    3    4    5    6    7
  8    9   10   11   12   13   14   15
```
(Each block is colored with the corresponding ANSI color)

**Step-by-step:**
1. `\e[48;5;Nm` — set background to 256-color N
2. Print the number with padding
3. `\e[0m` — reset
4. Every 8 colors, print newline

**Variations:**
- Display all 256 colors in a grid
- Foreground (`38;5;N`) vs background (`48;5;N`)
- Test your terminal's true color support with `\e[48;2;R;G;Bm`

### Example 12: readline editing with -e

**Command:**
```bash
read -e -p "Search: " -i "default search term" query
echo "You searched for: $query"
```

**Input:** User can edit "default search term" using arrow keys, Backspace, Ctrl+W, etc.

**Output:**
```
Search: default search term
You searched for: default search term
```

**Step-by-step:**
1. `-e` enables Readline — full line editing capability
2. `-i "text"` provides initial text that the user can edit
3. Arrow keys, Home, End, Ctrl+A, Ctrl+E, etc. all work
4. Tab completion may be available depending on configuration

**Variations:**
- `read -e -p "Path: " -i "$PWD/" filepath` — pre-fill with current directory
- `read -e -p "Command: " -i "ls -la" cmd` — editable command template

## Real-World Use Cases

### FOR the OS

- **Installation wizards**: `arch-chroot`, debconf, graphical installer TUI
- **Password prompts**: `sudo`, `ssh`, `login`, `passwd`, `gpg` all use `read -s` equivalent
- **Process monitors**: `top`, `htop`, `iotop` use cursor positioning for real-time updates
- **Text editors**: `vim`, `nano`, `emacs -nw` are fundamentally terminal control applications
- **File managers**: `ranger`, `midnight-commander`, `nnn`

### WITH the OS

- **`tput` for portable colors**: Instead of hardcoding `\e[31m`, use `tput setaf 1` for broader compatibility
- **`stty -echo; read pass; stty echo`**: Password input in pure POSIX sh
- **`stty raw; read -n 1 key; stty sane`**: Single-character input in POSIX sh (without bash extensions)
- **`printf '\e]0;My Script Title\a'`**: Set terminal window title

### AGAINST THE OS (Security Perspective)

- **`stty -echo` trap**: If a script crashes with `stty -echo` left enabled, the terminal becomes invisible — passwords aren't shown as you type. This is a usability attack vector.
- **ANSI injection in log files**: If a script outputs terminal control codes (e.g., as part of a filename), viewing the log with `cat` or `less -R` can execute escape sequences. This can be used for terminal UI redrawing attacks.
- **`\e[2J` (clear screen) in output**: An attacker can inject `\e[2J` into a log file or output — when viewed, the terminal appears blank, hiding malicious output.
- **Hidden input via `read -s`**: A malicious script can use `read -s` to steal passwords without visual feedback.
- **Terminal bell flooding**: Sending `\a` (bell) repeatedly can be a denial-of-service attack on the terminal.
- **Escape sequence history injection**: Some terminals process escape sequences that can inject commands into the shell's history (`\e[A` = up arrow = recall last command). Crafted output can "press" up arrow and Enter by echoing the right sequences.

### FOR DEFENSE

- **Always restore terminal settings**: Use `trap 'stty echo' EXIT` or save/restore with `stty -g` / `stty`
- **Strip escape codes from log output**: Use `sed 's/\e\[[0-9;]*[a-zA-Z]//g'` or `cat -v`
- **Check if stdout is a terminal before using colors**: `if [[ -t 1 ]]; then echo "Colors!"; fi`
- **Cache tput values**: `tput setaf 1` forks every time — save to variables in loops
- **Use `reset` command**: If you mess up your terminal settings permanently, `reset` restores sanity

## Memory Aids

- **`\e`** = the ESC key — think "Escape" sequences
- **`\e[0m`** = "Zero Mark" — reset everything to zero/default
- **`\e[31m`** = 3 = foreground, 1 = red (3x = foreground, 4x = background)
- **`tput`** = "Terminal PUT" — put settings/colors to the terminal
- **`stty`** = "Set TTY" — set terminal parameters
- **`-s`** for silent = "Shhh, don't show what I type"
- **`-n 1`** = "read oNe character"
- **`-t 5`** = "Timeout after 5 seconds"
- **`\r`** = "Return to start" (carriage return, like a typewriter)
- **`tput civis`** = "Cursor InVISible"
- **`tput cnorm`** = "Cursor NORMal"
- **`tput cup`** = "Cursor UP" (actually "cursor position" but think "moving cup")

## Trap Vault (8-12 traps)

### Trap 1: `stty -echo` left enabled after crash

**Problem:** Terminal becomes invisible after a script crashes.

**Bad Example:**
```bash
stty -echo
read -p "Enter code: " code
# Script crashes here — stty echo never restores
stty echo
```

**Root Cause:** The script exits before `stty echo` runs. The terminal remains in no-echo mode.

**Fix:** Use `trap` to restore on any exit:
```bash
old_stty=$(stty -g)
trap 'stty "$old_stty"' EXIT
stty -echo
read -p "Enter code: " code
# stty restored even on crash
```

### Trap 2: echo with `\e` doesn't expand

**Problem:** `echo "\e[31mred"` shows literal `\e[31mred`.

**Bad Example:**
```bash
echo "\e[31mRed text\e[0m"  # prints literal \e, not red
```

**Root Cause:** `echo` doesn't interpret escape sequences by default. On some shells/systems, `echo -e` is needed.

**Fix:** Use `printf`:
```bash
printf '\e[31mRed text\e[0m\n'
# Or with echo -e (not portable):
echo -e '\e[31mRed text\e[0m'
```

### Trap 3: Forgetting `\e[0m` — colors leak

**Problem:** Everything after your colored output is also colored.

**Bad Example:**
```bash
printf '\e[31mRed text'   # no reset!
echo " This is also red!"
```

**Root Cause:** The color attribute persists. Without reset, the terminal stays in red mode.

**Fix:** Always terminate with `\e[0m`:
```bash
printf '\e[31mRed text\e[0m\n'
```

### Trap 4: Colors in non-terminal output

**Problem:** Redirecting colored output to a file creates unreadable escape codes.

**Bad Example:**
```bash
./script.sh > output.log
cat output.log
# Shows: [31mRed text[0m
```

**Root Cause:** The script outputs escape codes regardless of where stdout goes.

**Fix:** Check if stdout is a terminal:
```bash
if [[ -t 1 ]]; then
    printf '\e[31mRed\e[0m\n'
else
    echo "Red"
fi
```

### Trap 5: tput is slow in loops

**Problem:** Script is sluggish during repeated color changes.

**Bad Example:**
```bash
for i in {1..100}; do
    echo "$(tput setaf 1)Number $i$(tput sgr0)"
done
# Slow — 200 tput forks!
```

**Root Cause:** `tput` forks a new process every time. 100 iterations × 2 tputs = 200 forks.

**Fix:** Cache tput output:
```bash
red=$(tput setaf 1)
reset=$(tput sgr0)
for i in {1..100}; do
    echo "${red}Number $i${reset}"
done
```

### Trap 6: `read -n 1` doesn't capture arrow keys

**Problem:** Arrow keys produce multi-byte sequences, not single characters.

**Bad Example:**
```bash
read -n 1 key
echo "You pressed: $key"  # gets ESC (\x1b), not the arrow key
```

**Root Cause:** Arrow keys send `\e[A` (up), `\e[B` (down), etc. — 3 bytes each. `-n 1` only reads the first byte.

**Fix:** Read 3 characters:
```bash
read -n 3 -t 0.1 key   # timeout for regular keys
case "$key" in
    $'\e[A') echo "UP" ;;
    $'\e[B') echo "DOWN" ;;
    *)       echo "Other: $key" ;;
esac
```

### Trap 7: `read -s` eats the newline

**Problem:** After `read -s`, the prompt's cursor sits on the same line.

**Bad Example:**
```bash
read -s -p "Password: " pass
echo "Got it"   # prints "Password: Got it"
```

**Root Cause:** `-s` suppresses echo of the Enter key too. No newline is printed after input.

**Fix:** Add `echo` after `read -s`:
```bash
read -s -p "Password: " pass
echo
echo "Got it"
```

### Trap 8: Terminal resize during script execution

**Problem:** Cached `tput cols` value becomes stale after terminal resize.

**Bad Example:**
```bash
cols=$(tput cols)
# ... long running operation ...
printf '%*s' "$cols" '' | tr ' ' '#'  # wrong width!
```

**Root Cause:** The terminal was resized while the script ran, but `$cols` still has the old value.

**Fix:** Query dimensions just before use, or trap SIGWINCH to update:
```bash
trap 'cols=$(tput cols)' SIGWINCH
cols=$(tput cols)
```

### Trap 9: `tput` fails outside a terminal

**Problem:** `tput: No value for $TERM and no -T specified` in cron or SSH non-tty.

**Bad Example:**
```bash
#!/bin/bash
# Runs in cron
cols=$(tput cols)  # fails!
```

**Root Cause:** `tput` needs `$TERM` set and a terminal. Cron doesn't have one.

**Fix:** Provide defaults:
```bash
cols=${COLUMNS:-80}
rows=${LINES:-24}
# Or:
cols=$(tput cols 2>/dev/null) || cols=80
```

### Trap 10: `read -e` with history causes issues

**Problem:** Readline's history interaction can be unexpected in scripts.

**Bad Example:**
```bash
read -e -p "Search: " term
if grep "$term" data.txt; then ... fi  # term could have special chars from history
```

**Root Cause:** `-e` enables history expansion (`!` commands) which can modify input in unexpected ways.

**Fix:** Disable history expansion or sanitize input:
```bash
set +H  # disable history expansion
read -e -p "Search: " term
```

### Trap 11: `\r` without overwriting can leave artifacts

**Problem:** Using `\r` to update a line leaves old characters when new text is shorter.

**Bad Example:**
```bash
printf 'Loading...'
printf '\rDone!'   # shows "Done!..."
```

**Root Cause:** `\r` returns to column 0 but doesn't clear the rest of the line. "Loading..." is 10 chars, "Done!" is 5 chars — the last 5 "ding..." remain visible.

**Fix:** Clear to end of line: `\e[K` or print spaces:
```bash
printf '\rDone!     '   # extra spaces to overwrite
# Or:
printf '\r\e[KDone!'
```

### Trap 12: Non-printable characters in PS1

**Problem:** Cursor position calculation goes wrong in prompt.

**Bad Example:**
```bash
PS1='\e[31m\u@\h\e[0m\$ '  # bash miscalculates prompt width
# Long commands wrap weirdly
```

**Root Cause:** Escape sequences are non-printing characters. If not wrapped in `\[ ... \]`, bash counts them as visible characters, breaking line wrapping.

**Fix:** Wrap escape sequences in `\[` and `\]`:
```bash
PS1='\[\e[31m\]\u@\h\[\e[0m\]\$ '
```

## See It In The Wild

- **`/usr/bin/clear`** — literally just outputs `\e[2J\e[H` (or uses `tput clear`)
- **`/usr/bin/reset`** — resets terminal to sane state (uses `stty sane` + escape codes)
- **`/usr/bin/less`** — uses terminal control for scrolling, search highlighting, and status bar
- **`/usr/bin/top`** — full-screen terminal application with cursor positioning
- **`/usr/bin/apt-get`** — uses progress bars during downloads

**Try this now:**

1. `tput longname` — see what terminal type you're using
2. `for c in {0..7}; do printf '\e[4%dm   \e[0m' $c; done; echo` — see the 8 standard background colors
3. `read -n 1 -p "Press any key: "; echo` — basic "press any key" prompt
4. `printf '\e[?25l'; sleep 2; printf '\e[?25h'` — hide cursor for 2 seconds
5. `while true; do printf '\r%s' "$(date)"; sleep 1; done` — real-time clock with `\r`

## Check Your Understanding (5-7 questions)

1. Why use `printf` over `echo` for colored output?
2. What does `\r` do in a printf format string?
3. How does `read -s` differ from `read -n 1`?
4. What environment variable holds the terminal width?
5. Why should you restore terminal settings with `stty echo` before exiting?
6. What's the difference between `tput setaf 1` and `printf '\e[31m'`?
7. Why do you need `\[...\]` around escape codes in `PS1`?
8. What happens if `tput` is called in a non-terminal environment?

---
*"The terminal is a canvas, and bash is the brush. `\e[31m` is red paint, `tput cup` is your easel, and `\r` is a fresh stroke."*
