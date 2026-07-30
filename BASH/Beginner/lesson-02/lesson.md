# Lesson 2: File Operations

## History & Origins

File operations are as old as Unix itself. The original Unix filesystem (1971) supported only 14-character filenames and had no concept of "undelete." That legacy persists: there is no trash can in the terminal.

The `cp` command was present in Version 1 Unix (1971). The original `cp` couldn't copy directories — that required combining `find` and `cpio` pipes. The `-r` flag for recursive copy was added in Version 7 Unix (1979), making `cp -r` the standard way to copy entire trees. The POSIX standard later (1988) added `-R` for recursive, but both work today.

`mv` (move) was also in Version 1 Unix, initially called `move` and renamed to `mv` for the 3-character limit. The rename operation (`mv old new` within the same filesystem) is atomic at the filesystem level — it just updates the directory entry. Moving across filesystems, however, is a copy+delete operation internally.

`rm` (remove) has been terrifying users since 1971. The famous `rm -rf /` bug is so legendary it has its own Wikipedia page. The `-r` flag (recursive) and `-f` flag (force) were added in the late 70s. There was never an `-i` flag in original Unix — asking for confirmation was added by GNU coreutils later.

`mkdir` first appeared in Version 1 Unix. Remarkably, `mkdir -p` (create parent directories) didn't exist until the System III release (1982) — for over a decade, you had to create each directory level manually or use a shell loop. `rmdir` (remove empty directory) was added alongside.

`touch` was invented for a specific purpose: updating file timestamps to control which files `make` considers out-of-date. The name "touch" comes from literally "touching" the file's access/modification time. Creating empty files was a side effect that became the primary use for most users.

`ln` (link) came from Version 1 Unix — it could only create hard links. The symbolic link (`ln -s`) was a much later innovation, introduced in 4.2BSD (1983) and adopted by System V Release 4 (1988). Symbolic links were considered radical — they broke the strict tree model by allowing the filesystem to be a directed graph with cycles.

`file` command appeared in Version 4 Unix (1973). It reads magic bytes — the first few bytes of a file — to identify what kind of file it is. The magic database (`/usr/share/misc/magic` or `/usr/share/file/magic/`) is the same concept used by `libmagic`.

**Fun anecdote:** Thompson and Ritchie argued about `rm` behavior. Thompson wanted `rm` to silently delete (you asked for it, you got it). Ritchie wanted warnings. The compromise: `rm` is silent, `rm -i` asks. Thompson's view won by default. Also, the name "rm" is sometimes humorously expanded to "remove meticulously" by those who've accidentally deleted their home directory.

## Syntax Reference

### `cp` — Copy

```bash
cp [options] source target
cp [options] source... directory     # Copy multiple sources to a directory
```

| Flag | Long | What it does |
|------|------|-------------|
| `-r` | `--recursive` | Copy directories recursively |
| `-R` | `--recursive` | Same (POSIX prefers `-R`, GNU treats as identical) |
| `-i` | `--interactive` | Prompt before overwrite |
| `-f` | `--force` | Remove existing destination files if needed |
| `-u` | `--update` | Copy only when source is newer than destination |
| `-v` | `--verbose` | Show what's being copied |
| `-p` | `--preserve` | Preserve mode, ownership, timestamps |
| `-a` | `--archive` | Preserve everything, recursive (`-dR --preserve=all`) |
| `-n` | `--no-clobber` | Don't overwrite existing files |
| `-l` | `--link` | Create hard links instead of copying |
| `-s` | `--symbolic-link` | Create symlinks instead of copying |
| `-b` | `--backup` | Backup existing destination files |
| `--backup=numbered` | | Numbered backups (`file.~1~`, `file.~2~`) |
| `-T` | `--no-target-directory` | Treat target as file, not directory |
| `-L` | `--dereference` | Follow symlinks, copy what they point to |
| `-P` | `--no-dereference` | Copy symlinks themselves (default) |

**Important nuance:** `cp dir1 dir2` is ambiguous. If `dir2` exists and is a directory, it copies `dir1` *into* `dir2` (creating `dir2/dir1`). If `dir2` doesn't exist, it copies `dir1` *as* `dir2` (rename). Use `-T` to avoid this: `cp -R -T dir1 dir2` always treats `dir2` as the target name.

### `mv` — Move (or Rename)

```bash
mv [options] source target
mv [options] source... directory
```

| Flag | Long | What it does |
|------|------|-------------|
| `-i` | `--interactive` | Prompt before overwrite |
| `-f` | `--force` | Don't prompt, just do it |
| `-u` | `--update` | Move only if source is newer or destination missing |
| `-v` | `--verbose` | Show what's being done |
| `-n` | `--no-clobber` | Don't overwrite existing |
| `-b` | `--backup` | Backup existing files |
| `-T` | `--no-target-directory` | Treat target as file |

Cross-filesystem `mv` performs a copy+delete. You can observe this with `strace` — you'll see `open()` + `read()` + `write()` (the copy) followed by `unlink()`.

