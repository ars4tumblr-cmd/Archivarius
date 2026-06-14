# Archivarius: Technical Brief for the IT Department

**Archivarius v3.0.3** | Corporate Search Engine  
Prepared by: Serhii Shtokal | June 2026  
Status: Concept proven by Live Demo. Specification v3.0.3 ready for development. Seeking support for implementation.  
Contact: github.com/ars4tumblr-cmd

---

## 1. What It Is and Why

Archivarius is an **on-premise corporate search engine** over the company's document repository (PDF, DOCX, TXT, HTML). Not a cloud service. Not SaaS. Not an AI assistant.

**Technically:** hybrid BM25 + vector search using a local multilingual embedding model.  
**Practically:** employees search documents via a browser — the system finds by content, not just by keywords.

---

## 2. System Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    CORPORATE SERVER                         │
│                                                             │
│  [Documents]                                                │
│      ↓ Ingestion Pipeline (PDF/DOCX/TXT/HTML)               │
│  [Chunking]                                                 │
│          ↓                        ↓                         │
│   [BM25 Indexer]        [Embedding Model]                   │
│   (sparse/keyword)       (multilingual-e5)                  │
│          ↓                        ↓                         │
│  ┌────────────────┐    ┌──────────────────────┐             │
│  │  BM25 Index    │    │  Qdrant Vector DB     │            │
│  │  (./data/bm25) │    │  (Docker container)   │            │
│  └────────┬───────┘    └──────────┬────────────┘            │
│           └──────────┬────────────┘                         │
│                      ↓ Hybrid Search + Reranking            │
│  [FastAPI Backend :8000]                                    │
│      ↓ JWT auth                                             │
│  [Next.js Frontend :3000]   [Web Admin :8080]               │
│                                                             │
│  [LibreTranslate :5000] <- optional, internal only          │
│                                                             │
│       ↕ NO outbound connections at runtime                  │
└─────────────────────────────────────────────────────────────┘
```

**Two isolated pipelines:**
- **Indexing:** background, triggered when documents are added or changed
- **Search:** online, triggered on each query (<500ms fast mode)

---

## 3. Component Stack

| Component | Technology | Version | License | Where it runs |
|-----------|-----------|---------|---------|--------------|
| Embedding model | multilingual-e5-large (Microsoft Research) | Pinned | MIT | Locally, CPU-only |
| Vector DB | Qdrant | Pinned in docker-compose | Apache 2.0 | Docker container |
| Keyword search | rank_bm25 | Pinned | Apache 2.0 | In-process |
| API framework | FastAPI + Python 3.11 | Pinned | MIT | Docker container |
| Frontend | Next.js 14 | Pinned | MIT | Docker container |
| Translation | LibreTranslate | Pinned | EUPL | Docker container (optional) |
| Orchestration | Docker Compose | Standard | Apache 2.0 | Host OS |

**All versions pinned.** The `latest` tag is forbidden for production.  
**SBOM (Software Bill of Materials — a full list of all components and their exact versions):** ready to provide for IT review.

---

## 4. Resource Requirements

### Minimum Configuration (without translator)

| Component | RAM idle | RAM peak |
|-----------|----------|----------|
| E5-Large embedding | 2.5 GB | 3.0 GB |
| Qdrant vector DB | 2.0–4.0 GB | 5.0 GB |
| BM25 + Cache | 0.6 GB | 1.3 GB |
| FastAPI backend | 0.3 GB | 1.0 GB |
| Next.js frontend | 0.2 GB | 0.5 GB |
| **Total** | **~5.6 GB** | **~10.8 GB** |
| **OS headroom** | ~5 GB on 16 GB total | |

### With Translator (LibreTranslate, optional)

| Configuration | RAM idle | RAM peak |
|--------------|----------|----------|
| Core + LibreTranslate | ~7.1 GB | ~14.3 GB |

**GPU:** not required. The system is designed for CPU-only deployment.

### Disk

| Data | Size |
|------|------|
| AI model (E5-Large) | ~6 GB |
| Qdrant storage | Depends on document corpus |
| BM25 index | ~0.5–2 GB |
| Logs + feedback | Rotation: 30/90 days |

---

## 5. Security Model

### 5.1 Network Isolation

```yaml
# docker-compose.yml — network boundary schema
services:
  backend:        # expose :8000 → only via reverse proxy
  qdrant:         # internal only, NOT exposed externally
  libretranslate: # internal only, NOT exposed externally
  frontend:       # expose :3000 → via nginx/reverse proxy

networks:
  internal:  # Qdrant and LibreTranslate — internal only
  external:  # backend and frontend — via reverse proxy
