# Archivarius: Security Assurance
## Corporate Security by Architecture — Not by Policy

**Version:** 3.0.3 | **Type:** On-premise corporate search engine | **Date:** June 2026

---

## 8 Key Security Claims

### 1. The System Is a Local Search Engine — Not a Cloud AI Service

Archivarius is not a SaaS product, not a cloud service, and not an AI assistant that connects to external models. It is **on-premise search infrastructure** — all components run on your corporate server.

---

### 2. Data Moves ONLY Within the Corporate Perimeter

```
[Documents] -> [Ingestion] -> [Indexing] -> [Search] -> [User]
     ^                                                     |
     |_______ Everything inside corporate network _________|
                  ZERO outbound connections at runtime
```

---

### 3. Data Boundary: What Lives Where

| Data type | Where stored | Leaves the perimeter? |
|-----------|-------------|----------------------|
| Original documents | `./documents/` on corporate server | вќЊ Never |
| Vector indices (embeddings) | `./data/qdrant/` on the same server | вќЊ Never |
| BM25 index | `./data/bm25_index/` on the same server | вќЊ Never |
| Query logs | `./data/logs/` — SHA-256 hash only, no query text | вќЊ Never |
| Feedback signals | `./data/feedback/` — hashed IDs only | вќЊ Never |
| AI model | `./models/` — downloaded once during setup | вќЊ Never |

---

### 4. Runtime Requires No External Services

| External resource | Status during operation |
|-------------------|------------------------|
| Hugging Face Hub | вќЊ Blocked (`HF_HUB_OFFLINE=1`) |
| OpenAI / Anthropic API | вќЊ Not used |
| Google Translate / DeepL | вќЊ Not used |
| Any cloud vector database | вќЊ Not used |
| External telemetry | вќЊ Not collected |
| **Summary** | **System operates fully offline** |

---

### 5. Access Control

| Layer | Mechanism |
|-------|-----------|
| Authentication | JWT Bearer tokens |
| Roles | `user` (search only), `admin` (management) |
| Secret management | Environment variables — not stored in version control |
| Admin endpoints | Recommended: intranet/VPN allowlist |
| File sandbox | MIME type check + extension denylist + file size limit |
| Network | Docker internal network for Qdrant and LibreTranslate — not exposed externally |

---

### 6. Supply Chain and Open Source Components

| Requirement | Implementation |
|-------------|---------------|
| Components available for inspection | All open source — Qdrant, FastAPI, rank_bm25 |
| Versions pinned | Pinned versions in `requirements.txt` and `docker-compose.yml` |
| SBOM | Software Bill of Materials — a complete list of every software component and its exact version. Ready to hand over to IT for security review. |
| Licence inventory | MIT, Apache 2.0, EUPL — all permit commercial/corporate use without restriction |
| Vulnerability scan | Performed before production rollout |
| AI model | multilingual-e5-large by Microsoft Research (MIT licence) — downloaded with checksum verification, reproducible |

---

### 7. Operational Readiness

| Requirement | Status |
|-------------|--------|
| Docker Compose deployment | ✅ Standard |
| Health endpoint `/api/v1/health` | ✅ Implemented |
| Prometheus-compatible metrics | ✅ Implemented |
| JSON structured logs | ✅ Implemented |
| Backup/restore procedure | ✅ Documented |
| IT Runbook | рџљ§ Will be created together with the prototype |
| Air-gap / offline provisioning | ✅ Supported |

---

### 8. Residual Risks and Mitigations

| Risk | Level | Mitigation |
|------|-------|-----------|
| Vulnerabilities in PDF/DOCX parsers | Medium | MIME check + pre-ingest gate; library update process |
| Embeddings may contain derived information | Low | Backup encryption; access control on `./data/` |
| Provisioning requires internet (one-time only) | Low | Offline artifact bundle available for air-gap environments |
| Behavioural data in feedback (GDPR-sensitive) | Low | 90-day TTL, hashed IDs, automated retention job |

---

## GDPR / NDA Control Matrix

| GDPR / NDA Requirement | Architectural Mechanism | Additional Production Control |
|------------------------|------------------------|-------------------------------|
| Data does not leave the perimeter | Offline runtime, local models, no external API | Egress firewall policy |
| Queries not stored as plaintext | SHA-256 hash in logs | Log schema validation |
| Right to erasure | `delete_document` API → clears all derived indices | Procedure in runbook |
| Retention policy | Configurable TTL for logs and feedback | Automated retention job |
| Access control | JWT roles | Secret rotation cadence |

---

## Corporate Readiness Profile

✅ **Security:** documents, indices, and queries remain within company infrastructure
✅ **Independence:** core search runs without cloud AI, SaaS, or external APIs
✅ **Operational simplicity:** Docker Compose, health checks, structured logs, backup strategy
✅ **Supply chain:** pinned versions, SBOM, licence inventory, all open source
✅ **GDPR ready:** by architecture — no additional measures required

---

*Full technical specification: [Archivarius v3.0.3](https://github.com/ars4tumblr-cmd/Archivarius/blob/main/docs/spec_v303.md)*
*Contact: Serhii Shtokal — [github.com/ars4tumblr-cmd](https://github.com/ars4tumblr-cmd)*

