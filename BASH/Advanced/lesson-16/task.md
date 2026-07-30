# Task 16: Port Scanner & HTTP Health Check Toolkit

## Objective

Build a network toolkit with three main components: a fast concurrent port scanner using `/dev/tcp`, a URL status checker with timing, and an HTTP health check with response validation. The toolkit must handle errors gracefully and support multiple output formats.

## Requirements (8 sub-tasks)

### Sub-task 1: Port Scanner (Sequential)
Write `port_scan.sh` that:
- Takes HOST and PORT_RANGE arguments (e.g., `port_scan.sh example.com 1-1000`)
- Scans ports sequentially
- Reports OPEN or CLOSED per port
- Uses `timeout 2` per connection attempt
- Distinguishes between "closed" (connection refused) and "filtered" (timeout)
- Reports elapsed time and scan rate (ports/sec) at the end

### Sub-task 2: Port Scanner (Concurrent)
Write `fast_port_scan.sh` that:
- Same interface but uses background jobs for parallelism
- Supports a `--max-parallel N` option (default 100)
- Uses a job queue: fill to N, wait for one to finish, start next
- Reports results in order (sorted by port number)
- Prevents resource exhaustion (sockets, ephemeral ports)
- Has `--top-20` option to scan only the most common ports

### Sub-task 3: URL Status Checker
Write `url_checker.sh` that:
- Reads a list of URLs from a file or stdin
- Checks each with `curl` 
- Reports: URL, HTTP status code, response time (seconds), size (bytes)
- Shows a progress bar during scanning
- Reports counts: OK (2xx), Redirect (3xx), Client Error (4xx), Server Error (5xx), Failed
- Outputs in aligned columns (like a table)

### Sub-task 4: HTTP Health Check
Write `http_health.sh` that:
- Takes HOST, PORT, PATH, and EXPECTED_CONTENT arguments
- Connects via `/dev/tcp`
- Sends HTTP/1.1 GET with proper headers
- Parses status line, headers, and body
- Checks for expected content in the response body
- Reports: status line, body length, content match (PASS/FAIL), response time
- Uses `timeout 5` for safety
- Returns 0 if 200 + content matches; 1 otherwise

### Sub-task 5: Service Banner Grabber
Write `banner_grab.sh` that:
- Takes HOST and PORTS list
- For each port, connects and reads any service banner
- Handles services that need a probe (HTTP: send GET, SMTP: send EHLO, FTP: send empty line)
- Reports service identity (SSH version, HTTP server, SMTP banner)
- Uses `timeout 3` per service
- Falls back to "no banner" if nothing received within 2 seconds

### Sub-task 6: Network Diagnostics
Write `net_diag.sh` that:
- Runs a battery of network diagnostics and reports results
- Checks: DNS resolution (dig/getent), ping, TCP connectivity to key ports (80, 443, 22)
- Measures latency for each test
- Detects: DNS failures, packet loss, firewall filtering, MTU issues
- Outputs a concise "health score" (0-100) with explanation
- All diagnostics run with timeouts (no hanging)

### Sub-task 7: Logging and Reporting
All scripts must:
- Support `--verbose` flag for detailed output
- Support `--json` flag for machine-readable JSON output
- Support `--log FILE` to write detailed log to file
- Use consistent exit codes: 0 = success, 1 = some failures, 2 = no targets reachable

### Sub-task 8: Integration Test Harness
Write `test_net_tools.sh` that:
- Tests each component against known hosts (example.com, google.com)
- Tests error handling (invalid hosts, closed ports, timeouts)
- Tests JSON output parsing
- Tests with and without verbose mode
- Reports code coverage (how many error paths were tested)
- Reports PASS/FAIL per test case

## Bonus Challenges

1. **Service fingerprinting:** Extend the banner grabber to identify services by response patterns. Map banners to service names and versions.

