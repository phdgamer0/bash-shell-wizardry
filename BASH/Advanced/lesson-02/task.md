# Task 2: Named Pipe Chat & Co-process Calculator

## Objective
Build a two-way chat system using named pipes, an interactive calculator using `coproc`, and a raw HTTP client using `/dev/tcp`. Each component exercises a different IPC mechanism. The final integration combines them into a multi-tool communication suite.

## Requirements

1. **Named pipe chat:** Two-way communication between two terminals using two FIFOs
2. **Co-process calculator:** Interactive `bc` wrapper using `coproc`
3. **HTTP via /dev/tcp:** Fetch web page headers and body, extract HTTP status code
4. **Cleanup:** Remove all pipes on exit, handle blocking reads with timeouts
5. **Error handling:** Graceful degradation when `/dev/tcp` or `coproc` features are unavailable

## Sub-tasks (8 cumulative)

### Task 2.1: Single FIFO Proof of Concept
Create a script `fifo_demo.sh` that creates a FIFO, writes a message to it in the background, and reads it in the foreground. Verify that the FIFO is created with the correct permissions.

```bash
#!/bin/bash
pipe=$(mktemp -u /tmp/fifo_demo.XXXX)
mkfifo -m 600 "$pipe"
echo "Hello FIFO" > "$pipe" &
read -r message < "$pipe"
echo "Received: $message"
rm "$pipe"
```

Expected output:
```
Received: Hello FIFO
```

### Task 2.2: Two-Way Chat with Two FIFOs
Build `chat.sh` — a script that:
1. Creates two FIFOs: `/tmp/chat_in` and `/tmp/chat_out`
2. Starts a background reader process that displays incoming messages
3. Reads user input from stdin and writes it to the outbound FIFO
4. Cleanly exits on Ctrl+D or "exit" command
5. Cleans up FIFOs in a trap

**Multiple approaches compared:**

```bash
# Approach A: Two separate FIFOs (simpler to reason about)
pipe_in=/tmp/chat$$.in
pipe_out=/tmp/chat$$.out
trap "rm -f $pipe_in $pipe_out" EXIT
mkfifo "$pipe_in" "$pipe_out"

# Approach B: Single FIFO with bidirectional open (trickier)
# Both ends open the same FIFO with <> but result is undefined
# (whoever reads first gets the data — race condition)
```

**Approach A is correct.** Two FIFOs provide clean bidirectional communication.

**Expected interaction:**
```
Terminal 1: $ ./chat.sh
Terminal 1: (starts reading from pipe_in, waits)
Terminal 2: $ echo "Hello from T2" > /tmp/chat_in
Terminal 1: Hello from T2
```

### Task 2.3: Chat Cleanup and Timeout
Add a `timeout` wrapper to the read loop so that if no message arrives for 60 seconds, the script prints a heartbeat message. Use `read -t`:

```bash
while read -t 60 -r line; do
  echo "[$(date +%H:%M:%S)] $line"
done < "$pipe_in"
echo "Timeout: no messages for 60 seconds"
```

**Why `read -t` is better than `timeout`:** `read -t` is a bash builtin. `timeout cat` forks a process. For a simple timeout on a single read, `read -t` is faster and more efficient.

### Task 2.4: Co-process Calculator — Basic
Build `calc_coproc.sh`:
```bash
#!/bin/bash
coproc BC { bc -l; }
if [[ $? -ne 0 ]]; then
  echo "Failed to start bc" >&2
  exit 1
fi

echo "Calculator ready. Enter expressions, 'q' to quit."
while read -r expr; do
  [[ "$expr" == "q" ]] && break
  echo "$expr" >&${BC[1]}
  read -u ${BC[0]} result
  echo "$result"
done
echo "quit" >&${BC[1]}
wait $BC_PID
```

**TRAP:** The first `read -u ${BC[0]}` blocks forever if `bc` doesn't produce output. Some expressions (like `if`) produce no output. Add a timeout:

```bash
read -t 1 -u ${BC[0]} result || result="(timeout or no output)"
```

### Task 2.5: Co-process Calculator — Advanced
Enhance with:
- Support for multi-line expressions
- Variable preservation across expressions
- Scale setting (`scale=10`)
- Error detection (check if output starts with error message from bc)

```bash
echo "scale=10" >&${BC[1]}  # Set precision
echo "4*a(1)" >&${BC[1]}    # Calculate pi
read -u ${BC[0]} pi
echo "Pi = $pi"
```

### Task 2.6: HTTP Client via /dev/tcp
Build `http_fetch.sh`:
```bash
#!/bin/bash
host="$1"
port="${2:-80}"
path="${3:-/}"

exec 3<>/dev/tcp/$host/$port 2>/dev/null || {
  echo "Cannot connect to $host:$port (/dev/tcp not available)" >&2
  exit 1
}

printf "GET %s HTTP/1.0\r\nHost: %s\r\n\r\n" "$path" "$host" >&3

# Read status line
read -r status <&3
echo "Status: $status"

# Read and display headers
while read -r header; do
  header=${header%$'\r'}  # Strip CR
  [[ -z "$header" ]] && break  # Empty line = end of headers
  echo "Header: $header"
done <&3

# Read body
cat <&3
exec 3<&-
```

**Expected output:**
```bash
$ ./http_fetch.sh example.com
Status: HTTP/1.0 200 OK
Header: Content-Type: text/html
Header: ...
```

**Multiple approaches compared:**

