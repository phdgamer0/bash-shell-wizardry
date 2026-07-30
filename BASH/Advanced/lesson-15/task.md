# Task 15: USB Auto-Mount & Event Monitor

## Objective

Create a udev rule that auto-mounts USB drives, an inotify watcher for configuration file changes with auto-reload, an ACPI event logger, and a D-Bus notification system. Focus on reliability, idempotency, and safety.

## Requirements (8 sub-tasks)

### Sub-task 1: udev Rule for USB Detection
Write `/etc/udev/rules.d/99-usb-automount.rules` that:
- Matches USB block devices (sd[b-z][0-9]) with a filesystem
- Runs a mount script on add
- Runs an unmount script on remove
- Only matches devices with `ID_BUS=usb` (not SATA or NVMe)

### Sub-task 2: Mount Script
Write `/usr/local/bin/usb-mount.sh` that:
- Takes the kernel device name as argument (e.g., `sdb1`)
- Gets the filesystem label via `blkid`
- Creates mount point at `/media/<LABEL>` (or `/media/usb_<DEVICE>` if no label)
- Is idempotent (checks if already mounted)
- Creates a `.metadata` file on the mount with mount time, user, and device info
- Logs via `logger -t usb-mount`

### Sub-task 3: Unmount Script
Write `/usr/local/bin/usb-umount.sh` that:
- Takes the kernel device name as argument
- Finds the mount point via `findmnt` or `mount | grep`
- Syncs before unmounting: `sync`
- Unmounts and removes the mount point directory
- Logs unmount with duration (how long was the drive mounted)

