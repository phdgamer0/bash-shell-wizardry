# Task 15: Regex Validator & Tester

Build a script that validates common formats using regex and extracts data with BASH_REMATCH. This is a practical tool you'll use regularly.

## Steps

### Sub-task 1: Skeleton Validator Script
Create `~/validate.sh` that accepts an input type flag and a value:

```bash
#!/bin/bash
# validate.sh — Regex-based input validator

usage() {
    cat <<EOF
Usage: $0 --email <address>
       $0 --ip <address>
       $0 --phone <number>
       $0 --alphanum <string>
       $0 --date <YYYY-MM-DD>
       $0 --test <pattern> <string>
EOF
    exit 2
}

if [ $# -lt 2 ]; then
    usage
fi

mode="$1"
value="$2"

case "$mode" in
    --email)     validate_email "$value" ;;
    --ip)        validate_ip "$value" ;;
    --phone)     validate_phone "$value" ;;
    --alphanum)  validate_alphanum "$value" ;;
    --date)      validate_date "$value" ;;
    --test)      test_pattern "$2" "$3" ;;
    *)           echo "Unknown mode: $mode" >&2; usage ;;
esac
```

**Approach 1:** `case` statement for mode dispatch
**Approach 2:** Functions per mode
**Approach 3:** Associative array mapping modes to functions

<details><summary>Hint: Argument parsing</summary>
`$1` is the mode flag, `$2` is the value. For `--test`, `$2` is the pattern and `$3` is the string. Use `shift` to consume the mode.
</details>

### Sub-task 2: Implement Email Validator
```bash
validate_email() {
    local email="$1"
    local pattern='^([a-zA-Z0-9._%+-]+)@([a-zA-Z0-9.-]+)\.([a-zA-Z]{2,})$'

    if [[ "$email" =~ $pattern ]]; then
        echo "Email: $email"
        echo "  Username: ${BASH_REMATCH[1]}"
        echo "  Domain:   ${BASH_REMATCH[2]}"
        echo "  TLD:      ${BASH_REMATCH[3]}"
        echo "  VALID"
    else
        echo "Email: $email"
        echo "  INVALID"
    fi
}
```

**Approach 1:** Simple regex as shown
**Approach 2:** More permissive: `^.+@.+$`
**Approach 3:** Stricter: check TLD length, no consecutive dots

Test cases:
```bash
$ ~/validate.sh --email "user@example.com"
Email: user@example.com
  Username: user
  Domain:   example
  TLD:      com
  VALID

$ ~/validate.sh --email "not-an-email"
Email: not-an-email
  INVALID

$ ~/validate.sh --email "user@.com"
Email: user@.com
  INVALID
```

<details><summary>Hint: Email regex limitations</summary>
No single regex can validate all valid email addresses (RFC 5322 is incredibly complex). This is a "good enough" validation. Real email validation requires sending a verification email.
</details>

### Sub-task 3: Implement IP Validator
```bash
validate_ip() {
    local ip="$1"
    local pattern='^([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})\.([0-9]{1,3})$'

    if [[ "$ip" =~ $pattern ]]; then
        local valid=true
        for i in 1 2 3 4; do
            local octet="${BASH_REMATCH[$i]}"
            if [ "$octet" -gt 255 ]; then
                echo "  Octet $i: $octet (INVALID > 255)"
                valid=false
            else
                echo "  Octet $i: $octet (OK)"
            fi
        done
        $valid && echo "  VALID" || echo "  INVALID"
    else
        echo "IP: $ip"
        echo "  INVALID — bad format"
    fi
}
```

Test cases:
```bash
$ ~/validate.sh --ip "192.168.1.1"
IP: 192.168.1.1
  Octet 1: 192 (OK)
  Octet 2: 168 (OK)
  Octet 3: 1 (OK)
  Octet 4: 1 (OK)
  VALID

$ ~/validate.sh --ip "192.168.300.1"
IP: 192.168.300.1
  Octet 1: 192 (OK)
  Octet 2: 168 (OK)
  Octet 3: 300 (INVALID > 255)
  INVALID
```

**Approach 1:** Regex + numeric check as shown
**Approach 2:** Pure regex with range checking (very verbose): `^(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.` etc.
**Approach 3:** Use `ipcalc` or `ifconfig` to validate (external tools)

### Sub-task 4: Phone Number Validator
```bash
validate_phone() {
    local phone="$1"
    # Match (555) 123-4567 or 555-123-4567
    local pattern='^\(?([0-9]{3})\)?[-. ]?([0-9]{3})[-. ]?([0-9]{4})$'

    if [[ "$phone" =~ $pattern ]]; then
        echo "Phone: $phone"
        echo "  Area code: ${BASH_REMATCH[1]}"
        echo "  Exchange:  ${BASH_REMATCH[2]}"
        echo "  Line:      ${BASH_REMATCH[3]}"
        echo "  VALID"
    else
        echo "Phone: $phone"
        echo "  INVALID"
    fi
}
```

**Approach 1:** Handle multiple formats with `?` (optional `(`, `)`, `-`, `.`, space)
**Approach 2:** Strip non-digits first: `digits=$(tr -dc '0-9' <<< "$phone")` then check length
**Approach 3:** Use separate patterns for each format