### `rm` — Remove

```bash
rm [options] target...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-r`, `-R` | `--recursive` | Remove directories and their contents |
| `-f` | `--force` | Ignore nonexistent files, never prompt |
| `-i` | `--interactive` | Prompt before every removal |
| `-I` | | Prompt once when removing more than 3 files or recursively |
| `-v` | `--verbose` | Show what's being removed |
| `-d` | `--dir` | Remove empty directories (like `rmdir`) |
| `--no-preserve-root` | | Don't treat `/` specially |
| `--preserve-root` | | Do not remove `/` (default) |

**The `rm` risks:** `rm -rf /` with `--no-preserve-root` is the uninstall-the-universe command. Modern bash has `--preserve-root` enabled by default. But `rm -rf /*` bypasses that protection because it's not literally `/` as an argument — it's `/` followed by a wildcard that expands to everything in root.

### `mkdir` — Create Directory

```bash
mkdir [options] directory...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-p` | `--parents` | Create parent directories as needed, no error if exists |
| `-v` | `--verbose` | Show what's being created |
| `-m` | `--mode=MODE` | Set permissions (e.g., `mkdir -m 700 secretdir`) |

Without `-p`, `mkdir` fails if any component of the path doesn't exist:
```bash
$ mkdir a/b/c                    # Fails if a or a/b don't exist
mkdir: cannot create directory 'a/b/c': No such file or directory
$ mkdir -p a/b/c                 # Works — creates a, a/b, a/b/c
```

### `rmdir` — Remove Empty Directory

```bash
rmdir [options] directory...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-p` | `--parents` | Remove parent directories if they become empty |
| `-v` | `--verbose` | Show what's being removed |

`rmdir` only removes **empty** directories. For non-empty ones, use `rm -r`.

```bash
$ rmdir -p a/b/c                 # Removes c, then b (if empty), then a (if empty)
```

### `touch` — Create/Update Timestamps

```bash
touch [options] file...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-a` | | Change only access time |
| `-m` | | Change only modification time |
| `-c` | `--no-create` | Don't create the file if it doesn't exist |
| `-d` | `--date=STRING` | Use specific date (e.g., `touch -d "2023-01-15" file`) |
| `-r` | `--reference=FILE` | Use the same timestamp as another file |
| `-t` | `--timestamp=STAMP` | Use timestamp format `[[CC]YY]MMDDhhmm[.ss]` |

Timestamps stored in the inode:
- `atime` — last access time (read or executed)
- `mtime` — last modification time (content changed)
- `ctime` — last status change time (metadata changed — NOT creation time)

Note: Linux doesn't store file creation/birth time in inodes (though some filesystems do via `statx()`).

### `ln` — Link

```bash
ln [options] target linkname        # Create a link
ln [options] target... directory    # Create links in directory
```

| Flag | Long | What it does |
|------|------|-------------|
| `-s` | `--symbolic` | Create symbolic link (instead of hard) |
| `-f` | `--force` | Remove existing destination files |
| `-i` | `--interactive` | Prompt before removal |
| `-n` | `--no-dereference` | If linkname is a symlink to a dir, treat it as a file |
| `-v` | `--verbose` | Show what's being done |
| `-r` | `--relative` | Create relative symlinks (GNU coreutils 8.16+) |

**Hard link limitations:**
- Cannot cross filesystem boundaries (inode numbers are only unique within a filesystem)
- Cannot link to directories (would create cycles)
- Cannot link to files on different mount points

**Symbolic link notes:**
- Can point to directories
- Can cross filesystems
- Can point to non-existent targets (dangling symlinks)
- The path stored in the symlink is the path as given (absolute or relative)

### `file` — Detect File Type

```bash
file [options] file...
```

| Flag | Long | What it does |
|------|------|-------------|
| `-b` | `--brief` | Brief output (no filename prefix) |
| `-i` | `--mime` | Output MIME type strings |
| `-z` | `--uncompress` | Look inside compressed files |
| `-s` | `--special-files` | Read block/character device files |
| `-L` | `--dereference` | Follow symlinks |
| `-f` | `--files-from=FILE` | Read list of files to check from FILE |
| `-k` | `--keep-going` | Don't stop at first match |

The `/usr/share/misc/magic` database defines patterns. The `file` command reads the first few bytes and compares them. Examples:

```bash
$ file /bin/ls
/bin/ls: ELF 64-bit LSB pie executable, x86-64, version 1 (SYSV)
$ file -i /bin/ls
/bin/ls: application/x-pie-executable
$ file /etc/passwd
/etc/passwd: ASCII text
$ file unknown_file
unknown_file: gzip compressed data, was "backup.tar", last modified: ...
```

## Under the Hood

### What happens during `cp file1 file2`