### Sub-task 4: inotify Config Watcher
Write `/usr/local/bin/config-watcher.sh` that:
- Monitors `/etc/myapp/` (create it if it doesn't exist) for changes
- Logs every event with timestamp to `/var/log/configwatch.log`
- Triggers `/usr/local/bin/reload-app.sh` on `.conf` or `.yml` changes
- Uses debouncing (2-second window)
- Runs as a daemon (background, PID file, SIGTERM handling)
- Monitors its own log file size and rotates if > 10MB

### Sub-task 5: ACPI Event Logger
Write `/usr/local/bin/acpi-logger.sh` that:
- Runs `acpi_listen` and logs events to `/var/log/acpi_events.log`
- Classifies events: lid close/open, AC plug/unplug, battery events
- Logs with timestamps
- Runs in a minimal loop (no polling — acpi_listen blocks)
- Handles the case where acpi_listen is not available

### Sub-task 6: D-Bus Notification Script
Write `/usr/local/bin/send-notify.sh` that:
- Takes arguments: urgency (low/normal/critical), title, message
- Sends desktop notification via D-Bus
- Falls back to `notify-send` if available
- Falls back to `logger` if neither D-Bus nor notify-send works
- Supports both session and system bus (try session first, fall back to system)
- Returns 0 on success, non-zero on failure

### Sub-task 7: Integration Tests
Write `/usr/local/bin/test-integration.sh` that:
- Tests udev rules with `udevadm test` (simulated, no hardware needed)
- Tests mount script with a loopback device (creates a file, formats as ext4, mounts)
- Tests config watcher by touching files in the watched directory
- Tests ACPI logger by sending a test event (if supported)
- Tests D-Bus notification (requires desktop environment)
- Reports PASS/FAIL for each test

### Sub-task 8: Safety Hardening
Create a hardening script that:
- Sets `umask 077` in all mount/umount scripts
- Checks for `/media/` directory existence, creates with 755 if missing
- Mounts USB drives with `noexec,nosuid,nodev` options
- Limits mount point path length (label sanitization)
- Ensures all scripts are owned by root with 744 permissions

## Bonus Challenges

1. **USB whitelist:** Add a configuration file `/etc/usb-automount.allow` listing allowed USB serial numbers. Reject (don't mount) unmatched devices.

2. **Encrypted USB:** Add LUKS detection and auto-unlock via keyfile on the device (or password prompt via D-Bus).

3. **Sanitization:** Sanitize device labels to prevent path traversal (e.g., a label like `../../../etc` could escape `/media/`).

4. **Multi-session D-Bus:** Store `$DBUS_SESSION_BUS_ADDRESS` to a file when a user logs in, and use it from udev-triggered scripts (which don't have it).

5. **journald integration:** Send all udev/usb events to journald via `logger` or `systemd-cat` with structured fields.

## Hints

<details>
<summary>Hint 1: udev rule with security options</summary>

```bash
ACTION=="add", SUBSYSTEM=="block", KERNEL=="sd[b-z][0-9]", ENV{ID_BUS}=="usb", \
  ENV{ID_FS_TYPE}!="", ENV{SYSTEMD_WANTS}="usb-mount@%k.service"

ACTION=="remove", SUBSYSTEM=="block", KERNEL=="sd[b-z][0-9]", ENV{ID_BUS}=="usb", \
  RUN+="/usr/local/bin/usb-umount.sh %k"
```

Better approach: use `ENV{SYSTEMD_WANTS}` to trigger a systemd service instead of RUN+=. This is non-blocking and properly supervised.
</details>

<details>
<summary>Hint 2: Mount script with safety</summary>

```bash
#!/bin/bash
set -euo pipefail
DEVICE="/dev/$1"
LABEL=$(blkid -s LABEL -o value "$DEVICE" 2>/dev/null || echo "usb_$(basename $DEVICE)")
# Sanitize label: remove non-alphanumeric except underscore/hyphen
LABEL_CLEAN=$(echo "$LABEL" | sed 's/[^a-zA-Z0-9_-]/_/g')
# Truncate to 64 chars
LABEL_CLEAN="${LABEL_CLEAN:0:64}"
MOUNT="/media/$LABEL_CLEAN"

# Check if already mounted
findmnt -n "$DEVICE" >/dev/null 2>&1 && echo "Already mounted" && exit 0

mkdir -p "$MOUNT"
mount -o noexec,nosuid,nodev "$DEVICE" "$MOUNT" 2>/dev/null || {
  rmdir "$MOUNT" 2>/dev/null
  logger -t usb-mount "FAILED to mount $DEVICE"
  exit 1
}

# Write metadata
echo "MOUNTED=$(date '+%Y-%m-%d %H:%M:%S')" > "$MOUNT/.mount_info"
echo "DEVICE=$DEVICE" >> "$MOUNT/.mount_info"
echo "USER=$SUDO_USER" >> "$MOUNT/.mount_info"

logger -t usb-mount "Mounted $DEVICE on $MOUNT"
echo "$MOUNT"
```
</details>

<details>
<summary>Hint 3: inotify config watcher with daemon pattern</summary>

```bash
#!/bin/bash
set -euo pipefail

WATCH_DIR="${1:-/etc/myapp}"
LOG_FILE="/var/log/configwatch.log"
PID_FILE="/var/run/configwatch.pid"
DEBOUNCE=2
LAST_EVENT=0

cleanup() {
  echo "$(date '+%Y-%m-%d %H:%M:%S') — Stopping config watcher" >> "$LOG_FILE"
  rm -f "$PID_FILE"
  exit 0
}

trap cleanup SIGTERM SIGINT

echo "$$" > "$PID_FILE"
echo "$(date '+%Y-%m-%d %H:%M:%S') — Starting config watcher on $WATCH_DIR" >> "$LOG_FILE"

# Rotate log if needed
rotate_log() {
  local size=$(stat -c%s "$LOG_FILE" 2>/dev/null || echo 0)
  if [ "$size" -gt $((10 * 1024 * 1024)) ]; then
    mv "$LOG_FILE" "${LOG_FILE}.1"
    gzip "${LOG_FILE}.1"
    echo "$(date '+%Y-%m-%d %H:%M:%S') — Log rotated" > "$LOG_FILE"
  fi
}

inotifywait -m "$WATCH_DIR" -e modify,create,delete,move \
  --format '%w%f %e %T' --timefmt '%Y-%m-%d %H:%M:%S' |
while read file event time; do
  NOW=$(date +%s)
  rotate_log
  echo "$time — $event: $file" >> "$LOG_FILE"

  if [ "$((NOW - LAST_EVENT))" -ge "$DEBOUNCE" ]; then
    if [[ "$file" == *.conf || "$file" == *.yml ]]; then
      /usr/local/bin/reload-app.sh 2>&1 >> "$LOG_FILE"
    fi
  fi
  LAST_EVENT=$NOW
done
```
</details>

<details>
<summary>Hint 4: Testing with loopback device</summary>

```bash
# Create a test filesystem image
dd if=/dev/zero of=/tmp/test_usb.img bs=1M count=100
mkfs.ext4 /tmp/test_usb.img

# Mount via loopback
LOOP=$(sudo losetup -f --show /tmp/test_usb.img)
echo "Loop device: $LOOP"

# Test the mount script
sudo /usr/local/bin/usb-mount.sh "$(basename $LOOP)"

# Check
ls -la /media/*
cat /media/*/.mount_info

# Cleanup
sudo /usr/local/bin/usb-umount.sh "$(basename $LOOP)"
sudo losetup -d "$LOOP"
```
</details>

<details>
<summary>Hint 5: D-Bus notification function</summary>

```bash
send_notification() {
  local urgency="$1" title="$2" message="$3"
  local icon="dialog-information"
  [ "$urgency" = "critical" ] && icon="dialog-error"
  [ "$urgency" = "normal" ] && icon="dialog-warning"

  # Method 1: dbus-send (native D-Bus)
  if [ -n "${DBUS_SESSION_BUS_ADDRESS:-}" ]; then
    dbus-send --session --dest=org.freedesktop.Notifications \
      /org/freedesktop/Notifications \
      org.freedesktop.Notifications.Notify \
      string:"$title" uint32:0 string:"$icon" \
      string:"$title" string:"$message" \
      array:string:"" dict:string:string:"" int32:5000 2>/dev/null && return 0
  fi

  # Method 2: notify-send
  if command -v notify-send &>/dev/null; then
    notify-send -u "$urgency" "$title" "$message" && return 0
  fi

  # Method 3: logger (always works)
  logger -t notify "$title: $message"
  return 0
}
```
</details>

<details>
<summary>Hint 6: udev rule testing</summary>

```bash
# Simulate rule processing
sudo udevadm info -a -n /dev/sdb1  # Get device attributes
sudo udevadm test --action=add $(udevadm info -q path -n /dev/sdb1) 2>&1 | grep -E 'RUN|IMPORT|NAME'

# Trigger for real (plug in a USB first)
sudo udevadm trigger --action=add --subsystem-match=block
```
</details>

<details>
<summary>Hint 7: ACPI event parsing</summary>

```bash
parse_acpi_event() {
  local line="$1"
  local timestamp=$(date '+%Y-%m-%d %H:%M:%S')

  if echo "$line" | grep -q "button/lid"; then
    local state=$(echo "$line" | awk '{print $4}')
    if [ "$state" = "00000000" ]; then
      echo "$timestamp LID CLOSED"
    else
      echo "$timestamp LID OPENED"
    fi
  elif echo "$line" | grep -q "ac_adapter"; then
    local state=$(echo "$line" | awk '{print $4}')
    if [ "$state" = "00000001" ]; then
      echo "$timestamp AC PLUGGED IN"
    else
      echo "$timestamp AC UNPLUGGED"
    fi
  elif echo "$line" | grep -q "battery"; then
    echo "$timestamp BATTERY EVENT: $line"
  else
    echo "$timestamp UNKNOWN: $line"
  fi
}
```
</details>

## Expected Output

```bash
$ sudo udevadm control --reload-rules
$ sudo udevadm trigger
$ # Plug in a USB drive

$ tail -f /var/log/syslog | grep usb-mount
Jul 31 10:00:01 host usb-mount: Mounted /dev/sdb1 on /media/MYUSB
Jul 31 10:00:01 host kernel: [12345.678] usb 3-2: New USB device found, idVendor=1234, idProduct=5678

$ ls -la /media/MYUSB/
total 12
drwxr-xr-x  3 root root 4096 Jul 31 10:00 .
drwxr-xr-x  3 root root 4096 Jul 31 10:00 ..
-rw-r--r--  1 root root  103 Jul 31 10:00 .mount_info
drwxr-xr-x  2 root root 4096 Jul 31 12:00 documents

$ cat /media/MYUSB/.mount_info
MOUNTED=2026-07-31 10:00:00
DEVICE=/dev/sdb1
USER=root

$ # Remove the USB drive
Jul 31 10:05:00 host usb-mount: Unmounted /dev/sdb1 from /media/MYUSB (mounted for 5m 0s)

$ # Config watcher test
$ /usr/local/bin/config-watcher.sh --daemon
$ touch /etc/myapp/test.conf
$ cat /var/log/configwatch.log
2026-07-31 10:10:00 — CREATE: /etc/myapp/test.conf
2026-07-31 10:10:00 — Reload triggered for /etc/myapp/test.conf

$ # ACPI logger
$ /usr/local/bin/acpi-logger.sh &
[1] 12345
$ (close and open laptop lid)
$ cat /var/log/acpi_events.log
2026-07-31 10:15:00 LID CLOSED
2026-07-31 10:15:30 LID OPENED
2026-07-31 10:16:00 AC PLUGGED IN

$ # D-Bus notification test
$ /usr/local/bin/send-notify.sh critical "Test Alert" "This is a test notification"
✅ Notification sent via dbus-send

$ # Integration tests
$ sudo /usr/local/bin/test-integration.sh
=== Integration Tests ===
[TEST] udev rule syntax:       ✅ PASS (rules parsed without error)
[TEST] Mount via loopback:     ✅ PASS (mounted /dev/loop0 at /media/test_fs)
[TEST] Idempotent mount:       ✅ PASS (already mounted — skipped)
[TEST] Config watcher:         ✅ PASS (file change detected)
[TEST] ACPI logger:            ⚠️  SKIP (no ACPI hardware or acpi_listen missing)
[TEST] D-Bus notification:     ✅ PASS (notification received)
[TEST] Unmount cleanup:        ✅ PASS (directory removed after unmount)

=== Summary ===
PASS: 6/7
SKIP: 1 (ACPI — no hardware)
FAIL: 0
```

## Self-Check Questions

1. Why should udev scripts not run long operations directly? What pattern should you use instead?

2. What happens if two USB drives have the same label? How does your mount script handle this?

3. How does `inotifywait -m` differ from `inotifywait` without `-m`? When would you use each?

4. Why must udev rules be idempotent? What happens if `mount` is called twice for the same device?

5. What is the `max_user_watches` limit and how do you change it?

6. Why is sanitizing USB labels important for security? What could go wrong with a label like `../../../evil`?

7. How does debouncing improve the config watcher? What problem does it solve?

8. Why doesn't `dbus-send --session` work from cron or udev scripts? How could you work around this?
