# Lesson 4: Dynamic Code & Reflection

## History & Origins

Dynamic code loading in shell scripting dates back to the `.` (dot) command in the Bourne shell (1977). The idea was simple: include common functions from a shared file, like a primitive library system. The `source` alias (synonym for `.`) was added in bash and csh for readability.

The `declare` builtin (originally `typeset` in ksh) gave shells the ability to inspect and modify variable and function attributes dynamically. Bash adopted it in bash 2.0 (1996). The `-f` flag for function inspection and `-p` for variable attributes turned bash into a self-reflective environment.

Programmable completion (`compgen`, `complete`, `compopt`) was introduced in bash 2.04 (1999). It allowed users to write dynamic tab-completion handlers in bash itself, replacing the old static `/etc/bash_completion` approach. This was revolutionary — suddenly every command could have context-aware completion written entirely in shell script.

The `eval` builtin (from the Bourne shell) has always been the sharp edge of dynamic code. It's the shell's equivalent of C's `system()` — a way to turn a string into executable code. Its dangers were recognized early, but it remains necessary for certain patterns like programmatic variable names and dynamic redirections.

## Syntax Reference

### Loading Code
```
source filename [args]    # Load and execute file in current shell
. filename [args]         # POSIX synonym for source
```

### Reflection — Functions
```
declare -f [name]         # Print function definition(s)
declare -F [name]         # Print only function names (not bodies)
declare -f                # Print ALL function definitions
typeset -f [name]         # Same as declare -f (ksh compat)
compgen -A function       # List all defined function names
```

### Reflection — Variables
```
declare -p [name]         # Print variable attributes and value
declare -p                # Print ALL variable definitions
set                       # Print all variables and functions
compgen -A variable       # List all variable names
compgen -A arrayvar       # List array variable names
compgen -A export         # List exported variables
```

### Reflection — Commands
```
type -a name              # Show ALL definitions (alias, function, builtin, file)
type -t name              # Show only type (alias, function, builtin, file, keyword)
hash                      # Display command hash table
hash -l                   # List hashed commands with paths
```

### Programmable Completion
```
complete -F func cmd      # Set completion function for cmd
complete -p [cmd]         # Print completion settings
compgen -W "wordlist"     # Generate completions from word list
compgen -d                # Generate directory completions
compgen -f                # Generate file completions
compgen -A function       # Generate function name completions
```

### Dynamic Code Generation
```
eval "code as string"     # Parse and execute generated code
printf '%q'               # Quote for safe eval
declare -n ref=var        # Nameref (bash 4.3+) — reference variable by name
${!var}                   # Indirect expansion — get value of variable named by $var
```

## Under the Hood

### The Source Operation

When you run `source file.sh`, bash does:

1. Opens `file.sh` (same as `exec N<file.sh`)
2. Reads the file content into memory
3. Passes the content through the parser (same parser used for interactive input)
4. Executes each parsed command in the current shell context
5. Closes the file

Key difference from running `bash file.sh`:
- `source` uses the CURRENT shell — no subshell, no fork, no exec
- `bash file.sh` forks a NEW shell — variables set in `file.sh` don't affect the caller
- `source` inherits all traps, set options, and shell state
- `bash file.sh` starts fresh with default state

```
Source:
Parent Shell ──reads──▶ file.sh ──parses──▶ executes in parent context
                    (no fork/exec)

Subshell:
Parent Shell ──fork──▶ Child Shell ──reads──▶ file.sh ──parses──▶ executes
                    (new process)                               (child exits, parent unchanged)
```

### The Eval Operation

When you run `eval "some code"`, bash:

1. Concatenates all arguments with spaces
2. Passes the result through the FULL parsing pipeline:
   - Tokenization (breaking into words)
   - Expansion (variable, arithmetic, command substitution)
   - Quote removal
   - Redirection parsing
   - Command execution

This is the identical pipeline used for normal commands. The only difference is the source of the string.

```
Normal:  echo $var  ──▶ tokenize ──▶ expand ──▶ quote_remove ──▶ execute
                              ▲
Eval:    "echo $var" ─────────┘
                    (bypasses initial parse, enters at expansion)
```

