# Unit 4 Theory: RAG + Embeddings + Vector Databases

Read this before starting the unit tasks. Unit 4 introduces genuinely new concepts.

---

## The Problem RAG Solves

**LLMs have a knowledge cutoff.** They were trained on data up to a certain date and know nothing about your runbooks, your systems, or your incident history.

**RAG (Retrieval-Augmented Generation)** = give the LLM the relevant information it needs at the moment you ask the question.

```
Without RAG:
  Question --> LLM --> Answer (based only on training data, knows nothing about your systems)

With RAG:
  Question --> Search your docs --> Find relevant chunks --> Give chunks + question to LLM --> Answer with your context
```

The LLM does not need to "learn" your runbooks. It just needs them included in the prompt when answering.

---

## What is an Embedding?

**Embedding** = a list of numbers that represents the *meaning* of a piece of text

```
"Disk is full"              -> [0.23, -0.11, 0.87, 0.04, ...]   (384 numbers)
"Storage capacity exceeded" -> [0.21, -0.09, 0.85, 0.06, ...]   (similar numbers = similar meaning)
"The cat sat on the mat"    -> [-0.45, 0.33, -0.12, 0.71, ...]  (very different numbers)
```

**Key point:** Similar meanings produce similar numbers. Unrelated meanings produce very different numbers.

An **embedding model** is what converts text into these number arrays. In Unit 4 you have two options:
- `all-MiniLM-L6-v2` via `sentence-transformers` (Python library) — produces 384 numbers
- `nomic-embed-text` via Ollama — produces 768 numbers, keeps your whole stack on Ollama

---

## What is a Vector Database?

**Vector database** = a database built to store and search embeddings

You use **Qdrant**. It stores each document chunk as:
- A **vector** (the embedding numbers)
- A **payload** (metadata: filename, section title, the original text)

**Similarity search** — how Qdrant finds relevant content:
```
Your question --> embedding --> [0.22, -0.10, 0.86, ...]
                                          |
                      Qdrant finds vectors with similar numbers
                                          |
                      Returns the top 3 most similar document chunks
```

---

## Semantic Search vs Keyword Search

| | Keyword Search | Semantic Search |
|---|---|---|
| How it works | Looks for exact word matches | Compares meaning via embeddings |
| Query: "disk full" finds "storage capacity exceeded"? | No (different words) | Yes (same meaning) |
| Query: "service crash" finds "process terminated unexpectedly"? | No | Yes |

Semantic search is much more useful for runbook retrieval because engineers describe the same problem in many different ways.

---

## What is Chunking?

**Chunking** = splitting a large document into smaller pieces before embedding

**Why?** An embedding works best on a short, focused piece of text. A 20-page runbook embedded as a single vector loses all specificity — you can't meaningfully compare it to a specific question.

**Strategy in Unit 4:** Split by H2 headings. Each section becomes one chunk with its own embedding.

```
01-incident-response.md
├── ## Initial Triage          -> chunk 1, embedded separately
├── ## Escalation Criteria     -> chunk 2, embedded separately
└── ## Post-Incident Review    -> chunk 3, embedded separately
```

---

## The Full RAG Flow

```
INGEST (run once, or when docs change):

  Markdown files
      | read and split by heading
      v
  Chunks (text + metadata)
      | embed each chunk
      v
  Vectors stored in Qdrant


QUERY (run each time a question is asked):

  Question
      | embed the question
      v
  Qdrant similarity search
      | returns top 3 matching chunks
      v
  Build prompt:
    "Answer this: [question]
     Using only this context: [chunk1] [chunk2] [chunk3]
     Cite your sources."
      |
      v
  Ollama generates answer
      |
      v
  Return answer + citations (filename + section)
```

---

## What are Citations?

A citation tells you *which document and section* the answer came from:

```json
"citations": [
  { "file": "01-incident-response.md", "section": "Initial Triage" },
  { "file": "02-database-troubleshooting.md", "section": "Connection Errors" }
]
```

This is important for trust — you can verify the answer by reading the source. It also prevents the LLM from making things up (hallucinating), because you can check.

---

## Why This Matters for AI-102

The patterns you build locally map directly to Azure's managed services:

| This unit (local) | Azure equivalent |
|-------------------|-----------------|
| Qdrant | Azure AI Search |
| sentence-transformers / nomic-embed-text | Azure OpenAI Embeddings |
| Manual RAG pipeline | Azure AI Search + RAG pattern |
| Citations | Grounding / responsible AI in Azure |

When you reach AI-102 study material, you will already understand *why* these services exist and what problem they solve.
