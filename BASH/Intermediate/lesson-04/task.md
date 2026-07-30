# Task 4: Parameter Expansion — Replace & Case

## Overview

Write a script `clean_names.sh` that normalizes filenames and processes CSV data using **only parameter expansion** — no `sed`, `tr`, or `awk`.

## Sub-tasks

### 1. Name Normalizer

Given a name like `"  jOHN  DOE  "`, trim whitespace, convert to lowercase, and replace spaces with single underscores.

**Requirements:**
- Trim leading and trailing whitespace
- Convert to lowercase
- Replace any sequence of spaces with a single underscore
- Use only parameter expansion

```bash
#!/bin/bash
name="$1"
# Lowercase
clean="${name,,}"
# Replace spaces with underscores (but multiple spaces = multiple underscores)
# Better: squeeze spaces first — but bash can't do that natively
# Approach: use a loop to handle multiple spaces
while [[ $clean == *"  "* ]]; do
    clean="${clean//  / }"    # collapse double spaces
done
clean="${clean# }"            # trim leading
clean="${clean% }"            # trim trailing
clean="${clean// /_}"         # spaces to underscores
echo "$clean"
```

**Step-by-step for "  jOHN  DOE  ":**
1. `name="${1,,}"` → "  john  doe  " (lowercase)
2. Collapse double spaces loop: " john doe " (single spaces)
3. `${clean# }` → "john doe " (leading space removed)
4. `${clean% }` → "john doe" (trailing space removed)
5. `${clean// /_}` → "john_doe"

<details>
<summary>Hint: Title Case alternative</summary>
If you want Title_Case instead of lowercase (like John_Doe):
```bash
name="$1"
# First collapse spaces
while [[ $name == *"  "* ]]; do name="${name//  / }"; done
name="${name# }"; name="${name% }"
# Now capitalize each word
result=""
IFS=' ' read -ra words <<< "$name"
for word in "${words[@]}"; do
    word="${word,,}"
    result+="_${word^}"       # capitalize first letter
done
echo "${result:1}"           # remove leading underscore
```
</details>

### 2. CSV Cleaner

Given a CSV line like `"John","Doe","123 Main St","NYC"`, strip double quotes, lowercase all, and replace commas with pipes.

```bash
#!/bin/bash
line="$1"
nq="${line//\"/}"          # remove all double quotes
lower="${nq,,}"            # lowercase all
piped="${lower//,/|}"      # commas to pipes
echo "$piped"
```

**Step-by-step for '"John","Doe","NYC"':**
1. `line='"'John'","'Doe'","'NYC'"'` — actually let's be precise: input is `"John","Doe","NYC"`
2. `nq="${line//\"/}"` → `John,Doe,NYC` (all `"` removed)
3. `lower="${nq,,}"` → `john,doe,nyc`
4. `piped="${lower//,/|}"` → `john|doe|nyc`

<details>
<summary>Hint: Escaping the double quote</summary>
To match a literal `"` in a pattern, escape it with backslash: `\"`. The backslash tells bash "this double quote is literal, not a string delimiter."
</details>

### 3. Extension Normalizer

Loop through files matching `*.TXT`, `*.txt`, `*.Text`, `*.text` and rename them all to `.txt`.

```bash
#!/bin/bash
shopt -s nocaseglob          # case-insensitive glob matching
for f in *.txt; do
    [ -f "$f" ] || continue
    base="${f%.*}"
    ext="${f##*.}"
    ext_lower="${ext,,}"
    if [[ "$ext" != "$ext_lower" ]]; then
        mv -v "$f" "${base}.${ext_lower}"
    fi
done
```

<details>
<summary>Hint: Without nocaseglob</summary>
```bash
for f in *.txt *.TXT *.Text *.text; do
    [ -f "$f" ] || continue
    ...
done
```
</details>

### 4. Slug Generator

Take a string like "Hello World! This is GREAT." and convert to a URL slug: lowercase, spaces to hyphens, remove non `[a-z0-9-]`.

```bash
#!/bin/bash
text="$1"
slug="${text,,}"               # lowercase
slug="${slug// /-}"            # spaces to hyphens
# Remove non-alphanumeric (keeping hyphens) — we need character class
# Bash parameter expansion can't do regex, so we use tr here
slug=$(echo "$slug" | tr -dc 'a-z0-9-')
echo "$slug"
```

**Pure bash approach** (without tr, using character class in parameter expansion — but bash doesn't support `[^a-z]` negation in replacement patterns. Actually, `${var//[!a-z0-9-]/}` DOES work!):

```bash
slug="${text,,}"                   # lowercase
slug="${slug// /-}"                # spaces to hyphens
slug="${slug//[!a-z0-9-]/}"        # remove everything else
echo "$slug"
```

Wait — `[!a-z0-9-]` is glob pattern syntax inside `${var//pattern/}`. Globs support `[!...]` for negation. So this works!

**Step-by-step for "Hello World! This is GREAT.":**
1. `${text,,}` → "hello world! this is great."
2. `${slug// /-}` → "hello-world!-this-is-great."
3. `${slug//[!a-z0-9-]/}` → "hello-world-this-is-great"

<details>
<summary>Hint: Testing character classes</summary>
Test if a char class works: `str="abc123!@#"; echo "${str//[!a-z0-9]/}"` should give `abc123`.
If it doesn't, use `tr -dc 'a-z0-9'` as a fallback.
</details>

### 5. Bonus: CSV to HTML Table Converter

```bash
#!/bin/bash
# Convert CSV to HTML table row
line="$1"
# Strip quotes
nq="${line//\"/}"
# Replace commas with </td><td>
html="<tr><td>${nq//,/</td><td>}</td></tr>"
echo "$html"
```

### 6. Bonus: String Replacement Counter

```bash
#!/bin/bash
str="$1"
old="$2"
new="$3"
# Count occurrences before replacement
count_before="${#str}"
replaced="${str//$old/$new}"
change=$(( (${#replaced} - count_before) / (${#new} - ${#old}) ))
echo "Replaced $change occurrence(s)"
```

## Expected Output

```
$ ./clean_names.sh "  jOHN  DOE  "
john_doe

$ ./clean_names.sh "  ALICE   M.  SMITH  "
alice_m_smith

$ echo '"John","Doe","123 Main St","NYC"' | ./clean_names.sh --csv
john|doe|123 main st|nyc

$ ./clean_names.sh --slug "Hello World! This is GREAT."
hello-world-this-is-great

$ ./clean_names.sh --title "  jOHN  DOE  "
John_Doe

$ ./clean_names.sh --csv '"Alice","Smith","NYC","10001"'
alice|smith|nyc|10001
```

## Self-Check

- What does `${var,,}` do vs `${var,}`?
- How do you replace only the first occurrence of a pattern?
- Why is `${var/#old/new}` different from `${var/old/new}`?
- Can `${var//pattern/replacement}` handle regex patterns?
- When would `${var^^}` fail on an older system?
- How do you remove all characters except letters and numbers from a string?
- What's the difference between `[!a-z]` in a glob pattern vs `[^a-z]` in a regex?
