# Task 14: Custom systemd Service & Timer

## Objective

Create a complete systemd-managed backup system: a service unit for the backup script, a timer unit for daily scheduling, and a monitoring/verification workflow. You'll write, deploy, test, and debug systemd units from scratch.

## Requirements (8 sub-tasks)

### Sub-task 1: Backup Script
Write `/usr/local/bin/backup.sh` that:
- Reads config from an environment file (`/etc/backup/backup.conf`)
- Creates a tar.gz archive of `/home` to `$BACKUP_DIR` with date-stamped filename
- Removes archives older than `$RETENTION_DAYS` days
- Logs progress to stdout (which systemd captures to journal)
- Exits with code 0 on success, non-zero on failure
- Uses `set -euo pipefail`

### Sub-task 2: Config File
Write `/etc/backup/backup.conf`:
```
BACKUP_DIR=/var/backups
RETENTION_DAYS=30
EXCLUDE_PATTERNS="*.log *.tmp .cache"
COMPRESS_LEVEL=6
```

### Sub-task 3: Service Unit
Write `/etc/systemd/system/backup.service`:
- Uses `Type=oneshot` (the script runs and exits)
- Loads environment from `/etc/backup/backup.conf`
- Runs as user `backup` (create the user if it doesn't exist)
- Uses `StandardOutput=journal`
- Sets `IOSchedulingClass=idle` (don't compete with interactive tasks)
- Sets `CPUSchedulingPolicy=batch`

### Sub-task 4: Timer Unit
Write `/etc/systemd/system/backup.timer`:
- Runs at 3:00 AM daily
- `Persistent=true` (catch up if system was off)
- `RandomizedDelaySec=1800` (jitter up to 30 minutes)
- `OnCalendar=daily` for the schedule
- Description: "Daily Backup Timer"

### Sub-task 5: Verification
- Run `systemd-analyze verify` on both unit files
- Run `systemctl daemon-reload`
- Test starting the service immediately (bypass the timer): `systemctl start backup.service`
- Check logs with `journalctl -u backup.service -f`
- Verify the backup archive was created in `$BACKUP_DIR`
- Run again — verify the retention logic works (no duplicate archives)

### Sub-task 6: Failure Testing
- Intentionally break the backup script (e.g., make BACKUP_DIR non-existent)
- Verify systemd shows the service as `failed`
- Test `Restart=on-failure` by adding it to the service unit
- Verify the service auto-recovers when the issue is fixed
- Test `systemctl reset-failed backup.service`

### Sub-task 7: Timer Testing
- Start the timer: `systemctl start backup.timer`
- View timer status: `systemctl list-timers --all | grep backup`
- Verify the Next Run time is correct
- Test `systemd-analyze calendar daily` to verify calendar expression
- Create a second timer for weekly full backups (Sundays at 4 AM)

### Sub-task 8: Monitoring Script
Write `/usr/local/bin/backup_monitor.sh` that:
- Checks if the backup timer is running and enabled
- Checks the most recent backup timestamp — alert if > 28 hours old
- Verifies backup file integrity with `tar -tzf`
- Reports backup size and archive count
- Runs as a systemd timer itself (hourly check)

## Bonus Challenges

1. **Docker integration:** Create a systemd service that runs a Docker container for backup processing. Use `ExecStartPre` to pull the latest image.

2. **Remote backup:** Add SSH remote backup support: rsync the local archive to a remote server using SSH keys managed by systemd's `LoadCredential`.

3. **Journal export:** Write a script that exports the last 7 days of backup journal logs to a file, compressed, for email delivery.

4. **Mail notification:** Add `OnFailure=backup-failure-notification@%n.service` to send email when the backup fails.

5. **Service graph:** Generate a dependency graph of all backup-related services using `systemd-analyze dot`.

## Hints

<details>
<summary>Hint 1: Service unit template</summary>

```bash
cat > /etc/systemd/system/backup.service << 'EOF'
[Unit]
Description=Daily Backup Service
Documentation=https://internal.wiki/backup
After=network-online.target

[Service]
Type=oneshot
User=backup
Group=backup
EnvironmentFile=/etc/backup/backup.conf
ExecStart=/usr/local/bin/backup.sh
ExecStartPost=/usr/local/bin/backup_verify.sh
IOSchedulingClass=idle
CPUSchedulingPolicy=batch
Nice=19
StandardOutput=journal
StandardError=journal
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
EOF
```
</details>

<details>
<summary>Hint 2: Timer unit</summary>

```bash
cat > /etc/systemd/system/backup.timer << 'EOF'
[Unit]
Description=Daily Backup Timer
Requires=backup.service

[Timer]
OnCalendar=daily
Persistent=true
RandomizedDelaySec=1800

[Install]
WantedBy=timers.target
EOF
```
</details>

<details>
<summary>Hint 3: Backup script</summary>

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR="${BACKUP_DIR:-/var/backups}"
RETENTION_DAYS="${RETENTION_DAYS:-30}"
COMPRESS_LEVEL="${COMPRESS_LEVEL:-6}"
DATE=$(date '+%Y-%m-%d')
FILENAME="home-backup-${DATE}.tar.gz"
LOG_FILE="${LOG_FILE:-/var/log/backup/backup.log}"

mkdir -p "$BACKUP_DIR"
echo "Starting backup: $DATE"
echo "Backup directory: $BACKUP_DIR"

tar -czf "${BACKUP_DIR}/${FILENAME}" \
    --exclude="*.log" --exclude="*.tmp" \
    --exclude=".cache" \
    /home 2>&1

echo "Created: ${BACKUP_DIR}/${FILENAME}"
SIZE=$(stat -c%s "${BACKUP_DIR}/${FILENAME}" 2>/dev/null)
echo "Size: $((SIZE / 1024 / 1024)) MB"

# Retention
find "$BACKUP_DIR" -name 'home-backup-*.tar.gz' -mtime +$RETENTION_DAYS -delete 2>/dev/null
echo "Retention: removed archives older than $RETENTION_DAYS days"

echo "Backup complete"
exit 0
```
</details>

<details>
<summary>Hint 4: Testing backup service</summary>

```bash
# Create backup user
sudo useradd -r -s /usr/sbin/nologin -M backup

# Create directories
sudo mkdir -p /var/backups /etc/backup
sudo chown backup:backup /var/backups

# Deploy
sudo cp backup.sh /usr/local/bin/backup.sh
sudo chmod +x /usr/local/bin/backup.sh
sudo cp backup.conf /etc/backup/backup.conf

# Reload and test
sudo systemctl daemon-reload
sudo systemd-analyze verify /etc/systemd/system/backup.service
sudo systemctl start backup.service
sudo journalctl -u backup.service -n 20 --no-pager
```
</details>

<details>
<summary>Hint 5: Verify with journalctl</summary>

```bash
# Follow the service output
sudo journalctl -u backup.service -f

# Show only the script's output (not systemd's meta-messages)
sudo journalctl -u backup.service -o cat --since "5 min ago"

# JSON output for programmatic use
sudo journalctl -u backup.service -o json-pretty -n 1

# Check last successful run timestamp
sudo journalctl -u backup.service -o cat | grep "Backup complete" | tail -1
```
</details>

<details>
<summary>Hint 6: Timer calendar testing</summary>

```bash
# Test calendar expressions
systemd-analyze calendar daily
systemd-analyze calendar "*-*-* 03:00:00"
systemd-analyze calendar "Mon..Fri *-*-* 09:00:00"

# Check timer status
systemctl list-timers --all | grep backup
systemctl status backup.timer
```
</details>

<details>
<summary>Hint 7: Failure simulation</summary>

```bash
# Create a failure by removing backup directory
sudo rm -rf /var/backups
sudo systemctl start backup.service

# Check failure
sudo systemctl status backup.service
# → active (failed) or failed

# Fix
sudo mkdir -p /var/backups
sudo chown backup:backup /var/backups
sudo systemctl reset-failed backup.service
sudo systemctl start backup.service

# Test with Restart=on-failure — service should auto-restart after fix
```
</details>

<details>
<summary>Hint 8: Monitoring script as a service</summary>

```bash
cat > /usr/local/bin/backup_monitor.sh << 'EOF'
#!/bin/bash
set -euo pipefail

LAST_BACKUP=$(find /var/backups -name '*.tar.gz' -printf '%T@\n' | sort -nr | head -1)
NOW=$(date +%s)
AGE=$(( (NOW - $(printf "%.0f" "$LAST_BACKUP")) / 3600 ))

echo "Last backup: $((AGE)) hours ago"

if [ "$AGE" -gt 28 ]; then
  echo "WARNING: Backup is over 28 hours old!"
  exit 1
fi

# Verify integrity of latest backup
LATEST=$(find /var/backups -name '*.tar.gz' -printf '%p\n' | sort -nr | head -1)
tar -tzf "$LATEST" > /dev/null 2>&1 && echo "Integrity: OK" || echo "Integrity: FAILED"

echo "Monitor check passed"
exit 0
EOF
```
</details>

## Expected Output

```bash
$ sudo systemd-analyze verify /etc/systemd/system/backup.service /etc/systemd/system/backup.timer
# No output = success

$ sudo systemctl daemon-reload
$ sudo systemctl start backup.service
$ sudo journalctl -u backup.service --since "1 minute ago"
Jul 31 10:00:01 host systemd[1]: Starting Daily Backup Service...
Jul 31 10:00:01 host backup.sh[12345]: Starting backup: 2026-07-31
Jul 31 10:00:01 host backup.sh[12345]: Backup directory: /var/backups
Jul 31 10:00:05 host backup.sh[12345]: Created: /var/backups/home-backup-2026-07-31.tar.gz
Jul 31 10:00:05 host backup.sh[12345]: Size: 342 MB
Jul 31 10:00:05 host backup.sh[12345]: Retention: removed 0 old archives
Jul 31 10:00:05 host backup.sh[12345]: Backup complete
Jul 31 10:00:05 host systemd[1]: Finished Daily Backup Service.

$ sudo systemctl status backup.service
● backup.service - Daily Backup Service
     Loaded: loaded (/etc/systemd/system/backup.service; disabled; vendor preset: enabled)
     Active: inactive (dead) since Wed 2026-07-31 10:00:05 UTC; 10s ago
    Process: 12345 ExecStart=/usr/local/bin/backup.sh (code=exited, status=0/SUCCESS)
   Main PID: 12345 (code=exited, status=0/SUCCESS)

$ sudo systemctl start backup.timer
$ systemctl list-timers --all | grep backup
Thu 2026-08-01 03:00:00 UTC 17h left  Wed 2026-07-31 10:00:05 UTC 1s ago  backup.timer

$ sudo systemctl status backup.timer
● backup.timer - Daily Backup Timer
     Loaded: loaded (/etc/systemd/system/backup.timer; disabled; vendor preset: enabled)
     Active: active (waiting) since Wed 2026-07-31 10:00:10 UTC; 5s ago
    Trigger: Thu 2026-08-01 03:00:00 UTC; 17h left

$ sudo systemctl enable backup.timer
Created symlink /etc/systemd/system/timers.target.wants/backup.timer → /etc/systemd/system/backup.timer.

$ # Failure testing
$ sudo mkdir -p /var/backups && sudo chown backup:backup /var/backups
$ sudo systemctl reset-failed backup.service
$ sudo systemctl start backup.service
$ sudo journalctl -u backup.service -n 3
Jul 31 10:01:00 host systemd[1]: Starting Daily Backup Service...
Jul 31 10:01:02 host backup.sh[12389]: Starting backup: 2026-07-31
Jul 31 10:01:05 host systemd[1]: backup.service: Main process exited, code=exited, status=1/FAILURE
Jul 31 10:01:05 host systemd[1]: backup.service: Failed with result 'exit-code'.

$ # After fixing the issue and adding Restart=on-failure:
$ sudo systemctl start backup.service
$ sudo journalctl -u backup.service -n 5
Jul 31 10:02:00 host systemd[1]: Starting Daily Backup Service...
Jul 31 10:02:02 host backup.sh[12456]: Starting backup: 2026-07-31
Jul 31 10:02:05 host backup.sh[12456]: Created: /var/backups/home-backup-2026-07-31.tar.gz
Jul 31 10:02:05 host backup.sh[12456]: Backup complete
Jul 31 10:02:05 host systemd[1]: Finished Daily Backup Service.
```

## Self-Check Questions

1. What is the difference between `Type=simple` and `Type=oneshot` in a systemd service?

2. Why is `Restart=on-failure` preferred over `Restart=always` for most services?

3. What does `Persistent=true` in a timer unit do, and why is it important for laptop backups?

4. How does `EnvironmentFile` differ from setting variables inline in `ExecStart`?

5. What happens if `StartLimitBurst` is exceeded for a service?

6. Why do you need `systemctl daemon-reload` after editing a unit file?

7. How do you test a timer-based service without waiting for the timer to fire?

8. What is the difference between `systemctl enable` and `systemctl start`?
