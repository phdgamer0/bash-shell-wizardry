# Lesson 15: System Integration — Advanced (udev, inotify, dbus)

## History & Origins

**udev** was created in 2003 by Greg Kroah-Hartman to replace `devfs` and `devfsd` in the Linux kernel. The old system used a static `/dev` directory with every possible device node pre-created (tens of thousands of entries). udev dynamically creates device nodes only for devices actually present, using uevents from the kernel. It runs in userspace (as `udevd`), receiving events via netlink.

The name "udev" stands for "userspace dev." Before udev, device management was split between the kernel (`devfs`) and userspace scripts. udev unified this: the kernel sends events, and udev processes rules written in a flexible format that can match on device attributes, set permissions, create symlinks, and run scripts.

**inotify** replaced `dnotify` in Linux 2.6.13 (2005). Dnotify was clunky: it required the directory to be opened, used signals for notification, and was per-directory only. inotify uses file descriptors, supports per-file watches, and provides event information including the filename for directory watches. John McCutchan, Robert Love, and Amy Griffis were primary authors.

The key difference from polling: with polling (`find -newer`), you're constantly checking for changes, wasting CPU. With inotify, the kernel tells you exactly when a change happens. It's event-driven, not poll-based.

**D-Bus** (Desktop Bus) was created by Red Hat in the early 2000s as part of the freedesktop.org project. It replaced earlier IPC mechanisms like CORBA and DCOP (KDE). D-Bus provides a low-latency, low-overhead message bus system with:
- **System bus** — for system-wide messages (hardware events, network manager)
- **Session bus** — per-user-session messages (desktop notifications, media players)

D-Bus messages use a binary protocol and follow an object-oriented interface definition. While typically used via libraries (libdbus, GDBus, QtDBus), you can send and receive D-Bus messages from bash using `dbus-send` and `dbus-monitor`.

## Syntax Reference

```
# udev
udevadm monitor [--property] [--kernel] [--udev]     # Monitor uevents
udevadm info -a -p /sys/class/...                     # Query device attributes
udevadm control --reload-rules                         # Reload udev rules
udevadm trigger [--type=devices] [--action=change]     # Trigger uevents
udevadm test /sys/...                                  # Test a device's matching
udevadm settle                                         # Wait for udev queue to empty

# udev rule syntax (files in /etc/udev/rules.d/ or /usr/lib/udev/rules.d/)
# Format: KEY==value for match, KEY="value" for assignment
SUBSYSTEM=="block", ACTION=="add", RUN+="/usr/bin/myscript.sh %k"
ATTR{size}=="media", ENV{ID_FS_LABEL}=="USB*", SYMLINK+="myusb"

# inotify (inotify-tools)
inotifywait -m -e modify,create,delete /path/to/watch
inotifywait -m -r /path --format '%w%f %e %T' --timefmt '%H:%M:%S'
inotifywatch -t 60 /path                                # Stats for 60 sec

# inotify event masks
IN_ACCESS       # File was read
IN_MODIFY       # File was written
IN_ATTRIB       # Metadata changed (permissions, timestamps)
IN_CLOSE_WRITE  # File was closed after writing
IN_OPEN         # File was opened
IN_MOVED_FROM   # File was moved out of watched dir
IN_MOVED_TO     # File was moved into watched dir
IN_CREATE       # File was created in watched dir
IN_DELETE       # File was deleted from watched dir
IN_DELETE_SELF  # Watched file/directory was deleted
IN_MOVE_SELF    # Watched file/directory was moved
IN_CLOSE_NOWRITE # File was closed (without write)

# D-Bus
dbus-send --system --type=method_call --dest=org.freedesktop.NetworkManager \
  /org/freedesktop/NetworkManager org.freedesktop.NetworkManager.CheckConnectivity
dbus-send --session --dest=org.freedesktop.Notifications \
  /org/freedesktop/Notifications org.freedesktop.Notifications.Notify \
  string:"MyApp" uint32:0 string:"info" string:"Hello" string:"" array:string:"" \
  dict:string:string:"" int32:5000
dbus-monitor --system                        # Monitor system bus messages
dbus-monitor --session                       # Monitor session bus messages

# sysfs
cat /sys/class/net/eth0/operstate
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
echo 1 > /sys/class/leds/input3::capslock/brightness   # (if writable)

# /proc/sys/fs/inotify limits
cat /proc/sys/fs/inotify/max_user_watches     # Default: 8192
cat /proc/sys/fs/inotify/max_user_instances   # Default: 128
cat /proc/sys/fs/inotify/max_queued_events    # Default: 16384

# ACPI
acpi_listen                                   # Listen for ACPI events
```

