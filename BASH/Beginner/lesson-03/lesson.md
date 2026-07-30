# Lesson 3: Globbing Deep Dive

## History & Origins

Globbing — pathname expansion — is older than most Unix commands. The word "glob" comes from "global" — specifically from the `/etc/glob` program in Version 6 Unix (1975). In early Unix, the shell itself didn't expand wildcards. Instead, you'd write something like `echo glob *` and the standalone `/etc/glob` program would expand the `*` into matching filenames. This was so universally useful that the feature was absorbed into the shell directly by Version 7 Unix (1979) with the Bourne shell.

The name "glob" stuck. The program `/etc/glob` implemented "global" pattern matching. The `glob()` function in C (from POSIX) is named after it. In the kernel, there's no globbing — it's purely a shell-level expansion that happens before a command runs.

The `*` wildcard came from the concept of a "wildcard" in card games, where a joker can represent any card. The `?` came from the same source — a single-character wildcard. The `[abc]` character class was inspired by regular expressions (which predate Unix globbing; regex was invented by Stephen Cole Kleene in the 1950s).

**Brace expansion** (`{a,b,c}`) was NOT part of traditional globbing. It was introduced by the C shell (`csh`) in the late 1970s as a "generation" feature that creates multiple strings from a pattern. Bash adopted it from `csh`. It happens *before* glob expansion, not as part of it. This is a critical distinction: brace expansion generates strings; glob expansion matches files.

**Extended globbing** (`shopt -s extglob`) was added to bash in the early 1990s, inspired by the Korn Shell (`ksh`). The patterns `@(...)`, `*(...)`, `+(...)`, `?(...)`, `!(...)` give globbing regex-like power.

**`globstar`** (`**`) was added in bash 4.0 (2009) — one of the most requested features. Before that, recursive file matching required `find` or external tools.

**Fun anecdote:** The reason `*` doesn't match hidden files (files starting with `.`) is an accident of history. The original `ls` simply skipped entries starting with `.` in its output. When globbing was added to the shell, it matched whatever `ls` showed — so dotfiles were invisible. When Dennis Ritchie was asked about this, he reportedly said it was a bug that became a feature. By the time anyone noticed, too many scripts relied on `*` NOT matching `.profile` and `.login`.

## Syntax Reference

### Basic glob characters

```bash
*                    # Any string, including empty (but not dotfiles)
?                    # Any single character
[abc]                # One character: a, b, or c
[!abc]               # One character: NOT a, b, or c
[^abc]               # Same as [!abc] (POSIX allows both)
[a-z]                # Range: any character from a to z
[0-9]                # Range: any digit
[a-zA-Z]             # Multiple ranges combined
[a-zA-Z0-9_]         # Alphanumeric plus underscore
[!a-z]               # NOT in range a-z
```

### Range nuances

The behavior of ranges depends on your locale. In `C` locale, `[a-z]` matches exactly `abc...xyz`. In `en_US.UTF-8`, it may match `aAbBcC...xXyYzZ` or treat `A` and `a` as the same character for collation:

```bash
$ LANG=C echo [a-z]*    # Only lowercase
$ LANG=en_US.UTF-8 echo [a-z]*  # May include uppercase too
```

Use `LC_ALL=C` for consistent behavior.

### Brace expansion (NOT a glob)

```bash
{a,b,c}              # Generates: a b c
{1..5}               # Generates: 1 2 3 4 5
{01..05}             # Generates: 01 02 03 04 05 (zero-padded)
{1..10..2}           # Generates: 1 3 5 7 9 (step of 2)
{a..z}               # Generates: a b c ... z
{A..Z}               # Generates: A B C ... Z
{1..5}{a,b}          # Cartesian product: 1a 1b 2a 2b 3a 3b 4a 4b 5a 5b
```

Brace expansion happens FIRST, before any other expansion. This means `{a,b}*` first becomes `a* b*`, then glob expands.

### Extended globbing (requires `shopt -s extglob`)

