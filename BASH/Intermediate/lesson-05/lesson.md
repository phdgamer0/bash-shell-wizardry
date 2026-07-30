# Lesson 5: Indexed Arrays

## History & Origins

Arrays in shell scripting originated in the **Korn shell (ksh88)**, which introduced one-dimensional indexed arrays. David Korn recognized that shell scripts needed structured data beyond simple variables.

Bash adopted arrays in **bash 2.0** (1996). The syntax `${arr[@]}` and `${arr[*]}` was modeled after `$@` and `$*`. The `+=` append syntax came in **bash 3.1** (2005). Negative indexing (`arr[-1]`) arrived in **bash 4.3** (2014).

Arrays fill the gap between simple variables and external data processing tools.

## Syntax Reference

| Expression | What it does | Example |
|------------|-------------|---------|
| `arr=(v1 v2 v3)` | Create array | `names=("Alice" "Bob")` |
| `arr+=("v4")` | Append element(s) | `names+=("Diana")` |
| `arr[5]="v6"` | Assign by index | `names[5]="Eve"` |
| `${arr[i]}` | Access by index | `echo "${names[0]}"` |
| `${arr[@]}` | All elements (separate words) | `for i in "${arr[@]}"` |
| `${arr[*]}` | All elements (one word) | `echo "${arr[*]}"` |
| `${#arr[@]}` | Element count | `echo "Count: ${#arr[@]}"` |
| `${!arr[@]}` | Array indices (keys) | `for i in "${!arr[@]}"` |
| `unset arr[i]` | Remove element | `unset 'arr[1]'` |
| `unset arr` | Remove entire array | `unset arr` |
| `${arr[-1]}` | Last element (bash 4.3+) | `echo "${arr[-1]}"` |

### The `[@]` vs `[*]` Distinction

**Most important array concept**:

- `"${arr[@]}"`: Each element is a **separate word** (preserves spaces)
- `"${arr[*]}"`: All elements as **one word** joined by IFS first char
- `${arr[@]}` (unquoted): Word-splits each element — DANGEROUS with spaces

```bash
$ arr=("file with spaces.txt" "normal.txt")
$ printf "%s\n" ${arr[@]}
file
with
spaces.txt
normal.txt
# ^^ BAD — 4 lines

$ printf "%s\n" "${arr[@]}"
file with spaces.txt
normal.txt
# ^^ GOOD — 2 lines preserved
```

### Creating Arrays

```bash
# Literal
arr=("apple" "banana" "cherry")

# From a glob
arr=(*.txt)

# From file (bash 4.0+)
readarray arr < file.txt      # each line is an element
mapfile -t arr < file.txt     # same, with -t to trim newlines

# From here-string
read -ra arr <<< "1 2 3"

# Sparse arrays
arr[0]="first"
arr[5]="sixth"       # indices 1-4 are unset
```

### Iteration Patterns

```bash
# Iterate values
for val in "${arr[@]}"; do echo "$val"; done

# Iterate indices
for i in "${!arr[@]}"; do echo "Index $i: ${arr[$i]}"; done

# C-style
for ((i=0; i<${#arr[@]}; i++)); do echo "$i: ${arr[$i]}"; done
```

### Stack & Queue Operations

**Stack (LIFO)**:
```bash
stack+=("first")     # push
stack+=("second")
peek="${stack[-1]}"  # peek
unset 'stack[-1]'    # pop
```

**Queue (FIFO)**:
```bash
queue=("job1" "job2")
next="${queue[0]}"           # dequeue (peek)
queue=("${queue[@]:1}")      # shift (remove first)
```

**Better queue** (index pointer for large arrays):
```bash
queue=("job1" "job2" "job3")
head=0
next="${queue[$head]}"; ((head++))  # dequeue without copying
```

### Slicing

```bash
nums=(0 1 2 3 4 5 6 7 8 9)
slice=("${nums[@]:3:4}")   # 3 4 5 6
last3=("${nums[@]: -3}")   # 7 8 9
```

## Under the Hood

### What bash does

Bash arrays use a `ARRAY` struct containing a `char **elements` pointer, element count, and max index. Memory grows in ~1.5x chunks. Sparse arrays leave NULL holes.

- Index access `arr[i]`: O(1) direct lookup
- Append: Amortized O(1) with occasional reallocation
- Slice: O(n) — copies pointers
- Removing first element via slice: O(n) — copies entire array minus first

### System Calls

Array operations: **zero system calls**. Everything is in-process.

