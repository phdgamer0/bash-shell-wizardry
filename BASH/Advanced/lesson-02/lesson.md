# Lesson 2: Named Pipes & Co-processes

## History & Origins

Named pipes (FIFOs) were introduced in System III UNIX (1982) as an extension to the anonymous pipe model from Version 3 UNIX (1973). Anonymous pipes only work between related processes (parent-child sharing the pipe via fork). Named pipes extend this to any two processes on the same filesystem, using a file as the rendezvous point.

The FIFO name comes from "First In, First Out" — the data ordering discipline. Unlike a regular file, a FIFO has no seek position; reads consume data and it's gone forever. This makes FIFOs behave more like a stream than a file.

Co-processes (`coproc`) were introduced in bash 4.0 (2009). Before `coproc`, bash had no native way to have bidirectional communication with a subprocess. You had to set up named pipes manually or use `expect`-style tools. The `coproc` builtin was a cleaner, more portable alternative.

The `/dev/tcp` and `/dev/udp` pseudo-devices were added in bash 2.0 (1996) as compile-time options. They provide raw socket access without needing external tools like `nc` or `curl`. Many security-focused distributions disable this feature because it bypasses system firewall rules.

## Syntax Reference

### Named Pipes (FIFOs)
```
mkfifo [-m mode] pipe_name    # Create a named pipe
mkfifo -m 600 /tmp/myfifo     # Create with restricted permissions
rm pipe_name                  # Remove a named pipe
```

### Co-processes
```
coproc [NAME] command          # Run command in background with I/O
coproc [NAME] { command; }     # Braces for compound commands
coproc NAME { cmd1; cmd2; }    # Named co-process (bash 4.0+)
NAME_PID                       # PID of co-process
NAME[0]                        # Read FD from co-process
NAME[1]                        # Write FD to co-process
```

### Special Redirections
```
cat < /dev/tcp/host/port       # Open TCP connection (bash builtin)
cat < /dev/udp/host/port       # Open UDP connection (bash builtin)
exec N<>/dev/tcp/host/port     # Persistent TCP connection
```

### Process Substitution (review)
```
<(command)  # Output of command treated as a file (for reading)
>(command)  # Input to command treated as a file (for writing)
```

## Under the Hood

### FIFO Kernel Mechanics

When you `mkfifo /tmp/myfifo`, the kernel creates an inode with a special FIFO type (`S_IFIFO`). This inode doesn't point to data blocks on disk — it points to a pipe buffer in the kernel's memory space. The pipe buffer size is defined by `/proc/sys/fs/pipe-max-size` (default 1MB on most Linux systems).

```
User Space                    Kernel Space
┌──────────────┐              ┌────────────────────┐
│ Writer       │  write(4)    │ Pipe Buffer (ring)  │
│ bash -c ...  │ ────────────▶│ ┌──┬──┬──┬──┬──┐  │
└──────────────┘              │ │D1│D2│D3│D4│  │  │
                              │ └──┴──┴──┴──┴──┘  │
┌──────────────┐              │ read pos → ← write pos
│ Reader       │  read(3)    └────────────────────┘
│ cat          │ ◀────────────
└──────────────┘
```

Key kernel behaviors:
- When a process writes to a FIFO, the data is copied into the kernel pipe buffer
- When a process reads, data is copied from the buffer to user space and removed
- A write of up to `PIPE_BUF` (4KB on Linux) is guaranteed to be atomic — no mixing with other writers
- Writes larger than `PIPE_BUF` may be interleaved
- Opening a FIFO blocks until both reader and writer are present (unless O_NONBLOCK is used)
- The `struct file` for a FIFO uses `pipefs` operations, not regular filesystem operations

### What strace Shows

Creating a FIFO:
```
mknodat(AT_FDCWD, "/tmp/myfifo", S_IFIFO|0666) = 0
```

