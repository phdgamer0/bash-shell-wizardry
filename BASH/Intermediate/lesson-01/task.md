# Task 1: Arithmetic

## Overview

Build a set of three shell scripts that demonstrate mastery of bash arithmetic — from basic integer math to floating point with `bc`, and algorithmic thinking with loop arithmetic.

## Sub-tasks

### 1. Calculator (calc.sh)

Write a script that takes an operator (`+`, `-`, `*`, `/`, `%`, `**`) and two numbers, and performs the operation.

**Requirements:**
- Use `$(( ))` for integer operations (`+`, `-`, `*`, `%`, `**`)
- Use `bc` for division (to get floating point)
- Handle division by zero gracefully with an error message
- Use a `case` statement to dispatch the operator
- The operator is the first argument, num1 is second, num2 is third

**Solution approach 1 — case statement:**
```bash
#!/bin/bash
op=$1; a=$2; b=$3
case $op in
    +) echo $((a + b)) ;;
    -) echo $((a - b)) ;;
    '*') echo $((a * b)) ;;
    /) [[ $b -eq 0 ]] && echo "Error: division by zero" || echo "scale=4; $a / $b" | bc ;;
    %) echo $((a % b)) ;;
    '**') echo $(($a ** $b)) ;;
    *) echo "Unknown operator: $op" >&2; exit 1 ;;
esac
```

**Solution approach 2 — function-based:**
```bash
#!/bin/bash
calc() {
    local op=$1 a=$2 b=$3
    case $op in
        +|-) echo $(($a $op $b)) ;;
        '*') echo $((a * b)) ;;
        /)  if (( b == 0 )); then echo "Error: div by zero" >&2; return 1
            else printf "%.4f\n" $(echo "scale=4; $a / $b" | bc)
            fi ;;
        '%') echo $((a % b)) ;;
        '**') echo $(($a ** $b)) ;;
    esac
}
calc "$@"
```

**Solution approach 3 — one-liner with eval (warning: eval is dangerous, educational only):**
```bash
#!/bin/bash
[ "$1" = "/" ] && { [ "$3" -eq 0 ] && echo "Error" || echo "scale=4; $2/$3" | bc; } || echo $(($2 ${1//\*/\\*} $3))
```

<details>
<summary>Hint: Quoting the * operator</summary>
The `*` is a glob character in the shell. When running `./calc.sh * 5 3`, the shell expands `*` to all filenames! Run it as `./calc.sh '*' 5 3` or `./calc.sh \* 5 3`.
</details>

<details>
<summary>Hint: Division by zero detection</summary>
```bash
if [[ $b -eq 0 ]]; then
    echo "Error: division by zero"
    exit 1
fi
```
Or use arithmetic: `(( b == 0 )) && { echo "Error"; exit 1; }`
</details>

### 2. Disk Usage Reporter (disk_usage.sh)

Read two numbers (used and total blocks from `df` output) and print:
- Usage percentage (integer)
- Available percentage (2 decimal places with bc)
- A visual bar `[########....]`

**Requirements:**
- Accept used and total as arguments (or read from df if no args)
- The bar should be 20 characters wide, `#` for used, `-` for free
- Use `bc` for decimal free percentage
- Use `(( ))` for integer percentage

**Solution approach 1 — df integration:**
```bash
#!/bin/bash
if [[ $# -lt 2 ]]; then
    read used total <<< $(df / | tail -1 | awk '{print $3, $2}')
else
    used=$1; total=$2
fi

int_pct=$(( used * 100 / total ))
free_pct=$(echo "scale=2; ($total - $used) * 100 / $total" | bc)

hashes=$(( int_pct / 5 ))
dashes=$(( 20 - hashes ))

printf "Usage: %d%%\n" $int_pct
printf "Free:  %.2f%%\n" $free_pct
printf "["
for ((i=0; i<hashes; i++)); do printf "#"; done
for ((i=0; i<dashes; i++)); do printf "-"; done
printf "]\n"
```

**Solution approach 2 — pure arithmetic with visual refinement:**
```bash
./disk_usage.sh 500 1000
# Output:
# Usage: 50%
# Free:  50.00%
# [##########----------]
```

