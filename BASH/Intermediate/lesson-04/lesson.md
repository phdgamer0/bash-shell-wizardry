# Lesson 4: Parameter Expansion — Replace & Case

## History & Origins

The pattern replacement `${var/old/new}` arrived in **bash 2.0** (1996), inspired by ksh93's pattern replacement operators. Case modification (`${var,,}`, `${var^^}`, etc.) came much later — **bash 4.0** (2009). Before that, the only way to change case was to fork `tr`, `sed`, or `awk`.

The pattern anchoring (`#` prefix, `%` suffix) is unique to bash — not found in ksh or POSIX sh. It gives positional control over replacements without needing regex anchors.

## Syntax Reference

| Expression | Effect | Bash Version |
|------------|--------|--------------|
| `${var/pattern/replacement}` | Replace **first** match of pattern | 2.0+ |
| `${var//pattern/replacement}` | Replace **all** matches | 2.0+ |
| `${var/#pattern/replacement}` | Replace if pattern matches **start** | 2.0+ |
| `${var/%pattern/replacement}` | Replace if pattern matches **end** | 2.0+ |
| `${var,}` / `${var,,}` | Lowercase first / all chars | 4.0+ |
| `${var^}` / `${var^^}` | Uppercase first / all chars | 4.0+ |
| `${var~}` / `${var~~}` | Swap case first / all chars | 4.0+ |

### Pattern Rules

Patterns use **glob-style** matching, NOT regex:
- `*` matches any sequence, `?` matches a single char
- `[abc]` matches set, `[a-z]` matches range, `[!abc]` matches complement
- `.(pattern)` requires `shopt -s extglob`

### Replacement String

The replacement undergoes parameter expansion and command substitution but NOT word splitting or pathname expansion:

```bash
$ var="hello world"
$ echo "${var/hello/$(echo there)}"
there world
```

To include a literal `/` in replacement, use a variable: `slash="/"; echo "${var/$slash/-}"`.

### Anchored Replacement

`${var/#pattern/rep}` only replaces if pattern matches at the **beginning** (like `^` in regex).
`${var/%pattern/rep}` only replaces if pattern matches at the **end** (like `$` in regex).

```bash
$ str="abcabc"
$ echo "${str/#abc/XYZ}"   # "XYZabc"
$ echo "${str/%abc/XYZ}"   # "abcXYZ"
```

### Case Modification Details

| Operator | On "john DOE" | On "" | On unset |
|----------|--------------|-------|----------|
| `${name,}` | john DOE | "" | "" |
| `${name,,}` | john doe | "" | "" |
| `${name^}` | John DOE | "" | "" |
| `${name^^}` | JOHN DOE | "" | "" |
| `${name~}` | JOHN doe | "" | "" |
| `${name~~}` | JOHN doe | "" | "" |

- `,` and `^` affect only the first character. `,,` and `^^` affect all.
- These work on arrays too: `${arr[@],,}` lowercases all elements (bash 4.4+)
- Locale-dependent: Turkish locale has special `I`/`i` rules

## Under the Hood

### For `${var//old/new}`

1. Look up `var`, get string value.
2. Parse the pattern `old` — convert glob to internal representation.
3. Parse replacement `new` — expand `$sub`/`$(cmd)` inside.
4. Scan left-to-right using pattern matching engine.
5. For each match position: calculate match length; for `//` skip past and continue; for `/` stop after first; for `/#` only check position 0; for `/%` only check end.
6. Build result string (allocate, copy un-matched portions + replacements).

### For `${var,,}`

1. Look up `var` value.
2. Iterate each character, call `tolower()`/`toupper()`/`toggle_case()`.
3. Build new string.
4. Return.

### System Calls

**Zero system calls** for all replace and case operations. Everything in-process.

Compare:
```bash
# Bash — 0 syscalls
lower="${var,,}"

# External — 3+ syscalls
lower=$(echo "$var" | tr '[:upper:]' '[:lower:]')
```

### Performance

For large strings (1M chars), external tools using optimized C may be faster:
- `${var,,}` on 1M: ~10ms
- `tr` on 1M: ~5ms (fork overhead amortized)

