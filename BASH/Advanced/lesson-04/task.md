# Task 4: Plugin Loader & Function Inspector

## Objective
Build a secure plugin loader that loads scripts from a directory safely, write a function inspector using `declare -f` and `declare -p`, generate dynamic code safely using namerefs and arrays, and implement integrity verification. This task mirrors how real-world shell frameworks (bash-it, oh-my-bash) manage plugins.

## Requirements

1. **Plugin loader:** Load all `.sh` files from `~/.myapp/plugins/`, run their `init` functions, and track loaded plugins
2. **Function inspector:** Write a script that lists all defined functions, shows their source (body), and indicates if they are exported or readonly
3. **Safe dynamic code:** Create a script that generates wrapper functions dynamically for a predefined list of commands, without using `eval` where possible; where `eval` is unavoidable, use `printf '%q'`
4. **Validation:** Check that no plugin contains suspicious patterns (backticks, `eval`, `exec`) before loading — but understand this is NOT real security
5. **Integrity verification:** Compute and verify SHA256 hashes of plugins before loading
6. **Plugin isolation:** Load each plugin in a subshell to verify it doesn't crash, then source in the main shell
7. **Function diffing:** Compare function definitions before and after loading plugins to detect modifications

## Sub-tasks (8 cumulative)

### Task 4.1: Basic Plugin Loader
Create a plugin directory and basic loader:

```bash
#!/bin/bash
PLUGIN_DIR=~/.myapp/plugins
mkdir -p "$PLUGIN_DIR"

# Create two test plugins
cat > "$PLUGIN_DIR/hello.sh" << 'EOF'
init() { echo "Hello plugin initialized"; }
hello() { echo "Hello, $1!"; }
EOF

cat > "$PLUGIN_DIR/date.sh" << 'EOF'
init() { echo "Date plugin initialized"; }
show_date() { date "$@"; }
EOF

# Plugin loader
echo "Loading plugins..."
loaded=0
failed=0
for plugin in "$PLUGIN_DIR"/*.sh; do
  [ -f "$plugin" ] || continue
  if source "$plugin" 2>/dev/null; then
    echo "  [OK] $(basename "$plugin")"
    if declare -F init &>/dev/null; then
      init
      unset -f init
    fi
    ((loaded++))
  else
    echo "  [FAIL] $(basename "$plugin")"
    ((failed++))
  fi
done
echo "Loaded $loaded plugins, $failed failed"
```

### Task 4.2: Function Inspector
Build a script that reports on the shell's current function state:

```bash
#!/bin/bash
inspect_functions() {
  local fn type_info
  echo "=== Function Inventory ==="
  for fn in $(compgen -A function | sort); do
    type_info=$(type -t "$fn")
    attrs=""
    # Check if exported
    declare -F "$fn" | grep -q '\-x' && attrs+="exported "
    echo "Function: $fn ($type_info) [$attrs]"
    echo "--- Body ---"
    declare -f "$fn" | head -5
    echo "---"
    echo
  done
}

# Also inspect all variables:
inspect_variables() {
  echo "=== Variable Inventory ==="
  for var in $(compgen -A variable | sort | head -20); do
    declare -p "$var" 2>/dev/null | head -1
  done
}
```

**Multiple approaches compared:**

```bash
# Approach A: compgen + declare (as above) — complete but slow for many items
# Approach B: declare -f alone — no iteration needed, but output is verbose
$ declare -f  # Prints all functions at once

# Approach C: typeset -f (ksh style) — same as declare -f
```

### Task 4.3: Safe Dynamic Wrapper Generation
Generate wrapper functions that prefix `my_` to each command:

```bash
# The SAFE way using arrays and dispatch:
declare -A COMMAND_DISPATCH
COMMANDS=(ls cat echo date)

for cmd in "${COMMANDS[@]}"; do
  # Store the full path to avoid PATH injection
  cmd_path=$(command -v "$cmd")
  COMMAND_DISPATCH["my_${cmd}"]="$cmd_path"
done

my() {
  local cmd="my_$1"
  shift
  if [[ -n "${COMMAND_DISPATCH[$cmd]}" ]]; then
    "${COMMAND_DISPATCH[$cmd]}" "$@"
  else
    echo "Unknown command: $1" >&2
    return 1
  fi
}

# Usage:
$ my ls -la
$ my cat /etc/hostname
```

**Why this is safer than eval:**
1. No code generation — the dispatch is pre-defined
2. Commands are validated against array keys
3. Full paths stored to prevent PATH hijacking
4. Arguments are passed directly, not through string parsing

### Task 4.4: Plugin Validation
Validate plugins before loading:

```bash
validate_plugin() {
  local file="$1"
  local issues=0
  
  # Check for dangerous patterns (imperfect — attackers obfuscate)
  if grep -qE '`|eval |exec |\$\(|import|base64|decode' "$file" 2>/dev/null; then
    echo "  [WARN] Suspicious patterns found in $(basename "$file")"
    ((issues++))
  fi
  
  # Check file permissions
  local perms
  perms=$(stat -c "%a" "$file" 2>/dev/null)
  if [[ "${perms: -1}" -gt 0 ]]; then
    echo "  [WARN] $(basename "$file") is world-writable"
    ((issues++))
  fi
  
  return $issues
}
```

**IMPORTANT CAVEAT:** Grep-based validation is NOT security. An attacker can encode payloads in base64, use `$(printf '\145\166\141\154')` to hide `eval`, or exploit shell features that the grep pattern misses. Real security comes from:
1. File ownership and permissions
2. Signed/checksummed plugins
3. Running plugins in restricted contexts
4. Not loading untrusted plugins in the first place

### Task 4.5: Integrity Verification
Compute and verify SHA256 hashes:

```bash
PLUGIN_HASH_DIR=~/.myapp/hashes

verify_plugin() {
  local plugin="$1"
  local hash_file="$PLUGIN_HASH_DIR/$(basename "$plugin").sha256"
  
  if [[ ! -f "$hash_file" ]]; then
    echo "  [WARN] No hash file for $(basename "$plugin") — skipping"
    return 1
  fi
  
  if sha256sum -c "$hash_file" --status 2>/dev/null; then
    echo "  [OK] Hash matches for $(basename "$plugin")"
    return 0
  else
    echo "  [FAIL] Hash mismatch for $(basename "$plugin")!"
    return 1
  fi
}

# To generate hash file:
# sha256sum ~/.myapp/plugins/hello.sh > ~/.myapp/hashes/hello.sh.sha256
```

**Trust model:** This only works if the hash files themselves are secured (read-only, owned by root, stored on a read-only filesystem).

### Task 4.6: Plugin Isolation Test
Run each plugin in a subshell to verify it doesn't have syntax errors or crash:

```bash
test_plugin() {
  local plugin="$1"
  # Run in a subshell with timeout
  timeout 2 bash -c "source '$plugin'" 2>&1 || {
    echo "  [FAIL] Plugin test failed: $(basename "$plugin")"
    return 1
  }
}