## Under the Hood (GO DEEP)

### udev Event Flow

When a device is connected:

1. **Kernel detects** the hardware change (e.g., USB insertion triggers hub port interrupt)
2. **Kernel creates** a `struct device` and generates a **uevent** (userspace event) via `kobject_uevent_env()`
3. **uevent** is sent via **netlink** socket (`NETLINK_KOBJECT_UEVENT`) to userspace
4. **udevd** (the udev daemon) receives the uevent
5. **udevd** parses the event (ACTION, SUBSYSTEM, DEVTYPE, DEVPATH, etc.)
6. **udevd** walks through rules in priority order: `/etc/udev/rules.d/*.rules`, `/usr/lib/udev/rules.d/*.rules`, `/run/udev/rules.d/*.rules`
7. For each rule, udevd matches KEYS (like SUBSYSTEM, ATTR, KERNEL) against the event's properties
8. If all match, udevd executes the ASSIGNMENTS (like NAME, SYMLINK, MODE, OWNER, RUN, PROGRAM, IMPORT)
9. **RUN+=** commands are executed asynchronously (fork/exec) with a timeout (default 30s)
10. **IMPORT** commands run synchronously — their stdout is parsed as environment variables for the event

**Important:** udev rules run in a minimal environment:
- No PATH (use absolute paths)
- No terminal (no stdout/stderr visible to user)
- Short timeout (30s default for RUN)
- They block the udev event queue — long-running scripts slow down ALL device detection
- They run as root (unless specified otherwise)

### inotify Kernel Mechanics

inotify sits in the VFS (Virtual File System) layer. When a file operation occurs:

1. **Syscall handler** (e.g., `vfs_write()`, `vfs_unlink()`, `vfs_rename()`) completes the operation
2. Before returning, the handler calls `fsnotify()` with the event type and inode
3. `fsnotify()` checks if any inotify group is watching this inode
4. If yes, it allocates an `inotify_event` struct and appends it to the inotify FD's event queue
5. The `read()` on the inotify FD returns the event

**The one-shot vs. continuous distinction:**
- Without `-m` (one-shot): inotifywait exits after the first event. Useful for trigger-once scripts
- With `-m` (monitor): inotifywait keeps running, printing every event indefinitely. Used in long-running watchers

