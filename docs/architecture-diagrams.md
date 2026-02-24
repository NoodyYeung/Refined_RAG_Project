# Architecture Diagrams: Refined SAP HANA RAG System

**Version:** 1.0
**Date:** 2026-02-24
**Status:** Draft
**Author:** Claude Code via OpenClaw
**Note:** Diagrams rendered as Mermaid (GitHub-compatible) + ASCII art for plain-text viewing.
**Miro:** Miro board creation pending Miro MCP configuration.

---

## Diagram 1: Data Architecture — Sources → Processing → Storage

```mermaid
flowchart LR
    subgraph Sources["🗄️ Data Sources (SAP HANA)"]
        H1["HANA DB1\n160 GB\nFinance / HR"]
        H2["HANA DB2\n260 GB\nSales / Supply Chain"]
        EM["Email Archive\n~50 GB\nIMAP / Exchange"]
        DO["Documents\n~50 GB\nPDF / DOCX"]
    end

    subgraph Extract["📤 Extraction Layer"]
        HC["HANA Connector\n(hdbcli, read-only)"]
        EC["Email Connector\n(IMAP / Graph API)"]
        DE["Document Extractor\n(pdfplumber / OCR)"]
    end

    subgraph Process["⚙️ Processing Pipeline (Airflow)"]
        SD["Schema Discovery\n& Validation"]
        PII["PII Scanner\n(Presidio)"]
        CK["Chunker\n(per-type strategy)"]
        EM2["Embedder\n(E5-large / OAI)"]
    end

    subgraph Store["💾 Storage Layer"]
        QD["Qdrant\nVector Store\n~228 GB"]
        PG["PostgreSQL\nMetadata DB"]
        MN["MinIO\nDocument Store"]
        RD["Redis\nQuery Cache"]
    end

    subgraph Serve["🚀 Serving Layer"]
        API["FastAPI\nRAG API\n:8000"]
        RR["Re-ranker\n(cross-encoder)"]
        LLM["LLM\n(MiniMax-M2.5)"]
    end

    subgraph Agents["🤖 AI Agents"]
        AG1["OpenClaw Agent"]
        AG2["LangChain Agent"]
        AG3["Custom Agent"]
    end

    H1 --> HC
    H2 --> HC
    EM --> EC
    DO --> DE

    HC --> SD
    EC --> SD
    DE --> SD

    SD --> PII
    PII --> CK
    CK --> EM2

    EM2 --> QD
    SD --> PG
    SD --> MN
    QD --> RD

    QD --> API
    PG --> API
    RD --> API
    API --> RR
    RR --> LLM
    LLM --> API

    API --> AG1
    API --> AG2
    API --> AG3
```

---

## Diagram 2: RAG Approach by Data Type (Decision Flow)

```mermaid
flowchart TD
    START["📥 Data Category"] --> Q1{Is data\nprimarily TEXT?}

    Q1 -->|Yes| Q2{Long-form\nor short?}
    Q1 -->|No\nTabular/Structured| Q3{User needs\naggregations?}

    Q2 -->|Long-form\nEmails, Documents| V1["🔵 VECTOR-ONLY RAG\nSemantic chunking\n512 tokens + overlap\nCollection: email_vectors\ndocuments_vectors"]
    Q2 -->|Short-form\nMaster data descriptions| V2["🔵 VECTOR-ONLY RAG\nEntity-as-chunk\nCollection: material_vectors\ncustomer_vectors"]

    Q3 -->|Yes KPIs, Reports| S1["🟡 SQL-RAG ONLY\nPre-built query templates\nNo embeddings\nBW / Analytics domain"]
    Q3 -->|No Record lookup| Q4{Sensitivity\nlevel?}

    Q4 -->|CRITICAL HR| R1["🔴 RESTRICTED VECTOR\nIsolated collection\nNamed-user access only\nCollection: hr_vectors"]
    Q4 -->|HIGH Finance| H1["🟠 HYBRID RAG\nVector + SQL\nFI_MANAGER group\nCollection: finance_vectors"]
    Q4 -->|MEDIUM Sales/SC| H2["🟢 HYBRID RAG\nVector + SQL\nStandard auth\nCollections: sales_vectors\nsupply_chain_vectors"]

    style V1 fill:#2196F3,color:#fff
    style V2 fill:#2196F3,color:#fff
    style S1 fill:#FF9800,color:#fff
    style R1 fill:#F44336,color:#fff
    style H1 fill:#FF5722,color:#fff
    style H2 fill:#4CAF50,color:#fff
```

---

## Diagram 3: Hybrid System Architecture (Components & Flow)

