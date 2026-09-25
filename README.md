# Week 2: Retrieval-Augmented Generation

This repository contains the beginner-friendly notebooks and practice documents
used in Week 2. The lessons move from a basic vector-search workflow to choosing
the right retrieval method and improving retrieval quality.

## The big picture

```text
Your documents
      |
      v
prepare and index the content
      |
      v
retrieve useful evidence ----> give that evidence to an AI model
                                      |
                                      v
                               grounded answer
```

The goal is not to memorize every line of code. Focus on **why** each retrieval
method exists, what kind of question it handles, and what evidence reaches the
model.

## Learning path

1. [`01-basic-rag/`](01-basic-rag/) - build a complete RAG workflow with a PDF,
   Gemini embeddings, Pinecone, and LangChain.
2. [`02-advanced-rag/`](02-advanced-rag/) - compare vector retrieval, graph
   retrieval, SQL, web search, Corrective RAG, and simple routing.
3. [`03-advanced-retrieval/`](03-advanced-retrieval/) - improve document search
   with structured chunking, metadata, parent-child retrieval, hybrid search,
   reranking, query expansion, and HyDE.

Each folder contains one numbered notebook, its sample documents, an API-key
template, and a detailed guide.

## Easiest way to run the notebooks

Open a lesson folder, read its `README.md`, and run the notebook from top to
bottom in Jupyter or VS Code. You can also upload the notebook and its sample
files to Google Colab.

Run notebooks from inside their lesson folder so relative paths to PDFs and
other files continue to work.

## Run locally with the shared virtual environment

The repository includes one shared environment for all three lessons. From the
repository root:

```bash
source .venv/bin/activate
jupyter lab
```

In Jupyter or VS Code, select the kernel named **Python (Week 2 RAG)**. Then
open a notebook and run it from its lesson folder so the relative data paths
resolve correctly.

To recreate the environment later:

```bash
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m ipykernel install --user --name week-2-rag --display-name "Python (Week 2 RAG)"
```

## API keys

The lessons use different services:

- Lesson 1: Google AI and Pinecone
- Lessons 2 and 3: OpenAI
- Lesson 2 web-search section: Exa

Copy the lesson's `.env.example` to `.env` and replace the placeholders with
your own keys. API usage may incur charges.

Never commit API keys, passwords, `.env` files, or other secrets.

## Important limitations

These notebooks are teaching examples, not production systems. They use small
datasets, simplified routing and safety checks, and in-memory components where
possible. The companies, products, people, incidents, policies, and figures in
the sample documents are fictional training material.