Test:
```bash
$ ~/validate.sh --phone "(555) 123-4567"
Phone: (555) 123-4567
  Area code: 555
  Exchange:  123
  Line:      4567
  VALID

$ ~/validate.sh --phone "555-123-4567"
Phone: 555-123-4567
  Area code: 555
  Exchange:  123
  Line:      4567
  VALID

$ ~/validate.sh --phone "555-ABC-4567"
Phone: 555-ABC-4567
  INVALID
```

### Sub-task 5: Alphanumeric Check
```bash
validate_alphanum() {
    local str="$1"
    local pattern='^[a-zA-Z0-9_]+$'

    if [[ "$str" =~ $pattern ]]; then
        echo "Alphanumeric: $str"
        echo "  PASS"
    else
        echo "Alphanumeric: $str"
        echo "  FAIL — contains non-alphanumeric characters"
        # Show which characters fail
        local bad="${str//[a-zA-Z0-9_]/}"
        echo "  Invalid chars: $(echo -n "$bad" | sed 's/\(.\)/\1 /g')"
    fi
}
```

### Sub-task 6: Interactive Regex Tester
```bash
# If no arguments, enter interactive mode
if [ $# -eq 0 ]; then
    echo "=== Interactive Regex Tester ==="
    echo "Enter an empty pattern to quit."
    echo

    while true; do
        read -p "Pattern: " -r pat
        [ -z "$pat" ] && break
        read -p "String:  " -r str

        if [[ "$str" =~ $pat ]]; then
            echo "  MATCH"
            echo "  Full match: ${BASH_REMATCH[0]}"
            for i in "${!BASH_REMATCH[@]}"; do
                [ "$i" -eq 0 ] && continue
                echo "  Group $i: ${BASH_REMATCH[$i]}"
            done
        else
            echo "  NO MATCH"
        fi
        echo
    done
    exit 0
fi
```

**Approach 1:** `while true` loop as shown
**Approach 2:** `select` menu for mode selection
**Approach 3:** Readline-like with history (requires `rlwrap`)

### Sub-task 7: Date Validator
```bash
validate_date() {
    local date_str="$1"
    local pattern='^([0-9]{4})-([0-9]{2})-([0-9]{2})$'

    if [[ "$date_str" =~ $pattern ]]; then
        year="${BASH_REMATCH[1]}"
        month="${BASH_REMATCH[2]}"
        day="${BASH_REMATCH[3]}"

        # Basic range checks
        if [ "$month" -lt 1 ] || [ "$month" -gt 12 ]; then
            echo "Date: $date_str — INVALID (month out of range)"
            return 1
        fi

        if [ "$day" -lt 1 ] || [ "$day" -gt 31 ]; then
            echo "Date: $date_str — INVALID (day out of range)"
            return 1
        fi

        # Could add month-specific day limits here
        echo "Date: $date_str"
        echo "  Year: $year"
        echo "  Month: $month"
        echo "  Day: $day"
        echo "  VALID"
    else
        echo "Date: $date_str — INVALID (bad format, use YYYY-MM-DD)"
    fi
}
```

### Sub-task 8: Full Integration Test
Test all validators with edge cases:

```bash
$ ~/validate.sh --email ""
Email:
  INVALID

$ ~/validate.sh --email "user@"
Email: user@
  INVALID

$ ~/validate.sh --ip "0.0.0.0"
IP: 0.0.0.0
  Octet 1: 0 (OK)
  ...
  VALID

$ ~/validate.sh --ip "256.0.0.0"
IP: 256.0.0.0
  Octet 1: 256 (INVALID > 255)
  INVALID

$ ~/validate.sh --phone ""
Phone:
  INVALID

$ ~/validate.sh --test '^[a-z]+$' 'hello123'
Pattern: ^[a-z]+$
String:  hello123
Match:   NO

$ ~/validate.sh --test '^([a-z]+)([0-9]+)$' 'hello123'
Pattern: ^([a-z]+)([0-9]+)$
String:  hello123
Match:   YES
Groups:
  [0] hello123
  [1] hello
  [2] 123
```

## Expected Output

```bash
$ ~/validate.sh --email "user@example.com"
Email: user@example.com
  Username: user
  Domain:   example.com
  TLD:      com
  VALID

$ ~/validate.sh --ip "192.168.300.1"
IP: 192.168.300.1
  Octet 1: 192 (OK)
  Octet 2: 168 (OK)
  Octet 3: 300 (INVALID > 255)
  INVALID

$ ~/validate.sh --phone "555-123-4567"
Phone: 555-123-4567
  Area code: 555
  Exchange:  123
  Line:      4567
  VALID

$ ~/validate.sh --test '^[a-z]+$' 'hello123'
Pattern: ^[a-z]+$
String:  hello123
Match:   NO

$ ~/validate.sh --test '^([a-z]+)([0-9]+)$' 'hello123'
Pattern: ^([a-z]+)([0-9]+)$
String:  hello123
Match:   YES
Groups:
  [0] hello123
  [1] hello
  [2] 123
```

## Self-Check Questions

1. Why does `[[ "abc" =~ "[a-z]" ]]` fail to match?

2. What is in `${BASH_REMATCH[0]}` vs `${BASH_REMATCH[1]}`?

3. How do you make a `=~` match case-insensitive?

4. Does bash support `\d` like Perl regex? If not, what should you use?

5. What happens to `BASH_REMATCH` between two `=~` operations?

6. How would you validate that an octet is 0-255 using regex alone (without numeric comparison)?

7. Why can't a single regex perfectly validate all email addresses?
