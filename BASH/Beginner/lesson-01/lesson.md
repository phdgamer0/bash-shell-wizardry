# Lesson 1: Navigating the Filesystem

## History & Origins

The Unix filesystem — the tree you're about to navigate — was designed at Bell Labs in the early 1970s by Ken Thompson, Dennis Ritchie, and the rest of the Unix crew. Before Unix, most operating systems had flat filesystems where every file lived in a single global namespace. Imagine dumping every file on your computer into one giant folder. That was the norm.

Thompson and Ritchie invented the hierarchical directory tree for Unix (around 1971–1973), inspired by the MULTICS project's more complex filesystem hierarchy. The root `/` was chosen because the filesystem was literally rooted at one point, and `..` for parent directories appeared in Version 2 Unix (1972) — the earliest known filesystem to use `..`. The `.` for current directory was added soon after.

`pwd` (print working directory) first appeared in Version 5 Unix (1974). It was necessary because early Unix terminals didn't show your current directory in the prompt. You'd log in and simply have no idea where you were. So `pwd` was your only way to find out.

`cd` was also born in the early days, but interestingly, `cd` is not a standalone program — it's a shell builtin. Why? Because if `cd` were an external command, it would change *its own* working directory, then exit, and you'd be right back where you started. Only the shell itself can change the shell's directory. So `cd` has been a shell builtin since the Bourne shell (1977).

`ls` traces back to Version 1 Unix (1971) — literally one of the first commands ever written. The name stands for "list segment" from the assembly-level list directive. The `-l` flag (long format) was added later and `-a` (all files, including dotfiles) appeared in Version 7 Unix (1979). The convention of hiding files starting with `.` was a bug that became a feature — the `ls` developers simply skipped entries starting with `.` in directory listings, and people started naming config files like `.profile` and `.rc` to hide them.

Throughout the 1980s, different Unix vendors added flags: SysV added `-F` (classify), BSD added `-s` (size in blocks) and `-G` (color). When POSIX standardized `ls` in 1988, it defined the core flags, but GNU's version (`ls` from coreutils, 1992) went wild with 50+ options. Modern GNU `ls` has flags you'll never use but someone somewhere needs.

**Fun anecdote:** The `tree` command isn't even part of the official Unix spec. It was written by Steve Baker in 1993 as a freeware utility, distributed via Usenet, and later ported to Linux. It's not installed by default on many systems, which is hilarious because it's one of the most intuitive commands for beginners. Also, `realpath` — a GNU coreutils addition (2002) — didn't exist in old Unix. Old-timers used `readlink -f` or wrote shell loops to resolve symlinks.

## Syntax Reference

### `pwd` — Print Working Directory

```bash
pwd                  # Print current directory (logical — respects symlinks)
pwd -L               # Same as default: logical path (follows symlinks)
pwd -P               # Physical path: resolves all symlinks to real paths
```

The `-L` vs `-P` distinction matters: if `/home/phd/link` is a symlink to `/var/www`, then `pwd -L` shows `/home/phd/link` and `pwd -P` shows `/var/www`. Without flags, it depends on your shell's state — `set -o physical` makes `pwd` act like `pwd -P` by default.

### `ls` — List Directory Contents

```bash
ls [options] [file...]
```

**Selection flags:**

| Flag | Long | What it does |
|------|------|-------------|
| `-a` | `--all` | Include entries starting with `.` |
| `-A` | `--almost-all` | Like `-a` but excludes `.` and `..` |
| `-d` | `--directory` | List directory entries themselves, not their contents |
| `-L` | `--dereference` | Show symlink targets, not the symlinks themselves |
| `-R` | `--recursive` | Recursively list subdirectories |
| `-1` | `--format=single-column` | One entry per line |

**Format flags:**

| Flag | What it does |
|------|-------------|
| `-l` | Long format: permissions, links, owner, group, size, time, name |
| `-h` | Human-readable sizes (1K, 2M, 3G) — must pair with `-l` |
| `-s` | Print size in blocks |
| `-S` | Sort by size (largest first) |
| `-t` | Sort by modification time (newest first) |
| `-r` | Reverse sort order |
| `-X` | Sort alphabetically by extension |
| `-v` | Natural version sort (file1, file2, file10 — not file1, file10, file2) |
| `--sort=WORD` | Sort by `none`, `size`, `time`, `version`, or `extension` |

**Output flags:**

| Flag | Long | What it does |
|------|------|-------------|
| `-i` | `--inode` | Show inode numbers |
| `-F` | `--classify` | Append `*/=>@\|` to indicate file type |
| `-p` | `--indicator-style=slash` | Append `/` to directories |
| `-Q` | `--quote-name` | Enclose names in double quotes |
| `-m` | `--format=commas` | Comma-separated (pipe through column -t for alignment) |
| `-C` | `--format=vertical` | Columnar output (default for terminals) |
| `-x` | `--format=across` | Sort across, not down |
| `-la` | Combined | Long format + show all entries |
| `-lh` | Combined | Long format + human-readable sizes |
| `-lt` | Combined | Long format + sort by time |
| `-ltr` | Combined | Long format + sort by time, reversed (oldest last) |
| `-lita` | Combined | Long + inode + all — complete file identity |

**Color control:**

