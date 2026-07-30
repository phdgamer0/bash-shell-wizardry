# Task 16: Terminal Interaction

You'll build interactive terminal tools: a colored prompt setter, a login dialog, a progress bar, and a spinner. These are the building blocks of polished CLI tools that look professional and feel responsive.

## Sub-tasks

### 1. Colored prompt — `color_prompt.sh`

Write a script that demonstrates and optionally installs a colored `PS1` prompt.

**Requirements:**
- Show the current prompt (`echo "$PS1"`)
- Build a colored prompt with these features:
  - Red for root (`\u`), green for regular user
  - Yellow `@` symbol
  - Blue current directory (`\w`)
  - Green `$` (or `#` for root)
  - Reset at the end
- Use `\[ ... \]` around escape codes for correct line wrapping
- Option 1: Preview the prompt in a subshell
- Option 2: Append to `~/.bashrc` (with confirmation)
- Show a usage message

**Expected:**
```bash
$ ./color_prompt.sh
Current PS1: \[\e]0;\u@\h: \w\a\]${debian_chroot:+($debian_chroot)}\u@\h:\w\$

=== Colored Prompt Preview ===
Prompt: \[\e[32m\]\u\[\e[0m\]\[\e[33m\]@\[\e[0m\]\[\e[34m\]\h\[\e[0m\]:\[\e[34m\]\w\[\e[0m\]\[\e[32m\]\$\[\e[0m\]

The prompt above is colored. To install:
  ./color_prompt.sh --install

$ ./color_prompt.sh --install
Append to ~/.bashrc? [y/N] y
Prompt installed! Run: source ~/.bashrc
```

### 2. Password prompt — `login.sh`

Write a simulated login dialog with colored feedback.

**Requirements:**
- Prompt for username (regular `read -p`)
- Prompt for password (with `read -s`)
- Hardcoded credentials: username `admin`, password `secret123`
- Colored output:
  - Success: `\e[32m[SUCCESS]\e[0m` in green
  - Failure: `\e[31m[ERROR]\e[0m` in red
  - Warning: `\e[33m[WARN]\e[0m` in yellow
- Max 3 attempts
- Show remaining attempts on failure
- Lock after 3 failures, print "Account locked."
- On success, show "Welcome, [username]!"
- Clear the screen between attempts (optional, use `\e[2J\e[H`)

**Edge cases:**
- Empty username → "Username cannot be empty"
- Empty password → counted as attempt, "Password cannot be empty"
- Case-insensitive username comparison (optional)

**Expected:**
```bash
$ ./login.sh
Username: admin
Password:
[SUCCESS] Welcome, admin!

$ ./login.sh
Username: admin
Password:
[ERROR] Invalid credentials (2 attempts remaining)

Username: admin
Password:
[ERROR] Invalid credentials (1 attempt remaining)

Username: admin
Password: wrong
[ERROR] Invalid credentials (0 attempts remaining)
Account locked.
```

### 3. Progress bar — `progress.sh`

Write a progress bar that simulates copying files with real-time display.

**Requirements:**
- Accept options:
  - `-n N` — number of items (default 20)
  - `-t "title"` — title text (default "Processing")
  - `--fast` — faster animation (0.05s per step)
- Display format:
  ```
  Title: [####################                    ]  50% (10/20)
  ```
- Components:
  - Title text on the left
  - Bar: `[` + `#` for completed + spaces for remaining + `]`
  - Percentage right-aligned
  - Counter `(current/total)`
- Color the bar green when complete, yellow during
- Show elapsed time when finished: "Done! (2.5s)"
- Handle terminal width: use `tput cols` for max bar width

**Expected:**
```bash
$ ./progress.sh -n 10 -t "Copying files"
Copying files: [####################                    ]  50% (5/10)
Copying files: [########################################] 100% (10/10)
Done! (1.0s)

$ ./progress.sh --fast -n 50
Processing: [##############                                  ]  28% (14/50)
Processing: [########################                        ]  50% (25/50)
Processing: [################################################] 100% (50/50)
Done! (2.5s)
```

