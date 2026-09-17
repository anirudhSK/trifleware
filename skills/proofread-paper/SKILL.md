---
name: proofread-paper
description: Proofread a technical paper (computer systems / networking / architecture) before submission. Use when asked to proofread, copy-edit, or do a pre-submission or pre-deadline check on a paper draft; when checking a LaTeX paper for grammar, dangling references, missing citations, leaked review annotations, figure/number mismatches, or bibliography problems; or when asked "does this read right?" about a paper PDF. Reads the rendered PDF rather than the .tex source, runs a set of mechanical sweeps that catch classes of error reading cannot, fixes what is safe to fix, and reports the rest.
---

# Proofreading a systems paper

Goal: catch the errors a reviewer will notice in the first five minutes — unfinished
sentences, `??` references, annotations that leaked into the body, figure numbers that
point at the wrong figure, uncited systems, numbers that disagree with their own figure.

## The core rule: read the rendered PDF, not the source

**Proofread `pdftotext` output from the built PDF. Never proofread the `.tex` files.**

Source-level reading systematically misses whole classes of error, because each line
parses as well-formed on its own. Real examples that survived full source-level passes:

- A live review annotation. A `\claude{...}` macro sat at the end of a paragraph in the
  design section and **rendered as body text** — several hundred words of reviewer
  commentary in the middle of the paper. In the source it looks like every other
  annotation macro in the file, and the file was full of them. The difference is that all
  the others were behind a `%`. Only the PDF shows which ones are live.

- A missing space. The source read:

  ```latex
  ... alongside IPC on the CPU host.%\xx{I commented out the next sentence...}
  %The \sysname encoding scheme ...
  We had to address two main challenges ...
  ```

  The trailing `%` eats the newline, so the PDF reads **"on the CPU host.We had to
  address"**. Both source lines are individually correct; only the rendered output is wrong.

- A sentence with no main clause, from a half-commented paragraph:

  ```latex
  We use 8 cores to run the software baseline. Our results, shown in
  % To demonstrate \sysname's effectiveness ... (dead draft text)
  Figure~\ref{...} and~\ref{...}, \sysname outperforms its software implementation by ...
  ```

  Renders as "Our results, shown in Figures 14 and 15, \sysname outperforms ..." — a
  sentence with no main clause.

Corollary: **`\endinput` and `\begin{comment}` hide dead draft text that still looks
live.** Before editing any source line, check which side of an `\endinput` it is on, and
confirm the edit actually reached the PDF.

## Step 0 — Ground truth

```bash
make 2>&1 | tail -30          # or pdflatex→bibtex→pdflatex→pdflatex
pdfinfo main.pdf | grep Pages
```

Establish before reading:
- The venue's page limit and what counts (body only? do references/appendices count?).
- Which pages are body, which are references, which are appendix.
- Whether review-annotation macros are enabled (grep the macro file for
  `showcomments`/`showedits`-style booleans). If enabled, **any** un-commented
  annotation will print.

## Step 1 — Extract

Extract body, references, and appendix separately — they need different sweeps:

```bash
pdftotext -f 1  -l 11 main.pdf body.txt   # body pages
pdftotext -f 11 -l 14 main.pdf refs.txt   # bibliography
pdftotext -f 15 -l 15 main.pdf app.txt    # appendix
pdftotext            main.pdf all.txt     # everything, for greps
tr -s ' \n' ' ' < body.txt > flat.txt      # one-line form, for phrase greps
```

### The two-column trap — read this before flagging any broken sentence

`pdftotext` reads two-column pages and floats in **layout order**, not reading order. It
splices column ends, figure captions, axis tick labels, and table cells into the middle
of prose. So:

- An apparent sentence break at a column boundary or next to a figure is **almost always
  an extraction artifact**. Confirm against the source before editing.
- But a *genuine* break looks exactly the same. **The test:** are the two halves adjacent
  in the source with only comments between them? If yes, it is a real break. If there is
  intervening live text, it was an artifact.

Do not skip this check in either direction. Both false positives and false negatives here
are expensive.

## Step 2 — Read every sentence

The body of a 12-page paper is ~400–450 sentences: one sitting. Read them all. Reading
catches what greps cannot: non-sequiturs, a paragraph that stops mid-argument, a claim
the evaluation does not support, a definition used before it is given (use-before-define),
a "Bob" example that changes its own rules halfway through.

Note but do not yet fix. Collect into two lists (see Step 4).

## Step 3 — Mechanical sweeps

These catch what reading skates over. Each one below found a real bug in practice.

### Dangling and undefined references

```bash
grep -n "??" all.txt                                   # rendered undefined refs
grep -n "undefined\|Citation.*undefined" main.log | grep -v "Font shape"
```

