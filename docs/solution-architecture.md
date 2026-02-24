# Solution Architecture Proposal: Refined SAP HANA RAG System

**Version:** 1.0
**Date:** 2026-02-24
**Status:** Draft — Pending Pilot Validation
**Author:** Claude Code via OpenClaw

---

## 1. Architecture Overview

The Refined RAG system follows a **hybrid retrieval architecture** that handles both structured (tabular SAP data) and unstructured (email, documents) content through differentiated pipelines that converge at a unified search API.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         REFINED SAP HANA RAG                         │
│                         End-to-End Architecture                      │
├──────────────┬──────────────────────────────┬───────────────────────┤
│   DATA       │        PROCESSING            │        SERVING        │
│   SOURCES    │        PIPELINE              │        LAYER          │
├──────────────┴──────────────────────────────┴───────────────────────┤
│                                                                       │
│  SAP HANA DB1  ──► HANA Connector ──► Schema     ──►  Qdrant         │
│  (160 GB)           (hdbcli)          Normalizer      Vector Store    │
│                                           │                │         │
│  SAP HANA DB2  ──► HANA Connector ──►  Chunker   ──►  PostgreSQL     │
│  (260 GB)           (hdbcli)           (per type)     Metadata DB    │
│                                           │                │         │
│  Email Archive ──► Email Parser   ──►  PII Scan  ──►  MinIO          │
│  (IMAP/Graph)       (Python)          (Presidio)     Document Store  │
│                                           │                │         │
│  Documents     ──► PDF/OCR        ──►  Embedder  ──►  Redis Cache    │
│  (PDF/DOCX)        Extractor           (E5/OAI)                      │
│                                                            │         │
│                                                      FastAPI RAG API  │
│                                                            │         │
│                                                      AI Agents ◄────  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Component Architecture

### 2.1 Layer 1: Data Extraction

#### HANA Connector (`src/extraction/hana_connector.py`)
- Connects via `hdbcli` using read-only service account
- Extracts tables in configurable batch sizes (default: 10,000 rows)
- Tracks last-extracted timestamp per table for delta processing
- Outputs: Parquet files to MinIO staging area

#### Email Connector (`src/extraction/email_connector.py`)
- IMAP or Microsoft Graph API (configurable)
- Extracts: subject, body (plain text), sender, recipients, date, attachments
- Handles MIME decoding, HTML stripping
- Outputs: JSONL files to MinIO staging area

#### Document Extractor (`src/extraction/document_extractor.py`)
- PDF: `pdfplumber` (native text) + `pytesseract` (OCR fallback for scanned docs)
- DOCX: `python-docx`
- Outputs: Plain text + metadata JSON to MinIO

### 2.2 Layer 2: Processing Pipeline (Airflow DAGs)

```
DAG: hana_structured_pipeline
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐
  │ Extract │ ─► │ Validate │ ─► │ PII Scan │ ─► │  Chunk   │ ─► │ Embed  │
  │ (HANA)  │    │ Schema   │    │ Presidio │    │ Strategy │    │ E5/OAI │
  └─────────┘    └──────────┘    └──────────┘    └──────────┘    └────────┘
                                                                       │
                                                                  ┌────────┐
                                                                  │ Index  │
                                                                  │Qdrant  │
                                                                  └────────┘

DAG: unstructured_pipeline
  ┌─────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌────────┐
  │ Extract │ ─► │  Parse   │ ─► │ PII Scan │ ─► │  Chunk   │ ─► │ Embed  │
  │ Email/  │    │ + Clean  │    │ Presidio │    │ Semantic │    │ E5/OAI │
  │  Doc    │    │          │    │          │    │          │    │        │
  └─────────┘    └──────────┘    └──────────┘    └──────────┘    └────────┘
```

#### Chunking Strategies by Data Type

| Data Type              | Strategy                          | Chunk Size  | Overlap |
|------------------------|-----------------------------------|-------------|---------|
| Financial transactions | Row-as-chunk (1 record = 1 chunk) | N/A         | N/A     |
| Sales orders           | Header + line items together      | ~500 tokens | 0       |
| HR records             | Employee profile as single chunk  | ~800 tokens | 0       |
| Emails                 | Recursive character splitter      | 512 tokens  | 64      |
| PDF documents          | Semantic paragraph splitter       | 512 tokens  | 64      |
| Master data            | Entity-as-chunk                   | ~300 tokens | 0       |

### 2.3 Layer 3: Storage

#### Qdrant Vector Store
- **Collections per domain** (separate namespaces for access control):
  - `finance_vectors` — GL, cost centers, P&L
  - `sales_vectors` — Orders, invoices, customers
  - `hr_vectors` — Employee data (restricted)
  - `supply_chain_vectors` — PO, inventory
  - `email_vectors` — Email archive
  - `documents_vectors` — PDF/DOCX content

