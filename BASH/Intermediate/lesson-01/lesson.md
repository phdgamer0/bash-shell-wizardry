# Lesson 1: Arithmetic

## History & Origins

Bash arithmetic traces its lineage directly to the **Bourne shell** (1979, Stephen Bourne at Bell Labs), which introduced `expr` for integer math. The Bourne shell didn't have built-in arithmetic — every `expr` call forked a new process. This was slow, and in the 1980s, the **Korn shell (ksh)** by David Korn at Bell Labs introduced `$(( ))` as a built-in arithmetic expansion. Bash adopted it in the late 1980s as part of the **Bash 1.x** series (Brian Fox, 1989).

Modern Bash (2.0+, 1996) added `(( ))` as an arithmetic compound command with `let` also available for compatibility. `bc` — the arbitrary-precision calculator language — predates Bash entirely, written by Robert Morris and Lorinda Cherry at Bell Labs in 1975. It's not part of Bash but ships with every Unix system because POSIX demands it.

Shell arithmetic has evolved:
- **Bourne shell (1979)**: `expr 2 + 2` — external process, slow
- **POSIX shell**: `$(( 2 + 2 ))` — built-in integer math
- **ksh88**: arrays with arithmetic contexts
- **Bash 2.0**: `(( ))` compound command, `let`, `++`/`--`
- **Bash 4.x**: `printf -v` for assigning arithmetic results
- **bc**: always the external floating-point solution

## Syntax Reference

### Arithmetic Expansion — `$(( expression ))`

Returns the integer result of the expression. Works anywhere a word is expected.

```bash
echo $(( 2 + 3 ))       # → 5
x=$(( y + 1 ))          # assign to variable
echo $(( (a + b) * c )) # grouping
```

- Always returns exit code **0** (even for `$(( 0 ))`)
- Inside `$(( ))`, variables don't need `$` prefix (but `$var` works too)
- Whitespace is flexible: `$((2+3))`, `$(( 2 + 3 ))`, `$((2 +3))` all work
- All vars are treated as integers; undefined/unset vars default to **0**
- Supports **octal** (leading `0`) and **hex** (leading `0x`/`0X`): `$(( 077 ))` is 63, `$(( 0xFF ))` is 255
- **Nested** expansions: `echo $(($((2+3)) * 2))` (ugly but works)
- No floating point — truncates toward zero on division

### Arithmetic Compound Command — `(( expression ))`

Evaluates the expression and sets exit code:
- **Exit code 0** (true/SUCCESS) if result is **non-zero**
- **Exit code 1** (false/FAILURE) if result is **zero**

```bash
(( 5 > 3 )) && echo "yes"     # prints "yes" (exit code 0)
(( 0 )) && echo "yes" || echo "no"  # prints "no" (exit code 1)
(( a = 5 ))  # assignment inside — sets a to 5, exit code 0
(( a = 0 ))  # sets a to 0, exit code 1 (!)
```

- Can be used standalone or in `if`, `while`, `until`
- **Variable assignment** inside `(( ))` works: `(( x = y + 1 ))`
- Supports **post-increment/decrement**: `(( i++ ))`, `(( i-- ))`
- Supports **pre-increment/decrement**: `(( ++i ))`, `(( --i ))`
- Supports **compound assignment**: `(( i += 5 ))`, `(( i %= 3 ))`

### `let` builtin — Legacy

```bash
let x=2+3
let x++ y=z*2
```

- Older syntax, largely superseded by `(( ))`
- No spaces allowed around operators without quoting: `let x = 2 + 3` fails; `let "x = 2 + 3"` works
- Multiple expressions can be space-separated on one `let` line
- Exits with 1 if the last expression evaluates to 0

### `bc` — Arbitrary Precision / Floating Point

```bash
echo "scale=4; 22/7" | bc      # → 3.1428
echo "sqrt(144)" | bc          # → 12
echo "10.5 + 3.2 * 1.5" | bc   # → 15.30
echo "obase=16; 255" | bc      # → FF (base conversion)
```

- `bc` is a full language with variables, conditionals, and functions
- `-l` flag loads the math library: `s(x)` (sine), `c(x)` (cosine), `a(x)` (arctan), `l(x)` (natural log), `e(x)` (exponential), `sqrt(x)` (square root)
- `scale` sets decimal places (default 0)
- `obase` / `ibase` for input/output base conversion
- Can handle arbitrarily large numbers (thousands of digits)
- Division by zero on `bc -l` returns error; plain `bc` returns 0

