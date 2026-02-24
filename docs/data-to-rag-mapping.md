# Data-to-RAG Mapping: Refined SAP HANA

**Version:** 1.1 (Finalized from Draft)
**Date:** 2026-02-24
**Status:** Ready for Pilot Implementation
**Author:** Claude Code via OpenClaw

---

## 1. Overview

This document maps each category of Refined's SAP HANA data to:
- The appropriate **RAG approach** (structured SQL-RAG, vector-only, hybrid)
- **Chunking strategy**
- **Key fields** to extract and embed
- **Sensitivity classification**
- **Estimated vector count**

---

## 2. Mapping Table

| Domain | SAP Tables (Key) | Data Volume | RAG Approach | Chunk Strategy | Sensitivity | Pilot? |
|--------|-----------------|-------------|--------------|----------------|-------------|--------|
| Finance — GL | BKPF, BSEG | ~80 GB | Hybrid (SQL + vector) | Row-as-chunk | HIGH | No |
| Finance — Controlling | COEP, COSS | ~20 GB | Hybrid | Row-as-chunk | HIGH | No |
| Sales Orders | VBAK, VBAP | ~40 GB | Hybrid | Header+lines | MEDIUM | **YES** |
| Invoices / Billing | VBRK, VBRP | ~25 GB | Hybrid | Header+lines | MEDIUM | **YES** |
| Customers | KNA1, KNB1 | ~5 GB | Vector | Entity-chunk | MEDIUM | No |
| Supply Chain / PO | EKKO, EKPO | ~30 GB | Hybrid | Header+lines | MEDIUM | No |
| Inventory | MARD, MSEG | ~25 GB | Hybrid | Row-as-chunk | LOW | No |
| Material Master | MARA, MAKT | ~10 GB | Vector | Entity-chunk | LOW | No |
| HR — Employees | PA0000–PA9999 | ~15 GB | Vector (restricted) | Entity-chunk | CRITICAL | No |
| HR — Org Structure | HRP1000–HRP1001 | ~5 GB | Vector | Entity-chunk | HIGH | No |
| BW Analytics | InfoCubes, DSOs | ~60 GB | SQL-RAG only | Aggregation | MEDIUM | No |
| Email Archive | HANA tables / IMAP | ~50 GB | Vector-only | Semantic (512t) | VARIABLE | **YES** |
| Attachments / PDF | SOFFPHIO, SOFFCONT | ~50 GB | Vector-only | Semantic (512t) | VARIABLE | No |
| **Total** | — | **~415 GB** | — | — | — | ~115 GB |

---

## 3. Detailed Mapping by Domain

### 3.1 Finance — General Ledger (GL)

**SAP Tables:** `BKPF` (document header), `BSEG` (document line items)
**Volume estimate:** ~80 GB
**RAG Approach:** Hybrid — SQL-RAG for aggregations + vector search for semantic queries

**Extraction Fields:**
```
BKPF: BUKRS (company code), BELNR (doc number), BLDAT (doc date),
      BLART (doc type), BKTXT (doc header text), USNAM (user), XBLNR (ref)
BSEG: BUZEI (line), HKONT (GL account), DMBTR (amount), SGTXT (line text),
      KOSTL (cost center), AUFNR (order), MATNR (material)
```

**Chunk Format (example):**
```
GL Document 1234567 | Company: 1000 | Date: 2024-03-15 | Type: KR (Vendor Invoice)
Amount: EUR 45,230.00 | GL Account: 410000 (Raw Materials)
Cost Center: CC_PROD_001 | Reference: INV-2024-00892
Text: "Invoice from Supplier X for material batch B-2024-03"
```

**Sensitivity:** HIGH — Mask amounts above threshold; restrict to FI_MANAGER group
**PII Fields:** None directly (but USNAM maps to employee — pseudonymize)
**Estimated vectors:** ~8M (1 chunk per line item, 80M lines / 10 sample rows per doc)

---

### 3.2 Sales Orders & Invoices

**SAP Tables:** `VBAK` (order header), `VBAP` (order items), `VBRK` (billing header), `VBRP` (billing items)
**Volume estimate:** ~65 GB
**RAG Approach:** Hybrid — vector for semantic, SQL for aggregations

**Extraction Fields:**
```
VBAK: VBELN (order no), AUDAT (order date), KUNNR (customer),
      VKORG (sales org), NETWR (net value), BSTNK (PO number), VTEXT (desc)
VBAP: POSNR (line), MATNR (material), ARKTX (description),
      KWMENG (quantity), NETPR (net price), MWSBP (tax)
```

