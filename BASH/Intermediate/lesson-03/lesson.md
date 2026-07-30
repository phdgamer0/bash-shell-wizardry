# Lesson 3: Parameter Expansion — Slicing

## History & Origins

String slicing via parameter expansion has its roots in the **Korn shell (ksh88)**, which introduced `${#var}`, `${var:offset:length}`, and the `#`/`%` pattern removal operators. David Korn designed these to handle the most common string operations — measuring, extracting, and stripping — without forking external tools.

Bash adopted them in **bash 2.0** (1996). The pattern matching operators (`#`, `##`, `%`, `%%`) use **glob-style patterns** (not regex), keeping them consistent with filename expansion.

The key insight: most string manipulation in shell scripts involves **prefix/suffix removal** (stripping directory paths, file extensions, protocol prefixes). Glob patterns are sufficient for these tasks and easier to read than regex.

These operators are specified in **POSIX.1-2001** (for `#`, `##`, `%`, `%%`, `${#var}`) and extended in **bash** (for `${var:offset:length}` with negative offsets).

## Syntax Reference

| Expression | What it does | POSIX? | Example |
|------------|-------------|--------|---------|
| `${#var}` | Length of var in characters | Yes | `str=abc; echo ${#str}` → 3 |
| `${var:offset}` | Substring from offset to end | Bash | `str=abcde; echo ${str:2}` → cde |
| `${var:offset:length}` | Substring: length chars from offset | Bash | `str=abcde; echo ${str:1:3}` → bcd |
| `${var: -offset}` | Substring from end (negative) | Bash | `str=abcde; echo ${str: -2}` → de |
| `${var:(-offset)}` | Same, using parens | Bash | `str=abcde; echo ${str:(-2)}` → de |
| `${var#pattern}` | Remove shortest prefix match | Yes | `str=a.b.c; echo ${str#*.}` → b.c |
| `${var##pattern}` | Remove longest (greedy) prefix | Yes | `str=a.b.c; echo ${str##*.}` → c |
| `${var%pattern}` | Remove shortest suffix match | Yes | `str=a.b.c; echo ${str%.*}` → a.b |
| `${var%%pattern}` | Remove longest (greedy) suffix | Yes | `str=a.b.c; echo ${str%%.*}` → a |

### Pattern Rules for `#`, `##`, `%`, `%%`

Patterns use **glob-style** matching: `*` = any sequence, `?` = single char, `[abc]` = set, `[!abc]` = complement.

**These are NOT regex**. `.` matches literal dot. `*` is greedy but `#` = "shortest", `##` = "longest".

### Negative Offset Rules

The space or parentheses is **mandatory**:
- `${str:-2}` = default value (if `str` is unset, use "2")
- `${str: -2}` = last 2 characters (negative offset)
- `${str:(-2)}` = same, parentheses disambiguate

This ambiguity is a common bug source.

### Edge Cases

- **Empty string**: `${#""}` → 0
- **Offset beyond length**: `${str:999}` → ""
- **Length beyond remaining**: `${str:3:999}` → characters from 3 to end
- **Negative offset larger than length**: `${str: -999}` → entire string
- **No match for `#`/`%`**: `${str#nomatch}` → original string unchanged
- **`$@` and `$*`**: `${@:2:3}` gives elements 2-4 (array indexing, not character slicing)
- **Arrays**: `${arr[@]:1:2}` slices the array

## Under the Hood

### What bash does for `${#var}`

1. Look up `var` in hash table.
2. Get string value.
3. Call `strlen(value)` — O(n) character count.

### What bash does for `${var#pattern}`

1. Look up `var`, get string.
2. Parse the glob pattern.
3. Use shell's pattern matching engine (`strmatch`/`patmatch`) to find the shortest/longest match at start/end.
4. `memmove` remaining part or allocate new string.

### System Calls

**Zero system calls** for all slicing operations. Everything in-process.

Compare:
```bash
# Bash — 0 syscalls
ext="${file##*.}"

# External — 3+ syscalls
ext=$(echo "$file" | sed 's/.*\.//')
```

