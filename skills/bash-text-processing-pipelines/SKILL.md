---
name: bash-text-processing-pipelines
description: Chain Unix text filters — cut, sort, uniq, tr, paste, join, wc, head/tail, xargs — with grep/sed regular expressions and awk field processing to extract, reshape, and summarize text without writing a general-purpose program. Trigger for "parse this log file," "extract a column from this output," "count unique occurrences," "how do I use awk/sed/grep for this," "join two files on a common field," "sort by the Nth column," "regex in bash," or any request to build a shell pipeline over structured or semi-structured text. Companion skills: bash-defensive-scripting-fundamentals (quoting/IFS when wiring pipeline output into script logic), bash-cli-and-automation-scripting (wrapping a pipeline in a script with options).
---

# Bash Text Processing Pipelines

Most "write me a script that parses this" requests are really "chain the right filters together in the
right order." Before reaching for a general-purpose loop with `read`, check whether `cut`, `sort`,
`uniq`, `join`, `grep`, `sed`, or `awk` already does the job in one line — they're faster, better-tested,
and handle edge cases (encoding, huge files, field counting) that hand-rolled parsing usually gets
wrong.

## Column and record utilities

```bash
cut -d',' -f1,3 data.csv          # -d sets the delimiter, -f selects fields by number
cut -c1-10 data.txt               # character-position slice instead of field-based
paste -d',' names.txt scores.txt  # merge files line-by-line into columns, -d sets separator
join file1.csv file2.csv          # merge on a shared key field — BOTH files must be sorted on it first
nl file.txt                       # number lines (nl -ba numbers non-blank lines only)
wc -l file.txt                    # line count; wc -w words, wc -c bytes, wc -L longest-line length
head -n 20 file.txt                # first N lines
tail -n 20 file.txt                # last N lines
tail -f /var/log/app.log           # follow a growing file — the standard way to watch live logs
tr 'a-z' 'A-Z' < file.txt          # translate characters (not lines) — has no concept of "field"
tr -s ' ' < file.txt               # squeeze repeated spaces into one, useful before cut -d' '
```

`join` is unforgiving about sort order: both inputs must already be sorted on the join field (`sort` the
same way, on the same field, before calling `join`), or it errors out with `is not sorted` instead of
silently producing wrong output — that fail-loud behavior is a feature, don't work around it by
disabling the check.

`cut -f` and `awk '{print $N}'` overlap in purpose but differ in what "field" means: `cut` splits on a
single fixed delimiter character (`-d`), while `awk`'s default field splitting collapses runs of
whitespace into one separator — for whitespace-padded columnar output (`ps aux`, `df`, `ls -l`), `awk`
handles the ragged spacing correctly where `cut -d' '` does not.

## Sorting and de-duplicating

```bash
sort file.txt                     # lexical sort, whole line
sort -n file.txt                  # numeric sort — REQUIRED for numbers, lexical sort puts 10 before 2
sort -k2,2 file.txt               # sort on field 2 only (fields are 1-indexed, whitespace-delimited by default)
sort -t',' -k3,3n file.txt        # -t sets delimiter, -k3,3n sorts field 3 numerically
sort -k1,3 -k5n file.txt          # multiple sort keys: fields 1-3 lexically, then field 5 numerically as tiebreak
sort -u file.txt                  # sort and de-duplicate in one pass
uniq file.txt                     # collapse ADJACENT duplicate lines only — input MUST be sorted first
uniq -c file.txt                  # prefix each line with its count of consecutive occurrences
uniq -d file.txt                  # show only lines that had duplicates
```

The single most common `uniq` bug: it only removes duplicates that are already next to each other in
the input. `uniq` on unsorted data silently leaves duplicates in place because they're not adjacent —
always `sort | uniq` (or `sort -u`), never `uniq` alone on data you haven't sorted.

## `grep` and regular expressions

```bash
grep 'pattern' file.txt           # basic literal/BRE search
grep -v 'pattern' file.txt        # invert — show lines that DON'T match
grep -c 'pattern' file.txt        # count of matching LINES (not total match occurrences)
grep -i 'pattern' file.txt        # case-insensitive
grep -r 'pattern' dir/            # recurse into a directory
grep -E '(cron|yum)' file.txt     # extended regex — alternation, +, ?, {} without backslashing
grep -F 'literal.string' file.txt # fixed-string match — treats the pattern as literal text, no regex at all
```

`grep` (basic regex) needs `\+`, `\?`, `\|`, `\(...\)` backslash-escaped to get their special meaning;
`grep -E` (extended regex, same engine `egrep` used) takes them unescaped — pick one form and stay
consistent within a script rather than mixing escaped and unescaped metacharacters. When the search
string itself contains regex metacharacters you want treated literally (a literal `.` in an IP address, a
literal `$` in a price), use `grep -F`/`fgrep` instead of trying to escape every special character by hand.

## `sed`: stream editing

```bash
sed 's/old/new/' file.txt         # replace FIRST occurrence per line
sed 's/old/new/g' file.txt        # replace ALL occurrences per line (g = global)
sed -n '2p' file.txt              # -n suppresses default output; print only line 2
sed '3,5d' file.txt               # delete lines 3 through 5
sed '/pattern/d' file.txt         # delete lines matching pattern
sed -i 's/old/new/g' file.txt     # -i edits the file IN PLACE — no output, original is overwritten
sed -i.bak 's/old/new/g' file.txt # -i.bak keeps a backup before editing in place (GNU vs BSD syntax differs — see bash-script-debugging-and-portability)
sed '5q' file.txt                 # quit after line 5 — sed's version of `head -n 5`
```

