# Task 13: Batch File Operations with Loops

Use all three loop styles — `for in`, `while read`, and C-style `for` — to process files, rename batches, and handle text. By the end, you'll be comfortable with every loop variant bash offers.

## Steps

### Sub-task 1: Create Test Data
Set up a working directory with various file types:

```bash
mkdir -p /tmp/loop_lab
cd /tmp/loop_lab

# Create text files
for i in {1..10}; do touch "file_${i}.txt"; done
for i in {1..5}; do touch "data_${i}.csv"; done
for i in {1..3}; do mkdir -p "dir_$i"; done
for i in {1..3}; do touch "script_$i.sh"; done

# Create a fruits list
cat > fruits.txt << 'EOF'
apple
banana
cherry
date
elderberry
fig
grape
EOF

# Create a CSV-like file
cat > people.csv << 'EOF'
name,age,email
Alice,30,alice@example.com
Bob,25,bob@example.com
Charlie,35,charlie@example.com
EOF
```

**Approach 1:** Manual creation with brace expansion
**Approach 2:** Scripted with `for` loops
**Approach 3:** Use `seq` and `xargs` for creation

Run `ls -la /tmp/loop_lab` to verify everything is in place.

<details><summary>Hint: Brace expansion ranges</summary>
`{1..10}` expands to `1 2 3 4 5 6 7 8 9 10`. `{a..z}` gives all lowercase letters. You can nest: `{file_{1..5},data_{1..3}}.txt`.
</details>

### Sub-task 2: Batch Rename with `for in`
Rename all `.txt` files to `.bak` using parameter expansion:

```bash
cd /tmp/loop_lab
echo "Before: $(ls *.txt 2>/dev/null | wc -l) .txt files"

for f in *.txt; do
    if [ -f "$f" ]; then
        mv "$f" "${f%.txt}.bak"
        echo "Renamed: $f -> ${f%.txt}.bak"
    fi
done

echo "After: $(ls *.bak 2>/dev/null | wc -l) .bak files"
```

**Approach 1:** `${f%.txt}` removes shortest suffix match
**Approach 2:** `${f/.txt/.bak}` replaces first occurrence
**Approach 3:** `rename 's/\.txt$/.bak/' *.txt` (if `rename` command is available)

**Bonus challenge:** Rename multiple extensions at once:
```bash
for f in *.csv *.data; do
    mv "$f" "${f}.old"
done
```

<details><summary>Hint: ${f%.txt} explained</summary>
`${f%.txt}` removes the SHORTEST suffix matching `.txt` from the end of `$f`. If `f="file.txt"`, result is `"file"`. If `f="file.txt.txt"`, result is `"file.txt"` (shortest match). Use `%%` for longest match.
</details>

### Sub-task 3: File Reading with `while read`
Write a script that reads `fruits.txt` line by line and prints "Fruit: <name>" with line numbers.

```bash
#!/bin/bash
i=1
while IFS= read -r fruit; do
    echo "Line $i: $fruit"
    ((i++))
done < /tmp/loop_lab/fruits.txt
```

**Approach 1:** Use a counter variable as shown
**Approach 2:** Use `nl` and pipe: `nl -ba fruits.txt | while IFS= read -r line; do echo "Fruit: $line"; done`
**Approach 3:** Use `awk`: `awk '{print "Fruit:", $0}' fruits.txt`

Expected output:
```
Line 1: apple
Line 2: banana
Line 3: cherry
Line 4: date
Line 5: elderberry
Line 6: fig
Line 7: grape
```

**Bonus:** Read the CSV file and print each person's name and email:
```bash
while IFS=, read -r name age email; do
    [ "$name" != "name" ] && echo "$name <$email>"
done < /tmp/loop_lab/people.csv
```

<details><summary>Hint: Why IFS= and -r?</summary>
`IFS=` prevents leading/trailing whitespace from being stripped. `-r` prevents backslash interpretation. Together they form the "safe read" pattern. Without them, data like "  hello  " becomes "hello" and "hello\world" might have the backslash interpreted.
</details>

### Sub-task 4: C-Style Counting — Multiplication Table
Write a script that prints the multiplication table for 7 (7x1 through 7x12) using a C-style `for` loop:

```bash
#!/bin/bash
n=7
for (( i=1; i <= 12; i++ )); do
    echo "$n x $i = $((n * i))"
done
```

**Approach 1:** C-style `for` as shown
**Approach 2:** Brace expansion: `for i in {1..12}; do echo "$n x $i = $((n*i))"; done`
**Approach 3:** `while` loop with counter

Expected output:
```
7 x 1 = 7
7 x 2 = 14
7 x 3 = 21
7 x 4 = 28
7 x 5 = 35
7 x 6 = 42
7 x 7 = 49
7 x 8 = 56
7 x 9 = 63
7 x 10 = 70
7 x 11 = 77
7 x 12 = 84
```

