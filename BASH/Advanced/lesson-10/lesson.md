# Lesson 10: Exploitation — Privilege Escalation (DEFENSE FOCUSED)

## History & Origins

Privilege escalation (privesc) is the step between gaining limited access and achieving full control. On Unix systems, the traditional privilege boundary is the root user (UID 0). Everything root does is unrestricted; everything a normal user does is subject to permission checks.

The `setuid` (SUID) mechanism was introduced in Unix V7 (1979) as a way to allow ordinary users to run specific programs with elevated privileges. The idea was simple: a binary owned by root with the SUID bit set runs as root regardless of who executes it. This allowed `passwd` (which needs to write to `/etc/shadow`) to work without giving users full root access. SUID was a pragmatic compromise between security and usability.

SGID (setgid) followed the same model for group permissions. The sticky bit (`chmod +t`) on directories was added to prevent users from deleting each other's files in shared directories like `/tmp`.

Linux capabilities (introduced in kernel 2.2, 1999) were designed to break the monolithic root privilege into distinct units. Instead of "are you root? yes/no," capabilities allow fine-grained grants like `CAP_NET_RAW` (ability to create raw sockets) without full root. This was a significant improvement but introduced complexity that many administrators don't fully understand.

The sudo command (1980s) added another layer: configurable privilege elevation without SUID binaries. The `/etc/sudoers` file defines exactly which users can run which commands as which users. Vulnerabilities in sudo configuration are one of the most common privesc vectors.

## Audit Vectors

### SUID/SGID Binaries
```
find / -perm -4000 -type f 2>/dev/null          # Find SUID
find / -perm -2000 -type f 2>/dev/null          # Find SGID
find / -perm -4000 -o -perm -2000 -type f       # Both
```

### Capabilities
```
getcap -r / 2>/dev/null                          # All capabilities
getcap -r /usr/bin/* 2>/dev/null                 # In /usr/bin
```

### Writable Files and Directories
```
find / -perm -0002 -type f 2>/dev/null           # World-writable files
find / -perm -0002 -type d 2>/dev/null           # World-writable dirs
find / -writable -type f 2>/dev/null 2>&1        # Writable by you (bash 4.4+)
```

### Sudo Configuration
```
sudo -l                                          # What sudo commands you can run
sudo -ll                                         # Detailed sudo privileges
```

### Kernel Security Settings
```
cat /proc/sys/kernel/randomize_va_space           # ASLR (0=off, 1=partial, 2=full)
cat /proc/sys/kernel/yama/ptrace_scope            # ptrace restrictions
cat /proc/sys/kernel/core_pattern                 # Core dump handling
cat /proc/sys/kernel/dmesg_restrict               # dmesg access
```

## Under the Hood

### How SUID Works at the Kernel Level

When a process executes a SUID binary, the kernel sets the process's effective UID (eUID) to the file owner's UID. The process retains its real UID (rUID) from the original user.

```
Normal execution:      rUID=1000  eUID=1000
SUID binary (root):    rUID=1000  eUID=0   (effective = root)
```

The kernel checks eUID for permission decisions (file access, signal sending, etc.), while rUID tracks the original user. The `setuid()` system call can reset eUID to rUID (dropping privileges). The saved set-user-ID (sUID) is a copy of eUID before the last exec — it allows toggling between original and elevated privileges.

Important: SUID is IGNORED for shell scripts on most modern Unix systems for security reasons. Linux ignores SUID on scripts with `#!` shebangs because of race conditions and PATH vulnerabilities. Only compiled binaries are reliable SUID carriers.

### Capabilities in the Kernel

Capabilities are implemented as bitmasks in the process's security context:

```
struct cred {
    kernel_cap_t cap_effective;   // Currently effective capabilities
    kernel_cap_t cap_permitted;   // Permitted (can become effective)
    kernel_cap_t cap_inheritable; // Inheritable by children
    kernel_cap_t cap_bset;        // Bound set (capabilities never allowed)
};
```

When a binary has file capabilities (set by `setcap`), the kernel reads them from the extended attributes of the binary file and applies them to the process on exec.

### The Dangers of LD_PRELOAD with Sudo

`LD_PRELOAD` is a dynamic linker feature that loads a shared library before all others. If sudo is configured to preserve `LD_PRELOAD` environment variable (via `env_keep` in sudoers), a user can run a command with an arbitrary shared library that executes code in the elevated context:

```
User process:      LD_PRELOAD=/tmp/evil.so  →  sudo some_command
                                         ↓
Dynamic linker loads /tmp/evil.so first  →  constructor runs as root
```

### Writable PATH Components

