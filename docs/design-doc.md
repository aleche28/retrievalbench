# retrievalbench — Design Doc

> Retrieval evaluation framework: parsing × chunking × embedding × retrieval
> Status: Draft v0.2 — 2026-10-07 (v0.1 kept in `design-doc.v0.1.md`)

## Changes from v0.1

- **Name.** The project is now `retrievalbench`, replacing the placeholder `chunkeval`. The old name covered only chunking, but the grid also spans parsers, embedders and retrievers.
- **Framework scope.** The core evaluates a corpus and an evaluation set that the user provides. Tools for building evaluation sets (`gen-queries`, `import-langfuse`, `label`) move to a later phase (M4).
- **Metrics.** A recall-vs-context-tokens curve replaces the single token budget. Its normalized area is the headline metric. Packing is "rank order, no skipping, no truncation".
- **Context units.** Retrieval ranks index chunks. An expansion step then maps them to deduplicated context units, which carry the coverage ranges and the token cost. This settles `parent_child`.
- **Tokenizer.** One canonical tokenizer, set in the config, measures chunk sizes and budgets. Each embedder's own tokenizer is used only for truncation.
- **Chunk size.** A grid-level `chunk_size` applies to every chunker. A chunker can override it, and the tool warns when actual chunk lengths are not uniform. The baseline runs at every size that appears in the grid.
- **Retrieval depth.** By default the whole class corpus is ranked, so recall curves always reach `max_budget`.
- **Ground truth schema.** Relevant spans are grouped. Every group is required, and the spans inside a group are equivalent alternatives. Labels carry prefix/suffix context for alignment.
- **Alignment failures.** Queries are never dropped. A span that fails to align counts as zero coverage.
- **Statistics.** There are fixed dev/test splits, a cluster bootstrap by document, and a confirmation step on test limited to a shortlist with multiple-comparison control. Agreement between query sources is measured as rank correlation with a CI.
- **`no_answer` queries.** They are excluded from span metrics and reported separately. Hit rate and MRR are redefined on the union of what has been retrieved so far, bounded by the budget.

## 1. Summary

Most RAG systems chunk and embed every document the same way, by habit. This tool replaces habit with measurement. Given a real corpus and an evaluation set of queries, it runs a grid of **parsing × chunking × embedding × retrieval** configurations and reports how well each one retrieves the right content.

It is a **framework**. Users plug in their own data sources and their own evaluation set, and the tool produces comparable statistics across retrieval strategies. The quality of the evaluation set is the user's responsibility. The tool reports the signals it can, such as alignment failures, breakdowns by query source and agreement between sources, but it cannot correct a biased set. Tools for building evaluation sets are planned for a later phase (section 7.7).

It is a standalone CLI driven by config files. It can also be used as a pluggable stage (parse → chunk → embed) inside a bigger project.

## 2. Goals and non-goals

### Goals

- Ingest real documents and evaluate several chunking strategies and embedding models against each other on the same queries.
- Treat **document class** as the unit of evaluation: each class gets its own grid and results.
- Use **chunk-independent ground truth**, so different chunking strategies can be compared fairly.
- Produce **statistics with confidence intervals**, broken down by document class, query type, query source and language.
- Make every pipeline stage pluggable (parsers, chunkers, embedders, retrievers).

### Non-goals (for now)

- Measuring answer quality (generation). Retrieval only.
- Automatically recommending a configuration. The tool produces stats; humans decide.
- Running as a service. CLI + config files only.
- Building evaluation sets in the core. This is a later, optional phase (M4).

## 3. Document classes

| Class           | Description                                                          | Priority                   |
| --------------- | -------------------------------------------------------------------- | -------------------------- |
| `manuals`       | Structured PDF user manuals: TOC, chapters, sections, tables, images | **High** (first milestone) |
| `office`        | Office documents, shorter and less structured                        | Medium                     |
| `config_tables` | PDFs with one or two configuration tables                            | Low                        |

Classes are assigned by glob rules in the config. Automatic classification is a possible later feature.