# Then load if test passes:
for plugin in "$PLUGIN_DIR"/*.sh; do
  if test_plugin "$plugin"; then
    source "$plugin"
    echo "  [OK] Loaded $(basename "$plugin")"
  fi
done
```

**TRAP:** The subshell test runs in a DIFFERENT bash instance. Variables and functions defined by `source` in the subshell don't affect the parent. This test is for syntax validation only.

### Task 4.7: Function Diffing
Detect changes to core functions after plugin loading:

```bash
#!/bin/bash
declare -A BEFORE_AFTER

# Snapshot function definitions before loading
snapshot_functions() {
  local fn
  BEFORE_AFTER=()
  for fn in $(compgen -A function | sort); do
    BEFORE_AFTER["$fn"]=$(declare -f "$fn" 2>/dev/null)
  done
}

# Compare after loading
check_changes() {
  local fn
  for fn in $(compgen -A function | sort); do
    local current=$(declare -f "$fn" 2>/dev/null)
    if [[ "${BEFORE_AFTER[$fn]}" != "$current" ]]; then
      echo "  [CHANGED] Function '$fn' was modified or redefined"
    fi
  done
}

snapshot_functions
source "$1"  # Load the plugin
check_changes
```

### Task 4.8: Complete Plugin Manager
Combine everything into a single tool:

```bash
#!/bin/bash
# plugin_manager.sh — list, enable, disable, verify, inspect plugins

PLUGIN_DIR=~/.myapp/plugins
PLUGIN_HASH_DIR=~/.myapp/hashes

list_plugins() {
  echo "Available plugins:"
  for plugin in "$PLUGIN_DIR"/*.sh; do
    [ -f "$plugin" ] || continue
    base=$(basename "$plugin" .sh)
    # Check if loaded
    if declare -F "init_$base" &>/dev/null; then
      echo "  [*] $base (loaded)"
    else
      echo "  [ ] $base"
    fi
  done
}

load_plugin() {
  local name="$1"
  local plugin="$PLUGIN_DIR/$name.sh"
  if [[ ! -f "$plugin" ]]; then
    echo "Plugin $name not found" >&2
    return 1
  fi
  if verify_plugin "$plugin"; then
    source "$plugin"
    echo "Loaded: $name"
  else
    echo "Refusing to load: integrity check failed" >&2
    return 1
  fi
}

case "${1:-}" in
  list) list_plugins ;;
  load) load_plugin "$2" ;;
  inspect) inspect_functions ;;
  verify) verify_plugin "$2" ;;
  *) echo "Usage: plugin_manager {list|load|inspect|verify}" ;;
esac
```

## Bonus Challenges

1. **Bonus A:** Implement a plugin dependency system where plugins declare `# Plugin-Depends: net, utils` and the loader resolves dependency order.

2. **Bonus B:** Create a "namespace" system that loads plugins in a restricted environment where they can only access whitelisted commands (using `rbash` features).

3. **Bonus C:** Implement hot-reloading — watch plugin files for changes with `inotifywait` and reload them without restarting the parent script.

4. **Bonus D:** Build a function firewall — intercept all `declare -f` calls to hide sensitive function definitions from unprivileged code.

5. **Bonus E:** Create a plugin sandbox using Linux namespace isolation (`unshare`) — each plugin runs in its own mount, PID, and network namespace.

## Hints

<details>
<summary>Hint 1: compgen vs declare</summary>

`compgen -A function` lists function NAMES only. `declare -f` lists DEFINITIONS. Use compgen for iteration, declare -f for details.
</details>

<details>
<summary>Hint 2: Detecting function origin</summary>

`shopt -s extdebug; declare -F funcname` shows file and line number where a function was defined.
</details>

<details>
<summary>Hint 3: Hash storage</summary>

Store hashes in a directory separate from plugins, with `chmod 444` on hash files and `chown root:root`.
</details>

<details>
<summary>Hint 4: Array-based dispatch limits</summary>

An array can't hold function bodies (no closures). For dynamic function generation, you still need eval — but constrain what string enters eval.
</details>

<details>
<summary>Hint 5: Plugin lifecycle hooks</summary>

Support hooks: `pre_load`, `init`, `post_load`, `on_error`, `on_unload`. Define them in plugins and call them from the loader if they exist.
</details>

## Expected Output

```bash
$ ./plugin_loader.sh
Loading plugins...
  [OK] hello.sh — init() registered
  [OK] date.sh — init() registered
  [WARN] unsafe.sh — contains suspicious patterns
  [FAIL] broken.sh — syntax error on line 3
Loaded 2 of 4 plugins, 1 failed, 1 skipped

$ ./func_inspector.sh
=== Function Inventory ===
Function: hello (function) []
--- Body ---
hello ()
{
    echo "Hello, $1!"
}
---

Function: show_date (function) []
--- Body ---
show_date ()
{
    date "$@"
}
---

$ ./plugin_manager.sh verify hello.sh
Verifying hello.sh...
  [OK] Hash matches for hello.sh

$ ./plugin_manager.sh load unsafe.sh
Verifying unsafe.sh...
  [WARN] Suspicious patterns found in unsafe.sh
  [FAIL] Integrity check: no hash file
Refusing to load: integrity check failed
```

## Deep Self-Check

1. **Syntax error in plugin:** What exactly happens when `source` encounters a syntax error? Which lines load and which don't? Test with `source <(echo 'echo "hello"; invalid command; echo "world"')`.

2. **Function redefinition:** Load a plugin that defines `ls`. Does it override the system's `ls` function or the command? How does `type -t ls` change?

3. **Plugin isolation fallacy:** Why does running `source plugin.sh` in a subshell NOT actually isolate the plugin's effects? How would you truly sandbox a plugin?

4. **Hash collision demonstration:** Can you create a plugin that has the same SHA256 as a legitimate plugin but different content? (Answer: practically impossible with SHA256, but SHA1 has collisions demonstrated.)

5. **Performance of declare -f vs file read:** Time how long `declare -f` takes with 1000 functions vs reading 1000 function definitions from a file. Which is faster?

6. **Circular source detection:** Write a script that detects when sourcing creates a loop (A sources B, B sources A). Use `BASH_SOURCE` array.

7. **`set -e` and source:** If `set -e` is active and a sourced file returns non-zero, does the parent exit? Test it.

8. **Environment variable leakage through declare -p:** Source a plugin that sets an API key in a variable. Then run `declare -p` in the parent shell. Is the key visible? How would you protect it?

9. **Completion function timing:** Create a completion function that takes 5 seconds. Type TAB in a terminal. What happens? Why is this a DOS vector?

10. **BASH_SOURCE[0] vs $0:** What is the difference when sourcing a file? Trace `$0` and `${BASH_SOURCE[0]}` across source operations.
