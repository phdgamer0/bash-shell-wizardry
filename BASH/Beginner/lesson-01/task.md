# Task 1: Map the Filesystem

Your mission: explore `/usr`, `/var`, `/etc`, and your home directory. You'll use `pwd`, `cd`, `ls`, `tree`, `realpath`, `readlink`, and the directory stack. By the end you should have the filesystem layout burned into your brain.

## Setup

No setup needed — these directories are already on your system. Pick a comfortable starting point (home is good: `cd ~`).

## Sub-task 1: Your Coordinates

Run `pwd` and note your current location. Then run `pwd -P` and see if there's a difference. If you're not in a symlinked directory, they'll match.

```bash
$ pwd
/home/phd
$ pwd -P
/home/phd
$ ls -la /proc/$$/cwd    # Kernel's view of your cwd
```

**What to record:** Note the path. Is there a difference between `pwd` and `pwd -P`? Why or why not?

<details>
<summary>Why they might differ</summary>
`pwd -P` resolves all symlinks. `pwd` keeps the "logical" path you used to get here. If you `cd /proc/self/cwd`, `pwd` shows `/proc/self/cwd` but `pwd -P` shows the real path.
</details>

## Sub-task 2: Exploring `/usr` with tree and ls

Run these and compare:

```bash
$ tree -L 1 /usr
$ ls /usr
$ ls -la /usr
$ ls -la /usr/ | head -5   # Just the top 5 entries
```

**Expected output:**
```
/usr
├── bin
├── lib
├── local
├── sbin
├── share
└── src
```

Now go deeper — use `tree -L 2 /usr | head -30` to see the first 30 lines of the second level. Try `ls -lR /usr/bin | head` and see how overwhelming recursive ls is without a tree view.

**Question:** How many subdirectories are directly under `/usr`? Count them with `ls -d /usr/*/`.

<details>
<summary>Hint: counting directories</summary>
`ls -d /usr/*/` lists every entry that's a directory (the trailing `/` in the glob restricts to dirs). Pipe to `wc -l`.
</details>

## Sub-task 3: Resolve the Chain

Find where `python3` lives on your system:

```bash
$ which python3
$ file $(which python3)
$ readlink -f $(which python3)
$ realpath $(which python3)
$ ls -la $(which python3)
```

**Expected output:**
```
$ which python3
/usr/bin/python3
$ file /usr/bin/python3
/usr/bin/python3: symbolic link to python3.11
$ realpath /usr/bin/python3
/usr/bin/python3.11
```

Now try with multiple resolution steps. If `/usr/bin/python3 -> python3.11` and `python3.11` is a real binary, then `realpath` shows the final target. If `python3.11` itself is a symlink, it chains further.

**Question:** What other `/usr/bin` entries are symlinks? Count them: `ls -la /usr/bin | grep '^l' | wc -l`.

## Sub-task 4: Navigation Shortcuts Challenge

Start from your home directory. Execute these moves and write down where you end up after each:

```bash
$ cd /var/log
$ cd ../tmp
$ pwd
$ cd ../../home
$ pwd
$ cd ~
$ cd -
$ cd -
```

**Expected output:**
```
$ cd /var/log && pwd
/var/log
$ cd ../tmp && pwd
/var/tmp
$ cd ../../home && pwd
/home
$ cd ~ && pwd
/home/phd
$ cd - && pwd
/home
$ cd - && pwd
/home/phd
```

Now try the directory stack:

```bash
$ pushd /tmp
$ pushd /var
$ dirs -v
$ popd
$ popd
$ dirs
```

**Question:** What does `dirs -c` do? Try it. What does `pushd .` do (without arguments)?

<details>
<summary>Hint: pushd with no args</summary>
`pushd` (no args) swaps the top two directories on the stack. `pushd .` adds current directory to stack without changing directory.
</details>

## Sub-task 5: Hidden File Hunt

Count hidden files in your home directory:

```bash
$ ls -la ~ | grep '^\.' | wc -l
$ ls -la ~ | grep '^\.' | head   # See which ones
$ find ~ -maxdepth 1 -name '.*' | wc -l   # Alternative count
```