- **Vector dimensions:** 1024 (E5-large) or 3072 (text-embedding-3-large)
- **Index type:** HNSW (ef=128, m=16) — optimized for recall vs. speed
- **Metadata stored per vector:**
  ```json
  {
    "source_table": "VBAK",
    "record_id": "1234567",
    "domain": "sales",
    "sensitivity": "medium",
    "created_at": "2024-01-15",
    "sap_user_groups": ["SALES_BASIC", "SALES_MANAGER"],
    "chunk_index": 0,
    "pii_masked": true
  }
  ```

#### PostgreSQL Metadata Database
- Tables: `chunks`, `documents`, `ingestion_runs`, `query_audit_log`
- Stores: chunk → source record mapping, ingestion history, query logs
- Schema in `database/init/02-rag-schema.sql`

#### MinIO Object Store
- Buckets: `raw-extracts`, `processed-text`, `embeddings-cache`
- Lifecycle: raw extracts expire after 7 days; processed text retained indefinitely

### 2.4 Layer 4: Search & Serving

#### Search Pipeline (per query)
```
User Query
    │
    ▼
Query Preprocessing
  - Language detection
  - Intent classification (structured vs. unstructured)
  - Domain routing (which collections to search)
    │
    ├──► Dense Search (Qdrant ANN, top-20)
    │
    └──► Sparse Search (BM25 on PostgreSQL full-text index, top-20)
              │
              ▼
         RRF Fusion (Reciprocal Rank Fusion, merge top-20+20)
              │
              ▼
         Access Filter (remove docs user can't see)
              │
              ▼
         Re-ranker (cross-encoder, top-5)
              │
              ▼
         Context Assembly (build prompt context)
              │
              ▼
         LLM Synthesis (MiniMax-M2.5 / Ollama)
              │
              ▼
         Response + Citations
```

#### FastAPI Endpoints
```
POST /v1/search              — Vector search (raw results, no LLM)
POST /v1/chat/completions    — OpenAI-compatible RAG endpoint
GET  /v1/health              — Health check
GET  /v1/domains             — List available data domains
POST /v1/admin/reindex       — Trigger re-indexing (admin only)
GET  /v1/audit/queries       — Query audit log (admin only)
```

---

## 3. Security Architecture

### 3.1 Authentication Flow
```
AI Agent → API Gateway (JWT Validation) → FastAPI RAG API
                                               │
                                    SAP Group Lookup (LDAP/AD)
                                               │
                                    Vector Filter (metadata filter on
                                    sap_user_groups field in Qdrant)
```

### 3.2 Data Sensitivity Tiers

| Tier | Examples                    | Access Controls                        |
|------|-----------------------------|----------------------------------------|
| 0    | HR salaries, personal data  | Named individuals only, encrypted      |
| 1    | Financial P&L, forecasts    | Finance managers + C-suite             |
| 2    | Sales orders, customer data | Sales team + managers                  |
| 3    | Public product/material     | All authenticated users                |

### 3.3 PII Masking Rules
- **Names:** Replaced with `[PERSON]`
- **Email addresses:** Hashed with SHA-256 prefix (`[EMAIL:abc12]`)
- **Phone numbers:** Replaced with `[PHONE]`
- **Bank accounts / IBAN:** Replaced with `[FINANCIAL_ACCOUNT]`
- **Salaries / compensation:** Replaced with `[SALARY_DATA]` unless Tier 0 access
- SAP employee IDs: Kept as-is (non-PII reference key), but linked data masked

---

## 4. Data Flow Diagrams

### 4.1 Initial Ingestion Flow (One-Time / Full Load)
```
Week 1-2:  Schema Discovery → Table Ranking → Priority List
Week 2-4:  Extract High-Priority Tables (Sales, Finance) → Embed → Index
Week 4-6:  Extract Remaining Structured Data → Embed → Index
Week 6-8:  Extract Email + Documents → Embed → Index
Week 8+:   Switch to delta/CDC mode
```

### 4.2 Ongoing Delta Sync Flow
```
Every Night (02:00):
  Airflow DAG triggers
       │
       ├── Check SAP HANA change tables (CDHDR/CDPOS) → extract deltas
       ├── Re-embed changed records
       ├── Upsert vectors in Qdrant (by record_id)
       └── Log ingestion run to PostgreSQL
```

---

## 5. Infrastructure Sizing

### 5.1 Pilot Environment (Docker Compose)

| Service          | CPU   | RAM   | Storage |
|------------------|-------|-------|---------|
| Qdrant           | 4 vCPU | 16 GB | 100 GB SSD |
| PostgreSQL       | 2 vCPU | 4 GB  | 50 GB  |
| Airflow          | 2 vCPU | 4 GB  | 20 GB  |
| FastAPI (x2)     | 2 vCPU | 2 GB  | —      |
| MinIO            | 2 vCPU | 2 GB  | 200 GB |
| Redis            | 1 vCPU | 2 GB  | 10 GB  |
| Embedding Worker | 4 vCPU + GPU (opt) | 8 GB | — |