But populating from commands forks: `arr=($(ls))`, `readarray arr < <(find ...)`.

### Memory

- Array header in shell's variable hash table
- Element strings individually heap-allocated
- Sparse arrays waste memory proportional to gap between min/max index
- Large arrays (100K+) show noticeable overhead

### Equivalent Python

```python
arr = [1, 2, 3]                          # arr=(1 2 3)
arr.append("4")                          # arr+=("4")
len(arr)                                 # ${#arr[@]}
arr[-1]                                  # ${arr[-1]}
range(len(arr))                          # ${!arr[@]}
slice = arr[2:2+3]                       # ("${arr[@]:2:3}")
del arr[2]                               # unset 'arr[2]'
```

## Core Examples (12 minimum)

### Example 1: Create and access

```bash
$ fruits=("apple" "banana" "cherry")
$ echo "${fruits[0]}"
apple
$ echo "${fruits[-1]}"
cherry
```

**What if** `fruits[10]` is accessed? "" (empty) — no error.

### Example 2: All elements and length

```bash
$ echo "${fruits[@]}"
apple banana cherry
$ echo "${#fruits[@]}"
3
```

**What if** you forget `[@]`? `${#fruits}` = length of first element (5 = "apple").

### Example 3: Append and modify

```bash
$ fruits+=("date" "elderberry")
$ echo "${#fruits[@]}"
5
$ fruits[1]="blueberry"
$ echo "${fruits[1]}"
blueberry
```

**Trap**: `fruits+="date"` (without parens) appends "date" to `fruits[0]` as a string! Use `+=("value")`.

### Example 4: Iterating with indices

```bash
$ for i in "${!fruits[@]}"; do
>   echo "Index $i: ${fruits[$i]}"
> done
Index 0: apple
Index 1: blueberry
```

### Example 5: Iterating values

```bash
$ for fruit in "${fruits[@]}"; do
>   echo "I like $fruit"
> done
I like apple
I like blueberry
```

### Example 6: Slicing arrays

```bash
$ nums=(0 1 2 3 4 5 6 7 8 9)
$ slice=("${nums[@]:3:4}")
$ echo "${slice[@]}"
3 4 5 6
```

### Example 7: Stack operations (push/pop/peek)

```bash
$ stack=()
$ stack+=("first")
$ stack+=("second")
$ stack+=("third")
$ echo "${stack[-1]}"    # peek → "third"
$ unset 'stack[-1]'      # pop
$ echo "${stack[@]}"     # → "first second"
```

### Example 8: Queue operations

```bash
$ queue=("job1" "job2" "job3")
$ echo "Processing: ${queue[0]}"
Processing: job1
$ queue=("${queue[@]:1}")   # remove first
$ echo "Remaining: ${queue[@]}"
Remaining: job2 job3
```

### Example 9: Read lines into array

```bash
$ cat > /tmp/fruits.txt <<< $'apple\nbanana\ncherry'
$ readarray fruits < /tmp/fruits.txt
$ echo "${fruits[1]}"
banana
$ echo "${#fruits[@]}"
3
```

### Example 10: Array of filenames from glob

```bash
$ files=(*.txt)
$ echo "Found ${#files[@]} txt files"
$ for f in "${files[@]}"; do echo "Processing: $f"; done
```

### Example 11: Array from command substitution (careful!)

```bash
$ dirs=($(ls -d */))   # BAD — splits on spaces
$ dirs=(*/)             # GOOD — uses glob directly
```

### Example 12: Sparse array behavior

```bash
$ arr=([0]="zero" [3]="three" [7]="seven")
$ echo "${#arr[@]}"
3
$ echo "${!arr[@]}"
0 3 7
$ echo "${arr[5]}"      # unset → empty
```

### Example 13: Composite array patterns

```bash
$ students=("Alice:90" "Bob:85" "Charlie:95")
$ for s in "${students[@]}"; do
>   name="${s%:*}"
>   grade="${s#*:}"
>   echo "$name scored $grade"
> done
Alice scored 90
Bob scored 85
Charlie scored 95
```

## Real-World Use Cases

### FOR the OS