For small strings, bash is always faster (no fork overhead).

## Core Examples (12 minimum)

### Example 1: Replace first match

```bash
$ str="The cat sat on the cat mat"
$ echo "${str/cat/dog}"
The dog sat on the cat mat
```

**Step-by-step**: Scans left to right. First "cat" at position 4 → replaced. Only the first one changes.

### Example 2: Replace all matches

```bash
$ str="The cat sat on the cat mat"
$ echo "${str//cat/dog}"
The dog sat on the dog mat
```

### Example 3: Anchored replacement

```bash
$ url="http://example.com/http"
$ echo "${url/#http/https}"
https://example.com/http
$ echo "${url/%http/https}"
http://example.com/https
```

### Example 4: Case conversion

```bash
$ name="john DOE"
$ echo "${name,,}"
john doe
$ echo "${name^^}"
JOHN DOE
$ echo "${name^}"
John DOE
```

### Example 5: URL sanitizer pipeline

```bash
$ raw="  User Input With SPACES  "
$ clean="${raw,,}"              # lowercase
$ clean="${clean// /_}"         # spaces to underscores
$ clean="${clean#_}"            # strip leading
$ clean="${clean%_}"            # strip trailing
$ echo "$clean"
user_input_with_spaces
```

### Example 6: File extension normalization

```bash
$ for f in image.JPG photo.jpeg; do
>   base="${f%.*}"
>   ext="${f##*.}"
>   mv "$f" "${base}.${ext,,}"
> done
```

### Example 7: Remove matched patterns (empty replacement)

```bash
$ phone="(555) 123-4567"
$ echo "${phone//[!0-9]/}"    # remove all non-digits
5551234567
```

### Example 8: Case-insensitive URL normalization

```bash
$ url="HTTP://EXAMPLE.COM"
$ lower="${url,,}"
$ normalized="${lower/#http/https}"
https://example.com
```

### Example 9: Password masking

```bash
$ password="hunter2"
$ echo "${password//?/*}"
*******
```

### Example 10: First-char case toggle

```bash
$ sentence="the quick brown fox"
$ echo "${sentence^}"
The quick brown fox
$ sentence="THE QUICK BROWN FOX"
$ echo "${sentence,}"
tHE QUICK BROWN FOX
```

### Example 11: Replacement with variable pattern

```bash
$ old="abc"
$ new="XYZ"
$ str="abcdefabc"
$ echo "${str//$old/$new}"
XYZdefXYZ
```

### Example 12: Version stripping with `#` anchor

```bash
$ version="v1.2.3-beta"
$ echo "${version/#v/}"
1.2.3-beta
```

### Example 13: Case-insensitive comparison

```bash
$ user_input="Yes"
$ if [[ "${user_input,,}" == "yes" ]]; then echo "Confirmed"; fi
Confirmed
```

### Example 14: Full chain for slug generation

```bash
$ title="Hello World! This is GREAT."
$ slug="${title,,}"            # lowercase
$ slug="${slug// /-}"          # spaces to hyphens
$ slug="${slug//[!a-z0-9-]/}"  # remove special chars
$ echo "$slug"
hello-world-this-is-great
```

**Step-by-step**: 1. Lowercase: "hello world! this is great." 2. Spaces → hyphens: "hello-world!-this-is-great." 3. Remove all non `[a-z0-9-]` chars: "hello-world-this-is-great".

## Real-World Use Cases

### FOR the OS

- Config file templating: `${config//\${VAR}/$VALUE}`
- Log normalization: `severity="${line,,}"`
- Git hooks: `commit_msg="${commit_msg,,}"` for lowercase enforcement
- Service naming: `service="${DAEMON,,}"; service="${service// /-}"`

### WITH the OS

- Batch rename: `${f// /_}` to replace spaces
- Log scrubbing: `${line//$IP/REDACTED}`
- grep simulation: `${line//pattern/\\n}`

### AGAINST the OS (security)

