---
layout: post
title: "Building an AI PDF Knowledge Assistant with LLMs, RAG and Hybrid Search"
date: 2026-09-26
categories: [Generative AI, LLM, RAG]
tags: [LLM, RAG, Generative AI, Python, FAISS, FastAPI, Streamlit, NLP]
---

# Building an AI PDF Knowledge Assistant with LLMs, RAG and Hybrid Search

What if I could take **any collection of PDF documents, turn them into a private knowledge base, and then simply chat with those documents?**

That was the idea behind my latest personal project: **AI PDF Knowledge Assistant**.

Instead of building another generic chatbot that simply sends a question to an LLM, I wanted to build something where the **documents themselves become the knowledge source**.

The result is an end-to-end application that combines:

- Large Language Models
- Retrieval-Augmented Generation (RAG)
- Semantic search
- BM25 keyword search
- FAISS vector indexing
- Sentence Transformer embeddings
- Conversational memory
- Source/page citations
- FastAPI
- Streamlit
- Docker

The interesting part is that the chatbot can be customized simply by changing the PDFs.

---

## 💡 The Problem I Wanted to Solve

Large Language Models are excellent at understanding natural language, but a general LLM does not automatically know the contents of my private documents.

Suppose I have:

```text
📄 Vehicle Manual.pdf
📄 Diagnostic Guide.pdf
📄 Troubleshooting Guide.pdf
```

Instead of opening each document and searching manually, I wanted to ask:

> **"What are the possible causes of an engine misfire?"**

or:

> **"Which page describes the diagnostic procedure?"**

or:

> **"What does this document recommend as the first troubleshooting step?"**

The chatbot should search the uploaded documents, identify the relevant sections, and then use an LLM to generate an answer **grounded in those documents**.

That is where RAG becomes useful.

---

# 🧠 What is RAG?

RAG stands for **Retrieval-Augmented Generation**.

The basic idea is:

```text
User Question
      ↓
Retrieve Relevant Information
      ↓
Give Retrieved Information to LLM
      ↓
Generate Grounded Answer
```

Instead of expecting the LLM to know everything, I give it the relevant pieces of information from my own knowledge base.

For this project, the knowledge base is created dynamically from uploaded PDFs.

---

# 🏗️ High-Level Architecture

The complete system looks like this:

```text
                         ┌─────────────────┐
                         │      User       │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │   Streamlit UI  │
                         └────────┬────────┘
                                  │
                                  ▼
                         ┌─────────────────┐
                         │    FastAPI      │
                         │     Backend     │
                         └────────┬────────┘
                                  │
                 ┌────────────────┴────────────────┐
                 │                                 │
                 ▼                                 ▼
          PDF Ingestion                       User Question
                 │                                 │
                 ▼                                 ▼
          Text Extraction                   Semantic Search
                 │                                 │
                 ▼                                 │
             Chunking                              │
                 │                                 │
          ┌──────┴──────┐                          │
          ▼             ▼                          │
      Embeddings       BM25                        │
          │             │                          │
          ▼             ▼                          │
        FAISS      Keyword Search                  │
          │             │                          │
          └──────┬──────┘                          │
                 │                                 │
                 └────────── Hybrid Retrieval ─────┘
                                  │
                                  ▼
                         Relevant Context
                                  │
                                  ▼
                       Conversation History
                                  │
                                  ▼
                              LLM API
                                  │
                                  ▼
                    Grounded Answer + Sources
```

The system therefore has two major pipelines:

### Document pipeline

```text
PDF
 ↓
Text Extraction
 ↓
Chunking
 ↓
Embeddings + BM25
 ↓
FAISS + BM25 Index
```

### Question pipeline

```text
Question
 ↓
Semantic + Keyword Retrieval
 ↓
Hybrid Ranking
 ↓
Relevant Context
 ↓
LLM
 ↓
Answer + Source References
```

---

# 📄 Step 1 — Turning PDFs into a Knowledge Base

The first challenge was converting unstructured PDF documents into information that a retrieval system could search.

I used **PyMuPDF** for PDF text extraction.

One important design decision was to preserve the page number.

For every extracted page, I store metadata similar to:

```python
{
    "text": "...",
    "page": 5,
    "source": "vehicle_manual.pdf"
}
```

This becomes important later because the chatbot can tell the user **where the answer came from**.

So instead of only returning:

> The diagnostic procedure starts with checking the ignition system.

the chatbot can also provide:

```text
Source: vehicle_manual.pdf
Page: 42
```

---

# ✂️ Step 2 — Chunking the Documents

A complete PDF page or document can be too large to send directly to the LLM.

So the extracted text is divided into smaller overlapping chunks.

The current project uses:

```text
Chunk size   = 800 characters
Overlap      = 120 characters
```

Conceptually:

```text
Document
│
├── Chunk 1 ──────────────┐
│                         │
├── Chunk 2 ──────────────┤ ← overlap
│                         │
├── Chunk 3 ──────────────┤
│                         │
└── Chunk 4 ──────────────┘
```

The overlap helps prevent useful context from being lost when a sentence or concept falls across a chunk boundary.

Each chunk continues to carry its document and page metadata.

---

# 🔢 Step 3 — Creating Semantic Embeddings

Keyword matching alone isn't enough.

Consider these two sentences:

```text
"What causes an engine misfire?"
```

and:

```text
"Common reasons for cylinder combustion failure include
ignition, fuel delivery and air-intake problems."
```

They don't share many exact words, but they have similar meaning.

To capture this semantic relationship, I use **Sentence Transformers**.

The configured embedding model is:

```text
sentence-transformers/all-MiniLM-L6-v2
```

The process is:

```text
Text Chunk
    ↓
Sentence Transformer
    ↓
Numerical Vector
```

These vectors allow semantically similar questions and document sections to be matched.

---

# ⚡ Step 4 — FAISS Vector Search

The generated embeddings are stored in a **FAISS** index.

FAISS allows the application to efficiently search for vectors that are most similar to the user's question.

When a user asks:

> "What are the causes of an engine misfire?"

the question is converted into an embedding and searched against the document embeddings.

```text
User Question
      ↓
Question Embedding
      ↓
FAISS Similarity Search
      ↓
Relevant Chunks
```

The embeddings are normalized and an inner-product index is used for similarity search.

---

# 🔎 Step 5 — Why I Added BM25

This was one of the important design choices in the project.

Semantic search is powerful, but technical documents often contain exact terms such as:

```text
P0301
ECU
ABS
CAN
BMS
ISO 26262
```

If a user searches for:

> "What does P0301 mean?"

exact keyword matching can be extremely useful.

So I added **BM25 keyword retrieval** alongside semantic retrieval.

The system now has two ways of finding relevant information:

```text
                User Question
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
   Semantic Search          BM25 Search
       FAISS                 Keywords
          │                     │
          └──────────┬──────────┘
                     ▼
              Hybrid Ranking
```

---

# 🔀 Step 6 — Hybrid Retrieval

Rather than choosing between semantic search and keyword search, I combine both.

The current hybrid scoring strategy is:

```text
Hybrid Score =
    0.65 × Semantic Score
  + 0.35 × BM25 Score
```

The system first retrieves a larger candidate set and then combines the two signals before selecting the final top results.

This is useful because:

- semantic search captures **meaning**
- BM25 captures **exact terminology**
- the combination works well for domain-specific documents

The final retrieved chunks are then supplied to the LLM.

---

# 🤖 Step 7 — Connecting the LLM

Once the relevant context is retrieved, the LLM receives:

```text
Conversation History
        +
Retrieved Document Context
        +
Current Question
```

The project uses an OpenAI-compatible chat API.

The system prompt explicitly instructs the model to:

- use the supplied document context
- use conversation history for follow-up questions
- avoid inventing information
- avoid inventing citations or page numbers
- say when the documents do not contain enough evidence
- cite the source and page when appropriate

The temperature is intentionally kept low:

```text
temperature = 0.1
```

because this application is focused on **grounded question answering**, not creative text generation.

---

# 💬 Step 8 — Conversational Memory

I also wanted this to behave like a real chatbot rather than a collection of independent questions.

For example:

**User:**

> What does P0301 mean?

**Assistant:**

> P0301 indicates a cylinder 1 misfire.

Then:

**User:**

> What should I check first?

The second question depends on the context of the first question.

The application therefore maintains recent conversation history and sends it along with the retrieved document context.

The number of retained turns can be configured through:

```text
MAX_HISTORY_TURNS
```

---

# 📚 Step 9 — Source and Page Citations

One of the most important features is **traceability**.

A chatbot answer is much more useful when the user can verify it.

The retrieval pipeline keeps:

```text
Source document
Page number
Retrieval score
```

The UI displays the retrieved sources underneath the answer.

For example:

```text
Retrieved Sources

• vehicle_manual.pdf — page 42 — score 0.812
• diagnostic_guide.pdf — page 18 — score 0.734
```

This is especially useful for technical, research, and enterprise knowledge bases.

---

# 🖥️ Step 10 — Building the Chat Interface

The frontend is implemented using **Streamlit**.

The interface provides:

```text
┌────────────────────────────────────────────┐
│       📚 AI PDF Knowledge Assistant        │
├────────────────────────────────────────────┤
│                                            │
│  Upload PDFs                               │
│  [ Select files ]                          │
│                                            │
│  [ Index documents ]                       │
│                                            │
│  ────────────────────────────────────────  │
│                                            │
│  User: What is this document about?        │
│                                            │
│  AI: The document describes...             │
│                                            │
│  Retrieved Sources                         │
│  • document.pdf — page 4                   │
│                                            │
│  Ask a question...                         │
└────────────────────────────────────────────┘
```

The user doesn't need to know anything about embeddings, FAISS, BM25 or APIs.

They simply:

```text
Upload → Index → Ask → Get grounded answer
```

---

# ⚙️ Step 11 — FastAPI Backend

The application separates the user interface from the backend.

FastAPI provides the API layer.

The main endpoints are:

### Health check

```text
GET /health
```

### Document indexing

```text
POST /documents/index
```

### Question answering

```text
POST /chat
```

For example:

```json
{
  "question": "What is the main purpose of this document?",
  "history": []
}
```

The response contains the generated answer and the retrieved sources.

This separation makes the application easier to extend later with another frontend such as React.

---

# 🧩 Project Structure

```text
ai-pdf-knowledge-assistant/
│
├── app/
│   ├── api/
│   │   └── routes.py
│   │
│   ├── core/
│   │   └── config.py
│   │
│   ├── rag/
│   │   ├── chunking.py
│   │   ├── embeddings.py
│   │   ├── index.py
│   │   └── pipeline.py
│   │
│   ├── services/
│   │   ├── pdf_service.py
│   │   └── llm_service.py
│   │
│   └── main.py
│
├── frontend/
│   └── streamlit_app.py
│
├── tests/
│
├── data/
├── docs/
│
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env.example
└── README.md
```

I deliberately separated the project into:

```text
Frontend
Backend
RAG
Services
Configuration
Tests
Documentation
```

rather than keeping the entire chatbot inside one Python file.

---

# 🐳 Step 12 — Dockerizing the Application

The project also includes Docker support.

The application can be started with:

```bash
docker compose up --build
```

The architecture becomes:

```text
              Docker Compose
                    │
          ┌─────────┴─────────┐
          │                   │
          ▼                   ▼
      FastAPI              Streamlit
       :8000                 :8501
          │
          ▼
       RAG Pipeline
```

This makes the project easier to run consistently across environments.

---

# ☁️ Deployment Architecture

The current repository is designed primarily as a **portfolio/development deployment**.

For a public deployment, the application can be split into:

```text
                 User
                  │
                  ▼
          Streamlit Frontend
                  │
                  ▼
             FastAPI API
                  │
        ┌─────────┼─────────┐
        ▼         ▼         ▼
    PDF Storage  Vector    LLM API
                 Store
```

For example, the local FAISS index can eventually be replaced with a persistent vector database such as:

```text
Qdrant
pgvector
Pinecone
```

and uploaded PDFs can be stored in object storage.

This is important because local files on many cloud platforms are not guaranteed to survive redeployments.

---

# 🎯 The Most Important Part: Customization

The project is not tied to one particular domain.

The **PDF collection becomes the knowledge layer**.

For example, if I upload automotive documentation:

```text
Vehicle Manual
Diagnostic Manual
DTC Reference
Repair Guide
```

the chatbot becomes an:

> 🚗 **Automotive Knowledge Assistant**

If I replace those documents with research papers:

```text
Paper 1
Paper 2
Paper 3
```

the same application becomes:

> 🔬 **Research Paper Assistant**

If I upload company policies:

```text
Leave Policy
HR Handbook
Benefits Guide
```

it becomes:

> 🏢 **HR Policy Assistant**

No fundamental change to the RAG architecture is required.

---

# 🧪 Testing and Evaluation

