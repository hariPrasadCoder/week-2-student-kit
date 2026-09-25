# Lesson 3 - Advanced Retrieval Methodologies

This lesson improves a retrieval pipeline before asking the language model to answer.

Lesson 2 compared different retrieval engines. Lesson 3 goes deeper into the document-retrieval path itself. It asks:

> Once we decide to search documents, how do we retrieve better evidence?

The notebook builds one pipeline layer by layer:

```text
PDFs
  -> structure-aware parent sections
  -> small enriched child chunks
  -> vector search + BM25
  -> Reciprocal Rank Fusion
  -> cross-encoder reranking
  -> full parent context
  -> grounded answer with citations
```

The implementation is intentionally small. No vector database, search server, orchestration framework, or additional reranking API is required.

## Learning objectives

After completing the notebook, you should be able to explain:

1. Why arbitrary chunk boundaries damage retrieval quality.
2. Why every chunk needs source and section metadata.
3. Why the best unit for searching may differ from the best unit for answering.
4. Why semantic and keyword retrieval solve different problems.
5. Why a fast first-stage ranking benefits from a slower second-stage reranker.
6. How query expansion and HyDE help when user language differs from document language.
7. Why every additional layer should solve a measured retrieval failure.

## The fictional support scenario

Northstar technicians support an industrial device called the **Northstar Edge Controller**. Information is distributed across a product manual, firmware release notes, and a field-service handbook.

The corpus is designed to create realistic retrieval problems:

- `NX-417` and `NX-471` look similar but require different procedures.
- Several documents mention `NX-417`, so exact matching alone is not enough.
- A useful sentence such as "Wait 90 seconds" becomes unsafe when separated from its fault code, connector, and verification steps.
- Operators describe a symptom as "the screen keeps going dark after the update," while the release notes call it a "display wake event" problem identified as `UI-204`.
- A small matching passage may identify the topic but omit the complete procedure.

All companies, products, codes, procedures, and results are fictional and exist only for this lesson.

## The three source documents

Each PDF has one cover page and three clearly labelled sections. The explicit `SECTION` headings allow students to see structure-aware parsing without first learning a complex layout-analysis library.

### 1. Product Support Manual

`01_product_support_manual.pdf`

This is the product-specific technical reference.

#### NX-417 Sensor Link Loss

`NX-417` means the controller has stopped receiving a valid heartbeat from the primary position sensor for more than 1.5 seconds.

The controlled recovery includes:

- stop the line and apply the approved lockout procedure
- inspect the primary sensor and cable
- disconnect the sensor at port `J7`
- wait 90 seconds before reconnecting the harness
- confirm a stable heartbeat for 30 seconds
- run a reduced-speed verification cycle

The incident must be escalated if it occurs twice in the same shift or if sensor voltage remains below 22.8 V.

#### NX-471 Motor Feedback Mismatch

`NX-471` concerns disagreement between commanded movement and encoder feedback. It happens after motion begins and points toward the encoder, coupling, or mechanical load.

This section deliberately acts as a distractor. It shares words such as motor, feedback, sensor, and fault with the NX-417 section, but its procedure is different.

#### Display and Power Behavior

This section distinguishes normal display sleep, a display-only software problem, and full controller power loss. It directs readers to the firmware release notes when firmware 4.8.0 is installed.

### 2. Firmware Release Notes

`02_firmware_release_notes.pdf`

This document contains the history behind the dark-screen question.

#### Version 4.8.0 and UI-204

Firmware 4.8.0 introduced a known issue called `UI-204`. After automatic display sleep, some displays fail to process the first wake event. The screen remains dark even though:

- the controller stays online
- the cyan status ring remains illuminated
- remote monitoring continues
- production logic is unaffected

The temporary workaround is to hold the wake key for three seconds or restart only the display module.

#### Version 4.8.2

Firmware 4.8.2 contains the permanent correction. It queues a second wake event when the display leaves its lowest-power state.

#### Version 4.8.3

Firmware 4.8.3 adds better sensor-heartbeat telemetry. It does not change the NX-417 recovery procedure. This creates useful overlap for the reranking demonstration.

### 3. Field Service Handbook

`03_field_service_handbook.pdf`

This document explains how technicians apply product procedures safely at customer sites. It adds lockout, evidence collection, escalation, and offline-operation requirements.