### Memory

When stripping a prefix, bash allocates a new temporary string. The original variable value isn't modified. The new string is freed after the command completes (unless assigned back).

### Performance

- `${#var}` is O(n) — calls `strlen` each time
- Substring extraction allocates new string
- `#`/`##`/`%`/`%%` use same engine as globs; simple patterns are fast
- For typical usage, parameter expansion is 10-100x faster than external tools

### Equivalent C

```c
// ${file##*.}
char *file = lookup_variable("file")->value;
char *dot = strrchr(file, '.');
char *ext = dot ? dot + 1 : file;
```

```c
// ${path%/*}
char *path = lookup_variable("path")->value;
char *slash = strrchr(path, '/');
if (slash) {
    size_t len = slash - path;
    char *dir = xmalloc(len + 1);
    memcpy(dir, path, len);
    dir[len] = '\0';
}
```

### Equivalent Python

```python
# ${file##*.}
ext = file.rsplit('.', 1)[-1] if '.' in file else file

# ${path%/*}
dir = path.rsplit('/', 1)[0] if '/' in path else '.'
```

## Core Examples (12 minimum)

### Example 1: String length

```bash
$ str="Hello, World!"
$ echo "${#str}"
13
$ empty=""
$ echo "${#empty}"
0
```

**What if** the variable is unset? → 0. **Array?** `${#arr}` = length of first element. `${#arr[@]}` = element count.

### Example 2: Substring from offset

```bash
$ str="abcdefgh"
$ echo "${str:2}"
cdefgh
$ echo "${str:100}"
# empty
```

### Example 3: Substring with length

```bash
$ str="abcdefgh"
$ echo "${str:2:3}"
cde
$ echo "${str:6:10}"
gh    # length > available, clamped to end
$ echo "${str: -4:2}"
ef
```

### Example 4: Negative offset (critical: the space!)

```bash
$ str="hello.txt"
$ echo "${str: -4}"    # SPACE → last 4 chars
.txt
$ echo "${str:(-4)}"   # parentheses work too
.txt
$ echo "${str:-4}"     # NO space → default value syntax
hello.txt              # (str is set, so prints str)
```

**This is one of the most insidious bash bugs.** No error, no warning.

### Example 5: Strip shortest prefix with `#`

```bash
$ url="https://www.example.com/page"
$ echo "${url#https://}"
www.example.com/page
$ echo "${url#*/}"
/http://www.example.com/page   # shortest match of */
```

**Step-by-step for `${url#*/}`**: `#` = remove shortest prefix matching `*/`. Shortest match is first `/` → removes "https:" → leaves "/www.example.com/page".

### Example 6: Strip longest prefix with `##`

```bash
$ url="https://www.example.com/page"
$ echo "${url##*/}"
page    # greedy — everything up to last /
$ path="/home/user/docs/report.pdf"
$ echo "${path##*/}"
report.pdf
```

**What if** no slash? Returns the whole string unchanged.

### Example 7: Strip shortest suffix with `%`

```bash
$ file="archive.tar.gz"
$ echo "${file%.gz}"
archive.tar
$ echo "${file%.*}"
archive.tar    # shortest .* = last dot onwards
```

### Example 8: Strip longest suffix with `%%`

```bash
$ file="archive.tar.gz"
$ echo "${file%%.*}"
archive    # greedy — removes from first dot onwards
$ url="https://example.com/page?q=v#frag"
$ echo "${url%%\#*}"
https://example.com/page?q=v
```

### Example 9: Extract file path components

```bash
$ path="/home/user/docs/report.pdf"
$ echo "Filename: ${path##*/}"
report.pdf
$ echo "Directory: ${path%/*}"
/home/user/docs
$ echo "Extension: ${path##*.}"
pdf
$ echo "Basename: ${file%.*}"
report
```

This is the **killer use case**.

### Example 10: Extract URL components