Opening a FIFO for reading (blocks):
```
openat(AT_FDCWD, "/tmp/myfifo", O_RDONLY) = 3
# ... blocks here until writer opens ...
```

Opening with O_NONBLOCK:
```
openat(AT_FDCWD, "/tmp/myfifo", O_RDONLY|O_NONBLOCK) = 3
```

Writing 4KB to a FIFO (atomic):
```
write(4, "data...", 4096) = 4096  # Single write, no interleaving
```

Writing 8KB (may be split):
```
write(4, "data...", 8192) = 4096   # First 4KB
write(4, "data...", 4096) = 4096   # Remaining 4KB
```

### Co-process Kernel View

When you `coproc BC { bc -l; }`, bash does:
1. Creates a pipe for stdin of the coprocess (`pipe2()`)
2. Creates a pipe for stdout of the coprocess (`pipe2()`)
3. Forks (`clone()`)
4. In the child: dups pipe ends to stdin/stdout, execs the command
5. In the parent: stores the pipe FDs in the named array

```
bash (parent)                    bc (child)
┌──────────────┐                ┌──────────────┐
│ BC[1] (write)──pipe───────▶   │ stdin (read)  │
│              │                │              │
│ BC[0] (read) ◀───pipe─────   │ stdout (write)│
└──────────────┘                └──────────────┘
```

### `/dev/tcp` Kernel Mechanics

When bash encounters `<>/dev/tcp/host/port`, it does NOT access a device file. Bash's redirection parser recognizes the `/dev/tcp/` prefix and internally calls:

```
socket(AF_INET, SOCK_STREAM, 0) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
```

The `ls -la /dev/tcp` shows nothing because there is no file — it's entirely handled by bash's internal redirection code. This means `strace` shows socket calls where you'd expect file operations.

### Security Model Interactions

**SELinux and FIFOs:** FIFO files have security contexts just like regular files. If a confined domain tries to open a FIFO in a directory with a different context, SELinux may block the operation.

**Capabilities and raw sockets:** Opening `/dev/tcp` requires no special capabilities for client connections. But binding to a privileged port (<1024) requires `CAP_NET_BIND_SERVICE`.

**Namespace isolation:** FIFOs are filesystem objects, so they're confined by mount namespaces. A process in a container can't access a FIFO on the host filesystem unless the paths are shared.

**Network namespaces:** `/dev/tcp` connections use the process's network namespace. In containers with restricted network, TCP connections may fail or be routed through the container's network stack.

## Core Examples

### Example 1: Creating and Using a Named Pipe

```bash
$ mkfifo /tmp/myfifo
$ ls -l /tmp/myfifo
prw-r--r-- 1 user user 0 Jul 31 10:00 /tmp/myfifo
```

The `p` at the beginning of permissions indicates a FIFO (pipe). The file size is 0 because FIFOs don't store data on disk.

```bash
$ echo "hello" > /tmp/myfifo &
[1] 1234
$ cat /tmp/myfifo
hello
[1]+ Done
```

**Anatomy:**
1. `echo "hello" > /tmp/myfifo &` — backgrounded because writing to a FIFO without a reader blocks
2. `cat /tmp/myfifo` — opens the FIFO for reading, which unblocks the writer
3. Data flows: writer → kernel pipe buffer → reader
4. After the data is consumed, both processes see EOF on their respective ends

**What if variations:**
- Without `&`, the `echo` would hang forever waiting for a reader
- Two concurrent readers: one gets the data, the other gets nothing (race condition)
- Two concurrent writers: data may be interleaved if writes exceed PIPE_BUF

**Dangerous edge case:** Writing to a FIFO with no reader blocks indefinitely. In a script, this means the script hangs. Always use `timeout` or background + wait patterns.

### Example 2: Two-Way Chat with Named Pipes

