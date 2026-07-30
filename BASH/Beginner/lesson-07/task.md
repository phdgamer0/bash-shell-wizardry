# Task 7: Pipeline Chain Reaction

Build sophisticated pipelines of 5+ commands, handle errors mid-pipe, master PIPESTATUS, and understand subshell scoping with variables.

## Setup

```bash
$ mkdir -p /tmp/pipe_lab
$ cd /tmp/pipe_lab
```

## Sub-task 1: Word Frequency Analysis

Create a data file:

```bash
$ cat > /tmp/words.txt << 'EOF'
apple banana apple cherry banana banana date elderberry fig grape apple banana cherry apple
EOF
```

Now build a pipeline that:
1. Reads the file
2. Splits words onto separate lines (use `tr`)
3. Sorts alphabetically
4. Deduplicates and counts (`uniq -c`)
5. Sorts by frequency descending
6. Shows only the top 3

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort | uniq -c | sort -rn | head -3
      4 banana
      4 apple
      2 cherry
```

**Step by step:**
1. `cat` reads the entire line.
2. `tr ' ' '\n'` replaces spaces with newlines (one word per line).
3. `sort` puts identical words adjacent.
4. `uniq -c` counts each group, prefixing the count.
5. `sort -rn` sorts numerically descending.
6. `head -3` keeps the top 3.

**What if:**
```bash
$ tr ' ' '\n' < /tmp/words.txt | sort | uniq -c | sort -rn | head -3  # No cat — UUOC-free!
$ awk '{for(i=1;i<=NF;i++) print $i}' /tmp/words.txt | sort | uniq -c | sort -rn | head -3
```

**Question:** What's the difference between `tr ' ' '\n'` and `tr ' ' '\n'` with multiple spaces?

<details>
<summary>Multiple spaces with tr</summary>
`tr ' ' '\n'` replaces EACH space with a newline. If there are multiple consecutive spaces, you get empty lines. Use `tr -s ' ' '\n'` to squeeze spaces first, or `tr -s '[:space:]' '\n'` to squeeze all whitespace.
</details>

## Sub-task 2: Pipeline Exit Codes

Run the pipeline and check PIPESTATUS:

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort | uniq -c | sort -rn | head -3
      4 banana
      4 apple
      2 cherry
$ echo "${PIPESTATUS[@]}"
0 0 0 0 0 0
```

Now intentionally break one step:

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort -x | uniq -c | sort -rn | head -3
# sort -x is invalid
sort: invalid option -- 'x'
$ echo "${PIPESTATUS[@]}"
0 0 2 0 0 0
```

**Question:** Which command failed? How do you know? What exit code did it have?

<details>
<summary>Exit code analysis</summary>
`PIPESTATUS[2]` is 2 — that's the third command (`sort -x`). Exit code 2 is the standard "invalid option" exit for coreutils. The pipeline continued but `sort -x` produced no output, so `uniq -c` counted nothing, etc.
</details>

## Sub-task 3: Tee Inside a Pipeline

Modify the word frequency pipeline to save the sorted (pre-uniq) list to a file while continuing:

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort | tee /tmp/sorted_words.txt | uniq -c | sort -rn | head -3
      4 banana
      4 apple
      2 cherry

$ cat /tmp/sorted_words.txt
apple
apple
apple
apple
banana
banana
banana
banana
cherry
cherry
date
elderberry
fig
grape
```

**Step by step:**
1. Everything through `sort` produces sorted word list (one per line).
2. `tee` writes this list to `/tmp/sorted_words.txt` AND passes it through.
3. `uniq -c` and subsequent commands get the same data.

**Question:** Why would you want to save intermediate results? When is this useful?

<details>
<summary>Intermediate saves</summary>
Useful for: debugging (check intermediate output), reusing partial results without re-running, multi-step processing where you want to compare intermediate states, and checkpointing in long pipelines.
</details>

## Sub-task 4: Progress Bar with pv

If `pv` is installed, add a progress indicator:

```bash
$ pv /tmp/words.txt | tr ' ' '\n' | sort | uniq -c | sort -rn | head -3
91 B 0:00:00 [1.88MiB/s] [>                   ] 0%  # Or similar
      4 banana
      4 apple
      2 cherry
```

**Question:** If `pv` isn't installed, install it (`apt install pv` or `brew install pv`) or simulate it with `dd status=progress`.

## Sub-task 5: Variable Scope in Pipelines

Demonstrate the subshell variable scope problem:

```bash
$ count=0
$ cat /tmp/words.txt | tr ' ' '\n' | while read word; do
    count=$((count + 1))
    echo "Word $count: $word"
done
Word 1: apple
Word 2: apple
... (14 words)
Word 14: grape
$ echo "Total words: $count"
Total words: 0    # Lost!
```

Now fix it using process substitution or input redirection:

```bash
$ count=0
$ while read word; do
    count=$((count + 1))
done < <(cat /tmp/words.txt | tr ' ' '\n')
$ echo "Total words: $count"
Total words: 14    # Works!
```

**Question:** What's happening here? Why does `while read` in a pipeline lose the variable, but `while read < <(command)` doesn't?

<details>
<summary>Subshell explanation</summary>
In a pipeline (`cmd | while read`), each segment runs in a separate subshell (forked process). The `while` loop's subshell increments `count` but that subshell exits when done — the parent's `count` is unchanged. With `while read < <(cmd)`, the while loop runs in the CURRENT shell (no subshell), so variable modifications persist.
</details>

## Sub-task 6: Pipe Both Streams

Create a command that produces both stdout and stderr, then filter only errors:

