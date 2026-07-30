# Lesson 05: Transform with `y///`

## The `y///` Command (Transliterate)

The `y///` command translates characters one-by-one, similar to `tr`:

```bash
sed 'y/abc/ABC/' file.txt
```

This replaces every `a` with `A`, every `b` with `B`, and every `c` with `C`.

## `y///` vs `s///`

| Feature | `s///` | `y///` |
|---------|--------|--------|
| Operation | Pattern matching & replacement | Character-by-character mapping |
| Flags | `g`, `I`, `p`, `w`, etc. | None |
| Regex | Supports regex patterns | Literal characters only |
| Partial replacement | Yes, with regex | No — all listed chars are replaced |
| Length | Old and new can differ | Sets must be equal length |

```bash
# s/// — swap "cat" with "dog"
echo "cat" | sed 's/cat/dog/'

# y/// — shift case using character mapping
echo "cat" | sed 'y/cat/DOG/'  # c→D, a→O, t→G → DOG
```

## Common Use Cases for `y///`

Converting case:
```bash
sed 'y/ABCDEFGHIJKLMNOPQRSTUVWXYZ/abcdefghijklmnopqrstuvwxyz/' file.txt  # Upper → lower
sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/' file.txt  # Lower → upper
```

Fixing line endings:
```bash
sed 'y/\r\n/\n\r/' dosfile.txt  # Swap CR/LF
```

## Examples

### Example 1: Convert specific characters
```bash
$ echo "Hello World" | sed 'y/aeiou/AEIOU/'
HEllO WOrld
```

### Example 2: Make everything uppercase
```bash
$ echo "hello world" | sed 'y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/'
HELLO WORLD
```

### Example 3: Comparing `s///` and `y///`
```bash
$ echo "mississippi" | sed 's/s/S/'    # First s only → miSsisippi
$ echo "mississippi" | sed 's/s/S/g'   # All s → miSSiSSippi
$ echo "mississippi" | sed 'y/s/S/'    # All s → miSSiSSippi (same as g flag)
$ echo "mississippi" | sed 'y/isp/ISP/' # i→I, s→S, p→P → mISSIttISIPPI
```

## Task

Using **`target.txt`** (transactions.csv, 3000 lines, pipe-delimited):

1. Use `y///` to convert **all lowercase letters** in the file to uppercase
2. Use `s///` to replace all pipe `|` delimiters with commas `,`

The result should be a comma-separated file (CSV) with all uppercase content.

**Hint:** For the `y///` command, you'll need to map all 26 lowercase letters to uppercase: `y/abcdefghijklmnopqrstuvwxyz/ABCDEFGHIJKLMNOPQRSTUVWXYZ/`

**Expected result:** A CSV file where every character is uppercase and pipes are replaced with commas. Example: `2024-04-07 00:00:00|bob|WRITE|/home/user/.ssh/id_rsa|ALLOW` → `2024-04-07 00:00:00,BOB,WRITE,/HOME/USER/.SSH/ID_RSA,ALLOW`