**Bonus:** Create a full multiplication table (1-12) using nested loops, formatted with columns.

<details><summary>Hint: C-style for syntax</summary>
`for (( init; condition; increment ))` — the double parentheses are MANDATORY. Inside, you can use arithmetic expressions without `$`. Multiple expressions: `for ((i=0, j=10; i<j; i++, j--))`.
</details>

### Sub-task 5: Nested Loops — Directory Matrix
Create a hierarchical directory structure:

```bash
#!/bin/bash
cd /tmp/loop_lab
mkdir -p products

for color in red green blue; do
    for size in small medium large; do
        mkdir -p "products/${color}/${size}"
        echo "Created: products/${color}/${size}"
    done
done

# Show the result
echo
echo "Directory structure:"
find products -type d | sort
```

**Approach 1:** Nested `for` loops as shown
**Approach 2:** Brace expansion: `mkdir -p products/{red,green,blue}/{small,medium,large}`
**Approach 3:** Single loop with arrays: `colors=(red green blue); sizes=(small medium large); for c in "${colors[@]}"; do for s in "${sizes[@]}"; do ... done; done`

Expected directory tree:
```
products/
  red/
    small/
    medium/
    large/
  green/
    small/
    medium/
    large/
  blue/
    small/
    medium/
    large/
```

**Bonus:** Create files inside each directory:
```bash
for color in red green blue; do
    for size in small medium large; do
        echo "This is a ${size} ${color} widget" > "products/${color}/${size}/readme.txt"
    done
done
```

### Sub-task 6: Until Loop — Retry Pattern
Write a script that retries a command until it succeeds:

```bash
#!/bin/bash
max_attempts=5
attempt=1

until ping -c1 8.8.8.8 &>/dev/null || [ "$attempt" -ge "$max_attempts" ]; do
    echo "Attempt $attempt: ping failed, retrying..."
    ((attempt++))
    sleep 1
done

if [ "$attempt" -lt "$max_attempts" ]; then
    echo "Ping succeeded on attempt $attempt"
else
    echo "All $max_attempts attempts failed."
fi
```

**Approach 1:** `until` loop with OR condition
**Approach 2:** `while` loop inverted: `while ! ping ... && [ "$attempt" -lt "$max_attempts" ]`
**Approach 3:** Infinite loop with `break`: `while true; do ping ... && break; ((attempt++)); [ "$attempt" -ge "$max" ] && break; done`

<details><summary>Hint: Combining conditions in until</summary>
`until cmd1 || cmd2` — the loop runs while BOTH cmd1 AND cmd2 fail. When either succeeds, the loop stops. In our example: ping succeeds (exit 0) OR max attempts reached (exit 0) → stop.
</details>

### Sub-task 7: Process Substitution — Save While-Read Results
Show how process substitution preserves variables (unlike pipes):

```bash
#!/bin/bash
cd /tmp/loop_lab

# This does NOT work — variables lost in pipe:
count_pipe=0
ls *.bak 2>/dev/null | while IFS= read -r f; do
    ((count_pipe++))
done
echo "Pipe count: $count_pipe"  # Shows 0!

# This DOES work — process substitution keeps current shell:
count_proc=0
while IFS= read -r f; do
    ((count_proc++))
done < <(ls *.bak 2>/dev/null)
echo "Process sub count: $count_proc"  # Shows correct count
```

**Bonus:** Use `find -print0` with `while IFS= read -r -d ''` for perfectly safe filename handling.

## Expected Output Summary

```bash
$ ls *.bak
file_1.bak  file_10.bak  file_2.bak  file_3.bak  file_4.bak
file_5.bak  file_6.bak  file_7.bak  file_8.bak  file_9.bak

$ ./read_fruits.sh
Line 1: apple
Line 2: banana
Line 3: cherry
Line 4: date
Line 5: elderberry
Line 6: fig
Line 7: grape

$ ./mult_table.sh
7 x 1 = 7
7 x 2 = 14
...
7 x 12 = 84

$ find products -type d | sort
products
products/blue
products/blue/large
products/blue/medium
products/blue/small
products/green
products/green/large
products/green/medium
products/green/small
products/red
products/red/large
products/red/medium
products/red/small
```

## Self-Check Questions

1. Why is `for f in $(ls *.txt)` bad practice?

2. What does `IFS=` in `while IFS= read -r line` do?

3. What is the difference between `break` and `continue`?

4. How would you loop over numbers 1 to 100 without using `{1..100}`?

5. Why does `for f in *.txt` pass `*.txt` literally when no .txt files exist?

6. What happens to variables set inside a `while` loop that's part of a pipeline?

7. How does `break N` work with nested loops?
