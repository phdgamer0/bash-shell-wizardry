# Task 4: File Reading Relay

Read logs, watch files grow, inspect binary headers, reverse text, and master the pager. You'll use `cat`, `less`, `head`, `tail`, `xxd`, `od`, `tac`, `rev`, and `nl`.

## Setup

```bash
$ mkdir -p /tmp/read_lab
$ cd /tmp/read_lab
```

## Sub-task 1: Log Headers

Explore your system logs. Find a log file and read its header:

```bash
# Find available logs
$ ls /var/log/*.log 2>/dev/null | head -5
$ ls /var/log/*.log.1 2>/dev/null | head -5

# Read first 5 lines
$ head -n 5 /var/log/syslog 2>/dev/null || head -n 5 /var/log/dmesg 2>/dev/null

# Get line count
$ wc -l /var/log/syslog 2>/dev/null
```

**Expected output:**
```
$ head -5 /var/log/syslog
Jul 31 01:00:01 desktop kernel: Linux version 6.1.0 ...
Jul 31 01:00:01 desktop kernel: Command line: BOOT_IMAGE=...
Jul 31 01:00:01 desktop kernel: Kernel command line: ...
Jul 31 01:00:01 desktop kernel: Dentry cache hash table entries: ...
Jul 31 01:00:01 desktop kernel: Memory: 16288528K/...
```

**Question:** What format do syslog entries follow? (Hint: it's `<timestamp> <host> <service>[pid]: <message>`)

## Sub-task 2: Read from the Middle

Given a file with 45 lines:

```bash
$ wc -l /etc/passwd
45 /etc/passwd
```

Read lines 20-30 using two approaches:

```bash
# Approach 1: head then tail
$ head -n 30 /etc/passwd | tail -n 11

# Approach 2: tail then head
$ tail -n +20 /etc/passwd | head -n 11
```

**Question:** What's the difference between `tail -n +20` and `tail -n 20`? Why did I say `head -n 11` for lines 20-30 (that's 11 lines)?

<details>
<summary>tail -n forms</summary>
`tail -n +20` means "start from line 20." `tail -n 20` means "last 20 lines." Lines 20 through 30 inclusive is 11 lines (30 - 20 + 1 = 11).
</details>

## Sub-task 3: Live Monitoring with `tail -f`

In one terminal, start watching a log:

```bash
$ tail -f /var/log/syslog
```

In another terminal, generate a log message:

```bash
$ logger "Hello from Task 4! The time is $(date)"
```

Watch the message appear in the first terminal. Press Ctrl+C to stop.

**Question:** How is `tail -F` different from `tail -f`? When would you need `-F`?

