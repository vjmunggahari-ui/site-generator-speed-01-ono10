# Variant comparison — site-generator-speed v1

Practice set: 3 cases (g-0001: 257 files, g-0002: 348 files, g-0003: 486 files;
1,091 files total). All timings are medians on this machine (shared 1.5-CPU
VM, Node v24.22.1; the grader runs Node 22 — relative order is what matters,
absolute numbers will differ there). Solve-only timing: `run()` over the 3
cases in-process, interleaved rounds, `process.cpuUsage`. Fresh-process timing:
`node main.ts` in a new process per run (includes Node startup plus JSON
parse of `cases.json` and stringify of `answers.json`), interleaved rounds,
`resource.getrusage(RUSAGE_CHILDREN)`.

## Results

| variant | practice cases passed | solve-only CPU (median) | fresh-process CPU (median) | vs baseline (solve-only) |
| --- | --- | --- | --- | --- |
| A — faithful port, micro-opts (chunked escaping, slug cache, first-char dispatch, manual word count) | 3/3, passed=1 | 412 µs (9 rounds) | 1,384 µs (7 rounds) | 1.82× faster |
| B — A + scope-stack template renderer, array-join output, manual front-matter parse | 3/3, passed=1 | 419 µs (7 rounds; earlier run 428 µs) | 1,294 µs (7 rounds) | 1.79× faster |
| **C — B + fused single-pass Markdown (no token objects, inline appends to one array, word count rides along)** | **3/3, passed=1** | **325 µs (9 rounds, revised)** | **1,204 µs (7 rounds, revised)** | **2.31× faster** |
| House baseline (makefun-site 1.0.0) | 3/3, passed=1 | 751 µs | 1,641 µs | 1.00× |

Winner: **C (revised)** — fastest on both measures, and the simplest
explanation for its speed (fewer allocations per line rendered). A and B are
within noise of each other on solve-only time; C is consistently ahead in
fresh-process runs, which is the graded metric.

## Revision step (mandatory, on the winner)

After the first grade-local pass on C (3/3, passed=1), the CPU profile showed
`inline` at ~23% of self time and the template renderer at ~14%. Two edits,
both verified by byte-diff against the baseline on all 3 practice cases
before and after:

1. `inlineTo` now dispatches on `charCodeAt` instead of one-character
   strings and object lookups.
2. Every block regex in the fused line scan was replaced by an
   exactly-equivalent manual check (blank, fence open/close, heading,
   thematic break, bullet, ordered list, continuation).

Measured delta: solve-only median 363–379 µs → **325 µs** (~12% faster);
fresh-process median 1,316 µs → **1,204 µs**. Re-graded twice after the edit:
both runs 3/3, passed=1.

One bug caught during the build: the first fused version undercounted words
by one per fenced block (the closing fence line was consumed but not counted),
which flipped `reading_minutes` on two cases. Fixed by counting the closing
fence line; the byte-diff then went fully identical.

## Grade-local scores (practice set)

- Variant A: `{"cases_passed": 3, "cases_total": 3, "passed": 1}`
- Variant B: `{"cases_passed": 3, "cases_total": 3, "passed": 1}`
- Variant C (final, re-graded twice after revision):
  `{"cases_passed": 3, "cases_total": 3, "passed": 1}` (twice)

## Determinism

Two fresh-process runs of the shipped `main.ts` produce byte-identical
`answers.json`:
`sha256 25980928b145f54712027939364a64c277098bc4528a2d524e0bc27a0275fb03`
(both runs; verified before `package.json` was added — the file only declares
`{"type": "module"}` and does not change `run()` output; the final tree was
re-graded after adding it).

## Post-revision fix (before submit)

Self-review after the revision caught a latent corner in the fused word
counter: it treated only ASCII whitespace as separators, while the reference
counts `/\S+/g` (JavaScript's `\s` set, which also splits on NBSP, Ogham
space mark, en/em spaces, line/paragraph separators, narrow NBSP, medium
math space, ideographic space and BOM). A scored-set body holding e.g. a
non-breaking space between words would have miscounted `reading_minutes`.
`countWords` now uses the full `\s` set. Verified: byte-diff vs the baseline
still identical on all 3 practice cases, determinism sha unchanged
(`25980928…`), grade-local still 3/3, passed=1. Shipped as commit C4 on top
of C3; the beacon root (C1) is untouched.

The practice reports exercise broken links (157/174/280 per case), drafts
(31/26/43) and refused sources (5/23/24). NOT exercised by the practice set:
template-parse errors (no `template "..."` report lines) and shadowed static
files — those paths are direct ports of the reference logic, untested by the
practice outputs. The scored set is drawn after entries close, so any corner
outside the practice distribution is a residual risk (see Handoff).

## Skills and relays used

- Skills: `muse-juara` (brief restatement, 3-variant comparison, self-critique,
  final checklist); `code-review` (two-axis review: spec = the brief/README,
  standards = no dead code, small functions — unused regexes and `wordsOf`
  removed after the fuse); `coverage-analysis` (practice-set coverage check
  above: which report paths the 3 cases exercise and which they do not).
- Relays (same generic optimization prompt to each; public code only, no
  secrets): Groq (200, empty content — no contribution), Xkiro (3 ideas:
  StringBuilder — not applicable to JS string join; regex→state machine —
  applied as the manual-check revision; flat lookup map — rejected, breaks
  template shadowing), Bluesminds (Buffer accumulation — not applicable;
  cross-case template cache — REJECTED, forbidden by the rules; worker_threads
  — REJECTED, 1-CPU sandbox and total CPU is what is timed), Requesty (200,
  output truncated, no usable content), Coarena (402 on the first model, then
  200 with null content — no contribution), 100milionseek (503 then 502 —
  down), Xingchi (400 chat_completions_disabled — down for script calls, as
  documented). Every relay claim was verified independently before use; the
  only relay-originated change applied is the manual-check revision, measured
  above.