In production each class lives in its **own index**, so each class can have its own chunker _and_ its own embedding model.

## 4. Architecture

### 4.1 Pipeline

```
source files
   │  parse (per parser)
   ▼
DocumentTree  ─────────────────►  cached by (doc_hash, parser, parser_version, params)
   │  chunk (per chunker)
   ▼
Chunks + ContextUnits ─────────►  cached by (tree_key, chunker, chunker_version, params, tokenizer)
   │  embed (per model)
   ▼
Vectors ───────────────────────►  cached by (hash(text_for_index), embedder_key)
   │  index + retrieve (per retriever)
   ▼
Ranked chunks per query
   │  expand to context units, deduplicate
   ▼
Ranked context units per query
   │  evaluate (recall-vs-tokens curve)
   ▼
Metrics + reports
```

Each stage is cached by content hash. Grids grow quickly, and embedding is the most expensive step, so a new chunker must never force re-parsing, and a new retriever must never force re-embedding.

Cache keys include code and model versions so results stay reproducible. `embedder_key` covers the model name and revision, query/passage prefixes, normalization and output dimensions. API models can change without notice, so the model version reported by the API is recorded whenever it is available.

### 4.2 Intermediate representation: `DocumentTree`

Every parser produces the same tree, and every chunker consumes it. This makes chunkers interchangeable.

```
DocumentTree
  doc_id, doc_hash, class, metadata (product, version, language, ...)
  text: canonical parsed text
  nodes: [
    Node
      kind: heading | paragraph | list | table | image | caption | code
      level (for headings)
      text
      char_range in canonical text
      page          # physical PDF page index, 1-based
      children
  ]
  removed: boilerplate ranges (headers, footers, page numbers, repeated warnings)
```

Requirements:

- Every node keeps its **char range** and **page**, so retrieved chunks can be mapped back to the source. `page` is always the physical page index, never the printed page number.
- Tables are preserved as structured data plus a text rendering (Markdown or similar).
- Images keep their captions and position. Image content (OCR or vision descriptions) is optional and out of scope for v1.
- Boilerplate is recorded and removed, not silently deleted, so its effect can be inspected. Removed ranges can contain real answers (a repeated safety warning, for example). Labels that align inside a removed range are reported separately (section 8.4).

### 4.3 Chunks and context units

A **chunk** is what gets indexed and ranked. A **context unit** is what gets returned to the LLM. For most chunkers they are the same thing. They differ for strategies like `parent_child` or sentence-window retrieval.

```
Chunk
  chunk_id, chunk_hash
  doc_id
  match_ranges: [char ranges in canonical text]   # what is indexed; may be non-contiguous
  text_for_index        # embedded and BM25-indexed; may include a prepended breadcrumb
  context_id            # the context unit this chunk expands to
  metadata: section path, page(s), node kinds

ContextUnit
  context_id
  doc_id
  context_ranges: [char ranges in canonical text]  # what is returned
  text_for_context      # what is returned to the LLM
```

- **Scoring and cost.** Only `context_ranges` count for coverage, and only `text_for_context` counts toward the token cost. Text added by a chunker, such as a breadcrumb, is never scored as retrieved content.
- **Default mapping.** By default each chunk is its own context unit, with `context_ranges = match_ranges` and `text_for_context` equal to the chunk text without the breadcrumb.
- **Diagnostics only.** `match_ranges` is used for diagnostics, such as "the right child matched, but its parent was too large for the budget".

### 4.4 Plugin interfaces

```python
class Parser:
    def parse(self, path, params) -> DocumentTree: ...

class Chunker:
    def chunk(self, tree: DocumentTree, params, tokenizer) -> tuple[list[Chunk], list[ContextUnit]]: ...

class Embedder:
    max_tokens: int
    tokenizer: Tokenizer                          # the model's own, used only for truncation
    def embed_documents(self, texts) -> Vectors: ...
    def embed_query(self, text) -> Vector: ...    # handles asymmetric prefixes (e.g. "query:"/"passage:")

class Retriever:
    def index(self, chunks, vectors): ...
    def search(self, query, query_vector, k) -> list[ScoredChunk]: ...
```