<details>
<summary>tail -F explained</summary>
`-F` watches the file by NAME, not by file descriptor. If `logrotate` renames `syslog` to `syslog.1` and creates a new `syslog`, `-f` follows the old renamed file (it's watching the inode). `-F` watches the name, so it re-opens the new `syslog`. Always use `-F` for logs that rotate.
</details>

## Sub-task 4: Hex Dump the ELF Header

Inspect the first bytes of `/bin/ls`:

```bash
$ head -c 64 /bin/ls | xxd
```

**Expected output:**
```
00000000: 7f45 4c46 0201 0103 0000 0000 0000 0000  .ELF............
00000010: 0200 3e00 0100 0000 2010 4000 0000 0000  ..>..... .@.....
00000020: 4000 0000 0000 0000 5083 0100 0000 0000  @.......P.......
00000030: 0000 0000 4000 3800 0a00 0900 0000 0000  ....@.8.........
```

**Identify:** Can you find the ELF magic (`7f 45 4c 46`)? What does `02` mean (64-bit)? What does `01` mean (little-endian)?

Now inspect other binaries:

```bash
$ head -c 20 /bin/ls | xxd
$ head -c 20 /usr/bin/python3 | xxd
$ head -c 20 /bin/tar | xxd
```

**Question:** What do they all have in common?

<details>
<summary>ELF header</summary>
All Linux executables start with `7f 45 4c 46` (.ELF). This is the ELF magic number. The next byte tells you 32-bit (01) or 64-bit (02). The byte after that is endianness: 01 = little-endian (x86), 02 = big-endian.
</details>

Now inspect a non-ELF file:

```bash
$ file /etc/passwd
$ head -c 20 /etc/passwd | xxd
```

**Question:** How does `file` determine the type?

## Sub-task 5: Palindrome Play

Create a file and reverse it:

```bash
$ printf "apple\nbanana\ncherry\ndesserts\nstressed\n" > /tmp/fruit.txt

# tac: reverse line order
$ tac /tmp/fruit.txt

# rev: reverse characters per line
$ rev /tmp/fruit.txt

# Combined: both!
$ cat /tmp/fruit.txt | rev | tac
```

**Expected output:**
```
$ tac /tmp/fruit.txt
stressed
desserts
cherry
banana
apple

$ rev /tmp/fruit.txt
elppa
ananab
yrrehc
stressed
desserts

$ cat /tmp/fruit.txt | rev | tac    # Characters reversed, then lines reversed
stressed
desserts
yrrehc
ananab
elppa
```

**Question:** What word is a palindrome when read line-by-line with `tac` from this file?

<details>
<summary>Palindrome detection</summary>
"stressed" reversed is "desserts" — different words. No line is a palindrome here. But "racecar" is a palindrome: `echo "racecar" | rev` outputs "racecar".
</details>

## Sub-task 6: Numbered Lines with nl

Number the passwd file in different styles:

```bash
$ nl -ba /etc/passwd | head -5    # Number all lines
$ nl -bt /etc/passwd | head -5    # Number non-blank only
$ nl -ba -w3 -s': ' /etc/passwd | head -5  # Custom format
```

**Expected output:**
```
$ nl -ba -w3 -s': ' /etc/passwd | head -5
  1: root:x:0:0:root:/root:/bin/bash
  2: daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
  3: bin:x:2:2:bin:/bin:/usr/sbin/nologin
  4: sys:x:3:3:sys:/dev:/usr/sbin/nologin
  5: sync:x:4:65534:sync:/bin:/bin/sync
```

Now compare with `cat -n`:

```bash
$ cat -n /etc/passwd | head -5
```

**Question:** How does `nl` differ from `cat -n` in terms of blank line handling?

## Sub-task 7: od vs xxd

Dump a tiny file with various tools:

```bash
$ echo -n "Hello" > /tmp/test.hex

$ od -c /tmp/test.hex
$ od -x /tmp/test.hex
$ od -t x1 /tmp/test.hex    # One-byte hex (no endianness issue)
$ xxd /tmp/test.hex
```

**Expected output:**
```
$ od -c /tmp/test.hex
0000000   H   e   l   l   o
0000005

$ od -t x1 /tmp/test.hex
0000000 48 65 6c 6c 6f
0000005

$ xxd /tmp/test.hex
00000000: 4865 6c6c 6f                             Hello
```

**Question:** Why does `od -x /tmp/test.hex` show `6548 6c6c 6f` on x86? What happened to the byte order?

<details>
<summary>Endianness</summary>
x86 is little-endian. `od -x` reads 2-byte words. The word `He` (bytes 48, 65) is stored as 0x6548 in memory on little-endian. `od -t x1` reads one byte at a time and avoids this issue.
</details>

## Sub-task 8: Less Interactive Challenge

Use `less` to navigate `/etc/services` (a large file):

```bash
$ less /etc/services
```

Navigate using:
- `G` to go to the end — how many lines?
- `1G` to go back to the start
- `/http` to find the HTTP port entry
- `n` to find the next match
- `N` to go back
- `q` to quit

**Question:** What port does HTTPS use? What about SSH? What's the first port defined in the file?

<details>
<summary>Common ports</summary>
HTTP = 80 (tcp). HTTPS = 443 (tcp). SSH = 22 (tcp). The first entry in `/etc/services` is usually `tcpmux` (1/tcp).
</details>

## Sub-task 9: Watch a File Grow with `less +F`

```bash
$ less +F /var/log/syslog
```

This is identical to `tail -f` but within `less`. Press Ctrl+C to stop follow mode, then you can scroll normally with `j/k`. Press `F` again to resume following.

**Question:** What advantage does `less +F` have over `tail -f`?

<details>
<summary>less +F advantage</summary>
With `less +F`, you can press Ctrl+C to stop following, scroll back in the file to examine history, then press `F` to resume following. `tail -f` alone doesn't let you scroll back without opening another terminal.
</details>

## Bonus Challenge: Binary Header Detective

Create a script that identifies file types by their magic bytes:

```bash
$ cat > /tmp/magic_id.sh << 'EOF'
#!/bin/bash
for f in "$@"; do
    magic=$(head -c 4 "$f" | xxd -p)
    case "$magic" in
        7f454c46) echo "$f: ELF executable" ;;
        89504e47) echo "$f: PNG image" ;;
        ffd8ffe0|ffd8ffe1) echo "$f: JPEG image" ;;
        25504446) echo "$f: PDF document" ;;
        504b0304) echo "$f: ZIP archive (or .docx/.xlsx)" ;;
        1f8b*)    echo "$f: GZIP compressed" ;;
        47494638) echo "$f: GIF image" ;;
        00000018|00000020) echo "$f: MP4 video" ;;
        *)        echo "$f: Unknown type ($magic)" ;;
    esac
done
EOF
$ chmod +x /tmp/magic_id.sh
$ /tmp/magic_id.sh /bin/ls /etc/passwd /tmp/*.png 2>/dev/null
```

**Question:** Why does `file` have hundreds of patterns while this script only has 8?

<details>
<summary>Magic database</summary>
The real `file` command uses a database at `/usr/share/misc/magic` with thousands of patterns. It checks offsets beyond byte 0, handles "indirect" magic (file type depends on content at an offset), and uses MIME type mappings.
</details>

## Self-Check

1. How would you read a file that's constantly being written to, starting from the current end?
2. What's the difference between `cat file1 file2 > combined` and `cat file1 file2 >> combined`?
3. You type `cat binary_file > /dev/tty` and your terminal goes haywire. What command restores it?
4. How does `tac` work internally for a 500MB file? What memory implications does it have?
5. Why does `nl -ba file` produce different output than `cat -n file` (hint: formatting)?
6. What are `^I` and `^M` when displayed by `cat -A`?
7. You need lines 100-200 of a 10,000-line file. Write two different commands to extract them.