**Chunk Format (example):**
```
Sales Order SO-2024-001234 | Customer: Acme Corp (C-10001) | Date: 2024-06-12
Sales Org: 1000 | Total Value: EUR 128,500.00
Items: (3) Material XYZ-001 "Industrial Widget" x500 @ EUR 85.00
       Material ABC-002 "Mounting Kit" x500 @ EUR 42.00 ...
PO Reference: PO-ACM-2024-0567 | Delivery: 2024-06-30
```

**Sensitivity:** MEDIUM — Customer names masked if individual (B2C); B2B company names OK
**Estimated vectors:** ~5M

---

### 3.3 HR & Payroll

**SAP Tables:** `PA0000` (actions), `PA0001` (org assignment), `PA0008` (basic pay), `PA0002` (personal data)
**Volume estimate:** ~15 GB
**RAG Approach:** Vector-only (no SQL-RAG — too sensitive for broad queries)

**CRITICAL RESTRICTIONS:**
- HR data gets its own isolated Qdrant collection: `hr_vectors`
- Access restricted to: HR team, line managers (own org only), C-suite
- Salary/compensation fields (`PA0008`): NEVER embedded; referenced by ID only
- All names → `[PERSON]`; employee IDs kept as reference keys

**Chunk Format (example):**
```
Employee EMP-10045 | Department: Engineering (ORG-ENG-001)
Position: Senior Developer | Grade: E5
Start Date: 2019-03-01 | Location: London (Plant 1000)
[NAME, SALARY, PERSONAL_DATA MASKED]
```

**Estimated vectors:** ~200K (small but highly restricted)

---

### 3.4 Supply Chain / Procurement

**SAP Tables:** `EKKO` (PO header), `EKPO` (PO items), `MARD` (stock overview), `MSEG` (goods movements)
**Volume estimate:** ~55 GB
**RAG Approach:** Hybrid

**Chunk Format (example):**
```
Purchase Order PO-4500012345 | Vendor: Supplier Y (V-20001) | Date: 2024-07-01
Plant: 1000 | Purchasing Org: 1000
Item 10: Material RAW-STEEL-001 "Cold-Rolled Steel Sheet"
         Qty: 10,000 KG @ EUR 1.20/KG = EUR 12,000.00
         Delivery Date: 2024-07-15 | Status: Goods Received
```

**Sensitivity:** MEDIUM (vendor pricing may be commercially sensitive)
**Estimated vectors:** ~6M

---

### 3.5 Material Master

**SAP Tables:** `MARA` (general data), `MAKT` (descriptions), `MARC` (plant data)
**Volume estimate:** ~10 GB
**RAG Approach:** Vector-only (entity-as-chunk — treat each material as one document)

**Chunk Format (example):**
```
Material: MAT-001234 | Type: FERT (Finished Product)
Description: "Industrial Motor Type A - 5kW Single Phase"
Unit: EA | Weight: 45 KG | Hazardous: No
Base UOM: EA | Sales UOM: EA | Purchase UOM: EA
Industry Sector: M (Mechanical Engineering)
```

**Sensitivity:** LOW — publicly available product information
**Estimated vectors:** ~500K

---

### 3.6 Email Archive

**Source:** SAP HANA email tables or IMAP/Exchange (TBC with Refined IT)
**Volume estimate:** ~50 GB
**RAG Approach:** Vector-only with semantic chunking

**Extraction Fields:**
```
message_id, thread_id, sender (masked), recipients (masked),
date, subject, body_text, attachment_count, attachment_names
```

**Chunk Format (example):**
```
Email | Date: 2024-08-12 | Thread: Project Phoenix Status
From: [PERSON] (Sales) | To: [PERSON] (Finance), [PERSON] (Operations)
Subject: Q3 Delivery Schedule — Customer Acme Corp

"Following up on our meeting last Tuesday regarding the revised delivery
timeline for order SO-2024-001234. The production team has confirmed..."
[Chunk 1/3 — continued]
```

**PII Handling:** Sender/recipient names → `[PERSON]`; emails → `[EMAIL:hash]`
**Estimated vectors:** ~2M (assuming ~50K emails, avg 3 chunks each + subject chunks)

---

### 3.7 Business Warehouse / Analytics (BW)