`§??` in the body meant a `\S\ref{sec:discussion}` survived after the section was renamed
to `sec:limitations`. The build only warns; it does not fail. **Undefined-reference count
must be zero before submission.**

### Leaked review annotations

```bash
grep -nE "\[Claude|TODO|FIXME|XXX|▶|◀|\\\\todo|\\\\fixme" all.txt
grep -n "author{" main.tex          # placeholder author / paper number
```

Also grep the extracted text for each co-author's annotation-macro initials. And check
for submission placeholders: `\author{Paper \#XXX}` needs the real HotCRP number.

### Figure and subfigure numbering

Compare, for each cross-reference, the number the text names against the number of the
figure whose caption matches:

```bash
grep -o "Figure [0-9]*[ab]*" flat.txt | sort | uniq -c
grep -o "Figure [0-9]*: [^.]*\." all.txt          # caption -> number map
```

**Root cause worth knowing.** Four references read "Figure 2a"/"Figure 2b" while pointing
at content captioned "Figure 3". Cause: two figures shared one `figure*`, and in the
second minipage the subcaptions came *before* the parent `\caption`:

```latex
\begin{minipage}[b]{0.48\linewidth}
    \begin{subfigure}...\caption{...}\label{fig:sub_a}\end{subfigure}   % numbered vs. figure 2
    \begin{subfigure}...\caption{...}\label{fig:sub_b}\end{subfigure}
    \caption{...}\label{fig:parent}                                      % now increments to 3
\end{minipage}
```

Subcaptions format as `\thefigure` + letter, and `\thefigure` is still the *previous*
figure's number until the parent `\caption` runs. Fix, preserving layout exactly:

```latex
\refstepcounter{figure}\label{fig:parent}   % claim the number up front
  ... subfigures (now 3a, 3b) ...
\addtocounter{figure}{-1}
\caption{...}                                % re-increments to the same number
```

### Reference coverage, both directions

Every figure/table/section that exists must be referenced at least once, and every number
referenced must exist. An orphan float is as much a defect as a dangling ref.

```bash
# every Figure N in the PDF referenced somewhere in prose? (0 = orphan float)
for n in $(seq 1 22); do
  printf "Fig %s: %s\n" "$n" "$(grep -o "Figure[s]* $n[^0-9]" flat.txt | wc -l)"
done
grep -rn "\\\\label{fig:\|\\\\label{tab:\|\\\\label{sec:" tex/ | grep -v ":[0-9]*: *%"
```

Found this way: an **orphan appendix** that nothing in the body referenced, while the
implementation section's intro still promised the material that had been moved into it —
and a later section pointed at the old, now-empty location for it.

### Plural cross-references

```bash
grep -o "Figure [0-9]*[ab]* and Figure\|Figure [0-9]*[ab]* and [0-9]" flat.txt
grep -rn "Figure~\\\\ref{[^}]*} and~\\\\ref" tex/    # should be Figures~
```

Same for `Section`/`Table`.

### Named-but-uncited systems

Scan the prose for every product, protocol, or system name and confirm it carries a
citation at first mention. A single pass found three transport protocols named in running
text as if cited, none of which carried a `\cite`. Easy to miss because the surrounding
names *are* cited.

```bash
# capitalized words NOT immediately followed by a citation bracket
# (needs -P for the lookahead; -E does not support it)
grep -oP "\b[A-Z][A-Za-z]+(?:[A-Z][A-Za-z]*)*\b(?! ?\[)" flat.txt | sort -u | less
```

(Eyeball the list; a regex cannot decide what is a system name. It will be noisy —
sentence-initial words, section headings — but uncited systems stand out.)

### Use before define

Every abbreviation must be expanded at its *first* occurrence in reading order — and the
first occurrence is the one in the PDF, not the one in the file you happen to open. The
abstract and the introduction each define independently; after that, define once.

```bash
# first page each acronym appears on, vs. the page its expansion appears on
for a in $(grep -oE "\b[A-Z]{2,6}s?\b" flat.txt | sort -u); do
  printf "%-8s first=%s\n" "$a" "$(grep -n "$a" body.txt | head -1 | cut -d: -f1)"
done
```

Eyeball the list against where each expansion occurs. Real examples caught this way: OCI,
RNIC, SVM and OWD-CC all used in the evaluation with no expansion anywhere; NCCL used in
four sections while its only expansion sat in a draft file that `main.tex` no longer
inputs.

The same failure mode applies to a term the paper renamed halfway through: grep for both
the old and the new name and check which one the reader meets first.

### Numbers in prose vs. numbers in the figure they cite

For every sentence that cites a figure or table and states a number, open the figure and
check. Verify tables are internally self-consistent too (e.g. a claimed "adds 6 cycles per
X" against the table's own row: 15, 21, 33, 57, 105, 201, 297 → differences 6, 12, 24, 48,
96, 96 ✓).