- Password masking: `${password//?/*}`
- SQL injection defense: `safe="${input//\'/\'\'}"`
- XSS prevention: `safe="${input//</&lt;}"`

### FOR DEFENSE

- Input sanitization: `safe="${filename//[^a-zA-Z0-9._-]/}"`
- Path traversal defense: `${path//\/..\//}`
- Case-insensitive blacklist: `[[ "${input,,}" == *"danger"* ]]`

## Memory Aids

- **`/` = slash separating old from new**: `${var/old/new}`
- **`//` = double slash = everywhere**: "double-check each spot"
- **`/#` = hash at start** = anchor at beginning
- **`/%` = percent at end** = anchor at end
- **`^^` = up up = uppercase**: "up, up, and away!"
- **`,,` = down down = lowercase**: "down, down, go low"
- **`~` = tilde = flip it** = toggle case

## Trap Vault (12 traps)

### Trap 1: Patterns are globs, not regex

```bash
str="hello123"
echo "${str/\d+/}"     # "hello123" — no match! \d+ is literal
# FIX: Use character class: echo "${str//[0-9]/}"
```

### Trap 2: Case conversion needs bash 4.0+

```bash
# On macOS (bash 3.2), this fails
echo "${var,,}"   # "bad substitution"
# FIX: Use tr for portability
```

### Trap 3: Non-overlapping replacement

```bash
str="aaaaa"
echo "${str//aa/-}"   # "--a" not "---" or "-----"
# Because after each match, scanning continues past the replacement
```

### Trap 4: `/` in pattern/replacement can't be escaped

```bash
var="a/b/c"
echo "${var///-}"   # syntax error
# FIX: slash="/"; echo "${var//$slash/-}"
```

### Trap 5: Empty pattern replaces nothing

```bash
var="hello"
echo "${var//$empty/X}"   # "hello" — no match
# FIX: Use explicit prefix: result="X$var"
```

### Trap 6: Replacement string is expanded

```bash
var="hello"
echo "${var//hello/$HOME}"   # "/home/user"
```

### Trap 7: `,` and `^` only affect first char

```bash
name="john DOE"
echo "${name^}"   # "John DOE" (only J capitalized)
echo "${name^^}"  # "JOHN DOE" (all)
```

### Trap 8: Locale-dependent case conversion

```bash
var="straße"
echo "${var^^}"   # May produce "STRASSE" or "STRAßE" depending on locale
```

### Trap 9: Anchored replacement anchors pattern, not replacement

```bash
${var/#hello/goodbye}  # only checks if var starts with hello
```

### Trap 10: `~` and `~~` not in POSIX

```bash
# Only bash 4.0+ supports case operators
```

### Trap 11: Character class negation

```bash
# Glob: [!abc] means "not a, b, or c"
# Regex: [^abc] means same — but in glob, use ! not ^
echo "${var//[!a-z]/}"  # removes non-lowercase-alpha
```

### Trap 12: `#` in replacement isn't a comment

```bash
echo "${var/#   # comment  /repl}"  # No — literal match
```

## See It In The Wild

- dotenv stripping: `${config//#*/}`
- Docker tags: `${VERSION//./-}` for "1.2.3" → "1-2-3"
- Ansible: `${hostname,,}` for lowercase

### Exploration

1. `${BASH_VERSION//./ }` → split version
2. `echo "${PATH//:/$'\n'}"` → PATH per line
3. `echo "${HOME//?/*}"` → mask path
4. `echo "${str/[!a-z]/}"` — remove first non-alpha

## Check Your Understanding (7 questions)

1. **Output?**: `str="Mississippi"; echo "${str//ss/SS}"; echo "${str/ss/SS}"`
2. **True or False**: `${var,,}` works in bash 3.2 (macOS default).
3. **Replace all "foo" with "bar" in `$text`** using parameter expansion.
4. **What does `${var/#abc/XYZ}` do if `var` is "abcabc"?**
5. **Convert filename to lowercase** (Photo.JPG → photo.jpg) using expansion.
6. **Why doesn't `${var/a*./REPLACE}` work like `sed 's/a.*\\./REPLACE/'`?**
7. **Remove all non-numeric from "+1 (555) 123-4567".**