```bash
?(pattern)           # Zero or one match of pattern (like regex ?)
*(pattern)           # Zero or more matches of pattern (like regex *)
+(pattern)           # One or more matches of pattern (like regex +)
@(pattern)           # Exactly one match of pattern
!(pattern)           # Anything EXCEPT pattern
```

Patterns can be combined with `|`:

```bash
*.@(txt|csv|tsv)     # Matches .txt OR .csv OR .tsv
!(a|b|c)*            # Doesn't start with a, b, or c
```

### Shell options that affect globbing

```bash
shopt -s globstar    # Enable ** for recursive matching
shopt -s nullglob    # Non-matching globs disappear (become empty string)
shopt -s dotglob     # * matches hidden files (starting with .)
shopt -s failglob    # Non-matching globs cause an error
shopt -s nocaseglob  # Case-insensitive glob matching
shopt -s extglob     # Enable extended patterns
shopt -u extglob     # Disable
shopt -u globstar    # Disable
```

You can also use `set -o noglob` (or `set -f`) to **disable all globbing**:

```bash
set -o noglob
echo *                # Prints literal *
set +o noglob
echo *                # Lists files
```

### The `GLOBIGNORE` variable

```bash
export GLOBIGNORE="*.bak:*.tmp"   # * will not match .bak or .tmp files
```

This is like a blacklist for glob expansion. It also implicitly enables `dotglob` behavior.

## Under the Hood

### The globbing process step by step

When bash encounters a command like `ls *.txt /etc/*.conf`:

1. **Tokenization**: The shell breaks the command line into words separated by metacharacters.
2. **Brace expansion** (if any): `{a,b}.txt` -> `a.txt b.txt`
3. **Tilde expansion**: `~/docs` -> `/home/phd/docs`
4. **Parameter expansion**: `$VAR`, `${VAR}`, `$(cmd)`
5. **Command substitution**: `$(echo *.txt)` — note: nested!
6. **Word splitting**: Splits the results of expansions on `$IFS`.
7. **Pathname expansion (globbing)**: This is where `*`, `?`, `[...]` are matched against filenames.
8. **Quote removal**: Strips all quotes (they've done their job).

### What happens during pathname expansion

1. The shell scans each word for unquoted glob characters (`*`, `?`, `[`).
2. If found, it calls the internal `glob()` function (equivalent to the POSIX `glob()` C function).
3. The `glob()` function:
   - Opens the parent directory using `opendir()`.
   - Reads entries with `readdir()` (one `struct dirent` at a time).
   - Checks each filename against the pattern using `fnmatch()`.
   - Collects matches into an array.
   - Sorts them alphabetically.
   - Returns the array.
4. If no matches found and `nullglob` is off, the original pattern (with `*`) is passed literally.
5. If `failglob` is on and no matches found, bash prints an error and doesn't execute the command.

### System calls involved

```bash
$ strace -e trace=openat,getdents64 bash -c 'ls *.txt' 2>&1 | head -20
```

You'll see:
```
openat(AT_FDCWD, ".", O_RDONLY|O_NONBLOCK|O_CLOEXEC|O_DIRECTORY) = 3
getdents64(3, /* entries */, 32768) = 256
getdents64(3, /* entries */, 32768) = 0
```

The shell opens the current directory, reads all entries, filters by pattern, sorts, then passes the results to `ls`.

### Memory and process model implications

- Globbing can produce an **arbitrarily large number of arguments**. The total size of the expanded arguments plus environment is limited by `ARG_MAX` (usually 2MB on Linux). Run `getconf ARG_MAX` to see your limit.
- If a glob produces too many matches, you'll get: `bash: /bin/ls: Argument list too long`
- The shell must hold all matching paths in memory before running the command.
- For `**` (globstar), the shell recursively walks the entire directory tree, which can be very slow and memory-intensive on large trees.
- `find` does NOT have the ARG_MAX limit because it processes files one at a time.

### The `globstar` performance caveat

```bash
shopt -s globstar
ls /usr/**/*.conf         # Bash walks the ENTIRE /usr tree
find /usr -name '*.conf'  # find walks the tree — similar but not same
```

`find` is typically faster and has no ARG_MAX issue for the `-exec` action (which processes in batches). For large trees, `find` is almost always better than `**`.

## Core Examples

### Example 1: Basic wildcard matching

```bash
$ ls /etc/*.conf
/etc/adduser.conf /etc/debconf.conf /etc/host.conf /etc/resolv.conf /etc/sysctl.conf
```

**Step by step:**
1. The shell scans `*.conf` and finds `*` (unquoted).
2. It calls `glob("/etc/*.conf")`.
3. `glob` opens `/etc/`, reads all 200+ entries.
4. For each entry, it calls `fnmatch("*.conf", entry_name, 0)`.
5. Only entries ending in `.conf` match.
6. Results are sorted alphabetically.
7. The sorted list replaces `*.conf` in the command line.
8. `ls` is executed with the expanded list.

**What if:**
```bash
$ echo /etc/*.conf       # See what matches without running another command
$ ls /etc/*.conf *.none  # The *.none is passed literally (no match)
ls: cannot access '*.none': No such file or directory
```

### Example 2: Single-character matching with ?

```bash
$ touch /tmp/{a,b,c,ab,abc}.txt
$ ls /tmp/?.txt
/tmp/a.txt /tmp/b.txt /tmp/c.txt
```

**Step by step:**
1. `?` matches exactly one character before `.txt`.
2. `ab.txt` has two characters before the dot — no match.
3. `abc.txt` has three — no match.

**What if:**
```bash
$ ls /tmp/??.txt      # Exactly two chars before .txt
/tmp/ab.txt
$ ls /tmp/???.txt     # Three chars
/tmp/abc.txt
```

### Example 3: Character classes

```bash
$ ls /usr/bin/[ab]* | head -5
/usr/bin/arch
/usr/bin/awk
/usr/bin/b2sum
/usr/bin/basename
/usr/bin/bash
```

**Step by step:**
1. `[ab]` matches a single character: either `a` or `b`.
2. `*` matches any remaining characters.
3. Together, `[ab]*` matches anything starting with `a` or `b`.
4. The `head -5` is then applied to the expanded list.

**What if:**
```bash
$ ls /usr/bin/[!ab]* | head -3   # Everything NOT starting with a or b
/usr/bin/cat
/usr/bin/chmod
/usr/bin/cp
$ ls /usr/bin/[0-9]* | head -3   # Starting with a digit
/usr/bin/2to3
/usr/bin/7z
/usr/bin/7za
```

### Example 4: Brace expansion for directory scaffolding

```bash
$ mkdir -p project/{src,test,docs}/{v1,v2}
$ tree project
project
├── docs
│   ├── v1
│   └── v2
├── src
│   ├── v1
│   └── v2
└── test
    ├── v1
    └── v2
```

**Step by step:**
1. Brace expansion runs FIRST: `{src,test,docs}/{v1,v2}` becomes:
   `src/v1 src/v2 test/v1 test/v2 docs/v1 docs/v2`
2. Then `mkdir -p` receives these 6 paths and creates each.
3. `-p` creates parent `project/` directory as well.

**What if:**
```bash
$ echo {1..10}              # Sequence: 1 2 3 4 5 6 7 8 9 10
$ echo {a..f}               # Letters: a b c d e f
$ echo {1..10..3}           # Step: 1 4 7 10
$ echo {01..10}             # Zero-padded: 01 02 ... 10
$ touch file-{1..100}.txt   # Creates 100 files!
```

### Example 5: Globstar recursive matching

```bash
$ shopt -s globstar
$ ls /etc/**/*.conf | wc -l
293
$ find /etc -name '*.conf' | wc -l
293
```

**Step by step:**
1. `**` matches zero or more directories recursively.
2. `/etc/**/*.conf` means "any `.conf` file anywhere under `/etc/`".
3. Bash walks the entire `/etc` tree using a recursive directory walk.
4. Compare with `find`, which also walks the tree.

**What if:**
```bash
$ ls **                # All files recursively from current directory
$ ls **/*.txt          # All .txt files recursively
$ ls subdir/**/*.py    # All .py files under subdir recursively
```

**Performance note:** On large trees, `**` can be very slow. Prefer `find` when performance matters.

### Example 6: Extended glob patterns

```bash
$ shopt -s extglob
$ ls /usr/bin/@(ls|cp|mv)
/usr/bin/cp  /usr/bin/ls  /usr/bin/mv
$ ls /usr/bin/!(*.d) | head -5
/usr/bin/[    # Everything that doesn't end in .d
/usr/bin/2to3
/usr/bin/7z
/usr/bin/ar
/usr/bin/as
```

**Step by step:**
1. `@(ls|cp|mv)` matches exactly one of: `ls`, `cp`, or `mv`.
2. `!(*.d)` matches anything that does NOT match `*.d`.
3. The `!()` pattern is a NEGATION — it matches everything except the pattern inside.
4. Extended patterns are recognized ONLY when `extglob` is enabled.

**What if:**
```bash
$ ls !(a*|b*)              # Everything NOT starting with a or b
$ ls +(0|[1-9][0-9]*)     # All positive integers
$ ls ?(README|readme).md  # README.md, readme.md, or .md (if empty)
```

### Example 7: nullglob and failglob

```bash
$ shopt -u nullglob
$ echo *.nonexistent
*.nonexistent               # Passed literally!

$ shopt -s nullglob
$ echo *.nonexistent
                            # Disappeared — empty output

$ shopt -s failglob
$ echo *.nonexistent
bash: no match: *.nonexistent
```

**Step by step:**
1. With `nullglob` off (default): no match -> pattern stays as-is -> `echo` prints `*.nonexistent`.
2. With `nullglob` on: no match -> pattern removed -> `echo` gets no arguments -> prints blank line.
3. With `failglob` on: no match -> bash prints an error and doesn't run the command.

**Why this matters:** In scripts, `for f in *.nonexistent` with `nullglob` off iterates over ONE item (the literal string `*.nonexistent`). With `nullglob` on, the loop body never executes.

**What if:**
```bash
$ shopt -s nullglob
$ files=(*.txt)      # Array is empty if no .txt files
$ echo ${#files[@]}  # Prints 0
```

### Example 8: dotglob and hidden files

```bash
$ ls -a
.  ..  .hidden  file.txt

$ echo *
file.txt                               # .hidden NOT matched

$ shopt -s dotglob
$ echo *
.hidden file.txt                       # .hidden IS matched

$ echo .*                              # Always matches hidden files
. .. .hidden                           # But includes . and ..!
```

**Step by step:**
1. Default: `*` skips entries starting with `.`.
2. `dotglob` makes `*` match ALL entries (including dotfiles).
3. `.*` explicitly matches dotfiles — but also matches `.` and `..`.

**What if:**
```bash
$ echo .[!.]* .??*   # Smarter: match dotfiles, exclude . and ..
.hidden
```

### Example 9: Word splitting after globbing

```bash
$ touch "my file.txt" "myfile.txt"
$ for f in my*.txt; do echo "File: $f"; done
File: my file.txt
File: myfile.txt
```

**Step by step:**
1. The glob `my*.txt` matches both files (including the one with a space).
2. The shell stores the expanded list as separate items.
3. The `for` loop iterates correctly because the glob handles spaces — each match is a distinct word.

**What if:** (the wrong way)
```bash
$ FILES=$(ls my*.txt)   # DON'T DO THIS
$ for f in $FILES; do echo "File: $f"; done
File: my
File: file.txt
File: myfile.txt
```
Using `ls` output breaks when filenames contain spaces. Always let the glob expand naturally.

### Example 10: GLOBIGNORE in action

```bash
$ touch a.txt b.txt c.txt a.bak b.bak
$ export GLOBIGNORE="*.bak"
$ echo *
a.txt b.txt c.txt
$ echo *.bak
# Nothing — even explicit matches are suppressed!
```

**Step by step:**
1. `GLOBIGNORE="*.bak"` tells the shell to exclude `*.bak` from any glob expansion.
2. It also implicitly enables `dotglob`.

**What if:**
```bash
$ unset GLOBIGNORE    # Back to normal
$ echo *
a.bak a.txt b.bak b.txt c.txt
```

### Example 11: Combining glob features to select files

```bash
$ touch /tmp/{data-{2023,2024}-{01..03},report-final}.csv
$ ls /tmp/*.csv | wc -l
7
$ ls /tmp/data-2024-*.csv
/tmp/data-2024-01.csv  /tmp/data-2024-02.csv  /tmp/data-2024-03.csv
$ ls /tmp/data-202[34]-0[12].csv
/tmp/data-2023-01.csv  /tmp/data-2023-02.csv  /tmp/data-2024-01.csv  /tmp/data-2024-02.csv
```

**Step by step:**
1. Brace expansion first creates all combinations.
2. Globs then filter further.
3. `data-202[34]-0[12].csv` matches a specific subset.
4. The shell sorts all results alphabetically.

### Example 12: The ARG_MAX trap

```bash
$ echo /usr/bin/* | wc -w     # Expands to all 2500+ filenames — works
$ echo /usr/bin/* /bin/* /sbin/* /usr/sbin/*   # Might hit ARG_MAX!
bash: /usr/bin/echo: Argument list too long
```

**Step by step:**
1. The shell expands all four globs into a massive argument list.
2. Total length exceeds `ARG_MAX` (usually 2MB of environment+args).
3. The kernel refuses to `execve()` the command and returns `E2BIG`.
4. The shell prints "Argument list too long."

**What if:**
```bash
$ printf '%s\n' /usr/bin/* /bin/* | wc -l  # Same problem — printf also exceeds ARG_MAX
$ find /usr/bin /bin -maxdepth 1 | wc -l    # find doesn't expand — works fine
```

## Real-World Use Cases

### 1. FOR the OS — Administration

- **`rm -rf /var/log/*.old`** — clean old log files.
- **`chmod 600 /etc/ssh/ssh_host_*`** — set permissions on SSH host keys.
- **`cat /proc/sys/net/ipv4/conf/*/rp_filter`** — read reverse path filter settings from all interfaces.
- **`cp /etc/*.conf /backup/configs/`** — backup configuration files.
- **`for f in /var/log/syslog*; do gzip "$f"; done`** — compress rotated logs.

### 2. WITH the OS — Development

- **`rm -rf node_modules/`** — clean up projects (brace expansion: `rm -rf {node_modules,.next,build}`).
- **`tar czf project-$(date +%Y%m%d).tar.gz project/{src,tests,docs}`** — archive specific directories.
- **`diff src/main.old src/main.new`** — file pairs with globs.
- **`for img in images/*.{jpg,jpeg,png}; do convert "$img" -resize 50% "${img%.*}-thumb.${img##*.}"; done`** — batch resize images.
- **`shopt -s globstar && wc -l src/**/*.{js,ts}`** — count lines of code across a project.

### 3. AGAINST the OS — Exploitation

- **Glob injection**: If a script evaluates user input as part of a glob, the user can inject `*` to cause unexpected behavior. E.g., `rm $USER_INPUT` where input is `*` -> deletes everything.
- **Hidden file evasion**: Attackers name files like `...txt` (triple dot — not hidden) or `" "` (space) to avoid simple globs.
- **`[a-z]*` brute force**: Iterating over ranges to discover files.
- **`echo /*`** reconnaissance: Attacker lists root directory. `echo /home/*/.*` lists all users' hidden files.

### 4. FOR DEFENSE — Detection & Auditing

- **Audit glob expansion**: `echo /etc/**/*.conf 2>/dev/null` to catalog all config files.
- **Count files per directory**: `for d in /*/; do echo "$d: $(ls "$d" | wc -l)"; done` — detect unexpected file counts.
- **`shopt -s dotglob; shopt -s nullglob`** in security scripts to ensure all files are processed.
- **Shellshock detection**: `env 'SHELLSHOCK=() { :;}; echo vulnerable' bash -c "echo test"` uses glob-like shell functions CVE.
- **Log analysis**: `grep -h 'Failed password' /var/log/auth.log* | wc -l` counts SSH bruteforce attempts.

## Memory Aids

### Mnemonics

- **`*` = "everything"** — the asterisk looks like a star, meaning "all the things."
- **`?` = "one thing"** — a question mark looks like it's asking "what character is here?"
- **`[abc]` = "choose one"** — like brackets around a set of options.
- **`[!abc]` = "NOT these"** — the `!` means "not" in many contexts.
- **`{a,b,c}` = "generate"** — curly braces like `{` and `}` enclose a list to be expanded.

### Pattern hooks

- Want all `.txt` files? → `*.txt`
- Want exactly one char? → `?.txt`
- Want files starting with `a` or `b`? → `[ab]*`
- Want everything except `.txt`? → `extglob` + `!(*.txt)`
- Want recursive? → `shopt -s globstar` + `**/*.py`
- Want to create dirs? → `mkdir -p {a,b,c}/{1,2,3}`

### Why it's named that

- **"glob"** comes from `/etc/glob` — the "global" command that replaced `*` with matching filenames.
- **"globbing"** is the verb form. You're "globbing" when you use wildcards.
- The C function `glob()` is named after the program.

### Common confusions

- **Globs are NOT regex**: `*` in glob = "any string" (regex: `.*`). `?` in glob = "any char" (regex: `.`). `[abc]` works the same. Avoid mixing them up when writing `find -name` (which uses globs) vs `grep` (which uses regex).
- **Brace expansion is not globbing**: `{a,b}*` becomes `a* b*` (two globs), not "a or b followed by anything."
- **`*` doesn't match dotfiles**: This is the #1 cause of "I can't see my .env file" confusion.
- **`**` is not in POSIX**: It's a bash extension. In `sh` mode, `**` is just two `*` patterns. Only works with `globstar`.
- **`*` at the start of a path**: `/*` matches everything in root. `*` matches everything in current dir. These are different!

## Trap Vault

### Trap 1: Globs inside quotes

**Problem:** `echo "*"` prints a literal `*`, not file names.

**Example:**
```bash
$ echo "*"
*
$ echo *
file1.txt file2.txt script.sh
```

**Why:** Globbing is disabled inside single quotes AND double quotes. The shell doesn't expand patterns within quotes.

**Fix:** Keep patterns outside quotes:
```bash
$ echo "Files: " *
Files: file1.txt file2.txt script.sh
$ ls -l "${dir}/"*    # Variable quoted, glob unquoted
```

### Trap 2: Nullglob off causes literal pattern to pass through

**Problem:** Loop over `*.nonexistent` runs once with the literal pattern.

**Example:**
```bash
$ for f in *.nonexistent; do echo "Processing $f"; done
Processing *.nonexistent
```

**Why:** With `nullglob` off (default), a non-matching glob keeps its literal `*` character. The loop iterates once with `f="*.nonexistent"`.

**Fix:**
```bash
$ shopt -s nullglob
$ for f in *.nonexistent; do echo "Processing $f"; done
# No output — loop never executes
$ shopt -s failglob
$ for f in *.nonexistent; do echo "Processing $f"; done
bash: no match: *.nonexistent
```

### Trap 3: `.*` matches `.` and `..`

**Problem:** `rm -rf .*` tries to remove parent directories.

**Example:**
```bash
$ rm -rf .*
rm: cannot remove '.': Permission denied
rm: cannot remove '..': Permission denied
# But other dotfiles were successfully removed!
```

**Why:** `.*` expands to `.`, `..`, `.bashrc`, `.config`, etc. `rm` refuses to remove `.` and `..`, but happily removes everything else. You've lost your config files.

**Fix:**
```bash
$ rm -rf .[!.]* ..?*   # Match dotfiles but not . or ..
$ rm -rf ./*           # Safer: removes everything including hidden, but NOT . or ..
```

### Trap 4: Locale affects range matching

**Problem:** `[a-z]` may match uppercase letters in some locales.

**Example:**
```bash
$ LANG=en_US.UTF-8
$ echo [a-z]*
Apple Banana cherry    # Note: Apple and Banana start with uppercase!
$ LANG=C
$ echo [a-z]*
cherry                 # Only lowercase
```

**Why:** In many non-C locales, collation order is case-insensitive or dictionary-based. `[a-z]` matches everything that sorts between `a` and `z`, which includes uppercase variants.

**Fix:**
```bash
$ LC_COLLATE=C echo [a-z]*    # Override only collation
$ LC_ALL=C echo [a-z]*        # Override everything
```

### Trap 5: ARG_MAX and glob explosion

**Problem:** A glob that matches millions of files makes the command too long to execute.

**Example:**
```bash
$ echo /usr/bin/* /bin/* /sbin/* /usr/sbin/* /usr/local/bin/*
bash: /usr/bin/echo: Argument list too long
```

**Why:** The shell expands all globs, concatenates the results, and passes them to `execve()`. The total size of args + environment exceeds `ARG_MAX` (typically 2MB).

**Fix:**
```bash
$ find /usr/bin /bin /sbin /usr/sbin /usr/local/bin -maxdepth 1 | wc -l
$ printf '%s\0' /usr/bin/* | xargs -0 echo    # xargs batches them
```

### Trap 6: `**` with globstar on huge trees

**Problem:** `**/*.log` on `/var/log` walks EVERY directory under `/var/log/`.

**Example:**
```bash
$ shopt -s globstar
$ ls /var/log/**/*.log | wc -l
# Takes forever! Walking thousands of directories...
```

**Why:** `**` recursively descends into every subdirectory. On large filesystems, this can take minutes and use significant memory.

**Fix:**
```bash
$ find /var/log -name '*.log'    # Faster, uses C, not bash
$ shopt -u globstar               # Disable when not needed
```

### Trap 7: Globbing after variable expansion

**Problem:** `$var` where `var="*.txt"` still undergoes globbing.

**Example:**
```bash
$ pattern="*.txt"
$ ls $pattern          # Globbing happens AFTER variable expansion!
file1.txt file2.txt
$ ls "$pattern"        # Quoted: literal *.txt
ls: cannot access '*.txt': No such file or directory
```

**Why:** The order of expansions: parameter expansion happens FIRST, then word splitting, then pathname expansion. So `$pattern` becomes `*.txt`, which then gets glob-expanded.

**Fix:** Quote to suppress globbing: `ls "$pattern"`. If you want globbing, leave unquoted.

### Trap 8: Brace expansion with spaces

**Problem:** `{a, b, c}` (with spaces after commas) breaks.

**Example:**
```bash
$ echo {a, b, c}
{a, b, c}    # NOT brace expansion!
$ echo {a,b,c}
a b c
```

**Why:** Brace expansion requires NO spaces after commas (or around the braces). With spaces, the shell interprets `{a,` as a literal string.

**Fix:**
```bash
$ echo {a,b,c}
$ echo {a, b, c}    # Won't work
```

### Trap 9: Globbing and `find -name` use different pattern syntax

**Problem:** Your `find -name` pattern doesn't work because you used regex syntax.

**Example:**
```bash
$ find . -name "*.txt"    # Correct — this is a glob
$ find . -name ".*\.txt"  # WRONG — that's regex, not glob
$ find . -regex ".*\.txt" # Correct — this IS regex
```

**Why:** `find -name` uses glob patterns, not regular expressions. `*` in glob = "anything", `.*` in glob = "literal dot followed by anything." To use regex, use `-regex` (GNU find).

**Fix:** Remember: glob for filenames, regex for text content.

### Trap 10: Setting `GLOBIGNORE` without unsetting it

**Problem:** `GLOBIGNORE` persists for the entire shell session.

**Example:**
```bash
$ export GLOBIGNORE="*.bak"
$ ls *
# No .bak files shown — good
$ ls *.bak
# Nothing! Even explicit globs are filtered!
```

**Why:** `GLOBIGNORE` affects ALL glob expansion, even explicit `*.bak` patterns. It's a blacklist for glob expansion, not just a default filter.

**Fix:** Unset it when done:
```bash
$ unset GLOBIGNORE
```

## See It In The Wild

### Where you encounter globbing daily

- **`.gitignore`**: Uses glob patterns to exclude files from version control.
- **`.dockerignore`**: Same concept — excludes files from Docker build context.
- **`apt-get install *`**: Just kidding, but package manager file lists use globs.
- **`npm install --save`**: Looks at your `package.json` which doesn't use globs, but `npm pack` does.
- **`Makefile`**: Target patterns like `%.o: %.c` are glob-like.

### How to observe globbing with xtrace

```bash
$ set -x              # Enable debug mode
$ echo *.txt
+ echo file1.txt file2.txt
$ set +x              # Disable debug mode
```

With `set -x`, you see the exact command after expansion — invaluable for debugging.

### Try this now

```bash
# Watch glob expansion in slow motion
set -x
echo /etc/host*
set +x

# Find files with unusual names
touch /tmp/"weird file (v1.2).txt"
ls -la /tmp/*.txt

# Count everything in /etc with different glob techniques
echo /etc/*.conf /etc/*.d | wc -w
ls /etc/*.conf /etc/*.d | wc -l
```

## Check Your Understanding

<details>
<summary>1. Why does `echo *.txt` NOT show files when you run it inside double quotes like `echo "*.txt"`?</summary>

Double quotes suppress globbing. The shell interprets `"*.txt"` as a literal string, not a pattern. Globbing only happens on unquoted patterns.
</details>

<details>
<summary>2. What's the difference between `[!abc]` in glob vs `[^abc]` in regex?</summary>

In globs, `[!abc]` and `[^abc]` are identical — both mean "not a, b, or c." `!` is traditional in shells, `^` was added for consistency with regex. In regex, only `^` works inside brackets.
</details>

<details>
<summary>3. You have `shopt -s nullglob` set and run `for f in *.jpg; do ...`. If no jpg files exist, what happens?</summary>

The loop body never executes. `*.jpg` expands to nothing (nullglob removes the pattern), so `for f in ` has zero items to iterate over.
</details>

<details>
<summary>4. What's the difference between `[a-z]` in the C locale vs `en_US.UTF-8` locale?</summary>

In C locale, `[a-z]` matches exactly the 26 lowercase ASCII letters. In en_US.UTF-8, collation rules may cause `[a-z]` to include uppercase letters or be case-insensitive, depending on the locale's `LC_COLLATE` setting.
</details>

<details>
<summary>5. You're in a directory with files: `data.txt`, `data.csv`, `script.sh`. You run `echo [!.]?*` — what matches?</summary>

All three files match. `[!.]` matches any character that isn't `.` (none of them start with `.`), `?` matches the second character, `*` matches the rest.
</details>

<details>
<summary>6. When does brace expansion happen relative to glob expansion? Why does this matter?</summary>

Brace expansion happens FIRST, then glob expansion. This matters because `{a,b}*` becomes `a* b*` (two glob patterns) and then each pattern is expanded separately. If the order were reversed, `{a,b}*` would be treated as a single pattern that matches files literally named `{a,b}*`.
</details>

<details>
<summary>7. How can globs be used in a security exploit? Give one scenario.</summary>

Glob injection: If a script does `rm $USER_INPUT` and the user passes `*`, the command becomes `rm *` — deleting all files in the current directory. This is why you should always quote variables and sanitize input that will be used in file operations.
</details>
