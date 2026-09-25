# Lesson 1 - Basic RAG with Gemini and Pinecone

This lesson builds a complete Retrieval-Augmented Generation pipeline using one PDF and four tools:

- **Gemini embeddings** convert document passages and questions into vectors.
- **Pinecone** stores the vectors and performs semantic search.
- **Gemini 2.5 Flash** writes an answer from the retrieved passages.
- **LangChain and Python** connect the steps with a small amount of code.

The notebook is designed for beginners. Every intermediate result is visible: extracted text, chunk count, retrieved passages, and final answer.

## What problem does RAG solve?

A general language model can explain common topics, but it does not automatically know private or newly updated documents. When asked for a specific fact without evidence, it may produce a plausible but unsupported answer.

RAG changes the workflow:

```text
Question
  -> retrieve relevant passages from your documents
  -> place those passages inside the prompt
  -> ask the language model to answer from that evidence
```

Think of the language model as a smart student and retrieval as an open reference book. The student still writes the answer, but the facts come from the supplied material rather than memory alone.

## SQL search versus vector search

Traditional SQL queries are excellent when you know the exact row, identifier, filter, or calculation you need.

Vector search is useful when the question and document express the same idea with different words. A query about `resume preparation advice` can match a passage titled `Writing your resume` because the embedding represents meaning rather than only exact characters.

This notebook does not replace SQL. It demonstrates a different search problem: finding conceptually relevant text inside a long document.

## What is a vector?

An embedding model turns text into a list of numbers. In this notebook, each Gemini embedding contains 768 values.

Texts with similar meanings should occupy nearby locations in that mathematical space. Pinecone uses cosine similarity to identify the stored chunks closest to the question vector.

You do not need to interpret the individual numbers. Their useful property is distance: closer vectors usually represent more similar meaning.

## The source document

`ResumeBook.pdf` is a 54-page resume guide. Its chapters cover:

- why a strong resume matters
- the employer hiring process
- what employers examine and which red flags cause rejection
- finding suitable roles
- preparing a focused application
- cover letters and online professional presence
- writing a master resume
- standard resume sections
- quantifying accomplishments
- customizing a resume for each job
- resume examples and closing guidance

The PDF produces approximately 44,000 extracted characters in the supplied environment.

The document is intentionally much longer than a single prompt needs. That makes it useful for demonstrating retrieval: the system should locate a few relevant passages instead of sending all 54 pages to the model for every question.

## The complete pipeline

### Phase 1: ingestion

Ingestion prepares the knowledge base before users ask questions.

```text
ResumeBook.pdf
  -> extract text
  -> split into overlapping chunks
  -> embed every chunk with Gemini
  -> store vectors and text metadata in Pinecone
```

The supplied document becomes approximately 20 chunks using:

- chunk size: 400 words
- overlap: 40 words
- source ID: `resume-book`
- chunk IDs: `resume-book-0`, `resume-book-1`, and so on

The overlap repeats a small amount of text between neighboring chunks. This reduces the chance that an important explanation is lost because it crosses a chunk boundary.

### Phase 2: generation

Generation runs for each user question.

```text
User question
  -> Gemini query embedding
  -> Pinecone similarity search
  -> top three matching chunks
  -> grounded prompt
  -> Gemini answer
```

The retrieval function uses `Top-K = 3`, meaning Pinecone returns the three closest chunks. The notebook prints those chunks before generating the answer so students can inspect the evidence.

## What Pinecone stores

The serverless index is named:

```text
rag-demo-gemini
```

Its configuration is:

| Setting | Value | Why it matters |
|---|---:|---|
| Dimension | 768 | Must match the Gemini embedding length |
| Metric | Cosine | Measures similarity by vector direction |
| Cloud | AWS | Serverless hosting provider used in the example |
| Region | us-east-1 | Serverless index location |

Each Pinecone record contains:

```text
id       -> stable chunk identifier
values   -> 768-number embedding
metadata -> original chunk text and source name
```

Rerunning the ingestion cell uses the same IDs. Pinecone upserts those records, so the existing records are updated rather than duplicated.

## Grounding rules

The final prompt tells Gemini to:

- answer only from the retrieved context
- remain concise and clear
- say `I don't know` when the answer is absent
- include the source label

These instructions reduce unsupported answers, but a prompt alone is not a guarantee. A production system should also evaluate retrieval quality, citation correctness, and unsupported claims.

## Notebook walkthrough

### Cell 1: architecture and learning goals

Introduces the stack and shows how the user question moves through LangChain, Pinecone, and Gemini.

### Cell 2: prerequisites

Lists the required Google and Pinecone accounts, API keys, Python version, `.env` file, and PDF location.

### Cells 3-4: dependencies

Installs the packages listed in `requirements.txt`. Run this once per environment and restart the kernel if Jupyter requests it.

### Cells 5-6: API keys

Loads `.env` without printing either secret. If a value is missing, the notebook asks for it through a hidden prompt.

### Cell 7: RAG mental model

