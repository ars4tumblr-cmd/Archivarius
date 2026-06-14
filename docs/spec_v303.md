# Firemná Pamäť (Archivarius)
## Fundamental Technical Specification

---

**Version:** 3.0.3  
**Status:** Final + Security Assurance update  
**Document Type:** Tech Spec / Architecture Design / Roadmap  
**Principle:** Core = Search Engine. Intelligence is added later, when there is a base.

> **Changes from v3.0.2:** Added systemic representation of security aspects: §4.6а Security Assurance & Corporate Readiness, production hardening in §4.4, data governance notes in §4.3, clarified principle of locality in §7. The goal of changes — to capture system security properties in evidentiary model: assertion → architectural proof → control → limitation.

> **Changes from v3.0.1:** Added three new architectural components from MVP testing experience (June 2026): §3.1г Table Parsing PP-Line (tables as JSON objects), §3.4в Metadata Extraction (deterministic code filter), §3.5г Query Routing Algorithm (dynamic weights instead of static RRF). Documented lessons from cross-lingual ranking problems in MVP.

> **Changes from v2.5 (v3.0.1):** Integrated extended optimization specification. Added: Feedback System (§3.11), Query Cache (§4.7), Partial Reindex (§3.4а), Index Consistency (§3.4б), Query Intent Adaptation (§3.5а), Metadata Boosting (§3.5б), Score Explainability (§3.5в), Embedding Profiles (§3.3а). Updated resource model (§4.2). Clarified Roadmap (§8). Everything else — unchanged.

---

## Table of Contents

