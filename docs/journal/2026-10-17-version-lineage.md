# 2026-10-17: Version Lineage

The original vision is "how a text has updated **over time**": not just v1 → v2,
but v1 → v2 → … → vN. Pairwise alignment (`2026-10-17-node-matching-algorithm.md`)
is the building block. Lineage is what you get by chaining it, and it is the most
literal reading of the README's "similar to git content-addressing but for
embeddings".

## The shape of it

Run the matcher on each consecutive pair of versions (v_k, v_{k+1}). Each
correspondence becomes an edge in a graph whose nodes are (version, node) pairs. A
node's **lineage id** is assigned when it first appears (an *added* node in some
v_k) and propagated along unchanged, edited and moved edges. Splits fork a lineage
into children that record their parent. Merges join several lineages into one node
with several parents. Deletion ends a lineage. Reinsertion (content deleted in v_k
and reappearing in v_{k+3}) needs matching against *deleted* nodes from earlier
versions, not just the immediately preceding one. WikiWho handles exactly this case
for tokens, and TokTrack shows its results over the complete English Wikipedia (see
`2026-10-17-prior-art.md`).

The result is a per-node history: "this bullet was introduced in v2, reworded in v4,
moved under *Implementation* in v5, split into two in v7". That is what a person
editing a document intuitively knows, and what the project set out to make a machine
know.

## Identity is the hard part

Git's content-addressing is exact: a blob's id *is* the hash of its content, so
identity never drifts. With embeddings, identity has to be carried, not computed.
The id is assigned once and inherited through matches. That raises questions git
never faces:

- **Drift.** Under many small edits, a node can end up with no lexical or semantic
  overlap with its origin while every consecutive step was a confident "edited"
  match. Whether that is still the same node is a policy question. One possible
  answer is to track cumulative similarity to the lineage root and break or flag the
  lineage when it drops below a calibrated level.
- **Error propagation.** One wrong match early in the chain mislabels every later
  version. Consecutive-pair matching is cheap but greedy. Matching v_{k+1} against
  the union of all live lineage heads (and recently deleted ones) costs more but
  allows recovery and reinsertion.
- **Content address vs. lineage id.** Both are worth keeping: a content hash per
  (version, node) for the exact tier, and a lineage id per node history. The content
  hash answers "has this exact text existed before?" and the lineage id answers
  "where did this come from?".

## Data model sketch

polars fits naturally, in keeping with mdbed. One long table with one row per
(version, node): `version`, `node_ix`, `tag`, `heading_path`, `text`,
`content_hash`, `embedding`, `lineage_id`, `parent_lineage_ids`, `op` (added,
unchanged, edited, moved, split, merged). It would be built incrementally, one
version at a time. Queries like "full history of lineage L" or "everything added in
v5 that survives to vN" are then group-bys and filters.

## Current State

- No version sequence handling, lineage identity, or history storage code in lit-embs (src/lit_embs/__init__.py:1-2)
- `example/data/` holds one source and three independent rewrites of that source — a one-to-many fan-out from `source_prompt.md`, not a v1 → v2 → v3 sequence

## Missing

- Chaining of pairwise alignments across an ordered sequence of versions has no code
- Lineage id assignment and propagation through unchanged, edited, moved, split and merged correspondences has no code
- Reinsertion detection (matching new nodes against nodes deleted in earlier versions) has no code
- Lineage drift handling (cumulative similarity to lineage root) has no code
- A multi-version example (an ordered sequence of edits to one document) has no file in `example/data/`
