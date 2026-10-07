# Chunking & Embedding Evaluation Tool — Design Doc

> Working name: `chunkeval` (placeholder)
> Status: Draft v0.1 — 2026-10-07

## 1. Summary

Most RAG systems chunk and embed every document the same way, by habit. This tool replaces habit with measurement: given a real corpus and an evaluation set of queries, it runs a grid of **parsing × chunking × embedding × retrieval** configurations and reports how well each one retrieves the right content.

It is a standalone CLI driven by config files. It can also be used as a pluggable stage (parse → chunk → embed) inside a bigger project.

## 2. Goals and non-goals

### Goals

- Ingest real documents and evaluate several chunking strategies and embedding models against each other on the same queries.
- Treat **document class** as the unit of evaluation: each class gets its own grid and results.
- Use **chunk-independent ground truth**, so different chunking strategies can be compared fairly.
- Produce **statistics with confidence intervals**, broken down by document class, query type and query source.
- Make every pipeline stage pluggable (parsers, chunkers, embedders, retrievers).

### Non-goals (for now)

- Measuring answer quality (generation). Retrieval only.
- Automatically recommending a configuration. The tool produces stats; humans decide.
- Running as a service. CLI + config files only.

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
DocumentTree  ──────────────►  cached by (doc_hash, parser, params)
   │  chunk (per chunker)
   ▼
Chunks with source offsets ──►  cached by (tree_key, chunker, params)
   │  embed (per model)
   ▼
Vectors ─────────────────────►  cached by (chunk_hash, model)
   │  index + retrieve (per retriever)
   ▼
Ranked chunks per query
   │  evaluate
   ▼
Metrics + reports
```

Each stage is cached by content hash. Grids grow quickly, and embedding is the most expensive step, so a new chunker must never force re-parsing, and a new retriever must never force re-embedding.

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
      page
      children
  ]
  removed: boilerplate ranges (headers, footers, page numbers, repeated warnings)
```

Requirements:

- Every node keeps its **char range** and **page**, so retrieved chunks can be mapped back to the source.
- Tables are preserved as structured data plus a text rendering (Markdown or similar).
- Images keep their captions and position. Image content (OCR or vision descriptions) is optional and out of scope for v1.
- Boilerplate is recorded and removed, not silently deleted, so its effect can be inspected.

### 4.3 Chunks

```
Chunk
  chunk_id, chunk_hash
  doc_id
  source_ranges: [char ranges in canonical text]   # may be non-contiguous
  text_for_embedding   # may include a prepended breadcrumb
  text_for_context     # what is returned to the LLM (parent-child may differ)
  metadata: section path, page(s), node kinds
```

Only `source_ranges` count for evaluation. Text added by a chunker, such as a breadcrumb, is never scored as retrieved content.

### 4.4 Plugin interfaces

```python
class Parser:
    def parse(self, path, params) -> DocumentTree: ...

class Chunker:
    def chunk(self, tree: DocumentTree, params) -> list[Chunk]: ...

class Embedder:
    max_tokens: int
    def embed_documents(self, texts) -> Vectors: ...
    def embed_query(self, text) -> Vector: ...   # handles asymmetric prefixes (e.g. "query:"/"passage:")

class Retriever:
    def index(self, chunks, vectors): ...
    def search(self, query, query_vector, k) -> list[ScoredChunk]: ...
```

Embedders must **report truncation**: if a chunk exceeds `max_tokens`, it is counted and shown in the report. Silent truncation would bias results against large-chunk strategies.

### 4.5 Retrieval during evaluation

Use **exact (brute-force) vector search** during evaluation. Approximate indexes add their own recall loss, which would be confused with chunking or embedding effects. Corpus sizes per class should make exact search feasible.

Retrievers in v1:

- `dense`: cosine similarity
- `bm25`: lexical
- `hybrid`: dense + BM25, fused with Reciprocal Rank Fusion (weighted fusion as an option)

## 5. Initial grid for `manuals`

| Axis         | Options                                                                                          |
| ------------ | ------------------------------------------------------------------------------------------------ |
| Parser       | Docling; a PyMuPDF-based parser                                                                  |
| Chunker      | `fixed` (baseline); `recursive`; `hierarchical` (heading-based, with breadcrumb); `parent_child` |
| Embedder     | 2–3 models, e.g. one open-source multilingual, one API-based                                     |
| Retriever    | `dense`, `hybrid`                                                                                |
| Token budget | e.g. 1000, 2000                                                                                  |

Chunker details:

- **fixed**: N tokens with overlap. The baseline every other strategy must beat.
- **recursive**: split on paragraphs, then sentences, up to a max size.
- **hierarchical**: a section is the unit. Split only if it exceeds the max size. Prepend the breadcrumb ("Installation > Network > Proxy settings") to `text_for_embedding`.
- **parent_child**: embed small chunks, return the enclosing section as `text_for_context`. The token-budget metrics keep this comparison fair.

## 6. Configuration

```yaml
run_name: manuals-baseline
corpus:
  root: ./docs
  version: 2026-10-01
queries: ./eval/queries.v3.jsonl

classes:
  manuals:
    match: "manuals/**/*.pdf"
    grid:
      parsers:
        - docling: {}
        - pymupdf: {}
      chunkers:
        - fixed: { size: [256, 512], overlap: 64 }
        - recursive: { max_tokens: 512 }
        - hierarchical: { max_tokens: 800, breadcrumb: true }
        - parent_child: { child_tokens: 200, parent: section }
      embedders: [bge-m3, text-embedding-3-small]
      retrievers: [dense, hybrid]
  office:
    match: "office/**/*.{docx,pptx}"
    grid: { ... }

eval:
  token_budgets: [1000, 2000]
  metrics: [span_recall, span_precision, span_iou, hit_rate, mrr]
  overlap_threshold: 0.5 # fraction of a span a chunk must cover to count as a hit
  bootstrap_samples: 1000
  baseline: { chunker: fixed, size: 512 }
```

List-valued parameters expand into the grid. Each combination is a **configuration**.

## 7. Ground truth

### 7.1 Principle: labels don't depend on chunks

A label like "chunk #42 is relevant" only exists for one chunking strategy. Instead, ground truth is a set of **relevant spans in the source document**. Every strategy is scored by how well its retrieved chunks cover those spans.

### 7.2 Anchoring spans across parsers

The parser is itself a grid axis, so character offsets in one parser's output don't apply to another's. Labels are therefore stored as **page + exact answer text**, and a resolver aligns them (fuzzy match) to each parser's canonical text.

Alignment failures are reported per parser. They also work as a **parse quality signal**: a parser that loses or garbles labeled text will show it here.

### 7.3 Query sources

| Source       | How it's made                                 | Strengths                   | Weaknesses                                              |
| ------------ | --------------------------------------------- | --------------------------- | ------------------------------------------------------- |
| `synthetic`  | LLM generates questions from sampled spans    | Cheap, thousands of queries | Simple lookups; reuses document wording (flatters BM25) |
| `curated`    | Hand-written and hand-labeled by the team     | Trustworthy                 | Small                                                   |
| `production` | Real user queries from Langfuse, then labeled | Realistic                   | Needs labeling and cleaning                             |

The mix is the point. If synthetic and real queries **rank configurations the same way**, the synthetic set can be trusted at scale. Reports always split results by source to check this.

### 7.4 Synthetic generation rules

- Sample spans **independently of any chunking strategy**. Generating from a strategy's chunks favors that strategy.
- Ask for paraphrased questions to reduce lexical overlap with the source.
- Generate several query types (see 7.6), including multi-span queries built from two related sections.
- Filter with an LLM judge for answerability and specificity.
- Don't use the embedding models under test for generation or filtering.

### 7.5 Labeling production queries (pooling)

Production queries arrive without labels. Labeling only from the production retriever's results would bias the set toward that configuration. Instead, pool candidates TREC-style:

1. Run the query against **every configuration** in the grid.
2. Take the union of top-k results (mapped back to source spans).
3. An LLM proposes the relevant spans from the pool.
4. A human confirms or edits.

Cleaning before labeling: remove duplicates, PII and chit-chat. Keep unanswerable queries tagged `no_answer`; they're useful if score thresholds are ever evaluated.

### 7.6 Query schema

```json
{
  "query_id": "q_0142",
  "query": "How do I configure a proxy for firmware updates?",
  "class": "manuals",
  "type": "single_fact",
  "source": "production",
  "added_in_version": "v3",
  "relevant": [
    {
      "doc_id": "manual_x200_en",
      "doc_hash": "sha256:...",
      "page": 47,
      "text": "To use a proxy server, open Settings > Network ..."
    }
  ],
  "metadata": { "product_version": "X200 v4" }
}
```

Query types (initial set): `single_fact`, `multi_span`, `table_lookup`, `procedural`, `no_answer`.

### 7.7 Versioning and staleness

- The eval set is versioned (`queries.v3.jsonl`). Every report records the eval set version and corpus version. Results from different versions are not compared directly.
- Each label stores the source `doc_hash`. When a document changes, its labels are flagged stale and go back to labeling.

## 8. Metrics

### 8.1 Token-budget retrieval