```bash
ls --color=auto      # Color only when outputting to terminal (default)
ls --color=always    # Force color even when piped
ls --color=never     # No color

export LS_COLORS     # Environment variable controlling color scheme
```

**The long format (`-l`) decoded:**

```
-rw-r--r--  1  phd  phd  4096  Jul 31 01:00  filename
|          |  |    |    |    |        |        |
|          |  |    |    |    |        |        +-- Entry name
|          |  |    |    |    |        +-- Modification timestamp
|          |  |    |    |    +-- Size in bytes
|          |  |    |    +-- Group owner
|          |  |    +-- User owner
|          |  +-- Number of hard links (or subdirs for directories)
|          +-- File type:
|              - = regular file, d = directory, l = symlink
|              b = block device, c = character device,
|              p = named pipe, s = socket
+-- Permissions (3 groups of rwx x 3):
    r=read, w=write, x=execute, -=absent
    SetUID/SetGID/Sticky: S/s/T/t in execute position
```

### `cd` — Change Directory

```bash
cd [directory]       # Change to directory
cd                   # Change to $HOME
cd -                 # Change to previous directory (OLDPWD)
cd ~                 # Change to $HOME
cd ~user             # Change to user's home directory
cd ..                # Go up one level
cd ../..             # Go up two levels
cd /path/to/dir      # Absolute path
cd relative/path     # Relative path
cd "$OLDPWD"         # Same as cd -
cd "$(mktemp -d)"    # Go into a newly created temp directory (careful!)
```

The `cd` command updates `$OLDPWD` and sets `$PWD` to the new directory. Both are shell variables you can inspect:

```bash
echo "I was at: $OLDPWD"
echo "I am at: $PWD"
```

### `realpath` — Resolve Path to Absolute

```bash
realpath [options] PATH...
realpath -m, --canonicalize-missing   # Resolve even if path doesn't exist yet
realpath -e, --canonicalize-existing  # Resolve only if all components exist (default)
realpath -s, --strip                  # Don't resolve symlinks, just make absolute
realpath -z, --zero                   # NUL-terminate output (for xargs -0)
realpath --relative-to=DIR            # Show relative path from DIR
realpath --relative-base=DIR          # Show relative if under DIR, else absolute
```

### `readlink` — Read Symlink Target

```bash
readlink file                        # Print value of a symlink
readlink -f file                     # Canonicalize (similar to realpath)
readlink -e file                     # Canonicalize, all components must exist
readlink -m file                     # Canonicalize, allow missing components
readlink -n                          # No trailing newline
readlink -z                          # NUL-terminated output
```

### `dirname` and `basename`

```bash
dirname /usr/bin/ls                  # -> /usr/bin (directory portion)
basename /usr/bin/ls                 # -> ls (filename portion)
basename /usr/bin/ls .txt            # -> ls (strip suffix .txt... doesn't apply here)
basename /usr/bin/ls .sh             # -> ls (strip suffix)
```

These are NOT navigation commands per se, but they process paths — essential in scripts.

### `tree` — Directory Tree

```bash
tree [options] [directory]
tree -L N              # Only go N levels deep
tree -d                # Show only directories
tree -a                # Show hidden files too
tree -s                # Show file sizes
tree -h                # Human-readable sizes
tree -p                # Show permissions
tree -u                # Show user/UID
tree -g                # Show group/GID
tree -D                # Show last modified date
tree -F                # Append / * @ | for type indicators
tree -f                # Print full path prefix for each file
tree -o file           # Output to file (helpful for docs)
tree -I pattern        # Exclude files matching pattern (e.g., -I '*.pyc')
tree --prune           # Prune empty directories from output
tree --dirsfirst       # Sort directories before files
tree -C                # Colorize output
tree --inodes          # Show inode numbers
tree --device          # Don't cross filesystem boundaries
```

### `pushd`, `popd`, `dirs` — Directory Stack

```bash
pushd /tmp             # Push /tmp onto stack and cd to it
pushd +N               # Rotate stack so entry N is on top, cd there
pushd -N               # Same, but from end of stack
popd                   # Pop top of stack and cd there
popd +N                # Remove entry N from stack without cd'ing
dirs                   # Print directory stack
dirs -v                # Print stack with line numbers (for pushd +N)
dirs -c                # Clear the stack
dirs -l                # Print with full paths (no ~ abbreviation)
```

Your typical workflow:

```bash
cd /var/log               # Go somewhere
pushd /etc                # Save current, go to /etc
... work in /etc ...
popd                      # Back to /var/log
```

The stack is stored in `$DIRSTACK`. Look at it: `echo $DIRSTACK`.

## Under the Hood

### What happens when you type `ls`