Watch specifically for citation numbers baked *inside* figure PDFs disagreeing with the
bibliography — a figure whose legend hardcodes "[7]" against a system that the current
bibliography numbers `[6]`. Bibliography numbers shift every time a citation is added, so
any number rendered into a figure is stale by default. Figure internals cannot be fixed by
editing `.tex`; **report these**.

### Units and typography (NIST SP 811)

```bash
grep -oE "[0-9]+(GbE|Gbps|GB|MHz|ns|us|ms)" flat.txt     # missing value–unit space
grep -oE "[0-9]+ ?K(B|hz)?\b" flat.txt                    # K is kelvin; kilo is k
grep -oE "∼ [0-9]|\$\\\\sim[0-9]" flat.txt                # inconsistent tilde spacing
```

- Space between value and unit; correct prefix case (`k` kilo, `K` kelvin); no
  stand-alone prefixes ("4G tmpfs" → "4 GB").
- Distinguish bits (b) from bytes (B), and powers of 10 (MB) from 2 (MiB).
- Do not italicize units.
- Percent signs: follow venue convention, but be consistent within the paper.
- `Gbps`/`Mrps`/`krps` are not SI; keep networking spelling.
- **`savetrees` zeroes math-mode thin spaces**, so siunitx `\num{100000}` silently loses
  digit grouping. Check rendered output; `\num[mode=text]{...}` is the workaround.
- Ranges take an en-dash (`--`), parentheticals an em-dash (`---`), compound adjectives a
  hyphen. Resist the em-dash unless it earns its place.

### Prose style

```bash
grep -oiE "\b(don't|doesn't|can't|won't|isn't|aren't|it's|we're|that's)\b" flat.txt
grep -oE "et\.? al\b\.?" flat.txt        # must be "et al."
grep -oE "\b(e\.g\.|i\.e\.)[^,]" flat.txt # need the trailing comma
grep -oE "^[a-z_]+\(\)" flat.txt          # sentence starting with a lowercase identifier
```

- No contractions in formal writing.
- "e.g., " and "i.e., " — abbreviations, so periods and a comma.
- `that` for restrictive clauses, `which` for non-restrictive.
- its (possessive) / it's (contraction); past tense of *lead* is *led*.
- Prefer simple tenses; avoid participles and complex tenses.
- Avoid anthropomorphisms and pronoun pile-ups.
- Headings: title case or sentence case, but **consistent** across sections,
  subsections, and appendices. (Found: an appendix list mixing title case with sentence
  case, and one heading opening with a `#` abbreviation.) In title case, capitalize the
  second element of a hyphenated word.
- Captions: end with a period or do not, consistently; always use one after a full sentence.
- Define every abbreviation at first use; do not overuse them.

### Bibliography

```bash
grep -nE "^\s*title = \{[^{]" main.bib      # single-braced titles: bibtex will lowercase
grep -nE "[–—]" main.bib                     # literal en/em-dashes, esp. inside \url{}
grep -c "^@" main.bib
```

Real failures found:

- **Single-braced titles get case-mangled.** `title = {NanoTransport: A ... Layer for
  NICs}` rendered as "Nanotransport: A low-latency, programmable transport layer for
  nics." Use `{{Double Braces}}`, or brace proper nouns individually: `{NICs}`, `{TCP}`.
- **Hardcoded wrong case in the .bib**: `EBPF-Based` where the paper's real title is
  `eBPF-Based`. Double bracing preserves whatever is there, including a mistake.
- **Mangled URLs.** A literal en-dash inside `\url{}` rendered as mojibake
  (`fooâĂŞbarâĂŞbaz`). Percent-encode it: `foo%E2%80%93bar%E2%80%93baz`.
- Title case throughout, including `@misc` web pages.
- Every URL should actually resolve (no 404s).
- No duplicate entries; no duplicate citation of the same key in one sentence.
- Keep entries sparse: title, authors, venue, year. Drop ACM's "New York, NY" address
  field. Fix ACM/IEEE conference-name capitalization.
- List all authors; avoid "et al." in the bibliography.
- Use `\small` or larger for the bibliography.
- **Citation lists should be sorted.** `[6, 14, 13, 15, 18, 63, 26, 16, 44, 2, 17]` is a
  reviewer-visible tell. Fixable with `\usepackage{cite}` or natbib's `sort&compress` —
  but that is a **preamble change**; if the venue template must stay byte-identical,
  report it instead of doing it.
- Read the bibliography end to end before submitting. Cite only what you understand, and
  do not pad.

### Page limit, and how to shrink when you are over

```bash
pdftotext -f <limit+1> -l <limit+1> main.pdf - | head -1   # expect "References"
```

Confirm against the venue's actual rule (body only vs. everything).