## Supplementary Deep Dive: Advanced Replace & Case Patterns

### Removing path components

```bash
$ path="/home/user/docs/report.pdf"
$ # Remove everything up to and including last /
$ echo "${path##*/}"
report.pdf
$ # But also: replace directory separator with something
$ echo "${path//\//:}"
:home:user:docs:report.pdf
```

### Case-insensitive pattern matching with case modification

```bash
$ ext="JPG"
$ if [[ "${ext,,}" == "jpg" ]]; then
>   echo "It's a JPEG"
> fi
```

### Toggle case for C string convention

```bash
$ camelCase="helloWorld"
$ # Convert to SCREAMING_SNAKE_CASE
$ upper="${camelCase^^}"
$ echo "$upper"
HELLOWORLD
$ # Insert underscores before capitals (harder — need regex)
$ # Better with sed:
$ echo "helloWorld" | sed -E 's/([A-Z])/_\1/g' | tr '[:lower:]' '[:upper:]'
HELLO_WORLD
```

### Redacting sensitive data

```bash
$ credit_card="4111-1111-1111-1111"
$ # Show only last 4
$ echo "****-****-****-${credit_card: -4}"
****-****-****-1111

$ # Or replace all but last 4 with *
$ len=${#credit_card}
$ echo "$(printf '%*s' $((len-4)) '' | tr ' ' '*')${credit_card: -4}"
***********1111
```

### Character class tricks in patterns

```bash
$ # The [.], [*], [?] are literal inside character classes
$ str="hello.test"
$ echo "${str//[.?]/_}"       # Replace dot and question mark with _
hello_test

$ # Negation with ! inside []
$ str="a1b2c3"
$ echo "${str//[!0-9]/}"      # Remove non-digits
123
$ echo "${str//[0-9]/}"       # Remove digits
abc
```

### Multiple pattern anchoring

```bash
$ str="abc123def"
$ # Remove leading letters
$ echo "${str/#[a-z]*=([0-9])/}"  # extglob needed
$ # Simpler — use character class
$ echo "${str/#[a-z][a-z][a-z]/}"  # remove first 3 chars
123def
$ str="abcabc"
$ both="${str/#abc/---}"     # replace at start
$ both="${both/%abc/---}"    # replace at end
$ echo "$both"
---abc---                    # wait — second replacement fails because string is now ---abc
$ # Do it separately:
$ str="abcabc"
$ str="${str/#abc/---}"; echo "$str"   # ---abc
$ str="${str/%abc/---}"; echo "$str"   # ------ (wait, %abc matches the end abc)
```

### Combining replace with other expansions

```bash
$ str="  Hello World  "
$ # Strip whitespace, lowercase, dasherize
$ clean="${str#"${str%%[! ]*}"}"    # strip leading whitespace
$ clean="${clean%"${clean##*[! ]}"}" # strip trailing whitespace
$ clean="${clean,,}"
$ clean="${clean// /-}"
$ echo "$clean"
hello-world
```

### Using variables as patterns

```bash
$ original="hello world"
$ old="world"
$ new="there"
$ echo "${original/$old/$new}"
hello there
$ # Note: If $old contains /, it breaks! Use a delimiter variable:
$ sep="/"
$ # Actually you can't escape / in the pattern; use a variable
$ search="/path/to"
$ replace="/new/path"
$ # Can't do: "${var/$search/$replace}" because of / in search
$ # Workaround:
$ search_escaped="${search//\//\\/}"
$ replace_escaped="${replace//\//\\/}"
$ echo "$var" | sed "s/$search_escaped/$replace_escaped/g"
```

### Pattern matching with extglob

```bash
$ shopt -s extglob
$ str="abc123def456"
$ # Remove everything between letters using +(...)
$ echo "${str//+([0-9])/}"          # remove digit sequences
abcdef
$ echo "${str//+([a-z])/}"          # remove letter sequences
123456
$ echo "${str//+([a-z0-9])/}"       # remove everything? No — matches all
```

### Version number manipulation