```bash
$ find /root /tmp -name '*.txt' 2>&1 | grep -i "permission"
find: '/root': Permission denied

# Or with the bash-specific syntax:
$ find /root /tmp -name '*.txt' |& grep -i "permission"
find: '/root': Permission denied
```

Without `2>&1`:

```bash
$ find /root /tmp -name '*.txt' | grep -i "permission"
# Permission denied messages go to terminal (not piped), grep sees only actual results
```

**Question:** When would you want to pipe both streams vs piping only stdout?

<details>
<summary>Stream choice</summary>
Pipe both streams when you want to filter or process error messages (e.g., `|& grep -i error`). Pipe only stdout when you only care about normal output and want errors visible on the terminal separately.
</details>

## Sub-task 7: Named Pipe Communication

Create a two-process data flow with a named pipe:

```bash
$ mkfifo /tmp/data_pipe
$ ls /usr/bin > /tmp/data_pipe &    # Writer in background
[1] 12345
$ wc -l < /tmp/data_pipe            # Reader
2560
[1]+  Done

$ # Clean up
$ rm /tmp/data_pipe
```

**Question:** What happens if you run the writer WITHOUT the reader? What if you run the reader first?

<details>
<summary>FIFO blocking behavior</summary>
Opening a FIFO for writing BLOCKS until a reader opens the other end (and vice versa). This is the FIFO's built-in synchronization. If you write without a reader, the `open()` call blocks forever. With the reader in background, both open simultaneously and data flows.
</details>

## Sub-task 8: xargs in Action

Use `xargs` with the word list:

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort -u | xargs -I{} echo "Word: {}"
Word: apple
Word: banana
Word: cherry
Word: date
Word: elderberry
Word: fig
Word: grape
```

Now use `xargs` with parallel processing:

```bash
$ cat /tmp/words.txt | tr ' ' '\n' | sort -u | xargs -P 4 -I{} sh -c 'echo "Processing {}"; sleep 1'
# 4 words processed in parallel, completes in ~2 seconds instead of ~7
```

**Question:** What would happen if a filename or word contained spaces? How would you protect against it?

<details>
<summary>xargs and spaces</summary>
`xargs` by default splits on whitespace. A word with spaces would be split. Use `-0` (NUL delimiter) with `print0` or `tr '\n' '\0'` to null-delimit input, or use `xargs -d '\n'` to treat newlines as the only delimiter.
</details>

## Sub-task 9: Early Termination with head

Show how `head` stops upstream processes:

```bash
$ time (seq 1 10000000 | head -5)
1
2
3
4
5

real    0m0.005s    # Fast! seq didn't generate all 10M lines
```

Compare with a version that doesn't use pipes:

```bash
$ time (head -5 <(seq 1 10000000))
1
2
3
4
5

real    0m0.005s    # Similar — process substitution also uses pipes internally
```

**Question:** How much time would `seq 1 10000000` take without `head` limiting it?

<details>
<summary>seq performance</summary>
`seq 1 10000000` takes much longer (maybe 0.2-0.5s) and produces 76MB+ of output. The `| head -5` version takes ~5ms because `head` exits after 5 lines, `seq` gets SIGPIPE, and stops. This is a huge performance win.
</details>

## Sub-task 10: Multi-Stage Filtering

Build a pipeline that processes `/etc/passwd` through multiple stages:

```bash
$ cat /etc/passwd | \
    grep -v nologin | \
    grep -v /bin/false | \
    cut -d: -f1,7 | \
    tr ':' ' ' | \
    sort -k2,2 | \
    awk '{print $2, $1}' | \
    head -10
/bin/bash phd
/bin/bash root
/bin/sh sys
/bin/sync sync
```

**Step by step:**
1. `grep -v nologin` — remove users with nologin shell.
2. `grep -v /bin/false` — remove users with false shell.
3. `cut -d: -f1,7` — keep username and shell.
4. `tr ':' ' '` — change separator to space.
5. `sort -k2,2` — sort by shell path.
6. `awk '{print $2, $1}'` — swap columns (shell first, then user).
7. `head -10` — show first 10.

**Question:** How many processes ran in parallel here? How is data flowing between them?

<details>
<summary>Parallel processes</summary>
Each `|` creates a new process. That's 7 commands = 7 processes, all running concurrently. Data flows through the kernel pipe buffer — each process reads from its stdin (the previous pipe) and writes to its stdout (the next pipe). The slowest process determines the overall throughput.
</details>

## Bonus Challenge: The `sort -u` vs `sort | uniq` Race

Create a large file and compare the two approaches:

```bash
$ seq 1 1000000 > /tmp/numbers.txt
$ for i in $(seq 1 1000); do cat /tmp/numbers.txt; done > /tmp/bignumbers.txt
$ wc -l /tmp/bignumbers.txt
1000000000   # 1 billion lines

# Compare performance:
$ time sort -u /tmp/bignumbers.txt > /dev/null
# or:
$ time sort /tmp/bignumbers.txt | uniq > /dev/null
```

**Question:** Which is faster? Why? (Hint: `sort -u` deduplicates during the sort phase, avoiding a second pass.)

## Self-Check

1. Why do variables set in pipelines sometimes appear empty afterward?
2. What does `${PIPESTATUS[0]}` represent in a 4-command pipeline?
3. What happens if the middle command in a pipeline fails? Does the whole pipeline return non-zero?
4. How is `|&` different from `|`?
5. What causes the "Broken pipe" error? Is it always a real error?
6. How would you pipe both stdout and stderr to separate commands simultaneously?
7. Why does `yes | head -5` terminate quickly, but `yes > file` runs forever until disk is full?
