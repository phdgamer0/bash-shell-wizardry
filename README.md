# 🐧 Shell Mastery: AWK · SED · GREP · BASH

> **From zero to shell wizard — a progressive, hands-on curriculum for mastering the four pillars of Unix text processing and scripting.**

---

## What Is This?

This is a **complete self‑study curriculum** — 85 lessons + 10 integrated quizzes — covering four essential Unix/Linux skill areas:

| Pillar | Focus | Lessons | Lines of Material |
|--------|-------|---------|------------------:|
| **grep** | Searching & filtering text | 7 | ~15.3k in target data |
| **sed**  | Stream editing & transformation | 7 | ~17.2k in target data |
| **awk**  | Text processing & reporting | 8 | ~37.4k in target data |
| **bash** | Shell scripting & system mastery | 53 | ~53k lines of lessons |

Every lesson follows the same proven pattern:

1. **Read** `lesson.md` — deep conceptual material with history, under‑the‑hood mechanics, 8–12 examples, trap vault, and memory aids.
2. **Do** `task.md` — a cumulative, multi‑step exercise that forces you to use what you just learned (and everything before it).
3. **Check** against `expected-result.txt` (grep/sed/awk) or self‑check questions (bash).

All data files are **real system logs** (journald, Apache HTTPD, package managers, transaction records) at least 2 000 lines each — no toy data.

---

## Project Structure

```
Learning/
├── README.md              ← you are here
├── LICENSE                ← MIT — do what you want
│
├── _data/                 ← raw data files used across lessons
│   ├── system.log         (35k lines — systemd journal)
│   ├── access.log         (10k lines — Apache HTTPD)
│   ├── warnings.log       (12k lines — high‑priority journal)
│   ├── transactions.csv   (3k lines — pipe‑delimited audit trail)
│   ├── sales.csv          (2.5k lines — generated sales records)
│   ├── pacman.log         (1.5k lines — package transactions)
│   ├── network.log        (789 lines — NetworkManager)
│   ├── auth.log
│   └── logind.log
│
├── GREP/                  ═══ 7 lessons ═══
│   ├── Lesson-01-Introduction/
│   ├── Lesson-02-Basic-Regex/
│   ├── Lesson-03-Extended-Regex/
│   ├── Lesson-04-Context-Flags/
│   ├── Lesson-05-Perl-Regex/
│   ├── Lesson-06-Recursive-Search/
│   └── Lesson-07-Real-World-Patterns/
│
├── SED/                   ═══ 7 lessons ═══
│   ├── Lesson-01-Introduction/
│   ├── Lesson-02-Addressing/
│   ├── Lesson-03-Delete-Print/
│   ├── Lesson-04-Insert-Append/
│   ├── Lesson-05-Transform/
│   ├── Lesson-06-Multiline/
│   └── Lesson-07-Hold-Space-and-Scripting/
│
├── AWK/                   ═══ 8 lessons ═══
│   ├── Lesson-01-Introduction/
│   ├── Lesson-02-Patterns/
│   ├── Lesson-03-Variables/
│   ├── Lesson-04-Control-Flow/
│   ├── Lesson-05-Arrays/
│   ├── Lesson-06-String-Functions/
│   ├── Lesson-07-Numeric-Format/
│   └── Lesson-08-Advanced/
│
├── THE-HOLY-THREE/        ═══ 10 quizzes ═══
│   ├── Quiz-01/  (easy)       Quiz-06/  (medium)
│   ├── Quiz-02/  (easy)       Quiz-07/  (hard)
│   ├── Quiz-03/  (easy)       Quiz-08/  (hard)
│   ├── Quiz-04/  (medium)     Quiz-09/  (hard)
│   └── Quiz-05/  (medium)     Quiz-10/  (very hard — capstone)
│
└── BASH/                  ═══ 53 lessons ═══
    ├── Beginner/         (lessons 01–16)
    ├── Intermediate/      (lessons 01–17)
    └── Advanced/          (lessons 01–20)
```

---

## How to Use This

### Recommended Order

```
 1. GREP (all 7)       — learn to search and filter
 2. SED  (all 7)       — learn to transform text
 3. AWK  (all 8)       — learn to process and report
 4. BASH/Beginner      — learn bash as a shell
 5. BASH/Intermediate   — learn bash as a language
 6. BASH/Advanced       — learn bash as a system tool
 7. THE-HOLY-THREE/     — integrate everything via 10 quizzes
```

Each tool section is **self‑contained** — if you already know grep you can skip straight to sed. Lessons within a tool **are sequential** — lesson 05 assumes you did 01–04.