```bash
$ version="1.2.3-beta"
$ # Bump patch version
$ major="${version%%.*}"           # 1
$ rest="${version#*.}"              # 2.3-beta
$ minor="${rest%%.*}"               # 2
$ patch="${rest#*.}"                # 3-beta
$ patch_num="${patch%%[!0-9]*}"     # 3
$ echo "${major}.${minor}.$((patch_num + 1))${patch#$patch_num}"
1.2.4-beta
```

## Supplementary Deep Dive: Advanced Replace & Case Patterns

### Conditional replacement using pattern matching

```bash
$ text="Error: connection failed"
$ prefix="${text%%:*}"
$ body="${text#*: }"
$ case "$prefix" in
>   Error|ERROR|error)
>     echo "ALERT: $body"
>     ;;
>   Warn|WARN|warn)
>     echo "Notice: $body"
>     ;;
>   *)
>     echo "$text"
>     ;;
> esac
ALERT: connection failed
```

### Using sed alongside parameter expansion

```bash
$ # Sometimes sed is more readable for complex replacements
$ text="apple,banana,cherry"
$ echo "$text" | sed 's/,/, /g'        # add space after each comma
apple, banana, cherry
$ echo "$text" | sed 's/[aeiou]/\U&/g' # uppercase all vowels
ApplE, bAnAnA, chErry
```

### Recursive parameter expansion

```bash
$ var="__hello__"
$ pattern="_"
$ while [[ "$var" == *"$pattern"* ]]; do
>   var="${var/$pattern/}"
> done
$ echo "$var"
hello
```

### Variable name transformation patterns

```bash
$ # Convert snake_case to camelCase
$ snake="my_variable_name"
$ # Step 1: split on _
$ words=(${snake//_/ })
$ # Step 2: capitalize each word after first
$ result="${words[0]}"
$ for ((i=1; i<${#words[@]}; i++)); do
>   word="${words[i]}"
>   result+="${word^^?}"  # Actually ${word^} for first char
> done
$ # Simpler with sed:
$ echo "my_variable_name" | sed -E 's/_([a-z])/\U\1/g'
myVariableName
```

### Case transformation with locale sensitivity

```bash
$ # Turkish locale has special case rules
$ LC_ALL=tr_TR.UTF-8
$ echo "i" | tr '[:lower:]' '[:upper:]'  # İ (dotted) not I
İ
$ # In POSIX/C locale:
$ LC_ALL=C
$ echo "i" | tr '[:lower:]' '[:upper:]'
I
```

### Using ${!var} for dynamic variable names

```bash
$ # With case transformation for dynamic dispatch
$ action="build"
$ build_cmd="gcc -o output source.c"
$ test_cmd="pytest"
$ deploy_cmd="rsync ..."
$ cmd="${action}_cmd"
$ echo "${!cmd}"
gcc -o output source.c
```

### Pattern-based prefix/suffix removal with arrays

```bash
$ files=(output.log output.csv output.tmp debug.log)
$ # Strip common prefix
$ prefix="output"
$ echo "${files[@]#$prefix}"
.log .csv .tmp debug.log
```

### Using / and // with character classes

```bash
$ text="A1B2C3D4"
$ echo "${text//[0-9]/X}"    # replace all digits
AXBXCXDX
$ echo "${text/[0-9]/X}"      # replace first digit only
AXB2C3D4
```

### Escaping special characters in replacement strings

```bash
$ text="/path/to/file"
$ # Replace / with \/ — need to escape
$ # Naive: ${text/\//\\/}  # very confusing
$ # Better: use a variable
$ slash="/"
$ replacement="\/"
$ echo "${text//$slash/$replacement}"
\/path\/to\/file
```

### Case-insensitive pattern matching for replacement

```bash
$ # Bash doesn't have case-insensitive pattern matching
$ # Workaround: convert to lower then compare
$ text="Hello World"
$ pattern="world"
$ if [[ "${text,,}" == *"${pattern,,}"* ]]; then
>   echo "Found (case-insensitive)"
> fi
Found (case-insensitive)
```