```bash
$ mkfifo /tmp/pipe_A_to_B /tmp/pipe_B_to_A
$ # Terminal 1 (Alice)
$ while read -r line; do echo "Alice: $line"; done </tmp/pipe_B_to_A &
$ exec 3>/tmp/pipe_A_to_B
$ # Terminal 2 (Bob)
$ while read -r line; do echo "Bob: $line"; done </tmp/pipe_A_to_B &
$ exec 3>/tmp/pipe_B_to_A
```

**Anatomy:**
- Two FIFOs enable bidirectional communication
- Each terminal reads from its "in" pipe and writes to its "out" pipe
- The backgrounded `while read` handles incoming messages
- The foreground is used for sending

**Dangerous edge case:** If one process exits without cleaning up, the other may deadlock (write to a FIFO with no reader blocks). Always include cleanup in a trap.

### Example 3: Basic Co-process

```bash
$ coproc BC { bc -l; }
$ echo "3 * 4" >&${BC[1]}
$ read -u ${BC[0]} result
$ echo "$result"
12
$ kill %1
```

**Anatomy:**
1. `coproc BC { bc -l; }` — starts bc in the background with pipes connected
2. `BC[1]` is the write FD (input to bc), `BC[0]` is the read FD (output from bc)
3. `echo "3 * 4" >&${BC[1]}` — sends expression to bc
4. `read -u ${BC[0]} result` — reads bc's response
5. `kill %1` — terminates the co-process

**What if variations:**
- Without `-u` flag on `read`, it reads from stdin instead of the co-process FD
- If bc produces errors, they go to stderr (not captured by co-process pipe)
- Multiple expressions: need newlines and careful read management

**TRAP:** `coproc` without a name defaults to `COPROC`. You can use `COPROC[0]` and `COPROC[1]` but it's clearer to name them.

### Example 4: `/dev/tcp` — Fetch a Web Page

```bash
$ exec 3<>/dev/tcp/example.com/80
$ echo -e "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n" >&3
$ cat <&3
HTTP/1.0 200 OK
...content...
$ exec 3<&-
```

**Anatomy:**
1. `exec 3<>/dev/tcp/example.com/80` — opens TCP connection to example.com:80
2. `echo -e ... >&3` — sends HTTP GET request
3. `cat <&3` — reads HTTP response
4. `exec 3<&-` — closes connection

**What if variations:**
- Use HTTP/1.1 with `Connection: keep-alive` for persistent connections
- Send raw data (not HTTP) to test other TCP services
- Use `/dev/udp` for DNS queries or NTP

**Dangerous edge case:** This feature may be compiled out. In bash 5.x on Debian/Ubuntu, it's usually enabled. On Alpine or hardened systems, it may be disabled. Check with `enable -n test` isn't relevant — test by trying to connect.

### Example 5: Process Substitution vs Named Pipes

Process substitution uses named pipes (or `/dev/fd/`) internally:

```bash
$ diff <(ls /tmp) <(ls /var/tmp)
```

This is equivalent to:

```bash
$ mkfifo /tmp/fifo1 /tmp/fifo2
$ ls /tmp > /tmp/fifo1 &
$ ls /var/tmp > /tmp/fifo2 &
$ diff /tmp/fifo1 /tmp/fifo2
$ rm /tmp/fifo1 /tmp/fifo2
```

Process substitution handles the FIFO creation and cleanup automatically. Named pipes give you explicit control.

### Example 6: Connecting Commands via Named Pipe

```bash
$ mkfifo /tmp/fifo
$ grep ERROR /var/log/syslog > /tmp/fifo &
$ wc -l < /tmp/fifo
42
```

**Anatomy:**
1. `grep` runs in background, writing matching lines to the FIFO
2. `wc -l` reads from the FIFO, counting lines
3. When `wc -l` finishes, `grep`'s write end gets a broken pipe (SIGPIPE)

**What if variations:**
- Swap the commands: `wc -l` in background, `grep` in foreground — still works
- Multiple readers: only one reader gets the data; FIFO is not a multicast