### Inside a Lesson

```
Lesson-03-Extended-Regex/
├── task.md              ← what you need to do (cumulative multi‑step)
├── target.txt           ← the file you transform (2000–35 000 lines)
└── expected-result.txt  ← what the correct output looks like
```

**For bash lessons:**

```
lesson-07/
├── lesson.md            ← deep material (600–800+ lines each)
└── task.md              ← the hands‑on exercise
```

> ⚠️ **All files are `chmod 444` (read‑only).** You cannot accidentally overwrite them. Copy `target.txt` elsewhere if you need a writable copy.

---

## Lesson Roadmaps

### grep — "Find the Needle"

| # | Lesson | What You Learn |
|:-|--------|----------------|
| 01 | Introduction | Literal search, `-i`, `-n`, `-c`, `-v` |
| 02 | Basic Regex | `.`, `*`, `^`, `$`, `[]`, `[^]` |
| 03 | Extended Regex | `+`, `?`, `{n,m}`, `|`, `()`, `grep -E` |
| 04 | Context & Flags | `-A`, `-B`, `-C`, `-o`, `-l`, `-L`, `-r` |
| 05 | Perl Regex | `\d`, `\s`, `\b`, lookaheads, `grep -P` |
| 06 | Recursive Search | `--include`, `--exclude`, `-f`, `find \| grep` |
| 07 | Real‑World Patterns | Pipelines, extraction, counting, top‑N reports |

### sed — "The Stream Surgeon"

| # | Lesson | What You Learn |
|:-|--------|----------------|
| 01 | Introduction | `s///`, flags `g` `I` `p` `w`, `&` metachar |
| 02 | Addressing | Line numbers, ranges, `$`, `/regex/`, `!` |
| 03 | Delete & Print | `d`, `p`, `q`, `-n` quiet mode |
| 04 | Insert & Append | `i`, `a`, `c`, `r` read, `w` write |
| 05 | Transform | `y///`, transliteration, character classes |
| 06 | Multiline | `N`, `D`, `P` — joining multi‑line records |
| 07 | Hold Space & Scripting | `h` `H` `g` `G` `x`, branching `b` `t` |

### awk — "The Report Generator"

| # | Lesson | What You Learn |
|:-|--------|----------------|
| 01 | Introduction | `print`, `$0`–`$NF`, field splitting, `FS` |
| 02 | Patterns | `BEGIN`/`END`, `/regex/`, relational patterns |
| 03 | Variables | `OFS`, `ORS`, `RS`, `NR`, `NF`, `FILENAME` |
| 04 | Control Flow | `if`/`else`, `while`, `for`, `printf` formatting |
| 05 | Arrays | Associative arrays, `for (i in arr)`, `delete` |
| 06 | String Functions | `length`, `substr`, `split`, `gsub`, `match` |
| 07 | Numeric & Format | `sprintf`, `int`, `rand`, `printf` reports |
| 08 | Advanced | `getline`, `system()`, user functions, multi‑file |

### THE‑HOLY‑THREE — "The Gauntlet"

10 quizzes of increasing difficulty combining grep + sed + awk in pipelines. Quiz 10 is a capstone requiring cross‑source log correlation, severity extraction, and comprehensive reporting.

### BASH — 53 lessons across 3 tiers

<details>
<summary><strong>Beginner (16 lessons)</strong> — Using Bash as a Shell</summary>

| # | Lesson |
|:-:|--------|
| 01 | Navigating the Filesystem |
| 02 | File Operations |
| 03 | Globbing Deep Dive |
| 04 | Reading Files |
| 05 | Counting, Sorting & Extracting |
| 06 | Redirection Deep Dive |
| 07 | Pipes Deep Dive |
| 08 | Variables & Quoting |
| 09 | Environment & Shell Config |
| 10 | Writing & Running Scripts |
| 11 | Conditionals |
| 12 | File Tests |
| 13 | Loops |
| 14 | Functions |
| 15 | Regex with `=~` |
| 16 | Capstone: One‑liner Combos |
</details>

<details>
<summary><strong>Intermediate (17 lessons)</strong> — Bash as a Programming Language</summary>