```bash
# Approach A: exec with persistent FD (shown above) — efficient, one connection
# Approach B: Direct to command (less flexible, one-shot)
cat < /dev/tcp/$host/$port <(printf "GET %s HTTP/1.0\r\n\r\n" "$path")
# This doesn't work because /dev/tcp reads before write completes

# Approach C: Use exec with read/write — this is the only correct approach
```

### Task 2.7: Feature Detection
Add feature detection at the top of scripts that use `/dev/tcp` or `coproc`:

```bash
# Test /dev/tcp availability
if ! exec 3<>/dev/tcp/example.com/80 2>/dev/null; then
  echo "ERROR: /dev/tcp not available. Recompile bash with --enable-net-redirections"
  echo "Falling back to curl..."
  # fallback code
fi
exec 3<&-

# Test coproc availability
if ! coproc TEST { true; } 2>/dev/null; then
  echo "ERROR: coproc not available (bash < 4.0)"
  exit 1
fi
```

### Task 2.8: Integration — Multi-Tool Suite
Combine all three tools into a single command-line suite:

```bash
$ ./ipc_tool.sh chat       # Start chat mode
$ ./ipc_tool.sh calc       # Start calculator
$ ./ipc_tool.sh fetch example.com  # Fetch URL
```

Use subcommand dispatch with `case $1 in`.

## Bonus Challenges

1. **Bonus A:** Implement a multi-client chat where 3+ terminals can communicate. Use a central FIFO with a directory of per-client FIFOs.

2. **Bonus B:** Add TLS support to the HTTP client by piping through `openssl s_client -connect`.

3. **Bonus C:** Create a "pipe monitor" that shows the current size of data flowing through a FIFO in real-time (watch `/proc/PID/fd/` offsets).

4. **Bonus D:** Implement a remote-control system: a client sends commands via FIFO to a server that executes them. Add an allowlist of safe commands.

5. **Bonus E:** Build a database REPL that uses coproc to maintain a persistent PostgreSQL or MySQL connection, executing queries interactively.

## Hints

<details>
<summary>Hint 1: Chat two-way pattern</summary>

```bash
# In terminal 1:
mkfifo /tmp/pipe1 /tmp/pipe2
# Read from pipe1, write to pipe2
(while cat /tmp/pipe1; do :; done) &
exec >/tmp/pipe2
# Now type — it goes to /tmp/pipe2, read by terminal 2
```
</details>

<details>
<summary>Hint 2: Coproc read safety</summary>

Always use pessimistic timeout on coproc reads. Some commands buffer output (like `bc` doesn't print anything for `if (1) 2` until the `if` completes).
</details>

<details>
<summary>Hint 3: HTTP newlines</summary>

HTTP requires `\r\n` (carriage return + line feed), not just `\n`. Use `printf` with explicit `\r\n`.
</details>

<details>
<summary>Hint 4: FIFO blocking explanation</summary>

A FIFO open blocks because the kernel's `open()` system call for FIFOs waits until both reading and writing ends are opened. This is a unique behavior — no other file type does this.
</details>

<details>
<summary>Hint 5: Coproc with set -e</summary>

If `set -e` is active, `coproc` failure may exit the script. Use `coproc ... || true` to handle failures gracefully.
</details>

## Expected Output

```bash
$ ./calc_coproc.sh
Calculator ready. Enter expressions, 'q' to quit.
> 2 + 2
4
> scale=5; 22/7
3.14285
> define f(x) { return x^2; }
> f(5)
25
> q
Bye!

$ ./http_fetch.sh example.com
Connecting to example.com:80...
Status: HTTP/1.0 200 OK
Header: Accept-Ranges: bytes
Header: Content-Type: text/html
Header: Content-Length: 1256
[body content...]

$ ./chat.sh --name alice
[alice] Starting chat. Your name: alice
[alice] Connected. Waiting for messages...
[bob] Hello alice!
[alice] Hi bob!
[alice] ^D
[alice] Exiting...
```

## Deep Self-Check

1. **FIFO kernel buffer size:** What happens when a writer writes 2MB to a FIFO but the reader is slow? Use `dd` with `status=progress` to observe. Check the pipe buffer with `fcntl(F_GETPIPE_SZ)` — what's the default?

2. **O_NONBLOCK experiment:** Open a FIFO with `exec 3<>fifo`, then set O_NONBLOCK: what changes? Test by reading from an empty FIFO with and without O_NONBLOCK.

3. **Coproc signal handling:** What signal does the coprocess receive when the parent kills it? Test with `trap` inside the coprocess.

4. **`/dev/tcp` vs `nc` resource usage:** Run `strace -c` on both methods. Count system calls. Which uses fewer?

5. **Multiple writers atomicity:** Launch 5 concurrent writers to a FIFO writing lines of different lengths. Does any line get mixed with another? Check with PIPE_BUF-sized and larger writes.

6. **FIFO security context:** Run `ls -Z /tmp/myfifo` with SELinux. What context is assigned? Would a confined service (httpd_t) be able to write to a FIFO with `user_tmp_t` context?

7. **Coproc with buffered commands:** Some programs buffer output when not connected to a terminal (e.g., `grep` with `--line-buffered`). Which common commands need `stdbuf` or `unbuffer` to work with coproc?

8. **`/dev/tcp` DNS resolution detail:** Trace the DNS lookup for `/dev/tcp/example.com/80` using `strace -e trace=network`. What resolver functions are called?

9. **FIFO persist test:** Write to a FIFO, read part of the data, then close without reading the rest. Then reopen. Is the data still there? (Hint: check with `dd`)

10. **System-wide FIFO impact:** Create 1000 FIFOs in `/tmp`. Does system performance change? Check `ulimit -n` — each open FIFO consumes an FD on both reader and writer.