1. **`open("file1", O_RDONLY)`**: Opens source file. Kernel returns a file descriptor. The kernel reads inode, checks read permission.
2. **`stat("file1", &stat_buf)`**: Gets metadata to know size, ownership, mode.
3. **`open("file2", O_WRONLY|O_CREAT|O_TRUNC, mode)`**: Creates/truncates destination. Sets permissions based on umask.
4. **Loop**: `read(src_fd, buf, 8192)` -> `write(dst_fd, buf, bytes_read)` until EOF.
5. **`close(src_fd)`** and **`close(dst_fd)`**: Release file descriptors.
6. **`chown(dst, stat.st_uid, stat.st_gid)`** and **`chmod(dst, stat.st_mode)`**: If `-p` specified, preserve ownership/permissions.

With `strace`:

```
openat(AT_FDCWD, "file1", O_RDONLY) = 3
openat(AT_FDCWD, "file2", O_WRONLY|O_CREAT|O_TRUNC, 0666) = 4
read(3, "Hello world\n", 65536) = 12
write(4, "Hello world\n", 12) = 12
read(3, "", 65536) = 0
close(3) = 0
close(4) = 0
```

### What happens during `mv` within the same filesystem

No data is moved. The kernel just updates the directory entry:

```
rename("oldname", "newname") = 0
```

The `rename()` syscall is atomic. If the system crashes mid-rename, either the old name or the new name exists, not a partial state. This is why `mv` is safe for replacing config files in production.

### What happens during `mv` across filesystems

The kernel detects different filesystems (different `st_dev`) and falls back to copy+delete:

```
stat("src", ...) = 0
stat("dst", ...) = -1  # (doesn't exist)
open("src", O_RDONLY) = 3
open("dst", O_WRONLY|O_CREAT, 0666) = 4
# Read/write loop...
read(3, buf, 8192) = 8192
write(4, buf, 8192) = 8192
...
# After copy completes:
unlink("src") = 0
```

### What happens during `rm file`

1. **`unlink("file")`**: Removes the directory entry. If the link count drops to 0 and no process has the file open, the inode and data blocks are freed.
2. If another hard link exists, the inode and data persist (link count > 0).
3. If a process has the file open, the inode persists until the process closes it — even after `unlink()`.

### What happens during hard link creation

`ln existing new_link` calls:

```
link("existing", "new_link") = 0
```

The `link()` syscall creates a new directory entry pointing to the same inode. The inode's link count (`st_nlink`) increments. Both filenames are now equally "real" — there's no concept of "original" in the filesystem.

### What happens during symbolic link creation

`ln -s target symlink` calls:

```
symlink("target", "symlink") = 0
```

The `symlink()` syscall creates a special type of file whose contents are the path string "target". It's not a directory entry pointing to an existing inode — it's a new inode of type `S_IFLNK` whose data is the target path.

When you `open()` a symlink, the kernel automatically follows it (unless you use `O_NOFOLLOW` or functions like `lstat()` that specifically operate on the link itself).

### File descriptor implications

Every `open()` returns a file descriptor — a small integer (usually 3, 4, 5...) that the kernel uses to track open files. A process can have a limited number of open FDs (check with `ulimit -n`). If you exhaust them, `open()` fails with `EMFILE` ("Too many open files"). This is why tools like `find ... -exec` are careful not to keep files open unnecessarily.

## Core Examples

### Example 1: Copy with preservation

```bash
$ cp -p /etc/hosts ~/hosts.copy
$ ls -la ~/hosts.copy /etc/hosts
-rw-r--r-- 1 root root 228 Jul 30 12:00 /etc/hosts
-rw-r--r-- 1 phd  phd  228 Jul 30 12:00 /home/phd/hosts.copy
```

**Step by step:**
1. `cp -p` reads `/etc/hosts` and writes to `~/hosts.copy`.
2. `-p` tells `cp` to call `chown()`, `chmod()`, and `utimensat()` to replicate ownership, permissions, and timestamps from the source.

**What if:**
```bash
$ cp /etc/hosts ~/hosts.copy        # Without -p: uses current umask, current timestamp
$ ls -la ~/hosts.copy
-rw-r--r-- 1 phd phd 228 Jul 31 01:00 /home/phd/hosts.copy  # Owned by phd, time is now
```

### Example 2: Copying a directory tree

```bash
$ mkdir -p /tmp/cp_test/src
$ touch /tmp/cp_test/src/{a,b,c}.txt
$ cp -r /tmp/cp_test/src /tmp/cp_test/dest
$ tree /tmp/cp_test
/tmp/cp_test
├── src
│   ├── a.txt
│   ├── b.txt
│   └── c.txt
└── dest
    └── src
        ├── a.txt
        ├── b.txt
        └── c.txt
```

Note that `dest/src` was created — not just the contents of `src`. To avoid this nested structure, use `cp -rT src dest` or `cp -r src/. dest`.

**What if:**
```bash
$ cp -r /tmp/cp_test/src/. /tmp/cp_test/dest2    # Contents only
$ tree /tmp/cp_test/dest2
/tmp/cp_test/dest2
├── a.txt
├── b.txt
└── c.txt
```

### Example 3: `mv` as rename