### Example 7: Breaking a Deadlock with timeout

```bash
$ mkfifo /tmp/fifo2
$ timeout 3 cat /tmp/fifo2
$ echo "Done — timeout prevented hang"
```

`timeout` sends SIGTERM to `cat` after 3 seconds, which closes the read end. The kernel then signals SIGPIPE to any writer.

### Example 8: Co-process with Interactive Session

```bash
$ coproc SQL { sqlite3 mydb.db; }
$ echo "CREATE TABLE test (id INT, name TEXT);" >&${SQL[1]}
$ echo ".tables" >&${SQL[1]}
$ read -u ${SQL[0]} tables
$ echo "$tables"
test
$ echo ".quit" >&${SQL[1]}
$ wait $SQL_PID
```

**Anatomy:**
- Interactive programs that read from stdin and write to stdout work well with coproc
- Each command must be a full line (ending with newline)
- Some programs need a flush or a special command to produce output

### Example 9: Named Pipe as a Mutex

```bash
$ mkfifo /tmp/mutex
$ exec 3<>/tmp/mutex          # Open both ends (doesn't block)
$ flock -n 3 && echo "Got lock" || echo "No lock"
$ exec 3<&-
```

A FIFO can be used as a mutex because open with O_RDONLY blocks until another process opens for writing. But `flock` on a regular file is cleaner and more standard.

### Example 10: Multiplexing with Named Pipes and xargs

```bash
$ mkfifo /tmp/parallel_fifo
$ seq 1 10 > /tmp/parallel_fifo &
$ xargs -P 4 -I {} echo "Processing {}" < /tmp/parallel_fifo
$ rm /tmp/parallel_fifo
```

This distributes work across 4 parallel workers through a single FIFO.

### Example 11: FIFO Permissions and Security

```bash
$ mkfifo -m 600 /tmp/secure_pipe
$ ls -l /tmp/secure_pipe
prw------- 1 user user 0 Jul 31 10:00 /tmp/secure_pipe
$ # Only the owner can read/write
```

**Security implication:** FIFO permissions are checked at open time. If a FIFO is world-readable, any user can read from it. Always set restrictive permissions on FIFOs carrying sensitive data.

### Example 12: Detecting EOF on a Named Pipe

```bash
$ mkfifo /tmp/eof_test
$ (echo "data"; sleep 1; echo "more") > /tmp/eof_test &
$ while read -r line; do echo "Got: $line"; done < /tmp/eof_test
Got: data
Got: more
$ # Reader exits when writer closes (gets EOF)
$ rm /tmp/eof_test
```

A FIFO delivers EOF when the writer closes. If the writer opens again, new data arrives. This is unlike pipes from process substitution, which deliver EOF once.

### Example 13: Co-process Without Named Array (Bash 4.3+)

```bash
$ coproc { bc -l; }
$ echo "2+2" >&${COPROC[1]}
$ read -u ${COPROC[0]}
$ echo "$REPLY"
4
$ kill $COPROC_PID
```

Without a name, bash uses `COPROC` and `COPROC_PID`. Works identically to named, but can be confusing in scripts with multiple co-processes.

### Example 14: `/dev/tcp` vs netcat Performance

```bash
$ # Using bash builtin
$ time exec 3<>/dev/tcp/example.com/80; echo -e "GET / HTTP/1.0\r\n\r\n" >&3; cat <&3 >/dev/null; exec 3<&-
$ # Using netcat
$ time echo -e "GET / HTTP/1.0\r\n\r\n" | nc -w 3 example.com 80 >/dev/null
```

The bash builtin avoids forking an external process. For many connections, this is faster. But `nc` has more features (SSL, proxying, listening mode).

### Example 15: Pipeline with Named Pipe Buffer

