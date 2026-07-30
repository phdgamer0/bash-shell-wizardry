# Task 10: Here-docs, Here-strings & mapfile

You're building a **configuration management toolkit** — three scripts that generate configs, parse structured data, and render templates using heredocs, here-strings, and mapfile. This is the kind of tooling you'd use to deploy applications across servers, generate dynamic HTML reports, or process CSV exports from databases.

## Sub-tasks

### 1. Config generator — `gen_config.sh`

Write a script that reads variables from a `config.env` file (format: `NAME=VALUE`) and generates an Nginx-style configuration file using a heredoc template.

**Requirements:**
- Accept the env file path as the first argument (default: `config.env`)
- Use `mapfile -t` to read the env file
- Parse each line into `name` and `value` using `${var%%=*}` and `${var#*=}`
- Use `declare "$name=$value"` to bring variables into scope
- Generate the config via an unquoted heredoc so variables expand
- Output to stdout (or to a file with `-o filename`)
- Include at minimum: `server_name`, `listen`, `root`, `index`, `location /api` block with `proxy_pass`

**Edge cases to handle:**
- Lines starting with `#` are comments — skip them
- Empty lines — skip them
- Values with quotes — pass them through (e.g., `SERVER_NAMES="host1 host2"`)
- Variables that might not be in the env file — use `${VAR:-default}` patterns

**Expected:**
```bash
# config.env
PORT=3000
SERVER_NAME=example.com
DB_HOST=db.internal
DB_PORT=5432
ROOT=/var/www/example
INDEX=index.html

$ ./gen_config.sh
server {
    listen 3000;
    server_name example.com;
    root /var/www/example;
    index index.html;
    location /api {
        proxy_pass http://db.internal:5432;
    }
}

$ ./gen_config.sh -o /tmp/nginx.conf
Wrote config to /tmp/nginx.conf
```

### 2. CSV parser — `parse_csv.sh`

Write a CSV parser that reads a CSV file, splits fields using `mapfile` and `read -ra`, and displays data as a formatted table.

**Requirements:**
- Accept CSV file as argument
- Use `mapfile -t` to read all lines
- For each line, use `IFS=',' read -ra fields <<< "$line"` to split
- Print a formatted table with `printf` using aligned columns
- Column widths should auto-detect based on data (first pass to measure, second to print)
- Handle up to 6 columns
- Highlight the header row (uppercase, separator line)

**Edge cases:**
- Empty fields (`a,,c`) — should preserve empty fields (note: basic `IFS=',' read -ra` collapses consecutive delimiters; document this limitation)
- Trailing commas
- Spaces around fields (trim them)
- Quoted fields with internal commas (e.g., `"Smith, John"`) — optional bonus

**Expected:**
```bash
$ cat data.csv
Name,Email,Role,Department
Alice Johnson,alice@example.com,Admin,Engineering
Bob Smith,bob@test.com,User,Sales
Charlie Brown,charlie@example.com,User,Engineering

$ ./parse_csv.sh data.csv
NAME                 EMAIL                ROLE                 DEPARTMENT
──────────────────────────────────────────────────────────────────────────
Alice Johnson        alice@example.com    Admin                Engineering
Bob Smith            bob@test.com         User                 Sales
Charlie Brown        charlie@example.com  User                 Engineering
```

### 3. Template renderer — `render.sh`

Write a script that reads a template file with `{{VAR}}` placeholders and a `values.txt` with `key=value` pairs, and renders the output.

**Requirements:**
- Template file as first argument, values file as second
- Read values with `mapfile`, parse with `${line%%=*}` / `${line#*=}`
- Use `declare` to create variables from key=value pairs
- Use `sed` to replace `{{VAR}}` with the actual value (or use parameter expansion with `${!var}`)
- Support default values: `{{PORT:-80}}` → use value or default
- Output rendered text

**Edge cases:**
- Undefined variable — leave `{{VAR}}` as-is or warn
- Values with spaces — must handle correctly
- Template with no substitutions — output unchanged
- Multiple occurrences of same variable — all should be replaced

**Expected:**
```bash
$ cat template.html
<html>
<head><title>{{TITLE:-My Site}}</title></head>
<body>
  <h1>Welcome to {{SITE_NAME}}</h1>
  <p>Server: {{SERVER_NAME}}</p>
  <p>Port: {{PORT}}</p>
  <p>Debug: {{DEBUG:-false}}</p>
</body>
</html>

$ cat values.txt
TITLE=My App
SITE_NAME=Kali Dashboard
SERVER_NAME=web01.internal
PORT=8080

$ ./render.sh template.html values.txt
<html>
<head><title>My App</title></head>
<body>
  <h1>Welcome to Kali Dashboard</h1>
  <p>Server: web01.internal</p>
  <p>Port: 8080</p>
  <p>Debug: false</p>
</body>
</html>
```

### 4. Email validator — `validate.sh`

Write a function that validates an email address using a regex and here-string, then a script that reads a list of emails from a file and validates each.

