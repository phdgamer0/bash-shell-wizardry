# Lesson 07: Hold Space and Scripting

## The Hold Space

sed has two memory buffers:
- **Pattern Space** — the current line being processed (auto-printed by default)
- **Hold Space** — a secondary, persistent buffer that survives across lines

## Hold Space Commands

| Command | Meaning |
|---------|---------|
| `h` | **Copy** pattern space → hold space (overwrite) |
| `H` | **Append** pattern space → hold space (add newline + content) |
| `g` | **Copy** hold space → pattern space (overwrite) |
| `G` | **Append** hold space → pattern space (add newline + hold content) |
| `x` | **Exchange** — swap pattern space and hold space |

## Branching and Labels

sed supports simple control flow with labels and branches:

```bash
sed ':mylabel
s/foo/bar/
t mylabel' file.txt
```

| Command | Meaning |
|---------|---------|
| `:label` | Define a branch label |
| `b label` | Branch (jump) to label unconditionally |
| `t label` | Branch to label if the **last** `s///` made a substitution on the current line |

## Common Patterns

### Reverse all lines in a file
```bash
sed '1!G;h;$!d' file.txt
```

How it works:
- Line 1: `1!G` skipped, `h` copies line 1 to hold, `$!d` deletes line 1 (don't print yet)
- Line 2: `G` appends hold (line 1), `h` copies (line2\nline1) to hold, `d` deletes
- ...accumulates lines in reverse order in hold space...
- Last line: `G` appends hold, `h` copies (no delete), pattern space is printed with all lines reversed

### Branching for multiple substitutions
```bash
sed ':loop; s/ERROR/FAILED/; t loop; s/WARN/CAUTION/; t loop' file.txt
```
This keeps replacing ERROR→FAILED until none are left, then replaces WARN→CAUTION until none are left.

## Examples

### Example 1: Swap first two lines
```bash
$ cat data.txt
first
second
third
$ sed '1{h;d};2{x};' data.txt
second
first
third
```

### Example 2: Build reverse order
```bash
$ cat data.txt
line 1
line 2
line 3
$ sed '1!G;h;$!d' data.txt
line 3
line 2
line 1
```

### Example 3: Branching for exhaustive replacement
```bash
$ echo "ERROR WARN ERROR WARN" | sed ':a; s/ERROR/FAILED/; t a; s/WARN/CAUTION/; t a'
FAILED CAUTION FAILED CAUTION
```

## Task

Using **`target.txt`** (structured section log, 2200 lines):

1. **Reverse all lines** in the file using the hold space (`1!G;h;$!d` pattern)
2. In the **same** sed command, use **branching with labels** (`:label` and `t`) to replace **all** occurrences of "ERROR" with "FAILED" and **all** occurrences of "WARN" with "CAUTION"

**Hint:** The substitutions should happen **before** the reversal in each line. Combine the branching loop and the hold-space reversal in a single sed command. The `t` command only checks the **last** substitution made — structure your loop to handle ERROR first, then WARN.

**Expected result:** All lines in reverse order, with every "ERROR" changed to "FAILED" and every "WARN" changed to "CAUTION". The file should be completely reversed (last line first, first line last).