`sed -i` with no backup suffix destroys the original with no undo — always test the substitution
without `-i` first (or with `-i.bak`) until you're sure the pattern matches what you think it matches.
GNU sed and BSD/macOS sed disagree on the `-i` syntax (GNU accepts `-i` alone, BSD requires an
argument even if empty: `-i ''`) — a portable script should avoid depending on the difference.

## `awk`: field-aware text processing

```bash
awk '{print $1}' file.txt                     # print field 1 (default: split on whitespace)
awk -F: '{print $1}' /etc/passwd              # -F sets the field separator (colon here)
awk '$3 > 100 {print $1, $3}' data.txt        # pattern (condition) + action — only matching lines print
awk '/ERROR/ {print}' app.log                 # regex pattern as the condition, no explicit action needed to print the line
awk '$9 ~ /^5/ {print $0}' access.log         # ~ is the regex match operator; $9 is field 9
awk 'NR > 1 {print}' file.csv                 # NR = current record (line) number — skip a header row
awk 'END {print NR}' file.txt                 # END block runs once after all input — total line count
awk 'BEGIN {FS=","} {print $2}' file.csv      # BEGIN block runs once before input — set up FS, counters
```

`awk` programs are `pattern { action }` pairs: a bare pattern with no action defaults to printing the
whole line (`$0`); a bare action with no pattern (`{ print $1 }`) runs on every line. `BEGIN` and `END`
blocks bracket one-time setup and final summarization around the per-line processing, which is what
makes `awk` a full mini-language rather than just a `cut` replacement:

```bash
# Running total and average — BEGIN initializes, per-record accumulates, END reports:
awk '
BEGIN { sum = 0; count = 0 }
{ sum += $1; count++ }
END { printf "Average: %.2f\n", sum / count }
' numbers.txt
```

```bash
# Associative arrays for counting occurrences — the awk equivalent of `sort | uniq -c`,
# but in a single pass and without requiring sorted input:
awk '{ count[$1]++ } END { for (key in count) print key, count[key] }' access.log
```

`printf` inside `awk` (and in bash itself) gives control `print` doesn't: `%-11s` left-justifies a string
in an 11-character field, `%.2f` fixes a float to two decimals — reach for `printf` the moment output
needs to line up in columns or format numbers, rather than fighting `print`'s automatic spacing.

## `xargs`: turning a list into arguments

```bash
find . -name '*.log' | xargs rm              # convert each line of input into an argument to rm
find . -name '*.log' -print0 | xargs -0 rm   # -print0/-0 use NUL separators — safe with filenames containing spaces/newlines
echo file1 file2 | xargs -n1 echo Processing # -n1 runs the command once per argument instead of batching
find . -name '*.txt' | xargs -I{} mv {} {}.bak  # -I{} lets you place the argument anywhere in the command
```

Piping filenames into `xargs` without `-print0`/`-0` breaks on filenames containing spaces or
newlines, because `xargs` (like unquoted bash) splits on whitespace by default — for anything reading
`find` output, pair `-print0` with `xargs -0` as a matched set.

## Common pitfalls

- Running `uniq` on data that was never sorted, and getting duplicates in the output because they
  weren't adjacent.
- Using `sort` (default lexical order) on a numeric column and getting `1, 10, 11, 2, 3` instead of
  numeric order — always add `-n` (or `-k<N>n` for a specific numeric field).
- Calling `join` on two files that aren't sorted identically on the join field — it errors instead of
  guessing, which is correct behavior, not a bug to route around.
- Using `cut -d' '` on whitespace-padded command output (`ps`, `df`, `ls -l`) where repeated spaces
  shift field numbers — use `awk '{print $N}'` instead, which treats runs of whitespace as one
  separator.
- Forgetting that `sed -i` overwrites the file immediately with no confirmation — test the pattern
  without `-i` first.
- Mixing basic and extended regex syntax in the same `grep` invocation (escaped `\+` with `-E`, or
  unescaped `+` without `-E`) and being confused when the match count is wrong.
- Feeding `xargs` a list of filenames without `-print0`/`-0` and having it mis-split filenames that
  contain spaces.

## Checklist

- [ ] `sort` uses `-n` (or a numeric `-k` flag) whenever the sort key is numeric, not left at the
      lexical default
- [ ] `uniq` is always preceded by `sort` (or replaced with `sort -u`) unless adjacency is already
      guaranteed
- [ ] `join` inputs are sorted identically on the join field before the call
- [ ] `awk` is used instead of `cut -d' '` for whitespace-ragged columnar command output
- [ ] `sed -i` edits are tested without `-i` (or with a `.bak` suffix) before trusting them on real data
- [ ] Regex flavor (`grep` BRE vs `grep -E` ERE vs `grep -F` literal) is chosen deliberately, not mixed
- [ ] `find ... -print0 | xargs -0 ...` used instead of unguarded newline splitting when filenames might
      contain spaces
