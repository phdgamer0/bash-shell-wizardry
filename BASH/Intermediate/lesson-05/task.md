# Task 5: Indexed Arrays

## Overview

Build three scripts demonstrating array data structures: history tracker, stack, and queue. Each script should demonstrate a different aspect of array manipulation.

## Sub-tasks

### 1. History Tracker (history.sh)

Stores up to 10 command strings. Commands:
- `push <cmd>` — add a command (when >10, remove oldest)
- `show` — display all commands numbered
- `last N` — show the last N commands
- `clear` — clear all history

**Requirements:**
- Use an array to store commands
- When >10 entries, remove the oldest
- Show commands numbered from 1

```bash
#!/bin/bash
history=()
HISTFILE="${HOME}/.cli_history"
[[ -f "$HISTFILE" ]] && mapfile -t history < "$HISTFILE"

save() { printf '%s\n' "${history[@]}" > "$HISTFILE"; }

push() {
    history+=("$*")
    if (( ${#history[@]} > 10 )); then
        history=("${history[@]: -10}")   # keep newest 10
    fi
    save
}

show() {
    for i in "${!history[@]}"; do
        printf "%3d: %s\n" $((i+1)) "${history[$i]}"
    done
}

last() {
    local n=${1:-5}
    local start=$(( ${#history[@]} - n ))
    (( start < 0 )) && start=0
    for ((i=start; i<${#history[@]}; i++)); do
        printf "%3d: %s\n" $((i+1)) "${history[$i]}"
    done
}

clear() { history=(); save; }

case "$1" in
    push) shift; push "$@" ;;
    show) show ;;
    last) last "$2" ;;
    clear) clear ;;
    *) echo "Usage: $0 {push|show|last N|clear}" ;;
esac
```

**Step-by-step for rotation:**
When 11th element is pushed: `history=("${history[@]: -10}")` slices the last 10 elements, discarding the oldest (first element). The `:-10` negative offset means "start from 10th from end, take to end."

<details>
<summary>Hint: File persistence with mapfile</summary>
`mapfile -t arr < file` reads file lines into array (stripping newlines). `printf '%s\n' "${arr[@]}" > file` writes them back. This enables persistent history across script invocations.
</details>

### 2. Stack (stack.sh)

Stack with three functions: `push`, `pop`, `peek`. Use an array.

**Requirements:**
- Push 5 items: a, b, c, d, e
- Pop 2, peek at top, show all remaining
- Handle empty stack gracefully (don't pop from empty)

```bash
#!/bin/bash
stack=()

push() { stack+=("$1"); }

pop() {
    if (( ${#stack[@]} == 0 )); then
        echo "Stack empty" >&2
        return 1
    fi
    local top="${stack[-1]}"
    unset 'stack[-1]'
    echo "$top"
}

peek() {
    if (( ${#stack[@]} == 0 )); then
        echo "Stack empty" >&2
        return 1
    fi
    echo "${stack[-1]}"
}

show() { echo "Stack: ${stack[*]}"; }

# Demo
for item in a b c d e; do push "$item"; done
echo "Pushed: a, b, c, d, e"
echo "Pop: $(pop)"
echo "Pop: $(pop)"
echo "Peek: $(peek)"
show
```

**Alternative implementation using index pointer:**
```bash
stack=()
top=-1
push() { stack[++top]="$1"; }
pop() { [[ top -ge 0 ]] && echo "${stack[top--]}" || echo "Empty"; }
peek() { [[ top -ge 0 ]] && echo "${stack[top]}" || echo "Empty"; }
```

<details>
<summary>Hint: Quoting unset</summary>
Always quote the array index in `unset`: `unset 'stack[-1]'`. Without quotes, `[-1]` could be interpreted as a glob pattern.
</details>

### 3. Queue (queue.sh)

Simulate a print queue. Commands:
- `enqueue "JobName"` — add job
- `dequeue` — remove first job, show it
- `list` — show all pending jobs
- `length` — show queue length

```bash
#!/bin/bash
queue=()

enqueue() { queue+=("$1"); }

dequeue() {
    if (( ${#queue[@]} == 0 )); then
        echo "Queue empty" >&2
        return 1
    fi
    local job="${queue[0]}"
    queue=("${queue[@]:1}")     # remove first element
    echo "Processing: $job"
}

list() {
    if (( ${#queue[@]} == 0 )); then
        echo "Queue empty"
        return
    fi
    for i in "${!queue[@]}"; do
        printf "%3d: %s\n" $((i+1)) "${queue[$i]}"
    done
}

length() { echo "${#queue[@]}"; }
```

**Alternative — index pointer (avoids O(n) copy):**
```bash
queue=()
head=0
enqueue() { queue+=("$1"); }
dequeue() {
    if (( head >= ${#queue[@]} )); then echo "Queue empty"; return 1; fi
    echo "Processing: ${queue[$head]}"
    ((head++))
}
list() {
    for ((i=head; i<${#queue[@]}; i++)); do
        printf "%3d: %s\n" $((i-head+1)) "${queue[$i]}"
    done
}
# Reset: queue=(); head=0
```

<details>
<summary>Hint: Index pointer advantage</summary>
The slice approach `queue=("${queue[@]:1}")` copies the entire array except the first element — O(n). For large queues, use an index pointer `head` instead — O(1) dequeue.
</details>

### 4. Bonus: Circular Buffer

```bash
#!/bin/bash
SIZE=5
buf=()
head=0
count=0

write() {
    buf[$head]=$1
    ((head = (head + 1) % SIZE))
    ((count < SIZE && count++))
}

read_all() {
    local i
    for ((i=0; i<count; i++)); do
        idx=$(( (head - count + i + SIZE) % SIZE ))
        echo "${buf[$idx]}"
    done
}

# Test
write a; write b; write c; write d; write e; write f  # f overwrites a
read_all
```

### 5. Bonus: Multi-dimensional Simulation

```bash
#!/bin/bash
# Simulate a 2D matrix using flat array
rows=3 cols=4
matrix=()
for ((r=0; r<rows; r++)); do
    for ((c=0; c<cols; c++)); do
        matrix[r*cols + c]=$((r*10 + c))
    done
done

# Access element at (1,2)
echo "${matrix[1*cols + 2]}"   # 12
```

## Expected Output

```
$ ./stack.sh
Pushed: a, b, c, d, e
Pop: e
Pop: d
Peek: c
Stack: a b c

$ ./queue.sh
Enqueued: Print Job 1, Print Job 2, Print Job 3
Processing: Print Job 1
Processing: Print Job 2
Pending: Print Job 3

$ ./history.sh push ls -la
$ ./history.sh push pwd
$ ./history.sh push "echo test"
$ ./history.sh show
  1: ls -la
  2: pwd
  3: echo test
$ ./history.sh last 2
  2: pwd
  3: echo test
$ ./history.sh push "ping google.com"
$ ./history.sh push "df -h"
...
$ ./history.sh push "docker ps"
$ ./history.sh show   # shows only the latest 10
```

## Self-Check

- Why does `echo $arr` print only the first element?
- What's the difference between `"${arr[@]}"` and `"${arr[*]}"`?
- How do you get the number of elements in an array?
- How do you safely delete the last element of an array?
- What happens if you assign `arr="hello"` instead of `arr=("hello")`?
- Which is faster for large queues: `arr=("${arr[@]:1}")` or an index pointer?
- What does `${#arr}` give you vs `${#arr[@]}`?
