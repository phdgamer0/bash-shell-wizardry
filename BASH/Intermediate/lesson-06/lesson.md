# Lesson 6: Associative Arrays

## History & Origins

Associative arrays (dictionaries/hash maps) were introduced in **bash 4.0** (2009). Before this, the only way to create key-value mappings in bash was to simulate them with variable naming conventions (e.g., `var_key1=value1`) or use external tools like `awk`.

The syntax for declaring associative arrays — `declare -A map` — was chosen to distinguish them from indexed arrays (`declare -a`). The `${!map[@]}` syntax for listing keys reuses the `${!var}` indirection operator, extended to mean "all keys" when combined with `[@]`.

Key-value lookup semantics in bash have some notable differences from real hash tables:
- Keys are always strings (even if you pass a number, it's stringified)
- Values are always strings (but can be treated as numbers in `(( ))`)
- Iteration order is **undefined** (usually insertion order in practice, but never rely on it)
- Hash collisions are handled internally (bash uses a variation of Python's hash table algorithm)

## Syntax Reference

| Expression | What it does | Example |
|------------|-------------|---------|
| `declare -A map` | Declare associative array | `declare -A capitals` |
| `map[key]=value` | Assign value to key | `capitals[France]=Paris` |
| `${map[key]}` | Access value by key | `echo "${capitals[France]}"` |
| `${!map[@]}` | List all keys | `for k in "${!map[@]}"` |
| `${map[@]}` | List all values | `echo "${map[@]}"` |
| `${#map[@]}` | Number of entries | `echo "Count: ${#map[@]}"` |
| `unset "map[key]"` | Remove entry | `unset "capitals[France]"` |
| `unset map` | Remove entire map | `unset capitals` |
| `[[ -v map[key] ]]` | Check if key exists (bash 4.2+) | `[[ -v capitals[Japan] ]]` |
| `map+=([key]=val)` | Add/modify entry | `capitals+=([Spain]=Madrid)` |

### Key Rules

- Keys must be quoted if they contain special characters or spaces:
  ```bash
  map["my key"]=value    # OK — key has a space
  map[my key]=value      # ERROR — syntax error
  ```
- Keys are always stored as strings: `map[42]=value` stores key "42".
- Keys can be variables: `key="France"; echo "${capitals[$key]}"`.
- Key existence vs value: `${map[key]}` returns empty string if key doesn't exist. Use `-v` to distinguish "key exists with empty value" from "key doesn't exist".

### Iteration

```bash
# Iterate keys
for key in "${!map[@]}"; do
    echo "Key: $key"
done

# Iterate values
for value in "${map[@]}"; do
    echo "Value: $value"
done

# Iterate both
for key in "${!map[@]}"; do
    echo "$key → ${map[$key]}"
done
```

### Common Operations

**Copy associative array** (bash 4.0+):
```bash
declare -A copy=()
for k in "${!original[@]}"; do
    copy[$k]="${original[$k]}"
done
```

**Merge associative arrays**:
```bash
declare -A merged=()
for k in "${!map1[@]}" "${!map2[@]}"; do
    merged[$k]="${map1[$k]:-${map2[$k]}}"
done
```

**Check if key exists** (bash 4.2+):
```bash
[[ -v map[key] ]] && echo "exists" || echo "not exists"
```

**Count entries**: `${#map[@]}` — number of key-value pairs.

**Get all keys as array**: `keys=("${!map[@]}")` — creates an indexed array of keys.

## Under the Hood

### What bash does

Associative arrays in bash are implemented as **hash tables** (specifically, bash uses the `HASH_TABLE` type which is based on an open-addressing hash table with power-of-two sizing).

When you do `map[key]=value`:

1. `key` is stringified (if it's a number or special var result).
2. The hash function (a variant of PJW hash) computes a hash of the key string.
3. The hash is mapped to a bucket index in the hash table.
4. If the bucket is empty, the key-value pair is stored.
5. If the bucket is occupied (collision), bash probes for the next empty slot (open addressing).
6. The key string is stored separately from the hash for collision resolution.

When you access `${map[key]}`:

1. Hash the key string.
2. Compute bucket index.
3. If the bucket's stored key matches, return the value.
4. If not, probe sequentially until finding the matching key or an empty bucket (key doesn't exist).

### System Calls

Associative array operations: **zero system calls**. All in-process hash table operations.

### Performance

- **Insert**: Amortized O(1) — occasional resize when load factor exceeds threshold.
- **Lookup**: O(1) average case; O(n) worst case (if many collisions, but rare with good hash).
- **Delete**: O(1) average case.
- **Iterate all keys**: O(n) — must walk the hash table buckets.
- **Size**: Uses murmur-like hash (PJW variant); collisions are handled by open addressing.

### Memory

- Hash table starts small (default ~16 buckets) and doubles when ~75% full.
- Keys and values are stored as separate strings on the heap.
- Each entry has overhead: pointer to key string, pointer to value string, hash value, and a flag byte.
- For 10,000 entries, approximate memory: ~1-2 MB (including string data).

### Comparison with C/Python

**Python dict equivalent**:
```python
map = {}
map["key"] = "value"
print(map["key"])
for k in map:
    print(k, map[k])
"key" in map  # existence check
len(map)
del map["key"]
```

**C equivalent** (using POSIX hsearch/hcreate):
```c
#include <search.h>
ENUM *e;
e = (ENUM *) malloc(sizeof(ENUM));
e->key = "France";
e->data = "Paris";
hsearch(*e, ENTER);
// Lookup
ENUM *found = hsearch(*e, FIND);
```

## Core Examples (12 minimum)

### Example 1: Declare and populate

```bash
$ declare -A capitals
$ capitals[France]=Paris
$ capitals[Germany]=Berlin
$ capitals[Italy]=Rome
$ echo "${capitals[Germany]}"
Berlin
```

**Step-by-step**: 1. `declare -A` creates an empty hash table. 2. Each assignment inserts a key-value pair. 3. Access by key retrieves the value.

**What if** you forget `declare -A`? Bash creates an **indexed** array instead. `capitals[France]` would try to evaluate `France` as a number → 0, so `capitals[0]=Paris`.

### Example 2: Iterate over keys and values

```bash
$ for country in "${!capitals[@]}"; do
>   echo "$country → ${capitals[$country]}"
> done
Germany → Berlin
France → Paris
Italy → Rome
```

**Step-by-step**: `${!capitals[@]}` expands to all keys. Each key is used to index back into the array.

**Note**: Order is NOT guaranteed. On your system it might be different.

### Example 3: Check key existence

```bash
$ if [[ -v capitals[Japan] ]]; then
>   echo "Found"
> else
>   echo "Not found"
> fi
Not found
```

**Step-by-step**: `-v capitals[Japan]` checks if "Japan" exists as a key. It doesn't → false.

**What if you used `${capitals[Japan]}`**? Empty string — but that could also mean the value is empty. Always use `-v` to check existence.

### Example 4: Word frequency counter

```bash
$ declare -A freq
$ for word in apple banana apple cherry apple banana; do
>   ((freq[$word]++))
> done
$ for word in "${!freq[@]}"; do
>   echo "$word: ${freq[$word]}"
> done
banana: 2
cherry: 1
apple: 3
```

**Step-by-step**: `((freq[$word]++))` increments the integer value stored at key `$word`. First "apple" → 0+1=1. Second → 1+1=2. Third → 2+1=3.

**What if** the word contains spaces? Keys with spaces need quoting: `freq["my word"]=1`.

### Example 5: IP counter from log

```bash
$ declare -A ip_count
$ while read -r line; do
>   ip="${line%% *}"
>   ((ip_count[$ip]++))
> done <<< "192.168.1.1 GET /index.html
> 192.168.1.2 GET /about.html
> 192.168.1.1 GET /index.html"
$ for ip in "${!ip_count[@]}"; do
>   echo "$ip: ${ip_count[$ip]}"
> done
192.168.1.2: 1
192.168.1.1: 2
```

### Example 6: User database from CSV

```bash
$ declare -A users
$ users[jdoe]="John Doe,admin,active"
$ users[jsmith]="Jane Smith,user,inactive"
$ echo "${users[jdoe]%%,*}"    # extract first field
John Doe
```

### Example 7: Reverse lookup (value → key)

```bash
$ declare -A capital_of
$ capital_of[Paris]=France
$ capital_of[Berlin]=Germany
$ echo "${capital_of[Paris]}"
France
```

### Example 8: Counting file types

```bash
$ declare -A ext_count
$ for f in *; do
>   ext="${f##*.}"
>   ((ext_count[$ext]++))
> done
$ for ext in "${!ext_count[@]}"; do
>   echo ".$ext: ${ext_count[$ext]} files"
> done
```

### Example 9: Cache/memoization pattern

```bash
$ declare -A cache
$ compute() {
>   local key="$1"
>   if [[ -v cache[$key] ]]; then
>     echo "Cached: ${cache[$key]}"
>   else
>     local result=$(( key * 2 ))   # expensive computation
>     cache[$key]=$result
>     echo "Computed: $result"
>   fi
> }
$ compute 5
Computed: 10
$ compute 5
Cached: 10
```

### Example 10: Multi-value storage (CSV in value)

```bash
$ declare -A employee
$ employee[101]="Alice Smith,Engineering,Manager,alice@co.com"
$ employee[102]="Bob Jones,Sales,Associate,bob@co.com"
$
$ lookup() {
>   local id="$1"
>   IFS=',' read -r name dept role email <<< "${employee[$id]:-not found}"
>   echo "Name: $name, Dept: $dept, Role: $role, Email: $email"
> }
$ lookup 101
Name: Alice Smith, Dept: Engineering, Role: Manager, Email: alice@co.com
```

### Example 11: Config map from file

```bash
$ cat > /tmp/config << 'EOF'
db_host=localhost
db_port=5432
db_name=myapp
EOF
$
$ declare -A config
$ while IFS='=' read -r key value; do
>   config[$key]=$value
> done < /tmp/config
$ echo "${config[db_host]}:${config[db_port]}"
localhost:5432
```

### Example 12: Grouping items by property

```bash
$ declare -A groups
$ items=("fruit:apple" "fruit:banana" "animal:cat" "fruit:cherry" "animal:dog")
$ for item in "${items[@]}"; do
>   group="${item%%:*}"
>   name="${item#*:}"
>   groups[$group]+="$name "
> done
$ for g in "${!groups[@]}"; do
>   echo "$g: ${groups[$g]}"
> done
fruit: apple banana cherry
animal: cat dog
```

## Real-World Use Cases

### FOR the OS

- IP frequency in access logs (security monitoring)
- User configuration maps (ini-like config)
- File extension → MIME type mappings
- Process ID tracking by service name

### WITH the OS

- `df` output organized by filesystem type
- Process states grouped by PID
- Port → service mappings from `/etc/services`
- Hostname → IP resolution caching

### AGAINST the OS

- Detect port scans via IP frequency thresholding
- Brute-force detection: track failed login attempts per IP
- Resource usage tracking per user/process
- Config validation: mandatory key checking

### FOR DEFENSE

- Anti-DoS: rate-limit tracking per source IP
- Whitelist/blacklist lookups (O(1))
- Session token validation
- Audit trail with deduplication by key

## Memory Aids

- **`declare -A`**: The `A` = "Associative" (uppercase A for key-value)
- **`${!map[@]}`**: The `!` = "keys!" — like an exclamation pointing to the keys
- **"Maps are for mapping keys to values"**: Think of a treasure map — keys lead to values
- **"No order, like a bag of keys"**: Don't rely on insertion order
- **`-v` key check**: "v" = "verify" existence

## Trap Vault (12 traps)

### Trap 1: Forgetting `declare -A` creates an indexed array

```bash
# BAD
capitals[France]=Paris   # bash warns: "attempt to use array subscript"
# bash treats France as variable → evaluates to 0 → capitals[0]=Paris

# FIX
declare -A capitals
capitals[France]=Paris   # correct
```

### Trap 2: Iteration order is undefined

```bash
declare -A map=([a]=1 [b]=2 [c]=3 [d]=4)
for k in "${!map[@]}"; do echo "$k"; done
# Might print: c a b d (or any order)
# NEVER rely on insertion order!
```

### Trap 3: Key existence vs empty value

```bash
declare -A map
map[empty]=""
map[nonempty]="value"

[[ -v map[empty] ]] && echo "empty key exists"   # TRUE
[[ -n "${map[empty]}" ]] && echo "value nonempty"  # FALSE
# To check if key exists: use -v, not -n on value
```

### Trap 4: `declare -A` without `-g` in a function is local

```bash
setup_map() {
    declare -A map  # LOCAL to the function!
    map[key]=value
}
setup_map
echo "${map[key]}"  # empty — map doesn't exist outside

# FIX: declare -gA map or declare -A map outside the function
```

### Trap 5: Keys with spaces must be quoted

```bash
# BAD
declare -A map
map[key with spaces]=value   # syntax error

# FIX
map["key with spaces"]=value   # correct
```

### Trap 6: Associative arrays require bash 4.0+

```bash
# On macOS (bash 3.2 default):
declare -A map   # error: declare: -A: invalid option
# FIX: Upgrade bash, or use awk/Python for hash maps
```

### Trap 7: Copying associative arrays needs a loop

```bash
# BAD: creates indexed array!
new=("${old[@]}")   # new is NOW INDEXED even if old was assoc

# BAD: creates string of values
new_str="${old[@]}"

# GOOD: explicit copy
declare -A new
for k in "${!old[@]}"; do new[$k]="${old[$k]}"; done
```

### Trap 8: Wrong subscript syntax with indirection

```bash
# BAD: tries indirect expansion
key="France"
echo "${!key}"   # This does indirect variable lookup, not array access!

# GOOD: use variable as subscript
echo "${capitals[$key]}"   # Correct
```

### Trap 9: Numerical keys are stringified

```bash
declare -A map
map[1]=value
echo "${map[01]}"   # empty! 01 ≠ 1 as string
echo "${map[1]}"    # "value"
# Keys are compared as strings, not numbers
```

### Trap 10: Unset doesn't always release memory

```bash
declare -A map
for i in {1..10000}; do map[$i]=$i; done
unset map   # frees the array
# But if any references exist (e.g., passed by name to function), memory leaks
```

### Trap 11: `-v` check with special characters in key

```bash
declare -A map
map["[special]"]=value
[[ -v "map[special]" ]]        # BAD: bash sees this as conditional expression
# FIX: Quote properly
[[ -v 'map[special]' ]]        # OK with single quotes
```

### Trap 12: `read -A` vs `read -a`

```bash
# BAD: expecting associative array from read
read -A arr <<< "key value"   # -A doesn't exist! Use -a for indexed

# Associative arrays must be built element by element:
declare -A map
while IFS='=' read -r key value; do map[$key]=$value; done < file
```

## See It In The Wild

- **Bash completion**: `${COMPREPLY[@]}` — completion system
- **Log analyzing scripts**: IP frequency, user agent counting
- **Configuration loaders**: `source` config files into associative arrays
- **Docker inspect parsing**: Field extraction into maps

### Exploration

1. Create a map of the first 1000 primes as keys with their index as value. Check lookup speed.
2. Compare `${!map[@]}` with piping through `sort` — see how order differs.
3. Use an associative array to implement a simple in-memory database.
4. Can you nest associative arrays? (Bash doesn't support multi-dimensional — use key naming like `parent_child`)

## Check Your Understanding (7 questions)

1. **What does `declare -A` do, and why must you use it?**
2. **How to iterate over all keys in an associative array?**
3. **Are keys guaranteed to be in insertion order?**
4. **How to check if a specific key exists in an associative array?**
5. **What happens if you use an associative array in bash 3.2?**
6. **How do you distinguish "key exists with empty value" from "key doesn't exist"?**
7. **How would you copy one associative array to another?**

## Supplementary Deep Dive: Advanced Associative Array Patterns

### Nested/multi-key data with composite keys

```bash
$ declare -A config
$ config["db,host"]="localhost"
$ config["db,port"]="5432"
$ config["app,name"]="myapp"
$ config["app,version"]="1.0"
$
$ section="db"
$ echo "${config[$section,host]}:${config[$section,port]}"
localhost:5432
```

### Counting multiple dimensions

```bash
$ declare -A daily_counts
$ while read -r line; do
>   date="${line%% *}"
>   ip="${line##* }"
>   key="${date}:${ip}"
>   ((daily_counts[$key]++))
> done < access.log
$ # Query: how many requests from 192.168.1.1 on July 30?
$ echo "${daily_counts[2025-07-30:192.168.1.1]:-0}"
42
```

### Map with sorted output

Always sort keys when output is for humans:

```bash
$ print_sorted() {
>   local -n map=$1
>   local key
>   for key in "${!map[@]}" | sort; do
>     printf "%s → %s\n" "$key" "${map[$key]}"
>   done
> }
```

### Mapping to multiple values (list per key)

```bash
$ declare -A groups
$ groups[fruits]+="apple "
$ groups[fruits]+="banana "
$ groups[colors]+="red "
$ groups[fruits]+="cherry "
$ groups[colors]+="blue "
$
$ for group in "${!groups[@]}"; do
>   echo "$group: ${groups[$group]}"
> done
fruits: apple banana cherry
colors: red blue
```

### Building a cache with associative arrays

```bash
$ declare -A dns_cache
$ resolve_ip() {
>   local host="$1"
>   if [[ -v dns_cache[$host] ]]; then
>     echo "Cached: ${dns_cache[$host]}"
>     return
>   fi
>   local ip=$(getent hosts "$host" | awk '{print $1}')
>   dns_cache[$host]="$ip"
>   echo "Looked up: $ip"
> }
```

### Associative array as a set (unique values)

```bash
$ declare -A unique_words
$ words=(the cat in the hat sat on the mat)
$ for w in "${words[@]}"; do unique_words[$w]=1; done
$ echo "Unique words: ${!unique_words[@]}"
the cat in hat sat on mat
```

### Aggregation across multiple keys

```bash
$ declare -A total_by_user errors_by_user
$ while IFS=',' read -r user status; do
>   ((total_by_user[$user]++))
>   (( status >= 400 )) && ((errors_by_user[$user]++))
> done < /var/log/app.csv
$
$ for user in "${!total_by_user[@]}"; do
>   total=${total_by_user[$user]}
>   errors=${errors_by_user[$user]:-0}
>   pct=$(( errors * 100 / total ))
>   echo "$user: $errors/$total errors ($pct%)"
> done
```

### Associative array equality check

```bash
$ maps_equal() {
>   local -n a=$1 b=$2
>   if [[ "${!a[*]}" != "${!b[*]}" ]]; then return 1; fi
>   local k
>   for k in "${!a[@]}"; do
>     if [[ "${a[$k]}" != "${b[$k]}" ]]; then return 1; fi
>   done
>   return 0
> }
```

### Using associative arrays for enum-like constants

```bash
$ declare -A LOG_LEVEL
$ LOG_LEVEL=([DEBUG]=0 [INFO]=1 [WARN]=2 [ERROR]=3)
$ current_level=${LOG_LEVEL[INFO]}
$ log_message() {
>   local lvl=${LOG_LEVEL[$1]:-99}
>   (( lvl >= current_level )) && echo "[$1] $2"
> }
$ log_message DEBUG "detail"    # not shown (DEBUG < INFO)
$ log_message WARN "problem"    # shown (WARN > INFO)
[WARN] problem
```

### Group by: partitioning array elements

```bash
$ declare -A by_ext
$ files=(readme.txt image.jpg script.sh data.csv notes.txt)
$ for f in "${files[@]}"; do
>   ext="${f##*.}"
>   by_ext[$ext]+=" $f"
> done
$ for ext in "${!by_ext[@]}"; do
>   echo ".$ext:${by_ext[$ext]}"
> done
.txt: readme.txt notes.txt
.jpg: image.jpg
.sh: script.sh
.csv: data.csv
```
