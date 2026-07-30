# Task 6: Build a Logging Pipeline

Master redirection by building a pipeline that separates stdout and stderr, uses here-docs, here-strings, tee, and file descriptor manipulation.

## Setup

```bash
$ mkdir -p /tmp/redirect_lab
$ cd /tmp/redirect_lab
```

## Sub-task 1: Create a Dual-Stream Test Script

Create a script that produces both stdout and stderr:

```bash
$ cat > /tmp/test_cmd.sh << 'SCRIPT'
#!/bin/bash
echo "=== Script Starting ==="
echo "Processing data file..."
ls /tmp/datafile.txt 2>&1    # This will fail — no /tmp/datafile.txt
echo "Processing config file..."
cat /etc/nonexistent.conf 2>&1  # This will also fail
echo "=== Script Complete ==="
SCRIPT
$ chmod +x /tmp/test_cmd.sh
```

Run it without redirection first:
```bash
$ /tmp/test_cmd.sh
=== Script Starting ===
Processing data file...
ls: cannot access '/tmp/datafile.txt': No such file or directory
Processing config file...
cat: /etc/nonexistent.conf: No such file or directory
=== Script Complete ===
```

**Question:** Which lines went to stdout and which to stderr? How can you tell?

<details>
<summary>Distinguishing streams</summary>
The `echo` lines go to stdout. The `ls` and `cat` error messages go to stderr. You can verify by redirecting: `./test_cmd.sh > /tmp/out.txt 2> /tmp/err.txt` and checking the contents.
</details>

## Sub-task 2: Separate Streams to Files

Run the script with stdout and stderr in separate files:

```bash
$ /tmp/test_cmd.sh > /tmp/stdout.log 2> /tmp/stderr.log
$ cat /tmp/stdout.log
=== Script Starting ===
Processing data file...
Processing config file...
=== Script Complete ===

$ cat /tmp/stderr.log
ls: cannot access '/tmp/datafile.txt': No such file or directory
cat: /etc/nonexistent.conf: No such file or directory
```

**Question:** Why do the `ls` and `cat` error messages appear in stderr but their "Processing..." echo messages appear in stdout?

<details>
<summary>Stream assignment</summary>
The `echo "Processing..."` lines are explicitly written to stdout by `echo`. The error messages from `ls` and `cat` are written to stderr by those commands. The `2>&1` in the test script redirects stderr to stdout inside the script — wait, I wrote `2>&1` in the test script! Let me check what actually happens.

Actually, looking at the script: `ls /tmp/datafile.txt 2>&1` — this redirects `ls`'s stderr TO its stdout. So the error message goes to stdout too! That means both streams go to stdout. But then `cat /etc/nonexistent.conf 2>&1` also redirects stderr to stdout.

So actually ALL output goes to stdout! The stderr file would be empty. The test script has `2>&1` on each command. To properly separate, we should remove those.
</details>

Let's fix the test script to properly demonstrate separate streams:

```bash
$ cat > /tmp/test_cmd2.sh << 'SCRIPT'
#!/bin/bash
echo "Normal output line 1"
echo "Normal output line 2"
ls /nonexistent_dir
echo "Normal output line 3"
SCRIPT
$ chmod +x /tmp/test_cmd2.sh

$ /tmp/test_cmd2.sh > /tmp/out.log 2> /tmp/err.log
$ cat /tmp/out.log
Normal output line 1
Normal output line 2
Normal output line 3

$ cat /tmp/err.log
ls: cannot access '/nonexistent_dir': No such file or directory
```

## Sub-task 3: Tee for Live View + Save

Run the script and see output while saving to a file:

```bash
# Save stdout, but let errors on screen
$ /tmp/test_cmd2.sh > >(tee /tmp/tee_stdout.log)

# Or save both streams
$ /tmp/test_cmd2.sh 2>&1 | tee /tmp/tee_both.log
```

**Expected output:**
```
$ /tmp/test_cmd2.sh 2>&1 | tee /tmp/tee_both.log
Normal output line 1
Normal output line 2
ls: cannot access '/nonexistent_dir': No such file or directory
Normal output line 3

$ cat /tmp/tee_both.log
Normal output line 1
Normal output line 2
ls: cannot access '/nonexistent_dir': No such file or directory
Normal output line 3
```

**Question:** How would you separately capture stdout and stderr while still seeing both?

