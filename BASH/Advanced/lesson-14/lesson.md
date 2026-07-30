# Lesson 14: System Integration — systemd

## History & Origins

systemd was created by Lennart Poettering and Kay Sievers in 2010 as a replacement for SysV init, which had been the standard init system since the 1980s. SysV init used shell scripts in `/etc/init.d/` with symlinks in `/etc/rc?.d/` for runlevel management. It was slow (serial execution), had poor dependency handling, and made no attempt to parallelize boot.

The name "systemd" follows Unix convention of appending "d" for daemon. The rhyme with "System D" (Systeme D — French for "jury-rigging") is intentional: the creators acknowledged it as a pragmatic response to init's limitations.

Key milestones:
- **2010:** Initial release by Red Hat engineers
- **2011:** Fedora 15 adopted systemd as default
- **2012:** Arch Linux, openSUSE, and SUSE Enterprise adopted systemd
- **2014:** Debian 8 (Jessie) voted to adopt systemd — a major controversy in the community
- **2015:** Ubuntu 15.04 adopted systemd (replacing Upstart)
- **2016:** RHEL 7 and CentOS 7 shipped with systemd
- **2020s:** Nearly all major Linux distros use systemd. Exceptions: Alpine (OpenRC), Devuan (sysvinit), Slackware (BSD-style)

Controversy aside, systemd provides:
- Parallel service startup (faster boot)
- On-demand service activation (socket, D-Bus, timer)
- Dependency-based service ordering
- Unified logging (journald)
- Service supervision (auto-restart, resource limits)
- Container-friendly operation (each service can be sandboxed)
- Binary logging with structured metadata

For shell scripters, systemd means you can turn any script into a robust, auto-restarting, logged, scheduled daemon with minimal effort.

## Syntax Reference

```
# Unit file sections and keys
[Unit]
Description=                    # Human-readable description
Documentation=                  # URIs to documentation
After=                          # Start AFTER these units (ordering only)
Before=                         # Start BEFORE these units
Requires=                       # Hard dependency (failure stops this unit)
Wants=                          # Soft dependency (weaker than Requires)
Conflicts=                      # Negative dependency (can't run with these)
ConditionPathExists=            # Only start if path exists (conditional)
ConditionFileNotEmpty=          # Only start if file is non-empty

[Service]
Type=simple|forking|oneshot|notify|dbus|idle
ExecStart=                      # Main command
ExecStartPre=                   # Pre-start commands
ExecStartPost=                  # Post-start commands
ExecStop=                       # Stop command
ExecReload=                     # Reload command (SIGHUP alternative)
Restart=no|on-success|on-failure|on-abnormal|on-watchdog|on-abort|always
RestartSec=                     # Seconds to wait before restart
TimeoutStartSec=                # Max time for startup
TimeoutStopSec=                 # Max time for stop
User=                           # Run as user (drop privileges)
Group=                          # Run as group
WorkingDirectory=               # Chdir to this directory
Environment=                    # Set environment variable
EnvironmentFile=                # Read environment from file (- prefix = optional)
StandardOutput=                 # Where stdout goes (journal, syslog, file)
StandardError=                  # Where stderr goes
LimitNOFILE=                    # File descriptor limit
LimitNPROC=                     # Process limit
Nice=                           # Nice value (-20 to 19)
IOSchedulingClass=              # I/O priority
UMask=                          # File creation mask
ProtectSystem=full|strict|yes   # Sandbox: make /usr and /etc read-only
ProtectHome=yes|read-only|tmpfs # Sandbox: make /home inaccessible
PrivateTmp=yes                  # Sandbox: private /tmp
NoNewPrivileges=yes             # Prevent privilege escalation
CapabilityBoundingSet=          # Limit capabilities

[Install]
WantedBy=                       # Makes unit start when target is reached
RequiredBy=                     # Hard dependency on target
Alias=                          # Additional names for the unit
Also=                           # Also enable/disable these units

# Timer unit sections
[Timer]
OnCalendar=                     # Calendar event (e.g., daily, hourly, Mon *-*-* 03:00:00)
OnBootSec=                      # Run N seconds after boot
OnUnitActiveSec=                # Run N seconds after last activation
OnUnitInactiveSec=              # Run N seconds after last deactivation
Persistent=yes                  # Catch up on missed runs (e.g., after poweroff)
RandomizedDelaySec=             # Randomize start time within window
AccuracySec=                    # How accurate the timer is (default 1min)

# systemd commands
systemctl start NAME            # Start a unit
systemctl stop NAME             # Stop a unit
systemctl restart NAME          # Restart a unit
systemctl reload NAME           # Send SIGHUP
systemctl status NAME           # Show status (with recent logs)
systemctl enable NAME           # Enable at boot
systemctl disable NAME          # Disable at boot
systemctl mask NAME             # Prevent all starts (symlink to /dev/null)
systemctl unmask NAME           # Restore masked unit
systemctl daemon-reload         # Reload unit files after changes
systemctl list-units            # List active units
systemctl list-unit-files       # List all installed unit files
systemctl cat NAME              # Show unit file content
systemctl edit NAME             # Create override snippet
systemctl revert NAME           # Remove overrides
systemd-analyze verify FILE     # Validate unit file
systemd-analyze blame           # Show boot time per unit
systemd-analyze critical-chain  # Show boot bottleneck chain
journalctl -u NAME              # Show logs for a unit
journalctl -u NAME -f           # Follow logs
journalctl -u NAME -n 50        # Last 50 lines
journalctl --since "1 hour ago" # Time-based filter
journalctl _PID=1234            # Filter by PID
journalctl -p err               # Filter by priority
```