### 5.2 Production Environment (Kubernetes)

| Service          | Replicas | CPU/replica | RAM/replica |
|------------------|----------|-------------|-------------|
| Qdrant cluster   | 3        | 8 vCPU      | 32 GB       |
| PostgreSQL HA    | 1+1 replica | 4 vCPU   | 16 GB       |
| FastAPI          | 3–5 (auto-scale) | 2 vCPU | 4 GB   |
| Embedding Workers | 2–8 (auto-scale) | 4 vCPU | 8 GB  |
| Airflow          | 1 scheduler + 4 workers | varies | varies |

---

## 6. Integration Points

### 6.1 SAP HANA Integration
- Connection: `hdbcli` with encrypted connection (TLS)
- Authentication: Service user `RAG_READ_ONLY` (read-only, no table creation)
- Extraction: Direct SELECT queries + stored procedure calls for complex joins
- Change detection: Monitor `SYS.AUDIT_LOG` or application-level change tables

### 6.2 AI Agent Integration
```python
# Example: LangChain integration
from langchain.retrievers import RemoteRAGRetriever

retriever = RemoteRAGRetriever(
    url="http://rag-api:8000/v1/search",
    headers={"Authorization": f"Bearer {jwt_token}"},
    k=5
)

# Example: Direct API call
import httpx
response = httpx.post("http://rag-api:8000/v1/chat/completions", json={
    "model": "refined-rag",
    "messages": [{"role": "user", "content": "Show Q3 revenue by region"}],
    "metadata": {"user_id": "jsmith", "sap_groups": ["FI_MANAGER"]}
})
```

### 6.3 MiniMax-M2.5 / OpenClaw Integration
- RAG API acts as a tool within OpenClaw agent workflows
- Agent calls RAG API → receives context passages → constructs final answer via MiniMax

---

## 7. Deployment Strategy

### Phase 1: Pilot (Weeks 1–8)
- Deploy on single server with Docker Compose
- Ingest only Sales + Email domain (~30 GB pilot subset)
- 5 pilot users, weekly feedback sessions
- Success gate: precision@5 > 80%, latency < 3s

### Phase 2: Expanded Pilot (Weeks 9–16)
- Add Finance + Supply Chain domains
- Introduce access control tiers
- Load test with 20 concurrent users
- Success gate: no PII leakage, stable under load

### Phase 3: Production (Weeks 17–28)
- Migrate to Kubernetes
- Ingest full 420 GB
- Integrate with SSO/AD
- Full audit logging enabled
- SLA monitoring active

---

## 8. Risk Register

| Risk                                  | Likelihood | Impact | Mitigation                                    |
|---------------------------------------|------------|--------|-----------------------------------------------|
| HANA extraction impacts production    | MEDIUM     | HIGH   | Read-only accounts, extract during off-hours  |
| PII leakage in embeddings             | LOW        | CRITICAL | Presidio scan before embedding, regular audits |
| Vector store performance at scale     | MEDIUM     | HIGH   | HNSW tuning, horizontal scaling plan ready    |
| Email data not in HANA                | MEDIUM     | MEDIUM | Confirm connector; fall back to IMAP/Exchange |
| Embedding model quality               | LOW        | MEDIUM | Evaluate 3 models on sample data before commit |
| GDPR erasure requests                 | LOW        | HIGH   | Soft-delete + re-embed pipeline for erasure   |
| SAP schema changes break extractor    | LOW        | MEDIUM | Schema version monitoring, alerting on errors |

---

## 9. Architecture Decision Records (ADRs)

### ADR-001: Vector Store Selection — Qdrant vs pgvector vs Pinecone
- **Decision:** Qdrant (self-hosted)
- **Rationale:** Supports metadata filtering, self-hosted (data residency), production-grade at 50M+ vectors, active open-source project
- **Alternatives rejected:** pgvector (limited at scale), Pinecone (cloud = data residency issue)

### ADR-002: Embedding Model — Local vs API
- **Decision:** Local `intfloat/multilingual-e5-large` for pilot; API option as fallback
- **Rationale:** Data residency requirement; no SAP data leaves on-premise infrastructure
- **Alternatives:** OpenAI `text-embedding-3-large` (pending data residency approval)

### ADR-003: Hybrid Search (Dense + Sparse)
- **Decision:** Implement hybrid search with RRF fusion
- **Rationale:** Structured SAP data (part numbers, GL codes) benefits from exact-match BM25; semantic search handles natural language descriptions
- **Trade-off:** Increased complexity vs. significant retrieval improvement for enterprise data

### ADR-004: Chunking — Per-Type Strategy
- **Decision:** Different chunking for structured vs unstructured
- **Rationale:** A sales order row should not be split; an email body should be chunked semantically
- **Implementation:** `ChunkingStrategyFactory` class with per-domain configuration
