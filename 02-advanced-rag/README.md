# Lesson 2 - Advanced RAG 101

This lesson introduces a practical idea: **different questions need different retrieval engines**.

A normal RAG tutorial often stops after vector search. This notebook shows why that is not enough. Some questions require relationships, exact calculations, current web information, or a check that the retrieved evidence is still valid.

The notebook is designed for learners with limited coding confidence. The code is intentionally small, and every section focuses on:

1. What problem are we trying to solve?
2. Why is ordinary vector search insufficient?
3. What retrieval method fits the question?
4. What evidence reaches the language model?

## What you will build

The notebook combines six retrieval patterns:

| Method | What it retrieves | Example question |
|---|---|---|
| Vector RAG | Semantically similar policy passages | What is the damaged-product return window? |
| Graph retrieval | A path between connected entities | How is Helios Components connected to Project Atlas? |
| Text-to-SQL | Exact calculations from rows and columns | Which region grew the most from Q2 to Q3? |
| Exa Web RAG | Recent public web information | What recent events are affecting electronics supply chains? |
| Corrective RAG | Better evidence after detecting a weak result | What is the current warranty period? |
| Query routing | The retrieval engine that fits the question | Should this go to documents, graph, SQL, or the web? |

Vision RAG is explained conceptually but not implemented. It would require an additional page-image ingestion path and would make this introductory notebook much longer.

## The fictional business scenario

Northstar Electronics is preparing internal assistants for customer care, operations, finance, and project teams. Its information is spread across policies, project documents, incident reports, tables, and current external news.

The documents intentionally behave like a real company knowledge base:

- The same entities appear in several documents.
- Current and historical values coexist.
- Operational reports preserve old information for audit purposes.
- Some questions require calculations rather than prose retrieval.
- Recent external events do not exist in the internal documents.

Everything in the dataset is fictional. The documents were created specifically for this lesson.

## The five source documents

### 1. Customer Returns and Warranty Policy

`01_customer_returns_policy.pdf`

This is the controlled source for current customer policy. Version 3.2 became effective on 1 July 2026. It introduced:

- a 30-calendar-day return window for damaged or defective products
- a 24-month warranty for Northstar-branded devices

When the question asks for the **current** policy, this document should have greater authority than a historical report.

### 2. Vendor Governance Manual

`02_vendor_governance_manual.pdf`

This document explains supplier ownership, risk levels, escalation rules, and the role of Helios Components. It provides part of the relationship context used in the graph example.

### 3. Project Atlas Program Brief

`03_project_atlas_brief.pdf`

Project Atlas is a warehouse-automation pilot at the Bengaluru distribution center. Priya Nair leads the program. The project depends on Orion Control Modules, cabinet integration, vendor delivery, and safety validation.

### 4. FY2026 Q3 Operations Report

`04_q3_operations_report.pdf`

This report describes Q3 revenue, operational performance, the Helios component hold, and the resulting Project Atlas delay.

Page 7 contains a crucial historical note: an old dashboard displayed a 14-day damaged-product return window and a 12-month warranty. Those values came from policy version 2.8 and are no longer valid.

The report keeps the old values because audit records should preserve what happened. Historical evidence is useful, but it should not answer a current-policy question.

### 5. Incident and Risk Review Q3-17

`05_incident_and_risk_review.pdf`

This report explains how a production lot from Helios Components affected Orion Control Modules intended for cabinets assembled by Kestrel Logistics for Project Atlas. The incident delayed testing but did not reach live customer operations.

## Why Corrective RAG retries

The retry is intentional. It demonstrates a common production failure: retrieval finds a passage that is related to the question but unsuitable for answering it.

The document history is:

```text
Policy version 2.8
14-day damaged-product return window
12-month warranty
        |
        | superseded on 1 July 2026
        v
Policy version 3.2
30-day damaged-product return window
24-month warranty
```

The Q3 Operations Report retains the old 12-month value on page 7 to explain a dashboard error. The page also states that the value is obsolete and directs readers to policy version 3.2.

The Corrective RAG demonstration therefore follows this sequence:

1. Start with the deliberately outdated page from the operations report.
2. Ask an evaluator whether that evidence can answer the **current** warranty question.
3. The evaluator returns `RETRY` because the passage describes an obsolete value.
4. Retrieval runs again with a current-policy-focused query.
5. The controlled policy and supporting current references are retrieved.
6. The final answer states the 24-month warranty and cites its sources.

This is not a random retry, and it does not mean the first document is useless. The first document is valid historical evidence. It is simply the wrong authority for the current question.

## Notebook walkthrough

### Cells 1-3: lesson setup

The opening cells install the small dependency set, load the API keys, create the OpenAI and Exa clients, and locate the PDF folder.

Expected result:

```text
Ready: gpt-5-mini + text-embedding-3-small
```

### Cells 4-6: Vector RAG

The notebook extracts text from every PDF, keeps filename and page metadata, creates overlapping chunks, and embeds those chunks with OpenAI.

For the damaged-product question, vector similarity retrieves policy pages. The model receives only those passages and must cite the filename and page.

Expected conclusion:

```text
Damaged or defective products may be returned within 30 calendar days.
```

Why this method fits: the answer already exists as prose inside the policy.

### Cells 7-9: graph retrieval

The question asks for a chain of relationships rather than one similar paragraph. NetworkX stores entities as nodes and relationships as edges, then finds a path between Helios Components and Project Atlas.

Expected relationship:

```text
Helios Components -> Quality Incident Q3-17 -> Project Atlas
```

Important limitation: the relationships in this introductory example are manually curated. The path retrieval is real, but the notebook does not extract the graph automatically from the PDFs. A production Graph RAG system would also need entity extraction, entity resolution, relationship extraction, and a maintained graph store.

### Cells 10-12: Text-to-SQL

The notebook creates an in-memory SQLite table. No database file or server is required.

Each row represents one combination of:

```text
quarter + region + category + revenue
```

North appears six times because it has three product categories in Q2 and the same three categories in Q3. These are separate records, not duplicate rows.

The language model translates the question into a read-only SQL query. SQLite performs the calculation. A safety check rejects write operations and multiple statements.

Expected result:

```text
South | 550000
```

Why this method fits: SQL should perform exact aggregation. The language model writes the query but does not calculate the answer itself.

### Cells 13-14: Exa Web RAG

The internal PDFs cannot describe events that happened after they were written. Exa retrieves current web results and relevant highlights. OpenAI receives those highlights and must answer with source numbers and URLs.

The exact sources and answer will change over time. That variation is expected because this section performs live web search.

Why this method fits: the question asks about recent external events, so a static internal knowledge base is insufficient.

### Cells 15-16: Corrective RAG

This section performs the outdated-evidence experiment described earlier. It grades the first evidence, visibly prints `RETRY`, retrieves current policy evidence, and produces a cited answer.

Expected conclusion:

```text
The current warranty period is 24 months for qualifying Northstar-branded devices.
```

Why this method fits: similarity alone cannot decide whether a passage is current, authoritative, or complete.

### Cell 17: question router

The small router demonstrates the decision that happens before retrieval:

- policy wording goes to documents
- relationships go to the graph
- calculations go to SQL
- recent events go to the web

The router uses visible keywords to keep the lesson understandable. Production routers may use a classifier, a language model, rules, or a combination of these.

### Cell 18: decision matrix

The final table summarizes which retrieval engine fits each information shape. The central lesson is that Advanced RAG is not one replacement for vector search. It is a system that selects and checks the right evidence source.

## Repository contents

```text
02-advanced-rag/
|-- 01_advanced_rag_101.ipynb
|-- README.md
|-- .env.example
`-- output/
    `-- pdf/
        |-- 01_customer_returns_policy.pdf
        |-- 02_vendor_governance_manual.pdf
        |-- 03_project_atlas_brief.pdf
        |-- 04_q3_operations_report.pdf
        `-- 05_incident_and_risk_review.pdf
```

## Setup

Run the notebook from this folder so its relative PDF path resolves correctly.

```powershell
cd "02-advanced-rag"
```

Create a `.env` file in this folder or use the shared `.env` in the repository root:

```env
OPENAI_API_KEY=your_openai_api_key_here
EXA_API_KEY=your_exa_api_key_here
OPENAI_CHAT_MODEL=gpt-5-mini
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

Then open `01_advanced_rag_101.ipynb` and run the cells from top to bottom. Run the installation cell once. Restart the kernel if Jupyter requests it.

If a key is missing, the notebook asks for it through a hidden prompt.

## What is intentionally simplified

- Vectors stay in memory instead of a vector database.
- The graph is manually curated instead of extracted from documents.
- SQLite runs in memory instead of a production database.
- SQL safety uses a small allowlist and blocklist, not a database permission system.
- The router uses understandable keyword rules.
- Vision RAG is discussed but not implemented.
- Corrective RAG begins with a deliberately selected stale passage so the retry is visible and repeatable.

These choices keep the teaching environment small. They should not be copied unchanged into a production system.

## Expected costs and variability

- OpenAI API calls may incur usage charges.
- Exa usage may consume free credits or paid credits depending on the current plan.
- Web results and language-model wording can change between runs.
- The core facts stored in the fictional PDFs remain stable.

## Troubleshooting

### The notebook cannot find the PDFs

Confirm that the current working directory is `02-advanced-rag` and that `output/pdf/` contains five files.

### An API key prompt appears

The notebook did not find the key in `.env`. Check the filename, variable name, and notebook working directory.

### SQL output contains repeated region names

The table stores category-level rows. Each region appears once per category for each quarter. Use `GROUP BY quarter, region` when you want regional totals.

### Exa returns different articles

This is normal. Web RAG is intentionally live. Evaluate whether the returned evidence supports the answer instead of expecting identical URLs on every run.

## Questions to consider after completing the notebook

1. Which answers require fresh information rather than internal documents?
2. Which source should win when a historical report conflicts with a controlled policy?
3. Why should SQL calculate revenue instead of the language model?
4. What additional work would turn the curated graph into a complete Graph RAG pipeline?
5. Where would you add evaluation and access control before production use?
