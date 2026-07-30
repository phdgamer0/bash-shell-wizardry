# Task 6: Associative Arrays

## Overview

Write three scripts that process text using associative arrays for counting and lookup. Associative arrays are perfect for frequency analysis, grouping, and fast key-based retrieval.

## Sub-tasks

### 1. Word Frequency Counter (wordfreq.sh)

Read a text file (or stdin), count occurrences of each word (case-insensitive), and print sorted by frequency (highest first).

**Requirements:**
- Use an associative array for counting
- Case-insensitive: "The" and "the" count as the same word
- Sort output by frequency (highest first), then alphabetically for ties
- Handle punctuation: "hello," and "hello" should be the same word

```bash
#!/bin/bash
declare -A freq
input=${1:-/dev/stdin}

# Extract words: lowercase, strip punctuation
for word in $(tr '[:upper:]' '[:lower:]' < "$input" | tr -cs '[:alpha:]' '\n'); do
    ((freq[$word]++))
done

# Sort by frequency (descending), then alphabetically
for word in "${!freq[@]}"; do
    echo "${freq[$word]} $word"
done | sort -rn
```

**Step-by-step:**
1. `tr '[:upper:]' '[:lower:]'` lowercases the entire input.
2. `tr -cs '[:alpha:]' '\n'` replaces non-alpha characters with newlines (`-c` complement, `-s` squeeze).
3. The `for` loop iterates over each word (now one per line after translation).
4. `((freq[$word]++))` increments the count for each word. First occurrence: 0→1, second: 1→2, etc.
5. The final loop prints each word and its count, piped through `sort -rn` (numeric, reverse).

<details>
<summary>Hint: Better word extraction (preserving apostrophes)</summary>
```bash
tr '[:upper:]' '[:lower:]' < "$input" | grep -oE "[a-z']+" | while read -r word; do
    word="${word#\'}"; word="${word%\'}"  # strip surrounding quotes
    ((freq[$word]++))
done
```
Or use `tr` for simpler cases: `tr -sc "[:alpha:]'" '\n'`
</details>

<details>
<summary>Hint: Sort by frequency then alphabetically</summary>
```bash
for word in "${!freq[@]}"; do
    printf "%d %s\n" "${freq[$word]}" "$word"
done | sort -k1,1rn -k2,2
```
`-k1,1rn` sorts first field numerically descending; `-k2,2` sorts second field alphabetically ascending.
</details>

### 2. IP Counter from Logs (ipcount.sh)

Read Apache-style log lines from stdin, count unique IPs, and print the top 5 most frequent IPs.

**Requirements:**
- IP is the first whitespace-delimited field in each line
- Count occurrences of each IP
- Print top 5 sorted by count (descending)

```bash
#!/bin/bash
declare -A ip_count

while IFS= read -r line; do
    ip="${line%% *}"
    ((ip_count[$ip]++))
done

# Print top 5
for ip in "${!ip_count[@]}"; do
    echo "${ip_count[$ip]} $ip"
done | sort -rn | head -5
```

**Alternative — pipe from file:**
```bash
#!/bin/bash
declare -A ip_count
while read -r line; do
    ((ip_count[${line%% *}]++))
done < "${1:-/dev/stdin}"

# Top 5
for ip in "${!ip_count[@]}"; do
    echo "${ip_count[$ip]} $ip"
done | sort -rn | head -5
```

<details>
<summary>Hint: Simulating the access.log</summary>
Generate a test file:
```bash
for i in {1..100}; do
    echo "192.168.1.$((RANDOM % 10)) - - [30/Jul/2025:12:34:56 +0000] \"GET / HTTP/1.1\" 200 1234"
done > access.log
```
</details>

### 3. User Lookup Database (userdb.sh)

Read a CSV file (`username,fullname,role,status`) into an associative array and support lookups.

**Commands:**
- `lookup <username>` — show full details
- `list` — list all usernames
- `byrole <role>` — list users with given role