**First measure the overflow in typeset lines, not in vague "about a page".** Count the
body lines that spill past the limit; a two-column body page is ~54 lines per column, so
"half a column over" is ~27 lines:

```bash
pdftotext -layout -f <limit+1> -l <limit+1> main.pdf - | \
  awk '{c=substr($0,1,70); gsub(/^[ \t]+|[ \t]+$/,"",c); if(c!="") print NR": "c}'
```

Then spend that budget in this order — cheapest and least damaging first.

1. **Unreferenced floats.** A figure nothing in the prose points at is *already* a defect
   (see "Reference coverage"); cutting it or moving it to the appendix fixes the defect
   and buys the most space of anything on this list. Compute what each one is actually
   costing, rather than guessing from `width=`:

   ```bash
   # rendered height = column width (~241pt in a 2-col letter template) x aspect ratio
   pdfinfo fig.pdf | awk '/Page size/{cw=241*0.9; printf "%.0fpt (~%.1f lines)\n", \
     cw*($5/$3), cw*($5/$3)/11}'
   ```

   One unreferenced 0.9-linewidth plot came to 282pt ≈ 26 lines, plus caption and float
   separation ≈ 30 lines — the entire overflow in a single move.

2. **Stray vertical space.** Audit every hand-tuned `\vspace`, especially positive ones
   that crept in around floats:

   ```bash
   grep -rn "vspace" *.tex | grep -v ":[0-9]*: *%"
   ```

   Look for a `\vspace{2.0em}` sitting between a `\caption` and its `\label`, and for
   `\vspace*{-0.5in}` after `\maketitle` — the latter buys space but is exactly what a
   venue's format check looks for. Flag it rather than relying on it.

3. **Oversized figures.** Dropping `width=1.0\linewidth` to `0.85` on a plot whose axis
   labels are already large is usually free.

4. **Prose.** Only now, and prefer deleting a redundant sentence to re-wrapping a
   paragraph; a paragraph that loses its last short line saves a line for free.

Fix orphans/widows with `\pagebreak`, not `\vspace` and not `widowpenalty`.

**Report the shrink options; do not unilaterally delete a figure or a paragraph.** Which
content is expendable is the author's call, always. Give them the line count each option
buys so they can choose.

## Step 4 — Scope discipline (read before editing anything)

**Fix directly:** grammar, subject–verb agreement, articles, punctuation, hyphenation,
spelling, unit spacing and prefix case, citation formatting, plural cross-references,
undefined/wrong references, leaked annotations.

**Report, do not change:** numbers, measurements, experimental claims, and anything whose
correctness depends on data you cannot see — **even when it looks obviously wrong.**

> Cautionary example. A sentence of the form *"X is encoded as N bytes: a 1-byte tag, a
> 1-byte length, and the k-byte payload"* is an arithmetic claim with three numbers in it,
> and the figure it cites is the only thing that says which one is wrong. A previous round
> "fixed" such a sentence by changing the wrong number — the total, when the figure showed
> the payload width was what disagreed — because the fix was applied without opening the
> figure. **Report the arithmetic and let the author pick.**

The same applies to a pattern or rule quoted in prose that differs by a character from the
one drawn in the figure, and to any "up to Nx" headline claim that does not match the
figure it cites.

## Step 5 — Verify every fix reached the PDF

Do not trust that an edit landed. Rebuild and re-extract:

```bash
rm -f main.aux && make >/dev/null 2>&1
grep -n "undefined" main.log | grep -v "Font shape"    # must be empty
pdftotext main.pdf new_all.txt
grep -c "??" new_all.txt                                # must be 0
grep -c "Claude:\|TODO\|FIXME" new_all.txt              # must be 0
```

Then re-grep for each specific fix: the corrected figure numbers, the added citations, the
unit spacing, the previously misspelled words. Confirm figure captions still carry the
numbers you expect and that nothing shifted the page count past the limit.

A LaTeX build "succeeding" means nothing here — undefined references are warnings.

## Step 6 — Report

**Do not commit or push anything.** Leave every change in the working tree and hand the
author the diff to review. A proofreading pass touches almost every file in the paper,
often hours before a deadline and often against a repo that syncs with Overleaf — an
unreviewed commit there is worse than the typos it fixed. Commit only when the author
asks, after they have read the report.


Two lists, in this order:

**Fixed** — one line each, grouped (critical / typos / citations / consistency), each
naming what a reader would have seen.

**Needs an author decision** — anything in the "report, do not change" bucket, each with:
the location, what the text says, what the figure or table says, and the specific question
being asked. Also list anything blocked on information you do not have (paper number,
camera-ready URLs, venue-specific formatting calls).

Say plainly which of the two lists is empty if one is. If a fix was risky or changes
layout, flag it for review rather than burying it.