<details>
<summary>Hint: Building the progress bar</summary>
Use a loop to print `#` `pct/5` times and `-` `20 - pct/5` times. Or use `printf` with `%.0s` trick:
```bash
printf '#%.0s' $(seq 1 $((pct / 5)))
printf -- '-%.0s' $(seq 1 $((20 - pct / 5)))
```
</details>

### 3. Fibonacci (fib.sh)

Print the first N Fibonacci numbers using `(( ))` for the loop.

**Requirements:**
- Accept N as argument (default 10)
- Use `(( ))` style arithmetic for the loop
- Start with 0, 1, 1, 2, 3, 5, 8, 13...
- Space-separated output

**Solution approach 1 — while loop:**
```bash
#!/bin/bash
n=${1:-10}
a=0; b=1
for ((i=0; i<n; i++)); do
    echo -n "$a "
    ((c = a + b))
    ((a = b))
    ((b = c))
done
echo
```

**Solution approach 2 — arithmetic-only initialization:**
```bash
#!/bin/bash
n=${1:-10}; a=0; b=1; i=0
while (( i++ < n )); do
    echo -n "$a "
    ((a+=b, b=a-b, a=a-b))   # swap trick without temp var
done
echo
```
Step-by-step: `a+=b` makes a = original a + b. `b=a-b` makes b = (original a + b) - original b = original a. `a=a-b` makes a = (original a + b) - original a = original b. Actual Fibonacci: this doesn't quite work for Fibonacci — it swaps a and b. For real Fibonacci:
```bash
((t = a, a = b, b = t + b))   # classic swap & advance
```

<details>
<summary>Hint: Fibonacci logic with 3 variables</summary>
```bash
# Next = previous + current
# Save current before updating:
#   next = a + b
#   a = b
#   b = next
```
</details>

### 4. Bonus: Prime Checker (prime.sh)

Check if a number is prime using arithmetic. Use `(( ))` for the loop.

```bash
#!/bin/bash
n=${1:?Usage: $0 number}
if (( n < 2 )); then echo "$n is not prime"; exit; fi
if (( n == 2 )); then echo "$n is prime"; exit; fi
if (( n % 2 == 0 )); then echo "$n is not prime (divisible by 2)"; exit; fi

for ((i=3; i*i <= n; i+=2)); do
    if (( n % i == 0 )); then
        echo "$n is not prime (divisible by $i)"
        exit
    fi
done
echo "$n is prime"
```

### 5. Bonus: bc math library exploration

Explore `bc -l` functions:
```bash
$ echo "s(3.14159/2)" | bc -l   # sine of π/2
$ echo "c(0)" | bc -l            # cosine of 0
$ echo "e(1)" | bc -l            # e^1 = e
$ echo "l(100)" | bc -l          # ln(100)
$ echo "sqrt(144)" | bc          # square root
```

<details>
<summary>Hint: bc -l sets scale=20</summary>
With the `-l` flag, `scale` defaults to 20. You can override: `echo "scale=10; s(1)" | bc -l`
</details>

## Expected Output

```
$ ./calc.sh 22 / 7
3.1429

$ ./calc.sh 5 '*' 3
15

$ ./calc.sh 10 / 0
Error: division by zero

$ ./calc.sh 10 '%' 3
1

$ ./disk_usage.sh 500 1000
Usage: 50%
Free:  50.00%
[##########----------]

$ ./fib.sh 8
0 1 1 2 3 5 8 13

$ ./fib.sh 1
0

$ ./prime.sh 17
17 is prime

$ ./prime.sh 15
15 is not prime (divisible by 3)
```

## Self-Check

- What exit code does `(( 0 ))` return? What about `(( 1 ))`?
- Why does `echo $((5/2))` print `2` instead of `2.5`?
- How do you compute `sqrt(2)` to 4 decimal places in bash?
- What happens if you omit `scale` in `bc`?
- Why should you avoid `expr` in modern scripts?
- How does `$((08 + 1))` error, and what's the fix?
- What's the output of `echo $((2 ** 3 ** 2))`?
