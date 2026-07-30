# Task 2: Project Scaffolding with File Operations

Build a project directory structure, manipulate files, create links, and learn why `rm -rf` is terrifying. You'll use `mkdir`, `touch`, `cp`, `mv`, `rm`, `ln`, and `file`.

## Setup

```bash
$ mkdir -p /tmp/fileops_lab
$ cd /tmp/fileops_lab
```

## Sub-task 1: The One-Liner Directory Tree

Use a single `mkdir -p` command with brace expansion to create this structure:

```
project/
├── src/
│   ├── main/
│   └── tests/
├── docs/
│   ├── api/
│   └── guides/
├── config/
│   ├── dev/
│   └── prod/
├── scripts/
│   ├── build/
│   └── deploy/
└── data/
    ├── raw/
    └── processed/
```

**Expected command:**
```bash
$ mkdir -p project/{src/{main,tests},docs/{api,guides},config/{dev,prod},scripts/{build,deploy},data/{raw,processed}}
```

**Verify with:** `tree project`

<details>
<summary>Brace expansion gotchas</summary>
Brace expansion happens BEFORE glob expansion. `{a,b}/{c,d}` becomes `a/c a/d b/c b/d`. No special characters needed. Nesting: `{a,{b,c}}` works but is confusing. Just flatten: `{a,b,c}`.
</details>

## Sub-task 2: Touch Zone

Create these files within the tree:

```bash
$ touch project/src/main/app.py
$ touch project/src/main/utils.py
$ touch project/src/tests/test_app.py
$ touch project/docs/api/README.md
$ touch project/docs/guides/getting-started.md
$ touch project/config/dev/config.yaml
$ touch project/config/prod/config.yaml
$ touch project/scripts/build/build.sh
$ touch project/scripts/deploy/deploy.sh
```

Now verify with `tree project` (should show files too).

**Question:** How many files did you create? How many directories?

<details>
<summary>Counting</summary>
You created 8 files across the tree. The directories were 11 (project + 10 subdirs). Total: 19 entries.
</details>

## Sub-task 3: Copy and Rename Challenge

Perform these operations in sequence:

```bash
# 1. Copy app.py to backup/
$ cp project/src/main/app.py project/data/raw/app.py.bak

# 2. Copy the entire scripts/ directory to a new location
$ cp -r project/scripts project/scripts_backup

# 3. Rename getting-started.md to index.md
$ mv project/docs/guides/getting-started.md project/docs/guides/index.md

# 4. Copy config/dev/config.yaml to config/prod/ (overwriting)
$ cp project/config/dev/config.yaml project/config/prod/config.yaml

# 5. Now watch what happens when you copy a directory to an existing dir:
$ mkdir project/backup
$ cp project/src project/backup/
$ tree project/backup
```

**Question:** Why is `project/backup/src` nested inside `project/backup`? How would you copy the *contents* of `src/` into `backup/` directly?

<details>
<summary>Copying directory contents</summary>
`cp -rT src dest` copies contents of src into dest. Or `cp -r src/. dest` (the `/.` trick means "the contents of this directory").
</details>

## Sub-task 4: Link Master

Create links and understand their behavior:

```bash
# Create a hard link
$ ln project/src/main/app.py project/src/main/app.py.hard

# Create a symbolic link
$ ln -s project/src/main/app.py project/src/main/app.py.sym

# Compare inodes
$ ls -li project/src/main/
```

**Expected output:**
```
total 0
1234567 -rw-r--r-- 2 phd phd 0 Jul 31 01:00 app.py
1234567 -rw-r--r-- 2 phd phd 0 Jul 31 01:00 app.py.hard
1234568 lrwxrwxrwx 1 phd phd 5 Jul 31 01:00 app.py.sym -> app.py
```

Note the identical inode numbers for `app.py` and `app.py.hard`. The link count shows `2` for both.

Now test link behavior when original is removed:

```bash
$ rm project/src/main/app.py
$ cat project/src/main/app.py.hard   # Works!
$ cat project/src/main/app.py.sym    # Broken — what error?
$ file project/src/main/app.py.sym   # What does file say?
```

**Question:** Why does the hard link survive but the symlink breaks?

<details>
<summary>Link survival explanation</summary>
Hard links share the same inode. Removing one name decrements the link count but doesn't free the inode until the count reaches zero. Symlinks store a path string — they don't care about inodes. When the target path disappears, the symlink dangles.
</details>

## Sub-task 5: File Type Detectives

Use `file` to identify everything:

```bash
$ file project/src/main/app.py.hard
$ file project/src/main/app.py.sym
$ file project/scripts/build/build.sh
$ file /bin/ls
$ file /dev/null
$ file /tmp
```

Now create files with deceptive content and see if `file` catches them:

```bash
$ echo "#!/bin/bash" > /tmp/fake.txt
$ echo "echo 'not a script'" >> /tmp/fake.txt
$ file /tmp/fake.txt
$ echo -e '\x89PNG\r\n\x1a\n' > /tmp/fake.png
$ file /tmp/fake.png
```

**Question:** Can you fool `file` into thinking a text file is a JPEG? What about an ELF binary?

<details>
<summary>Spoofing file</summary>
JPEG magic bytes are `ff d8 ff`. ELF is `7f 45 4c 46`. Prepending these to any file changes what `file` reports. This is a common malware evasion technique.
</details>

## Sub-task 6: Atomic Config Replacement

Simulate a production config replacement:

```bash
$ echo "config_version=1" > project/config/prod/config.yaml
$ cat project/config/prod/config.yaml

# Simulate editing: create new version alongside
$ echo "config_version=2" > project/config/prod/config.yaml.new

# Now atomically replace
$ mv project/config/prod/config.yaml.new project/config/prod/config.yaml
$ cat project/config/prod/config.yaml
```

**Why this pattern:** Directly overwriting a file with `>` or `cp` risks partial writes. A process reading the file at that moment sees truncated or mixed content. `mv` is atomic — a reader either sees the old file or the new file, never a mangled combination.

**Question:** How could you verify this atomicity with a script that reads the file in a loop while another script updates it?

## Sub-task 7: The `rm` Gauntlet

Create a cleanup scenario:

```bash
$ mkdir -p /tmp/cleanup_test
$ cd /tmp/cleanup_test
$ touch report-2024-01.pdf report-2024-02.pdf report-2024-03.pdf
$ touch temp-001.tmp temp-002.tmp temp-003.tmp
$ mkdir archive
$ touch archive/old-data.tar.gz

# Task: safely remove only the .tmp files, with confirmation
$ rm -i *.tmp

# Task: remove the archive directory and contents
$ rm -r archive

# DANGER ZONE: What does this do?
$ DIR=""
$ rm -rf $DIR/*           # DON'T ACTUALLY RUN THIS
```

**Question:** What mechanisms protect against `rm -rf /` in modern systems?

<details>
<summary>Protection mechanisms</summary>
1. `--preserve-root` (default in coreutils): refuses to operate on `/`.
2. But `rm -rf /*` bypasses this (it's not `/` as an argument, it's everything inside).
3. The glob `/*` must expand first — if the shell runs out of memory for the argument list, it won't even execute.
4. Best protection: never run `rm -rf` with variables, always double-check paths.
</details>

## Bonus Challenge: The Symlink Maze

Create a web of symlinks and navigate them:

```bash
$ mkdir -p /tmp/maze/{room1,room2,room3}
$ echo "treasure" > /tmp/maze/room1/chest.txt
$ ln -s /tmp/maze/room1/chest.txt /tmp/maze/room2/clue.txt
$ ln -s ../room2/clue.txt /tmp/maze/room3/hint.txt
$ ln -s /tmp/maze/room3/hint.txt /tmp/maze/final_clue

# Follow the chain:
$ realpath /tmp/maze/final_clue
$ readlink -f /tmp/maze/final_clue

# Now create a LOOP:
$ ln -s /tmp/maze/room1 /tmp/maze/room1/self
$ realpath /tmp/maze/room1/self/self/self   # What happens?
```

**Question:** How many symlinks deep can the kernel follow before giving up?

<details>
<summary>Kernel symlink limit</summary>
Linux limits symlink resolution to 40 (MAXSYMLINKS in fs/namei.c). After that, it returns ELOOP ("Too many levels of symbolic links"). This prevents infinite loops from freezing the system.
</details>

## Self-Check

1. What's the difference between `cp -r src dest` and `cp -rT src dest` when `dest` exists and is a directory?
2. You want to move a 10GB file from ext4 to an NFS mount. Will `mv` be instant or slow? Why?
3. What does `touch -r reference.txt target.txt` do?
4. A colleague says "I'll just `rm -rf .*` to clean up hidden files." Why is this dangerous?
5. Your script runs `cp /etc/shadow /tmp/` and fails. What permissions do you need? What's missing?
6. After `ln a b`, `ls -li` shows both have inode 123456. You delete `a`. Does `b` still contain the data?
7. How does `file` determine that a file is a JPEG vs a PDF vs a shell script?