When a user runs `ls`, the shell searches each directory in `PATH` in order. If a writable directory appears in PATH before `/usr/bin`, an attacker can place a malicious executable named `ls` there, and it will execute instead of the real `ls`:

```
PATH=/home/user/bin:/usr/local/bin:/usr/bin:/bin
        ^^^^^^^^^^^^^
     Writable by user — can place malicious "ls" here
```

## Core Audit Examples

### Example 1: SUID Discovery

```bash
$ find / -perm -4000 -type f 2>/dev/null
/usr/bin/su
/usr/bin/sudo
/usr/bin/passwd
/usr/bin/mount
/usr/bin/umount
/usr/bin/newgrp
/usr/bin/chsh
/usr/bin/chfn
/usr/bin/gpasswd
/usr/bin/pkexec
/usr/lib/polkit-1/polkit-agent-helper-1
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
```

**Anatomy:**
- These are standard SUID binaries on most Linux systems
- Each has a specific purpose that requires privilege escalation
- The risk is when UNEXPECTED binaries have SUID set

**What if there are unexpected SUID binaries?**
- Custom scripts with SUID (especially dangerous)
- Old versions of libraries or games
- Backup software with SUID
- Any binary in `/home/`, `/tmp/`, `/var/tmp/`

### Example 2: Checking Sudo Permissions

```bash
$ sudo -l
Matching Defaults entries for user on host:
    env_reset, mail_badpass, secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin

User user may run the following commands on this host:
    (ALL : ALL) ALL
    (root) NOPASSWD: /usr/bin/less
    (root) /usr/sbin/service nginx restart
```

**What's dangerous:**
1. `(ALL : ALL) ALL` — user is effectively root via sudo
2. `NOPASSWD: /usr/bin/less` — less can spawn a shell with `:!bash`
3. `(root) /usr/sbin/service` — service runs init scripts as root

**Escape techniques from common sudo-allowed commands:**
- `less` → `:!bash` (shellex)
- `vim` → `:!bash` or `:shell`
- `more` → `!bash`
- `man` → `!bash`
- `awk` → `awk 'BEGIN {system("/bin/bash")}'`
- `find` → `find / -exec /bin/bash \;`
- `tcpdump` → `tcpdump -z /bin/bash -w /tmp/dump`
- `python/perl/ruby` → language shell escapes

### Example 3: Capabilities Discovery