```bash
$ url="https://www.example.com:8080/path/to/page?q=search#section"
$ echo "Protocol: ${url%%://*}"
https
$ rest="${url#*://}"
$ echo "Host+Port: ${rest%%/*}"
www.example.com:8080
$ echo "Path: ${rest#*/}"
path/to/page?q=search#section
```

### Example 11: Array slicing too!

```bash
$ nums=(zero one two three four five)
$ echo "${nums[@]:3:4}"
three four five
$ echo "${nums[@]: -3}"
three four five
```

### Example 12: `${#arr}` vs `${#arr[@]}`

```bash
$ arr=("hello" "world")
$ echo "${#arr}"       # length of first element = 5
5
$ echo "${#arr[@]}"    # number of elements = 2
2
$ echo "${#arr[1]}"    # length of second element = 5
5
```

### Example 13: Stripping the last character

```bash
$ str="hello!"
$ echo "${str%?}"
hello
$ echo "${str%??}"
hel
```

### Example 14: Pattern removal with arrays

```bash
$ paths=("/home/user/file.txt" "/var/log/syslog" "/tmp/test.sh")
$ for p in "${paths[@]}"; do echo "${p##*/}"; done
file.txt
syslog
test.sh
```

## Real-World Use Cases

### FOR the OS

- Parsing `/proc` files with `#` and `%`
- Log rotation: `basename="${logfile##*/}"`
- Config parsing: key=`${line%%=*}`, value=`${line#*=}`
- Username extraction from `/etc/passwd`: `${line%%:*}`
- Remove trailing slash: `dir="${dir%/}"`

### WITH the OS

- `basename`/`dirname` replacement: `${path##*/}`, `${path%/*}` (no fork)
- Custom `cut`: `${line#*delim}`
- String length checks: `${#str} -gt 80`

### AGAINST the OS (security)

- Path traversal prevention: `safe="${path##/}"`
- Extension whitelisting: `ext="${file##*.}"; case $ext in txt|log) allow;;`
- Detect hidden chars: Compare `${#file}` to expected length

### FOR DEFENSE

- Input sanitization: `safe="${file//[^a-zA-Z0-9._-]/}"`
- Chroot safety: `real_path="${path%%/../*}"`
- Reject suspicious paths: `if [[ $path == *..* ]]; then`

## Memory Aids

- **`#` is in the front** → `${var#pattern}` removes from the *front* (prefix)
- **`%` is in the back** → `${var%pattern}` removes from the *back* (suffix)
- **Double `##` = "more" = greedy** (removes more). Single `#` = "short"
- **Same for `%%` vs `%`**: greedy suffix vs short suffix
- **Negative offset needs space**: "Space before minus = slice from the bottom." No space = default value.
- **`${#var}`**: "The # asks 'how many characters?'"

## Trap Vault (12 traps)

### Trap 1: Negative offset without space → default value syntax

```bash
str="hello"
echo "${str:-2}"   # prints "hello", not "lo"
# FIX: echo "${str: -2}"
```

### Trap 2: `#` and `%` use glob patterns, not regex

```bash
str="file.txt"
echo "${str%.*}"   # works (.* matches dot+chars)
# But | is literal pipe, not regex alternation
# FIX: Use [[ =~ ]] for regex
```

### Trap 3: `##*/` vs `#*/` confusion

```bash
path="/home/user/docs/report.pdf"
echo "${path#*/}"    # "home/user/docs/report.pdf" (first / removed)
echo "${path##*/}"   # "report.pdf" (greedy)
```

### Trap 4: `%` vs `%%` for extensions

```bash
file="archive.tar.gz"
echo "${file%.*}"   # "archive.tar"
echo "${file%%.*}"  # "archive"
```

### Trap 5: `#` and `%` are anchored — they don't search

```bash
str="abc"
echo "${str#b}"   # "abc" (no match — b is not at start)
```

### Trap 6: Escape characters in patterns

```bash
str="file[1].txt"
echo "${str%.txt}"  # works for literal .txt
```

### Trap 7: Array/`$@` slicing is element-based, not character-based