### All Operators (precedence highest to lowest)

| Operator | Description | Associativity |
|----------|-------------|---------------|
| `++` `--` | post-increment/decrement | left-to-right |
| `++` `--` | pre-increment/decrement | right-to-left |
| `+` `-` `!` `~` | unary plus/minus/bitwise NOT/logical NOT | right-to-left |
| `**` | exponentiation | right-to-left |
| `*` `/` `%` | multiply/divide/remainder | left-to-right |
| `+` `-` | addition/subtraction | left-to-right |
| `<<` `>>` | bitwise shift left/right | left-to-right |
| `<` `<=` `>` `>=` | relational comparison | left-to-right |
| `==` `!=` | equality/inequality | left-to-right |
| `&` | bitwise AND | left-to-right |
| `^` | bitwise XOR | left-to-right |
| `|` | bitwise OR | left-to-right |
| `&&` | logical AND | left-to-right |
| `||` | logical OR | left-to-right |
| `?:` | ternary conditional (5>3 ? 10 : 20) | right-to-left |
| `=` `*=` `/=` `%=` `+=` `-=` `<<=` `>>=` `&=` `^=` `|=` | assignment | right-to-left |
| `,` | comma (evaluate both, return last) | left-to-right |

## Under the Hood

### What Actually Happens

When bash encounters `$(( 2 + 3 * 4 ))`:

1. **Tokenization** — Bash's parser identifies `$((` as arithmetic expansion start, finds matching `))`, extracts the expression
2. **Parsing** — Expression is parsed into an AST (Abstract Syntax Tree) using operator precedence
3. **Variable Resolution** — Variable names in the expression are resolved to their values (using the shell's variable table). Undefined variables = 0
4. **Evaluation** — Bash's own arithmetic evaluator (a C function around 2000+ lines in `expr.c`) recursively walks the AST, performing integer math via native C `long` integers
5. **Result** — Integer result is converted to string and placed in the expansion

### System Calls

For `$(( ))` and `(( ))`: None — **zero system calls** for the math itself. Everything happens in-process. Compare:

```bash
# Bash built-in — 0 syscalls
x=$(( 3 + 5 ))

# expr — multiple syscalls (fork, exec, wait, read, exit)
x=$(expr 3 + 5)   # fork()+exec()+waitpid() — 5+ syscalls
```

A `strace` of `expr 3 + 5` shows:
```
fork(...)         # create child
execve(/usr/bin/expr)  # load expr binary
...expr runs math in its own process...
write(1, "8", 1)  # output to stdout
exit(0)           # terminate
wait4(...)        # parent reaps child
```

That's at minimum 5 system calls vs **zero** for `$(( ))` — 5x more for a single addition.

### Memory Layout

Arithmetic in bash uses the shell's internal 64-bit signed integer (`long long` on modern systems). The expression is parsed into a small AST allocated on the heap. Temporary values live in registers or on the C stack. Variable assignments update the shell's variable dictionary (hash table).

### Performance Implications

- `$(( ))` and `(( ))` are the **fastest** way to do integer math in bash
- `bc` is fast for arbitrary precision but fork overhead dominates for small ops
- A loop doing 10,000 additions:
  - `$(( ))`: ~0.02s (in-process)
  - `bc`: ~2.5s (10,000 forks)
  - `expr`: ~8s (10,000 forks + execs)
  - **bash is ~100-400x faster** for large repetition

### Equivalent C Implementation

```c
// Bash does this internally for: x=$(( a + b * c ))
long a = lookup_variable("a")->value;  // defaults to 0 if unset
long b = lookup_variable("b")->value;
long c = lookup_variable("c")->value;
long result = a + b * c;  // C operator precedence
set_variable("x", result);
```

### Equivalent Python Implementation

```python
# Bash: x=$(( a + b * c ))
a = int(os.environ.get('a', '0'))
b = int(os.environ.get('b', '0'))
c = int(os.environ.get('c', '0'))
x = a + b * c
```

## Core Examples (12 minimum)

### Example 1: Basic addition, subtraction, multiplication

```bash
$ x=10 y=3
$ echo $((x + y))
13
$ echo $((x - y))
7
$ echo $((x * y))
30
```

**Step-by-step**: 1. `x=10` assigns 10 to x. 2. `y=3` assigns 3 to y. 3. `$((x + y))` reads x (10), reads y (3), adds them (13), expands to "13". 4. `echo` prints it.

**What if** `x` is unset? `$(( x + 5 ))` → 5 (unset = 0). **What if** `x=abc`? Syntax error.

### Example 2: Division and Modulo

```bash
$ echo $((17 / 5))
3
$ echo $((17 % 5))
2
$ echo $(( -17 / 5 ))
-3   # truncates toward zero
```

**Step-by-step**: 17/5 = 3 remainder 2. Bash truncates the fraction. For negative: `(-17)/5 = -3` (rounded toward zero).

**What if** `0` is the divisor? `$((5 / 0))` → bash error: division by 0.

### Example 3: Exponentiation

```bash
$ echo $((2 ** 10))
1024
$ echo $((2 ** 2 ** 3))
256    # right-associative: 2**(2**3) = 2**8 = 256
```

**Step-by-step**: `2 ** 2 ** 3` is parsed as `2 ** (2 ** 3)` because `**` is right-associative.

**What if** exponent is negative? → 0. Overflow? Wraps silently.

### Example 4: Bitwise operations

```bash
$ echo $((8 << 2))   # 8 * 2^2 = 32
32
$ echo $((7 & 3))    # 111 & 011 = 011 = 3
3
$ echo $((7 | 3))    # 111 | 011 = 111 = 7
7
$ echo $((~5))       # bitwise NOT → -(5+1) = -6
-6
```

**What if** shifting beyond bit width? Count is masked (count % 64 on 64-bit).

### Example 5: Increment and decrement

```bash
$ i=5
$ echo $((i++))     # post-increment: prints 5, then i becomes 6
5
$ echo $i
6
$ echo $((++i))     # pre-increment: i becomes 7, prints 7
7
```

**Step-by-step**: Post-increment: evaluate with current value, then increment. Pre-increment: increment first, then use.

### Example 6: Compound assignment operators

```bash
$ n=100
$ ((n %= 7)) && echo $n
2
$ ((n += 50))
$ echo $n
52
```

### Example 7: Arithmetic in for loops

```bash
$ for ((i=0; i<5; i++)); do echo -n "$i "; done; echo
0 1 2 3 4

$ for ((i=0, j=10; i<j; i++, j--)); do echo "i=$i j=$j"; done
```

### Example 8: The ternary operator

```bash
$ x=5
$ echo $(( x > 3 ? 100 : 200 ))
100
$ echo $(( x > 0 ? (y > 0 ? 1 : 2) : 3 ))
2
```

### Example 9: Floating point with bc

```bash
$ echo "scale=4; 22 / 7" | bc
3.1428
$ echo "scale=10; sqrt(2)" | bc -l
1.4142135623
$ echo "ibase=16; obase=10; FF" | bc
255
```

**What if** division by zero in `bc -l`? Runtime error. Without `-l`, returns 0.

### Example 10: Base conversion with bc

```bash
$ echo "obase=2; 42" | bc
101010
$ echo "obase=16; 255" | bc
FF
```

**Trap**: Set `obase` before `ibase`, or `obase=10` in hex means base 16!

### Example 11: Arithmetic in while loops

```bash
$ i=10
$ while (( i-- )); do echo -n "$i "; done; echo
9 8 7 6 5 4 3 2 1 0
```

### Example 12: Summing command output

```bash
$ total=0
$ for n in $(seq 1 100); do (( total += n )); done
$ echo $total
5050

$ echo $(( $(seq -s+ 1 100) ))   # Σ 1..100 = 5050
5050
```

## Real-World Use Cases

### FOR the OS

- **Monitoring scripts**: Calculate CPU usage from `/proc/stat` counters
- **Log rotation**: Compute file ages, decide when to rotate
- **Resource limits**: Memory percentages from `/proc/meminfo`
- **Timers**: `end=$SECONDS; echo $((end - start))`
- **Progress bars**: `percent=$((done * 100 / total))`

### WITH the OS

- **`df` + arithmetic**: Disk usage percentages
- **`free` + arithmetic**: Memory usage
- **`date +%s` + arithmetic**: Unix timestamp math
- **`wc -l` + arithmetic**: Count thresholds

### AGAINST the OS (sysadmin)

- **Fork-bomb detection**: `(( proc_count > MAX )) && kill -9 $bad_pid`
- **Threshold alerts**: `(( load_avg > 10 )) && mail -s "High load" admin`
- **Timeout enforcement**: `(( SECONDS_ELAPSED > TIMEOUT )) && kill $pid`

### FOR DEFENSE (hardening)

- **Input validation**: Reject non-integer input before arithmetic
- **Bounds checking**: `(( index >= 0 && index < max ))` before array access
- **Resource pressure detection**: `(( free_mb < 100 ))` and trigger cleanup

## Memory Aids

- **"Double-dollar makes the calculation happen"**: `$(( expr ))` produces a value; `(( expr ))` produces an exit code
- **`$(( ))` is for output; `(( ))` is for decisions**
- **"Unset is zero, zero is false"**: Debugging why `(( unset_var ))` is false (exit 1)
- **"Colon with bc"**: `echo "scale=N; expr" | bc`
- **"Modulo gives remainder, not fraction"**: `a % b` = what's left
- **let = l-e-t = "Let's just not use this"** — `(( ))` is clearer

## Trap Vault (12 traps)

### Trap 1: `$(( 0 ))` exits 0 (SUCCESS), ruining your if condition

```bash
# BAD — always runs
if $(( count > 5 )); then echo "Greater"; fi

# FIX: Use (( count > 5 )) without dollar sign
if (( count > 5 )); then echo "Greater"; fi
```

### Trap 2: Integer division truncation

```bash
echo $((5 / 3))   # → 1, not 1.666
# FIX: Use bc
echo "scale=3; 5 / 3" | bc
```

### Trap 3: Division by zero

```bash
x=0; echo $((5 / x))  # bash: division by 0
# FIX: Guard with (( x != 0 ))
```

### Trap 4: Octal confusion

```bash
month_day=08
echo $(( month_day + 1 ))  # error: 08 invalid octal
# FIX: echo $(( 10#$month_day + 1 ))
```

### Trap 5: Floating point silently truncated

```bash
x=5.5; echo $(( x + 1 ))  # syntax error
# FIX: Use bc
```

### Trap 6: Variable name shadows command

```bash
true=1
echo $(( true + 1 ))  # 2, treats true as variable not command
# FIX: Avoid variable names that shadow commands
```

### Trap 7: Assignment inside (( )) vs comparison

```bash
if (( a = b )); then  # ASSIGNS, not compares!
# FIX: Use == for comparison
if (( a == b )); then echo "equal"; fi
```

### Trap 8: let requires quoting or no spaces

```bash
let x = 2 + 3  # error
# FIX: let x=2+3  OR  let "x = 2 + 3"
```

### Trap 9: bc needs piped input

```bash
bc 5 / 3  # wrong — starts interactive
# FIX: echo "5/3" | bc
```

### Trap 10: Exponentiation is not POSIX

```bash
# ** is Bash/ksh extension, not in POSIX
echo $((2 ** 10))  # works in bash, not in sh
# FIX for POSIX: Use a loop or bc
```

### Trap 11: $ inside $(( ))

```bash
# $ is optional for vars inside $(( ))
sum=$(( $a + $b ))  # works but redundant
sum=$(( a + b ))    # preferred
# BUT $1, $2 positional params DO need $:
sum=$(( $1 + $2 ))
```

### Trap 12: Scale in bc is decimal digits, not significant figures

```bash
echo "scale=4; 0.01 * 0.01" | bc  # → .0001 (leading 0 dropped)
# FIX: printf "%.4f\n" $(echo "0.01 * 0.01" | bc)
```

## See It In The Wild

### Daily Encounters

- **Ctrl+R history search**: `(( HISTCMD++ ))`
- **PS1 prompt**: `SECONDS` arithmetic for elapsed time
- **make -j**: `$(nproc)` parallelization
- **Docker**: CPU quota from `cpu_period` / `cpu_quota`

### Exploration Exercises

1. `echo $(( RANDOM % 100 ))` — RANDOM changes each read
2. `echo $(( 0xdeadbeef ))` — hex to decimal
3. `time for i in {1..10000}; do x=$((i*i)); done` vs `time for i in {1..10000}; do x=$(expr $i \* $i); done`
4. `/proc/stat` CPU tick math

## Check Your Understanding (7 questions)

1. **True or False**: `(( 0 ))` exits with status 0 (success). Explain.
2. Write a one-liner using `$(( ))` to compute the sum of all odd numbers from 1 to 99.
3. Why does `echo $((08 + 1))` produce an error, and how do you fix it?
4. What's the difference between `$(( i++ ))` in an echo statement vs `(( i++ ))` alone?
5. How do you compute `5 / 3` to get `1.6667` using `bc`?
6. What does `echo $(( 2 ** 3 ** 2 ))` print, and why?
7. If `x=10` and `(( x *= 2 ))`, what's the exit code and the new value of `x`?

## Supplementary Deep Dive: Advanced Arithmetic Patterns

### Using printf for formatted arithmetic output

```bash
$ printf "Hex: %x\n" $((255))
Hex: ff
$ printf "Octal: %o\n" $((255))
Octal: 377
$ printf "Float: %.2f\n" $(echo "scale=2; 22/7" | bc)
Float: 3.14
```

### Random number generation with $RANDOM

The `RANDOM` variable generates a pseudo-random integer between 0 and 32767 each time it's read:

```bash
$ echo $((RANDOM % 100))       # 0-99
42
$ echo $((RANDOM % 50 + 1))    # 1-50
17
$ echo $((RANDOM * 1000 / 32767))  # 0-1000 (scaling)
356
```

**What if** you need better randomness? Use `/dev/urandom`:
```bash
$ od -An -N2 -tu2 /dev/urandom | tr -d ' '
52341
```

### Arithmetic with SECONDS for timing

`SECONDS` is a special bash variable that auto-increments every second:

```bash
$ start=$SECONDS
$ sleep 2.5
$ echo "Elapsed: $((SECONDS - start)) seconds"
Elapsed: 3 seconds   # integer truncation!
```

For sub-second timing, use `$EPOCHREALTIME` (bash 5.0+):
```bash
$ start=$EPOCHREALTIME
$ sleep 1
$ end=$EPOCHREALTIME
$ echo "Scale=3; $end - $start" | bc
1.002
```

### Base conversion in pure bash

```bash
$ echo $((2#1010))              # binary to decimal
10
$ echo $((16#FF))               # hex to decimal
255
$ echo $((8#777))               # octal to decimal
511
$ echo $((36#ZZ))               # base-36 to decimal
1295
```

### The comma operator for multiple expressions

```bash
$ echo $((x=5, y=10, x+y))      # evaluates to 15, sets x=5, y=10
15
$ echo $x
5
$ echo $((a++, b--, a+b))
```

### Arithmetic with bit flags

Use bitwise OR to combine flags, AND to test:

```bash
$ READ=1 WRITE=2 EXECUTE=4
$ perms=$((READ | WRITE))       # 3 = read+write
$ echo $(( perms & READ ))      # 1 — read permission set
1
$ echo $(( perms & EXECUTE ))   # 0 — exec not set
0
```

### Arithmetic exit code tricks

```bash
$ (( 1 )) && echo "truthy"      # truthy — 1 is non-zero
truthy
$ (( 0 )) || echo "falsy"       # falsy — 0 is zero
falsy
$ (( 5 - 5 )) || echo "zero"    # result is 0, so falsy
zero
```

### Checking if a number is even

```bash
$ is_even() { (( $1 % 2 == 0 )); }
$ is_even 4 && echo "even" || echo "odd"
even
$ is_even 7 && echo "even" || echo "odd"
odd
```

## Supplementary: Advanced Arithmetic Edge Cases

### Overflow and large numbers

```bash
$ echo $(( 2**63 - 1 ))    # max signed 64-bit integer
9223372036854775807
$ echo $(( 2**63 ))        # wraps negative
-9223372036854775808
```

### Arithmetic with base conversion

```bash
$ echo $(( 16#FF ))        # hex → decimal
255
$ echo $(( 8#777 ))        # octal → decimal
511
$ echo $(( 2#1010 ))       # binary → decimal
10
$ echo $(( 36#ZZ ))        # base-36 → decimal
1295
```

### Truth values and exit codes

```bash
$ (( 0 )) && echo "yes" || echo "no"    # 0 = false
no
$ (( 1 )) && echo "yes" || echo "no"    # nonzero = true
yes
```