2. **Network discovery:** Given a subnet, scan all hosts for open ports. Output a network topology map.

3. **Custom protocol:** Write a client for a non-HTTP protocol (like Redis, Memcached, or SMTP) using /dev/tcp.

4. **Traffic shaping:** Add rate limiting to the port scanner (max N packets/sec) to avoid triggering IDS/IPS systems.

5. **Asynchronous DNS:** Use `dig +short` in parallel for the port scanner to resolve hostnames faster.

## Hints

<details>
<summary>Hint 1: Concurrent port scanning with job control</summary>

```bash
#!/bin/bash
HOST="$1"
START="${2:-1}"
END="${3:-1024}"
MAX_PARALLEL="${MAX_PARALLEL:-100}"
ACTIVE=0

scan_port() {
  local port="$1"
  timeout 2 bash -c "exec 3<>/dev/tcp/$HOST/$port" 2>/dev/null && echo "OPEN:$port"
}

for port in $(seq "$START" "$END"); do
  scan_port "$port" &
  ACTIVE=$((ACTIVE + 1))
  if [ "$ACTIVE" -ge "$MAX_PARALLEL" ]; then
    wait -n  # Wait for any one job to finish
    ACTIVE=$((ACTIVE - 1))
  fi
done
wait  # Wait for remaining jobs
```
</details>

<details>
<summary>Hint 2: HTTP health check with /dev/tcp</summary>

```bash
#!/bin/bash
health_check() {
  local host="$1" port="${2:-80}" path="${3:-/}"

  START=$(date +%s%N)
  exec 3<>/dev/tcp/$host/$port 2>/dev/null || return 2

  printf "GET %s HTTP/1.0\r\nHost: %s\r\n\r\n" "$path" "$host" >&3

  read -r status <&3
  length=$(wc -c <&3 2>/dev/null)
  END=$(date +%s%N)
  ELAPSED=$(echo "scale=3; ($END - $START) / 1000000000" | bc)

  exec 3<&-

  echo "Status: $status"
  echo "Body length: $length bytes"
  echo "Response time: ${ELAPSED}s"

  if echo "$status" | grep -q "200"; then
    return 0
  fi
  return 1
}
```
</details>

<details>
<summary>Hint 3: URL checker with timing</summary>

```bash
#!/bin/bash
check_url() {
  local url="$1"
  local start="$EPOCHREALTIME"
  local result=$(curl -s -o /dev/null -w \
    '%{http_code}|%{time_total}|%{size_download}|%{time_namelookup}|%{time_connect}' \
    --connect-timeout 5 --max-time 10 "$url" 2>/dev/null)
  local end="$EPOCHREALTIME"

  IFS='|' read -r code time_total size dns tcp <<< "$result"

  if [ -z "$code" ]; then
    printf "%-50s %-6s %6s %8s\n" "$url" "FAIL" "-" "-"
  else
    printf "%-50s %-6s %5.2fs %8d\n" "$url" "$code" "$time_total" "$size"
  fi
}

[ $# -eq 0 ] && set -- "https://example.com" "https://google.com"
for url in "$@"; do check_url "$url"; done
```
</details>

<details>
<summary>Hint 4: Service banner grabbing</summary>

```bash
grab_banner() {
  local host="$1" port="$2"
  local banner=""

  case "$port" in
    80|8080)
      banner=$(timeout 3 bash -c 'exec 3<>/dev/tcp/'"$host"'/'"$port"'; printf "GET / HTTP/1.0\r\nHost: '"$host"'\r\n\r\n" >&3; head -1 <&3' 2>/dev/null)
      ;;
    22)
      banner=$(timeout 3 bash -c 'exec 3<>/dev/tcp/'"$host"'/'"$port"'; read -r line <&3; echo "$line"' 2>/dev/null)
      ;;
    25)
      banner=$(timeout 3 bash -c 'exec 3<>/dev/tcp/'"$host"'/'"$port"'; read -r line <&3; echo "EHLO test" >&3; read -r line2 <&3; echo "$line $line2"' 2>/dev/null)
      ;;
    *)
      banner=$(timeout 2 bash -c 'exec 3<>/dev/tcp/'"$host"'/'"$port"'; read -r -t 1 line <&3; echo "$line"' 2>/dev/null)
      ;;
  esac

  if [ -n "$banner" ]; then
    echo "$port: $banner"
  else
    echo "$port: no banner"
  fi
}
```
</details>