## Under the Hood (GO DEEP)

### Systemd Process Lifecycle

When systemd (PID 1) starts a service:

1. **Unit loading:** systemd reads the unit file (and any override snippets in `NAME.d/`). It resolves dependencies (`Wants=`, `Requires=`, `After=`, etc.) and builds a dependency graph.

2. **State transition:** `inactive` → `activating` → `active` → `deactivating` → `inactive`

3. **Process creation for Type=simple:**
   - systemd calls `fork()`
   - Child process: `setsid()` (new session), `setrlimit()` for limits, `setenv()` for environment, `chdir()` to WorkingDirectory
   - Child drops privileges: `setgid()`, `setuid()`, `initgroups()` (if User= is set)
   - Child applies sandboxing: `cap_set_proc()` for CapabilityBoundingSet, `unshare()` for PrivateTmp, `mount()` for ProtectSystem
   - Child `execve()` ExecStart
   - Parent (systemd) immediately considers the service "activating" → "active" (no wait for the process to initialize)

4. **For Type=forking:**
   - Same as simple, but systemd waits for the parent process to exit
   - The service is considered "active" when the parent exits
   - The child process (daemon) continues running as PID N
   - systemd tracks it via the PID file (`PIDFile=`) or via cgroup membership

5. **For Type=oneshot:**
   - systemd waits for the process to exit
   - Then marks the service as "active (exited)"
   - Used for commands that run once and complete (e.g., cleanup, setup)

6. **For Restart=:**
   - systemd monitors the service PID (or cgroup)
   - When the process exits, systemd checks the exit code and signal
   - Based on `Restart=` policy and `StartLimitBurst`/`StartLimitIntervalSec`, it decides whether to restart
   - Before restarting, it waits `RestartSec` seconds

### Cgroup Integration

systemd manages services via cgroups (control groups). Each service gets its own cgroup:
- All processes started by the service (including forked children) belong to this cgroup
- Resource limits (CPU, memory, I/O) are enforced at the cgroup level
- systemd tracks service status via cgroup existence, not PID

This means: even if the main process forks and the parent exits (`Type=forking`), all children remain in the service's cgroup and are tracked.

### Journald Logging

systemd's journal (`journald`) is a binary logging system:
- Logs are stored in `/var/log/journal/` (persistent) or `/run/log/journal/` (volatile)
- Each log entry has structured fields: `_PID`, `_UID`, `_COMM`, `MESSAGE`, `PRIORITY`, `SYSLOG_IDENTIFIER`, `_EXE`, `_CMDLINE`, `_SYSTEMD_UNIT`, etc.
- Advantages over syslog: structured data, binary safe, reliable (no log rotation issues), signed (forward secure sealing)
- Disadvantage: binary format requires journalctl to read; not easily parsed with standard text tools