**inotify watch limits exist per user ID.** When a process has `max_user_watches` watches and tries to add another, `inotify_add_watch()` returns `ENOSPC`. Each watch consumes ~1KB of kernel memory (non-swappable — it's pinned).

### D-Bus Message Structure

D-Bus messages have a fixed header and a variable body:
- **Header:** endianness, message type (method_call, method_return, signal, error), flags, serial number, destination, path, interface, member
- **Body:** marshalled arguments with type signatures (string, uint32, boolean, array, dict, variant)

The `dbus-send` command constructs a message from its arguments. Type signatures:
- `string:"hello"` — a string
- `uint32:42` — a 32-bit unsigned integer
- `boolean:true` — a boolean
- `double:3.14` — a double
- `array:string:"a","b"` — array of strings
- `dict:string:string:"key","val"` — dictionary

D-Bus uses a "well-known bus name" (like `org.freedesktop.NetworkManager`) and "object paths" (like `/org/freedesktop/NetworkManager`). Interfaces group methods (like `org.freedesktop.NetworkManager.CheckConnectivity`).

### sysfs Interface

sysfs (`/sys/`) exposes kernel objects as a filesystem. Each class of device has a directory:
- `/sys/class/net/` — network interfaces
- `/sys/class/block/` — block devices
- `/sys/class/leds/` — LED controls
- `/sys/class/backlight/` — display brightness
- `/sys/devices/system/cpu/` — CPU information
- `/sys/class/power_supply/` — battery/AC status

Writing to sysfs files changes kernel parameters. This is how you control hardware features (brightness, LEDs, CPU governor) without special tools.

### What strace Reveals

```bash
# udevadm monitor receives netlink messages:
$ sudo strace -e recvmsg udevadm monitor 2>&1 | head -5
recvmsg(4, {msg_namelen=12, ...}, 0) = 768
# Reads uevent data: "add@/devices/pci0000:00/.../sdb/sdb1"

# inotifywait initializes:
$ strace -e inotify_init,inotify_add_watch inotifywait -m /tmp/test
inotify_init()                          = 3
inotify_add_watch(3, "/tmp/test", IN_MODIFY|IN_CREATE|IN_DELETE) = 1
read(3,

# dbus-send connects to D-Bus daemon:
$ strace -e connect,sendmsg dbus-send --session --dest=... ...
connect(3, {sa_family=AF_UNIX, sun_path="/run/user/1000/bus"}, 30) = 0
sendmsg(3, {msg_name=NULL, ...}, 0) = 128
```

### Security Model Interactions

**udev security:**
- Rules run as root by default. A vulnerable udev script that trusts user input (like USB device labels) can be exploited via a malicious USB device.
- udev rules can be used to bypass screen lockers: a USB device insertion that runs a script could unlock the system.
- Modern systems use `systemd-logind` to manage seat and session access, which interacts with udev to determine which user gets access to which device.

**inotify security:**
- inotify watches are private to the process that created them. Other processes cannot see or interfere with them.
- However, watch limits are per-user. A malicious user can exhaust their own watch limit.
- File events on inaccessible files are still watched (the watch is on the inode, not the path). Opening a file you can't read still generates IN_OPEN.

**D-Bus security:**
- D-Bus uses SELinux/AppArmor policies to control which processes can send messages to which services.
- The session bus is per-user — processes running as your user can send messages to your session.
- The system bus is more restricted — by default, only privileged processes can own well-known names on the system bus.

## Core Examples (15 total)

### Example 1: udev rule for USB automount

```bash
$ cat > /etc/udev/rules.d/99-usb-mount.rules << 'EOF'
ACTION=="add",   SUBSYSTEM=="block", KERNEL=="sd[b-z][0-9]", ENV{ID_FS_TYPE}!="", RUN+="/usr/local/bin/usb-mount.sh %k"
ACTION=="remove", SUBSYSTEM=="block", KERNEL=="sd[b-z][0-9]", RUN+="/usr/local/bin/usb-umount.sh %k"
EOF
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger
```

**Anatomy:** First rule: on add, match block subsystem, sd[b-z][0-9] partitions, only if they have a filesystem. Runs mount script. Second rule: on remove, runs unmount. `%k` passes the kernel name (e.g., `sdb1`).

**Variations:** Match on `ENV{ID_FS_LABEL}=="BACKUP*"` for specific drives. Match on `ATTR{size}=="media"` for optical drives.

**Edge case:** udev rules must be tested carefully. A malformed rule can prevent ALL devices from being detected. Always have a rescue plan (recovery console, single-user mode).

### Example 2: USB mount script

```bash
$ cat > /usr/local/bin/usb-mount.sh << 'EOF'
#!/bin/bash
DEVICE="/dev/$1"
MOUNT_BASE="/media"
LABEL=$(blkid -s LABEL -o value "$DEVICE" 2>/dev/null || echo "USB_$(basename $DEVICE)")
TARGET="$MOUNT_BASE/$LABEL"

# Idempotency check
findmnt -n "$DEVICE" >/dev/null 2>&1 && exit 0

mkdir -p "$TARGET"
mount "$DEVICE" "$TARGET" 2>/dev/null || {
  rmdir "$TARGET" 2>/dev/null
  logger -t usb-mount "Failed to mount $DEVICE"
  exit 1
}
logger -t usb-mount "Mounted $DEVICE at $TARGET"
EOF
```

**Anatomy:** Idempotent (safe to run multiple times). Uses `findmnt` to check if already mounted. Uses `blkid` to get the filesystem label. Falls back to USB_DEVNAME if no label.

**Variations:** Add `chown $USER:$USER $TARGET` for user access. Add `--read-only` for forensic mounts.

**Edge case:** udev rules run as root but the mount point must be accessible by the user who will use it. The script must handle permissions appropriately.

### Example 3: inotify config file watcher

```bash
$ inotifywait -m -e modify,create,delete,move /etc/myapp/ --format '%w%f %e %T' --timefmt '%Y-%m-%d %H:%M:%S'
```

**Output:**
```
/etc/myapp/config.yml MODIFY 2026-07-31 10:00:05
/etc/myapp/plugins/plugins.conf CREATE 2026-07-31 10:01:12
```

**Anatomy:** `-m` for continuous monitoring. `-e` specifies which events. `--format` produces structured output.

**Variations:** Use `--exclude '\.(swp|~)$'` to ignore editor temp files. Use `--timefmt` for custom timestamps.

**Edge case:** Editor save patterns vary. Vim saves to a temp file then renames. Emacs saves inline. You may see CREATE + MOVED_FROM/MOVED_TO instead of MODIFY.

### Example 4: inotify-based auto-reload

```bash
$ cat > /usr/local/bin/reload_on_change.sh << 'EOF'
inotifywait -m -e modify,create,delete,move -r /etc/myapp/ |
while read dir event file; do
  echo "$(date) — $event: $dir$file"
  case "$file" in
    *.conf|*.yml|*.yaml)
      /usr/local/bin/reload_myapp.sh && echo "Reloaded OK"
      ;;
  esac
done
EOF
```

**Anatomy:** Watches recursively with `-r`. The `while read` loop processes events. Only .conf, .yml, .yaml changes trigger a reload.

**Variations:** Debounce events: wait 1 second after the last event before reloading (catches batch saves).

**Edge case:** `inotifywait -r` on directories with many files can exhaust watch limits. Monitor `/etc/myapp` specifically, not `/etc` generally.

### Example 5: inotifywatch statistics

```bash
$ inotifywatch -t 30 /var/log/
```

**Output:**
```
Establishing watches... done in 0.1s.
Finished in 30 seconds
----  watch  events
/var/log/  142
/var/log/  89
```

**Anatomy:** `-t 30` collects event counts for 30 seconds, then prints totals. Useful for profiling filesystem activity.

**Variations:** `inotifywatch -t 60 -r /var/log/` for recursive collection.

**Edge case:** `inotifywatch` exits after the timeout. It's not for continuous monitoring — use `inotifywait -m` for that.

### Example 6: D-Bus desktop notification

```bash
$ dbus-send --session --dest=org.freedesktop.Notifications \
  /org/freedesktop/Notifications \
  org.freedesktop.Notifications.Notify \
  string:"Monitor:" \
  uint32:0 \
  string:"dialog-information" \
  string:"Backup Complete" \
  string:"The daily backup finished successfully" \
  array:string:"" \
  dict:string:string:"" \
  int32:5000
```

**Anatomy:** Sends a notification to the desktop notification daemon. Arguments: app_name, replaces_id, app_icon, summary, body, actions (empty), hints (empty), expire_timeout (5000ms).

**Variations:** Use `uint32:0` for the ID (0 = new notification, non-zero = replace existing). Use `string:"dialog-warning"` for warning icon.

**Edge case:** `dbus-send --session` requires a D-Bus session bus. In SSH sessions or cron jobs, there is no session bus. Use `DBUS_SESSION_BUS_ADDRESS` environment variable or skip notifications in non-desktop contexts.

### Example 7: D-Bus systemd service control

```bash
$ dbus-send --system --dest=org.freedesktop.systemd1 \
  /org/freedesktop/systemd1 \
  org.freedesktop.systemd1.Manager.StartUnit \
  string:"backup.service" string:"replace"
```

**Anatomy:** Tells systemd to start a service via D-Bus. Equivalent to `systemctl start backup.service` but without needing a local systemd instance.

**Variations:** `StartUnit` returns an object path for tracking. `GetUnit` checks status. `ListUnits` lists active units.

**Edge case:** Not all systemd operations are available via D-Bus. For complex operations, use `systemctl` directly.

### Example 8: sysfs — Reading and writing kernel parameters

```bash
$ cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_cur_freq
2400000

$ cat /sys/class/thermal/thermal_zone0/temp
45000

$ echo 1 > /sys/class/leds/input3::capslock/brightness 2>/dev/null || echo "Not writable"
```

**Anatomy:** sysfs files are pseudo-files. Reading them invokes kernel code to generate the output. Writing invokes kernel code to apply the setting.

**Variations:** CPU governor: `cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor`. Backlight: `/sys/class/backlight/intel_backlight/brightness`.

**Edge case:** Not all sysfs files are writable by non-root. Many require `CAP_SYS_ADMIN` or root ownership. Check with `ls -la`.

### Example 9: ACPI event listener

```bash
$ acpi_listen &
[1] 12345
$ # Close laptop lid:
button/lid LID0 00000080 00000000
$ # Open lid:
button/lid LID0 00000081 00000000
$ # Plug AC:
ac_adapter ADP1 00000001 00000000
$ # Unplug AC:
ac_adapter ADP1 00000000 00000000
```

**Anatomy:** `acpi_listen` prints ACPI events as they occur. Format: `DEVICE_TYPE DEVICE_ID ACTION VALUE`. Lid close/open: `button/lid`. AC adapter: `ac_adapter`.

**Variations:** Use with a script to auto-suspend on lid close (if not already handled by systemd-logind). Use to log power events.

**Edge case:** Many laptops don't expose all ACPI events. Virtual machines often have no ACPI at all. Test on actual hardware.

### Example 10: Combining udev + inotify for USB monitoring

```bash
$ cat > /usr/local/bin/usb-monitor.sh << 'EOF'
#!/bin/bash
# This script is triggered by udev when USB is inserted
# It monitors a specific USB path for file changes
DEVNAME="$1"
MOUNT_POINT="/media/$DEVNAME"

# Give it a moment to mount
sleep 2

if [ -d "$MOUNT_POINT" ]; then
  logger -t usb-monitor "Starting watch on $MOUNT_POINT"
  inotifywait -m "$MOUNT_POINT" -e modify,create,delete,move |
  while read dir event file; do
    logger -t usb-monitor "$event: $dir$file"
  done
fi
EOF
```

**Anatomy:** udev triggers the script on USB insertion. The script waits for the mount, then starts inotify on the mount point. Every file event is logged.

**Variations:** Combine with rsync: auto-copy files from the USB to a designated sync directory.

**Edge case:** The `sleep 2` is fragile — mount timing varies by device. Use `while [ ! -d "$MOUNT_POINT" ]; do sleep 0.5; done` instead.

### Example 11: inotify with debouncing

```bash
$ cat > /usr/local/bin/debounced_watch.sh << 'EOF'
WATCH_DIR="${1:-/etc/myapp}"
LAST_EVENT=0
DEBOUNCE_SEC=1

inotifywait -m -r "$WATCH_DIR" -e modify,create,delete,move --format '%w%f' |
while read file; do
  NOW=$(date +%s)
  if [ $((NOW - LAST_EVENT)) -ge $DEBOUNCE_SEC ]; then
    echo "$(date) — Change detected: $file"
    /usr/local/bin/reload_myapp.sh
  fi
  LAST_EVENT=$NOW
done
EOF
```

**Anatomy:** Ignores events that arrive within 1 second of the last event. This prevents multiple reloads when an editor saves multiple files in quick succession.

**Variations:** Use a timer-based accumulator: wait 2 seconds of silence, then reload once.

**Edge case:** If changes keep coming faster than the debounce interval, the reload NEVER happens. Add a max-wait timeout.

### Example 12: D-Bus network monitor

```bash
$ dbus-monitor --system --profile \
  interface=org.freedesktop.NetworkManager
```

**Output:**
```
signal time=1722416400.123456 sender=:1.2 -> destination=(null destination) ...
   string "eth0"
   uint32 70  # Connected
```

**Anatomy:** `dbus-monitor --system` listens on the system bus. Filter by interface to see only NetworkManager events.

**Variations:** `--session` for desktop events. `--profile` for human-readable output.

**Edge case:** `dbus-monitor` captures ALL messages. On a busy D-Bus, the output is overwhelming. Always filter.

### Example 13: sysfs power/battery monitor

```bash
$ cat > /usr/local/bin/battery_monitor.sh << 'EOF'
BATTERY_PATH="/sys/class/power_supply/BAT0"

while true; do
  if [ -d "$BATTERY_PATH" ]; then
    capacity=$(cat "$BATTERY_PATH/capacity")
    status=$(cat "$BATTERY_PATH/status")
    echo "$(date) — Battery: $capacity% ($status)"
    if [ "$capacity" -lt 15 ] && [ "$status" = "Discharging" ]; then
      notify-send -u critical "Battery Low" "Only $capacity% remaining"
    fi
  fi
  sleep 60
done
EOF
```

**Anatomy:** Polls sysfs every 60 seconds for battery status. Triggers a critical notification below 15%.

**Variations:** Check for BAT0 vs BAT1 (some laptops have two batteries). Check `voltage_now` and `current_now` for power draw.

**Edge case:** Desktop systems don't have BAT0. Check for the directory before reading.

### Example 14: udev rule for GPIO/sensor devices

```bash
$ cat > /etc/udev/rules.d/99-sensor.rules << 'EOF'
SUBSYSTEM=="i2c", ATTR{name}=="*temperature*", MODE="0660", GROUP="sensor"
SUBSYSTEM=="input", ENV{ID_INPUT_TOUCHSCREEN}=="1", MODE="0640", GROUP="input"
EOF
```

**Anatomy:** First rule: set permission 0660 and group `sensor` for I2C temperature sensors. Second rule: set 0640 and group `input` for touchscreen devices.

**Variations:** Use `OWNER="user"` to give device ownership to a specific user.

**Edge case:** These rules match on device attributes. Use `udevadm info -a -p /sys/path` to discover available attributes for matching.

### Example 15: Complete event-driven backup on USB insertion

```bash
$ cat > /usr/local/bin/usb-backup.sh << 'EOF'
#!/bin/bash
set -euo pipefail

DEVICE="$1"
MOUNT="/media/backup"
LABEL="BACKUP_DRIVE"
SOURCE="/home/user/Documents"

# Wait for device
sleep 3

# Check label
ID=$(blkid -s LABEL -o value "/dev/$DEVICE" 2>/dev/null)
if [ "$ID" != "$LABEL" ]; then
  logger -t usb-backup "Not backup drive ($ID), skipping"
  exit 0
fi

# Mount
mkdir -p "$MOUNT"
mount "/dev/$DEVICE" "$MOUNT"

# Rsync
rsync -av --delete "$SOURCE/" "$MOUNT/documents/" 2>&1 | logger -t usb-backup

# Sync and unmount
sync
umount "$MOUNT"
logger -t usb-backup "Backup complete for $DEVICE"
EOF
```

**Anatomy:** udev triggers on USB insertion. The script checks if the USB has the backup drive label. If yes, mounts, rsyncs, unmounts. All actions logged.

**Variations:** Add encryption check (verify LUKS before mounting). Add email notification. Add progress reporting via LED.

**Edge case:** The script blocks udev's event processing during the rsync. For large backups, background the work with `&` and let udev continue processing other events.

## Real-World Use Cases

### FOR the OS

- **udev:** Device node creation, persistent device naming (by ID instead of sdX), firmware loading, permission setting
- **inotify:** File managers (Nautilus/Automount folder refresh), backup tools (automatic), log monitoring
- **D-Bus:** Desktop notifications, NetworkManager integration, systemd management, hardware abstraction

### WITH the OS

- **udev + inotify:** Auto-backup when USB drive inserted (as shown in Example 15)
- **inotify + systemd:** Path-activated services (see Lesson 14)
- **D-Bus + notifications:** Script completion alerts, threshold warnings, system health updates
- **sysfs + shell:** CPU frequency scaling control, backlight management, fan speed monitoring

### AGAINST the OS

- **Malicious USB (BadUSB):** udev triggers a script that installs malware on insertion
- **inotify exhaustion:** Fill up max_user_watches to blind filesystem monitoring
- **D-Bus injection:** Send fake notifications or manipulate system services via D-Bus if access is available
- **sysfs abuse:** Writing to `/sys/class/leds/` to signal data via blinking LEDs (optical covert channel)

### FOR DEFENSE

- **Limit udev scripts:** Keep them short, idempotent, and logged. Use `logger` for every action
- **Increase inotify limits:** For production monitoring systems, set `sysctl fs.inotify.max_user_watches=65536`
- **D-Bus policy:** Restrict which users can send to the system bus. Use `dbus-daemon --config-file` with custom policies
- **Monitor sysfs:** Check for unexpected writes to sysfs with auditd

## Memory Aids

**"UDEV = KERNEL EVENT -> USER ACTION"** — The flow:
1. **U**event from kernel
2. **D**evice attributes matched against rules
3. **E**xecute actions (RUN, SYMLINK, NAME, MODE)
4. **V**erify with udevadm test

**"INOTIFY: WATCH -> READ -> EVENT"** — The usage pattern:
1. **WATCH** — `inotifywait -m /path` (adds the watch)
2. **READ** — `read` the events from the FD
3. **EVENT** — process the filename, event type, and timestamp

**"D-BUS: SESSION vs SYSTEM"** — Which bus to use:
- **SESSION** — your desktop (notifications, media players, browser)
- **SYSTEM** — the whole machine (NetworkManager, systemd, hardware events)

**"sysctl FS.INOTIFY.MAX_USER_WATCHES"** — The most important inotify tuning parameter. Default 8192 is low for recursive monitoring.

## Trap Vault (15 traps)

**Trap 1:** udev rules run in a minimal environment with NO PATH. Always use absolute paths for programs called in RUN+= directives. `/bin/mount` not `mount`.

**Trap 2:** udev RUN+= scripts block the event queue. A long-running script (e.g., rsync of a large backup) will delay ALL device detection until it completes. Use `&` to background long operations.

**Trap 3:** inotify watches are PER-INODE, not per-path. If a monitored file is deleted and recreated (common with editor save), the new file has a new inode and the old watch is lost.

**Trap 4:** inotify on NFS/CIFS/fuse filesystems is unreliable or unavailable. Some network filesystems don't generate inotify events at all. Use polling as fallback.

**Trap 5:** `max_user_watches` limits are per-user, not per-system. One user can exhaust their entire watch budget. On multi-user systems, budget watches carefully.

**Trap 6:** `dbus-send --session` in a cron job or SSH session will FAIL because there's no D-Bus session bus. Always check for `$DBUS_SESSION_BUS_ADDRESS` before sending session bus messages.

**Trap 7:** udev rules support MATCHING and ASSIGNMENT but the same KEY can only be used one way per rule. `ATTR{size}=="value"` matches, `ATTR{size}="value"` assigns. Mixing them causes parsing errors.

**Trap 8:** `udevadm test --action=add /sys/block/sdb` simulates rule processing but does NOT actually run RUN+= commands. Use `udevadm trigger` to test actual execution.

**Trap 9:** inotify event names for directories are RELATIVE to the watched directory, not absolute. When processing events, prepend the watched path.

**Trap 10:** sysfs files may return errors or stale data if read while the device is in transition. Always check the return value of `cat` on sysfs files.

**Trap 11:** ACPI events vary between hardware. `button/lid LID0 00000080` on one laptop may be `button/lid LID 00000080` on another. Test on target hardware.

**Trap 12:** udev rules are evaluated in lexical order of filenames. `10-my.rules` runs before `99-my.rules`. Use a numeric prefix to control ordering.

**Trap 13:** D-Bus method names are case-sensitive and include the full interface name. `org.freedesktop.Notifications.Notify` is correct. `org.freedesktop.Notifications.notify` will fail with "method not found."

**Trap 14:** inotify `IN_MOVED_FROM` and `IN_MOVED_TO` come as a PAIR with the same cookie value. A rename within the same directory generates both. A rename from outside generates only one.

**Trap 15:** udev rules can use `IMPORT{program}="command"` to import key=value pairs from a program's stdout. This is synchronous and blocks the event queue. Keep imported programs fast.

## See It In The Wild

- **systemd-udevd:** The standard udev implementation on all major distros. Handles device detection for everything from USB mice to NVMe drives.

- **Udisks2:** The userspace disk management daemon. Uses udev for device detection and D-Bus for communication. When you plug in a USB drive, udisks2 receives the udev event and emits a D-Bus signal that your file manager picks up.

- **Incron:** An inotify-based cron replacement. Uses inotify to trigger commands on file system events. The `/etc/incron.d/` config files specify what command to run when a file changes.

- **Nautilus file manager:** Uses inotify to automatically refresh folder contents. When a file changes in a watched directory, the folder view updates.

- **Telepathy (instant messaging):** Uses D-Bus for service discovery. IM clients register on the session bus, and the Telepathy manager connects them.

- **Connected standby (Intel):** Uses sysfs to control device power states. Writing to `/sys/devices/.../power/control` sets devices to "on" or "auto" for power management.

## Check Your Understanding (10 questions)

1. **Q:** What is the event flow from USB insertion to udev script execution? **A:** Kernel detects USB → generates uevent → sends via netlink → udevd receives → matches rules → executes RUN+= actions asynchronously.

2. **Q:** Why should udev scripts be short and fast? **A:** They block the udev event queue. Long-running scripts delay all device detection. Background long operations or use systemd units triggered by udev.

3. **Q:** What's the difference between `inotifywait` with and without `-m`? **A:** Without `-m`, it exits after the first event (one-shot). With `-m`, it monitors continuously, printing every event until killed.

4. **Q:** How does the inotify watch limit (`max_user_watches`) affect recursive directory monitoring? **A:** Each subdirectory uses one watch. Recursively monitoring a deep directory tree can exhaust the limit quickly. Increase the limit or monitor only the top-level directory.

5. **Q:** When would you use the system D-Bus bus vs. the session D-Bus bus? **A:** System bus: system-wide services (NetworkManager, systemd, udev). Session bus: per-user desktop (notifications, file manager, media player).

6. **Q:** Why can't you reliably use `dbus-send --session` from a cron job? **A:** Cron jobs don't have a D-Bus session bus. The `$DBUS_SESSION_BUS_ADDRESS` environment variable is not set. You'd need to save and restore the bus address from the user's session.

7. **Q:** What is the purpose of the sticky bit on `/tmp` in the context of inotify? **A:** Same as with file system attacks — prevents users from deleting each other's temp files. In the context of inotify, if a watched file is deleted and recreated, the inotify watch is lost.

8. **Q:** How do you test a udev rule without actually plugging in a device? **A:** Use `udevadm test /sys/block/sdb` (or whatever the device path is). Add `--action=add` to simulate insertion.

9. **Q:** What's a practical application of de-bouncing inotify events? **A:** When monitoring a config directory for auto-reload, editors may create multiple temp files. Debouncing groups these into a single reload action.

10. **Q:** How can you discover a device's available attributes for udev matching? **A:** Use `udevadm info -a -p /sys/class/...` or `udevadm info -a -n /dev/sdX`. This shows all attributes that can be used in KEY=="value" matching.