**Question:** Why does `echo .*` often list `.` and `..` (which you probably don't want to count)? How can you exclude them?

<details>
<summary>Excluding . and ..</summary>
Use `echo .[!.]*` to get dotfiles that start with `.` followed by a non-dot character. Or use `ls -A` (almost-all).
</details>

## Sub-task 6: Symlink vs Real Path

Create a playground and explore symlink resolution:

```bash
$ mkdir -p /tmp/navtest/real
$ ln -s /tmp/navtest/real /tmp/navtest/link
$ cd /tmp/navtest/link
$ pwd
$ pwd -P
$ ls -la /proc/$$/cwd
```

Now, while inside `/tmp/navtest/link`, run:
```bash
$ cd ..
$ pwd
```

Where did you end up? Is it `/tmp/navtest` or `/tmp/navtest/real/..` -> `/tmp/navtest`? Both resolve to the same place in this case, but the *logical* path is `/tmp/navtest/link/..` -> `/tmp/navtest`.

**Question:** How would `cd -P ..` behave differently from `cd ..`?

<details>
<summary>cd -P vs logical cd</summary>
`cd -P ..` resolves the physical path first. If `link` points to `/tmp/navtest/real`, then `cd -P ..` from inside `link` goes to `/tmp/navtest` (because `/tmp/navtest/real/..` = `/tmp/navtest`). In default mode, `cd ..` also goes to `/tmp/navtest` (because `/tmp/navtest/link/..` = `/tmp/navtest`). They match here. They'd differ if the symlink pointed to a completely different tree.
</details>

## Sub-task 7: The `CDPATH` Experiment

Learn how `CDPATH` changes navigation:

```bash
$ mkdir -p /tmp/cdpath_test/subdir
$ export CDPATH=/tmp/cdpath_test
$ cd subdir          # Should find /tmp/cdpath_test/subdir
$ pwd
```

Now unset CDPATH and try again:

```bash
$ unset CDPATH
$ cd subdir          # Should fail unless you're already in /tmp/cdpath_test
bash: cd: subdir: No such file or directory
```

**Question:** Why might you want to set `CDPATH`? When might it be dangerous?

<details>
<summary>CDPATH pros/cons</summary>
`CDPATH` is great for quickly jumping to common project directories. It's dangerous if you have similarly named directories in different places and you `cd` into the wrong one. Also, in scripts, `CDPATH` can cause unexpected behavior — many scripts `unset CDPATH` or `export CDPATH=` at the start.
</details>

## Bonus Challenge: The `pushd` Matrix

Create a directory matrix and use the stack to navigate it efficiently:

```bash
$ mkdir -p /tmp/matrix/{a,b,c}/{1,2,3}
$ pushd /tmp/matrix/a/1
$ pushd /tmp/matrix/b/2
$ pushd /tmp/matrix/c/3
$ dirs -v
```

Now, without typing any full paths, navigate back to `/tmp/matrix/a/1` in one command (using `pushd +N`).
Then navigate to `/tmp/matrix/b/2` in one command.
Return to your original directory.

**Question:** What's the maximum stack depth? Try `for i in {1..100}; do pushd /tmp; done; dirs | wc -w`.

<details>
<summary>Stack depth</summary>
Bash's stack is limited only by memory. You can `pushd` hundreds of times, but `dirs` strips consecutive duplicates when displaying. `popd` will unwind them one by one.
</details>

## Self-Check

1. What's the difference between `ls -la` and `ls -A`? When would you use each?
2. You're in `/usr/local/bin`. Write the shortest absolute and relative `cd` commands to get to `/var/log`.
3. Why does `cd --` do nothing? What is `--` in command-line parsing?
4. How many `..` would you need to reach `/` from `/a/b/c/d/e/f/g`?
5. Your script uses `cd some_directory` and it fails. The directory exists. What else could cause `cd` to fail?
6. Running `realpath -s` vs `realpath -m` vs `realpath -e` — when would each be appropriate?
7. You see a file in `ls` output but `cd` into it fails. What's likely wrong?

## Expected Output Summary

```
$ pwd
/home/phd
$ tree -L 1 /usr | head -8
/usr
├── bin
├── lib
├── local
├── sbin
├── share
└── src

$ realpath /usr/bin/python3
/usr/bin/python3.11

$ cd /var/log; pwd; cd ../tmp; pwd
/var/log
/var/tmp

$ cd -
/home/phd

$ ls -la ~ | grep '^\.' | wc -l
32

$ pushd /tmp; pushd /var; dirs -v; popd; popd
/tmp ~
/var /tmp ~
 0  /var
 1  /tmp
 2  ~
/tmp ~
~

$ ls -d /usr/*/ | wc -l
6
```