```bash
set -- "my file.txt" "second"
echo "${@:1:1}"    # "my file.txt" (first positional param)
```

### Trap 8: `$*` vs `$@` with slicing

```bash
set -- a b c d
echo "${*:2:2}"   # "b c" (IFS first char separates)
echo "${@:2:2}"   # "b c" (separate words)
```

### Trap 9: Empty values after stripping

```bash
path="/"
filename="${path##*/}"   # "" (empty!)
# FIX: filename="${filename:-root}"
```

### Trap 10: extglob changes `#`/`%` pattern interpretation

```bash
shopt -s extglob
str="abc123"
echo "${str#+(a)}"   # + is extglob for "one or more"
```

### Trap 11: Multibyte/Unicode character counting

```bash
# In bash 4.3+ with UTF-8 locale, é is 1 character
# In C locale, é may be 2 bytes
```

### Trap 12: `#` and `%` are not invertible

```bash
str="a.b.c"
echo "${str#*.}"   # "b.c"
echo "${str%.*}"   # "a.b"
# To get "b": middle="${str#*.}"; middle="${middle%.*}"
```

## See It In The Wild

- Git hooks: `${GIT_DIR##*/}` for repo name
- .bashrc: `HOSTNAME_SHORT="${HOSTNAME%%.*}"`
- Dockerfiles: `IMAGE_NAME="${PROJECT##*/}"`
- Init scripts: `DAEMON="${0##*/}"`

### Exploration