```bash
$ mkfifo /tmp/buffer
$ # Producer (fast)
$ for i in {1..100000}; do echo "line $i"; done > /tmp/buffer &
$ # Consumer (slow)
$ while read -r line; do echo "Consumed: $line"; sleep 0.001; done < /tmp/buffer
$ rm /tmp/buffer
```

The kernel pipe buffer absorbs speed differences between producer and consumer. If the consumer is slower, the buffer fills and the producer's write blocks. If the consumer is faster, it waits for data.

## Real-World Use Cases

### FOR the OS
- **Syslog:** Traditional syslog daemons use FIFOs (`/dev/log`) to receive log messages from all processes
- **init scripts:** FIFOs coordinate startup order between services
- **named pipe filesystem:** FUSE-based filesystems can expose data as FIFOs

### WITH the OS
- **Log streaming:** `tail -f /var/log/syslog > /tmp/log_fifo &` with a processing pipeline reading the FIFO
- **Database batch processing:** Use coproc with `sqlite3` or `psql` for efficient batch operations
- **Parallel data processing:** Split data across FIFOs to multiple worker processes

### AGAINST the OS (defense perspective)
- **Covert channels:** Attackers can use FIFOs to tunnel data between processes that shouldn't communicate
- **Co-process injection:** An attacker who gains write access to a co-process pipe can inject commands into the interactive subprocess
- **`/dev/tcp` backdoor:** Even with outbound firewall rules, if bash has `/dev/tcp` compiled in, an attacker can make outbound connections that bypass netfilter (since they originate from the shell, not a separate process)

### FOR DEFENSE
- **Disable `/dev/tcp`:** Compile bash with `--disable-net-redirections` on sensitive systems
- **FIFO permission auditing:** Regularly scan for world-readable/writable FIFOs with `find / -type p -perm -0777`
- **Coproc monitoring:** Use auditd to monitor when processes create co-processes (rare event = suspicious)
- **Capability restrictions:** Remove `CAP_NET_BIND_SERVICE` from bash to prevent binding privileged ports via `/dev/tcp`

## Memory Aids

- **FIFO = File + Pipe:** Think of it as a pipe that exists as a file. It's like a tunnel entrance that stays in place even when no one is using it.
- **"Block means wait" for FIFOs:** Opening a FIFO blocks until both ends are connected. It's like two people arriving at opposite ends of a tunnel — neither can enter until the other is there.
- **coproc = Co-routine + Process:** It's a subprocess you can have a conversation with. Like a chatbot in your script.
- **`/dev/tcp` — No Device Behind the Curtain:** It's pure bash magic, not a real device. The name is a convenient lie.
- **PIPE_BUF = 4096 atom:** Think of it as "one write fits in one truck." Larger writes need multiple trips and may arrive intermixed.

## Trap Vault

1. **Write to FIFO with no reader = deadlock:** `echo "data" > /tmp/fifo` blocks until a reader opens the FIFO. If your script both writes and reads the same FIFO sequentially, it deadlocks.

2. **Read from FIFO returns EOF when writer closes:** After all writers close, `read()` returns 0 (EOF). Your loop exits. To keep reading, the writer must reopen or not close.

3. **FIFO permissions checked at open, not at write:** If you change permissions on a FIFO after it's opened, existing opens are unaffected. Only new opens check the new permissions.

4. **`/dev/tcp` not available on all systems:** Some distros compile bash without network redirections. Others have it. Never assume it's available. Check with `enable | grep /dev/tcp` — though that won't show it; just test a connection.

5. **Co-process FDs are bidirectional? No:** `coproc` creates TWO pipes — one for stdin, one for stdout. `NAME[0]` is read (from the coprocess's stdout), `NAME[1]` is write (to the coprocess's stdin). They are NOT the same FD.

6. **Bash 4.0 vs 4.3 coproc syntax:** In bash 4.0, `coproc NAME command` uses a simple command. Bash 4.3+ requires `coproc NAME { command; }` (braces) for all cases. The old simple-command form still works but is deprecated.