### What strace Reveals

```bash
# systemd fork/exec for a simple service:
$ sudo strace -p 1 -f -e clone,execve 2>&1 | head -20
clone(child_stack=0, flags=CLONE_VM|CLONE_VFORK|SIGCHLD) = 12345
[pid 12345] execve("/usr/local/bin/myscript.sh", ["/usr/local/bin/myscript.sh"], ...) = 0

# Journalctl reading from journal file:
$ strace journalctl -u myservice -n 1 2>&1 | tail -10
openat(AT_FDCWD, "/var/log/journal/abc123/system.journal", O_RDONLY) = 5
mmap(NULL, 134217728, PROT_READ, MAP_PRIVATE, 5, 0) = 0x7f1234560000
# Journal files are mmap'd for fast access

# systemctl enabling a service:
$ strace -f systemctl enable myservice 2>&1 | grep symlink
symlink("/etc/systemd/system/myservice.service", "/etc/systemd/system/multi-user.target.wants/myservice.service") = 0
# Enabling creates a symlink in the .wants directory
```

### Security Model

**Sandboxing options (from least to most restrictive):**
- `ProtectSystem=full`: /usr and /etc read-only
- `ProtectSystem=strict`: /usr, /etc, /home, /root all read-only (only WorkingDirectory writable)
- `ProtectHome=yes`: /home, /root, /run/user inaccessible
- `ProtectHome=tmpfs`: /home is an empty tmpfs
- `PrivateTmp=yes`: service gets its own /tmp and /var/tmp (via mount namespaces)
- `NoNewPrivileges=yes`: prevents setuid binary escalation
- `CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_DAC_OVERRIDE`: limits kernel capabilities
- `MemoryMax=100M`: memory usage limit (via cgroups)
- `TasksMax=10`: max number of processes

**Key insight:** Sandboxing is not security by default. systemd applies very few restrictions unless you explicitly enable them. A unit file without sandboxing runs as the specified user with full access to everything that user can access.

## Core Examples (15 total)

### Example 1: Simple service unit for a backup script

```bash
$ cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup Service
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh
User=backup
Group=backup
EnvironmentFile=-/etc/default/backup

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** `Type=oneshot` because the script runs and exits. `User/Group=backup` drops privileges. `EnvironmentFile=-/etc/default/backup` loads variables (the `-` prefix means "ignore if missing").

**Variations:** Use `Type=simple` for long-running daemons. Use `Type=forking` for traditional daemons that double-fork.

**Edge case:** On `Type=oneshot`, systemd reports the service as `active (exited)`. This is normal — the service ran successfully and exited.

### Example 2: Timer unit for daily execution at 3 AM

```bash
$ cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Run backup daily at 3am

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
RandomizedDelaySec=60

[Install]
WantedBy=timers.target
EOF
$ systemctl daemon-reload
$ systemctl enable backup.timer
$ systemctl start backup.timer
```

**Anatomy:** `OnCalendar=*-*-* 03:00:00` means "every day at 3:00 AM." `Persistent=true` means "if the system was off at 3 AM, run immediately when it wakes." `RandomizedDelaySec=60` adds up to 60 seconds of jitter to prevent thundering herd.

**Variations:** `OnCalendar=Mon..Fri *-*-* 09:00:00` for weekdays. `OnCalendar=*:0/15` for every 15 minutes. `OnBootSec=5min` for 5 minutes after boot.

**Edge case:** timers are separate UNIT FILES from service files. The timer triggers the service. You enable/start the timer, not the service (the service still needs to exist but is started by the timer).

### Example 3: Journalctl — viewing service logs

```bash
$ journalctl -u backup.service -n 20 --since "yesterday"
```

**Output:**
```
-- Logs begin at Mon 2026-07-01 00:00:00 UTC, end at Wed 2026-07-31 10:00:00 UTC --
Jul 31 03:00:01 host systemd[1]: Starting Daily Backup Service...
Jul 31 03:00:02 host backup.sh[12345]: BACKUP_DIR=/var/backups
Jul 31 03:00:05 host backup.sh[12345]: Backup created: /var/backups/daily-2026-07-31.tar.gz
Jul 31 03:00:05 host backup.sh[12345]: Retention: removed 3 old archives
Jul 31 03:00:05 host systemd[1]: Finished Daily Backup Service.
```

**Anatomy:** systemd logs the start/finish with the service name. The script's stdout goes to the journal (if `StandardOutput=journal`, which is the default). Each line is tagged with the PID that wrote it.

**Variations:** `journalctl -u backup.service -f` for follow mode. `journalctl -u backup.service -o json-pretty` for JSON output.

**Edge case:** By default, journald stores logs in `/run/log/journal/` (volatile — lost on reboot). For persistent storage, create `/var/log/journal/` or set `Storage=persistent` in `/etc/systemd/journald.conf`.

### Example 4: EnvironmentFile usage

```bash
$ cat > /etc/default/backup << 'EOF'
BACKUP_DIR=/var/backups
RETENTION_DAYS=30
LOG_LEVEL=info
COMPRESS=true
EOF

$ cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup

[Service]
Type=oneshot
EnvironmentFile=-/etc/default/backup
ExecStart=/usr/local/bin/backup.sh
EOF
```

**Anatomy:** The backup script can reference `$BACKUP_DIR`, `$RETENTION_DAYS`, etc. directly. `EnvironmentFile=-` with `-` means "don't fail if file doesn't exist."

**Variations:** Multiple `EnvironmentFile=` lines. `Environment=` for inline variables: `Environment=DEBUG=1 LOG_DIR=/var/log`.

**Edge case:** `EnvironmentFile` does NOT support shell variable expansion. You cannot use `$HOME` or `${VAR:-default}` in the environment file. Use `Environment=` for inline values or expand in the script itself.

### Example 5: Restart limits and failure handling

```bash
$ cat > /etc/systemd/system/resilient.service << 'EOF'
[Unit]
Description=Resilient Daemon
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/longrunning.sh
User=myapp
Restart=on-failure
RestartSec=10
StartLimitBurst=5
StartLimitIntervalSec=60

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** If the service exits with non-zero, systemd waits 10 seconds, then restarts. If it fails more than 5 times in 60 seconds, systemd stops trying and marks the service as `failed`.

**Variations:** `Restart=always` restarts even on clean exit (code 0). `Restart=on-abnormal` only on signals. `StartLimitIntervalSec=infinity` means no burst limit (risky — endless restart loop).

**Edge case:** A service that exits immediately with code 0 and has `Restart=always` will restart forever in a tight loop. `StartLimitBurst` prevents this only for failures, not for clean exits with `always`.

### Example 6: Service with multiple lifecycle hooks

```bash
$ cat > /etc/systemd/system/complex.service << 'EOF'
[Unit]
Description=Complex Service

[Service]
Type=oneshot
ExecStartPre=/usr/local/bin/precheck.sh
ExecStart=/usr/local/bin/main.sh
ExecStartPost=/usr/local/bin/notify.sh
ExecStop=/usr/local/bin/cleanup.sh
ExecReload=/usr/local/bin/reload.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** Lifecycle hooks: `Pre` runs before main (if it fails, main doesn't run). `Post` runs after main succeeds. `Stop` runs on `systemctl stop`. `Reload` runs on `systemctl reload`. `RemainAfterExit=yes` means the service stays "active" even after the main process exits.

**Variations:** `ExecStopPost` runs even if stop fails. `ExecReload` replaces SIGHUP handling.

**Edge case:** If `ExecStartPre` fails, the service goes to `failed` state. It does NOT try `ExecStart`. `ExecStop` is still called even if `ExecStart` failed.

### Example 7: Sandboxed service

```bash
$ cat > /etc/systemd/system/sandboxed.service << 'EOF'
[Unit]
Description=Sandboxed App
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/sandboxed_app
User=nobody
Group=nogroup
ProtectSystem=strict
ProtectHome=yes
PrivateTmp=yes
NoNewPrivileges=yes
CapabilityBoundingSet=~CAP_SYS_ADMIN ~CAP_NET_ADMIN

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** `ProtectSystem=strict` makes everything read-only except the service's own directories. `ProtectHome=yes` hides home directories. `PrivateTmp=yes` gives the service its own /tmp. `NoNewPrivileges` prevents any setuid/setgid escalation. `CapabilityBoundingSet` denies dangerous capabilities.

