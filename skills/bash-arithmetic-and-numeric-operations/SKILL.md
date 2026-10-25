---
name: bash-arithmetic-and-numeric-operations
description: Do math in bash correctly — bash's native integer-only arithmetic with $(( )) and declare -i, why `expr` needs escaped operators and spaces, and reaching for `bc` (interactive, program files, or piped) the moment a calculation needs decimals, exponents, or a different number base. Trigger for "bash math doesn't work," "why did echo 1+1 print 1+1," "floating point in bash," "how do I do division with decimals in a shell script," "convert hex to decimal in bash," "modulus in bash," "bc scale," or any arithmetic that a shell script needs to perform on values. Companion skills: bash-text-processing-pipelines (piping extracted numeric columns into bc), bash-defensive-scripting-fundamentals (quoting when building the expression from variables).
---

# Bash Arithmetic and Numeric Operations

Bash treats everything as a string by default, and it only knows how to do *integer* math even when
you explicitly ask it to compute — every one of the mistakes below comes from forgetting one of those
two facts.

## Why `echo 1+1` doesn't compute anything

```bash
echo 1+1        # prints "1+1" — echo has no idea this is meant to be arithmetic
echo $((1+1))   # prints "2" — $(( )) is bash's arithmetic-expansion syntax
```

`$(( ))` is the standard, idiomatic way to do integer arithmetic directly in bash — no external command
needed, and it understands variable names inside the expression without a leading `$`:

```bash
val1=10
val2=3
echo $((val1 + val2))        # 13 — addition
echo $((val1 - val2))        # 7  — subtraction
echo $((val1 * val2))        # 30 — multiplication, no backslash-escaping needed inside $(( ))
echo $((val1 / val2))        # 3  — integer division, truncates toward zero (NOT rounds)
echo $((val1 % val2))        # 1  — modulus (remainder)
echo $(((val1 + val2) * 2))  # 26 — parentheses group normally, no escaping needed
result=$((val1 * val2))      # assign the result directly, same as any command substitution
```

`((expr))` (double parens as a standalone command, no `$`) is the arithmetic equivalent of `[[ ]]` for
tests and for updating a variable in place — commonly used in loop counters and conditionals:

```bash
((count++))                  # increment — no $ needed for a bare arithmetic command
((count += 5))
if ((val1 > val2)); then echo "val1 wins"; fi   # arithmetic comparison, no -gt/-lt needed here
```

## `expr`: the older, pickier way

```bash
expr 1 + 1          # 2 — spaces around the operator are REQUIRED
expr 1+1            # "1+1" literally — no spaces means expr just echoes the input back unevaluated
expr 3 \* 3         # 9 — * must be backslash-escaped or the shell glob-expands it against filenames first
expr 3 * 3          # BREAKS — the shell expands * into a list of filenames in the current directory
                     # before expr ever sees it, producing a nonsense "too many arguments" error
expr \( 1 + 2 \) \* 3   # 9 — parentheses also need escaping, with spaces around them too
```

`$(( ))` has superseded `expr` for nearly all everyday integer arithmetic in bash — it doesn't require
escaping `*`/`(`/`)`, doesn't fork a separate process the way `expr` does, and reads more naturally.
`expr` still shows up in older scripts and POSIX `/bin/sh` contexts where `$(( ))` support can't be
assumed (see `bash-script-debugging-and-portability`), so recognize it, but write new integer math
with `$(( ))`.

## `declare -i`: integer-typed variables

```bash
declare -i result
result=val1+val2         # no $(( )) needed once the variable is declared -i — bash evaluates it as arithmetic
declare -i a=1 b=2 sum=a+b
echo "$sum"               # 3
```

`declare -i` makes bash treat every assignment to that variable as an arithmetic expression rather than
a string, including references to *other* variables by bare name inside the expression. It's a
convenience for scripts doing a lot of arithmetic on the same handful of variables, but it's easy to
forget a variable was declared this way and be surprised when assigning it a non-numeric string either
errors or silently becomes `0` — `$(( ))` at each call site is usually clearer to a reader than a `declare -i`
several lines above.

## The integer-only trap, and dividing without losing the remainder

```bash
echo $((7 / 2))     # 3, not 3.5 — bash arithmetic truncates, it never produces a fraction
echo $((-7 / 2))     # -3 — truncation is toward zero, not toward negative infinity (not a floor division)
```

