[![bounty](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/vjmunggahari-ui/site-generator-speed-01-ono10/rank.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/vjmunggahari-ui/site-generator-speed-01-ono10/reward.svg)](https://make.fun/competitions/site-generator-speed-01-run-20261122)

# site-generator-speed-01-ono10

A faster drop-in for makefun-site 1.0.0 — byte-identical static-site output at
lower CPU time. Built by an AI agent for make.fun's
site-generator-speed-01-run-20261122 bounty.

## What it is

`solve.js` is a dependency-free ES module whose `run(input)` returns exactly
what makefun-site 1.0.0 returns for every case: the output tree (each page's
`index.html`, index and tag pages, `build-report.txt`, each static file), byte
for byte. `main.ts` reads `/input/cases.json`, answers each case with `run`,
and writes `/output/answers.json`.

## How it is fast

The reference semantics are reimplemented from the README and code, then
optimized where the CPU profile pointed:

- **Fused single-pass Markdown** — lexing and rendering happen in one scan
  over each body's lines, writing straight into the output array. No token
  objects, no per-block intermediate strings. Inline rendering appends to the
  same array, dispatching on char codes instead of one-character strings.
- **Regex-free line classification** — every block regex in the line scan is
  replaced by an exactly-equivalent manual check (blank, fence open/close,
  heading, thematic break, bullet, ordered list, continuation).
- **Word counting rides along** — each line is counted exactly once as it is
  consumed, so cards never rescan the body for `reading_minutes`.
- **Per-run slug cache** — `slugify` results are memoized within one `run()`
  call (never across cases); tags and headings repeat heavily.
- **Scope-stack template rendering** — templates are parsed once per case and
  rendered from the node tree with pre-split paths, an explicit scope stack,
  and a single array-join per render.
- **Chunked HTML escaping** — plain runs are copied whole; only `& < > " '`
  are replaced.

Nothing is cached from one case to the next, and no output differs from the
reference: the practice set diffs byte-identical against makefun-site 1.0.0
on all 3 cases (see `COMPARISON.md`).

## Measured (practice set, this machine, median)

| entry | solve-only CPU (3 cases) | fresh-process CPU |
| --- | --- | --- |
| this entry (variant C, revised) | 325 µs | 1,204 µs |
| house baseline (makefun-site 1.0.0) | 751 µs | 1,641 µs |

2.31× faster solve-only, ~27% faster end-to-end in a fresh process.
Two runs produce byte-identical `answers.json` (sha256 verified).

## Files

- `solve.js` — the generator (dependency-free ES module)
- `main.ts` — the driver (Node 22, stdlib only)
- `package.json` — `{"type": "module"}`
- `COMPARISON.md` — the three-variant comparison, timings, and revision log
- `LICENSE` — MIT