```bash
$ touch /tmp/fileA
$ mv /tmp/fileA /tmp/fileB
$ ls /tmp/fileA
ls: cannot access '/tmp/fileA': No such file or directory
$ ls /tmp/fileB
/tmp/fileB
```

**Step by step:**
1. `rename("/tmp/fileA", "/tmp/fileB")` is called.
2. Kernel looks up the dentry for `/tmp/fileA`, renames it to `fileB` in the same directory.
3. The inode is unchanged. The data is unchanged. Only the name changed.
4. The operation is atomic — if the system crashes, either name exists, not both or neither.

**What if:**
```bash
$ mv /tmp/fileB /some/other/filesystem/fileC    # Cross-filesystem!
# Copy + delete happens. /tmp/fileB is gone, /some/other/.../fileC has same data.
```

### Example 4: `rm -i` safety net

```bash
$ touch /tmp/important
$ rm -i /tmp/important
rm: remove regular empty file '/tmp/important'? y
```

**Step by step:**
1. `rm` calls `unlink("/tmp/important")`.
2. If the file doesn't exist (and without `-f`), it errors.
3. With `-i`, `rm` prints a prompt and reads stdin before unlinking.
4. Only on `y` or `yes` (case-insensitive) does it proceed.

**What if:**
```bash
$ rm /tmp/important          # No prompt. No second chance. Gone.
$ rm -f /tmp/nonexistent     # No error — -f suppresses "not found"
$ rm -rf /                   # Refused by modern rm (--preserve-root)
$ rm -rf --no-preserve-root /  # RIP — never, EVER run this
```

### Example 5: Hard link vs symlink behavior under deletion

```bash
$ echo "shared data" > /tmp/original
$ ln /tmp/original /tmp/hard     # Hard link
$ ln -s /tmp/original /tmp/soft  # Symlink
$ rm /tmp/original
$ cat /tmp/hard
shared data
$ cat /tmp/soft
cat: /tmp/soft: No such file or directory
```

**Step by step:**
1. After `rm /tmp/original`, the inode's link count drops from 2 to 1 (the hard link keeps it alive).
2. The data still exists on disk, pointed to by the inode, which is pointed to by `/tmp/hard`.
3. The symlink `/tmp/soft` contains the path string `/tmp/original` — which no longer exists. Dangling.
4. `cat /tmp/soft` triggers the kernel to follow the symlink -> try to open `/tmp/original` -> `ENOENT`.

**What if:**
```bash
$ echo "new data" > /tmp/original    # Re-create original
$ cat /tmp/soft                       # Symlink works again (it's still pointing to /tmp/original)
new data
$ cat /tmp/hard                       # Still points to old inode — still has "shared data"
shared data
```

### Example 6: `mkdir -p` with permission modes

```bash
$ mkdir -p -m 700 myapp/{src,test}/{unit,integration}
$ ls -ld myapp myapp/src myapp/test myapp/src/unit
drwx------ 3 phd phd 4096 Jul 31 01:00 myapp
drwx------ 3 phd phd 4096 Jul 31 01:00 myapp/src
drwx------ 2 phd phd 4096 Jul 31 01:00 myapp/src/unit
drwx------ 3 phd phd 4096 Jul 31 01:00 myapp/test
```

**Step by step:**
1. `mkdir -p` parses the path and creates each component if needed: `myapp`, `myapp/src`, `myapp/src/unit`, `myapp/test`, `myapp/test/integration`.
2. `-m 700` applies to all created directories — permission bits set to `rwx------`.

**What if:**
```bash
$ mkdir -p myapp/src/unit   # No error if already exists
$ mkdir myapp/src/unit      # Error: File exists
```

### Example 7: `touch` for timestamp manipulation

```bash
$ echo "hello" > /tmp/testfile
$ ls -l /tmp/testfile
-rw-r--r-- 1 phd phd 6 Jul 31 01:00 /tmp/testfile
$ touch -t 202001011200 /tmp/testfile
$ ls -l /tmp/testfile
-rw-r--r-- 1 phd phd 6 Jan  1  2020 /tmp/testfile
$ touch -a /tmp/testfile
$ ls -l /tmp/testfile                 # mtime unchanged (we changed atime)
-rw-r--r-- 1 phd phd 6 Jan  1  2020 /tmp/testfile
$ ls -lc /tmp/testfile                # -c shows ctime (status change) — updated!
-rw-r--r-- 1 phd phd 6 Jul 31 01:05 /tmp/testfile
```