Every one of `$(( ))`, `((...))`, `expr`, and `declare -i` shares this limitation: **bash cannot represent a
non-integer result, ever.** There is no bash-native way to get `3.5` out of `7/2` — the moment a
calculation needs a fractional result, an exponent, a trig/log function, or a different rounding rule,
bash arithmetic is the wrong tool and `bc` (or `awk`, for numbers already inside a pipeline — see
`bash-text-processing-pipelines`) is the right one.

## `bc`: floating point, precision, and number-base conversion

```bash
echo "3/4" | bc -l           # .75000000000000000000 — -l loads the math library AND enables decimals
echo "scale=2; 3/4" | bc -l  # .75 — scale sets the number of decimal places shown
```

The `-l` flag matters even for a division that "looks" like it should just work: `bc` without `-l`
truncates like bash integer division does (`3/4` gives `0`), because the extended math library that
enables floating point is opt-in, not a default.

```bash
# Variables, expressions, and print (bc's echo) work naturally in bc's own language:
echo "pi=3.14159265358979; diameter=25; print pi*diameter" | bc -l

# Piping a whole block of bc statements at once (heredoc keeps it readable for anything non-trivial):
bc -l << 'EOF'
scale=2
radius=12.5
area = 3.14159265358979 * (radius^2)
print area, "\n"
EOF
```

```bash
# Number base conversion — ibase/obase, decimal is the default for both if not set:
echo "obase=16; 10" | bc          # A       — decimal to hex
echo "ibase=16; A" | bc           # 10      — hex to decimal (hex digits A-F MUST be upper-case)
echo "ibase=2; obase=16; 1011101" | bc   # 5D — binary to hex
```

Setting `ibase` changes how bc *reads* every subsequent number, including the value you assign to
`obase` itself — set `obase` before `ibase` when changing both, or a later `obase=10` gets
misinterpreted using the new `ibase` and doesn't do what you expect.

## Pulling numbers out of a pipeline and into `bc`

The pattern that ties text processing and arithmetic together: extract a numeric column, then sum or
compute on it without ever loading the values into bash variables one at a time.

```bash
# Sum a numeric column extracted from a CSV: cut the field, paste -s -d+ joins every line with a "+"
# between them producing one long addition expression, then bc evaluates it:
grep 'Windows' market_share.csv | cut -d, -f2 | paste -s -d+ | bc
```

`paste -s -d+` (serial mode, `+` as the delimiter) turns a column of numbers on separate lines into a
single `1.2+3.4+5.6`-style expression — a one-line alternative to an `awk '{sum+=$1} END{print sum}'`
loop when the numbers are already isolated in their own column.

## Common pitfalls

- Trying `echo $val1 + $val2` and getting the literal text `10 + 3` back instead of `13` — `echo` never
  evaluates arithmetic; wrap the expression in `$(( ))`.
- Using `expr 3 * 3` without escaping the `*` and getting a confusing "too many arguments" error
  because the shell glob-expanded `*` against filenames before `expr` ran.
- Assuming `$((7/2))` or `expr 7 / 2` will produce `3.5` — bash and `expr` are integer-only, unconditionally.
- Piping into plain `bc` (no `-l`) and getting a truncated integer-looking result for what should be a
  decimal division.
- Setting `ibase` before `obase` when both need to change, then having the `obase` assignment
  misread using the already-changed `ibase`.
- Typing lower-case hex digits (`a`–`f`) into `bc` when converting *into* a number — `bc` requires
  upper-case `A`–`F` for hex digit input.
- Forgetting that a `declare -i` variable silently coerces non-numeric assignments (often to `0`) instead
  of raising an obvious string-vs-number type error.

## Checklist

- [ ] Integer arithmetic in new code uses `$(( ))` or `((...))`, not `expr`, unless the script must run
      under a shell without arithmetic-expansion support
- [ ] Any `*`, `(`, or `)` passed to `expr` is backslash-escaped and separated by spaces from operands
- [ ] Any calculation that could produce a non-integer result is routed through `bc -l` (or `awk`), never
      assumed to "just work" in bash's native integer arithmetic
- [ ] `bc` invocations that need decimal output include `-l` and set `scale` explicitly if the default
      precision is wrong for the use case
- [ ] `obase` is set before `ibase` when both are being changed in the same `bc` session
- [ ] Hex digits fed into `bc` as input are upper-case