**Variations:** `ProtectKernelTunables=yes` makes sysctl read-only. `ProtectKernelModules=yes` prevents kernel module loading. `SystemCallFilter=@system-service` limits allowed syscalls.

**Edge case:** If your application needs to write to `/etc` or other protected paths, it will fail silently with `ProtectSystem=strict`. Test with `ProtectSystem=full` first (read-only /usr and /etc, but /var and home writable).

### Example 8: systemd-analyze verification

```bash
$ systemd-analyze verify /etc/systemd/system/backup.service
```

**Output (no errors = silent):**
```
# Success — no output means unit file is valid
```

**Output (with errors):**
```
/etc/systemd/system/backup.service:6: Unknown key name 'ExecSturt' for section 'Service', ignoring.
/etc/systemd/system/backup.service:7: Unknown lvalue 'Userne' in section 'Service'
backup.service: Failed to create backup.service/start: Unit backup.service has a bad unit file setting.
```

**Anatomy:** `systemd-analyze verify` parses the unit file and checks for unknown keys, syntax errors, and missing dependencies. Always run this after editing unit files.

**Variations:** `systemd-analyze verify /etc/systemd/system/*.service` verifies all units at once.

**Edge case:** `systemd-analyze verify` checks SYNTAX but not SEMANTICS. A unit file with valid syntax but wrong paths or missing scripts will still verify clean.

### Example 9: systemd-analyze blame

```bash
$ systemd-analyze blame
```

**Output:**
```
5.234s networkd-dispatcher.service
4.101s NetworkManager-wait-online.service
2.345s systemd-resolved.service
1.234s ssh.service
0.456s mybackup.service
```

**Anatomy:** Shows boot time in seconds per service. Useful for identifying boot bottlenecks. The times are for the service's start (not necessarily where the service finished initialization).

**Variations:** `systemd-analyze critical-chain` shows the boot dependency chain. `systemd-analyze plot > boot.svg` generates a graphical timeline.

**Edge case:** Services ordered with `After=` do NOT wait for each other by default — `After=` only controls ordering, not dependency. Use `Requires=` for hard dependencies.

### Example 10: Service override snippets

```bash
$ systemctl edit backup.service
```

This creates ` /etc/systemd/system/backup.service.d/override.conf`:

```ini
[Service]
Environment=EXTRA_FLAGS=--verbose
RestartSec=30
```

**Anatomy:** Override snippets allow modifying system-installed unit files without editing the original file. Snippets take precedence over the original unit. Use `systemctl revert` to remove overrides.

**Variations:** Create manual: `mkdir -p /etc/systemd/system/backup.service.d/ && cat > override.conf << 'EOF' ... EOF`

**Edge case:** If the original unit file is updated by a package update, overrides are preserved. But if the original changes a key that the override doesn't, the override's old value persists.

### Example 11: Template unit files

```bash
$ cat > /etc/systemd/system/worker@.service << 'EOF'
[Unit]
Description=Worker Service %i

[Service]
Type=simple
ExecStart=/usr/local/bin/worker.sh %i
User=worker
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** The `@` in the filename marks this as a template. `%i` is the instance parameter. Start with `systemctl start worker@1.service`, stop with `systemctl stop worker@1.service`.

**Variations:** Multiple instances: `systemctl start worker@{1,2,3}.service`. Use `%j` for the unescaped instance name.

**Edge case:** Template instances are NOT automatically enabled. Enable explicitly: `systemctl enable worker@1.service`.

### Example 12: Path unit — trigger on file change

```bash
$ cat > /etc/systemd/system/reload.path << 'EOF'
[Unit]
Description=Watch config and reload

[Path]
PathModified=/etc/myapp/config.yml
Unit=reload.service