**Step by step:**
1. Initial `touch /tmp/testfile` creates the file (if it didn't exist) and sets all timestamps to now.
2. `touch -t 202001011200` sets mtime and atime to Jan 1, 2020, 12:00.
3. `touch -a` updates only atime to current time. mtime stays at 2020.
4. `ls -lc` shows ctime — always changes when metadata changes.

**What if:**
```bash
$ touch -c /tmp/nonexistent    # -c: don't create if doesn't exist
$ touch /tmp/nonexistent       # Creates the file
```

### Example 8: Symlink types and relative paths

```bash
$ mkdir -p /tmp/link_test/sub
$ echo "data" > /tmp/link_test/sub/file.txt
$ ln -s sub/file.txt /tmp/link_test/rel_link     # Relative
$ ln -s /tmp/link_test/sub/file.txt /tmp/link_test/abs_link  # Absolute
$ mv /tmp/link_test /tmp/link_test_moved
$ cat /tmp/link_test_moved/rel_link
data
$ cat /tmp/link_test_moved/abs_link
cat: /tmp/link_test_moved/abs_link: No such file or directory
```

**Step by step:**
1. `rel_link` contains the string `sub/file.txt` — it's resolved relative to the directory containing the link.
2. When moved, the link's target is still `sub/file.txt` (still valid relative to the new location).
3. `abs_link` contains `/tmp/link_test/sub/file.txt` — this path no longer exists after the move.
4. Always use relative symlinks for relocatable trees!

**What if:**
```bash
$ ln -sr /tmp/link_test/sub/file.txt /tmp/link_test/auto_rel   # -r = automatic relative
$ cat /tmp/link_test/auto_rel
data
```

### Example 9: `file` magic bytes detection

```bash
$ echo "Test" > /tmp/test.txt
$ gzip -c /tmp/test.txt > /tmp/test.gz
$ file /tmp/test.txt /tmp/test.gz
/tmp/test.txt: ASCII text
/tmp/test.gz:  gzip compressed data, was "test.txt", last modified: ...
$ file -b /tmp/test.gz
gzip compressed data, was "test.txt", last modified: ...
$ file -i /tmp/test.gz
/tmp/test.gz: application/gzip
```

**Step by step:**
1. `file` reads the first bytes of each file.
2. `test.txt` starts with byte `54` (`T`) — matches ASCII text detection.
3. `test.gz` starts with `1f 8b` — the gzip magic number. `file` reads further to find the original filename and modification time embedded in the gzip header.
4. `-i` outputs MIME types instead of human-readable descriptions.

**What if:**
```bash
$ echo "#!/usr/bin/python3" > script.py
$ file script.py
script.py: Python script, ASCII text executable
$ echo "#!/bin/bash" > script.sh
$ file script.sh
script.sh: Bourne-Again shell script, ASCII text executable
```

### Example 10: The atomic `mv` for config file replacement

```bash
$ echo "version=1" > /tmp/config.txt
$ echo "version=2" > /tmp/config_new.txt
$ mv /tmp/config_new.txt /tmp/config.txt    # Atomic replacement
$ cat /tmp/config.txt
version=2
```

**Step by step:**
1. Write the new content to `config_new.txt` (a temporary file).
2. `mv /tmp/config_new.txt /tmp/config.txt` atomically replaces the old file.
3. Any process that already had `config.txt` open still sees the old content (they have the old inode). New opens see the new inode.
4. This is the standard pattern for safe configuration updates in production.

**What if:**
```bash
$ cp config_new.txt config.txt    # NOT atomic! Partial writes can be read.
$ > config.txt                    # Truncates — anyone reading sees empty file.
```

### Example 11: `rm` with complex patterns

```bash
$ mkdir /tmp/cleanup_test && cd /tmp/cleanup_test
$ touch a.txt b.txt c.txt .hidden .hidden2
$ rm *.txt
$ ls -a
.  ..  .hidden  .hidden2
```

**Step by step:**
1. The shell expands `*.txt` to `a.txt b.txt c.txt`.
2. `rm` receives three arguments: `a.txt b.txt c.txt`.
3. `unlink("a.txt")`, `unlink("b.txt")`, `unlink("c.txt")`.
4. Hidden files are NOT matched by `*` — they survive.

**What if:**
```bash
$ rm -rf .*          # Dangerous! Expands to ., .., .hidden, .hidden2
# This tries to remove . and .. too! Usually fails with "cannot remove '.' or '..'"
$ rm -rf .[!.]* .??* # Safer: matches .hidden, .hidden2 but not . or ..
```

### Example 12: `stat` for in-depth file information

```bash
$ echo "hello" > /tmp/stat_test
$ stat /tmp/stat_test
  File: /tmp/stat_test
  Size: 6         	Blocks: 8          IO Block: 4096   regular file
Device: 801h/2049d	Inode: 2621450     Links: 1
Access: (0644/-rw-r--r--)  Uid: ( 1000/    phd)   Gid: ( 1000/    phd)
Access: 2026-07-31 01:00:00.000000000 +0000
Modify: 2026-07-31 01:00:00.000000000 +0000
Change: 2026-07-31 01:00:00.000000000 +0000
 Birth: 2026-07-31 01:00:00.000000000 +0000
```

**Step by step:**
1. `stat` calls `stat("/tmp/stat_test")`.
2. Kernel returns the inode's contents: device ID (major 8, minor 1 = sda1), inode number, link count, permissions, ownership, timestamps.
3. "Birth" time is the creation time — only some filesystems (ext4, Btrfs, XFS) support it via `statx()`.
4. The difference between Modify and Change: modify is content changed, change is metadata changed (like chmod, chown, or rename).

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak`** — always back up config files before editing.
- **`mv /var/log/syslog /var/log/syslog.old && systemctl restart rsyslog`** — log rotation.
- **`rm -rf /var/tmp/*`** — clean temp files on a full disk.
- **`mkdir -p /srv/{www,git,backups}/{prod,staging}`** — scaffold server directories.
- **`ln -sf /usr/share/zoneinfo/America/New_York /etc/localtime`** — set system timezone.

### 2. WITH the OS — Development

- **`cp -r template-project my-new-project`** — project scaffolding.
- **`mv main.py main_v2.py`** — versioning before major changes.
- **`touch src/*.go`** — force recompile of all Go files.
- **`file upload.bin`** — identify unknown downloaded files.
- **`ln -s ~/projects/current ~/project-link`** — always point to the current project.

### 3. AGAINST the OS — Exploitation

- **Symlink attacks**: If a privileged script creates files in `/tmp` with predictable names, an attacker can `ln -s /etc/shadow /tmp/predictable_name` before the script runs. The script then writes to `/etc/shadow` (a privilege escalation). Prevention: `O_CREAT | O_EXCL` or `mkstemp()`.
- **`rm -rf / --no-preserve-root`** — the ultimate denial of service.
- **File descriptor exhaustion**: Open thousands of file descriptors to crash a daemon.
- **Hard link privilege escalation**: Before Linux protected `/etc/shadow`, hard linking it to a world-readable location allowed reading password hashes.

### 4. FOR DEFENSE — Detection & Auditing

- **`auditctl -w /etc/passwd -p wa -k password_changes`** — monitor changes to critical files.
- **`find / -type f -perm -4000 -ls`** — list all setuid binaries.
- **`stat /bin/ls`** — check if binaries have been tampered with (compare ctime to package install date).
- **`find / -nouser -o -nogroup`** — find orphaned files that could be backdoors.
- **`ls -la /proc/*/fd/`** — see what files processes have open.

## Memory Aids

### Mnemonics

**`cp -a`** = "cp archive" — preserves everything. Think "copy exactly."

**`mv`** = "move" but also "rename" — same command, same mechanism within a filesystem.

**`rm -rf`** = "remove recursively forcefully" — say this out loud before pressing Enter.

**`mkdir -p`** = "mkdir parents" — creates parents as needed.

**`ln -s`** = "link symbolic" — the `-s` is for "soft" (as opposed to "hard").

### Why it's named that

- **`cp`**: Short for "copy." 3 chars because early Unix had limits.
- **`mv`**: Short for "move." But within a filesystem, it's really "rename."
- **`rm`**: Short for "remove." Not "delete" — Unix "removes" the directory entry, but the data may remain until overwritten. This is why `rm` is not secure: `rm` doesn't overwrite the data. Use `shred` for that.
- **`ln`**: Short for "link." Creating a new directory entry pointing to an existing inode.
- **`touch`**: From "touching" the timestamps. The file creation was a side effect.

### Common confusions

- **`cp -r` vs `cp -R`**: Same thing. GNU and POSIX disagree on which is standard; use either.
- **`mv` across filesystems**: Not atomic. It's a `cp` + `rm`. Can take a long time.
- **`rm -r` vs `rmdir`**: `rm -r` deletes non-empty dirs. `rmdir` only deletes empty ones. Use `rmdir` when you want to verify emptiness.
- **Hard links vs symlinks**: A hard link is an additional name for the same data. A symlink is a path string that points to a name (not data).
- **`touch -c`**: No-create mode — only updates timestamps on existing files.

## Trap Vault

### Trap 1: `rm -rf` with variables

**Problem:** Empty variable turns `rm -rf $DIR/*` into `rm -rf /*`.

**Example:**
```bash
$ DIR=""            # Or DIR not set
$ rm -rf $DIR/*     # Becomes: rm -rf /*
rm: it is dangerous to operate recursively on '/'
rm: use --no-preserve-root to override
```

**Why:** Shell expands `$DIR/*` to `/*` when `$DIR` is empty. The glob `/*` matches everything in root.

**Fix:**
```bash
$ rm -rf "${DIR:?}"/*        # Fails if DIR is unset or empty
$ [ -n "$DIR" ] && rm -rf "$DIR"/*   # Guard with check
```

### Trap 2: `cp -r` vs `cp -r` with trailing slash

**Problem:** Trailing slash changes copy behavior.

**Example:**
```bash
$ mkdir /tmp/a /tmp/b
$ touch /tmp/a/file.txt
$ cp -r /tmp/a /tmp/b       # Creates /tmp/b/a/file.txt
$ cp -r /tmp/a/. /tmp/b     # Creates /tmp/b/file.txt
```

**Why:** `cp -r /tmp/a` treats `a` as a unit — copies the directory itself into target. `cp -r /tmp/a/.` copies the *contents* of `a` (the `.` means "this directory, but not the name").

**Fix:** Use `-T` for clarity: `cp -rT /tmp/a /tmp/b` copies contents, not the directory wrapper.

### Trap 3: `mv` silently overwrites

**Problem:** `mv target existing_dir` moves *into* existing_dir, not replacing it.

**Example:**
```bash
$ mkdir /tmp/dest /tmp/source
$ touch /tmp/source/file.txt
$ mv /tmp/source /tmp/dest     # /tmp/dest now contains /tmp/dest/source/file.txt
$ ls /tmp/dest
source
```

**Why:** When the last argument is an existing directory, `mv` interprets it as "move into this directory." To replace, you'd need `rm -rf /tmp/dest && mv /tmp/source /tmp/dest`.

**Fix:**
```bash
$ mv -T /tmp/source /tmp/dest   # -T: treat dest as file, not directory
```

### Trap 4: Broken symlinks are invisible to `ls` errors

**Problem:** `ls` doesn't report broken symlinks as errors.

**Example:**
```bash
$ ln -s /nonexistent broken
$ ls
broken      # Shows up fine — no error!
$ cat broken
cat: broken: No such file or directory
```

**Why:** `ls` lists directory entries. It doesn't verify that symlink targets exist. That's the user's job.

**Fix:**
```bash
$ find . -xtype l    # Find broken symlinks (-xtype l means "type l that can't be resolved")
$ find . -type l ! -exec test -e {} \; -print  # Alternative
```

### Trap 5: `cp` preserves permissions by default for root, but not for users

**Problem:** Non-root `cp` resets ownership to the current user.

**Example:**
```bash
$ ls -l /etc/hosts
-rw-r--r-- 1 root root 228 Jul 30 12:00 /etc/hosts
$ cp /etc/hosts /tmp/hosts
$ ls -l /tmp/hosts
-rw-r--r-- 1 phd phd 228 Jul 31 01:00 /tmp/hosts  # Owned by phd!
```

**Why:** `cp` by default creates the new file with the caller's UID/GID. Only root can preserve ownership (via `-p` or `-a`). Non-root users who try `cp -p` get a "chown: Operation not permitted" warning but the copy still proceeds.

**Fix:**
```bash
$ cp -p /etc/hosts /tmp/hosts   # Attempts to preserve — will warn but keep trying
```

### Trap 6: `rmdir` won't remove non-empty directories

**Problem:** `rmdir` silently fails on non-empty directories.

**Example:**
```bash
$ mkdir /tmp/mydir
$ touch /tmp/mydir/file.txt
$ rmdir /tmp/mydir
rmdir: failed to remove '/tmp/mydir': Directory not empty
```

**Why:** `rmdir` calls the `rmdir()` syscall which the kernel refuses if the directory's link count is > 2 (`.` and `..` plus directory entries).

**Fix:**
```bash
$ rm -r /tmp/mydir          # Recursive remove
$ find /tmp/mydir -delete   # Alternative (but careful with find -delete)
```

### Trap 7: Touch doesn't always create a new file

**Problem:** `touch -c` doesn't create files — but people expect `touch` to always create.

**Example:**
```bash
$ touch -c /tmp/newfile
$ ls /tmp/newfile
ls: cannot access '/tmp/newfile': No such file or directory
```

**Why:** `touch -c` (or `--no-create`) explicitly skips file creation. The man page says "do not create any files." It's for scripts that want to update timestamps only on existing files.

**Fix:**
```bash
$ touch /tmp/newfile         # Creates if doesn't exist
```

### Trap 8: Symlink permissions are meaningless

**Problem:** Changing symlink permissions with chmod doesn't work as expected.

**Example:**
```bash
$ echo "secret" > /tmp/secret
$ ln -s /tmp/secret /tmp/link
$ chmod 000 /tmp/link
chmod: cannot access '/tmp/link': Permission denied  # Wait, what?
```

Actually `chmod` follows the symlink and tries to change the target's permissions. If you want to change the symlink itself... you can't. Symlink permissions are always `lrwxrwxrwx` and are never enforced — the target's permissions are what matters.

**Fix:** You can't. Symlink permissions are always 0777 (shown as lrwxrwxrwx) but they're ignored by the kernel. The target's permissions are what count. Use `chown -h` to change symlink ownership.

### Trap 9: `cp -a` may not preserve everything across filesystems

**Problem:** `cp -a` preserves ACLs, xattrs, capabilities — but only if the target filesystem supports them.

**Example:**
```bash
$ cp -a myprog /mnt/usb/myprog   # USB is FAT32
cp: preserving permissions for '/mnt/usb/myprog': Operation not supported
cp: preserving ACL for '/mnt/usb/myprog': Operation not supported
$ ls -l /mnt/usb/myprog
-rwxrwxrwx 1 phd phd 1234 Jul 31 01:00 /mnt/usb/myprog  # All permissions lost!
```

**Why:** FAT32 doesn't support Unix permissions. `cp -a` tries its best but falls back to defaults. Always check the filesystem capabilities.

**Fix:**
```bash
$ tar cf - myprog | (cd /mnt/usb && tar xf -)   # Preserves metadata in the tar stream
$ rsync -a myprog /mnt/usb/                       # Rsync warns about unsupported features
```

### Trap 10: `file` can be wrong

**Problem:** `file` uses magic bytes, which can be spoofed.

**Example:**
```bash
$ echo "#!/bin/bash" > /tmp/fake.sh
$ echo "malicious code" >> /tmp/fake.sh
$ file /tmp/fake.sh
/tmp/fake.sh: Bourne-Again shell script, ASCII text executable
$ echo -e '\x7fELF' > /tmp/not_really_elf
$ file /tmp/not_really_elf
/tmp/not_really_elf: ELF 64-bit...   # Wrong! It's just the magic bytes
```

**Why:** `file` only checks the first bytes. An attacker can prepend `ELF`, `#!/bin/bash`, `PK`, or `%PDF` to any file to fake its type. This is how some malware evades file-type filters.

**Fix:** `file` is a hint, not a guarantee. For security-critical decisions, verify content independently.

## See It In The Wild

### Where you encounter file operations daily

- **`apt-get install`** — runs `mv` and `cp` extensively during package installation. Watch with `apt-get install -d <pkg> && ls /var/cache/apt/archives/`.
- **`systemctl edit <service>`** — creates a drop-in override file in `/etc/systemd/system/`.
- **`npm install`** — creates `node_modules/` directories and hardlinks/symlinks within them.
- **`git checkout`** — modifies files, changes timestamps, manages `.git/` directory entries.
- **`docker build`** — copies files into container layers using `COPY` instructions.

### How to observe file operations with strace

```bash
# Watch cp in action
strace -e trace=openat,read,write cp /etc/hosts /tmp/hosts_copy 2>&1

# Watch mv rename
strace -e trace=rename mv /tmp/a /tmp/b 2>&1

# Watch rm unlink
strace -e trace=unlink,unlinkat rm /tmp/testfile 2>&1

# Watch file magic matching
strace -e trace=read,openat file /bin/ls 2>&1 | head -30
```

### Try this now

```bash
# Create a file and watch its inode through operations
echo "hello" > /tmp/inode_test
stat /tmp/inode_test
ln /tmp/inode_test /tmp/inode_hard
stat /tmp/inode_test             # Link count is now 2
mv /tmp/inode_test /tmp/inode_renamed
stat /tmp/inode_renamed          # Same inode, same data
rm /tmp/inode_hard
stat /tmp/inode_renamed          # Link count back to 1
rm /tmp/inode_renamed            # Gone forever

# See what else has a file open
lsof /var/log/syslog | head -5
```

## Check Your Understanding

<details>
<summary>1. You run `cp /etc/passwd /tmp/`. The file `/tmp/passwd` is owned by you, not root. Why?</summary>

`cp` creates the new file with the calling user's UID/GID. Only root can use `-p` to preserve the original owner. Non-root users who specify `-p` get a permission error but the copy proceeds with current user ownership.
</details>

<details>
<summary>2. What's the difference between `cp file dir/` and `cp file dir` (when `dir` exists)?</summary>

None. Both copy `file` into directory `dir`, creating `dir/file`. The trailing slash is optional for existing directories. The behavior differs when `dir` doesn't exist: `cp file dir/` fails (can't create a directory with `/`), while `cp file dir` creates `dir` as a copy of `file`.
</details>

<details>
<summary>3. Why can't you hard-link a file across filesystems?</summary>

Hard links point to inodes. Inodes are local to a filesystem. The inode number 12345 on ext4 is a completely different piece of data than inode 12345 on XFS. Cross-filesystem hard links would require global inode uniqueness. Use symlinks instead.
</details>

<details>
<summary>4. You `rm` a file but `df` shows no free space increase. Why?</summary>

Another process has the file open. The inode and data blocks are not freed until the last file descriptor is closed, even though the directory entry has been removed. Check with `lsof | grep deleted`.
</details>

<details>
<summary>5. What's the difference between `touch -a` and `touch -m`?</summary>

`-a` updates only the access time (atime). `-m` updates only the modification time (mtime). Both update the change time (ctime) because ctime always updates when any metadata changes.
</details>

<details>
<summary>6. You have `fileA` and want to create `fileB` that shares the same inode. What command do you use?</summary>

`ln fileA fileB` — a hard link. Both names point to the same inode. Verify with `ls -li`.
</details>

<details>
<summary>7. Why does `mkdir -p a/b/c` not fail when `a/b/c` already exists, but `mkdir a/b/c` does?</summary>

`-p` explicitly suppresses the "File exists" error (and returns exit code 0). Without `-p`, `mkdir` reports the error and returns exit code 1. Also, `-p` creates parent directories as needed.
</details>