1. **Shell reads** `ls -la /home/phd` and parses it into a command + arguments.
2. **The shell checks** if `ls` is a builtin (it's not; `cd` is, but `ls` is not).
3. **PATH lookup**: The shell searches `$PATH` (e.g., `/usr/local/bin:/usr/bin:/bin`) for an executable named `ls`. Found at `/usr/bin/ls`.
4. **`fork()`**: The shell creates a child process (a near-perfect copy of itself).
5. **`execve("/usr/bin/ls", ["ls", "-la", "/home/phd"], envp)`**: The child replaces itself with the `ls` program. The kernel loads the ELF binary, sets up memory, maps shared libraries (libc, etc.).
6. **`ls` starts**: It calls `getopt()` to parse options `-la` (which breaks into `-l` and `-a`).
7. **`stat("/home/phd")`**: `ls` calls the `stat()` syscall to learn about the target. The kernel's Virtual Filesystem Switch (VFS) dispatches to the underlying filesystem driver (ext4, XFS, btrfs, etc.).
8. **`open("/home/phd", O_RDONLY|O_DIRECTORY)`**: `ls` opens the directory to read its entries. The kernel returns a file descriptor (FD 3 or whatever's next available).
9. **`getdents(fd, buf, size)`**: `ls` repeatedly calls `getdents()` (or `getdents64()` on 64-bit systems) to read directory entries. Each call returns multiple `struct linux_dirent` entries containing:
   - `d_ino` — inode number
   - `d_off` — offset to next entry
   - `d_reclen` — length of this entry
   - `d_type` — file type (DT_REG, DT_DIR, DT_LNK, etc.)
   - `d_name` — filename (null-terminated)
10. **`lstat()` per entry** (when `-l` is used): For each entry, `ls` calls `lstat()` to get inode metadata: permissions, size, timestamps, owner, group, number of hard links, etc. If `-L` (dereference) is given, it uses `stat()` instead, which follows symlinks.
11. **Formatting**: `ls` formats the output buffer, sorting entries (alphabetically by default, or by time with `-t`, or size with `-S`). It aligns columns by measuring the longest entry.
12. **`write(1, buf, len)`**: `ls` writes the formatted output to stdout (file descriptor 1). The terminal driver (or next pipe) receives it.
13. **`exit(0)`**: `ls` exits with status 0. The shell receives SIGCHLD, collects the exit status via `waitpid()`, and prints the next prompt.

### System calls involved in `ls -la /some/dir`

```
stat("/some/dir", {st_mode=S_IFDIR|0755, ...}) = 0
openat(AT_FDCWD, "/some/dir", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, /* 128 entries */, 32768) = 2048
newfstatat(AT_FDCWD, ".", {st_mode=S_IFDIR|0755, ...}, AT_SYMLINK_NOFOLLOW) = 0
newfstatat(AT_FDCWD, "..", {st_mode=S_IFDIR|0755, ...}, AT_SYMLINK_NOFOLLOW) = 0
newfstatat(AT_FDCWD, "file1", {st_mode=S_IFREG|0644, ...}, AT_SYMLINK_NOFOLLOW) = 0
... (for each entry) ...
getdents64(3, /* 0 entries */, 32768) = 0
close(3) = 0
write(1, "total 128\ndrwxr-xr-x  ...\n", ...) = ...
exit_group(0) = ?
```

### What happens when you type `cd /tmp`

This is different — `cd` is a **shell builtin**, so no `fork()` or `execve()` happens.

1. The shell's `cd` builtin receives `/tmp` as an argument.
2. It calls the internal function `chdir("/tmp")` — a syscall that changes the kernel's notion of the current working directory for this process.
3. The kernel updates `current->fs->pwd` (the process's `fs_struct.pwd` dentry/vfsmount pair).
4. The shell updates its internal variables:
   - `OLDPWD=$PWD` (save the old directory)
   - `PWD=/tmp` (set the new directory)
5. If `/tmp` doesn't exist or is inaccessible, `chdir()` returns -1, the shell prints an error, and `PWD`/`OLDPWD` are NOT updated.
6. The shell prints the new prompt (which may include the new `$PWD` depending on `$PS1`).

### The inode and dentry model

The kernel separates file metadata from file names:

- **inode**: Stores all metadata (owner, permissions, size, timestamps, data block pointers) but NOT the name.
- **dentry** (directory entry): Maps a name to an inode. What `ls` reads via `getdents64()`.
- A single inode can have multiple dentries (hard links) — different names, same data.

This is why `..` is a hard link to the parent directory's inode — it's an entry in every directory that points back up.

### File descriptor implications of navigation

When you `cd ..`, the kernel doesn't actually "go up" in some abstract space. The process's `pwd` dentry is replaced by a reference to the parent dentry. If any process has an open file descriptor that refers to a subdirectory, that subdirectory's inode remains alive (not freed) even if it's been unlinked — one reason you sometimes can't unmount a filesystem.

## Core Examples

### Example 1: Where in the world am I?

```bash
$ pwd
/home/phd/Desktop/KaliStuff/Learning
```

**Step by step:**
1. You typed `pwd`. The shell's builtin `pwd` executes (no fork).
2. It reads the process's current working directory from the kernel (`/proc/self/cwd` symlink) or from `$PWD` (if using logical path).
3. It prints the path to stdout.

**What if:**
```bash
$ cd /var/log
$ pwd
/var/log
$ (cd /tmp; pwd)    # Subshell — pwd shows /tmp, but parent shell is still /var/log
/var/log
```

### Example 2: The anatomy of `ls -la`

```bash
$ ls -la /home/phd
total 48
drwxr-xr-x 15 phd  phd  4096 Jul 31 01:00 .
drwxr-xr-x  3 root root  4096 Jul 31 01:00 ..
-rw-------  1 phd  phd   220 Jul 31 01:00 .bash_logout
-rw-r--r--  1 phd  phd  3771 Jul 31 01:00 .bashrc
drwxr-xr-x  2 phd  phd  4096 Jul 31 01:00 Desktop
```

**Step by step:**
1. `ls` opens `/home/phd` with `openat(AT_FDCWD, "/home/phd", O_RDONLY|O_DIRECTORY)` -> FD 3.
2. It calls `getdents64(3, buf, 32768)` — the kernel returns directory entries.
3. For each entry, it calls `newfstatat(AT_FDCWD, entry_name, &stat, AT_SYMLINK_NOFOLLOW)`.
4. It formats `total 48` (sum of 512-byte blocks used by all files).
5. It formats each line: mode, link count, owner, group, size in bytes, mtime, name.
6. It sorts (default: alphabetically, `.` first, then `..`, then dotfiles, then others).
7. It calls `write(1, buf, len)` to output everything.

**What if:**
```bash
$ ls /home/phd                # No -a — hidden files invisible
Desktop  Documents  Downloads
$ ls -A /home/phd             # Almost-all: no . or ..
.bash_logout  .bashrc  Desktop
```

### Example 3: Absolute vs relative paths in action

```bash
$ pwd
/home/phd/Desktop
$ cd /var/log                  # Absolute: starts with /
$ pwd
/var/log
$ cd ../tmp                    # Relative: go up to /var, then into tmp
$ pwd
/var/tmp
$ cd ../../home/phd            # Relative: up twice to /, then into home/phd
$ pwd
/home/phd
```

**Step by step:**
1. `cd /var/log`: Shell calls `chdir("/var/log")`. Kernel looks up `var` in root `/`, then `log` in `var`. The process's pwd updates to `/var/log`.
2. `cd ../tmp`:
   - Shell calls `chdir("../tmp")`.
   - Kernel resolves `..` relative to `/var/log` -> `/var`.
   - Then resolves `tmp` in `/var` -> `/var/tmp`.
3. `cd ../../home/phd`:
   - Kernel resolves `..` from `/var/tmp` -> `/var`.
   - Resolves `..` from `/var` -> `/`.
   - Resolves `home` in `/` -> `/home`.
   - Resolves `phd` in `/home` -> `/home/phd`.

**What if:**
```bash
$ cd /var/log && cd ../tmp     # Success (both exist)
$ cd /var/log && cd ../nonexistent  # Second cd fails, stays in /var/log
bash: cd: ../nonexistent: No such file or directory
```

### Example 4: The `-` back-jump

```bash
$ pwd
/home/phd/Desktop
$ cd /tmp
$ pwd
/tmp
$ cd -
/home/phd/Desktop
$ cd -
/tmp
```

**Step by step:**
1. `cd /tmp` sets `OLDPWD=/home/phd/Desktop`, `PWD=/tmp`.
2. `cd -` is equivalent to `cd "$OLDPWD"`. It swaps: now `OLDPWD=/tmp`, `PWD=/home/phd/Desktop`.
3. `cd -` again toggles back.

**What if:**
```bash
$ cd -        # No previous? OLDPWD is unset?
bash: cd: OLDPWD not set
```
`OLDPWD` only exists after you've done at least one `cd`.

### Example 5: Symlink resolution with pwd

```bash
$ ls -l /proc/self/cwd
lrwxrwxrwx 1 phd phd 0 Jul 31 01:00 /proc/self/cwd -> /home/phd/Desktop
$ cd /proc/self/cwd
$ pwd
/proc/self/cwd
$ pwd -P
/home/phd/Desktop
```

**Step by step:**
1. `cd /proc/self/cwd`: The kernel follows the symlink and sets the working directory to `/home/phd/Desktop`. But bash's `pwd` by default shows the **logical** path (what you typed), not the physical one.
2. `pwd -P` asks the kernel directly via `getcwd()`, which returns the resolved physical path.

**What if:**
```bash
$ set -o physical              # Make all pwd and cd use physical paths
$ cd /proc/self/cwd
$ pwd
/home/phd/Desktop
```

### Example 6: The `tree` of an application directory

```bash
$ tree -L 2 /var/www/myapp
/var/www/myapp
├── app
│   ├── controllers
│   ├── models
│   └── views
├── config
│   ├── database.yml
│   └── routes.rb
├── public
│   ├── images
│   ├── index.html
│   └── stylesheets
└── Gemfile
```

**Step by step:**
1. `tree` recursively opens each directory, calling `getdents64()` on each.
2. `-L 2` limits recursion to 2 levels deep.
3. It builds an in-memory tree structure, then prints it with Unicode box-drawing characters (`├──`, `└──`, `│`).
4. Files without children get `├──` or `└──` as appropriate. The last entry in a directory gets `└──`.

**What if:**
```bash
$ tree -a -L 1 ~               # Including hidden, 1 level
/home/phd
├── .bashrc
├── .config
├── .local
├── Desktop
└── Documents
```

### Example 7: Using dirname and basename

```bash
$ fullpath="/usr/local/bin/script.sh"
$ dirname "$fullpath"
/usr/local/bin
$ basename "$fullpath"
script.sh
$ basename "$fullpath" .sh
script
```

**Step by step:**
1. `dirname` scans the string from the right, finds the last `/`, returns everything to the left. If no `/`, returns `.`.
2. `basename` strips everything up to and including the last `/`. If a suffix is given (`.sh`), it removes that suffix from the result.

**What if:**
```bash
$ dirname "/usr/local/bin/"    # Trailing slash handled
/usr/local/bin
$ dirname "simplefile"         # No slash -> "."
.
$ basename "/"               # Edge case
/
```

### Example 8: Directory stack in action

```bash
$ pwd
/home/phd
$ pushd /tmp
/tmp ~
$ pushd /var
/var /tmp ~
$ dirs -v
 0  /var
 1  /tmp
 2  ~
$ popd
/tmp ~
$ popd
~
```

**Step by step:**
1. `pushd /tmp`: Saves `/home/phd` on stack, `cd`s to `/tmp`. The stack is now `(/tmp /home/phd)`.
2. `pushd /var`: Saves `/tmp` on stack, `cd`s to `/var`. Stack is `(/var /tmp /home/phd)`.
3. `popd`: Removes `/var` from stack, `cd`s to `/tmp`. Stack is `(/tmp /home/phd)`.
4. `popd`: Removes `/tmp` from stack, `cd`s to `/home/phd`. Stack is `(/home/phd)`.

**What if:**
```bash
$ dirs                 # Clear the stack
$ cd /tmp && cd /var && cd /opt
# No stack involved — just three separate cd commands
$ pushd .              # Push current directory without going anywhere
```

### Example 9: realpath for script robustness

```bash
$ ls -l /usr/bin/python3
lrwxrwxrwx 1 root root 9 Jul 31 01:00 /usr/bin/python3 -> python3.11
$ realpath /usr/bin/python3
/usr/bin/python3.11
$ realpath --relative-to=/home/phd /usr/bin/python3
../../usr/bin/python3.11
$ SCRIPT_DIR=$(dirname "$(realpath "$0")")
```

**Step by step:**
1. `realpath` calls `realpath("/usr/bin/python3")`.
2. The kernel follows each component. `/usr` -> real, `/usr/bin` -> real, `/usr/bin/python3` -> symlink to `python3.11`.
3. By default, `realpath` resolves symlinks and `.` and `..` components, returning the canonical absolute path.

**What if:**
```bash
$ realpath -m /tmp/../nonexistent/../path    # -m: missing components allowed
/path
$ realpath -e /tmp/../nonexistent            # -e: error on missing
realpath: 'nonexistent': No such file or directory
```

### Example 10: Listing with inode numbers

```bash
$ ls -lia /tmp
total 36
 2621441 drwxrwxrwt 12 root root  4096 Jul 31 01:00 .
       2 drwxr-xr-x 20 root root  4096 Jul 31 01:00 ..
 2621450 -rw-r--r--  1 phd  phd     0 Jul 31 01:00 test.txt
```

**Step by step:**
1. `-i` adds the inode number column on the left.
2. Inode 2 is always the root inode of a filesystem (each filesystem has its own root with inode 2).
3. Inodes allow the kernel to find the physical data on disk without reading the filename.
4. Hard-linked files share inode numbers.

**What if:**
```bash
$ echo "data" > /tmp/fileA
$ ln /tmp/fileA /tmp/fileB      # Hard link: same inode
$ ls -li /tmp/fileA /tmp/fileB  # Identical inode numbers
2621450 -rw-r--r-- 2 phd phd 5 Jul 31 01:00 /tmp/fileA
2621450 -rw-r--r-- 2 phd phd 5 Jul 31 01:00 /tmp/fileB
```

### Example 11: The `cd` path search (CDPATH)

```bash
$ export CDPATH=~/projects
$ cd myapp                      # If ~/projects/myapp exists, cd there
~/projects/myapp
$ cd myapp                      # If it doesn't exist under CDPATH, tries $PWD
bash: cd: myapp: No such file or directory
```

**Step by step:**
1. `cd myapp` is a relative path.
2. The shell checks `CDPATH` (colon-separated list) before checking the current directory.
3. First match in `~/projects/myapp` is found -> cd there.
4. If `CDPATH` had multiple dirs, it would check each left to right.
5. If no match found in any `CDPATH` entry, it tries the current directory's `myapp`.

**What if:**
```bash
$ unset CDPATH                  # Back to default behavior
$ cd myapp                      # Only checks current directory
```

### Example 12: The `ls` directory argument vs its contents

```bash
$ ls /usr                       # Lists CONTENTS of /usr
bin  lib  local  sbin  share  src
$ ls -d /usr                    # Lists /usr ITSELF
/usr
$ ls -ld /usr                   # Long format of /usr ITSELF
drwxr-xr-x 6 root root 4096 Jul 31 01:00 /usr
```

**Step by step:**
1. Without `-d`, `ls` opens the directory argument and lists its children.
2. With `-d`, `ls` stats the directory itself and shows only that entry.
3. `-ld` combines: `-l` (long format) + `-d` (directory entry, not contents).

**What if:**
```bash
$ ls /usr /tmp                  # Multiple arguments, lists each
$ ls -d /usr /tmp               # Just the two directory entries
```

## Real-World Use Cases

### 1. FOR the OS — Administration

- **Checking filesystem usage**: `df -h` then `cd` to the fullest partition, `du -sh * | sort -rh | head` to find top disk consumers.
- **Finding where config files live**: `ls -la /etc | grep myapp` or `tree /etc/myapp`.
- **Checking symlink chains**: `readlink -f /etc/alternatives/editor` resolves the chain of alternatives.
- **Scripting with relative paths**: `SCRIPT_DIR=$(cd "$(dirname "$0")" && pwd -P)` — the canonical way to find where your script lives.
- **Auditing PATH**: `echo "$PATH" | tr ':' '\n'` — one directory per line.

### 2. WITH the OS — Development

- **Project navigation**: Create `~/.bashrc` aliases: `alias ..='cd ..'`, `alias ...='cd ../..'`, `alias g='cd ~/git'`.
- **Finding your way in deep trees**: `cd project/src/components/Button` with tab completion — bash's Tab completes ambiguous paths.
- **Directory stack for multi-project work**: `pushd /projectA/src`, work, `popd`, `pushd /projectB/src`.
- **`ls -ltr`** to see recently modified files at the bottom (your editor's last saves).
- **`cd -` for toggling** between test and source directories.

### 3. AGAINST the OS — Exploitation

- **Path traversal attacks**: An attacker uses `../../../etc/passwd` in a web request to read files outside the web root. Understanding path resolution is how you prevent this: always `realpath` user-supplied paths and check they stay within the intended base directory.
- **Symlink races (TOCTOU)**: In a race condition, an attacker swaps a file with a symlink to `/etc/shadow` between the time you check permissions and the time you open it. `realpath` with `O_NOFOLLOW` helps.
- **Hidden file hiding**: Attackers hide malware in dotfiles because `ls` doesn't show them by default. Always `ls -la` in investigations.
- **`/proc` exploration**: `/proc/self/cwd`, `/proc/self/root`, `/proc/self/exe` can leak process information. Processes can be manipulated via `/proc/PID/cwd`.

### 4. FOR DEFENSE — Detection & Auditing

- **Monitor file system changes**: `ls -laR /etc/` (periodically, snapshot) to detect unauthorized changes.
- **`find / -nouser -o -nogroup`** to find orphaned files (potential backdoor staging).
- **Check `.` and `..` permissions**: If `.` is world-writable, users can delete each other's files (sticky bit should be set: `chmod +t /tmp`).
- **Audit suid/sgid files**: `find / -perm -4000 -ls` finds all setuid binaries. Compare with a known-good list.
- **Detect hidden files**: `shopt -s dotglob; ls -la ~/.*` in user home directories for suspicious dotfiles.

## Memory Aids

### Mnemonics

**`ls -ltr`** = "LisT Recently" — shows newest files at the bottom. Easy to scroll and see what you just edited.

**`pwd`** = "Print Working Directory" — literally what it says. Thinking of it as "Where am I?" might be easier.

**`cd -`** = "cd dash" or "cd back" — the dash is like a minus sign taking you back.

**Why it's named `pwd`?** In early Unix, commands were limited to 5 characters (later 8). "Print working directory" was too long. They kept the initials. Similarly, `ls` is "list segment" (from PDP-11 assembly), not "list" — but everyone calls it list.

**Why two dots for parent?** Thompson thought `.` was for current directory, so `..` for its parent was natural extension. The next level up would be `...` but that never worked. Actually, `...` is not a valid path — if you want three levels up, you need `../../..`.

### How to remember navigation shortcuts

| Shortcut | Meaning | Mnemonic |
|----------|---------|----------|
| `.` | This directory | "Dot is right here" |
| `..` | Parent directory | "Two dots = dot dot = back up" |
| `~` | Home | "Tilde looks like a house roof" |
| `-` | Previous | "Dash = back in time" |
| `/` | Root | "Forward slash = the root of the tree" |

### Pattern hooks

- `ls` without args -> "just list files in current dir"
- `ls -l` -> "long listing, I want details"
- `ls -a` -> "all, show me everything including hidden"
- `ls -la` -> "long listing of everything" — most common combo
- `ls -ltr` -> "long listing, time-sorted, reversed" — last modified at bottom
- `ls -d */` -> "just directories, not their contents" (the `*/` glob matches only directories)

### Common confusions

- **`ls -s` vs `ls -S`**: Lowercase `-s` = print size in blocks. Uppercase `-S` = sort by size. Easy to typo.
- **`pwd -P` vs `readlink -f`**: They do the same thing on most systems, but `pwd -P` is a shell builtin and only works on the current directory. `readlink -f` works on any path.
- **`/` is the root, but `~` is NOT the same as `/home/$USER`**: They are if `/home` is a real directory and not a symlink. But `~` is always the user's home directory as defined in `/etc/passwd`.
- **`cd ..` removes one directory from the path, not one level of symlinks**: If you're in `/var/www/link` where `link -> /etc`, `cd ..` goes to `/var/www`, not `/`.

## Trap Vault

### Trap 1: Spaces in directory names

**Problem:** Unquoted or unescaped spaces break navigation.

**Example:**
```bash
$ cd My Documents
bash: cd: too many arguments
```

**Why:** The shell splits command line on whitespace. `cd My Documents` becomes `cd` with two arguments: `My` and `Documents`. `cd` only accepts one directory argument.

**Fix:**
```bash
$ cd "My Documents"
$ cd My\ Documents
```

### Trap 2: `ls` of an empty directory

**Problem:** `ls empty_dir` returns nothing, not even an error.

**Example:**
```bash
$ mkdir /tmp/empty
$ ls /tmp/empty
$                # No output, no error — did it work?
```

**Why:** `ls` considers an empty directory a valid, successful operation. It returns exit code 0, but produces no output.

**Fix:** Use `ls -la` to at least show `.` and `..` entries, confirming the directory is there but empty:
```bash
$ ls -la /tmp/empty
total 0
drwxr-xr-x  2 phd phd  40 Jul 31 01:00 .
drwxrwxrwt 22 root root 640 Jul 31 01:00 ..
```

### Trap 3: `cd` with no arguments doesn't stay put

**Problem:** `cd` with no arguments goes HOME, not nowhere.

**Example:**
```bash
$ cd /tmp
$ cd
$ pwd
/home/phd
```

**Why:** POSIX specifies that `cd` with no operand defaults to `$HOME`. This is by design — it's the fastest way to get home.

**Fix:** To go nowhere (stay in current dir), use `cd .` or just don't call `cd`. If you want to avoid accidentally going home in a script, use `cd "${DIR:-.}"` to default to current dir.

### Trap 4: `-` is not positional — only remembers ONE previous directory

**Problem:** `cd -` doesn't cycle through history, it just toggles last two.

**Example:**
```bash
$ cd /tmp
$ cd /var
$ cd /opt
$ cd -
/var
$ cd -
/opt
$ cd -
/var
```

**Why:** `$OLDPWD` stores only the immediately previous directory. Every `cd` updates `$OLDPWD` to the previous value of `$PWD`. So `cd -` toggles between two most recent directories. Use `pushd`/`popd` for a stack.

**Fix:**
```bash
$ pushd /tmp && pushd /var && pushd /opt
$ popd && popd && popd
```

### Trap 5: Hidden files and `ls` scope

**Problem:** Plain `ls` doesn't show hidden files, leading to "file not found" confusion.

**Example:**
```bash
$ cd ~
$ ls
Desktop  Documents  Downloads
$ cat .bashrc                                 # Works fine with explicit name
$ ls *.bashrc                                  # No match — glob doesn't match hidden
ls: cannot access '*.bashrc': No such file or directory
```

**Why:** Glob `*` does NOT match filenames starting with `.` unless `dotglob` is set. This prevents accidental mass-operations on dotfiles. But it also means you can forget they exist.

**Fix:**
```bash
$ shopt -s dotglob
$ ls *
.bashrc  .config  Desktop  Documents  Downloads
$ ls -a                                          # Simpler
$ echo .*                                         # Matches . and .. too — be careful!
```

### Trap 6: Symlink loops in the filesystem

**Problem:** A symlink pointing back to its own ancestor creates an infinite loop.

**Example:**
```bash
$ mkdir -p /tmp/loop
$ ln -s /tmp/loop /tmp/loop/self
$ ls /tmp/loop/self/self/self/self
# Infinite output or "too many levels of symbolic links"
```

**Why:** The kernel detects symlink loops at a depth of 40 (Linux `MAXSYMLINKS`). After 40 symlink resolutions, it gives up with `ELOOP`.

**Fix:** Use `realpath` to resolve before navigating:
```bash
$ realpath /tmp/loop/self/self/self
realpath: /tmp/loop/self/self/self: Too many levels of symbolic links
```

### Trap 7: `ls -R` on large trees is unfriendly

**Problem:** `ls -R` on `/usr` produces thousands of lines without clear structure.

**Example:**
```bash
$ ls -R /usr | head -20
bin:
bash
cat
chmod
cp
...
# No tree structure, just flat listing of each subdirectory
```

**Why:** `ls -R` lists each directory's contents with just the directory name as a heading. It's hard to read. `tree` is much better for visualization.

**Fix:**
```bash
$ tree -L 2 /usr
$ find /usr -maxdepth 2 -type d | sort
```

### Trap 8: `cd` into a broken symlink

**Problem:** You can `cd` to a symlink, but if the target disappears, you're stuck.

**Example:**
```bash
$ ln -s /tmp/realdir ~/mylink
$ mkdir /tmp/realdir
$ cd ~/mylink
$ pwd
/home/phd/mylink
$ rmdir /tmp/realdir
$ cd .                            # Try to refresh? No error.
$ pwd
/home/phd/mylink                  # Still there!
$ ls
bash: ls: No such file or directory
$ pwd -P
pwd: error retrieving current directory: getcwd: cannot access parent directories
```

**Why:** The kernel's `pwd` dentry still references the (now-deleted) directory. The shell's `$PWD` still holds the logical path. But any operation that resolves the path fails because the target is gone. You're in limbo.

**Fix:** `cd /tmp` (absolute path works since it doesn't go through the broken link). Or `cd $(pwd -P)` if `pwd -P` can still resolve it (it can't in this case).

### Trap 9: `dirname` and `basename` with `~`

**Problem:** `~` expansion happens before command execution, but not inside strings.

**Example:**
```bash
$ dirname ~/Documents
/home/phd/Documents               # Works!
$ dirname "~/Documents"
.                                 # Oops!
```

**Why:** `~` expansion is a shell feature, not a command feature. Inside double quotes, `~` is literal. `dirname` sees `~/Documents` as a relative path and returns `.`.

**Fix:**
```bash
$ dirname ~/"Documents"           # Works because ~ is outside quotes
$ dirname "$HOME/Documents"       # Even safer
```

### Trap 10: `ls` sorts differently in different locales

**Problem:** In non-C locales, `ls` sorting may put lowercase before uppercase, or ignore case entirely.

**Example:**
```bash
$ LANG=en_US.UTF-8 ls
Apple  banana  Cherry
$ LANG=C ls
Apple  Cherry  banana
```

**Why:** In `en_US.UTF-8`, collation rules may ignore case or sort differently than ASCII byte order. The `C` locale sorts by raw byte values, where uppercase letters (65-90) come before lowercase (97-122).

**Fix:** For consistent behavior across systems, prefix with `LC_ALL=C`:
```bash
$ LC_ALL=C ls /some/dir           # Forces byte-by-byte sorting
```

## See It In The Wild

### Where you encounter path navigation daily

- **Your shell prompt**: Most prompts show `\w` (current directory), but shortened — `~` for home, or just basename. Configure via `$PS1`: `export PS1='\u@\h:\w\$ '`.
- **`tab` completion**: Bash's programmable completion. Try `cd /v` + Tab -> `/var/`.
- **`/proc/self/cwd`**: Every process has this symlink to its current directory. `ls -l /proc/$$/cwd` shows your shell's current directory from the kernel's perspective.
- **`lsof`**: Shows open files. `lsof +D /var/log` lists everything currently open under that directory.
- **`find`** and **`locate`**: These find files by name/type, not by navigating interactively. But they rely on the same path resolution.

### How to observe navigation with strace

```bash
$ strace -e trace=chdir,fchdir pwd 2>&1 | tail -5
$ strace -e trace=getdents64,newfstatat ls -la /tmp 2>&1 | head -30
$ strace -f -e trace=chdir bash -c 'cd /tmp; cd /var; pwd' 2>&1
```

The last command shows that `cd` calls `chdir()` syscall, while `pwd` doesn't (it reads `$PWD`):

```
chdir("/tmp")                           = 0
chdir("/var")                           = 0
write(1, "/var\n", 5)                   = 5
```

### Real files that demonstrate filesystem concepts

- **`/etc/fstab`**: The filesystem table — shows what mounts where and with what options.
- **`/etc/mtab`**: Currently mounted filesystems (often a symlink to `/proc/mounts`).
- **`/proc/mounts`**: Kernel's view of mounts. Shows mount points as the kernel sees them.
- **`/sys/block/`**: Sysfs — shows block devices in a hierarchy. The sysfs filesystem itself is a demonstration of kernel object hierarchy.
- **`/run`**: A tmpfs (memory-backed filesystem) on modern Linux — files here are fast but volatile.

### Try this now

```bash
# Watch chdir in action
strace -e chdir bash -c 'cd /tmp; cd /var; cd /opt 2>/dev/null; pwd'

# See where your process lives in /proc
ls -l /proc/$$/cwd

# Explore the hidden directory entries
xxd /tmp | head -20          # Raw directory entry data

# Count all symlinks on your system
find /usr -type l | wc -l

# Find the longest path
find /usr -printf '%p\0' | tr '\0' '\n' | awk '{ print length, $0 }' | sort -rn | head -5

# The actual kernel source for path resolution:
# fs/namei.c — the "path walking" code. Over 5000 lines of C.
```

## Check Your Understanding

<details>
<summary>1. You're in `/usr/local/bin`. You type `ls ../../share`. What directory's contents will you see?</summary>

`/usr/share`. `../../share` resolves from `/usr/local/bin`:
- `..` from `/usr/local/bin` -> `/usr/local`
- `..` from `/usr/local` -> `/usr`
- `share` from `/usr` -> `/usr/share`
</details>

<details>
<summary>2. You run `cd /var/log; cd ../..`. Where are you?</summary>

`/`. `/var/log/..` is `/var`, then `../..` from `/var/log` is `/`. So you're at root!
</details>

<details>
<summary>3. Your script sets `cd /some/path || exit 1`. Why might `cd` fail even though the directory exists?</summary>

Permissions. You need `+x` (execute/search) permission on the directory AND all ancestor directories. If `/some` exists but is owned by root with mode 0700, you can't descend into it. Also, a component along the path could be a broken symlink, or the filesystem could be read-only.
</details>

<details>
<summary>4. What is the difference between `ls -la /etc` and `ls -la /etc/`?</summary>

None whatsoever. A trailing slash in a path just means "this is a directory."
</details>

<details>
<summary>5. After running `cd /tmp`, `OLDPWD` is still unset. Why?</summary>

`OLDPWD` is only set after you've done a `cd` that changed directories. In bash, `OLDPWD` is set after the first `cd`. Before any `cd`, `OLDPWD` might not exist.
</details>

<details>
<summary>6. You type `pwd` and it shows `/home/user/project`. But the kernel shows a different path. How can you see the kernel's version?</summary>

Use `pwd -P` (physical) or read `/proc/self/cwd` via `readlink`: `readlink /proc/self/cwd`. The difference is symlinks in the path.
</details>

<details>
<summary>7. Why does `cd ../sibling_dir` not work when you're in a directory that was reached via a symlink?</summary>

If you're at `/var/www/link` where `link -> /opt/app`, then `cd ../sibling_dir` resolves `..` against the *logical* path `/var/www`. But `sibling_dir` might not exist in `/var/www`. In physical mode (`set -o physical`), `..` resolves against `/opt`.
</details>
