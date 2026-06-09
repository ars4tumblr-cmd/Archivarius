# Archivarius — Corporate Document Search

> **Find anything in your company's documents. In any language. In 2–3 seconds. Fully offline.**

[![Live Demo](https://img.shields.io/badge/Live%20Demo-HuggingFace-yellow)](https://huggingface.co/spaces/Ars4ars/Archivarius-Demo-V2)
[![Spec](https://img.shields.io/badge/Spec-v3.0.3-blue)](#documentation)
[![Stack](https://img.shields.io/badge/Stack-FastAPI%20%7C%20Qdrant%20%7C%20Next.js-informational)](#tech-stack)

---

## What Is Archivarius?

Archivarius is an **on-premise hybrid search engine** for corporate document archives. It combines semantic vector search with keyword search (BM25) to let employees find relevant information across multilingual document collections — contracts, specifications, correspondence, decisions — without knowing the exact filename, folder, or language.

**Key property:** the system runs entirely within your infrastructure. No data leaves your network at runtime. Ever.

---

## The Problem It Solves

Every company accumulates documents. Finding the right one is painful:

- The relevant clause *exists somewhere* — but in which file and how was it worded?
- The document is in German or Slovak — you have to search in that language or use Google Translate **with your confidential data**
- A new employee spends weeks asking colleagues instead of working
- A problem was solved two years ago — but where is that experience?

**The real risk:** when there's no convenient internal tool, people use Google Translate, ChatGPT, and Copilot — sending confidential corporate documents to external servers. ([Samsung, 2023](https://www.bbc.com/news/technology-65514920))

---

## How It Works

```
Employee query (any language)
        ↓
  [Hybrid Search Engine]
   BM25 (keyword) + Vector (semantic)
        ↓
  Ranked results with source snippets
        ↓
  One-click local translation (LibreTranslate)
        ↓
  Original document at exact location
```

**Cross-language search without translation:** a query in English finds matching passages in German, Slovak, Ukrainian, or any other supported language — because the model understands *meaning*, not just words.

---

## Security Model

| Property | Implementation |
|----------|---------------|
| **Zero outbound connections at runtime** | No external API calls after initial setup |
| **All computation local** | AI model runs on company server |
| **Data never leaves the perimeter** | Documents, indices, logs — all local |
| **Query logs** | SHA-256 hashed, not plaintext |
| **Access control** | JWT Bearer tokens, user/admin roles |
| **File validation** | MIME type check + extension denylist before indexing |
| **GDPR** | Compliant by architecture — no additional configuration needed |

> This is not a security policy. It is a technical impossibility for data to leave the network during normal operation.

---

## Tech Stack

All components are **open source with no licensing costs**:

| Component | Technology | Licence |
|-----------|-----------|---------|
| Semantic search | [multilingual-e5-large](https://huggingface.co/intfloat/multilingual-e5-large) (Microsoft Research) | MIT |
| Vector database | [Qdrant](https://qdrant.tech/) | Apache 2.0 |
| Keyword search | [rank_bm25](https://github.com/dorianbrown/rank_bm25) | Apache 2.0 |
| API server | [FastAPI](https://fastapi.tiangolo.com/) + Python 3.11 | MIT |
| Frontend | [Next.js 14](https://nextjs.org/) | MIT |
| Local translation | [LibreTranslate](https://libretranslate.com/) | EUPL |
| Deployment | Docker Compose | Apache 2.0 |

**No GPU required.** Minimum: 16 GB RAM, modern CPU.

### Model Independence

The architecture is modular — the embedding model is swappable via configuration change only (no code modification required). Three built-in profiles:

| Profile | Model | RAM | Use case |
|---------|-------|-----|----------|
| `fast` | multilingual-e5-small | ~0.5 GB | Low-resource server, PoC |
| `balanced` | multilingual-e5-base | ~1.2 GB | 16 GB RAM under load |
| `quality` | multilingual-e5-large | ~2.5 GB | **Recommended default** |

---

## Language Coverage

Built on `multilingual-e5-large` — supports **100+ languages** with no additional configuration.

| Region | Languages |
|--------|----------|
| Central & Eastern Europe | UA, SK, PL, CZ, HU, RO, HR, SI |
| Western Europe / DACH | DE, AT, EN, FR, IT, ES, NL |
| Global | ZH, JA, KO, AR, HI, and 90+ more |

---

## Deployment

```bash
# Step 1 — Clone and configure
git clone https://github.com/ars4tumblr-cmd/Archivarius.git
cd Archivarius
cp .env.example .env   # edit secrets

# Step 2 — Download AI model (~560 MB, one-time)
python scripts/download_model.py

# Step 3 — Start
docker compose up -d

# Step 4 — Verify
curl http://localhost:8000/api/v1/health
# → {"status": "ok", ...}
```

After first-time setup, the system runs **fully offline**. No internet connection required for operation.

### Server Requirements

| Parameter | Minimum | Recommended |
|-----------|---------|------------|
| RAM | 16 GB | 32 GB |
| CPU | Any modern | 4+ cores |
| Storage | 10 GB + document volume | SSD |
| GPU | **Not required** | Optional (speeds up indexing) |
| OS | Windows 10/11 + WSL2 / Ubuntu 22.04 | Linux |

---

## Current Status

| What exists | Status |
|------------|--------|
| Technical specification v3.0.3 | ✅ Complete (see `/docs`) |
| Live Demo | ✅ [Available](https://huggingface.co/spaces/Ars4ars/Archivarius-Demo-V2) |
| Architecture & security design | ✅ Complete |
| Phase 1 prototype | 🚧 In development |

> The live demo is a simplified proof-of-concept. The full prototype is under development per spec v3.0.3.

---

## Documentation

| Document | Description |
|----------|-------------|
| [Architecture Overview](docs/architecture.md) | System design, data flow, component interaction |
| [Security Model](docs/security.md) | Full security specification, GDPR compliance |
| [Technical Specification v3.0.3](docs/spec_v303.md) | Complete system specification |

---

## Scalability

| Scenario | Approach |
|----------|----------|
| Single branch | 1 server, Docker Compose |
| Multiple branches | Independent instance per branch, own document base |
| Centralised | One server + VPN access |
| Corporate network | Nginx reverse proxy + TLS + AD/LDAP (Phase 3 roadmap) |

---

## Roadmap

- **Phase 1 (current):** Core hybrid search — ingestion, chunking, BM25 + vector search, FastAPI, Next.js frontend
- **Phase 2:** Query cache, feedback loop, admin panel, security hardening
- **Phase 3:** RAG/Chat (Ollama + Mistral/Llama3), OCR for scanned documents, AD/LDAP integration, multi-node deployment

---

## Contact

**Idea & Business Case:** Serhii Shtokal — via [GitHub](https://github.com/ars4tumblr-cmd)  
**Demo:** https://huggingface.co/spaces/Ars4ars/Archivarius-Demo-V2