Embedders must **report truncation**. If a chunk exceeds `max_tokens` (measured with the embedder's own tokenizer), it is counted and shown in the report. Silent truncation would bias results against large-chunk strategies.

### 4.5 Tokenizers

There are two kinds of tokenizer, and they serve different purposes:

- **Canonical tokenizer** (`tokenizer` in the config). Measures chunk sizes, context-unit lengths, budgets and the source tokens used in span metrics. It should match the target LLM, since budgets model the LLM's context.
- **Embedder tokenizer.** Each embedder uses its own, only to detect truncation.

A chunk size of 512 therefore means the same thing for every chunker and every embedder.

### 4.6 Retrieval during evaluation

Use **exact (brute-force) vector search** during evaluation. Approximate indexes add their own recall loss, which would be confused with chunking or embedding effects. Corpus sizes per class should make exact search feasible.

Retrievers in v1:

- `dense`: cosine similarity over `text_for_index`
- `bm25`: lexical over `text_for_index`, with language-aware tokenization (stemming and stopwords chosen by document language)
- `hybrid`: dense + BM25 over the same units, fused with Reciprocal Rank Fusion (weighted fusion as an option)

Dense and BM25 index the same text, so hybrid fusion merges two rankings of the same units.

**Retrieval depth.** The recall curve needs enough context units to reach `max_budget` **after** context expansion and deduplication. This matters most for `parent_child`, where many children collapse into one parent. Exact search makes it cheap to rank the whole class corpus, so the default is:

- `dense` and `bm25` rank every chunk.
- `hybrid` fuses the two full rankings.
- Ranks are consumed until the cumulative context cost reaches `max_budget` or the corpus runs out.

If a corpus is too large for that, `retrieval_depth` caps the number of chunks ranked per list. The report then warns about every query whose curve stops before `max_budget` because the depth ran out. The depth used is recorded in `run.json`.

### 4.7 Context expansion

After retrieval, ranked chunks are mapped to context units:

1. Replace each chunk with its context unit.
2. Deduplicate by `context_id`. Each context unit keeps the rank of its best-ranked chunk.

This matches the usual behavior of parent-document retrievers. It also ensures a parent retrieved through several children is charged against the budget only once.

## 5. Initial grid for `manuals`

| Axis       | Options                                                                                          |
| ---------- | ------------------------------------------------------------------------------------------------ |
| Parser     | Docling; a PyMuPDF-based parser                                                                  |
| Chunk size | e.g. 256, 512 (grid-level default; chunkers may override)                                        |
| Chunker    | `fixed` (baseline); `recursive`; `hierarchical` (heading-based, with breadcrumb); `parent_child` |
| Embedder   | 2–3 models, e.g. one open-source multilingual, one API-based                                     |
| Retriever  | `dense`, `hybrid`                                                                                |

Chunker details (`size` is measured with the canonical tokenizer, on the chunk's source text only; a breadcrumb is not counted):

- **fixed**: windows of `size` tokens with overlap. The baseline every other strategy must beat.
- **recursive**: split on paragraphs, then sentences, up to `size`.
- **hierarchical**: a section is the unit. Split only if it exceeds `size`. Prepend the breadcrumb ("Installation > Network > Proxy settings") to `text_for_index`.
- **parent_child**: index children of `size` tokens. The context unit is the enclosing section, so the recall curve (section 8.1) charges the full section length to every hit.

### 5.1 Chunk size uniformity

`chunk_size` applies to every chunker in the grid, and each of its values expands into the grid. A chunker can override it with its own `size`. An overriding chunker runs only at its own size and is not crossed with the `chunk_size` list. This keeps it possible to compare strategies at different sizes, but a difference between two strategies may then come from size alone.

The baseline chunker is run at **every size that appears in the grid**, overrides included. Every configuration therefore has a same-size baseline to be compared against (section 8.2).

The tool **warns** without blocking:

- **Explicit override.** It warns when a chunker overrides `chunk_size`.
- **Actual lengths after chunking.** For each nominal size, it compares the **actual** mean length of indexed chunks across chunkers, measured with the canonical tokenizer and excluding breadcrumbs. It warns when they differ by more than `size_warning_tolerance`. A nominal size can be far from the actual one: `hierarchical` with `size: 800` may produce chunks averaging 300 tokens, because most sections are short.

The warning looks at indexed chunk length because that controls retrieval granularity. The cost of returned context is already accounted for by the recall curve. Breadcrumb lengths are reported separately (section 8.4).

## 6. Configuration

```yaml
run_name: manuals-baseline
corpus:
  root: ./docs
  version: 2026-10-01
queries: ./eval/queries.v3.jsonl
tokenizer: cl100k_base # canonical tokenizer: chunk sizes, budgets, span tokens

classes:
  manuals:
    match: "manuals/**/*.pdf"
    grid:
      parsers:
        - docling: {}
        - pymupdf: {}
      chunk_size: [256, 512] # default for all chunkers; list values expand into the grid
      chunkers:
        - fixed: { overlap: 64 }
        - recursive: {}
        - hierarchical: { breadcrumb: true }
        - parent_child: { parent: section } # chunk_size applies to the children
        - hierarchical: { size: 800, breadcrumb: true } # override: runs only at 800, adds fixed@800 baseline, size warning
      embedders: [bge-m3, text-embedding-3-small]
      retrievers: [dense, hybrid]
  office:
    match: "office/**/*.{docx,pptx}"
    grid: { ... }

eval:
  split: dev # dev | test (see 8.2)
  split_assignment: { test_fraction: 0.3, group_by: doc_id, seed: 13 } # for queries without a split field
  max_budget: 4000 # upper bound of the recall curve
  retrieval_depth: all # or a number of chunks per ranked list (see 4.6)
  budgets: [1000, 2000] # spot values shown in tables
  metrics: [span_recall_auc, span_recall, span_precision, span_iou, hit, mrr]
  overlap_threshold: 0.5 # fraction of a span that must be covered to count as a hit
  bootstrap: { samples: 1000, cluster_by: doc_id }
  baseline: { chunker: fixed }
  size_warning_tolerance: 0.25
```

List-valued parameters expand into the grid. Each combination is a **configuration**.

## 7. Ground truth

### 7.1 Principle: labels don't depend on chunks

A label like "chunk #42 is relevant" only exists for one chunking strategy. Instead, ground truth is a set of **relevant spans in the source document**. Every strategy is scored by how well its retrieved context covers those spans.

### 7.2 Anchoring spans across parsers

The parser is itself a grid axis, so character offsets in one parser's output don't apply to another's. Labels are stored as **page + exact text + surrounding context** (a short prefix and suffix, in the style of the W3C `TextQuoteSelector`). A resolver aligns each span to each parser's canonical text with fuzzy matching. It uses the page to narrow the search and the prefix/suffix to tell apart repeated strings such as "Press OK" or "30 s".

Rules:

- **Failures never drop a query.** A span that fails to align counts as **zero coverage** for that parser. If a query were dropped instead, a parser that loses hard content (tables, multi-column pages) would be scored on an easier subset and look better. Every configuration is scored on exactly the same query set.
- **Failures are reported per parser.** They also work as a **parse quality signal**: a parser that loses or garbles labeled text will show it here.
- **Spans in removed ranges are flagged separately.** A span that aligns inside a removed boilerplate range is reported apart from other failures.

### 7.3 Query schema

```json
{
  "query_id": "q_0142",
  "query": "How do I configure a proxy for firmware updates?",
  "class": "manuals",
  "type": "single_fact",
  "source": "production",
  "language": "en",
  "split": "dev",
  "added_in_version": "v3",
  "relevant": [
    {
      "group": "g1",
      "spans": [
        {
          "doc_id": "manual_x200_en",
          "doc_hash": "sha256:...",
          "page": 47,
          "text": "To use a proxy server, open Settings > Network ...",
          "prefix": "4.3 Proxy settings ",
          "suffix": " and enter the address"
        },
        {
          "doc_id": "manual_x200_en",
          "doc_hash": "sha256:...",
          "page": 6,
          "text": "Proxy: Settings > Network > Proxy ...",
          "prefix": "Quick start ",
          "suffix": ""
        }
      ]
    }
  ],
  "metadata": { "product_version": "X200 v4" }
}
```

Semantics of `relevant`:

- **Groups are all required.** A `multi_span` query has two or more groups.
- **Spans within a group are equivalent alternatives.** Either one answers that part of the query, for example the same instruction in the quick start and in the full chapter. Retrieving any alternative counts.
- **`no_answer` queries have an empty `relevant` list.**

Query types (initial set): `single_fact`, `multi_span`, `table_lookup`, `procedural`, `no_answer`.

`split` is optional. Queries without it are assigned deterministically, using `eval.split_assignment` (section 8.2).

### 7.4 Validation

`retrievalbench validate-queries <config>` checks the query file referenced by the config. It needs the config to know the corpus and parsers. It checks:

- the schema;
- alignment of every span against every parser in the config;
- staleness (whether each `doc_hash` still matches the corpus);
- the split assignment.

Results go to a validation report.

### 7.5 Versioning and staleness

- The eval set is versioned (`queries.v3.jsonl`). Every report records the eval set version and corpus version. Results from different versions are not compared directly.
- Each label stores the source `doc_hash`. When a document changes, its labels are flagged stale. Stale labels are reported, and they are excluded from runs unless the user explicitly includes them. The relabeling workflow is part of M4.

### 7.6 Evaluation set quality is the user's responsibility

The framework scores whatever evaluation set it is given. A set built from a biased process will give biased rankings, and the tool cannot detect every such bias. What it does provide:

- breakdowns by query source, type and language;
- **rank agreement between sources** (section 8.3), which shows whether a cheap source such as synthetic queries ranks configurations the same way as a trusted one;
- alignment-failure and removed-range reports.

### 7.7 Building evaluation sets (later phase, M4)

Tools for building evaluation sets are planned as an advanced, optional phase. They produce files in the schema of section 7.3 and are not required to use the framework.

#### Query sources

| Source       | How it's made                                 | Strengths                   | Weaknesses                                              |
| ------------ | --------------------------------------------- | --------------------------- | ------------------------------------------------------- |
| `synthetic`  | LLM generates questions from sampled spans    | Cheap, thousands of queries | Simple lookups; reuses document wording (flatters BM25) |
| `curated`    | Hand-written and hand-labeled by the team     | Trustworthy                 | Small                                                   |
| `production` | Real user queries from Langfuse, then labeled | Realistic                   | Needs labeling and cleaning                             |

#### Synthetic generation rules (`gen-queries`)

- Sample spans **independently of any chunking strategy**. Generating from a strategy's chunks favors that strategy.
- Sample spans from more than one parser's output, or from a parser-neutral text. Sampling from one parser favors that parser and the chunkers built on its nodes.
- Ask for paraphrased questions to reduce lexical overlap with the source.
- Generate several query types, including multi-span queries built from two related sections.
- Filter with an LLM judge for answerability and specificity.
- Don't use the embedding models under test for generation or filtering.
- Follow span-width guidelines so that synthetic spans have widths comparable to hand-labeled ones. Span width strongly affects precision and IoU.

#### Importing production queries (`import-langfuse`)

- Remove duplicates, PII and chit-chat.
- Production queries are often follow-ups in a conversation ("and for the X300?"). Rewrite them into standalone queries using the **same query rewriting production applies**, or skip them. Otherwise the eval measures queries the system never actually sees.
- Keep unanswerable queries tagged `no_answer`. They're useful if score thresholds are ever evaluated.

#### Labeling with pooling (`label`)

Production queries arrive without labels. Labeling only from the production retriever's results would bias the set toward that configuration. Instead, pool candidates TREC-style:

1. Run the query against **every configuration** in the grid (or a representative subset).
2. Take the union of retrieved context units (mapped back to source spans).
3. An LLM proposes the relevant spans and equivalent alternatives from the pool.
4. A human confirms or edits. The labeler can also **search the full document**, so a query that every configuration missed isn't wrongly labeled `no_answer`.

The pooled ranges are stored with the query as `judged` ranges. When a configuration added later retrieves text outside them, the report shows that **unjudged fraction**. Without it, unjudged text would silently count as irrelevant and penalize new configurations.

## 8. Metrics

### 8.1 Recall-vs-context-tokens curve

Comparing top-k across strategies is unfair: five 200-token chunks and five 1000-token chunks are different amounts of context. A single fixed budget is also fragile, because the result then depends on how the last chunk that doesn't fit is handled. So the tool measures recall as a function of the context tokens spent.

**Procedure, per query and configuration:**

1. Walk the ranked context units (section 4.7) in rank order, **without skipping or truncating** anything. This is how a real context builder behaves.
2. After unit `i`, let `c_i` be the cumulative cost: the sum of `text_for_context` lengths in the canonical tokenizer. Let `T_i` be the union of source tokens in the `context_ranges` of units `1..i`. Overlapping units pay twice in cost but count once in `T`, which correctly penalizes heavy overlap.
3. For a budget `b`, let `i(b)` be the largest `i` with `c_i ≤ b`, and let `T(b) = T_{i(b)}`. If the first unit alone exceeds `b`, then `T(b)` is empty. That is the honest result: the configuration cannot serve that budget.

**Relevant tokens with groups:**

- For group `g`, the coverage of alternative `a` is `|a ∩ T(b)| / |a|`. The group's coverage is the best of its alternatives.
- `R*(b)` is the union of the best alternative in each group. `R_all` is the union of every span in every alternative.

**Metrics:**

| Metric               | Definition                                                                                                      |
| -------------------- | --------------------------------------------------------------------------------------------------------------- |
| `span_recall@b`      | mean over groups of the group coverage at `b`                                                                   |
| `span_recall_auc`    | `(1 / max_budget) ∫₀^max_budget span_recall@b db`, a step-function integral in [0, 1]                           |
| `span_precision@b`   | `\|T(b) ∩ R_all\| / \|T(b)\|`; retrieving an equivalent alternative counts as relevant. Undefined if `T(b)` is empty |
| `span_iou@b`         | `\|T(b) ∩ R_all\| / \|T(b) ∪ R*(b)\|`                                                                           |
| `hit@b`              | 1 if every group has an alternative covered at least `overlap_threshold` by `T(b)`, else 0                       |
| `mrr@b`              | `1 / i`, where `i` is the first rank at which some group reaches `overlap_threshold` coverage by `T_i`, with `c_i ≤ b`; 0 otherwise |

**Headline and reporting:**

- `span_recall_auc` is the headline metric. It is a single number that doesn't depend on any packing rule.
- `span_recall@b` is reported at the `budgets` spot values.
- The full curve is plotted for each configuration.

**Why hit and MRR are defined this way.** Both measure coverage on the **union** retrieved so far, not on a single chunk. A per-chunk definition would mean a 200-token child could never "hit" a 1000-token procedure, which would structurally favor large chunks.

**Excluded queries.** `no_answer` queries are excluded from span metrics, because their recall and IoU are undefined. They are counted and reported separately. Queries where `T(b)` is empty are excluded from the `span_precision@b` mean. They are counted in the oversize rate (section 8.4).

### 8.2 Dev/test splits and statistical reporting

Selecting the best of many configurations and reporting its score on the same queries inflates that score (the winner's curse). With ~100 configurations, a few will also beat the baseline at 95% by chance. To avoid both:

- **Fixed splits.** Every query belongs to `dev` or `test`. The split is either set in the query file or assigned deterministically by hashing the `group_by` key (default `doc_id`), so near-identical queries on the same document don't leak across splits. `no_answer` queries have no document, so they fall back to `query_id`. Splits are stable across eval set versions.
- **Explore on dev.** `retrievalbench run` uses `dev` by default. The full grid is explored there.
- **Confirm on test.** `retrievalbench run --split test --configs <shortlist>` evaluates only a shortlist chosen on dev, plus the baseline. Paired comparisons on test use Holm–Bonferroni correction across the shortlist. `run.json` records every test evaluation, and the report shows how many configurations have been evaluated on test so far.
- **Cluster bootstrap.** Queries from the same document are correlated, so CIs are computed by resampling **documents**, not individual queries (`cluster_by: doc_id`). A query whose spans come from several documents is assigned to the document of its first span. With few documents per class, CIs will be wide, and the report should say so.
- **Paired comparison against the baseline.** Each configuration is compared with the baseline chunker **under the same parser, embedder, retriever and chunk size**. The comparison uses a paired cluster bootstrap on per-query differences. With a few hundred queries, a two-point difference is often noise, and the report must say so.

### 8.3 Agreement between query sources

For each pair of query sources with enough queries (e.g. `synthetic` vs `production`), the report gives:

- the Kendall's τ between the configuration rankings each source produces;
- a bootstrap CI for τ;
- the number of queries per source.

High agreement is evidence that the cheaper source can be trusted at scale. Low agreement, or a wide CI, means it can't yet.

### 8.4 Cost and diagnostic stats

Reported per configuration, without being combined into a score:

- number of chunks and context units; mean, p50, p95 and max length of indexed chunks (excluding breadcrumbs), breadcrumbs, and `text_for_context` (canonical tokens)
- chunk size uniformity warnings (section 5.1)
- **oversize rate**: fraction of queries where the first context unit alone exceeds each spot budget
- truncated chunks per embedder
- embedding tokens and index size
- parse, chunk, embed and search time
- label alignment failures per parser, and labels aligned inside removed ranges
- unjudged fraction, when `judged` ranges exist (section 7.7)

## 9. Evaluation stages

### Stage 1 — Per class

Run each class's grid in isolation and find the best configuration per class. This is the first milestone.

### Stage 2 — End to end

Freeze the per-class winners and evaluate how queries reach the right index, using a query set mixing all classes:

- **Routing**: classify the query, search one index. Router accuracy is measured separately and its errors show up in end-to-end recall.
- **Fan-out + RRF**: search all indexes in parallel, merge by rank. Raw scores are not comparable across indexes (different models and chunk sizes), so score-based merging is excluded.
- **Fan-out + reranker**: merge candidates and rerank with a cross-encoder.

Per-class winners don't guarantee the best combined system; stage 2 is where that shows up. It requires a `class` label on every query, including production ones.

## 10. CLI

The package, import name, CLI command and repository are all `retrievalbench`, with no hyphen or underscore anywhere. `rbench` is installed as a short alias for the CLI.

| Command                                    | Purpose                                                            | Milestone |
| ------------------------------------------ | ------------------------------------------------------------------ | --------- |
| `retrievalbench parse <config>`            | Parse documents, cache trees, report parse stats                   | M0        |
| `retrievalbench inspect <doc>`             | Show a document's tree, chunks and context units per strategy      | M0        |
| `retrievalbench validate-queries <config>` | Check schema, alignment per parser, staleness and splits           | M0        |
| `retrievalbench run <config>`              | Run the grid on a split (default `dev`; uses caches)               | M0        |
| `retrievalbench report <run>`              | Produce the results report                                         | M1        |
| `retrievalbench compare <run> <run>`       | Compare two runs on the same eval version                          | M1        |
| `retrievalbench gen-queries <config>`      | Generate synthetic queries                                         | M4        |
| `retrievalbench import-langfuse <config>`  | Pull, clean, rewrite and deduplicate production queries            | M4        |
| `retrievalbench label <config>`            | Pooled labeling workflow with LLM proposals and human confirmation | M4        |

## 11. Outputs

Each run writes a directory containing:

- `results.parquet`: per-query, per-configuration metrics (raw data for custom analysis)
- `curves.parquet`: per-query, per-configuration recall-vs-tokens curves
- `summary.csv`: per-configuration metrics with confidence intervals
- `report.md` or `report.html`: tables per class, broken down by query type, source and language, with recall curves, the baseline comparison, source agreement, warnings and cost stats
- `run.json`: config, tool version, tokenizer, split, eval set version, corpus version, cache keys, and the log of test-split evaluations

## 12. Milestones

| #   | Scope                                                                                                                                                                                                                                                                   |
| --- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| M0  | `DocumentTree`; one parser; `fixed` chunker; one embedder; dense retrieval; canonical tokenizer; query schema, span resolver and `validate-queries`; recall curve and span metrics. Manuals only, with the existing synthetic queries converted to the span schema. First check that they were not generated from the current system's chunks. |
| M1  | `recursive`, `hierarchical`, `parent_child` chunkers; context expansion; second parser; BM25 and hybrid; dev/test splits; cluster bootstrap and paired comparisons; size warnings; source agreement; report.                                                            |
| M2  | `office` and `config_tables` classes.                                                                                                                                                                                                                                   |
| M3  | Stage 2: routing, fan-out + RRF, reranker.                                                                                                                                                                                                                              |
| M4  | Evaluation set tooling (advanced phase): `gen-queries`, `import-langfuse`, `label` with pooling and `judged` ranges, relabeling workflow for stale labels.                                                                                                             |

## 13. Risks and open questions

- **Parse quality dominates.** A bad heading tree silently ruins hierarchical chunking. Mitigation: two parsers in the grid, alignment-failure stats, `inspect` command.
- **Biased evaluation sets.** The framework can't fix a biased set. It can only expose it through the source split and rank agreement. Only real queries can confirm synthetic results.
- **Near-duplicate manuals across product versions.** Groups of equivalent alternatives cover duplicate text that is equally valid. Open: should `product_version` be a hard filter, a metadata field, or part of the relevance label?
- **Tables and images.** v1 renders tables as text and ignores image content. Open: is that enough for `table_lookup` queries? Fuzzy alignment of table cells against a Markdown rendering is a likely source of alignment failures.
- **Reranker in stage 1.** Cross-encoders typically truncate inputs around 512 tokens, so the best chunker without a reranker may not be the best with one. Open: if production reranks, add the reranker as a stage-1 retriever option.
- **Few documents per class.** A split grouped by document and the cluster bootstrap both need enough documents. With few manuals, splits may be unbalanced and CIs wide. Open: fall back to grouping by section?
- **Test-set discipline.** Repeated evaluation on `test` erodes its value. The tool logs test usage, but the discipline is the user's.
- **Choice of `max_budget`.** The AUC depends on the range it integrates over. Default to a realistic production context size, and report the AUC for each budget range used.
- **Labeling effort (M4).** Pooled labeling over a large grid produces big candidate pools. Open: cap pool size per query, or pool only from a representative subset of configurations?
- **Overlap threshold.** The 0.5 value for hit and MRR is arbitrary. Report sensitivity to it, and rely mainly on `span_recall_auc`.

## 14. References

- BEIR — heterogeneous benchmark for retrieval evaluation
- MTEB — Massive Text Embedding Benchmark
- Ragas, LlamaIndex evaluation modules — RAG evaluation frameworks
- Chroma research on evaluating chunking strategies (token-level metrics)
- TREC pooling methodology for building relevance judgments; bpref and condensed-list metrics for incomplete judgments
- Reciprocal Rank Fusion (Cormack et al., 2009)
- W3C Web Annotation Data Model — `TextQuoteSelector` (prefix/exact/suffix anchoring)
- Holm, S. (1979) — sequentially rejective multiple test procedure