```bash
$ getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/usr/bin/tar = cap_dac_read_search+ep
/usr/bin/mtr = cap_net_raw+ep
/usr/lib/x86_64-linux-gnu/gstreamer-1.0/gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

**Dangerous capabilities:**
- `cap_dac_read_search` — bypass file read permission checks (can read any file)
- `cap_dac_override` — bypass file write permission checks (can write any file)
- `cap_setuid` — set arbitrary UID (become any user)
- `cap_net_raw` — raw sockets (packet crafting)
- `cap_sys_admin` — many administrative operations (near-root)
- `cap_sys_ptrace` — debug any process (inject code)

### Example 4: Writable PATH Detection

```bash
$ IFS=':' read -ra path_dirs <<< "$PATH"
$ for dir in "${path_dirs[@]}"; do
>   if [ -w "$dir" ]; then
>     echo "Writable: $dir"
>   fi
> done
Writable: /home/user/bin
```

**Attack scenario:** If `/home/user/bin` is in PATH before `/usr/bin`, and a privileged script calls `ls` (without full path), the attacker's `ls` in `/home/user/bin` executes instead.

### Example 5: Writable Scripts Owned by Root

```bash
$ find / -type f -perm -0002 -user root 2>/dev/null
/opt/scripts/backup.sh
/tmp/cron_script.sh
```

**Attack scenario:** Root-owned world-writable file + cron job = privesc. An attacker modifies the file, and when root runs it (via cron or manually), the injected code executes as root.

### Example 6: Cron Job Injection

```bash
$ # Check writable cron directories
$ ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/ /var/spool/cron/
$ # Check if we can read /etc/crontab
$ cat /etc/crontab
$ # Check for scripts in cron directories with writable paths
$ for f in /etc/cron*/*; do
>   if [ -f "$f" ] && [ -w "$f" ]; then
>     echo "Writable cron: $f"
>   fi
> done
```

### Example 7: Docker/Container Escape Vectors

```bash
$ # Check if we're in a container
$ cat /proc/1/cgroup | grep -i docker
$ # Check for dangerous mounts
$ mount | grep -E '(docker.sock|/host|/proc)'
$ # Check capabilities within container
$ cat /proc/self/status | grep Cap
$ capsh --decode=$(grep CapEff /proc/self/status | awk '{print $2}')
```

### Example 8: Checking for Unmounted Shares

```bash
$ # Check /etc/fstab for interesting mounts
$ cat /etc/fstab | grep -v '^#' | grep -v '^$'
$ # Check for NFS shares with insecure options
$ showmount -e localhost 2>/dev/null
$ # NFS with 'no_root_squash' allows client to access as root
```

### Example 9: Kernel Exploit Mitigation Check

```bash
$ cat /proc/sys/kernel/randomize_va_space
2  # 2 = full ASLR (best)
$ cat /proc/sys/kernel/yama/ptrace_scope
1  # 1 = restricted ptrace (good)
$ cat /proc/sys/kernel/dmesg_restrict
1  # 1 = non-root can't read dmesg
$ cat /proc/sys/kernel/kptr_restrict
2  # 2 = /proc/kallsyms hidden from non-root
$ cat /proc/sys/kernel/unprivileged_bpf_disabled
1  # 1 = BPF disabled for unprivileged users
```

### Example 10: Service Misconfiguration

```bash
$ # Check for services running as root that shouldn't be
$ ps -eo pid,user,comm | awk '$2 == "root" {print}'
$ # Check for processes with capabilities
$ getpcaps $(pgrep -f 'some_service') 2>/dev/null
$ # Check for SUID wrappers in service directories
$ find /opt /usr/local -type f -perm -4000 2>/dev/null
```

## Automated Audit Script Approach

A comprehensive privesc audit script should systematically check:

1. **SUID/SGID files** — Compare against package manager database to find anomalies
2. **Sudo rules** — Flag dangerous commands (shell escapes, package managers, interpreters)
3. **Capabilities** — Focus on dangerous ones (dac_override, dac_read_search, setuid)
4. **Writable PATH** — Check each PATH component for world-writability
5. **World-writable root files** — Especially scripts and configuration files
6. **Unmounted filesystems** — NFS with no_root_squash, SSHFS
7. **Cron jobs** — Writable scripts, writable cron directories
8. **Kernel security** — ASLR, ptrace_scope, kptr_restrict
9. **Process analysis** — Unusual root processes, exposed sensitive processes
10. **Running services** — Services running as root that don't need to

## Defense Strategy Summary

| Vector | Risk | Detection | Mitigation |
|---|---|---|---|
| SUID scripts | Very High | `find / -perm -4000 -type f` | Remove SUID from scripts; use capabilities |
| Dangerous sudo | High | `sudo -l` audit | Remove dangerous commands; require password |
| Capabilities | Medium | `getcap -r /` | Audit capability grants; remove dangerous ones |
| Writable PATH | Medium | PATH audit | Remove writable dirs from PATH; use absolute paths |
| Cron injection | High | Check cron files | Restrict cron directory permissions |
| Writable root files | Critical | `find / -user root -perm -0002` | Fix permissions; use immutable flag |
| Kernel exploits | Variable | Kernel version check | Patch regularly; enable all mitigations |
| Container escapes | High | Check mounts/capabilities | Drop all non-essential capabilities; run non-root |
| Service misconfig | Medium | Process audit | Run services as dedicated users; drop privileges |

## Why This Matters

Privilege escalation is the step between "limited access" and "full compromise." Most successful attacks use known privesc techniques — not zero-days. The OWASP Top 10 and CWE-250 (Execution with Unnecessary Privileges) both highlight this as a critical vulnerability class.

System administrators who routinely audit these vectors close the most common paths attackers use. A host with no SUID anomalies, properly configured sudo, minimal capabilities, and up-to-date kernel is a host that forces attackers to find harder (and more detectable) methods.

## Memory Aids

- **"SUID = run as owner, regardless of who runs"** — Check which binaries are owner=root with SUID bit.
- **"sudo -l = list your power"** — Always check what sudo allows. `NOPASSWD` is a red flag.
- **"cap_dac_override = read/write any file"** — This capability is nearly equivalent to root for file access.
- **"Writable PATH = Trojan horse"** — If PATH has writable directories, every command invocation is suspect.
- **"cron runs as whatever it wants"** — A writable cron script that runs as root is root access.
- **"The kernel is the last line of defense"** — ASLR, kptr_restrict, and ptrace_scope make exploits harder.

## Trap Vault

1. **SUID on shell scripts is ignored (mostly):** Linux ignores SUID on `#!` scripts for security reasons. But a script called via a SUID binary wrapper (C program that calls `system("./script.sh")`) runs as root.

2. **`sudo` with NOPASSWD is not the only risk:** Password-protected sudo can be abused if the user's terminal is left unlocked, or if the password is captured by a keylogger or `LD_PRELOAD`.

3. **`cap_dac_read_search` applies to the entire filesystem:** A binary with this capability can read `/etc/shadow`, `/root/.ssh/id_rsa`, or any other file. It's not restricted to specific directories.

4. **`getcap` only finds files with extended attributes:** Capabilities stored in the filesystem's extended attributes (`security.capability`). `getcap` reads these. If the filesystem doesn't support xattrs (some embedded systems, older filesystems), capabilities don't work and fall back to SUID.

5. **PATH is set by the SHELL, not the kernel:** A script with SUID doesn't change the PATH. A cron job with a restricted PATH is safer but not immune.

6. **World-writable files owned by root are common in some setups:** `/tmp` files, log files, shared memory. Not every world-writable file is a vulnerability — context matters.

7. **`find -writable` requires bash 4.4+:** On older systems, use `find -perm` with the user's umask. Or use `-readable` and `-exec test -w {} \;`.

8. **NFS `no_root_squash` is extremely dangerous but rarely seen:** It means the root user on the NFS client is treated as root on the NFS server. If you find this in `/etc/exports`, fix it immediately.

9. **`LD_PRELOAD` through `env_keep` in sudoers:** Some sudo configurations preserve environment variables. `LD_PRELOAD` in the environment allows arbitrary code execution. Check `sudo -l` for `env_keep` entries.

10. **Kernel exploits are weaponized quickly after CVE release:** A missing kernel patch is the most dangerous privesc vector. `uname -a` tells attackers exactly which exploit to use. Stay patched.

11. **`/proc/sys/kernel/core_pattern` with a pipe to a script:** If core_pattern starts with `|`, core dumps are piped to a script. If the script is writable, any crash can trigger code execution.

12. **Snap packages have their own SUID mechanism:** `snap` uses mount namespaces and apparmor profiles. A vulnerability in a snap package can escape the snap sandbox and execute on the host.

13. **`pkexec` (PolicyKit) is a common SUID binary:** It's used for GUI privilege escalation. Vulnerabilities in pkexec (like CVE-2021-4034, "PwnKit") have been critical. Keep polkit updated.

14. **`systemd` services running as root:** Many systemd services run as root by default. Check `/etc/systemd/system/*.service` for `User=` not set or set to `root`.

15. **Container capabilities accumulate:** Even with `--cap-drop=ALL`, a container might get capabilities from the runtime. Check with `capsh --print` inside containers.

## See It In The Wild

### System Audit of SUID Binaries
```bash
$ # Find all SUID and check against dpkg database
$ for f in $(find / -perm -4000 -type f 2>/dev/null); do
>   pkg=$(dpkg -S "$f" 2>/dev/null)
>   echo "$f: $pkg"
> done
```

### Check if You Can Write to Any Root-owned Cron Script
```bash
$ find /etc/cron* /var/spool/cron -type f -writable 2>/dev/null
```

### Interactive Shell Test for Dangerous Sudo Commands
```bash
$ while read -r cmd; do
>   if echo "$cmd" | grep -qE '(less|vim|vi|nano|more|man|python|perl|ruby|awk|sed|find|tcpdump|strace)'; then
>     echo "DANGEROUS: $cmd"
>   fi
> done < <(sudo -l 2>/dev/null)
```

### Check if You Can ptrace Other Processes
```bash
$ cat /proc/sys/kernel/yama/ptrace_scope
$ # 0 = anyone can ptrace any process (dangerous)
$ # 1 = restricted (default)
$ # 2 = admin-only (strict)
$ # 3 = no ptrace (maximum security)
```

## Check Your Understanding

1. Why does Linux ignore the SUID bit on shell scripts with `#!` shebangs? What attack would be possible if SUID worked on scripts?

2. What is the difference between `cap_dac_read_search` and `cap_dac_override`? Which is more dangerous?

3. How could a user with `sudo /usr/bin/less` access escalate to a root shell? What technique does `less` provide for shell access?

4. Why is `NOPASSWD` in sudoers a security concern? What additional risk does it add beyond the command itself?

5. How does `LD_PRELOAD` work with sudo to enable privilege escalation? What sudoers setting controls this?

6. What is `no_root_squash` in NFS? Why is it dangerous?

7. How would you audit your system for world-writable files owned by root? What command would you use?

8. What does `cat /proc/sys/kernel/randomize_va_space` tell you about exploit mitigation? What do the values 0, 1, and 2 mean?

9. How does a writable PATH directory enable privilege escalation? Give a scenario involving a cron job.

10. What capabilities, if granted to a binary, are equivalent to full root access? List at least 4.