7. **FIFO deadlock with the same process:** Writing to and reading from the same FIFO in the same process deadlocks because both ends are the same process. Use two FIFOs or a bidirectional open (`<>`).

8. **`coproc` and `set -m`:** If job control is disabled (non-interactive shell), coproc may not work correctly. Use `set -m` (monitor mode) to enable job control in scripts.

9. **`read -u` without coproc:** `read -u` works on any FD. Using it with a regular file reads one line. Using it with a pipe or FIFO may block waiting for data.

10. **FIFO not removed after crash:** Unlike anonymous pipes, named pipes persist in the filesystem. A crashed script leaves FIFO files behind. Next run may fail if the old FIFO still exists and contains stale data.

11. **`/dev/tcp` DNS resolution:** Bash resolves hostnames using the system resolver (glibc's `gethostbyname` or `getaddrinfo`). If DNS is broken, `/dev/tcp` connections fail. No way to specify a custom DNS server.

12. **coproc read blocking:** `read -u ${COPROC[0]}` blocks forever if the co-process never produces output. Always use `timeout` or `read -t` (timeout) for safety.

13. **FIFO and `cat` for large data:** `cat /tmp/fifo` reads until EOF. If the writer writes 1GB, `cat` reads it all into memory (or at least through the kernel pipe buffer). The writer is limited by the pipe buffer size for burst writes but total throughput is bounded by memory bandwidth.

14. **`mkfifo` with existing file:** `mkfifo` fails if the file already exists. Always check first: `[[ -p /tmp/fifo ]] || mkfifo /tmp/fifo`.

15. **Coproc stderr not captured:** The co-process pipe only captures stdout. Stderr goes to the terminal (or wherever the parent's stderr is redirected). To capture stderr, redirect it explicitly: `coproc { cmd 2>&1; }`.

## See It In The Wild

### Exploring FIFO limits
```bash
$ cat /proc/sys/fs/pipe-max-size
1048576
$ cat /proc/sys/fs/pipe-user-pages-soft
16384
$ cat /proc/sys/fs/pipe-user-pages-hard
32768
```

### Watching FIFO activity with strace
```bash
$ strace -e trace=openat,read,write,close \
  bash -c 'mkfifo /tmp/t; exec 3<>/tmp/t; echo x >&3; read y <&3; rm /tmp/t'
```

### Listing FIFOs on the system
```bash
$ find / -type p -ls 2>/dev/null
```

### Checking if bash has `/dev/tcp`
```bash
$ bash -c 'echo test > /dev/tcp/example.com/80' 2>&1
# If you get "No such file or directory", it's disabled
# If you get "Connection refused" or timeout, it's enabled but blocked
```

### Co-process in a real tool
Check your system for scripts using coproc:
```bash
$ grep -r 'coproc' /usr/bin/ 2>/dev/null
$ grep -r 'coproc' /etc/ 2>/dev/null
```

## Check Your Understanding

1. Why does opening a FIFO for writing (`exec 3>fifo`) block until a reader opens? What happens in the kernel?

2. What is PIPE_BUF and why does it matter when multiple processes write to the same FIFO?

3. How does `coproc` differ from running a command with `&` in the background and using redirection?

4. Why would a system administrator disable `/dev/tcp` in bash? What attacks does this prevent?

5. What happens to data in a FIFO when no process has it open? Does it persist?

6. Write the equivalent of `diff <(cmd1) <(cmd2)` using explicit named pipes. What are the differences in behavior?

7. If a co-process reads from stdin but never writes to stdout, what does `read -u ${NAME[0]}` do?

8. How does `timeout` prevent FIFO deadlocks? What exactly happens in the kernel when the timeout fires?

9. Why is `exec 3<>fifo` (bidirectional open) useful? When would it cause problems?

10. How would an attacker use `/dev/tcp` to bypass outbound firewall rules? What's the defense?