```mermaid
flowchart TB
    subgraph User["👤 User / Agent Query"]
        Q["Natural Language Query\n'Show Q3 revenue by region'\n'Find emails about Project X'"]
    end

    subgraph Auth["🔐 Authentication"]
        JWT["JWT Token\nValidation"]
        LDAP["SAP Group\nLookup (LDAP)"]
        GROUPS["User's SAP\nAuth Groups"]
    end

    subgraph Router["🗺️ Query Router"]
        LANG["Language\nDetection"]
        INTENT["Intent\nClassifier"]
        DOMAIN["Domain\nRouter"]
    end

    subgraph Search["🔍 Search Engines"]
        DENSE["Dense Search\nQdrant ANN\nTop-20"]
        SPARSE["Sparse Search\nBM25 / FTS\nTop-20"]
        SQL_S["SQL Query\nGenerator\n(for BW/aggs)"]
    end

    subgraph Fusion["🔀 Result Fusion"]
        RRF["Reciprocal Rank\nFusion (RRF)"]
        FILTER["Access Control\nFilter"]
        RERANK["Re-ranker\ncross-encoder\nTop-5"]
    end

    subgraph Synthesis["✨ Answer Synthesis"]
        CTX["Context\nAssembly"]
        LLM2["MiniMax-M2.5\nLLM Synthesis"]
        CITE["Citation\nExtraction"]
    end

    subgraph Response["📤 Response"]
        ANS["Answer Text\n+ Source Citations\n+ Confidence Score"]
        LOG["Audit Log\n(PostgreSQL)"]
    end

    Q --> JWT
    JWT --> LDAP
    LDAP --> GROUPS
    GROUPS --> LANG
    LANG --> INTENT
    INTENT --> DOMAIN

    DOMAIN -->|Semantic query| DENSE
    DOMAIN -->|Keyword query| SPARSE
    DOMAIN -->|Aggregation query| SQL_S

    DENSE --> RRF
    SPARSE --> RRF
    SQL_S --> CTX

    RRF --> FILTER
    FILTER -->|Apply SAP groups| RERANK
    RERANK --> CTX

    CTX --> LLM2
    LLM2 --> CITE
    CITE --> ANS
    ANS --> LOG
```

---

## Diagram 4: Data-to-RAG Category Mapping (Visual Summary)

```mermaid
quadrantChart
    title Data-to-RAG Approach by Volume & Sensitivity
    x-axis Low Volume --> High Volume
    y-axis Low Sensitivity --> High Sensitivity
    quadrant-1 Hybrid RAG (Careful)
    quadrant-2 SQL-RAG / Restricted
    quadrant-3 Vector RAG (Fast pilot)
    quadrant-4 Hybrid RAG (Priority)
    HR Payroll: [0.15, 0.95]
    Finance GL: [0.75, 0.8]
    Email Archive: [0.6, 0.45]
    Sales Orders: [0.55, 0.4]
    Supply Chain: [0.5, 0.35]
    BW Analytics: [0.7, 0.3]
    Material Master: [0.3, 0.15]
    Documents: [0.6, 0.35]
    HR Org: [0.2, 0.65]
    Customers: [0.25, 0.3]
```

---

## Diagram 5: Ingestion Pipeline Timeline (Phases)

```mermaid
gantt
    title Refined RAG — Data Ingestion Phases
    dateFormat  YYYY-MM-DD
    section Phase 0: Pilot
    Schema Discovery & Profiling   :p0a, 2026-03-01, 14d
    Sensitivity Scan (top 50 tables) :p0b, after p0a, 7d
    Pilot: Sales Orders Pipeline    :p0c, after p0b, 14d
    Pilot: Email Archive Pipeline   :p0d, after p0b, 14d
    Pilot Validation & Tuning       :p0e, after p0c, 14d

    section Phase 1: Finance + SC
    Finance GL + CO Ingestion       :p1a, after p0e, 21d
    Supply Chain Ingestion          :p1b, after p0e, 14d
    Access Control Integration      :p1c, after p1a, 7d

    section Phase 2: Full Dataset
    HR Domain (restricted)          :p2a, after p1c, 14d
    BW SQL-RAG Templates            :p2b, after p1c, 7d
    Documents & Attachments         :p2c, after p1c, 21d
    Delta/CDC Mode Setup            :p2d, after p2a, 14d

    section Phase 3: Production
    Kubernetes Migration            :p3a, after p2d, 21d
    SSO Integration                 :p3b, after p2d, 14d
    Full Monitoring & Alerting      :p3c, after p3a, 7d
    Production Go-Live              :milestone, after p3c, 0d
```

---

## Diagram 6: Security & Data Flow

```mermaid
sequenceDiagram
    actor Agent as AI Agent
    participant API as FastAPI RAG API
    participant Auth as Auth Service (LDAP)
    participant QD as Qdrant
    participant PG as PostgreSQL
    participant LLM as MiniMax-M2.5

    Agent->>API: POST /v1/chat/completions\n{query, user_id, jwt_token}
    API->>Auth: Validate JWT + Lookup SAP groups
    Auth-->>API: {user_id, sap_groups: ["SALES_BASIC"]}

    API->>QD: Search sales_vectors\n{query_vector, filter: sap_groups ∈ ["SALES_BASIC"]}
    QD-->>API: Top-20 results with metadata

    API->>PG: Fetch chunk text for result IDs
    PG-->>API: Chunk texts + source references

    API->>API: Re-rank top-5 results

    API->>LLM: Synthesize answer\n{query, context: [top-5 chunks]}
    LLM-->>API: Answer text

    API->>PG: Audit log: {user_id, query_hash, domains_accessed, timestamp}

    API-->>Agent: {answer, citations, confidence_score}
```