[Install]
WantedBy=multi-user.target
EOF

$ cat > /etc/systemd/system/reload.service << 'EOF'
[Unit]
Description=Reload myapp

[Service]
Type=oneshot
ExecStart=/usr/local/bin/reload_myapp.sh
EOF
```

**Anatomy:** Path units watch a filesystem path. When the file is modified, the linked service is started. Uses inotify internally.

**Variations:** `PathExists=/etc/setup_done` starts service when file appears. `PathChanged=` for write+close events.

**Edge case:** Path units have the same inotify limitations: max watches, per-inode tracking, no NFS support.

### Example 13: Socket-activated service

```bash
$ cat > /etc/systemd/system/echo.socket << 'EOF'
[Unit]
Description=Echo Server Socket

[Socket]
ListenStream=7777
Accept=yes

[Install]
WantedBy=sockets.target
EOF

$ cat > /etc/systemd/system/echo@.service << 'EOF'
[Unit]
Description=Echo Server Connection %i

[Service]
Type=simple
ExecStart=/usr/local/bin/echo_handler.sh
StandardInput=socket
StandardOutput=socket
EOF
```

**Anatomy:** systemd listens on port 7777. For each incoming connection, it spawns a new `echo@.service` instance with stdin/stdout connected to the socket. The service processes one connection and exits.

**Variations:** `ListenStream=0.0.0.0:8080` for specific interface. `Accept=no` for multi-connection services (like nginx).

**Edge case:** `Accept=yes` creates a new process per connection — fine for low-volume, high-latency services. For high-volume, use `Accept=no` and handle multiplexing in your application.

### Example 14: Resource limits

```bash
$ cat > /etc/systemd/system/limited.service << 'EOF'
[Unit]
Description=Resource Limited Service

[Service]
Type=simple
ExecStart=/usr/local/bin/memory_eater.sh
MemoryMax=500M
TasksMax=10
CPUQuota=50%
IOWeight=100
LimitNOFILE=1024
LimitNPROC=20

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** Limits: 500MB max memory, 10 tasks (processes/threads), 50% CPU, low I/O priority, 1024 file descriptors, 20 processes.

