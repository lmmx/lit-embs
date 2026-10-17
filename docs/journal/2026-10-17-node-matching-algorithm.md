# 2026-10-17: Node Matching Algorithm

This is the core of lit-embs: given the nodes of document A and document A′ (from
mdbed), decide which node became which, and classify each correspondence. mdbed's
attempt stops at "every pair with cosine ≥ 0.7, all against all", and even that
crashes (mdbed `docs/journal/2026-10-17-similarity-search.md`,
`2026-10-17-cross-document-pairing.md`). A soup of above-threshold edges is not an
alignment. A heading such as "Introduction" clears 0.7 against many short unrelated
nodes, one node pairs with arbitrarily many others, and there is no notion of
"unmatched", so nothing is ever reported as added or deleted.

## Target output

For each consecutive pair of versions, a set of correspondences, each classified:

- **unchanged**: exact match after normalisation
- **edited**: one-to-one, same structural slot, similar but not identical
- **moved**: one-to-one, different structural slot (parent heading or position),
  possibly also edited
- **split**: one node in A to several in A′
- **merged**: several in A to one in A′
- **deleted** / **added**: no counterpart

## Tiered matching

Embedding similarity is the most expensive and the least precise signal, so it should
only run on what cheaper, exact signals leave unclaimed. That is the WikiWho and git
lesson (see `2026-10-17-prior-art.md`):

1. **Exact tier.** Hash normalised node text (whitespace, case and markdown markup
   stripped). Identical hashes match outright. This is the literal "git
   content-addressing" from the README.
2. **Lexical tier.** For what remains, a cheap fuzzy match (token-set Jaccard or
   normalised edit distance) catches typo fixes and small rewordings with high
   confidence.
3. **Embedding tier.** Only the residue is embedded and compared by cosine.

Each tier removes claimed nodes from the pool before the next tier runs. That stops
a confident exact match from being stolen by a spuriously high cosine.

## Choosing the assignment engine

Four candidates, each handling splits and merges differently:

- **Hungarian with a null cost.** Pad the cost matrix with "unmatched" dummies at a
  fixed cost. The result is optimal one-to-one with added/deleted for free, and
  simple and fast at document scale. It *cannot* represent splits or merges: a split
  shows up as one edit plus one or more additions.
- **Vecalign-style monotone DP with block matches.** Splits and merges come
  natively, through concatenated multi-node blocks. Self-calibrating skip cost. But it
  is order-preserving, so it is blind to moves at the level it runs on.
- **Optimal transport (Sinkhorn, unbalanced).** A soft many-to-many plan, where
  "how much of X went to Y" reads off directly, so splits and merges are first
  class. Moves are free because OT ignores order. But it needs a rounding or
  thresholding step to turn mass into discrete classifications, and it gives no
  structural preference by default.
- **Embedding-weighted GumTree.** Top-down anchors from the exact tier, bottom-up
  containers by dice over matched descendants, then recovery within containers. Moves
  are first class. Structure informs content and vice versa.

These compose naturally rather than compete. GumTree-style top-down/bottom-up decides
*which sections correspond*, which handles moves at section level. Within a matched
section pair, where order is mostly preserved, a Vecalign-style monotone DP aligns the
children with block matches, which handles splits and merges. Optimal transport (or
Hungarian) runs on the global residue of nodes left unmatched by both, which catches
content that moved *across* sections.

## Thresholds and calibration

A fixed global cosine threshold (mdbed's 0.7) cannot separate "edited" from
"replaced". bge-small cosine between *unrelated* nodes in the same document is far from
zero, and that baseline varies by document and by node length. Two concrete
mechanisms:

- **Per-document-pair calibration (from Vecalign).** Sample random node pairs between
  A and A′, take the distribution of their similarities, and define "unrelated" as a
  quantile of that distribution (`c_skip = CDF⁻¹(β)`). Normalise each candidate
  pair's cosine against its own average similarity to random nodes, which penalises
  hub nodes such as short headings.
- **Calibration on labelled data.** Fit the edited/replaced boundary and `β` on
  arXivEdits gold alignments, or on IteraTeR edit types, instead of guessing.

## Node granularity interacts with all of this

What gets embedded is decided upstream in mdbed (`docs/journal/2026-10-17-markdown-node-extraction.md`
there): whether nodes carry their heading path, and how short headings are handled.
GumTree's container step offers a clean answer for headings: match a heading by the
dice of its section's contents, not by the cosine of its own text.

## Open questions

- Does including the heading path in each node's embedding text help (context-aware
  matching) or hurt (a reworded parent heading makes every child look "moved")?
  Testable on `example/data/` once gold alignments exist.
- What counts as "moved": a change of parent heading, a change of ordinal position
  beyond some distance, or both?
- Does unbalanced OT's soft plan beat block-DP for split/merge detection on
  arXivEdits?

## Current State

- No node matching, assignment or classification code in lit-embs (src/lit_embs/__init__.py:1-2)
- mdbed's `find_similar_nodes` selects every pair at or above a fixed cosine threshold of 0.7 with no assignment step, and raises before producing output (lmmx/mdbed src/mdbed/embedding.py:73-162)

## Missing

- Exact-hash, lexical and embedding matching tiers have no code
- One-to-one assignment with unmatched cost (Hungarian), monotone block DP (Vecalign-style), unbalanced optimal transport, and GumTree-style top-down/bottom-up tree matching have no code
- Classification of correspondences into unchanged, edited, moved, split, merged, added, deleted has no code
- Per-document-pair similarity calibration (random-pair quantile skip cost, hub normalisation) has no code
- Gold alignments for `example/data/` have no file, so no matcher output can be scored against the repository's own example data