### The declare -f Inspection

`declare -f` prints the stored parse tree of a function in a re-parsable format. Bash stores functions internally as:

```
struct function {
    char *name;
    WORD_LIST *body;    // Pre-parsed command list
    int flags;          // exported, readonly, trace, etc.
};
```

When you call `declare -f funcname`, bash serializes this internal representation back to text. This is why `declare -f` output can be sourced back — it's designed for round-tripping.

### Programmable Completion Internals

When you press TAB, bash:
1. Determines the command being completed
2. Checks if a `complete -F` function is registered for it
3. If yes, calls the function with `COMP_WORDS`, `COMP_CWORD`, `COMP_LINE`, and `COMP_POINT` set
4. The function sets `COMPREPLY` array with possible completions
5. Bash displays/filters the completions

The completion function runs in the current shell, so it can access variables, run commands, and use any bash feature to generate completions.

### Security Model Interactions

**SELinux and source:** `source` respects file permissions. If a file is in a directory with `noexec` mount flag, `source` may still work (it reads and interprets, doesn't exec). But SELinux may block read access based on the file's context.

**AppArmor and source:** Similar — path-based access controls apply. A confined profile may restrict which directories a script can source from.

**Capabilities and reflection:** `declare -f` and `declare -p` don't require special capabilities — they just read internal shell state. But they can reveal sensitive information (passwords in variables, API keys).

### Comparison with C/libc

| Concept | C | Bash |
|---|---|---|
| Load code at runtime | `dlopen()`/`dlsym()` | `source` |
| Reflect on functions | `dlsym()` lookup | `declare -f` |
| Dynamic code gen | `jit()` or `system()` | `eval` |
| Variable reflection | `dlsym()` on symbols | `declare -p` |
| Plugin pattern | `dlopen` from dir | `for f in dir/*; do source "$f"; done` |

The key difference: C's `dlopen` is an explicit dynamic loader with symbol resolution and type checking. Bash's `source` is just "run this file." There's no sandbox, no namespace isolation, no type checking.

## Core Examples

### Example 1: Loading Code at Runtime

```bash
$ cat > /tmp/plugin.sh << 'EOF'
hello() { echo "Hello, $1!"; }
goodbye() { echo "Goodbye, $1!"; }
EOF
$ source /tmp/plugin.sh
$ hello World
Hello, World!
$ goodbye World
Goodbye, World!
```

**Anatomy:**
1. `source /tmp/plugin.sh` reads the file and defines two functions
2. The functions are now available in the current shell
3. `hello` and `goodbye` execute as if they were defined locally

**What if variations:**
- `. /tmp/plugin.sh` — same as source (POSIX form)
- `source /tmp/plugin.sh World` — passes arguments to the sourced script

**Dangerous edge case:** Sourcing from a world-writable directory opens you to TOCTOU attacks. Between checking the file and sourcing it, an attacker can modify it.

### Example 2: Function Introspection with declare -f

```bash
$ hello() { echo "Hi $1"; }
$ declare -f hello
hello ()
{
    echo "Hi $1"
}
```

**Anatomy:**
- `declare -f hello` prints the complete function body
- The output is valid bash that can be re-sourced

**What if variations:**
- `declare -F hello` — prints just `declare -f hello` (name only, no body)
- `declare -f` — prints ALL function definitions
- `compgen -A function` — lists all function names (useful in scripts)

### Example 3: Variable Introspection with declare -p

```bash
$ declare -r myvar=42
$ declare -p myvar
declare -r myvar="42"
```

**Anatomy:**
- `declare -p myvar` shows the variable's attributes (`-r` for readonly) and value
- This is invaluable for debugging: you see both type and value
- Without the variable name, `declare -p` prints ALL variables

**What if variations:**
- `declare -p PATH` — shows PATH with all its attributes
- `declare -p BASH_VERSINFO` — shows array type and values

### Example 4: Listing All Functions with compgen

```bash
$ compgen -A function | head -3
hello
declare
...
```

`compgen -A function` is the scriptable way to list all functions. Unlike `declare -f`, it only outputs names, not definitions.

### Example 5: Generating Code Dynamically

```bash
$ for i in {1..3}; do
>   eval "task_$i() { echo \"Running task $i\"; }"
> done
$ task_2
Running task 2
```

**Anatomy:**
1. `eval "task_$i() { ... }"` defines a function with a dynamic name
2. The loop creates `task_1`, `task_2`, `task_3`
3. Each function body contains the literal value of `$i` at definition time

**TRAP:** Errors in generated code point to the `eval` line, not the generated code. Debugging is harder.

**What if variations:**
- Use `$((++counter))` for sequential task numbers
- Use associative arrays instead of eval: `tasks[$i]="echo Running task $i"` — no eval needed

### Example 6: Plugin Loader Pattern

```bash
$ cat > loader.sh << 'EOF'
PLUGIN_DIR=/tmp/plugins
for plugin in "$PLUGIN_DIR"/*.sh; do
  [ -f "$plugin" ] && source "$plugin" 2>/dev/null || echo "Failed to load $plugin"
done
EOF
```

**Anatomy:**
1. Loop over all `.sh` files in the plugin directory
2. `source "$plugin"` loads each one into the current shell
3. Plugins define functions that the main script can call

### Example 7: Creating a Custom Completion

```bash
$ _mycmd_complete() {
>   local words="start stop restart status"
>   COMPREPLY=($(compgen -W "$words" -- "${COMP_WORDS[1]}"))
> }
$ complete -F _mycmd_complete mycmd
$ mycmd [TAB][TAB]
start stop restart status
```

**Anatomy:**
1. The function `_mycmd_complete` is called when pressing TAB after `mycmd`
2. `COMP_WORDS` is an array of the current command line words
3. `COMP_CWORD` is the index of the current cursor position
4. `COMPREPLY` must be set to an array of possible completions
5. `compgen -W "$words" -- "${COMP_WORDS[1]}"` filters the word list to matches

### Example 8: Nameref Variables (bash 4.3+)

```bash
$ declare -n ref=myvar
$ myvar="hello"
$ echo "$ref"
hello
$ ref="world"  # Modifies myvar through the reference
$ echo "$myvar"
world
```

**Anatomy:**
- `declare -n ref=myvar` creates a reference (like a pointer) to `myvar`
- Reading `$ref` reads `myvar`
- Writing to `ref` writes to `myvar`
- Namerefs are safer than `eval` for dynamic variable access

### Example 9: Indirect Expansion vs Nameref

```bash
$ # Indirect expansion (older, limited)
$ var="hello"
$ ref="var"
$ echo "${!ref}"
hello

$ # Nameref (bash 4.3+, cleaner, supports arrays)
$ declare -n ref2=var
$ echo "$ref2"
hello
```

Indirect expansion `${!name}` works for simple variables but not for arrays. Namerefs work for all variable types.

### Example 10: Dynamic Plugin Validation

```bash
$ validate_plugin() {
>   local file="$1"
>   local suspicious=0
>   while IFS= read -r line; do
>     case "$line" in
>       *'\x60'*|*'$('*|*';'*|*'exec'*) suspicious=1 ;;
>     esac
>   done < "$file"
>   return $suspicious
> }
$ validate_plugin /tmp/plugins/safe.sh && source /tmp/plugins/safe.sh
```

**TRAP:** This is NOT real security. An attacker can obfuscate code. Real security comes from file permissions, integrity checking, and running plugins in restricted environments.

### Example 11: Function Overriding Detection

```bash
$ original_ls() { command ls "$@"; }
$ ls() { echo "Overridden!"; original_ls "$@"; }
$ declare -f ls
ls ()
{
    echo "Overridden!";
    original_ls "$@"
}
```

You can detect function overrides by comparing `type -t cmd` before and after loading a plugin.

### Example 12: Self-Modifying Script

```bash
$ cat > selfmod.sh << 'EOF'
#!/bin/bash
if [[ ! -f /tmp/initialized ]]; then
  echo 'init_done() { echo "Initialized!"; }' >> "$0"
  touch /tmp/initialized
fi
init_done
EOF
```

A script that modifies itself! This is dangerous and fragile (if the script is read-only, it fails). But it demonstrates that `eval` and `source` aren't the only ways to create dynamic code.

### Example 13: Reflection for Debugging

```bash
$ debug_var() {
>   local var_name="$1"
>   declare -p "$var_name" 2>/dev/null || echo "$var_name is not set"
>   echo "Type: $(declare -p "$var_name" 2>/dev/null | cut -d' ' -f2)"
> }
$ my_array=(a b c)
$ debug_var my_array
declare -a my_array=([0]="a" [1]="b" [2]="c")
Type: -a
```

### Example 14: Safe Dynamic Script Generation Using Arrays

```bash
$ # Instead of eval, use arrays and a dispatch table:
$ declare -A tasks
$ tasks[clean]="rm -rf /tmp/build"
$ tasks[build]="make -j4"
$ tasks[test]="make test"
$ task_name="build"
$ ${tasks[$task_name]}  # Safer than eval — constrained to array values
```

But even this is dangerous if `$task_name` comes from user input (it could be any key, and the value could be anything).

### Example 15: Using compgen for Dynamic Prompts

```bash
$ select_from_list() {
>   local prompt="$1"; shift
>   local items=("$@")
>   PS3="$prompt: "
>   select item in "${items[@]}"; do
>     echo "$item"
>     break
>   done
> }
$ select_from_list "Pick a file" *.txt
1) file1.txt
2) file2.txt
Pick a file: 2
file2.txt
```

## Real-World Use Cases

### FOR the OS
- **bash_completion:** The entire `/etc/bash_completion.d/` system is a plugin loader for completions
- **profile scripts:** `/etc/profile.d/*.sh` are sourced on login to set up the environment
- **init system compatibility:** `/etc/init.d/` scripts sometimes source `/lib/lsb/init-functions` for common helpers

### WITH the OS
- **Module systems:** Scripting frameworks use plugin directories (like Oh My Zsh, bash-it)
- **Testing frameworks:** `bats` (Bash Automated Testing System) uses source to load test files
- **Configuration reloading:** `source /etc/myapp.conf` to reload config without restart
- **Dynamic PROMPT_COMMAND:** Modify PS1 and PROMPT_COMMAND dynamically for context-aware prompts

### AGAINST the OS (defense perspective)
- **Evaluated injection:** If an attacker can write to a sourced file, they inject code into any script that sources it
- **Completion hijacking:** Malicious completion files in `/etc/bash_completion.d/` execute on every TAB press
- **declare -f data leak:** If an attacker can run `declare -p` in a compromised session, they see all variables including secrets
- **`eval` in profiles:** Some profile scripts use `eval` with environment variables — classic injection vector

### FOR DEFENSE
- **Validate plugin signatures:** Check SHA256 hashes before sourcing plugins
- **Use namerefs instead of eval:** Replace `eval "echo \$$var"` with `declare -n ref=$var; echo "$ref"`
- **Restrict completion directories:** Ensure `/etc/bash_completion.d/` is root-owned and not world-writable
- **Profile integrity monitoring:** Use AIDE or Tripwire to detect changes to sourced scripts
- **Run plugins in subshells:** `(source plugin.sh; run_hook)` — limits plugin's ability to modify the parent

## Memory Aids

- **"Source shares, subshell spares"** — `source` shares the current shell; `bash file.sh` spawns a new one
- **"declare -f shows the blueprint"** — `declare -f` prints the function definition, which is the reverse-engineered source code
- **"eval is a second chance at parsing"** — `eval` re-runs the parser on its string argument
- **"compgen lists, complete assigns"** — `compgen` generates possibilities; `complete` binds them to commands
- **"Nameref = Pointer Lite"** — `declare -n ref=var` is like `int *ref = &var` in C
- **"Indirect = double dollar"** — `${!var}` means "the variable whose name is in `$var`"

## Trap Vault

1. **Source is NOT importing — it's executing:** `source file.sh` doesn't just define functions — it runs EVERY line in the file. A file with `rm -rf /` at the top level will delete files when sourced.

2. **`declare -f` output may not be exactly the original source:** Bash normalizes function formatting. Comments are stripped. Some whitespace is normalized. The output is semantically equivalent but not textually identical.

3. **`eval` and quoting levels:** Each level of eval strips one level of quoting. `eval "eval \"echo hello\""` requires careful quote counting. Triple-eval is a debugging nightmare.

4. **Source with relative paths depends on PWD:** `source foo.sh` looks in the current directory. If PWD changed, it sources a different `foo.sh`. Always use absolute paths or `dirname "$0"`.

5. **Completion functions run on every TAB:** If your completion function is slow, every TAB press hangs the terminal. Keep completion functions fast.

6. **`declare -n` cannot reference itself:** `declare -n ref; declare -n ref=ref` causes an infinite loop and crashes bash in some versions.

7. **`unset -f` vs `unset` for functions:** `unset func` unsets a variable named `func`. `unset -f func` unsets a function. If a variable and function share a name, `unset` only removes the variable.

8. **`source` in a function changes the function's scope:** `source` in a function defines variables in the function's local scope unless they're declared with `-g`. This can leak or isolate variables unexpectedly.

9. **`compgen -A function` includes builtins and keywords:** It lists ALL callable items, not just user-defined functions. Filter with `type -t` to distinguish.

10. **`BASH_SOURCE` vs `BASH_LINENO`:** `BASH_SOURCE` tracks the source file chain. `BASH_LINENO` tracks line numbers. Both are arrays indexed by call depth. Useful for debugging source chains.

11. **Sourcing from a symlink:** `source /path/to/symlink` follows the symlink. `BASH_SOURCE` contains the resolved path, not the symlink path. This can confuse scripts that check their own path.

12. **`eval` in trap handlers:** `trap "eval \$dangerous" EXIT` double-evals. The trap's string is parsed when the trap fires, and eval re-parses it. One eval is enough.

13. **`declare -p` of array shows indexed format:** `declare -p arr` shows `declare -a arr=([0]="a" [1]="b")`, even if you defined it as `arr=(a b)`. This is the canonical format.

14. **Completion variables are read-only in the completion function:** You can't modify `COMP_WORDS` or `COMP_CWORD` in a completion function. They're set by bash before calling your function.

15. **`source` with `/dev/stdin`:** `source /dev/stdin` reads commands from stdin. This is useful for remote execution: `curl http://example.com/payload | source /dev/stdin` — which is also extremely dangerous.

## See It In The Wild

### Observing shopt for debugging
```bash
$ shopt -p extdebug  # Extended debugging mode for functions
$ shopt -s extdebug
$ declare -F func  # Shows file and line number of function definition
```

### Tracing source operations
```bash
$ strace -e trace=openat,read bash -c 'source /etc/os-release; echo $ID'
```

### Inspecting the function stack
```bash
$ func() { echo "In func"; caller 0; }
$ source <(echo 'func')
In func
1 source /dev/fd/63
```

### Real bash_completion example
```bash
$ cat /usr/share/bash-completion/completions/git
# (Thousands of lines of completions — see how they handle subcommands)
```

### Checking for dangerous eval patterns in system scripts
```bash
$ grep -rn 'eval' /etc/bash.bashrc /etc/profile /etc/profile.d/ 2>/dev/null
```

## Check Your Understanding

1. What is the difference between `source file.sh` and `bash file.sh` in terms of process creation and variable visibility?

2. How does `eval` differ from normal command execution? What happens to a double-quoted string inside eval?

3. Why is `declare -n` safer than `eval` for creating variable references?

4. How would you write a plugin loader that checks file integrity (SHA256) before sourcing?

5. What does `declare -f` do with functions that contain syntax errors? Can you see the error?

6. How does programmable completion work? List the key variables bash sets before calling `_mycmd_complete`.

7. What security risks exist in sourcing files from a world-writable directory? How would a TOCTOU attack work?

8. How can you list all variables that have been marked readonly? Use `declare -p` and filter.

9. What is the difference between `${!var}` and `declare -n ref=$var`? When would you use one over the other?

10. How would you debug a script that sources many plugins and you're not sure which plugin redefined a function?