1. `${#-}` — length of shell options string
2. `${BASH_VERSION%.*}` — major.minor version
3. `${PATH//:/$'\n'}" | head -5 — PATH entries
4. Set `x="a:b:c:d"`, try `${x#*:}` vs `${x##*:}`

## Check Your Understanding (7 questions)

1. **What's the output?**: `path="/a/b/c/d.txt"; echo "${path#/*/}"; echo "${path##/*/}"`
2. **How do you extract the last 3 characters of a string?**
3. **Difference between `${file%.*}` and `${file%%.*}` when `file="config.backup.tar.gz"`?**
4. **Why does `${str:-3}` not give the last 3 characters? How to fix?**
5. **What does `${var#pattern}` return if pattern doesn't match the start?**
6. **How to get the first 5 characters?**
7. **Extract year from `2025-07-30` using only parameter expansion.**

## Supplementary Deep Dive: Advanced Slicing Patterns

### Extracting file components in one line

```bash
$ path="/home/user/docs/report.pdf"
$ # Everything in one expression (not recommended for readability, but fun)
$ echo "dir=${path%/*} file=${path##*/} base=${file%.*} ext=${file##*.}"
dir=/home/user/docs file=report.pdf base=report ext=pdf
```

### Removing a known prefix/suffix with fixed string

```bash
$ prefix="/home/user"
$ str="/home/user/docs/file.txt"
$ echo "${str#$prefix}"       # remove prefix if match → /docs/file.txt
/docs/file.txt
$ suffix=".txt"
$ echo "${str%$suffix}"       # remove suffix if match → /home/user/docs/file
/home/user/docs/file
```

### Getting the Nth character

```bash
$ str="abcdefghij"
$ n=4
$ echo "${str:$((n-1)):1}"    # 4th character
d
$ echo "${str:0:1}"           # 1st character
a
$ echo "${str: -1}"           # last character
j
```

### Getting everything except the last character

```bash
$ str="hello"
$ echo "${str%?}"             # remove last char
hell
```

### URL parsing edge case — no port

```bash
$ url="https://example.com/path"
$ rest="${url#*://}"
$ echo "${rest%%/*}"
example.com
# vs with port:
$ url="https://example.com:8080/path"
$ echo "${rest%%/*}"
example.com:8080
```

### Extracting between two delimiters

```bash
$ str="prefix:middle:suffix"
$ # Extract "middle"
$ temp="${str#*:}"           # remove prefix → middle:suffix
$ echo "${temp%:*}"          # remove suffix → middle
middle

$ # One-liner:
$ echo "${str#*:}" ; echo "${str%:*}"
# That prints two lines. Combine:
$ between() {
>   local str="$1" left="$2" right="$3"
>   local tmp="${str#*$left}"
>   echo "${tmp%$right*}"
> }
$ between "a{b}c" "{" "}"
b
```

### Working with IFS and slicing

```bash
$ IFS=: read -ra fields <<< "a:b:c:d"
$ echo "${fields[1]}"        # b (by field splitting)
$ # Same using only parameter expansion:
$ str="a:b:c:d"
$ echo "${str#*:}"           # b:c:d
$ echo "${str%:*}"           # a:b:c
$ echo "${str#*:}" | { read first; echo "${first%:*}"; }
# That forks. Better:
$ rest="${str#*:}"
$ echo "${rest%:*}"          # b:c
```

### Combining ${#} with slicing for validation

```bash
$ validate_phone() {
>   local phone="$1"
>   local cleaned="${phone//[^0-9]/}"
>   if (( ${#cleaned} != 10 )); then
>     echo "Invalid: must have 10 digits"
>     return 1
>   fi
>   echo "Area: ${cleaned:0:3}"
>   echo "Prefix: ${cleaned:3:3}"
>   echo "Line: ${cleaned: -4}"
> }
$ validate_phone "(555) 123-4567"
Area: 555
Prefix: 123
Line: 4567
```

### Removing trailing slash

```bash
$ dir="/home/user/docs/"
$ echo "${dir%/}"
/home/user/docs
$ dir="/"
$ echo "${dir%/}"
# empty— careful with root!
$ echo "${dir:-/}"
/
```

## Supplementary Deep Dive: Advanced Slicing Patterns

### Negative length values

```bash
$ str="abcdefgh"
$ echo "${str:3:-1}"    # from index 3 to 1 from end
defg
$ echo "${str:1:-2}"    # from index 1 to 2 from end
bcdef
```

### Slicing with computed offsets

```bash
$ offset=3
$ length=2
$ str="Hello World"
$ echo "${str:$offset:$length}"
lo
$ # Computed negative offset
$ end_offset=5
$ echo "${str:0: -end_offset}"  # syntax error
$ echo "${str:0: -5}"           # works with literal
Hello
```

### Using slicing for string truncation

```bash
$ # Truncate to first N chars
$ truncate() {
>   local max=$1
>   shift
>   local str="$*"
>   echo "${str:0:max}"
> }
$ truncate 10 "This is a very long string"
This is a
```

### Slicing with parameter expansion for file names

```bash
$ # Extract version from filename
$ filename="app-v2.3.1-beta.tar.gz"
$ # Find position of "v" and extract version
$ version="${filename#*v}"
$ version="${version%%-*}"
$ echo "$version"
2.3.1
```

### Finding substring position (bash 4.2+)

```bash
$ str="Hello World"
$ pattern="World"
$ # Find position using ${var%%pattern} trickery
$ prefix="${str%%$pattern*}"
$ echo "Pattern starts at: ${#prefix}"
6
```

### Slice-based padding

```bash
$ # Left-pad to 10 chars
$ pad_left() {
>   local str="$1"
>   local len="$2"
>   local pad="${3:- }"
>   while (( ${#str} < len )); do str="${pad}${str}"; done
>   echo "$str"
> }
$ pad_left "42" 5 "0"
00042
```

### Extracting delimited substrings with slicing

```bash
$ # Domain from email
$ email="user@example.com"
$ at_pos="${email%%@*}"
$ at_pos="${#at_pos}"        # position of @
$ domain="${email:$((at_pos+1))}"
$ echo "$domain"
example.com
```

### Auto-truncate with ellipsis

```bash
$ truncate_ellipsis() {
>   local str="$1"
>   local max="$2"
>   if (( ${#str} > max )); then
>     echo "${str:0:max-3}..."
>   else
>     echo "$str"
>   fi
> }
$ truncate_ellipsis "Hello World" 8
Hello...
```