```bash
#!/bin/bash
declare -A users
DBFILE="${1:-users.csv}"

# Load database
while IFS=',' read -r username fullname role status; do
    users[$username]="$fullname,$role,$status"
done < "$DBFILE"

lookup() {
    local data="${users[$1]:-not found}"
    if [[ "$data" == "not found" ]]; then
        echo "User '$1' not found"
        return 1
    fi
    IFS=',' read -r fullname role status <<< "$data"
    echo "Username: $1"
    echo "Full Name: $fullname"
    echo "Role: $role"
    echo "Status: $status"
}

list_users() {
    for user in "${!users[@]}"; do
        echo "$user"
    done | sort
}

byrole() {
    local target_role="$1"
    for user in "${!users[@]}"; do
        IFS=',' read -r _ role _ <<< "${users[$user]}"
        if [[ "$role" == "$target_role" ]]; then
            echo "$user"
        fi
    done | sort
}

case "${2:-help}" in
    lookup) lookup "$3" ;;
    list) list_users ;;
    byrole) byrole "$3" ;;
    *) echo "Usage: $0 <dbfile> {lookup <user>|list|byrole <role>}" ;;
esac
```

<details>
<summary>Hint: Building a test users.csv</summary>
```bash
cat > users.csv << 'EOF'
jdoe,John Doe,admin,active
jsmith,Jane Smith,user,inactive
bob,Bob Brown,user,active
alice,Alice Wang,admin,active
charlie,Charlie Kim,user,active
EOF
```
</details>

### 4. Bonus: Reverse Index (value → list of keys)

```bash
#!/bin/bash
declare -A index
# Build reverse index: for each word, store which files it appears in
for file in *.txt; do
    while read -r word; do
        index[$word]+=" $file"
    done < <(tr -cs '[:alpha:]' '\n' < "$file" | sort -u)
done

# Show which files contain "error"
echo "Files containing 'error':${index[error]}"
```

### 5. Bonus: Config Map with Validation

```bash
#!/bin/bash
declare -A REQUIRED_KEYS=(
    [DB_HOST]="Database host"
    [DB_PORT]="Database port (numeric)"
    [DB_NAME]="Database name"
)

declare -A config
error=0

# Read config file
while IFS='=' read -r key value; do
    [[ -z "$key" || "$key" == "#"* ]] && continue
    config[$key]=$value
done < "${1:-config.ini}"

# Validate required keys
for key in "${!REQUIRED_KEYS[@]}"; do
    if [[ ! -v config[$key] ]]; then
        echo "MISSING: $key (${REQUIRED_KEYS[$key]})" >&2
        ((error++))
    fi
done

(( error == 0 )) && echo "Config valid" || echo "$error errors found"
```

## Expected Output

```
$ echo "The cat and the dog and the cat" | ./wordfreq.sh
3 the
2 cat
1 and
1 dog

$ cat access.log | ./ipcount.sh
192.168.1.1: 42
192.168.1.3: 15
10.0.0.2: 8
192.168.1.7: 3
10.0.0.5: 2

$ cat users.csv
jdoe,John Doe,admin,active
jsmith,Jane Smith,user,inactive
bob,Bob Brown,user,active

$ ./userdb.sh users.csv lookup jdoe
Username: jdoe
Full Name: John Doe
Role: admin
Status: active

$ ./userdb.sh users.csv byrole user
bob
jsmith

$ ./userdb.sh users.csv list
bob
jdoe
jsmith
```

## Self-Check

- What does `declare -A` do, and why is it necessary?
- How do you iterate over all keys in an associative array?
- Are keys guaranteed to be in insertion order?
- How do you check if a specific key exists in an associative array?
- What happens if you try to use an associative array in bash 3.2?
- How do you distinguish "key exists with empty value" from "key doesn't exist"?
- What's the performance characteristic of associative array lookup vs indexed array?