<details>
<summary>Separate tee for each stream</summary>
```bash
$ /tmp/test_cmd2.sh > >(tee /tmp/stdout_only.log) 2> >(tee /tmp/stderr_only.log >&2)
```
This uses process substitution: stdout goes to a tee that both writes to file and passes through; stderr goes to another tee that writes to file and passes through to stderr.
</details>

## Sub-task 4: Here-Doc Configuration

Create a config file using a here-document:

```bash
$ cat << EOF > /tmp/myapp.conf
# My Application Configuration
APP_NAME=MyApp
APP_DIR=/opt/myapp
LOG_DIR=$HOME/logs
MAX_CONNECTIONS=100
TIMEOUT=30
EOF

$ cat /tmp/myapp.conf
# My Application Configuration
APP_NAME=MyApp
APP_DIR=/opt/myapp
LOG_DIR=/home/phd/logs    # $HOME was expanded!
MAX_CONNECTIONS=100
TIMEOUT=30
```

Now create the SAME file but with a quoted delimiter:

```bash
$ cat << 'EOF' > /tmp/myapp_literal.conf
# My Application Configuration
APP_NAME=MyApp
APP_DIR=/opt/myapp
LOG_DIR=$HOME/logs
MAX_CONNECTIONS=100
TIMEOUT=30
EOF

$ cat /tmp/myapp_literal.conf
# My Application Configuration
APP_NAME=MyApp
APP_DIR=/opt/myapp
LOG_DIR=$HOME/logs    # $HOME is literal!
MAX_CONNECTIONS=100
TIMEOUT=30
```

**Question:** Which version would you use for a config file template that will be deployed to multiple systems?

<details>
<summary>Template choice</summary>
Use the quoted delimiter (`<< 'EOF'`) for templates. The expanded version hardcodes `/home/phd`, which would be wrong on another system. The literal version keeps `$HOME` as a variable reference for the target system to expand.
</details>

## Sub-task 5: Here-String with bc

Use here-strings for quick calculations:

```bash
$ bc <<< "(5 + 3) * 2 - 1"
15
$ bc <<< "scale=2; 10 / 3"
3.33
$ bc <<< "sqrt(144)"
12
$ bc <<< "obase=16; 255"   # Decimal to hex
FF
```

**Question:** What's the difference between `echo "(5+3)*2-1" | bc` and `bc <<< "(5+3)*2-1"`?

<details>
<summary>Pipe vs here-string</summary>
Functionally identical. The pipe uses a subshell (for `echo`) and a pipe. The here-string is internal to bash (no external command needed to create the string). Both feed the same input to `bc`. The here-string is slightly more efficient.
</details>

## Sub-task 6: File Descriptor Manipulation

Create a script that opens, uses, and closes file descriptors:

```bash
$ cat > /tmp/fd_demo.sh << 'SCRIPT'
#!/bin/bash
# Open fd 3 for writing
exec 3> /tmp/fd_output.txt
echo "First line via fd 3" >&3
echo "Second line via fd 3" >&3
# Close fd 3
exec 3>&-
echo "This goes to normal stdout"
# Verify fd 3 is closed
echo "Trying to write to fd 3..." >&3 2>&1
SCRIPT
$ chmod +x /tmp/fd_demo.sh
$ /tmp/fd_demo.sh
This goes to normal stdout
bash: 3: Bad file descriptor
```

**Expected output:**
```
$ cat /tmp/fd_output.txt
First line via fd 3
Second line via fd 3

$ /tmp/fd_demo.sh
This goes to normal stdout
bash: 3: Bad file descriptor
```

**Question:** Why does the script complain about "Bad file descriptor"? Is this an error or expected?

<details>
<summary>Closing fds</summary>
Expected! After `exec 3>&-`, fd 3 is closed. The attempt to write to it fails with "Bad file descriptor." This is normal and shows that the fd was properly closed. In a real script, you'd organize code so no writes happen after closing.
</details>

## Sub-task 7: The Redirect Order Experiment

Verify the redirect order trap:

```bash
$ /tmp/test_cmd2.sh 2>&1 > /tmp/wrong_order.log
# Which stream appears on the terminal? Which goes to the file?
```

Now compare with the correct order:

```bash
$ /tmp/test_cmd2.sh > /tmp/right_order.log 2>&1
$ cat /tmp/right_order.log
Normal output line 1
Normal output line 2
ls: cannot access '/nonexistent_dir': No such file or directory
Normal output line 3
```

**Question:** Which output appeared on the terminal for the "wrong order" version? Why?

<details>
<summary>Wrong order analysis</summary>
With `2>&1 > /tmp/wrong_order.log`:
1. `2>&1`: stderr goes to where stdout currently is (the terminal).
2. `> /tmp/wrong_order.log`: stdout goes to the file.
Result: stderr -> terminal, stdout -> file. You see the error on screen, but normal output goes to file.
</details>

## Sub-task 8: Discarding Output

Create commands that produce output you want to discard:

```bash
# Discard all output
$ /tmp/test_cmd2.sh > /dev/null 2>&1
$ echo "Exit code: $?"
Exit code: 0

# Discard only errors
$ /tmp/test_cmd2.sh > /tmp/only_out.txt 2> /dev/null
$ cat /tmp/only_out.txt
Normal output line 1
Normal output line 2
Normal output line 3

# Discard only stdout
$ /tmp/test_cmd2.sh 2> /tmp/only_err.txt 1> /dev/null
$ cat /tmp/only_err.txt
ls: cannot access '/nonexistent_dir': No such file or directory
```

**Question:** What exit code does `ls /nonexistent > /dev/null 2>&1` return? Is it 0?

<details>
<summary>Exit code with discarding</summary>
`ls /nonexistent` returns exit code 2 (error) even when output is discarded. Redirecting output doesn't change the exit code. So `ls /nonexistent > /dev/null 2>&1; echo $?` prints `2`.
</details>

## Sub-task 9: Noclobber Protection

Enable noclobber and observe the difference:

```bash
$ set -o noclobber
$ echo "first" > /tmp/protected.txt
$ echo "second" > /tmp/protected.txt
bash: /tmp/protected.txt: cannot overwrite existing file
$ echo "second" >| /tmp/protected.txt   # Force override
$ set +o noclobber                      # Disable noclobber
```

**Question:** Why is `noclobber` not enabled by default? When would you use it?

<details>
<summary>Noclobber use cases</summary>
`noclobber` prevents accidental overwrites, which is great for interactive use. It's not default because many scripts rely on `>` to create or overwrite files, and the option would break them. Use it in interactive shells or safety-critical script sections.
</details>

## Bonus Challenge: The Ultimate Logging Wrapper

Create a script that logs ALL output (both stdout and stderr) to a timestamped file while still showing it on screen:

```bash
$ cat > /tmp/log_wrapper.sh << 'SCRIPT'
#!/bin/bash
LOGFILE="/tmp/script_$(date +%Y%m%d_%H%M%S).log"
echo "All output logged to: $LOGFILE"
exec > >(tee -a "$LOGFILE") 2>&1
echo "This goes to both terminal and log"
ls /nonexistent
echo "So does this"
SCRIPT
$ chmod +x /tmp/log_wrapper.sh
$ /tmp/log_wrapper.sh
All output logged to: /tmp/script_20260731_120000.log
This goes to both terminal and log
ls: cannot access '/nonexistent': No such file or directory
So does this

$ cat /tmp/script_20260731_120000.log
All output logged to: /tmp/script_20260731_120000.log
This goes to both terminal and log
ls: cannot access '/nonexistent': No such file or directory
So does this
```

**Question:** What's the potential issue with `exec > >(tee ...) 2>&1`? What happens to the `tee` process when the script exits?

<details>
<summary>Tee process lifecycle</summary>
The `tee` inside `>(...)` runs as a background process. If the script exits quickly, `tee` might not have finished writing all output. You may need to add a `sleep 0.1` or synchronize with `wait`. Also, the log file path is resolved when the script starts — moving it mid-execution won't affect the running process.
</details>

## Self-Check

1. What's the difference between `>` and `>>`? When would you use each?
2. Why does `2>&1 > file` NOT redirect stderr to the file?
3. What does `command &> /dev/null` do? Is it portable?
4. When would you use `<< 'EOF'` instead of `<< EOF`?
5. What does `tee` do that simple `>` cannot?
6. How would you capture both stdout and stderr to separate files while showing both on screen?
7. What happens when you `exec 3> /tmp/foo` in a script? How is fd 3 different from stdout?