---

## ASCII Art: System Overview (Plain Text)

For environments where Mermaid is not rendered:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                   REFINED SAP HANA → RAG SYSTEM OVERVIEW                    │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────┐     ┌──────────────────────────────────────────┐
│      DATA SOURCES           │     │            PROCESSING PIPELINE            │
│                             │     │                                            │
│  ┌─────────────────────┐    │     │  ┌─────────┐  ┌────────┐  ┌──────────┐  │
│  │   SAP HANA DB1      │    │     │  │ Extract │  │  PII   │  │  Chunk   │  │
│  │   160 GB            │────┼────►│  │  Layer  │─►│  Scan  │─►│ Strategy │  │
│  │  Finance/HR         │    │     │  │ (hdbcli)│  │Presidio│  │ per-type │  │
│  └─────────────────────┘    │     │  └─────────┘  └────────┘  └────┬─────┘  │
│                             │     │                                 │        │
│  ┌─────────────────────┐    │     │                            ┌────▼─────┐  │
│  │   SAP HANA DB2      │    │     │                            │  Embed   │  │
│  │   260 GB            │────┼────►│                            │ E5-large │  │
│  │  Sales/Supply       │    │     │                            │or OAI API│  │
│  └─────────────────────┘    │     │                            └────┬─────┘  │
│                             │     └─────────────────────────────────┼────────┘
│  ┌─────────────────────┐    │                                        │
│  │   Email Archive     │    │     ┌──────────────────────────────────┼────────┐
│  │   ~50 GB            │────┼────►│           STORAGE LAYER          │        │
│  │  IMAP/Exchange      │    │     │                                  ▼        │
│  └─────────────────────┘    │     │  ┌──────────────────────────────────┐    │
│                             │     │  │         Qdrant Vector Store       │    │
│  ┌─────────────────────┐    │     │  │  finance_vectors  sales_vectors   │    │
│  │   Documents         │    │     │  │  hr_vectors       email_vectors   │    │
│  │   ~50 GB            │────┼────►│  │  ~28.5M vectors   ~228 GB        │    │
│  │  PDF/DOCX           │    │     │  └──────────────────┬───────────────┘    │
│  └─────────────────────┘    │     │                     │                    │
└─────────────────────────────┘     │  ┌──────────┐  ┌───┴────┐  ┌────────┐   │
                                    │  │PostgreSQL│  │ Redis  │  │ MinIO  │   │
                                    │  │Metadata  │  │ Cache  │  │  Docs  │   │
                                    │  └────┬─────┘  └───┬────┘  └────────┘   │
                                    └───────┼────────────┼────────────────────┘
                                            │            │
                                    ┌───────▼────────────▼────────────────────┐
                                    │              SERVING LAYER               │
                                    │                                          │
                                    │  ┌─────────────────────────────────┐    │
                                    │  │     FastAPI RAG API :8000        │    │
                                    │  │  POST /v1/search                 │    │
                                    │  │  POST /v1/chat/completions       │    │
                                    │  └──────────────┬──────────────────┘    │
                                    │                 │                        │
                                    │  ┌──────────────▼──────────────────┐    │
                                    │  │  Re-ranker → LLM Synthesis      │    │
                                    │  │  (cross-encoder + MiniMax-M2.5) │    │
                                    │  └──────────────┬──────────────────┘    │
                                    └─────────────────┼───────────────────────┘
                                                      │
                                    ┌─────────────────▼───────────────────────┐
                                    │              AI AGENTS                   │
                                    │  OpenClaw │ LangChain │ Custom Agents    │
                                    └─────────────────────────────────────────┘
```

---

## Notes on Miro Visualization

These diagrams are prepared as Mermaid-compatible markdown. Once Miro MCP is configured, the following boards should be created:

| Board Name                           | Source Diagram | Priority |
|--------------------------------------|----------------|----------|
| Refined RAG — Data Architecture      | Diagram 1      | HIGH     |
| Refined RAG — Decision Flow          | Diagram 2      | HIGH     |
| Refined RAG — System Architecture    | Diagram 3      | HIGH     |
| Refined RAG — Data Mapping Matrix    | Diagram 4      | MEDIUM   |
| Refined RAG — Ingestion Timeline     | Diagram 5      | MEDIUM   |
| Refined RAG — Security Flow          | Diagram 6      | HIGH     |

To create Miro boards when MCP is available:
```bash
# Example: create board via Miro MCP (once configured in ~/.claude/settings.json)
# miro create_board --name "Refined RAG Architecture" --team_id <team_id>
# miro create_shape --type flowchart --content <diagram_content>
```