<details>
<summary>Hint 5: --top-20 ports list</summary>

```bash
TOP20_PORTS=(21 22 23 25 53 80 110 111 135 139 143 443 445 993 995 1433 1521 2049 3306 3389 5432 5900 6379 8080 8443)
```
</details>

<details>
<summary>Hint 6: JSON output function</summary>

```bash
json_output() {
  local host="$1" port="$2" status="$3"
  printf '{"host":"%s","port":%d,"status":"%s"}\n' "$host" "$port" "$status"
}
```
</details>

<details>
<summary>Hint 7: Rate limiting the scanner</summary>

```bash
scan_with_rate_limit() {
  local host="$1" start="$2" end="$3" rate="${4:-100}"
  local delay=$(echo "scale=3; 1 / $rate" | bc)
  for port in $(seq "$start" "$end"); do
    timeout 1 bash -c "exec 3<>/dev/tcp/$host/$port" 2>/dev/null && \
      echo "OPEN: $port" &
    sleep "$delay"
  done
  wait
}
# Note: This limits connections per second but doesn't prevent OS socket buffer exhaustion
```
</details>

## Expected Output

```bash
$ ./fast_port_scan.sh scanme.org --top-20 --json
{"host":"scanme.org","port":22,"status":"open"}
{"host":"scanme.org","port":80,"status":"open"}
{"host":"scanme.org","port":443,"status":"filtered"}
Scan complete: 3 open, 1 filtered, 20 closed in 2.3s (10.0 ports/sec)

$ ./url_checker.sh urls.txt --table
URL                                      CODE   TIME    SIZE
https://google.com                       200    0.15s   14523
https://example.com                      200    0.08s   1256
https://httpstat.us/404                  404    0.12s   256
https://httpstat.us/500                  500    0.14s   178
https://nonexistent.example.test         FAIL   5.01s   -

Summary: 2 OK, 0 redirect, 1 client error, 1 server error, 1 failed

$ ./http_health.sh example.com 80 / "Example Domain"
Status: HTTP/1.1 200 OK
Body length: 1256 bytes
Content check: Example Domain — FOUND
Response time: 0.09s
Health: ✅ PASS

$ ./http_health.sh example.com 80 / "NonExistentText"
Status: HTTP/1.1 200 OK
Body length: 1256 bytes
Content check: NonExistentText — NOT FOUND
Response time: 0.09s
Health: ❌ FAIL (content mismatch)

$ ./banner_grab.sh example.com 22 80 443 3306
22: SSH-2.0-OpenSSH_9.0
80: HTTP/1.1 200 OK
443: (TLS — cannot read banner via /dev/tcp)
3306: no banner (timeout)
```

## Self-Check Questions

1. Why is `/dev/tcp` port scanning slower than `nmap`? What's the difference in approach?

2. How does `timeout` prevent the script from hanging on unresponsive hosts?

3. What is the difference between HTTP/1.0 and HTTP/1.1 in raw requests over `/dev/tcp`?

4. Why should background jobs in a port scanner use `wait -n` or `wait` for synchronization?

5. How would you detect that `/dev/tcp` is unavailable at compile time in a script?

6. Why can't `/dev/tcp` be used for HTTPS connections?

7. How does increasing `--max-parallel` affect port scanning performance? What limits it?

8. What's the difference between "closed" (connection refused) and "filtered" (timeout) in port scanning?
