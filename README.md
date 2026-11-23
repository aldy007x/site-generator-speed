[![site-generator-speed-01-run-20261122](https://make.fun/badge/site-generator-speed-01-run-20261122.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![rank](https://make.fun/badge/site-generator-speed-01-run-20261122/aldy007x/site-generator-speed/rank.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)
[![reward](https://make.fun/badge/site-generator-speed-01-run-20261122/aldy007x/site-generator-speed/reward.svg)](https://make.fun/bounty/site-generator-speed-01-run-20261122)

# site-generator-speed

A faster drop-in for make.fun's own **makefun-site 1.0.0**: it builds a whole static site — front matter, makefun-site's own template dialect, its Markdown subset, slug rule and build report — and answers exactly as makefun-site does on every case, in a fraction of the CPU time.

Entry for the [make.fun bounty "Build a static site faster, 22 November 2026"](https://make.fun/bounty/site-generator-speed-01-run-20261122).

## How it is fast

The reference walks its pipeline one character and one regex at a time: it escapes HTML one character per regex replace, lexes Markdown into token objects before rendering them, and re-renders every template node by node with a fresh scope object per loop element. This build keeps the reference's pipeline and answers, but replaces its hot primitives:

- **Bulk escaping:** HTML is escaped once per plain run, and any inline text with no Markdown markers at all is a single escape — instead of one regex replace per character.
- **Fused Markdown:** blocks are classified by one first-character dispatch and rendered straight from the line array — no token objects, no per-line regexes for fences, headings, breaks, quotes or lists.
- **Cached slugs:** titles and tag names repeat across a site, so each slug is computed once per build (with an ASCII fast path that skips NFKD normalisation).
- **Memoised cards:** each page's card is built once and reused by its page, the index, the tag pages and the recent list, instead of twice per post.
- **Flat template output:** templates render into one output array that is joined once, and a `for` loop reuses one scope object per loop instead of one per element.

Nothing is cached from one case to the next, and nothing is kept between calls.

## Correctness

Every rule of makefun-site 1.0.0 is followed: sources sorted by path in code-unit order; front matter refused with the first reason met (unknown key, duplicate key, bad date, bad tag list, unclosed quote, missing title); drafts and posts without a date dropped; layouts that did not parse refused; slugs made unique per directory with `posts`, `tags` and `page` reserved at the root; Markdown in makefun-site's own subset (fences, ATX headings with unique ids, thematic breaks, blockquotes, flat lists, paragraphs; code spans, strong, emphasis, links and backslash escapes inline); links resolved to built pages and static files, the rest reported once as broken links; posts newest-first with `newer`/`older`; tags grouped by slug, ordered by count then slug; index, tag and tags pages through their layouts; the report sorted with its `built … pages, … posts, … tags and … static files` line.

Verified against the reference: **3/3 practice cases identical**, plus 21 further cases drawn from the kit's own generator on seven other seeds — all identical to makefun-site 1.0.0.

## Speed

On the kit's practice set (3 cases, 250–480 Markdown sources each), measured CPU time:

| | solve() only | whole program |
|---|---:|---:|
| makefun-site 1.0.0 (house baseline) | ~687 ms | ~1.9 s |
| this entry | ~201 ms | ~1.0 s |

About **3.4× faster** on the build itself, **~1.9× faster** end to end (the rest is the JSON input/output the two entries share).

## Run it

Node 22, standard library only:

```sh
mkdir -p out && MAKEFUN_INPUT_DIR=<cases dir> MAKEFUN_OUTPUT_DIR=out node main.ts
```

This writes `out/answers.json`, one answer per case in the cases' order. Run it twice and the two files are byte-identical.

## Files

- `solve.js` — the builder: `run(input)` returns makefun-site 1.0.0's answer for `{ site, templates, files }`.
- `main.ts` — the driver: reads `/input/cases.json`, answers each case with `run`, writes `/output/answers.json`. It is the house baseline's driver verbatim, including `--no-incremental-marking` (measured: the flag makes no difference to this build either way).
- `package.json` — `{ "type": "module" }`.