**Variations:** `MemoryHigh=400M` for soft limit (throttles but doesn't kill). `MemorySwapMax=0` disables swap for the service.

**Edge case:** Resource limits are enforced via cgroups. They work for the service's entire process tree, not just the main PID. A fork bomb inside the service is contained.

### Example 15: Oneshot service with RemainAfterExit

```bash
$ cat > /etc/systemd/system/setup.service << 'EOF'
[Unit]
Description=System Initialization
Before=myapp.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/init_system.sh
ExecStop=/usr/local/bin/cleanup.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
EOF
```

**Anatomy:** `RemainAfterExit=yes` means the service stays `active` even after the startup script finishes. This allows `ExecStop` to be called on shutdown. Ideal for initialization scripts that have a corresponding teardown.

**Variations:** Without `RemainAfterExit`, the service goes to `inactive (dead)` after the script finishes, and `ExecStop` is never called.

**Edge case:** `ExecStop` is only called if the service was in an active or activating state. If the service is already inactive, `systemctl stop` won't call `ExecStop`.

## Real-World Use Cases

### FOR the OS

- Web servers (nginx, Apache) — systemd service with socket activation
- Databases (PostgreSQL, MySQL) — forking services with PID file tracking
- SSH daemon — socket-activated for on-demand connections
- Time synchronization (chronyd, ntpd) — simple services with restart
- Docker — socket-activated daemon

### WITH the OS

- **Backup scripts:** systemd timer + oneshot service for daily backups
- **Log rotation:** systemd timer for periodic logrotate execution
- **Health checks:** systemd timer runs a health check script, restarts unhealthy services
- **Service orchestration:** systemd service dependencies ensure database starts before web server
- **Container management:** systemd-nspawn containers managed as systemd services

### AGAINST the OS

- Attackers create malicious systemd services for persistence (see Lesson 11)
- Attackers use `Restart=always` to keep backdoors alive
- Attackers exploit weak service sandboxing to escape service confinement
- Attackers modify legitimate service files to include malicious commands

### FOR DEFENSE

- **Sandbox all services:** `ProtectSystem=strict`, `NoNewPrivileges=yes`, `CapabilityBoundingSet=`
- **Read-only unit files:** `chattr +i /etc/systemd/system/*.service`
- **Monitor service changes:** inotify on `/etc/systemd/system/` for new/modified files
- **Auditd for systemd:** `auditctl -w /etc/systemd/system/ -p wa -k systemd_units`
- **Use PrivateTmp:** prevents /tmp-based symlink attacks against services
- **Resource limits:** prevent fork bombs and memory exhaustion from service processes

## Memory Aids

**"UNIT, SERVICE, TIMER"** — The three key concepts:
- **Unit** = any systemd resource type (service, timer, socket, path, mount, etc.)
- **Service** = the most common unit type (run a program)
- **Timer** = schedule for a service (replaces cron)

**"SIMPLE FORKS ONESHOT"** — The three Type= values you'll use 90% of the time:
- **Simple** = long-running daemon, systemd doesn't wait
- **Forking** = traditional double-fork daemon, systemd waits for parent
- **Oneshot** = runs and exits, systemd waits for completion

**"RELOAD EDIT ENABLE"** — The workflow after changing a unit file:
1. **Reload** daemon: `systemctl daemon-reload`
2. **Edit** the unit: `systemctl edit NAME` (or edit file directly)
3. **Enable** if needed: `systemctl enable NAME`

**"VERIFY BEFORE START"** — Always validate before trying to run:
- `systemd-analyze verify /path/to/NAME.service`

## Trap Vault (15 traps)

**Trap 1:** After editing a unit file, you MUST run `systemctl daemon-reload`. systemd caches unit files. Changes are NOT picked up automatically.

**Trap 2:** `Type=simple` means systemd considers the service started as soon as ExecStart forks. If your script backgrounds itself or runs a subprocess, systemd may lose track of the actual daemon process. Use `Type=forking` with `PIDFile=` in this case.

**Trap 3:** `ExecStop` is only called for services that are in `active` state. For `Type=oneshot` WITHOUT `RemainAfterExit=yes`, the service is `inactive (dead)` after the script completes — `ExecStop` is never called.

**Trap 4:** `EnvironmentFile` does NOT support shell expansion. `$VAR`, `${VAR:-default}`, and `$(command)` in the environment file are treated as literal text. Use `Environment=` with proper quoting or expand in your script.

**Trap 5:** Timer units are SEPARATE from service units. You enable/start the timer, not the service. If users run `systemctl start backup.service`, the script runs immediately — but the timer still fires on schedule too.

**Trap 6:** `Restart=always` restart even on clean exit (exit code 0). If your script exits normally but you have `Restart=always`, systemd will restart it forever in a tight loop (unless `StartLimitBurst` kicks in).

**Trap 7:** `User=` drops privileges but does NOT create a new session or reset environment variables. The service still inherits some environment from systemd. Use `Environment=` or `EnvironmentFile=` to set a clean environment.

**Trap 8:** systemd service names are case-sensitive. `backup.service` and `Backup.service` are different units. Always use lowercase for consistency.

**Trap 9:** `ExecStartPre` failures prevent `ExecStart` from running. But `ExecStartPost` still runs (even if `ExecStart` failed) — unless you set `RemainAfterExit=no`.

**Trap 10:** `PrivateTmp=yes` creates a mount namespace for /tmp. Within the service, /tmp appears to be a regular temp directory, but it's isolated. Other processes (including the same service's previous runs) see a DIFFERENT /tmp.

**Trap 11:** `journalctl -u service-name` shows logs from the current boot only by default. Use `--since yesterday` or `-b -1` (previous boot) to see logs from earlier boots.

**Trap 12:** `systemctl disable` removes the symlink from `.wants/` or `.requires/` but does NOT stop the running service. `systemctl stop` is separate from `systemctl disable`.

**Trap 13:** `systemctl mask` creates a symlink to `/dev/null` — the unit is utterly unstartable (even by other units). `systemctl enable` on a masked unit fails silently. Use `systemctl unmask` first.

