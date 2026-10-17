# 2026-10-17: Status Audit

lit-embs ("literate embeddings") holds the vision half of a two-repo idea. The idea
is to be able to say how a text has changed over time by correlating parts of one
version with parts of another. A person editing a document can easily see what went
where; a machine cannot. The README states the goal in three lines: compare
documents "under change of structure", report "what went where", and report "what
was added/removed (similar to git content-addressing but for embeddings)".

The sibling repo `lmmx/mdbed` was the first concrete stage: parse markdown into
nodes, embed the nodes, compare them. lit-embs was to sit above that and turn node
similarities into an account of the edit. Both repos were scaffolded on the same day,
2025-03-01, from the same template (identical README, near-identical pyproject, the
same example data).

## What is here

Very little code. lit-embs has six authored commits on 2025-03-01 (initial commit,
brief README, skeleton, lint, example data, more packaging), followed only by
pre-commit-ci autoupdates #13 through #42. The entire package is:

```python
def main() -> None:
    print("Hello from literate-embs!")
```

Even that cannot be reached through the installed script. pyproject.toml registers
`literate-embs = "literate_embs:main"` (pyproject.toml:53), but the package directory
is `src/lit_embs/`, so the console script fails with an import error. The name looks
like a leftover from an earlier working title, "literate-embs".

The one substantive asset is the test data in `example/data/`. It is identical to
mdbed's copy:

| File | Lines | What it is |
|---|---|---|
| `source_prompt.md` | 63 | Whiteboard notes for a blog post, "Notes towards better maps of codebase territories": bullets with arrows into sub-points |
| `claude_3-7_generation.md` | 50 | Claude 3.7's rewrite of the notes into an outline |
| `chatgpt_4-5_generation.md` | 55 | GPT-4.5's rewrite |
| `chatgpt_o1_generation.md` | 151 | o1's rewrite, the longest and most expanded |

That is a well-chosen test set for this problem: the same content, restructured three
different ways, with known provenance (every rewrite derives from `source_prompt.md`).
The rewrites exercise exactly the hard cases: reordering, splitting one bullet into a
section, merging, rewording, and invented additions. It has no gold alignment
annotations yet. "Which source bullet became which rewrite node" is still only in a
human's head.

The only declared runtime dependency is `polars-fastembed>=0.1.2`
(pyproject.toml:27). That inherits the x86_64 Linux wheel gap documented in mdbed's
`docs/journal/2026-10-17-polars-fastembed-wheels.md`: no polars-fastembed release
since 0.1.4 ships an x86_64 Linux wheel.

## Where the idea got to

The framing is sound, and the mdbed audit shows where the work is. The embed half
(markdown → nodes → vectors) is mostly built in mdbed, with real parser bugs. The
matching half, which is the actual content of lit-embs, has no code anywhere. The
design for that half, and the prior art it should draw on, is recorded in three
companion entries:

- `2026-10-17-prior-art.md`: GumTree, Vecalign, WikiWho/TokTrack, git move
  detection, IteraTeR, arXivEdits, Wikipedia edit-intention work.
- `2026-10-17-node-matching-algorithm.md`: turning similarities into an alignment,
  meaning tiered matching, the choice of assignment algorithm, and threshold
  calibration.
- `2026-10-17-version-lineage.md`: extending pairwise alignment to persistent node
  identity across many versions.

## Current State

- `src/lit_embs/__init__.py` defines `main()`, which prints "Hello from literate-embs!" and performs no other operation (src/lit_embs/__init__.py:1-2)
- pyproject.toml declares one runtime dependency, `polars-fastembed>=0.1.2`, plus `polars` / `polars-lts-cpu` extras and a `dev` extra with pytest, pre-commit, pdm, pdm-bump (pyproject.toml:26-50)
- `example/data/` holds four markdown files — `source_prompt.md` (63 lines) and three LLM rewrites of it, `claude_3-7_generation.md` (50), `chatgpt_4-5_generation.md` (55), `chatgpt_o1_generation.md` (151) — identical to `example/data/` in lmmx/mdbed
- Six authored commits dated 2025-03-01 make up the history, followed by pre-commit-ci autoupdate commits #13 through #42 only

## Stubbed

- `main()` prints a fixed greeting in place of any CLI behaviour (src/lit_embs/__init__.py:1-2)

## Missing

- Document comparison, node alignment, move/edit/add/delete classification, and content-addressed node identity have no code in lit-embs
- Gold alignment annotations for `example/data/` (which `source_prompt.md` node corresponds to which node of each rewrite) have no file in the repository
- Tests have no code — `pytest` appears in the `dev` extra but the repository contains no test files

## Divergence

- pyproject.toml registers the console script `literate-embs = "literate_embs:main"`, but the package directory is `src/lit_embs/`, so the `literate-embs` script raises `ModuleNotFoundError` (pyproject.toml:53)
- README describes comparing documents under change of structure, "What went where" and "What was added/removed", but the package contains only the greeting `main()` (README.md, src/lit_embs/__init__.py:1-2)
- pyproject.toml classifies the package as "Development Status :: 4 - Beta" while the package contains no functional code (pyproject.toml:12)