Explains why the system retrieves evidence before generation and distinguishes semantic vector search from exact database lookups.

### Cells 8-9: PDF extraction

Reads all 54 pages with PyPDF and combines their text into one source document.

The output presents a clean PDF summary containing the source name, document count, extracted character count, word count, and a short text preview.

The PDF contains a few older internal object references. PyPDF can still extract the text successfully; the notebook hides those harmless parser warnings to keep the lesson output readable.

### Cells 10-11: chunking

Splits the long document into overlapping 400-word chunks and assigns stable IDs.

The output presents a clean chunking summary containing the total chunk count, chunk size, overlap, first chunk ID, and a readable preview.

### Cells 12-13: embeddings and Pinecone

Creates the Pinecone index when necessary, generates one Gemini embedding per chunk, and upserts the vectors with their text metadata.

The cell displays embedding progress after every five chunks, followed by a vector-database summary. It shows the live Pinecone index name, whether the index already existed, the current embedding model, vector dimension, similarity metric, and number of vectors upserted.

### Cells 14-15: retriever and answer chain

Creates:

- `retrieve_docs(question, k=3)` to embed a question and retrieve the three closest chunks
- `ask_question(question)` to print the context, format the grounded prompt, and call Gemini

The final setup message makes the runtime sequence explicit: query, embed, retrieve three matches, ground the prompt, and generate.

### Cells 16-17: first question

Runs:

```text
Give me tips to prepare a resume
```

The notebook first prints three clearly separated retrieved passages from the resume guide. Each passage includes its real Pinecone similarity score, source, and chunk ID. Gemini's response then appears under a separate `Grounded answer` heading.

The exact wording may change between runs, but the answer should remain grounded in the displayed context.

### Cell 18: summary

Connects the five core stages: chunk, embed, store, retrieve, and generate. It also identifies production additions such as stronger chunking, page-level citations, evaluation, access control, and monitoring.

## Folder contents

```text
01-basic-rag/
|-- 01_rag_101_gemini_pinecone.ipynb
|-- ResumeBook.pdf
|-- README.md
|-- requirements.txt
|-- .env.example
`-- .env                 # private and ignored by Git
```

## Setup

Run the notebook from this folder so `ResumeBook.pdf` and `requirements.txt` resolve correctly.

```powershell
cd "01-basic-rag"
jupyter lab
```

Copy `.env.example` to `.env`, then add your keys:

```env
GOOGLE_API_KEY=your_google_api_key_here
PINECONE_API_KEY=your_pinecone_api_key_here
```

Open `01_rag_101_gemini_pinecone.ipynb` and run the cells from top to bottom.

Never place real API keys directly in the notebook and never commit `.env`.

## Questions to try

Questions that should be answerable from the resume guide:

- Give me tips to prepare a resume.
- How long should my resume be?
- What are common resume red flags?
- Which sections should a standard resume contain?
- Why should I customize my resume for each job?
- What should I include in the education section?
- Do I need a cover letter?

Try one question that the guide cannot answer, such as:

```text
What was Northstar Electronics' revenue last quarter?
```

The desired behavior is `I don't know` because the retrieved resume content does not contain that fact.

## What is intentionally simplified

- The entire PDF is treated as one source before word-based chunking.
- Chunks do not retain page numbers.
- The notebook uses one Pinecone index and no namespace.
- All vectors are generated in a simple loop.
- There is no batch ingestion, caching, evaluation suite, or user access control.
- The source citation identifies the book but not a page.
- The prompt performs basic grounding but does not independently verify every claim.

These choices keep the complete RAG loop understandable. Later lessons in this repository introduce stronger retrieval methods.

## Troubleshooting

### `ResumeBook.pdf` cannot be found

Start Jupyter from the `01-basic-rag` folder and confirm the PDF is beside the notebook.

### An API-key prompt appears

The corresponding value is missing or blank in `.env`. Check the filename and variable names, then restart the kernel.

### Pinecone reports a dimension mismatch

The existing `rag-demo-gemini` index was probably created with a different embedding dimension. Delete that test index in the Pinecone console or use a new index name, then rerun the ingestion cell.

### The first Pinecone operation takes time

A newly created serverless index may need a short initialization period. Wait briefly and rerun the ingestion cell if the service reports that the index is not ready.

### The answer is weak or unrelated

Inspect the printed context first. If the three retrieved chunks are unrelated, the problem is retrieval. If the chunks are relevant but the answer ignores them, the problem is generation or prompting.

### Rerunning does not create duplicate records

That is expected. Upsert replaces records with matching IDs. Change the IDs or use a namespace only when you intentionally want a separate dataset.

## Cost and cleanup

Gemini and Pinecone usage may consume free-tier allowances or create charges according to the current account plans.

When you finish experimenting, you can delete the `rag-demo-gemini` index from the Pinecone console to remove the stored vectors and stop any associated index usage.

## Source-material notice

The included resume guide was supplied for this lesson. Confirm that you have permission to redistribute third-party source material before publishing or reusing it outside the intended course repository.
