# AWK Lesson 07: Numeric & Format Functions

AWK is great for numeric calculations and formatted reporting.

## printf Format Specifiers

`printf` gives you precise control over output formatting:

| Specifier | Meaning |
|-----------|---------|
| `%s`      | String |
| `%d`      | Decimal integer |
| `%f`      | Floating point number |
| `%8.2f`   | Float, width 8, 2 decimal places, right-aligned |
| `%-10s`   | String, width 10, left-aligned |
| `%05d`    | Integer, width 5, zero-padded |

### Example 1: Formatted Table

```bash
awk '{
    printf "%-10s %8.2f %5d\n", $2, $4, $5
}' file.csv
```

Left-aligns column 2 (10 chars), right-aligns column 4 (8 chars, 2 decimals), right-aligns column 5 (5 digits).

## sprintf()

`sprintf()` works like `printf()` but RETURNS the string instead of printing it.

```bash
awk '{ formatted = sprintf("%-10s %8.2f", $1, $2); print formatted }' file.csv
```

## Arithmetic

Standard operators: `+`, `-`, `*`, `/`, `%`, `^` (exponentiation)

```bash
awk '{ total = $4 * $5; print $1, total }' file.csv
```

## rand() and srand()

- `rand()` returns a random float between 0 and 1
- `srand([seed])` seeds the random number generator

```bash
awk 'BEGIN { srand(); print int(rand() * 100) }'
```

---

## Your Task

File: `target.txt` (sales.csv - pipe-delimited)

Format: `ProductID|ProductName|Category|Price|Quantity|Date`

Write an AWK command that generates a formatted sales report:

1. Print header: `Product`, `Qty`, `Price`, `Total` (appropriately aligned)
2. For each line, calculate `Total = Price * Quantity`
3. Print each row with:
   - Product name: left-aligned, 20 characters wide (`%-20s`)
   - Quantity: right-aligned, 5 digits (`%5d`)
   - Unit Price: right-aligned, 8 chars with 2 decimals (`%8.2f`)
   - Total: right-aligned, 10 chars with 2 decimals (`%10.2f`)
4. After all lines, print a separator, then:
   - `GRAND TOTAL` with the sum of all totals
   - Average price per item (`sum_prices / count`)

Expected format:

```
Product                Qty    Price      Total
-------------------- ----- -------- ----------
Doohickey Delta        478   199.06   95150.68
Utensil Omicron         48   375.66   18031.68
...
-------------------- ----- -------- ----------
GRAND TOTAL                          1234567.89
Average price per item: 234.56
```