Its NX-417 section is especially important for hierarchical retrieval: a small child chunk can locate the 90-second instruction, but the complete parent section supplies the safety checks and escalation conditions.

## The three retrieval failures demonstrated

### Failure 1: arbitrary chunk boundaries

A fixed-size splitter counts characters or tokens. It does not understand titles, headings, procedures, or page transitions.

The notebook deliberately shows a 500-character boundary cutting between the end of the document overview and the start of the NX-417 section. The resulting pieces are valid strings but poor information units.

The structural parser instead creates nine parent sections from the nine visible `SECTION` headings.

### Failure 2: orphan chunks

Consider this sentence:

```text
Wait 90 seconds before reconnecting the sensor harness.
```

By itself, the sentence does not identify:

- the NX-417 fault
- the J7 connector
- the lockout requirement
- the reason for the delay
- the verification steps

The notebook enriches every child with its document name and section heading before indexing it.

### Failure 3: one ranking signal is insufficient

Vector search understands concepts, so it can connect sensor failure, heartbeat loss, and recovery language. It can also rank the semantically similar NX-471 section.

BM25 protects exact strings such as `NX-417`, `UI-204`, `J7`, and firmware versions. It does not understand meaning or paraphrases as well as vectors.

The notebook combines both rankings with Reciprocal Rank Fusion instead of comparing their unrelated raw scores.

## Notebook walkthrough

### Cells 1-3: setup

These cells install the dependencies, load the OpenAI key, select the models, and locate the three PDFs.

The local cross-encoder model downloads the first time the reranking section runs. Later runs use the cached model.

Expected result:

```text
Ready: 3 PDFs | gpt-5-mini | text-embedding-3-small
```

### Cells 4-5: structural parsing

The parser reads each PDF page and looks for the visible `SECTION` heading. Cover pages are skipped. Each section becomes a parent record containing:

- a stable parent ID
- filename
- page number
- section heading
- complete section text

Expected result:

```text
Created 9 parent sections from 3 PDFs
```

Why it matters: later steps can retrieve a precise passage and still recover the full section that produced it.

### Cells 6-7: child chunks and context enrichment

Each parent is split into approximately 70-word children with a 15-word overlap. The searchable representation adds:

```text
Document: <filename>
Section: <section heading>
<child passage>
```

The original passage remains available separately. Metadata enrichment changes what is indexed without rewriting the source document.

Expected result:

```text
9 parents -> 28 searchable children
```

The cell also prints the blind 500-character cut so students can compare arbitrary splitting with structural parsing.

### Cells 8-9: parent-child retrieval

The notebook locates the child containing the 90-second instruction and follows its `parent_id` back to the full NX-417 section.

The child is the **search target**. The parent is the **answer context**.

Why it matters: small passages improve retrieval precision, while full sections reduce incomplete or unsafe answers.

### Cells 10-12: hybrid search and RRF

OpenAI embeddings create the dense vector index. BM25 creates a keyword index from tokenized child text. A small stop-word list prevents common question words from dominating keyword ranking.

The demonstration query is simply:

```text
NX-417
```

This mirrors a real support user pasting an error code without writing a complete question.

Expected behavior:

- Vector search retrieves NX-417 but may also surface the semantically similar NX-471 section.
- BM25 strongly favors passages containing the exact NX-417 identifier.
- RRF combines both lists and keeps NX-417 at the top.

The RRF contribution for a result is calculated as:

```text
1 / (60 + rank)
```

The exact raw vector and BM25 scores never need to be compared.

### Cells 13-14: cross-encoder reranking

The first retrieval stage keeps several candidates because fast retrieval may rank related passages imperfectly.

The local `cross-encoder/ms-marco-MiniLM-L-6-v2` model then reads the recovery question and each candidate together. Unlike independent embeddings, the cross-encoder directly judges the query-passage pair.

After reranking, the winning child passages are replaced with their full parent sections.

The displayed reranker scores are useful for ordering candidates inside this run. They should not be treated as calibrated probabilities.

### Cells 15-16: query expansion and HyDE

The user asks:

```text
Why does my screen keep going dark after the update?
```

The documents use more specific language such as firmware, display wake event, UI-204, backlight, and automatic sleep.