- Command history tracking (like bash's own `HISTFILE`)
- Job queues for batch processing
- Multi-value configuration
- Staging multiple files for batch operations

### WITH the OS

- `readarray` + `find` for file lists
- Process lists from `/proc` into arrays
- Parsing mount outputs, df rows into structured data

### AGAINST the OS (maintenance)

- Rotating log file lists: oldest removal with slice
- Process watch lists: arrays of PIDs to monitor
- Batch cleanup: arrays of temp files to remove

### FOR DEFENSE

- Whitelist valid commands in an array
- Track PIDs for child process management
- Rate-limit tracking with rolling window arrays

## Memory Aids

- **`arr=( )`**: Parentheses hug the values like arms holding them together
- **`${arr[@]}`**: The `@` is like a spreading fan — each element spreads apart
- **`${arr[*]}`**: The `*` is like a glue stick — all elements stick together
- **`${!arr[@]}`**: The `!` is like pointing fingers — "here are the indices!"
- **`arr[-1]`**: Negative index = looking from the end (last = -1)

## Trap Vault (12 traps)

### Trap 1: Unquoted `${arr[@]}` splits on spaces

```bash
arr=("file with spaces.txt" "normal.txt")
for f in ${arr[@]}; do echo "$f"; done  # BAD — 4 iterations
for f in "${arr[@]}"; do echo "$f"; done  # GOOD — 2 iterations
```

### Trap 2: `$arr` prints only first element

```bash
arr=("a" "b" "c")
echo $arr    # "a" — not "a b c"!
# Use "${arr[@]}" for all elements
```

### Trap 3: `arr="string"` creates a string, not an array

```bash
arr="hello"          # string variable
echo "${arr[0]}"     # "hello" (bash treats string as arr[0] for compat)
echo "${arr[1]}"     # "" (not an array element)
# FIX: arr=("hello")
```

### Trap 4: `+=` without parens appends to first element

```bash
arr=("a" "b")
arr+="c"             # appends "c" to arr[0] → arr=("ac" "b")
arr+=("c")           # appends "c" as new element → arr=("a" "b" "c")
```

### Trap 5: `unset arr[i]` without quoting may glob

```bash
arr=("a" "b" "c")
unset arr[1]         # works, but if you had a file called "arr[1]"? glob!
unset 'arr[1]'       # safe — quoting prevents glob expansion
```

### Trap 6: `${#arr}` vs `${#arr[@]}` confusion

```bash
arr=("hello" "world")
echo "${#arr}"       # 5 (length of arr[0] = "hello")
echo "${#arr[@]}"    # 2 (number of elements)
```

### Trap 7: Sparse arrays after `unset`

```bash
arr=(0 1 2 3)
unset 'arr[1]'
echo "${#arr[@]}"    # 3
echo "${arr[@]}"     # "0 2 3"
echo "${!arr[@]}"    # "0 2 3" — index 1 is gone
# Don't assume contiguous indices after unset
```

### Trap 8: Index 0 vs unset confusion

```bash
# "${arr[@]}" and "${arr[*]}" both expand to nothing for empty arrays
# But ${arr[0]} also expands to nothing for empty arrays
# There's no way to distinguish "empty array" from "first element is empty string"
```

### Trap 9: Negative index unsupported in older bash

```bash
# Bash < 4.3: arr[-1] gives last element? NO — error
arr=("a" "b")
echo "${arr[-1]}"   # bash 3.2: error or unexpected behavior
# FIX: Use ${arr[${#arr[@]}-1]} for portability
```

### Trap 10: `declare -a` vs `declare -A`

```bash
declare -a arr    # indexed array (explicit, same as arr=())
declare -A map    # associative array (REQUIRED for hash maps)
```

### Trap 11: readarray with `-t` vs without

```bash
readarray arr < file.txt   # each line INCLUDES newline!
mapfile -t arr < file.txt  # strips newlines
# The newline at end of each line can cause bugs — always use -t for text
```

### Trap 12: Copying arrays with slice vs assignment

```bash
copy=("${arr[@]}")          # creates a NEW array (proper copy)
copy2=$arr                  # BAD — creates string "arr[0]"
copy3="${arr[@]}"           # BAD — creates a string, not array
```

## See It In The Wild

- `$@` and `$*` — the original "arrays" in bash
- `$HISTFILE` — bash's own command history array
- `$COMPREPLY` — completion system uses arrays
- `$PIPESTATUS` — array of exit codes from pipelines

### Exploration

1. Create a sparse array (indices 0, 5, 10). What's `${#arr[@]}`?
2. Compare timing of `for i in "${arr[@]}"` vs `for ((i=0; i<${#arr[@]}; i++))` on large arrays
3. Try `read -ra arr <<< "a b c"` vs `arr=("a" "b" "c")` — same result?

## Check Your Understanding (7 questions)

1. **Why does `$arr` print only the first element?**
2. **Difference between `"${arr[@]}"` and `"${arr[*]}"`?**
3. **How to get the number of elements in an array?**
4. **How to safely delete the last element?**
5. **What happens if you assign `arr="hello"` instead of `arr=("hello")`?**
6. **How to iterate over both indices and values?**
7. **What does `unset 'arr[2]'` do to array indices?**

## Supplementary Deep Dive: Advanced Array Patterns

### Finding min/max in arrays

```bash
$ nums=(42 17 8 99 53 3 71)
$ min=${nums[0]}; max=${nums[0]}
$ for n in "${nums[@]}"; do
>   (( n < min )) && min=$n
>   (( n > max )) && max=$n
> done
$ echo "Min: $min, Max: $max"
Min: 3, Max: 99
```

### Checking if element exists in array

```bash
$ in_array() {
>   local needle="$1"; shift
>   local item
>   for item; do
>     [[ "$item" == "$needle" ]] && return 0
>   done
>   return 1
> }
$ fruits=("apple" "banana" "cherry")
$ in_array "banana" "${fruits[@]}" && echo "Found" || echo "Not found"
Found
$ in_array "grape" "${fruits[@]}" && echo "Found" || echo "Not found"
Not found
```

### Array deduplication (preserving order)

```bash
$ dedup() {
>   local -A seen
>   local result=()
>   local item
>   for item; do
>     if [[ -z "${seen[$item]}" ]]; then
>       result+=("$item")
>       seen[$item]=1
>     fi
>   done
>   echo "${result[@]}"
> }
$ items=(a b a c b d a)
$ dedup "${items[@]}"
a b c d
```

### Array intersection

```bash
$ intersect() {
>   local -A seen
>   local result=()
>   local item
>   for item; do seen[$item]=1; done
>   while [[ $# -gt 0 ]]; do shift; done  # reset
>   # second array is (what remains? No — need two arrays)
>   # Better approach:
> }
$ # Simpler: pipe through sort/uniq
$ arr1=(1 2 3 4 5)
$ arr2=(4 5 6 7 8)
$ printf '%s\n' "${arr1[@]}" | sort > /tmp/arr1.txt
$ printf '%s\n' "${arr2[@]}" | sort > /tmp/arr2.txt
$ comm -12 /tmp/arr1.txt /tmp/arr2.txt | tr '\n' ' '
4 5
```

### Array rotation

```bash
$ rotate_left() {
>   local -n arr=$1
>   (( ${#arr[@]} <= 1 )) && return
>   local first="${arr[0]}"
>   arr=("${arr[@]:1}" "$first")
> }
$ items=(a b c d e)
$ rotate_left items
$ echo "${items[@]}"
b c d e a
```

### Arrays as function return values

Bash can't return arrays from functions directly. Use nameref (bash 4.3+):

```bash
$ get_files() {
>   local -n _result=$1
>   _result=(*.txt)
> }
$ declare -a files
$ get_files files
$ echo "${files[0]}"
readme.txt
```

### Multi-dimensional arrays via key naming

```bash
$ # Simulate 2D array with flat naming
$ declare -A matrix
$ for ((r=0; r<3; r++)); do
>   for ((c=0; c<3; c++)); do
>     matrix["$r,$c"]=$((r*10 + c))
>   done
> done
$ echo "${matrix[1,2]}"   # row 1, col 2 = 12
12
$ echo "${!matrix[@]}"    # all coordinates
0,0 0,1 0,2 1,0 1,1 1,2 2,0 2,1 2,2
```

### Array with filenames containing spaces

```bash
$ # CORRECT way
$ files=()
$ while IFS= read -r -d '' f; do
>   files+=("$f")
> done < <(find . -name "*.txt" -print0)
$ echo "Found ${#files[@]} files"
$ for f in "${files[@]}"; do
>   echo "Processing: $f"
> done
```

### Random element selection

```bash
$ random_pick() {
>   local -n arr=$1
>   echo "${arr[$((RANDOM % ${#arr[@]}))]}"
> }
$ colors=(red green blue yellow orange)
$ random_pick colors
blue
```

### Using arrays for batch command building

```bash
$ cmd=(git commit -m "Fix bug")
$ echo "${cmd[@]}"
git commit -m Fix bug
$ # Oops — need to keep "Fix bug" as one arg
$ cmd=(git commit -m "Fix bug")
$ "${cmd[@]}"  # properly executes with "Fix bug" as single argument
```