### 4. Spinner — `spinner.sh`

Write a spinner that shows activity while a command runs.

**Requirements:**
- Run an arbitrary command with a spinner
- Usage: `./spinner.sh -- command [args...]`
- Spinner types (selectable with `-s TYPE`):
  - `bars`: `|/-\`
  - `dots`: `⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏` (Unicode braille)
  - `clock`: `🕐🕑🕒🕓🕔🕕🕖🕗🕘🕙🕚🕛` (emoji)
  - `arrow`: `◴◷◶◵`
- Show elapsed time next to spinner: `⠋ (3.2s)`
- Detect keypress to abort: `read -t 0.1 -n 1`
- Show result:
  - If command succeeds: `\e[32m✔ Done!\e[0m`
  - If command fails: `\e[31m✘ Failed (exit N)\e[0m`
  - If aborted: `\e[33m⚠ Aborted\e[0m`
- Trap SIGINT to abort gracefully

**Expected:**
```bash
$ ./spinner.sh -- sleep 3
⠋ (0.5s)
⠙ (1.0s)
⠹ (1.5s)
⠸ (2.0s)
⠼ (2.5s)
✔ Done!

$ ./spinner.sh -s bars -- sleep 1
| (0.2s)
/ (0.4s)
- (0.6s)
\ (0.8s)
✔ Done!

$ ./spinner.sh -- false
✘ Failed (exit 1)

$ ./spinner.sh -- sleep 5
# Press 'q' during execution
⚠ Aborted
```

## Solution Approaches

### Approach A: Interactive-first
1. `login.sh` — simplest, uses `read -s` and colors
2. `color_prompt.sh` — PS1 construction with `\[...\]`
3. `progress.sh` — `\r` and cursor control
4. `spinner.sh` — background process monitoring

### Approach B: Animation-focused
1. `progress.sh` — single-line updates with `\r`
2. `spinner.sh` — background process with animation
3. `login.sh` — colored interactive prompts
4. `color_prompt.sh` — PS1 configuration

### Approach C: Utility-first
1. `color_prompt.sh` — practical, installable
2. `login.sh` — security-oriented
3. `progress.sh` — visual feedback
4. `spinner.sh` — polish

<details>
<summary>Hint 1: Colored PS1 with \[...\]</summary>

```bash
if [[ $EUID -eq 0 ]]; then
    user_color='\[\e[31m\]'   # red for root
else
    user_color='\[\e[32m\]'   # green for user
fi
at_color='\[\e[33m\]'         # yellow @
dir_color='\[\e[34m\]'        # blue directory
prompt_color='\[\e[32m\]'     # green $
reset='\[\e[0m\]'

PS1="${user_color}\u${at_color}@${reset}${dir_color}\h${reset}:${dir_color}\w${reset}${prompt_color}\$ ${reset}"
```
</details>

<details>
<summary>Hint 2: Login with attempt counting</summary>

```bash
#!/bin/bash
USERNAME="admin"
PASSWORD="secret123"
attempts=0
max_attempts=3

while (( attempts < max_attempts )); do
    read -p "Username: " input_user
    read -s -p "Password: " input_pass
    echo

    if [[ -z "$input_user" ]]; then
        echo -e "\e[33m[WARN]\e[0m Username cannot be empty"
        continue
    fi

    if [[ "$input_user" == "$USERNAME" && "$input_pass" == "$PASSWORD" ]]; then
        echo -e "\e[32m[SUCCESS]\e[0m Welcome, $input_user!"
        exit 0
    else
        ((attempts++))
        remaining=$((max_attempts - attempts))
        if (( remaining > 0 )); then
            echo -e "\e[31m[ERROR]\e[0m Invalid credentials ($remaining attempts remaining)"
        else
            echo -e "\e[31m[ERROR]\e[0m Invalid credentials (0 attempts remaining)"
        fi
    fi
done
echo -e "\e[31mAccount locked.\e[0m"
exit 1
```
</details>

<details>
<summary>Hint 3: Progress bar with terminal-aware width</summary>

```bash
#!/bin/bash
total=20
bar_width=40

for i in $(seq 1 "$total"); do
    pct=$((i * 100 / total))
    filled=$((i * bar_width / total))
    bar=$(printf '#%.0s' $(seq 1 "$filled"))
    spaces=$(printf ' %.0s' $(seq 1 $((bar_width - filled))))
    printf '\rProcessing: [%s%s] %3d%% (%d/%d)' "$bar" "$spaces" "$pct" "$i" "$total"
    sleep 0.1
done
echo
printf 'Done! (%ds)\n' "$SECONDS"
```
</details>

<details>
<summary>Hint 4: Spinner with background process</summary>

```bash
#!/bin/bash
spinner() {
    local pid=$1
    local spin='⠋⠙⠹⠸⠼⠴⠦⠧⠇⠏'
    local i=0
    while kill -0 "$pid" 2>/dev/null; do
        printf '\r%c (%ds)' "${spin:i++%${#spin}:1}" "$SECONDS"
        sleep 0.1
    done
}

# Run command in background
"$@" &
pid=$!
spinner $pid
wait $pid
rc=$?

if (( rc == 0 )); then
    printf '\r\e[32m✔ Done!\e[0m\n'
else
    printf '\r\e[31m✘ Failed (exit %d)\e[0m\n' "$rc"
fi
```
</details>

<details>
<summary>Hint 5: Readline-based input with -e</summary>

```bash
# Command history in spinner
read -e -p "Command: " -i "sleep 3" cmd
./spinner.sh -- $cmd
```
</details>

<details>
<summary>Hint 6: Keypress abort for spinner</summary>

```bash
# Use non-blocking read to detect abort key
abort=0
while kill -0 "$pid" 2>/dev/null && (( !abort )); do
    printf '\r%c' "${spin:i++%4:1}"
    read -t 0.1 -n 1 key
    [[ "$key" == "q" ]] && abort=1
done

if (( abort )); then
    kill "$pid" 2>/dev/null
    printf '\r\e[33m⚠ Aborted\e[0m\n'
fi
```
</details>

## Bonus Challenges

1. **Dashboard**: Write `dashboard.sh` that shows a real-time system monitor with `top`-like data using cursor positioning
2. **Typewriter effect**: `typewriter.sh "Hello world"` — prints text one character at a time with configurable speed
3. **Menu system**: Combine `select` with cursor positioning for a full-screen menu with highlighted selections
4. **Terminal color test**: Write `color-test.sh` that tests true color (24-bit), 256-color, and basic ANSI support
5. **Snake game**: Implement a simple terminal game using cursor positioning and `read -n 1` for controls

## Expected Output Summary

```
color_prompt.sh:
  Shows and installs colored PS1
  Uses \[...\] for correct wrapping
  Root/user color differentiation

login.sh:
  Username + password prompts
  3 attempts with countdown
  Colored success/error/warning messages
  Account lockout

progress.sh:
  Real-time progress bar
  Terminal-width aware
  Title, percentage, counter
  Elapsed time at completion

spinner.sh:
  Multiple spinner styles (bars, dots, clock, arrow)
  Elapsed time display
  Keypress abort
  Colored result (success/failure/aborted)
```

## Self-Check

- Why use `printf` over `echo` for colored output?
- What does `\r` do in a printf format string?
- How does `read -s` differ from `read -n 1`?
- What environment variable holds the terminal width?
- Why should you restore terminal settings before exiting?
- What's the purpose of `\[ ... \]` in PS1?
- How does `\e[K` differ from `\r` in clearing a line?
- Why cache `tput` output instead of calling it in a loop?