Query expansion asks OpenAI to create three more explicit searches for this document collection.

HyDE stands for **Hypothetical Document Embeddings**. It generates a short passage that resembles the kind of documentation that might answer the question. The system searches using that passage's embedding.

The HyDE passage is not evidence and must never be cited as fact. It is only a search representation. The final answer must come from retrieved source documents.

### Cells 17-18: complete pipeline

The final pipeline performs:

1. Query expansion.
2. HyDE passage generation.
3. Dense and BM25 retrieval for the original and expanded searches.
4. Dense retrieval using the HyDE embedding.
5. RRF across all candidate lists.
6. Cross-encoder reranking.
7. Child-to-parent replacement.
8. Answer generation from the selected parent sections.

Expected selected evidence:

```text
02_firmware_release_notes.pdf
SECTION 1 - VERSION 4.8.0 KNOWN ISSUE UI-204
SECTION 2 - VERSION 4.8.2 DISPLAY FIX
```

Expected conclusion:

```text
Firmware 4.8.0 can lose the first wake event after automatic display sleep.
Firmware 4.8.2 provides the permanent correction.
```

The temporary workaround and citations should also appear in the answer.

### Cell 19: takeaway

The final table maps each retrieval layer to the failure it addresses. The lesson does not argue that every system needs every layer. Add complexity only after an evaluation identifies a specific retrieval weakness.

## Repository contents

```text
03-advanced-retrieval/
|-- 01_advanced_retrieval_methodologies.ipynb
|-- README.md
|-- .env.example
`-- data/
    |-- 01_product_support_manual.pdf
    |-- 02_firmware_release_notes.pdf
    `-- 03_field_service_handbook.pdf
```

## Setup

Run the notebook from this folder so the relative `data/` path resolves correctly.

```powershell
cd "03-advanced-retrieval"
```

Create a `.env` file in this folder or use the shared `.env` in the repository root:

```env
OPENAI_API_KEY=your_openai_api_key_here
OPENAI_CHAT_MODEL=gpt-5-mini
OPENAI_EMBEDDING_MODEL=text-embedding-3-small
```

Open `01_advanced_retrieval_methodologies.ipynb` and run the cells from top to bottom. Run the installation cell once and restart the kernel if Jupyter requests it.

## What is intentionally simplified

- PDF headings follow a controlled `SECTION` pattern.
- Real-world PDFs may require layout parsing, OCR, table extraction, or manual rules.
- All vectors and BM25 indexes remain in memory.
- The stop-word list is intentionally small.
- RRF uses a common fixed constant instead of tuned evaluation parameters.
- The reranker is a small general-purpose local model.
- Query transformations use prompts rather than a separately evaluated query model.
- The notebook has no access control, caching layer, monitoring, or persistent index.

These simplifications make each retrieval decision visible. A production system should select components based on evaluation data, scale, latency, security, and cost.

## Expected costs and variability

- OpenAI embedding and response calls may incur usage charges.
- The cross-encoder model downloads from Hugging Face on first use and requires internet access for that first download.
- Query expansions, HyDE text, and final wording may vary between runs.
- Retrieved source sections and core conclusions should remain stable because the PDF corpus is fixed.

## Troubleshooting

### The notebook reports zero PDFs

Confirm that the current working directory is `03-advanced-retrieval` and that `data/` contains three PDFs.

### The reranker takes longer on its first run

The cross-encoder model is being downloaded and cached. This normally happens only once per environment.

### Query expansion produces different wording

This is expected. Inspect whether the new queries preserve the user's intent instead of expecting identical sentences.

### The HyDE passage contains an unsupported claim

HyDE is hypothetical by design. It should influence retrieval only. The answer-generation step must cite retrieved PDF sections, not the HyDE passage.

### Vector and BM25 results disagree

That disagreement is the reason for hybrid search. Inspect whether vector search found semantic similarity and whether BM25 protected the exact identifier.

## Questions to consider after completing the notebook

1. Which section boundary would you use if a heading spans several PDF pages?
2. When should a table or procedure stay intact instead of becoming several children?
3. How would you measure whether query expansion improves recall?
4. What latency does the cross-encoder add, and is the relevance gain worth it?
5. Which retrieved unit should receive access-control checks: child, parent, or both?