**Requirements:**
- Function `validate_email()` that uses `[[ $email =~ pattern ]]` and feeds the email via `<<<`
- Read emails from a file (one per line) using `mapfile`
- Print VALID/INVALID for each
- Count and report totals

**Expected:**
```bash
$ cat emails.txt
alice@example.com
bob@test
charlie@.com
dave@sub.domain.com
invalid@

$ ./validate.sh emails.txt
alice@example.com       VALID
bob@test                INVALID
charlie@.com            INVALID
dave@sub.domain.com     VALID
invalid@                INVALID
Summary: 2 valid, 3 invalid
```

## Solution Approaches

### Approach A: Sequential (beginners)
1. Start with `gen_config.sh` — simplest heredoc usage
2. Move to `parse_csv.sh` — introduces `mapfile` + `read -ra`
3. Tackle `render.sh` — combines both with `sed`/`envsubst`
4. Finish with `validate.sh` — brief here-string validation

### Approach B: All-at-once mapfile
1. Start with `validate.sh` since it's the shortest and uses `mapfile` simply
2. `parse_csv.sh` — same mapfile pattern, adds field splitting
3. `gen_config.sh` — heredoc generation
4. `render.sh` — combines all concepts

### Approach C: Pipeline-focused
1. `render.sh` first — centers on `envsubst` and heredoc patterns
2. `parse_csv.sh` — data processing pipeline
3. `gen_config.sh` — output generation
4. `validate.sh` — validation as a capstone

<details>
<summary>Hint 1: Parsing config.env with mapfile</summary>

```bash
mapfile -t lines < "config.env"
for line in "${lines[@]}"; do
  [[ "$line" =~ ^#.*$ || -z "$line" ]] && continue
  name="${line%%=*}"
  value="${line#*=}"
  declare "$name=$value"
done
```

Use `typeset -p PORT` after to verify.
</details>

<details>
<summary>Hint 2: Auto-detecting column widths</summary>

```bash
max_widths=(0 0 0 0 0 0)
for row in "${rows[@]}"; do
  IFS=',' read -ra fields <<< "$row"
  for i in "${!fields[@]}"; do
    len=${#fields[$i]}
    (( len > max_widths[i] )) && max_widths[i]=$len
  done
done

for row in "${rows[@]}"; do
  IFS=',' read -ra fields <<< "$row"
  for i in "${!fields[@]}"; do
    printf "%-*s" $((max_widths[i] + 2)) "${fields[$i]}"
  done
  echo
done
```
</details>

<details>
<summary>Hint 3: Template rendering with sed or parameter expansion</summary>

Option A — using `sed`:
```bash
while IFS='=' read -r key val; do
  sed -i "s/{{${key}}}/${val}/g" "$output"
done < "$values_file"
```

Option B — using indirect expansion (requires declare first):
```bash
declare "$key=$val"
```

Option C — transform `{{VAR}}` to `$VAR` then uses `envsubst`:
```bash
sed 's/{{\([^}]*\)}}/${\1}/g' "$template" | envsubst
```
`envsubst` is part of GNU gettext.
</details>

<details>
<summary>Hint 4: Email regex pattern</summary>

```bash
email_regex='^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$'
```
Usage:
```bash
if [[ "$email" =~ $email_regex ]]; then
  echo "VALID"
fi
```
</details>

<details>
<summary>Hint 5: Handling empty fields in CSV</summary>

The default `IFS=',' read -ra` collapses consecutive delimiters. To preserve them:
```bash
line="${line//,,/,\"\",}"
IFS=',' read -ra fields <<< "$line"
```
</details>

## Bonus Challenges

1. **Recursive template**: Allow `{{INCLUDE:other_file}}` to pull in content from another file
2. **CSV with headers as keys**: Parse the first CSV row as field names, then output each row as JSON objects
3. **Config encrypt**: Add `--encrypt` / `--decrypt` options that use `gpg` or `openssl` to protect sensitive config values
4. **YAML output**: Instead of Nginx config, generate YAML or JSON from the env file

## Expected Output Summary

```
gen_config.sh:  Reads env file → generates config via heredoc, variables expand automatically
parse_csv.sh:   Reads CSV with mapfile, splits with read -ra, auto-sizes columns
render.sh:      Reads template with {{VAR}} placeholders, renders with envsubst/sed
validate.sh:    Reads emails from file, validates with regex via [[ =~ ]], summary counts
```

## Self-Check

- What's the exact difference between `<< EOF` and `<< 'EOF'`? When would each be a security concern?
- Why does `md5sum <<< "hello"` differ from `printf "hello" | md5sum`?
- What happens to the variable `$var` after `echo "data" | read var` — and why should you use `<<<` instead?
- How would you read exactly 5 lines from a file into an array?
- What's the performance difference between `mapfile` and `while read` for a 10MB file? A 1GB file?
- How do you prevent variable expansion in a heredoc without using quotes on the delimiter?
- What does `${var%%=*}` do that's different from `${var%=*}` or `${var#*=}`?