**SAP Tables:** InfoCubes, DSOs, CompositeProviders in BW/4HANA
**Volume estimate:** ~60 GB
**RAG Approach:** SQL-RAG only (pre-aggregated data; semantic search less useful)

**Strategy:** Instead of embedding BW content, generate **SQL query templates** for common KPI questions and execute against BW via the HANA SQL interface.

**Examples of SQL-RAG patterns:**
```sql
-- "What is Q3 revenue by sales region?" →
SELECT REGION, SUM(NET_REVENUE) as Q3_REVENUE
FROM BW_SALES_CUBE
WHERE FISCAL_PERIOD BETWEEN '20240701' AND '20240930'
GROUP BY REGION ORDER BY Q3_REVENUE DESC;
```

**Estimated vectors:** 0 (SQL-based, no embeddings)

---

## 4. RAG Approach Decision Matrix

```
                    ┌─────────────────────────────────────┐
                    │    Is the data primarily TEXT?      │
                    └─────────────────────────────────────┘
                           Yes │                │ No (Tabular)
                               ▼                ▼
                    ┌──────────────┐    ┌──────────────────────┐
                    │ Vector-Only  │    │ Does the user need    │
                    │    RAG       │    │ aggregations/KPIs?   │
                    │ (Emails,     │    └──────────────────────┘
                    │  Documents)  │         Yes │     │ No
                    └──────────────┘             ▼     ▼
                                       ┌──────────┐ ┌────────────┐
                                       │ SQL-RAG  │ │   Hybrid   │
                                       │  Only    │ │  (Vector   │
                                       │  (BW)    │ │   + SQL)   │
                                       └──────────┘ └────────────┘
```

---

## 5. Embedding Coverage Estimate

| Domain                    | Approach        | Estimated Vectors | Index Size |
|---------------------------|-----------------|-------------------|------------|
| Finance — GL              | Hybrid          | 8,000,000         | ~32 GB     |
| Finance — Controlling     | Hybrid          | 2,000,000         | ~8 GB      |
| Sales Orders + Billing    | Hybrid          | 5,000,000         | ~20 GB     |
| Customers                 | Vector          | 200,000           | ~0.8 GB    |
| Supply Chain              | Hybrid          | 6,000,000         | ~24 GB     |
| Inventory                 | Hybrid          | 3,000,000         | ~12 GB     |
| Material Master           | Vector          | 500,000           | ~2 GB      |
| HR (restricted)           | Vector          | 200,000           | ~0.8 GB    |
| HR Org Structure          | Vector          | 100,000           | ~0.4 GB    |
| BW Analytics              | SQL-RAG         | 0                 | 0 GB       |
| Email Archive             | Vector          | 2,000,000         | ~8 GB      |
| Documents / Attachments   | Vector          | 1,500,000         | ~6 GB      |
| **TOTAL**                 |                 | **~28.5M**        | **~114 GB** |

> **Vector size assumption:** 1024-dimensional float32 = 4 KB per vector; plus HNSW graph overhead (~4KB) = ~8 KB per vector total in Qdrant.
> At 28.5M vectors: ~228 GB Qdrant storage needed in production.

---

## 6. Pilot Scope (Phase 0)

### Pilot Dataset 1: Sales Orders + Invoices
- Tables: VBAK, VBAP, VBRK, VBRP
- Subset: 2024 data only (~5 GB)
- Expected vectors: ~250K
- Test queries:
  - "Show me all orders from Acme Corp in Q3 2024"
  - "What's the average order value for automotive customers?"
  - "Find invoices over EUR 100,000 that are overdue"
  - "Which sales rep has the highest revenue this year?"

### Pilot Dataset 2: Email Archive
- Source: 2024 emails only (~2 GB sample)
- Expected vectors: ~100K
- Test queries:
  - "Find emails discussing the Phoenix project"
  - "What did management say about budget cuts last quarter?"
  - "Show emails about supplier delivery issues in July"

---

## 7. Exclusions & Deferred Items

| Category                  | Reason for Exclusion/Deferral             | Revisit |
|---------------------------|-------------------------------------------|---------|
| HR Payroll (PA0008)       | Salary data — too sensitive for embedding | Phase 4 |
| BW/Analytics raw cubes    | SQL-RAG preferred; no embedding value     | Phase 3 |
| SAP Workflow tables        | Low query value; complex structure        | Phase 4 |
| Archive / historized data  | Assess business need before including    | Phase 3 |
| External vendor portals    | Out of HANA scope                         | Future  |