**Trap 14:** systemd reads unit file paths in order of precedence: `/etc/systemd/system/` > `/run/systemd/system/` > `/usr/lib/systemd/system/`. System-installed units go in the `/usr/lib` path. User modifications go in `/etc/systemd/system/`.

**Trap 15:** `StandardOutput=file:/path/to.log` writes output to the file, but log rotation is your responsibility. systemd does NOT rotate log files created this way. For managed logging, use the default journal output.

## See It In The Wild

- **systemd-timer replacing cron:** Major distros (Fedora, Arch) now recommend systemd timers over cron for periodic tasks. Timers offer calendar syntax, persistent catch-up, random delay, and dependency on other units.

- **Socket-activated SSH:** systemd's `sshd.socket` listens on port 22. SSH is only started when a connection arrives. This saves memory on idle systems and allows seamless restart of sshd without dropping connections.

- **Container zombie reaping:** systemd inside containers (`systemd --user`) automatically reaps zombie processes. This prevents PID 1 zombie accumulation that plagued Docker containers.

- **Service sandboxing in Kubernetes:** Pod security contexts mirror systemd sandboxing concepts: read-only root filesystem, non-root user, capability drops. Understanding systemd sandboxing translates directly to container security.

- **systemd-analyze for boot optimization:** System administrators use `systemd-analyze blame` to identify slow services and enable parallel startup. This is standard practice for optimizing boot times on everything from embedded systems to servers.

## Check Your Understanding (10 questions)

1. **Q:** What's the difference between `Type=simple` and `Type=oneshot`? When would you use each? **A:** `simple` marks the service as started immediately after fork (for long-running daemons). `oneshot` waits for the process to exit (for scripts that run and complete). Use `simple` for servers, `oneshot` for scheduled tasks.

2. **Q:** Why is `Restart=on-failure` preferred over `Restart=always` for most services? **A:** `on-failure` only restarts on non-zero exit codes or abnormal termination. `always` restarts even on clean exit (code 0), which can cause infinite loops if the script exits normally.

3. **Q:** What does `Persistent=true` in a timer unit do, and when is it useful? **A:** It catches up on missed runs. If the system was powered off during a scheduled run, the service runs immediately at boot instead of waiting for the next scheduled time. Critical for daily backups on laptops.

4. **Q:** How does `EnvironmentFile` differ from setting environment variables in `ExecStart`? **A:** `EnvironmentFile` reads variables from a file (key=value). Setting them in `ExecStart` requires prefixing the command: `ExecStart=/bin/sh -c 'VAR=value myscript'`. EnvironmentFile is cleaner and separates config from code.

5. **Q:** What happens if `StartLimitBurst` is exceeded? **A:** systemd stops trying to restart the service. The service transitions to `failed` state. Manual intervention (`systemctl reset-failed NAME` then `systemctl start NAME`) is required to retry.

6. **Q:** Why do you need `systemctl daemon-reload` after editing a unit file? **A:** systemd caches unit files in memory. `daemon-reload` parses all unit files again from disk. Without it, systemctl uses the old cached version.

7. **Q:** What is the difference between `systemctl enable` and `systemctl start`? **A:** `enable` creates symlinks so the service starts at boot. `start` runs the service NOW. They are independent operations — a service can be enabled but not started (it'll start on next boot) or started but not enabled (it's running now but won't survive reboot).

8. **Q:** How does systemd's socket activation save resources? **A:** systemd listens on the socket itself. The service doesn't start until a connection arrives. This means idle services consume zero resources (no process, no memory) until they're actually needed.

9. **Q:** What does `ProtectSystem=strict` do to a service's filesystem view? **A:** It makes `/usr`, `/etc`, `/home`, `/root`, and `/var` (except the service's log directories) read-only. The service can only write to its `WorkingDirectory`, `/tmp` (with `PrivateTmp`), and log paths.

10. **Q:** How do template units (`NAME@.service`) work, and what is `%i`? **A:** Template units define a service pattern. `%i` is the instance identifier. `systemctl start worker@1.service` creates a service instance with `%i=1`. Multiple instances of the same template can run simultaneously.