Comparing top-k across strategies is unfair: five 200-token chunks and five 1000-token chunks are different amounts of context. Instead, retrieved chunks are taken in rank order **until a token budget B is filled**, using `text_for_context` length (so parent-child pays for the larger text it returns).

Let `R` be the set of source tokens in relevant spans, and `T` the set of source tokens covered by retrieved chunks within budget B. `T` is a set, so overlapping chunks don't double-count. Overlap still consumes budget, which correctly penalizes heavy overlap.

| Metric             | Definition                                                                            |
| ------------------ | ------------------------------------------------------------------------------------- |
| `span_recall@B`    | \|R ∩ T\| / \|R\|                                                                     |
| `span_precision@B` | \|R ∩ T\| / \|T\|                                                                     |
| `span_iou@B`       | \|R ∩ T\| / \|R ∪ T\|                                                                 |
| `hit_rate@B`       | fraction of queries where every relevant span is covered above `overlap_threshold`    |
| `mrr`              | reciprocal rank of the first chunk covering a relevant span above `overlap_threshold` |

`span_recall@B` is the headline metric: how much of what the LLM needs makes it into the context.

### 8.2 Statistical reporting

- **Bootstrap confidence intervals** over queries for every metric.
- **Paired comparison against the baseline** (paired bootstrap on per-query differences). With a few hundred queries, a two-point difference is often noise, and the report must say so.

### 8.3 Cost and diagnostic stats

Reported per configuration, without being combined into a score:

- number of chunks, mean and max chunk length
- truncated chunks per embedder
- embedding tokens and index size
- parse, chunk, embed and search time
- label alignment failures per parser

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

| Command                              | Purpose                                                            |
| ------------------------------------ | ------------------------------------------------------------------ |
| `chunkeval parse <config>`           | Parse documents, cache trees, report parse stats                   |
| `chunkeval inspect <doc>`            | Show a document's tree and chunks per strategy (debugging)         |
| `chunkeval gen-queries <config>`     | Generate synthetic queries                                         |
| `chunkeval import-langfuse <config>` | Pull, clean and deduplicate production queries                     |
| `chunkeval label <config>`           | Pooled labeling workflow with LLM proposals and human confirmation |
| `chunkeval validate-queries <file>`  | Check schema, alignment and staleness                              |
| `chunkeval run <config>`             | Run the grid (uses caches)                                         |
| `chunkeval report <run>`             | Produce the results report                                         |
| `chunkeval compare <run> <run>`      | Compare two runs on the same eval version                          |

## 11. Outputs

Each run writes a directory containing:

- `results.parquet`: per-query, per-configuration metrics (raw data for custom analysis)
- `summary.csv`: per-configuration metrics with confidence intervals
- `report.md` or `report.html`: tables per class, broken down by query type and source, with the baseline comparison and cost stats
- `run.json`: config, tool version, eval set version, corpus version, cache keys

## 12. Milestones

| #   | Scope                                                                                                                               |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- |
| M0  | `DocumentTree`, one parser, `fixed` chunker, one embedder, dense retrieval, span metrics. Manuals only, existing synthetic queries. |
| M1  | `recursive`, `hierarchical`, `parent_child` chunkers; second parser; BM25 and hybrid; bootstrap CIs; report.                        |
| M2  | Query tooling: `gen-queries`, `import-langfuse`, `label` with pooling, versioning and staleness checks.                             |
| M3  | `office` and `config_tables` classes.                                                                                               |
| M4  | Stage 2: routing, fan-out + RRF, reranker.                                                                                          |

## 13. Risks and open questions

- **Parse quality dominates.** A bad heading tree silently ruins hierarchical chunking. Mitigation: two parsers in the grid, alignment-failure stats, `inspect` command.
- **Synthetic bias.** Mitigated by span-based generation, paraphrasing and the source split in reports, but only real queries can confirm it.
- **Near-duplicate manuals across product versions.** A "right text, wrong version" result may be scored inconsistently. Open: should `product_version` be a hard filter, a metadata field, or part of the relevance label?
- **Tables and images.** v1 renders tables as text and ignores image content. Open: is that enough for `table_lookup` queries?
- **Labeling effort.** Pooled labeling over a large grid produces big candidate pools. Open: cap pool size per query, or pool only from a representative subset of configurations?
- **Overlap threshold.** The 0.5 value for hits is arbitrary. Report sensitivity to it, or rely mainly on span recall.

## 14. References

- BEIR — heterogeneous benchmark for retrieval evaluation
- MTEB — Massive Text Embedding Benchmark
- Ragas, LlamaIndex evaluation modules — RAG evaluation frameworks
- Chroma research on evaluating chunking strategies (token-level metrics)
- TREC pooling methodology for building relevance judgments
- Reciprocal Rank Fusion (Cormack et al., 2009)