```

**Runtime:** zero outbound connections from containers. Model loaded locally (`HF_HUB_OFFLINE=1`).

### 5.2 Authentication

- **Method:** JWT Bearer tokens
- **Roles:** `user` (search, view), `admin` (indexing, configuration)
- **Secret:** via environment variable, never in code or git
- **TTL:** 24 hours (configurable)
- **Admin endpoints:** recommended to restrict to intranet/VPN allowlist

### 5.3 File Sandbox

- Ingestion accepts **only** files from `documents_dir`
- MIME-type verification (protection against path traversal)
- Forbidden extensions: `.exe`, `.sh`, `.py`, `.js`
- Maximum file size: 100 MB (configurable)

### 5.4 Logging (Privacy)

- Queries logged as `SHA-256 hash`, not plaintext
- Feedback signals: `hashed user_id` + `query_hash` (not query text)
- Retention: operational logs — 30 days, feedback — 90 days (configurable)

### 5.5 GDPR / NDA Compliance

| Requirement | Architectural Mechanism |
|-------------|------------------------|
| Data does not leave externally | Local models, Qdrant, BM25; no external API |
| Queries not logged in plaintext | SHA-256 hash in logs/cache/feedback |
| Access rights | JWT Bearer, roles user/admin |
| Retention policy | Configurable, default: 30/90 days |
| Documents not executed as code | MIME check, extension denylist |

---

## 6. Deployment

### 6.1 Standard Launch

```bash
# Prerequisites: Docker Engine or Docker Desktop
git clone <repo> archivarius
cd archivarius
python scripts/setup_models.py      # download models (one time, ~6 GB)
docker compose up -d                # start all services
curl http://localhost:8000/api/v1/health   # health check
```

### 6.2 Air-Gap / Offline

For fully isolated environments:
1. Prepare **offline artifact bundle** (Docker images + model files + checksums + SBOM)
2. Transfer to isolated server
3. `docker compose up -d` — with no internet access whatsoever

### 6.3 Production Hardening (Required)

- [ ] Image versions pinned to digest (not `latest`)
- [ ] TLS termination via nginx or corporate reverse proxy
- [ ] Qdrant and LibreTranslate — internal network only
- [ ] Admin endpoints — behind VPN/intranet allowlist
- [ ] JWT secret via env/secret store (not git)
- [ ] `HF_HUB_OFFLINE=1` in production runtime
- [ ] Backup encryption for `./data/` directory

### 6.4 Deployment Environments

| Environment | OS | Start |
|-------------|-----|-------|
| Production | Linux (Ubuntu 22.04 LTS) | `docker compose up -d` |
| Development | macOS / Linux | `docker compose up` |
| Windows (office) | Windows 10/11 + WSL2 | `start_bridge.bat` |

---

## 7. Operational Support

### 7.1 Health Monitoring

```
GET /api/v1/health
→ { "status": "ok", "components": { "qdrant": "ok", "embedding": "ok", ... } }

GET /api/v1/stats
→ { "total_documents": 1247, "total_chunks": 48392, "index_size_mb": 2150 }
```

### 7.2 Prometheus-Compatible Metrics

- `search_latency_ms` — search response time
- `indexing_documents_total` — number of indexed documents
- `ram_usage_bytes` — RAM consumption
- `index_consistency_status` — index consistency (1=ok, 0=problem)

### 7.3 Backup

```bash
# Minimal system state backup
cp -r ./data/qdrant      ./backup/qdrant_$(date +%Y%m%d)
cp    ./data/file_hashes.json ./backup/
cp -r ./data/feedback    ./backup/feedback_$(date +%Y%m%d)
```

Backup contains indices and embeddings → store encrypted (treated as equivalent to corporate documents in terms of confidentiality level).

### 7.4 Document Updates

New documents → copied to `./documents/` → automatically indexed.  
No manual steps. Changes in documents detected via SHA-256 hash.

---

## 8. Supply Chain / Component Inventory

- All components: open source, public repositories — anyone can review the code
- Python dependencies: pinned versions in `requirements.txt` (not "latest version", but specific ones)
- Docker images: pinned to specific version
- AI model: downloaded from Hugging Face, with checksum verification
- **SBOM (Software Bill of Materials — a complete list of all components, their versions, and licenses):** ready to hand over to IT for review. License inventory: MIT/Apache/EUPL — corporate use permitted free of charge.
- **Security vulnerability scanning:** before production rollout, all components are checked against known vulnerabilities in public databases (CVE — Common Vulnerabilities and Exposures — is a public database of known security flaws in software, like a catalog of patched and unpatched "holes").

---

## 9. IT FAQ

**Q: Who will maintain the system?**  
A: The system is designed so that maintenance is minimal (Docker lifecycle: start/stop/restart/backup/restore). During the pilot — technical support from the developer. For production — either an internal company developer or an external contractor familiar with the Docker/Python stack. A Runbook (step-by-step operational guide covering startup, shutdown, restore, and backup) will be provided with the prototype.

**Q: Does it need internet access?**  
A: Only during initial setup (downloading the AI model and Docker components, ~6–10 GB). After that — runtime is completely offline.

**Q: What are the licenses?**  
A: MIT, Apache 2.0, EUPL — all permit corporate use without payments. Full component list (SBOM — Software Bill of Materials, i.e. an inventory of all libraries and versions) is available at [Archivarius v3.0.3](https://github.com/ars4tumblr-cmd/Archivarius/blob/main/docs/spec_v303.md).

**Q: Is a GPU required?**  
A: No. The system is designed for CPU-only. GPU can be added for acceleration (optional).

**Q: How does it fit with Azure / M365 / SharePoint?**  
A: It does not replace but complements. Archivarius indexes documents independently of the storage platform. Integration with AD/LDAP for authentication is possible.

**Q: What if the system breaks?**  
A: Docker restart. If needed — full restore from backup (procedure described in the Runbook). The system is idempotent — repeated re-indexing produces the same result.

**Q: Can access to specific documents be restricted?**  
A: Base version: user/admin roles. Extended version (Phase 3): multi-user roles (admin/editor/viewer).

**Q: Where are the documents physically stored?**  
A: `./documents/` — originals (read-only for the system). `./data/` — indices, embeddings, logs. All on the same server, under IT control.

---

*Full technical specification: [Archivarius v3.0.3](https://github.com/ars4tumblr-cmd/Archivarius/blob/main/docs/spec_v303.md)*  
*Live Demo: https://huggingface.co/spaces/Ars4ars/Archivarius-Demo-V2*  
*Contact: Serhii Shtokal — [github.com/ars4tumblr-cmd](https://github.com/ars4tumblr-cmd)*