1. [Goal and philosophy of the system](#1-goal-and-philosophy-of-the-system)
2. [Architectural model](#2-architectural-model)
3. [MODULES (Business Logic)](#3-modules-business-logic)
   - 3.1 [Ingestion Module](#31-ingestion-module-data-loading)
   - 3.2 [Chunking Module](#32-chunking-module)
   - 3.3 [Embedding Module](#33-embedding-module)
   - 3.3а [Embedding Profiles](#33а-embedding-profiles-new-subsection)
   - 3.4 [Indexing Module](#34-indexing-module)
   - 3.4а [Partial Reindex](#34а-partial-reindex-new-subsection)
   - 3.4б [Index Consistency](#34б-index-consistency-new-subsection)
   - 3.5 [Search Module](#35-search-module)
   - 3.5а [Query Intent Adaptation](#35а-query-intent-adaptation-new-subsection)
   - 3.5б [Metadata Boosting](#35б-metadata-boosting-new-subsection)
   - 3.5в [Score Explainability](#35в-score-explainability-new-subsection)
   - 3.5г [Query Routing Algorithm](#35г-query-routing-algorithm-new-subsection)
   - 3.6 [Reranking Module](#36-reranking-module)
   - 3.7 [API Module](#37-api-module)
   - 3.8 [Frontend Module](#38-frontend-module)
   - 3.9 [Web Admin](#39-web-admin)
   - 3.10 [Translation Module (Optional Service)](#310-translation-module-optional-service)
   - 3.11 [Feedback System (new module)](#311-feedback-system-new-module)
4. [ASPECTS (System Concerns)](#4-aspects-system-concerns)
   - 4.1 [Async Execution](#41-async-execution)
   - 4.2 [Resource Management](#42-resource-management)
   - 4.3 [Storage Strategy](#43-storage-strategy)
   - 4.4 [Deployment](#44-deployment)
   - 4.5 [Monitoring](#45-monitoring)
   - 4.6 [Security](#46-security)
   - 4.6а [Security Assurance & Corporate Readiness](#46а-security-assurance--corporate-readiness)
   - 4.7 [Query Cache (new aspect)](#47-query-cache-new-aspect)
5. [Processing pipeline](#5-processing-pipeline)
6. [What is excluded and why](#6-what-is-excluded-and-why)
7. [Key principles](#7-key-principles)
8. [Roadmap](#8-roadmap)

---

## 1. Goal and philosophy of the system

### 1.1 What the system is for

**Firemná Pamäť** is a corporate search engine over a document repository.

Three tasks it solves:

- **Centralized storage** — single entry point for all corporate documents (PDF, DOCX, TXT, HTML).
- **Fast search** — full-text (by keywords) and semantic (by content). If a document exists — it will be found.
- **Stability on limited resources** — the system should work stably on an office server with 16–32 GB RAM without GPU.

### 1.2 What the system is NOT

The system is not an AI assistant and not a chatbot. It is a **search infrastructure**.

> **Principle:** Core = Search Engine, not "AI combo". Complex AI logic is not built into the core. It is connected externally as a separate layer after the base works stably.

This principle protects from:
- memory overload (LLM + embedder + reranker simultaneously = death on 16 GB RAM),
- instability (AI pipelines break, search doesn't),
- complexity of maintenance.

### 1.3 Who should read this

| Role | What to look for |
|------|-----------------|
| Developer | Module logic, API contracts, technical decisions |
| DevOps | Deployment, Resource Management, Monitoring aspects |
| Analyst / PM | Roadmap, exclusions and decision reasoning |
| New contributor | Section 2 (architecture) + section 7 (principles) |

---

## 2. Architectural model

### 2.1 Two levels of description

The system is described by two orthogonal layers:

```
┌─────────────────────────────────────────────────────────────────────┐
│                            MODULES                                   │
│  Ingestion → Chunking → Embedding → Indexing                         │
│  Search → Reranking → API → Frontend                                 │
│  Feedback System                                                     │
│  (what the system does, business logic)                              │
├─────────────────────────────────────────────────────────────────────┤
│                            ASPECTS                                   │
│  Async | Resources | Storage | Deploy | Monitor | Security | Cache   │
│  (how the system does it, system concerns)                           │
└─────────────────────────────────────────────────────────────────────┘
```

**Modules** are isolated units of business logic. Each module performs one specific task, takes a clearly defined input and returns a clearly defined output. Modules are interchangeable — you can replace embedder or reranker without touching the rest.

**Aspects** are cross-cutting characteristics of the system. They don't "live" in one module, they describe how the entire system behaves under load, during deployment, when failures occur.

### 2.2 Data Flow

```
[File System]
       │
       ▼
 [Ingestion] ──► [Chunking] ──► [Embedding] ──► [Indexing]
                                                     │
                                               ┌─────┴─────┐
                                           [BM25 idx] [Vector idx]
                                               └─────┬─────┘
                                                     │
 [User Query] ──► [Cache?] ──► [Search] ──► [Merge] ──► [Reranking] ──► [API] ──► [Frontend]
                                                                              │
                                                                         [Feedback]
                                                                              │
                                                                    [Relevance Signals]
```

There are two independent streams:
- **Indexing pipeline** — background, runs when adding/changing documents.
- **Search pipeline** — online, runs on each user query.

They do not block each other.

Feedback System is an asynchronous third stream: collects user behavior signals and gradually influences relevance weights.

---

## 3. MODULES (Business Logic)

---

### 3.1 Ingestion Module (Data Loading)

**Purpose:** take a file or directory and return a normalized document object.

#### Input

- Single file: `pdf`, `docx`, `txt`, `html`
- Directory (recursive traversal)
- Request metadata: who uploaded, when, from where

#### Output

```python
NormalizedDocument:
  id: str              # SHA-256 of file content
  source: str          # path to original file
  text: str            # clean UTF-8 text
  metadata: dict       # file type, size, date, author (if in metadata)
  timestamp: datetime  # indexing time
```

#### Processing logic

**Step 1 — File type determination**

Type is determined by real MIME-type (via `python-magic`), not just extension. This protects from files with incorrect extensions.

**Step 2 — Text extraction**

| Format | Parser | Notes |
|--------|--------|-------|
| PDF | `pdfplumber` (main), `PyPDF2` (fallback) | `pdfplumber` better handles tables and columns |
| DOCX | `python-docx` (xml extractor) | Extracts text from paragraphs, tables, headers |
| TXT | raw read | Encoding: UTF-8, with fallback to `chardet` |
| HTML | `BeautifulSoup4` | Removes tags, keeps text |

> **IMPORTANT:** OCR is absent. Scanned PDF without text layer will be ignored or returned with warning. This is a deliberate decision — OCR adds instability and consumes resources.

**Step 3 — Text normalization (Text Encoding & Normalization Pipeline)**

Normalization is not "cleaning up spaces". It's a guarantee that stable, predictable text reaches the embedder and search index. Without this step, the system can give different results for the same document depending on how it was saved or which OS it came from.

Pipeline executes strictly sequentially:

**3.1 — Determining input text encoding**

Parsers (pdfplumber, python-docx) return Python `str` (Unicode) — encoding is already handled by the library. But for `TXT` and some `HTML` files, encoding is unknown and determined as:

```python
import chardet

def detect_encoding(raw_bytes: bytes) -> str:
    result = chardet.detect(raw_bytes)
    encoding = result.get("encoding") or "utf-8"
    confidence = result.get("confidence", 0)
    if confidence < 0.7:
        encoding = "utf-8"   # low confidence → safe fallback
    return encoding
```

| Source | Input encoding | What we do |
|--------|-----------------|-----------|
| PDF (pdfplumber) | Unicode str | nothing, already correct |
| DOCX (python-docx) | Unicode str | nothing, already correct |
| TXT | unknown | `chardet` → decode → UTF-8 |
| HTML | may be in `<meta charset>` | `BeautifulSoup` reads charset, fallback `chardet` |

**3.2 — Converting to UTF-8 with protection from incorrect bytes**

```python
# Safe conversion: incorrect bytes are discarded, don't break the process
text = raw_text.encode("utf-8", errors="ignore").decode("utf-8")
```

> **Why `errors="ignore"` and not `errors="replace"`?** Replace inserts character `U+FFFD` (&#65533;) — it gets into the index and breaks the embedding vector. Ignore simply discards the incorrect byte. Losing one character is better than "garbage" in semantic space.

**3.3 — Unicode NFC normalization**

```python
import unicodedata
text = unicodedata.normalize("NFC", text)
```

The same letter can be represented in Unicode in different ways:

- `é` = one character (NFC, precomposed)
- `é` = `e` + combining accent (NFD, decomposed)

Without normalization, a search for the word `résumé` won't find a document where this word is written in NFD form, although they look the same visually. NFC is the storage standard, it's the most compact and compatible.

**3.4 — Removing incorrect and control characters**

```python
import re

# Null bytes and control characters (except \n, \r, \t)
text = re.sub(r'[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]', '', text)

# Private Unicode areas (PUA) — often PDF font garbage
text = re.sub(r'[\uE000-\uF8FF]', '', text)

# Zero-width characters (invisible, but break tokenization)
text = re.sub(r'[\u200B\u200C\u200D\uFEFF]', '', text)
```

> **Why PUA?** PDF files often use private Unicode areas for custom fonts. After parsing it looks like valid characters, but it's garbage — they have no meaning and will definitely break embedding.

**3.5 — Normalizing whitespace while preserving structure**

```python
# Multiple spaces → one
text = re.sub(r'[ \t]+', ' ', text)

# More than two line breaks → two (preserve paragraph boundaries)
text = re.sub(r'\n{3,}', '\n\n', text)

# Spaces at start/end of lines
text = '\n'.join(line.strip() for line in text.splitlines())

# Final trim
text = text.strip()
```

> **Why not `strip()` on the entire text at once?** The `\n\n` structure between paragraphs is a signal for Chunking Module (step 3.2 in pipeline). If all line breaks are merged into one — chunking loses paragraph boundaries and breaks text arbitrarily.

**Result of step 3 — guarantees on output:**

| Property | Guarantee |
|----------|-----------|
| Encoding | UTF-8, valid |
| Unicode form | NFC (precomposed) |
| Null bytes | absent |
| Control chars | absent (except `\n`, `\t`) |
| PUA characters | absent |
| Zero-width | absent |
| Paragraph structure | preserved (`\n\n`) |
| Language composition | any (UA, SK, EN, PL, DE and mix) |

**Language support:** UTF-8 covers all languages used by the system (UA, SK, EN, PL, DE) and any mix in one document. The E5-Large embedding model processes multilingual text natively — normalization doesn't affect language, it only affects representation stability.

**Step 4 — Deduplication (Incremental Indexing)**

For each file, `SHA-256` is computed from content. Hashes are stored in `./data/file_hashes.json`. If hash matches — file is skipped, resources are not wasted.

> **Why SHA-256 and not file modification date?** File date changes when copying, even if content hasn't changed. SHA-256 guarantees reindexing only on real changes.

**Step 5 — Pre-index Quality Filter**

After normalization, before sending to Chunking, document passes quality check:

| Rejection condition | Action |
|---------------------|--------|
| Text < 50 chars after normalization | `skip` + log warning "too short after normalization" |
| Readability score below threshold (OCR noise, tables without text) | `skip` + log warning "low readability score" |
| Entire text consists of PUA | `skip` + log error "no readable text extracted" |
| Non-alphabetic character ratio > 60% | `skip` + log warning "non-alphabetic content dominant" |

> **Why this filter?** Noise in the index is expensive: one bad document adds dozens of irrelevant chunks, diluting results and degrading embedding space quality. Better to reject on input and log than to fight noise in search.

#### Error handling

| Situation | Behavior |
|-----------|----------|
| File cannot be read | Log + skip, doesn't break entire batch |
| Empty text after parsing | Log warning, document not indexed |
| Unknown format | Log + skip |
| File too large (> 100 MB) | Warning, process in parts |
| `chardet` didn't determine encoding (confidence < 0.7) | Fallback UTF-8, log warning with filename |
| Text after normalization < 50 chars | Log warning "too short after normalization", skip |
| Entire text consists of PUA characters | Log error "no readable text extracted" — typical for encrypted PDF |

#### Configuration

```yaml
# settings.yaml
ingestion:
  supported_formats: [pdf, docx, txt, html]
  max_file_size_mb: 100
  hash_store_path: ./data/file_hashes.json
  normalization:
    unicode_form: NFC
    encoding_detection_confidence_threshold: 0.7
    min_text_length_chars: 50
    strip_pua: true
    strip_zero_width: true
  quality_filter:
    enabled: true
    min_readability_score: 0.3       # 0.0–1.0, heuristic
    max_non_alpha_ratio: 0.6         # non-alphabetic character ratio

paths:
  documents_dir: ./documents
```

---

### 3.1г Table Parsing PP-Line (new subsection)

**Purpose:** process tables in documents as structured JSON objects — not as plain text.

#### Current approach problem

When `pdfplumber` or `python-docx` extracts a table as plain text, structure is lost: cells blend without context, column headers separate from values. Result in index looks like:

```
"Product Quantity Price Widget A 1500 €45 000 Widget B 800 €22 000"
```

This is unreadable for embedding model and BM25. Table semantics is lost.

#### Solution: three-stage Table PP-Line

```
[Detecting table in document]
         │
         ▼
[Stage 1 — Table Metadata]
  What is this table about?
  → "Quarterly sales report Q3 2025"
         │
         ▼
[Stage 2 — Key Fields Recognition]
  Identifying column headers and key dimensions
  → ["Product", "Quantity", "Revenue"]
         │
         ▼
[Stage 3 — Structured Cell Parsing]
  Parsing cells WITH context (column × row)
  → JSON object with preserved structure
```

#### Output: JSON object per table

```json
{
  "table_type": "sales_report",
  "table_meta": "Quarterly sales report Q3 2025",
  "source_document": "reports/Q3_2025.pdf",
  "page": 4,
  "headers": ["Product", "Quantity", "Revenue"],
  "rows": [
    {"Product": "Widget A", "Quantity": "1500", "Revenue": "€45 000"},
    {"Product": "Widget B", "Quantity": "800",  "Revenue": "€22 000"}
  ]
}
```

#### How JSON is indexed

| What | Where | How |
|------|-------|-----|
| `table_meta` (table description) | Vector Index | Semantic search by content |
| Cell values (`Widget A`, `€45 000`) | BM25 Index | Exact search for numbers and codes |
| `table_type`, `source_document`, date | `metadata_tags` | Deterministic filter (§3.4в) |

#### Configuration

```yaml
ingestion:
  table_parsing:
    enabled: true
    min_rows: 2
    min_cols: 2
    meta_extraction: auto
    output_format: json
```

> **Phased plan:** Phase 1 — detection + JSON conversion (Regex/heuristic). Phase 2 — `table_meta` generated by local LLM.

---

### 3.2 Chunking Module

**Purpose:** split long document text into smaller fragments (chunks) suitable for vectorization and search.

#### Input

- `NormalizedDocument` from previous step

#### Output

```python
List[Chunk]:
  id: str           # SHA-256 of chunk text
  document_id: str  # reference to parent document
  text: str         # fragment text
  index: int        # ordinal number in document
  metadata: dict    # inherited from document + position
```

#### Algorithm

**Parameters:**
- `chunk_size`: 300–800 tokens (default: 512)
- `overlap`: 10–20% of chunk_size (default: 50 tokens)

**Strategy (in order of priority):**

1. **Header-based split** — if document has clear headers (H1, H2, sections with dots), split by them. Semantic integrity of sections is preserved.

2. **Heuristic semantic split** — if headers not detected: analyze similarity between adjacent sentences (cosine similarity by sentence embeddings). Break where similarity drops sharply — this is a semantic block boundary. More expensive than paragraph-based, but more accurate.

3. **Paragraph-based split** — fallback if semantic split unavailable or disabled. Split by paragraphs (`\n\n`).

4. **Token-based split** — final fallback for monolithic text without structure. Uniform split by tokens with overlap.

> **Why overlap?** Without it, answer to query might "break" between two chunks. For example, definition starts at end of chunk 3, explanation — at start of chunk 4. With 50 token overlap both chunks contain both definition and explanation start.

> **Risk of heuristic semantic split:** over-splitting — too small chunks lose context. Controlled by `min_chunk_size` parameter. If chunk after semantic split is below threshold — it's merged with next.

#### Configuration

```yaml
chunking:
  chunk_size: 512           # tokens
  chunk_overlap: 50         # tokens
  max_chunk_size: 800       # hard maximum — protection from over-context
  min_chunk_size: 50        # chunks smaller — discarded or merged
  strategy: auto            # auto | headers | semantic | paragraphs | tokens
  semantic_split:
    enabled: true           # heuristic semantic split
    similarity_threshold: 0.5  # similarity drop threshold for break
```

---

### 3.3 Embedding Module

**Purpose:** transform text fragment into numeric vector representing its semantic content.

#### Input

- One chunk or batch of chunks (batch recommended)

#### Output

```python
List[EmbeddingVector]:
  chunk_id: str
  vector: List[float]  # dimensionality depends on model
```

#### Technical decisions

**Model:** `intfloat/multilingual-e5-large`

| Parameter | Value |
|-----------|-------|
| Architecture | Bi-Encoder (Sentence Transformer) |
| Vector dimensionality | 1024 |
| Languages | 100+ (including UK, SK, EN, PL, DE) |
| Model size | ~560 MB |
| Location | `./models/multilingual-e5-large/` |

> **Why E5-Large?** It ranks top in MTEB rankings for multilingual search. Key advantage: cross-lingual search without translation — query in English will find relevant fragment in Slovak document.

**Multilingual behavior (real, not ideal):**

E5-Large handles vector representation well in cross-lingual mode. However BM25 degrades with queries in one language over documents in another — exact token match doesn't work. Consequence: for multilingual queries vector search becomes main channel, BM25 — auxiliary. This is accounted for in Query Intent Adaptation (§3.5а).

**Operation mode: Singleton + CPU-first**

Model is loaded into memory once at service start and stays there. Repeated loads don't happen.

```python
device: "cpu"   # default for office servers without GPU
# device: "cuda" # optional, if NVIDIA GPU available
```

**Batching**

Chunks are processed in batches, not one by one:
- `batch_size`: 16–32 (default: 16 for CPU, 32 for GPU)
- Reducing batch_size lowers peak RAM consumption

**Caching**

If chunk with this `chunk_id` already in index — embedding is not recomputed. Critical for incremental indexing: when updating one file, rest of documents not recomputed.

**Offline Mode**

```bash
HF_HUB_OFFLINE=1   # forbidden to access Hugging Face Hub
TRANSFORMERS_OFFLINE=1
```

Model loaded locally. No network calls during operation.

#### Configuration

```yaml
embedding:
  model_name: intfloat/multilingual-e5-large
  model_path: ./models/multilingual-e5-large
  device: cpu           # cpu | cuda
  batch_size: 16
  normalize_embeddings: true
  offline_mode: true
```

---

### 3.3а Embedding Profiles (new subsection)

**Purpose:** allow administrator to switch embedding model depending on available server resources, without code changes.

#### Problem

`multilingual-e5-large` (~560 MB, 2.5 GB RAM) — optimal choice for quality. But on weaker server or insufficient RAM lighter alternative needed without config rewriting.

#### Profiles

| Profile | Model | RAM | Latency (CPU) | Quality | When to use |
|---------|-------|-----|--------------|---------|------------|
| `fast` | `multilingual-e5-small` | ~0.5 GB | ~50ms | reduced | weak server, PoC |
| `balanced` | `multilingual-e5-base` | ~1.2 GB | ~120ms | medium | 16 GB RAM with load |
| `quality` | `multilingual-e5-large` | ~2.5 GB | ~300ms | high | **recommended default** |

> **Important:** only one model active at a time. Switching profile requires service restart and full reindexing — vectors from different models incompatible.

#### Configuration

```yaml
embedding:
  profile: quality          # fast | balanced | quality
  profiles:
    fast:
      model_name: intfloat/multilingual-e5-small
      model_path: ./models/multilingual-e5-small
      batch_size: 32
    balanced:
      model_name: intfloat/multilingual-e5-base
      model_path: ./models/multilingual-e5-base
      batch_size: 24
    quality:
      model_name: intfloat/multilingual-e5-large
      model_path: ./models/multilingual-e5-large
      batch_size: 16
```

> **Profile change = new indexing.** Web Admin should warn about this when changing profile and block change without confirmation.

---

### 3.4 Indexing Module

**Purpose:** store chunks and their vectors in structures suitable for fast search.

#### Components

Module manages two parallel indexes:

##### BM25 Index (exact search)

- **Implementation:** `rank_bm25` or built-in to vector DB sparse index
- **What it indexes:** tokenized chunk text
- **For what:** finds documents containing exact query words
- **Where stored:** `./data/bm25_index/`

> BM25 is statistical algorithm. It doesn't "understand" text, but perfectly finds specific codes, abbreviations, names ("ISO-9001", "GDPR", "§ 123"), which semantic model might interpret too broadly.

##### Vector Index (semantic search)

- **Implementation:** Qdrant (local Docker container)
- **What it indexes:** 1024-dimensional vectors from E5-Large
- **For what:** finds documents similar by content, even if words don't match
- **Where stored:** `./data/qdrant/`

> Qdrant chosen for: HNSW search speed, sparse+dense support in one collection, quiet Docker startup, open source.

**Data schema (Qdrant point):**

```json
{
  "id": "sha256_chunk_id",
  "vector": [0.023, -0.441, "..."],
  "payload": {
    "text": "chunk text",
    "document_id": "sha256_doc_id",
    "source": "./documents/contract.pdf",
    "chunk_index": 3,
    "timestamp": "2025-01-15T10:23:00Z",
    "file_type": "pdf",
    "importance": 1.0,
    "boost_score": 0.0
  }
}
```

#### Module operations

| Operation | Trigger | Description |
|-----------|---------|-------------|
| `index_document` | new/changed file | adds chunks to both indexes |
| `delete_document` | deleted file | removes all document chunks by `document_id` |
| `reindex_all` | admin command | full repository reindexing |
| `partial_reindex` | auto on change | updates only changed chunks (§3.4а) |
| `check_integrity` | diagnostics | verifies index consistency (§3.4б) |

#### Configuration

```yaml
indexing:
  qdrant:
    host: localhost
    port: 6333
    collection_name: documents
  bm25:
    index_path: ./data/bm25_index
    language: multilingual
```

---

### 3.4а Partial Reindex (new subsection)

**Purpose:** when document changes, update only chunks that actually changed — instead of full document reindexing.

#### Problem

Current SHA-256 mechanism (§3.1, Step 4) detects change at file level. If one page of 100-page PDF changed — all its chunks recomputed. On CPU this is expensive.

#### Solution: chunk-level deduplication

Each chunk has own `id = SHA-256(chunk.text)`. When reindexing document:

```
[New document]
       │
       ▼
[Chunking → new chunk_id]
       │
       ├─ chunk_id already in index? → skip (don't recompute embedding)
       │
       └─ chunk_id new or changed? → embed + upsert
       
[Old chunk_id that no longer exists] → delete from both indexes
```

#### State schema

```python
# ./data/chunk_registry.json
{
  "document_id_abc123": {
    "chunk_ids": ["chunk_001", "chunk_002", "chunk_003"],
    "indexed_at": "2025-01-15T10:23:00Z"
  }
}
```

At partial reindex:
1. Compare new `chunk_ids` with `chunk_registry`
2. New or changed → embed + upsert to Qdrant and BM25
3. Missing → delete from Qdrant and BM25
4. Update `chunk_registry`

#### Effect

| Scenario | Without partial reindex | With partial reindex |
|----------|------------------------|----------------------|
| Change 1 of 50 pages | ~50 embeddings | ~2–5 embeddings |
| Metadata update without text | ~50 embeddings | 0 embeddings |
| New document | N embeddings | N embeddings (no change) |

#### Configuration

```yaml
indexing:
  partial_reindex:
    enabled: true
    chunk_registry_path: ./data/chunk_registry.json
```

---

### 3.4б Index Consistency (new subsection)

**Purpose:** guarantee BM25 and Vector Index always in consistent state.

#### Problem

BM25 and Qdrant — two separate stores. If indexing fails (crash, timeout) they can diverge: chunk in Qdrant but missing in BM25, or vice versa. Results in incomplete or incorrect search results without obvious errors.

#### Solution: Versioned Atomic Update

**Index versioning:**

```python
# ./data/index_manifest.json
{
  "version": 47,
  "bm25_version": 47,
  "vector_version": 47,
  "last_updated": "2025-01-15T10:23:00Z",
  "status": "consistent"   # consistent | updating | inconsistent
}
```

**Staged update pipeline:**

```
[New chunks ready for write]
       │
       ▼
[1. Staging: temporary write to bm25_index_staging/ and qdrant_collection_staging]
       │
       ▼
[2. Validation: verify both staging indexes contain same chunk_ids count]
       │
       ├─ Validation failed → rollback staging, log error, index unchanged
       │
       └─ Validation passed →
              │
              ▼
       [3. Atomic swap: staging → production]
              │
              ▼
       [4. Update index_manifest.json: version++, status="consistent"]
```

> **Real atomic swap at file level** implemented via rename (OS atomic operation). For Qdrant — via alias switching (Qdrant supports collection aliases).

**Integrity check (check_integrity):**

```python
def check_integrity() -> dict:
    bm25_ids  = set(bm25_index.all_chunk_ids())
    qdrant_ids = set(qdrant.scroll_all_ids())
    
    only_bm25   = bm25_ids - qdrant_ids    # in BM25, not in Vector
    only_qdrant = qdrant_ids - bm25_ids    # in Vector, not in BM25
    
    return {
        "consistent": len(only_bm25) == 0 and len(only_qdrant) == 0,
        "only_in_bm25": list(only_bm25),
        "only_in_vector": list(only_qdrant),
        "total_bm25": len(bm25_ids),
        "total_vector": len(qdrant_ids),
    }
```

`check_integrity` runs:
- automatically after each `index_document` or `partial_reindex`
- manually via Web Admin
- at service start (if `status != "consistent"` in manifest)

#### Configuration

```yaml
indexing:
  consistency:
    enabled: true
    manifest_path: ./data/index_manifest.json
    auto_check_after_index: true
    staging_dir: ./data/staging
```

---

### 3.4в Metadata Extraction (new subsection)

**Purpose:** during ingestion automatically identify and extract structured identifiers from document text to separate `metadata_tags` field — enable deterministic search without scoring.

#### Deterministic WHERE filter

```python
# Not: score("ISO-9001") >= threshold
# But: WHERE metadata_tags CONTAINS "ISO-9001"  → 100% recall
```

#### What is extracted

| Type | Examples | Regex/method |
|------|----------|--------------|
| Standards and norms | `ISO-9001`, `EN-ISO 13849` | `[A-Z]{2,}[\s\-]?\d+` |
| Paragraphs | `§ 47`, `Art. 15` | `§\s*\d+`, `Art\.\s*\d+` |
| Contract numbers | `CON-2025-441` | `[A-Z]{2,4}-\d{4}-\d+` |
| INCOTERMS | `DAP`, `DDP`, `FOB` | dictionary of 11 codes |
| Dates | `2025-09-30`, `30.09.2025` | ISO + European format |

#### Updated Qdrant point schema

```json
{
  "payload": {
    "text": "chunk text",
    "metadata_tags": ["ISO-9001", "DAP", "§ 47"],
    "table_meta": null
  }
}
```

#### Phased plan

| Phase | Implementation | Code recall |
|-------|-----------------|------------|
| Phase 1 (MVP) | Query Routing §3.5г — BM25=0.85 | ~85–90% |
| Phase 2 (Production) | Metadata Extraction + WHERE filter | ~100% |

> **MVP lesson:** in Phase 2 `Article` Query Routing mode replaced by WHERE filter — 100% recall.

#### Configuration

```yaml
indexing:
  metadata_extraction:
    enabled: true
    extract_norms: true
    extract_paragraphs: true
    extract_incoterms: true
    extract_dates: true
    extract_doc_numbers: true
    custom_patterns: []
```

---

### 3.5 Search Module

**Purpose:** take text query and return list of relevant chunks.

#### Input

```python
SearchQuery:
  text: str              # user query
  filters: dict          # optional: file_type, date_range, source
  top_k: int             # results to return (default: 10)
  mode: str              # "fast" | "precise" (enables reranking)
```

#### Output

```python
List[SearchResult]:
  chunk_id: str
  text: str
  score: float           # final score after merge
  bm25_score: float      # raw BM25 score (for explainability)
  vector_score: float    # raw vector score (for explainability)
  source: str            # file path
  metadata: dict
  boost_applied: float   # metadata boost applied (§3.5б)
```

#### Search pipeline

```
Query
  │
  ▼
[0. Cache Lookup (§4.7)]
  │  cache hit → return cached result
  │  cache miss → continue
  ▼
[1. Query Normalization & Enrichment]
  │  - lowercase, strip
  │  - Query Enrichment: synonyms, short query expansion
  │  - Query Intent Detection (§3.5а): determine query type
  ▼
[2. BM25 Search]
  │  - top_k * 3 candidates (reserve for merge)
  │  - exact token match
  ▼
[3. Vector Search]
  │  - query also vectorized via E5-Large
  │  - top_k * 3 candidates from Qdrant
  │  - HNSW approximate nearest neighbor
  ▼
[4. Merge (Hybrid Fusion — Query Routing §3.5г)]
  │  - Query Routing: dynamic weights by query type
  │  - Article (code detected): BM25=0.85, Semantic=0.15
  │  - Short (≤3 words): BM25=0.45, Semantic=0.55
  │  - Long (>3 words): BM25=0.10, Semantic=0.90
  │  - result: single sorted list
  ▼
[5. Metadata Boost Application (§3.5б)]
  │  - score adjustment by date, source, importance
  ▼
[6. Feedback Boost Application (§3.11)]
  │  - raise/lower score by accumulated signals
  ▼
[7. Filter Application]
  │  - if filters passed — filter by file_type, date_range, source
  ▼
[Output: top_k candidates with full score data]
```

> **Note on RRF:** RRF (Reciprocal Rank Fusion) was initial baseline algorithm (v3.0.1). MVP testing (June 2026) revealed weakness in cross-lingual queries — static weights don't allow adaptation to query type. Replaced by Query Routing (§3.5г) with dynamic weights.

**Query Enrichment:**

Short queries (1–2 words) give weak embedding vectors — model lacks sufficient context. Query Enrichment adds context:

| Enrichment type | Example | When applied |
|-----------------|---------|-------------|
| Synonym expansion | "zmluva" → "zmluva OR kontrakt OR dohoda" | BM25 channel |
| Short query expansion | "GDPR" → "GDPR regulation data protection" | vector channel |
| Keyword extraction | long natural query → key terms | BM25 channel |

> **Enrichment applied selectively:** for BM25 and vector channels independently. Original query always preserved and used for logging.

#### UX logic: two-phase response

For speed feeling frontend gets response in two stages:

1. **Phase 1 (< 100ms):** BM25 results — instant, no vectorization
2. **Phase 2 (< 1s):** full results after merge — updates list

Creates impression of instant result even under heavier load.

#### Configuration

```yaml
search:
  top_k: 10
  bm25_candidates_multiplier: 3
  vector_candidates_multiplier: 3
  rrf_k: 60
  min_score_threshold: 0.1
  default_mode: fast           # fast | precise
  query_enrichment:
    enabled: true
    synonym_expansion: true
    short_query_threshold: 3   # words — if less, expansion applied
```

---

### 3.5а Query Intent Adaptation (new subsection)

**Purpose:** automatically adapt BM25/vector balance depending on query character — without manual adjustment by user.

#### Problem

Fixed weights for BM25 and vector not optimal for all query types:
- Short exact query ("ISO 9001", "§ 47", "GDPR") → BM25 more precise
- Long semantic query ("how is leave pay calculated for partial month") → vector more precise

#### Adaptation logic

```
Query analyzed by three features:
  1. Length (word count)
  2. Presence of specific patterns (codes, articles, §, numbers)
  3. Query language vs majority document language (detected auto)

↓

[Intent Score] = weighted(length, patterns, language_match)

↓

If Intent = "keyword":
  bm25_weight = 0.7, vector_weight = 0.3

If Intent = "semantic":
  bm25_weight = 0.3, vector_weight = 0.7

If Intent = "mixed" (default):
  bm25_weight = 0.5, vector_weight = 0.5
```

**Pattern detector:**

```python
KEYWORD_PATTERNS = [
    r'\b§\s?\d+\b',          # paragraphs: § 47
    r'\b[A-Z]{2,}-\d+\b',    # codes: ISO-9001, EN-1234
    r'\b\d{4,}\b',            # long numbers: articles, codes
    r'\b[A-Z]{3,}\b',         # abbreviations: GDPR, BOZP, DPH
]
```

> **This is not classification model.** Simple regex heuristics and word counting. Cheap, fast, predictable. No additional ML.

**Weighted Fusion (instead of pure RRF in adaptive mode):**

```python
final_score = bm25_weight * normalize(bm25_score) + vector_weight * normalize(vector_score)
```

Normalization done min-max over current candidate sample (not globally).

#### Configuration

```yaml
search:
  intent_adaptation:
    enabled: true
    mode: auto                 # auto | fixed
    fixed_bm25_weight: 0.5    # used if mode=fixed
    keyword_threshold: 3      # words — if less, lean to keyword intent
    pattern_detection: true
```

---

### 3.5б Metadata Boosting (new subsection)

**Purpose:** adjust result relevance based on business document attributes — date, source, importance.

#### Problem

Search algorithm doesn't know business context: outdated document might score higher than current one. Old regulation superseded by new, but search doesn't know.

#### Boost parameters

| Parameter | Type | Effect | Source |
|-----------|------|--------|--------|
| `document_date` | float, 0.0–1.0 | fresher documents get boost | auto from metadata |
| `source_priority` | float, 0.0–2.0 | some sources more priority | `settings.yaml` |
| `importance` | float, 0.0–2.0 | manual importance mark | administrator via Web Admin |

**Formula:**

```python
boost = (date_weight * date_score) + (source_weight * source_priority) + (importance_weight * importance)

final_score = base_score * (1 + boost_factor * boost)
```

**Date decay:**

```python
import math
from datetime import datetime, timedelta

def date_score(document_date: datetime, half_life_days: int = 365) -> float:
    age_days = (datetime.now() - document_date).days
    return math.exp(-0.693 * age_days / half_life_days)
    # exp(-ln2 * age/half_life) → 1.0 for new, 0.5 after half_life, ~0.0 for very old
```

> **Why exponential decay, not linear?** Linear decay gives negative values for very old documents. Exponential asymptotically approaches 0 — old documents don't disappear, just less priority.

**Source priority:**

```yaml
# settings.yaml
search:
  metadata_boost:
    source_priorities:
      "documents/regulations/": 1.8    # regulatory base — high priority
      "documents/contracts/": 1.5
      "documents/archive/": 0.5        # archive — reduced priority
```

#### Configuration

```yaml
search:
  metadata_boost:
    enabled: true
    date_boost:
      enabled: true
      weight: 0.3
      half_life_days: 365
    source_boost:
      enabled: true
      weight: 0.4
    importance_boost:
      enabled: true
      weight: 0.3
    boost_factor: 0.2      # max boost impact on final score (20%)
```

> **boost_factor = 0.2** means: boost can raise or lower score maximum 20%. Prevents irrelevant but "important" document replacing truly relevant one.

---

### 3.5в Score Explainability (new subsection)

**Purpose:** make visible why specific result got its score — for debugging and system trust.

#### Response structure

```python
SearchResult:
  chunk_id: str
  text: str
  source: str
  metadata: dict

  # Score breakdown (explainability)
  scores: {
    "bm25_raw":       0.43,   # normalized BM25 score
    "vector_raw":     0.71,   # cosine similarity from Qdrant
    "rrf_merged":     0.68,   # after RRF or weighted fusion
    "metadata_boost": 0.05,   # +5% from metadata boosting
    "feedback_boost": -0.03,  # -3% from feedback signals
    "final":          0.70    # final score determining order
  }
```

> **Not for end user.** Score breakdown hidden in UI by default. Available via:
> - Web Admin (always visible)
> - Search API response (always returned)
> - Frontend "debug mode" (enabled via URL parameter `?debug=1`)

#### Configuration

```yaml
search:
  explainability:
    enabled: true             # false = return only final score
    include_in_api: true      # include score breakdown in API response
```

---

### 3.5г Query Routing Algorithm (new subsection)

**Purpose:** dynamically assign BM25 and semantic channel weights by detected query type — instead of static mixing (RRF).

#### Why RRF insufficient

RRF (Reciprocal Rank Fusion) assumes every query has same requirements for BM25 and vector search. MVP testing (June 2026) revealed failure:

> Query `delivery term` (semantic) → BM25 finds English documents with word *delivery* → result: irrelevant English document with high BM25 score displaces relevant Slovak document.

Reason: static BM25 weight too high for semantic cross-lingual queries.

#### Three Query Routing modes

| Mode | Detection condition | BM25 weight | Semantic weight | When |
|------|-------------------|-----------|-----------------|------|
| **Article** | Regex: `\b[A-Z]{2,}[\s\-]?\d+\b` detects code | **0.85** | 0.15 | `ISO-9001`, `DAP`, `§ 47` |
| **Short** | Word count ≤ 3 (and not Article) | 0.45 | **0.55** | `delivery term`, `GDPR` |
| **Long** | Word count > 3 | 0.10 | **0.90** | `how is compensation calculated for complaint` |

#### Implementation

```python
import re

ARTICLE_PATTERN = re.compile(r'\b[A-Z]{2,}[\s\-]?\d+\b')

def detect_query_route(query: str) -> tuple[str, float, float]:
    """
    Returns: (route_name, bm25_weight, semantic_weight)
    """
    if ARTICLE_PATTERN.search(query):
        return ("Article", 0.85, 0.15)
    
    word_count = len(query.strip().split())
    if word_count <= 3:
        return ("Short", 0.45, 0.55)
    else:
        return ("Long", 0.10, 0.90)

def hybrid_score(bm25_norm: float, sem_norm: float, query: str) -> float:
    route, bm25_w, sem_w = detect_query_route(query)
    return bm25_w * bm25_norm + sem_w * sem_norm
```

#### Diagnostic transparency

Each search result includes detected mode label — for debugging and audit:

```
Score: 0.657
[Long Route | BM25=0.10  Lex=0.00, Sem=0.73]
```

#### Evolution in production

| Phase | Article mode | Short/Long mode |
|-------|--------------|-----------------|
| Phase 1 (MVP) | BM25=0.85 — compensates for missing Metadata Extraction | Remains |
| Phase 2 (Production) | **Replaced** by Metadata WHERE filter (§3.4в) — 100% recall | **Remains** as production function |

> **Conclusion:** Query Routing not temporary MVP solution. Short/Long modes are valid production architecture standard used in Elasticsearch, Vespa, Qdrant. Article mode transitional, will be replaced by §3.4в.

#### Configuration

```yaml
search:
  query_routing:
    enabled: true
    article_pattern: '\b[A-Z]{2,}[\s\-]?\d+\b'
    short_query_threshold: 3
    weights:
      article: { bm25: 0.85, semantic: 0.15 }
      short:   { bm25: 0.45, semantic: 0.55 }
      long:    { bm25: 0.10, semantic: 0.90 }
    debug_labels: true
```

---

### 3.6 Reranking Module

**Purpose:** improve result order for `precise` search mode.

#### Input

- Candidate list from Search Module (10–20 items)
- Original query

#### Output

- Same list, but sorted by new `rerank_score`

#### Logic

**Model:** lightweight cross-encoder (BAAI/bge-reranker-v2-m3 or similar)

Unlike bi-encoder (encoding query and document separately), cross-encoder takes pair `(query, chunk)` together and computes real semantic relevance.

```
Query + Chunk_1 → Cross-Encoder → Score: 0.92
Query + Chunk_2 → Cross-Encoder → Score: 0.71
Query + Chunk_3 → Cross-Encoder → Score: 0.88
```

> **Why not always?** Cross-encoder much more accurate, but slower (O(n) model calls). Running on 1000 candidates takes seconds. So: first fast search → then rerank only final 10–20 candidates.

**Score blending:**

```python
final_score = alpha * rerank_score + (1 - alpha) * rrf_score
# alpha = 0.7 (reranker more accurate, but trust RRF too)
```

#### Configuration

```yaml
reranking:
  enabled: false              # disabled by default (fast mode)
  model_path: ./models/bge-reranker-v2-m3
  max_candidates: 20
  score_alpha: 0.7
  device: cpu
```

---

### 3.7 API Module

**Purpose:** HTTP interface between frontend/external clients and internal modules.

#### Framework: FastAPI (async)

> **Why FastAPI?** Fully async — server doesn't block while embedding model processes request. Auto OpenAPI docs generation. Built-in validation via Pydantic.

#### Endpoints

**Search:**

```
POST /api/v1/search
Body: { "query": str, "filters": dict, "top_k": int, "mode": str }
Returns: { "results": List[SearchResult], "took_ms": int }

GET  /api/v1/search?q=...&top_k=10&mode=fast
```

**Indexing:**

```
POST /api/v1/index
Body: { "path": str }
Returns: { "job_id": str, "status": "queued" }

GET  /api/v1/index/status/{job_id}
Returns: { "status": str, "progress": float, "indexed": int, "errors": list }
```

**Status:**

```
GET  /api/v1/health
Returns: { "status": "ok|degraded|down", "components": dict, "uptime_s": int,
           "translation_enabled": bool }

GET  /api/v1/stats
Returns: { "total_documents": int, "total_chunks": int, "index_size_mb": float }
```

**Feedback:**

```
POST /api/v1/feedback
Body: { "query_id": str, "chunk_id": str, "signal": "click" | "skip" }
Returns: { "status": "recorded" }
```

#### Overload protection

```python
# Semaphore on number of parallel search requests
search_semaphore = asyncio.Semaphore(4)

# Queue for indexing jobs
indexing_queue = asyncio.Queue(maxsize=20)
```

> ML models on CPU consume 100% core power. Without concurrency limit 10 simultaneous requests crash server. Better queue with 1–2s wait than service refusal.

#### Authorization

- JWT Bearer tokens
- Token lifetime: 24 hours (configurable)
- `/health` and `/search` endpoints — public (configurable)
- `/index`, `/admin/*`, `/feedback` endpoints — protected

#### Logging

Structured JSON log of each request:

```json
{
  "timestamp": "2025-01-15T10:23:00Z",
  "method": "POST",
  "path": "/api/v1/search",
  "user_id": "user_abc",
  "query_hash": "sha256_of_query",
  "took_ms": 234,
  "results_count": 10,
  "cache_hit": false,
  "intent_detected": "semantic"
}
```

> Query logged as hash, not plaintext — protects from storing confidential search queries in logs.

#### Configuration

```yaml
api:
  host: 0.0.0.0
  port: 8000
  max_concurrent_searches: 4
  max_queue_size: 20
  log_path: ./data/logs/app.log

auth:
  jwt_secret_key: ${JWT_SECRET_KEY}
  jwt_expiration_hours: 24
  public_endpoints: [/api/v1/health, /api/v1/search]
```

---

### 3.8 Frontend Module

**Purpose:** web interface for search and results viewing.

#### Technology: Next.js 14 (SPA)

> **Why Next.js instead of plain HTML?** Need complex components: multi-select filters, live status updates, lazy result loading. Gradio/Streamlit don't allow this without hacks.

#### Main screens

**1. Search Screen**

- Search field with debounce (300ms)
- Two-phase result: BM25 → full merge
- Result cards: chunk text, source, score, "open file" button
- Filters: file type, date, source
- Score breakdown in `?debug=1` mode

**2. Document Browser**

- List of all indexed documents
- Indexing status (in progress / indexed / error)
- Search by filename

**3. Status Bar**

- `SystemHealthIndicator` component: queries `/api/v1/health` every 15 seconds
- Shows: API status, Qdrant status, document count

#### UX details

- **i18n:** language support — UA, EN, SK, PL, DE via `LanguageContext`
- **Design:** dark theme, glassmorphism (Tailwind CSS)
- **Accessibility:** results accessible via keyboard navigation
- **Feedback:** implicit (result click = positive signal; scroll past = neutral; fast return to search = negative signal)

#### Configuration

```yaml
# .env.local
NEXT_PUBLIC_API_URL: http://localhost:8000
NEXT_PUBLIC_DEFAULT_LANGUAGE: uk
```

---

### 3.9 Web Admin

**Status:** separate tool, not part of core search pipeline.

**Purpose:** operational system management for administrator.

#### Functions

| Function | Description |
|----------|-------------|
| Start indexing | Initiate single file or full repository reindexing |
| View queue | See which jobs queued, executing, completed with error |
| View logs | Tail of `app.log` and `startup.log` in browser |
| Component status | Qdrant, Embedding model, API — `up/down/degraded` |
| Index integrity | Compare document counts in BM25 and Vector index (§3.4б) |
| Metadata management | Set `importance` for documents (§3.5б) |
| Feedback overview | Aggregated feedback signals: top clicked, top skipped (§3.11) |
| Embedding profile | View and change active profile (§3.3а), with reindexing warning |

#### Access

- Protected by JWT with `admin` role
- Recommended: accessible only from local network (nginx firewall rule)

---

### 3.10 Translation Module (Optional Service)

**Status:** `optional` — not part of core search pipeline. Connected separately.

**Purpose:** allow user to translate specific found text fragment with one click without leaving search interface.

> **Principle of this module:** clicked → translated → forgot. Translation not stored, not auto-run, doesn't affect index. This is UI convenience, not business logic.

#### Architecture

```
[Frontend: "Translate" button on result card]
       │  click (one fragment)
       ▼
[API: POST /api/v1/translate]
       │  { text, target_lang }
       ▼
[Translation Service]
   ├─ check: service enabled?
   │     NO → return { status: "disabled" }
   │     YES ↓
   └─ HTTP POST → [LibreTranslate container :5000]
                        │
                        ▼
              { translated_text }
       │
       ▼
[Frontend: replace text in card]
```

#### Input / Output

```python
# Request
TranslateRequest:
  text: str          # chunk text (or selected fragment)
  target_lang: str   # language code: "uk", "en", "sk", "pl", "de"
  source_lang: str   # default: "auto"

# Response (success)
TranslateResponse:
  status: "ok"
  translated_text: str
  detected_source_lang: str

# Response (service disabled)
TranslateResponse:
  status: "disabled"
  message: "Translation service is not enabled"

# Response (error)
TranslateResponse:
  status: "error"
  message: str
```

#### Behavior

| Scenario | System behavior |
|----------|-----------------|
| Service enabled, LibreTranslate responds | Return translation via tuning pipeline |
| Service enabled, LibreTranslate unavailable | `status: "error"`, no crash |
| Service disabled (`enabled: false`) | `status: "disabled"`, UI doesn't show button |
| Text empty | Validation at API level, `400 Bad Request` |
| Text > 1000 chars | Truncate to 1000 with warning in response |
| Timeout (5s) on sentence | Sentence returned in original, rest translated |
| Term with glossary in text | Replaced after translation, not translated by engine |

#### What this module does NOT do

- ❌ Auto-translate all search results
- ❌ Store translations (not in index, not in DB)
- ❌ Affect search relevance
- ❌ Translate during indexing
- ❌ Use external API (only local LibreTranslate)

> **Why these limits important:** auto-translating 10 results = 10 sequential HTTP requests to LibreTranslate = 3–8 seconds latency. Lazy approach (click-only) gives < 1 second per fragment and doesn't waste CPU on what user won't read.

#### Translation Tuning Pipeline (6 steps)

LibreTranslate without preprocessing gives unstable results. Tuning fixes this without engine replacement.

**Step T1 — Cleaning:**

```python
def clean_text(text: str) -> str:
    text = re.sub(r"\s+\n", " ", text)
    text = re.sub(r"\n+", " ", text)
    text = re.sub(r"\s+", " ", text)
    return text.strip()
```

**Step T2 — Sentence Splitting:**

```python
def split_sentences(text: str) -> list[str]:
    sentences = re.split(r'(?<=[.!?])\s+', text)
    return [s.strip() for s in sentences if s.strip()]
```

**Step T3 — Language Detection:**

```python
def detect_language(text: str) -> str:
    try:
        return detect(text)
    except Exception:
        return "auto"
```

**Step T4 — Translation Call (per sentence, timeout=5s)**

**Step T5 — Glossary Application (after translation):**

```python
GLOSSARY: dict[str, str] = {
    "Qdrant": "Qdrant", "BM25": "BM25",
    "embedding": "embedding", "index": "index",
    # corporate terms added here
}
```

**Step T6 — Assembly with per-sentence fallback** (original if sentence timed out)

| Problem without tuning | Solution |
|-----------------------|----------|
| `\n` inside PDF breaks context | T1: cleaning |
| Long paragraph → unstable translation | T2: sentence split |
| `auto` on short sentence → wrong | T3: one-time detect |
| "Qdrant" → "Квадрант" | T5: glossary |
| One timeout breaks everything | T6: per-sentence fallback |

#### LibreTranslate: technical details

| Parameter | Value |
|-----------|-------|
| Implementation | Docker container `libretranslate/libretranslate` |
| Port | 5000 (internal network) |
| RAM | ~1–2 GB |
| Language pairs | `LT_LOAD_ONLY: "uk,en,sk,pl,de"` |

#### Configuration

```yaml
translation:
  enabled: false
  provider: libretranslate
  base_url: http://libretranslate:5000
  timeout_seconds: 5
  max_text_length: 1000
  default_target_lang: uk
  supported_langs: [uk, en, sk, pl, de]
  tuning:
    sentence_split: true
    detect_language: true
    apply_glossary: true
  glossary:
    Qdrant: Qdrant
    BM25: BM25
    embedding: embedding
    index: index
```

---

### 3.11 Feedback System (new module)

**Purpose:** collect user behavior signals during search and use them to gradually improve relevance — without manual labeling and without retraining models.

> **Principle:** system stops being static. Relevance grows over time through real user behavior.

#### Signal types

| Signal | Type | How collected | Weight |
|--------|------|---------------|--------|
| `click` | positive | click on result card in Frontend | +1 |
| `skip` | negative | result shown but not clicked when lower result clicked | -0.5 |
| `dwell` | positive (Phase 3 planned) | long time on page after click | — |

> **`skip` determined indirectly:** if user clicked result #3 without clicking #1 and #2 — for #1 and #2 `skip` recorded. Not always right (maybe #1 and #2 also useful), but on aggregate data provides signal.

#### Signal processing pipeline

```
[Frontend: click or scroll-past]
       │
       ▼
[API: POST /api/v1/feedback]
  Body: { query_id, chunk_id, signal }
       │
       ▼
[Feedback Collector: async write]
  ./data/feedback/YYYY-MM-DD.jsonl   ← log by lines, one signal = one line
       │
       ▼
[Feedback Aggregator: background process, hourly]
  1. Read accumulated signals
  2. Aggregate by chunk_id
  3. Filter noise (minimum N signals to apply)
  4. Compute boost_delta
  5. Update boost_score in Qdrant payload
       │
       ▼
[Search Module: apply boost when searching (§3.5, step 6)]
```

#### Data schema

**Raw log (feedback/2025-01-15.jsonl):**

```json
{"ts": "2025-01-15T10:23:00Z", "query_id": "q_abc", "query_hash": "sha256...", "chunk_id": "ch_xyz", "signal": "click", "user_id": "u_001", "rank_shown": 3}
{"ts": "2025-01-15T10:23:01Z", "query_id": "q_abc", "query_hash": "sha256...", "chunk_id": "ch_aaa", "signal": "skip", "user_id": "u_001", "rank_shown": 1}
```

**Aggregated state (feedback/aggregated.json):**

```json
{
  "ch_xyz": {
    "clicks": 47, "skips": 3, "boost_score": 0.12,
    "last_updated": "2025-01-15T11:00:00Z"
  },
  "ch_aaa": {
    "clicks": 2, "skips": 31, "boost_score": -0.08,
    "last_updated": "2025-01-15T11:00:00Z"
  }
}
```

#### boost_score formula

```python
def compute_boost(clicks: int, skips: int,
                  min_signals: int = 10,
                  max_boost: float = 0.2) -> float:
    total = clicks + skips
    if total < min_signals:
        return 0.0                     # insufficient data — don't apply
    
    click_ratio = clicks / total       # 0.0–1.0
    raw_boost = (click_ratio - 0.5) * 2   # -1.0 to +1.0
    return max(-max_boost, min(max_boost, raw_boost * max_boost))
```

> **`min_signals = 10`** — noise protection. 1–2 clicks don't change relevance. Signal applies only with sufficient statistics.
> 
> **`max_boost = 0.2`** — feedback can't raise or lower result more than 20%. Prevents "capture" of results through anomalous behavior.

#### Risks and countermeasures

| Risk | Countermeasure |
|------|----------------|
| Random clicks / noise | `min_signals` threshold |
| Position bias (top results get more clicks) | Normalization by `rank_shown` (Phase 3 planned) |
| Malicious click manipulation | Rate limiting on `/api/v1/feedback`, `user_id` filter |
| Drift (old signals affect new results) | Signal TTL: signals > 90 days not counted |

#### Configuration

```yaml
feedback:
  enabled: true
  storage_path: ./data/feedback
  aggregation:
    interval_minutes: 60          # aggregator run frequency
    min_signals: 10               # minimum signals to apply boost
    max_boost: 0.2                # max feedback impact on score
    signal_ttl_days: 90           # older signals ignored
  rate_limit:
    max_per_user_per_hour: 100    # manipulation protection
```

---

## 4. ASPECTS (System Concerns)

---

### 4.1 Async Execution

**Principle:** no slow process blocks HTTP response.

#### Execution threads

```
HTTP Request Thread (FastAPI event loop)
  │
  ├── Search: async → cache check → await semaphore → await embedding → await qdrant → return
  │
  ├── Index: async → put to queue → return job_id (immediately)
  │                      │
  │                 Background Worker
  │                      │
  │                 process job → update status
  │
  └── Feedback: async → fire-and-forget write → return "recorded"
                              │
                         Aggregator Worker (hourly)
                              │
                         update boost_scores
```

**Search Pipeline (online):**

- Fully async via `await`
- Limited by semaphore (max 4 parallel)
- Target latency: < 500ms for fast mode, < 2s for precise mode

**Indexing Pipeline (background):**

- Client gets `job_id` and `status: queued` immediately
- Real processing — in background worker
- Progress available via `/api/v1/index/status/{job_id}`
- Doesn't compete with search for resources (separate queue)

**Translation (async wrapper):**

- `translate_snippet` — sync function, but runs via `run_in_executor` to not block event loop

**Feedback (fire-and-forget):**

- Signal write doesn't block search request
- Aggregation — separate background worker, doesn't affect search latency

#### Implementation

```python
# Indexing queue worker
async def indexing_worker():
    while True:
        job = await indexing_queue.get()
        await process_indexing_job(job)
        indexing_queue.task_done()

# Search with semaphore
async def search(query: SearchQuery):
    async with search_semaphore:
        results = await _execute_search(query)
    return results

# Feedback: fire-and-forget
async def record_feedback(signal: FeedbackSignal):
    asyncio.create_task(_write_feedback(signal))
    return {"status": "recorded"}
```

---

### 4.2 Resource Management

#### RAM profile

**Base configuration (without translation, without reranker):**

| Component | RAM idle | RAM peak | Notes |
|-----------|----------|----------|-------|
| E5-Large embedding | 2.5 GB | 3.0 GB | Singleton |
| Qdrant vector DB | 2–4 GB | 5.0 GB | Depends on index size |
| BM25 index | 0.5 GB | 1.0 GB | In-memory |
| FastAPI backend | 0.3 GB | 1.0 GB | |
| Next.js frontend | 0.2 GB | 0.5 GB | |
| Query Cache | 0.1 GB | 0.3 GB | LRU, configurable (§4.7) |
| **Total (core)** | **~5.6 GB** | **~10.8 GB** | OS margin: ~5 GB on 16 GB |

**With enabled options:**

| Option | Added RAM idle | Added RAM peak |
|--------|----------------|--------------------|
| LibreTranslate | +1.0 GB | +2.0 GB |
| Reranker (bge-reranker-v2-m3) | +0.5 GB | +1.5 GB (only in precise mode) |
| **Total (with translation + reranker)** | **~7.1 GB** | **~14.3 GB** |

> On 16 GB RAM system with all options leaves ~1.7 GB margin. Minimal but workable. On 32 GB — comfortable with ability to increase Qdrant cache and batch_size.

**Compared to v2.5 (where reranker and cache not accounted):**

| Configuration | v2.5 (estimated) | v3.0 (refined) |
|--------------|-----------------|-----------------|
| Core | ~5.5 / ~10.5 GB | ~5.6 / ~10.8 GB |
| With translation | ~6.5 / ~12.5 GB | ~7.1 / ~14.3 GB |

#### Resource management policies

**Lazy Loading:**
- Embedding model loaded on first request, not at startup
- Reranker loaded only if `mode=precise`

**Batch Control:**
- `embedding.batch_size: 16` on CPU (peak RAM ~300 MB per batch)
- On OOM error — auto decrease batch_size by half

**Qdrant Memory Limits:**

```yaml
# docker-compose.yml
qdrant:
  deploy:
    resources:
      limits:
        memory: 4G
```

**Async Concurrency Control:**
- Translation: limited `run_in_executor` thread pool (default: 4 threads)
- Indexing: one background worker, queue max 20 jobs

---

### 4.3 Storage Strategy

#### Directory structure

```
./
├── documents/              # Original files (read-only for system)
├── models/                 # AI models (read-only after setup)
│   ├── multilingual-e5-large/
│   ├── multilingual-e5-base/
│   ├── multilingual-e5-small/
│   └── bge-reranker-v2-m3/
├── data/                   # Everything system generates
│   ├── file_hashes.json        # Incremental indexing state
│   ├── chunk_registry.json     # Partial reindex state (§3.4а)
│   ├── index_manifest.json     # Index consistency state (§3.4б)
│   ├── bm25_index/             # BM25 index
│   ├── staging/                # Staging area for atomic index swap
│   ├── qdrant/                 # Vector DB storage
│   ├── jobs/                   # Indexing job state (JSON files)
│   ├── feedback/               # Feedback signals (§3.11)
│   │   ├── 2025-01-15.jsonl
│   │   └── aggregated.json
│   └── logs/                   # app.log, startup.log
└── config/
    └── settings.yaml
```

#### Principles

**Idempotency:** re-running document indexing gives same result. No duplicates created.

**Reproducibility:** deleting `./data/` and reindexing rebuilds exact same state. Exception: `feedback/` — behavior signals not auto-recovered.

**Portability:** to migrate system to another server just copy entire directory. No external dependencies.

**Controlled data boundary:** all system data in three local zones: `./documents/` (sources), `./models/` (local models), `./data/` (indexes, job state, feedback, logs). Gives IT simple control model: backup, retention, encryption at rest, access control applied to defined directories, not scattered external services.

**Deletion and reindex:** when document deleted from `documents_dir`, `delete_document` or next partial/full reindex must remove related chunks from BM25 and Qdrant. For GDPR/NDA scenarios critical operational control: source deletion has reproducible derived index cleanup procedure.

**Retention policy:** `feedback/`, `jobs/`, `logs/` shouldn't be stored indefinitely. Default production policy: operational logs — 30 days, feedback signals — 90 days, job state — 30 days after completion. Specific terms approved by IT/security environment owner.

#### Backup strategy

```bash
# Minimal backup (index state + feedback)
cp -r ./data/qdrant     ./backup/qdrant_$(date +%Y%m%d)
cp ./data/file_hashes.json ./backup/
cp -r ./data/feedback   ./backup/feedback_$(date +%Y%m%d)

# Full backup (including models)
tar -czf archivarius_full_$(date +%Y%m%d).tar.gz ./data ./models ./config
```

Production backup stored encrypted or on encrypted volume. Contains indexes, embeddings, feedback — from NDA perspective equivalent to corporate documents, even without plaintext documents in backup.

---

### 4.4 Deployment

#### Containerization (Container-First)

```yaml
# docker-compose.yml (simplified schema)
services:
  backend:
    build: ./backend
    ports: ["8000:8000"]
    volumes: ["./data:/app/data", "./models:/app/models", "./documents:/app/documents"]
    
  qdrant:
    image: qdrant/qdrant:<pinned-version>
    expose: ["6333"]
    volumes: ["./data/qdrant:/qdrant/storage"]
    deploy:
      resources:
        limits:
          memory: 4G
    
  frontend:
    build: ./frontend
    ports: ["3000:3000"]
    
  admin:
    build: ./admin
    ports: ["8080:8080"]
    
  # libretranslate: conditional service, connected if translation.enabled: true
  libretranslate:
    image: libretranslate/libretranslate:latest
    environment:
      LT_LOAD_ONLY: "uk,en,sk,pl,de"
      LT_DISABLE_WEB_UI: "true"
    networks:
      - internal
    profiles: ["translation"]   # docker compose --profile translation up
```

#### Production hardening

Production deployment consists of base compose schema plus mandatory hardening profile. Hardening profile is production requirement, not optional improvement:

- Docker images pinned to specific versions or digest; `latest` forbidden for production.
- Qdrant and LibreTranslate available only in Docker internal network; only reverse proxy exposed externally.
- Frontend/API published via nginx or corporate reverse proxy with TLS termination.
- Admin endpoints accessible only after JWT auth and preferably limited to intranet/VPN allowlist.
- `SECRET_KEY`, JWT secrets, service credentials passed via env/secret store, not git-tracked config.
- Models loaded during provisioning, after which runtime operates with `HF_HUB_OFFLINE=1`.
- For air-gap environments provisioning via pre-approved offline artifact bundle: Docker images, model files, SBOM, license inventory, checksums.

#### Deployment environments

| Environment | OS | Start |
|------------|-----|--------|
| Production | Linux (Ubuntu 22.04 LTS) | `docker compose up -d` |
| Development | macOS / Linux | `docker compose up` |
| Windows (office) | Windows 10/11 + WSL2 | `start_bridge.bat` or `ArchivariusLauncher.vbs` |

#### Provisioning (deployment on new server)

```bash
# Step 1: prerequisites (once)
# Install Docker Desktop (Windows/Mac) or Docker Engine (Linux)

# Step 2: clone
git clone <repo> archivarius
cd archivarius

# Step 3: download models (once, requires internet)
python scripts/setup_models.py

# Step 4: run
docker compose up -d

# Step 5: verify
curl http://localhost:8000/api/v1/health
```

For completely isolated environment, step 3 replaced by importing offline artifact bundle pre-approved by IT-security before transfer to company network segment.

---

### 4.5 Monitoring

#### Metrics (Prometheus-compatible)

| Metric | Type | Description |
|--------|------|-------------|
| `search_latency_ms` | Histogram | Search response time |
| `search_requests_total` | Counter | Total requests |
| `search_cache_hits_total` | Counter | Cache hit count (§4.7) |
| `search_queue_size` | Gauge | Current queue size |
| `indexing_documents_total` | Counter | Documents indexed |
| `indexing_errors_total` | Counter | Indexing errors |
| `ram_usage_bytes` | Gauge | RAM consumption |
| `qdrant_vectors_total` | Gauge | Index vector count |
| `feedback_signals_total` | Counter | Feedback signals collected |
| `index_consistency_status` | Gauge | 1=consistent, 0=inconsistent |

#### Health checks

```
GET /api/v1/health

Response:
{
  "status": "ok",
  "components": {
    "qdrant": "ok",
    "embedding": "ok",
    "bm25": "ok",
    "cache": "ok",
    "feedback": "ok",
    "translation": "disabled"
  },
  "index_consistent": true,
  "uptime_s": 3600,
  "version": "3.0.0"
}
```

#### Logging

Two log levels:

- **Operational log** (`app.log`) — JSON, each request, latency, errors, cache_hit, intent_detected
- **Startup log** (`startup.log`) — service start, model loading, initialization errors

Log rotation: `logrotate` or built-in rotating file handler (max 50 MB, 5 files).

---

### 4.6 Security

#### Minimal security model

**Auth for admin endpoints:**
- JWT Bearer token
- Roles: `user` (search, view, feedback), `admin` (index, manage)
- Secret key via env variable (never in code)

**Sandbox for files:**
- Ingestion module accepts only files from `documents_dir`
- MIME-type check (protection from path traversal)
- Max file size: 100 MB
- Forbidden extensions: `.exe`, `.sh`, `.py`, `.js`

**Network isolation (Docker):**
- Qdrant, LibreTranslate, backend — in one internal Docker network
- Qdrant and LibreTranslate not exposed (only backend talks to them)
- Frontend — only via nginx reverse proxy

#### Data privacy

- E5-Large model — local, `HF_HUB_OFFLINE=1`
- No external API
- Queries logged as hash, not plaintext
- Feedback signals logged with `user_id` (hashed), not query text

> **"Fortress" principle:** no corporate text bytes leave local network.

### 4.6а Security Assurance & Corporate Readiness

**Purpose:** define Archivarius corporate security system model: data boundaries, external service independence, supply chain requirements, production hardening, operational readiness for IT office support.

Security Assurance describes system architectural guarantees and production controls to be executed during deployment. Legal compliance with GDPR, NDA or internal company policies depends on specific deployment profile, access roles, retention policy, and operational procedures.

#### 8-card security workflow

| Card | Topic | Specification requirement |
|------|-------|--------------------------|
| 1 | Security position | System is local corporate search engine, not cloud AI service |
| 2 | Architecture & data flow | Data moves through ingestion → indexing → search inside controlled environment |
| 3 | Data boundary | Documents, embeddings, indexes, feedback, logs stored locally |
| 4 | External independence | Runtime requires no external API, SaaS, or cloud inference |
| 5 | Access control | JWT, roles, admin endpoint protection, file sandbox |
| 6 | Supply chain | Open-source components fixed via SBOM, pinned versions, checksums |
| 7 | Operations | Docker deployment, health checks, metrics, logs, backup/restore part of support model |
| 8 | Residual risk | Production controls explicitly separated from core application controls |

#### Corporate security

**Architectural requirement:** Archivarius processes corporate documents in company-controlled environment and doesn't send document text to external services during runtime.

Requirement implemented by:

- embedding model runs locally (`HF_HUB_OFFLINE=1`);
- BM25 index stored in `./data/bm25_index/`;
- Qdrant runs as local Docker service with storage in `./data/qdrant/`;
- query logs contain no plaintext queries, only hash;
- feedback contains hashed `user_id` and `query_hash`, not plaintext query;
- ingestion limited `documents_dir`, MIME check, file size limit, forbidden executable/script extensions;
- external LLM/RAG not in core system.

**NDA boundary:** plaintext documents, derived indexes, embeddings, feedback signals, logs are corporate data. Shouldn't leave company control perimeter without explicit owner permission. Production deployment must provide network isolation, access control, backup encryption, approved retention policy.

#### External factor independence

**Deployment capability:** system operates in **offline-runtime, air-gap-compatible after offline provisioning** mode.

Full air-gap achieved after Docker images, model artifacts, config preparation. After artifact setup runtime requires no:

- external AI API;
- cloud vector database;
- SaaS search backend;
- external translation API;
- remote telemetry for basic operation.

For strictly isolated environments offline artifact bundle used:

- Docker images with digest/checksum;
- model directories with checksum;
- SBOM and license inventory;
- `settings.yaml` for offline mode;
- restore/start/health-check instruction without internet access.

#### Open-source and supply chain

Open-source stack used as supply chain transparency mechanism, not standalone security guarantee. Production supply-chain profile requires:

- components available for inspection and versioning;
- dependencies fixed in SBOM;
- Docker images and Python packages pinned to specific versions or digest;
- model artifacts have checksum;
- license inventory confirms corporate usage right;
- vulnerability scan before production rollout.

Production policy: no component with `latest` or unverified remote download allowed in production without security approval.

#### GDPR/NDA control matrix

| Requirement | Architectural mechanism | Production control | Limitation / residual risk |
|------------|---------------------|--------------------|--------------------------|
| Data doesn't go out during runtime | local models, Qdrant, BM25; no external API | firewall/network egress policy, offline mode verification | provisioning may need internet without offline bundle |
| Queries not logged plaintext | query hash in logs/cache/feedback | log schema validation, periodic log review | hash may be personal data if linked to user/session |
| Feedback minimizes personal data | hashed `user_id`, `query_hash`, TTL 90 days | retention job, salt/pepper policy, access restriction | behavioral data still personal per GDPR |
| Access divided by roles | JWT Bearer, roles `user/admin` | secret rotation, token expiry, admin allowlist | MFA/SSO not described in base version |
| Documents not executed as code | MIME check, extension denylist, size limit | malware scan policy outside app or pre-ingest gate | parser vulnerabilities possible via PDF/DOCX libraries |
| Indexes local | `./data/qdrant`, `./data/bm25_index` | disk encryption, backup encryption, access control | embeddings may contain document derivative info |
| System maintained by office IT | Docker Compose, health endpoint, logs, metrics | runbook, patching cadence, restore test | without runbook support depends on developer knowledge |
| Open-source stack transparent | Qdrant, FastAPI, local model stack | SBOM, license inventory, vulnerability scan | open-source doesn't eliminate supply-chain risks |

#### Corporate readiness profile

Corporate readiness profile fixes three system properties part of production profile:

1. **Corporate security:** documents, indexes, queries remain in company infrastructure.
2. **Independence:** core search works without cloud AI, SaaS search, external API during runtime.
3. **Operational simplicity:** single-server Docker deployment, local directories, health checks, logs, metrics, backup strategy.

Production acceptance depends on hardening profile execution: pinned dependencies, TLS/reverse proxy, network isolation, SBOM/license inventory, backup encryption, retention/deletion procedures.

---

### 4.7 Query Cache (new aspect)

**Purpose:** cache search query results to reduce latency and CPU load on repeated or similar queries.

#### Mechanism

**Cache key:**

```python
cache_key = sha256(
    query.text.lower().strip() +
    str(sorted(query.filters.items())) +
    str(query.top_k) +
    query.mode
)
```

> Normalize before hashing: `"  Zmluva  "` and `"zmluva"` give same key.

**Cache type:** in-memory LRU (Least Recently Used)

```python
from functools import lru_cache
# or cachetools.LRUCache for more control

cache = LRUCache(maxsize=1000)   # max 1000 unique queries
```

**TTL (Time-To-Live):**

```python
# Each entry stored with timestamp
def is_stale(entry: CacheEntry, ttl_seconds: int = 300) -> bool:
    return (datetime.now() - entry.cached_at).seconds > ttl_seconds
```

> **TTL = 5 minutes (default).** Compromise: long enough for cache useful on burst load (multiple people searching same thing), short enough new indexing shows without delay.

#### Invalidation

| Event | Cache action |
|-------|--------------|
| New indexing completed | Full cache clear |
| `reindex_all` | Full cache clear |
| TTL triggered | Auto LRU eviction |
| Cache size hits `maxsize` | LRU evict oldest |

> **Why full clear on indexing, not point-invalidation?** Point-invalidation needs knowing which queries affected by new docs — complex logic. Full clear simple, safe, runs rarely (only on indexing).

#### Resource effect

| Parameter | Impact |
|-----------|--------|
| RAM | +0.1–0.3 GB (depends on `maxsize` and result size) |
| CPU | -30–50% on repeated queries |
| Latency | < 5ms cache hit (vs 200–800ms uncached) |

#### Configuration

```yaml
cache:
  enabled: true
  maxsize: 1000             # max unique queries in cache
  ttl_seconds: 300          # 5 minutes
  invalidate_on_reindex: true
```

---

## 5. Processing pipeline

### 5.1 Indexing Pipeline

```
                    INDEXING PIPELINE

[File System]
     │  file added/changed
     ▼
[1. Ingestion]
   ├─ type determination (MIME)
   ├─ text extraction
   ├─ SHA-256 check (already indexed?)
   ├─ normalization (UTF-8, NFC, cleanup)
   └─ Pre-index Quality Filter
     │  NormalizedDocument
     ▼
[2. Chunking]
   ├─ heuristic semantic split (main)
   ├─ paragraph-based split (fallback)
   └─ token-based split (fallback)
     │  List[Chunk]
     ▼
[3. Partial Reindex Check]
   ├─ chunk_id already in registry? → skip embedding
   └─ new chunk_id → continue
     │  List[NewChunk]
     ▼
[4. Embedding]
   ├─ batch (16–32 chunks)
   ├─ E5-Large inference (active profile)
   └─ normalize vectors
     │  List[(Chunk, Vector)]
     ▼
[5. Staging Write]
   ├─ upsert to qdrant_staging
   └─ update bm25_staging
     │
     ▼
[6. Consistency Validation]
   ├─ chunk_ids match between staging indexes?
   │    NO → rollback staging, log error
   │    YES ↓
   └─ Atomic Swap: staging → production
     │
     ▼
[7. Registry & Manifest Update]
   ├─ update chunk_registry.json
   ├─ update file_hashes.json
   └─ update index_manifest.json (version++, status=consistent)
     │
     ▼
[8. Cache Invalidation]
   └─ clear query cache
     │
     ▼
  [Done] ─→ job status: "completed"
```

### 5.2 Search Pipeline

```
                    SEARCH PIPELINE

[User Query]
     │  POST /api/v1/search
     ▼
[API Layer]
   ├─ auth check
   ├─ input validation
   └─ semaphore acquire (max 4)
     │  SearchQuery
     ▼
[0. Cache Lookup]
   ├─ cache hit → return result (< 5ms)
   └─ cache miss → continue
     │
     ▼
[Search Module]
   ├─ [Query Normalization + Enrichment]
   │    lowercase, strip, synonym expansion
   │
   ├─ [Intent Detection (§3.5а)]
   │    keyword | semantic | mixed
   │
   ├─ [BM25 Search] ─────────────────────┐
   │    tokenization + enriched query     │
   │    top_k*3 candidates               │
   │                                     │
   └─ [Vector Search] ──────────────────►[Hybrid Fusion]
        embed query (E5-Large)           │  RRF or Weighted (by intent)
        top_k*3 candidates               │
                                         │  merged candidates
                                         ▼
                                  [Metadata Boost (§3.5б)]
                                  [Feedback Boost (§3.11)]
                                         │
                                  [Filter Application]
                                         │
                               ┌─────────┴─────────┐
                          mode=fast            mode=precise
                               │                    │
                               │              [Reranking]
                               │               cross-encoder
                               │                    │
                               └─────────┬──────────┘
                                         │  top_k results + score breakdown
                                         ▼
                                  [Cache Write]
                                         │
                                  [API Response]
                                         │
                                         ▼
                                  [Frontend Display]
                                         │
                                  [Feedback Collection (async)]
```

---

## 6. What is excluded and why

| Feature | Exclusion reason |
|---------|-----------------|
| **OCR** | Instability (Tesseract depends on scan quality). Resources (300–500ms per page). Most corporate docs are text PDFs. |
| **LLM (RAG, chat)** | Uses 4–8 GB RAM (Mistral 7B). Doesn't fit on 16 GB with embedder+qdrant. Layer 2+, added after search stabilization. |
| **Auto-translate all results** | 10 sequential HTTP requests = 3–8 seconds delay. Manual per-click translation (§3.10) implemented. |
| **Real-time monitoring (Grafana)** | Setup overhead. Prometheus metrics — yes (§4.5), full Grafana stack — no for base config. |
| **Multi-user sessions** | Auth layer complexity. Base JWT sufficient for corporate intranet. |
| **Auto-train model on feedback** | Fine-tuning E5-Large needs GPU and resources. Feedback affects only boost_score (§3.11), not model. |
| **Global query cache** (Redis/Memcached) | Overkill for single instance. In-memory LRU (§4.7) sufficient without extra service. |
| **Position bias correction in feedback** | Complex normalization logic. Deferred to Phase 3 Roadmap. |

> **Exclusion rule:** feature excluded if (a) unstable, or (b) resource-heavy, or (c) not core search use case. Always can add. Hard to remove.

---

## 7. Key principles

### 1. Simplicity > features

System with 5 stable features better than system with 15 features where 7 unstable. Each new feature — potential failure point and support cost.

### 2. Search = core value

Everything subordinate to search quality and speed. If new feature worsens search latency — not accepted without explicit tradeoff.

### 3. Async everywhere

No HTTP handler blocked. Long operations (indexing, embedding, feedback aggregation) — always background. User always gets response immediately.

### 4. Minimal resources

Target: 16 GB RAM, no GPU. All decisions account for this limit. If something doesn't fit — excluded or deferred.

### 5. Modularity

Each module — independent unit with clear interface. Can replace embedder (e.g., to e5-small for weaker server), switch profile (§3.3а), disable feedback or cache — without touching rest.

### 6. Locality (Zero-Data-Leak)

No external API during runtime. All computation on company server or in controlled local Docker environment. Not option, requirement.

Correct corporate formula: **offline-runtime / air-gap-compatible after offline provisioning**. System shouldn't require cloud AI, SaaS search, remote vector database, or external telemetry for basic operation. If environment fully isolated, Docker images and model artifacts transferred as pre-approved offline bundle.

From NDA perspective, plaintext documents, embeddings, indexes, feedback, logs are corporate data with same protection in backup, access control, retention policy.

### 7. Predictability

System behaves same on repeated queries. Deterministic embeddings. Reproducible results. Feedback and cache only deviation sources, both controlled and logged.

### 8. Adaptivity (new)

System gradually improves through user behavior (§3.11) and adapts to query character (§3.5а). But adaptation layer on top stable core, not instead.

> **Final idea:** system should be fast, stable, predictable. Not "smart". Intelligence added later when base exists. Base now exists.

---

## 8. Roadmap

### Phase 1 — Core (completed)

**Goal:** stable search over documents.

- [x] Ingestion (PDF, DOCX, TXT, HTML)
- [x] Chunking with overlap
- [x] E5-Large embedding (multilingual)
- [x] Qdrant vector index
- [x] BM25 index
- [x] Hybrid search + RRF merge
- [x] FastAPI + JWT
- [x] Next.js frontend (search + results)
- [x] Docker Compose deployment
- [x] Incremental indexing (SHA-256)

### Phase 2 — Quality & Ops (current)

**Goal:** search quality improvement, operational maturity, adaptivity.

**Tier 1 (mandatory — high priority):**

- [ ] **Query Cache** (§4.7) — in-memory LRU, TTL=5min, invalidation on reindex
- [ ] **Pre-index Quality Filter** (§3.1) — noise rejection before embedding
- [ ] **Partial Reindex** (§3.4а) — chunk-level deduplication, chunk_registry
- [ ] **Index Consistency** (§3.4б) — versioned atomic swap, check_integrity
- [ ] **Feedback System** (§3.11) — click/skip signals, boost_score aggregation

**Tier 2 (important — medium priority):**

- [ ] **Embedding Profiles** (§3.3а) — fast / balanced / quality switching
- [ ] **Query Intent Adaptation** (§3.5а) — keyword vs semantic detection, adaptive weights
- [ ] **Metadata Boosting** (§3.5б) — date decay, source priority, importance
- [ ] **Score Explainability** (§3.5в) — full score breakdown in API and Admin
- [ ] Reranking (precise mode) — cross-encoder `bge-reranker-v2-m3`
- [ ] Web Admin (indexing queue, logs, status, feedback overview, metadata management)
- [ ] Prometheus metrics endpoint
- [ ] **Security Assurance Pack** (§4.6а) — SBOM, license inventory, production hardening checklist, offline artifact bundle procedure

**Tier 3 (enhancement — lower priority):**

- [ ] **Heuristic Semantic Split** for Chunking (§3.2) — similarity-based split
- [ ] **Query Enrichment** (§3.5) — synonym expansion, short-query expansion
- [ ] Full health check with components
- [ ] Document-level search filters (type, date)
- [ ] Windows Launcher (VBS + tray icon)
- [ ] `ArchivariusInstaller.iss` (Inno Setup)
- [ ] **Translation Module** (LibreTranslate, lazy, click-based) — `enabled: false` by default

### Phase 3 — Extended Features

**Goal:** additional capabilities on top stable core.

- [ ] OCR (optional module, if GPU or strong CPU available)
- [ ] RAG / Chat (Ollama + Mistral/Llama3) — separate service, not in core
- [ ] Multi-user roles (admin / editor / viewer)
- [ ] Document versioning (tracking changes between versions)
- [ ] Export results (CSV, PDF report)
- [ ] Position bias correction in Feedback System (normalization by `rank_shown`)
- [ ] Dwell time signal for Feedback (time on page after click)

### Phase 4 — Scale & Intelligence

**Goal:** scaling and AI expansion.

- [ ] Multi-node deployment (multiple servers)
- [ ] GPU acceleration (CUDA for embedding and reranking)
- [ ] Fine-tuned embedding model on corporate data
- [ ] Entity extraction (auto-identifying persons, dates, companies)
- [ ] Taxonomy / tagging (auto document classification)

---

*Document is living — updated when architectural decisions are made.*  
*Changes submitted via Pull Request with discussion.*  
*Version 3.0.3 — added Security  & Corporate Readiness, production hardening, and security review matrix.*  