| # | Lesson |
|:-:|--------|
| 01 | Arithmetic |
| 02 | Parameter Expansion — Defaults |
| 03 | Parameter Expansion — Slicing |
| 04 | Parameter Expansion — Replace & Case |
| 05 | Indexed Arrays |
| 06 | Associative Arrays |
| 07 | `find` Deep Dive |
| 08 | `xargs` & Parallel |
| 09 | String Manipulation |
| 10 | Here‑docs, Here‑strings & `mapfile` |
| 11 | `case` & `select` Menus |
| 12 | Process Management |
| 13 | Subshells & Grouping |
| 14 | Debugging — Tracing |
| 15 | Debugging — Strict Mode |
| 16 | Terminal Interaction |
| 17 | Date, Time & Random |
</details>

<details>
<summary><strong>Advanced (20 lessons)</strong> — Bash as a System Tool & Weapon</summary>

| # | Lesson |
|:-:|--------|
| 01 | File Descriptors & `exec` |
| 02 | Named Pipes & Co‑processes |
| 03 | Security & Injection |
| 04 | Dynamic Code & Reflection |
| 05 | `getopts` Basics |
| 06 | Advanced Argument Parsing |
| 07 | Performance & Benchmarking |
| 08 | Profiling & Optimization |
| 09 | Exploitation — Reverse Shells (Defense) |
| 10 | Exploitation — Privilege Escalation (Defense) |
| 11 | Exploitation — Persistence (Defense) |
| 12 | Exploitation — File System Attacks (Defense) |
| 13 | Exploitation — Defense & Detection |
| 14 | System Integration — systemd |
| 15 | Advanced System Integration (udev, inotify, dbus) |
| 16 | Networking with Bash |
| 17 | Large‑scale Data Processing |
| 18 | Capstone — CLI Tool |
| 19 | Capstone — Monitoring System |
| 20 | Capstone — Security Scanner |
</details>

---

## What Makes This Different

- **Real data, not toy examples.** You're working with actual systemd journals, Apache access logs, package manager transactions, and structured audit trails.
- **Cumulative tasks.** Each task requires everything you learned before it. No isolated drills.
- **Trap Vaults.** Every lesson documents 8–15 common mistakes with before/after examples — the kind of bugs that eat hours.
- **Under the hood.** Not just "what it does" but "how it works" — syscalls, kernel interactions, process model.
- **Defense‑focused exploitation.** Advanced lessons cover reverse shells, privesc, and persistence from a detection and prevention standpoint.
- **Read‑only files.** All target files are `chmod 444`. You can't accidentally corrupt your learning materials.
- **~272 000 words** of material across 192 files.

---

## Quick Start

```bash
# Start with grep lesson 01
cd GREP/Lesson-01-Introduction

# Read the task
cat task.md

# Work on target.txt using grep
grep -in "fail" target.txt      # Step 1
grep -ci "fail" target.txt      # Step 2

# Compare with expected result
cat expected-result.txt
```

For bash:
```bash
cd BASH/Beginner/lesson-01

# Read the deep lesson material
# (pipe through less — it's long)
cat lesson.md | less

# Do the task
cat task.md

# Write your solution, run it, verify
```

---

## Prerequisites

- **A Unix/Linux system** (macOS works too — most tools are available)
- **bash 4.0+** (check with `bash --version`; associative arrays need 4.0+)
- **Standard Unix tools:** `grep`, `sed`, `awk`, `find`, `xargs`, `sort`, `uniq`, `cut`, `tr`, `tee`, `date`
- **Optional:** `strace` for the advanced "Under the Hood" sections
- **No root required** for grep/sed/awk/Beginner/Intermediate lessons
- Some Advanced lessons (systemd, udev, `/dev/tcp`) benefit from root or a VM

---

## Tips for Success

1. **Type everything.** Don't copy-paste. Muscle memory matters.
2. **Break the expected result.** Try to make your output NOT match first, understand why, then fix it. Failure teaches more than success.
3. **Use `man`.** Every lesson teaches the most important flags, but `man grep` etc. is the real source of truth.
4. **Modify the traps.** When you see a trap, reproduce it, then fix it. The trap vaults are exercises pretending to be warnings.
5. **For bash — write scripts.** The terminal is fine for one-liners, but real mastery comes when you write, debug, and refactor actual `.sh` files.
6. **Sequence matters.** grep → sed → awk → bash beginner → bash intermediate → bash advanced → holy three. Each builds on the last.
7. **Read the history sections.** Understanding why `grep` is named "g/re/p" or why `pwd` exists makes the tool memorable, not just learnable.

---

## License

MIT License — see [LICENSE](LICENSE).

Do whatever you want with this material. Learn from it, share it, build on it, teach it. Attribution is appreciated but not required.

---

*Created with patience, caffeine, and a deep appreciation for the Unix philosophy.*