Building a chatbot is only one part of an LLM project.

I also want to evaluate whether the retrieval system is actually finding the right information.

The repository contains basic tests, and the next evaluation layer can measure:

```text
Recall@K
MRR
Context Precision
Context Recall
Answer Faithfulness
Answer Relevance
```

A useful evaluation dataset could contain:

```text
Question
Expected Answer
Relevant Document
Relevant Page
```

For example:

```text
Question:
What are the main causes of P0301?

Expected source:
diagnostic_guide.pdf

Expected page:
42
```

This allows the retrieval system and generation system to be evaluated separately.

---

# 🔐 Security Considerations

Because this application accepts uploaded documents, security matters.

For a public production deployment, I would add:

- Authentication
- File-size limits
- File-type validation
- Rate limiting
- Secure document storage
- Malware scanning
- HTTPS
- User-level document isolation
- API key protection

And most importantly:

> **Confidential company or customer documents should never be uploaded to a personal deployment without authorization.**

For the portfolio version, I would use public, self-created, or appropriately licensed documents.

---

# 🚧 Current Limitations

This project is intentionally a practical portfolio implementation rather than a full enterprise document platform.

Current limitations include:

- Local FAISS persistence
- Local document storage
- No authentication
- No multi-user isolation
- No OCR pipeline for image-only PDFs
- No cross-encoder reranking yet
- No streaming token responses
- Basic retrieval evaluation
- Session-based conversation memory

These limitations also define the next stage of the project.

---

# 🔮 Future Improvements

The next version could introduce:

### 1. OCR support

Support scanned/image-only PDFs.

### 2. Reranking

Add a cross-encoder reranker after initial retrieval.

```text
FAISS + BM25
      ↓
Candidate Documents
      ↓
Cross Encoder
      ↓
Final Context
```

### 3. Query rewriting

Transform vague questions into better retrieval queries.

### 4. Persistent vector database

Move from local FAISS storage to Qdrant or pgvector.

### 5. User authentication

Allow different users to maintain separate document collections.

### 6. Streaming responses

Display the LLM response token-by-token.

### 7. Observability

Track:

```text
Question
Retrieved chunks
Retrieval scores
LLM latency
Token usage
Answer
```

### 8. Automated RAG evaluation

Create a benchmark dataset and continuously evaluate retrieval and answer quality.

---

# 🧠 What This Project Demonstrates

This project brought together several concepts that are important in modern GenAI applications:

```text
                  ┌──────────────────────┐
                  │       LLM            │
                  └──────────┬───────────┘
                             │
       ┌─────────────────────┼────────────────────┐
       │                     │                    │
       ▼                     ▼                    ▼
     RAG                 Embeddings          Prompting
       │                     │                    │
       ▼                     ▼                    ▼
 Hybrid Search            FAISS             Grounding
       │                                          │
       └──────────────────┬───────────────────────┘
                          ▼
                  Production API
                          │
                   ┌──────┴──────┐
                   ▼             ▼
               Streamlit       Docker
```

More importantly, it demonstrates the difference between **calling an LLM API** and **engineering an LLM-powered application**.

The LLM is only one component.

The surrounding system — ingestion, chunking, retrieval, ranking, context construction, memory, citations, APIs, testing and deployment — is what turns it into an application.

---

# 📌 Key Takeaway

The core idea behind this project can be summarized in one line:

> **Turn your documents into a searchable knowledge base, retrieve the right context, and let an LLM answer from that context.**

```text
Your PDFs
    ↓
Knowledge Base
    ↓
Hybrid Retrieval
    ↓
Relevant Context
    ↓
LLM
    ↓
Grounded Chatbot
```

And because the knowledge base is based on uploaded PDFs, the same application can be customized for completely different domains.

That is what I wanted to build: **not just another chatbot, but a reusable LLM application that can be adapted to a specific knowledge domain.**

---

## 🛠️ Tech Stack

**Python | FastAPI | Streamlit | OpenAI-compatible LLM API | Sentence Transformers | FAISS | BM25 | PyMuPDF | Pydantic | Docker | Pytest**

---

## 📂 Project

The complete source code, Docker configuration, tests, documentation and deployment files are available in the project repository.

**GitHub:** `Add your repository link here`

---

*Built as a personal Generative AI project to explore practical RAG architecture, document-grounded LLM applications, hybrid retrieval and end-to-end deployment.*
