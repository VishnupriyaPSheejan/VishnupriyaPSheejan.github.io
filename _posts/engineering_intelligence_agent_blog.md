# Engineering Intelligence Agent: Building an Agentic RAG System for Technical Troubleshooting

Modern engineering teams generate enormous amounts of technical knowledge: application logs, error reports, specifications, manuals, incident records, design documents, and troubleshooting notes. The challenge is not only storing this information, but making it searchable and useful when an engineer needs an answer quickly.

I built **Engineering Intelligence Agent**, an end-to-end **Agentic RAG platform** that combines semantic retrieval, LLM reasoning, multi-agent orchestration, evidence-based analysis, and an interactive application layer.

The project is designed to support technical investigation use cases such as:

- Root-cause analysis of engineering failures
- Technical log investigation
- Specification and documentation search
- Incident analysis
- Engineering decision support
- VLSI/EDA log and design-document analysis
- Software troubleshooting and developer assistance

---

## 1. What Problem Does It Solve?

A traditional document search system can retrieve a relevant paragraph, but an engineer often needs more than retrieval.

For example:

> **"Why did this timing analysis fail, what evidence supports the diagnosis, and what should I check next?"**

A useful system should:

1. Understand the question.
2. Plan an investigation.
3. Retrieve relevant technical evidence.
4. Analyze the evidence.
5. Check whether the analysis is supported.
6. Produce a concise engineering response.
7. Show the evidence behind the response.

That is the motivation behind this project.

---

# 2. High-Level Architecture

```text
                ┌───────────────────────────┐
                │   Engineering Documents  │
                │ Logs / PDFs / DOCX / MD   │
                │ TXT / CSV / Specifications│
                └─────────────┬─────────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ Document Ingestion│
                    └─────────┬─────────┘
                              │
                              ▼
                    ┌───────────────────┐
                    │ spaCy Preprocessing│
                    │ Text Cleaning      │
                    │ Chunking           │
                    └─────────┬─────────┘
                              │
                              ▼
                 ┌──────────────────────────┐
                 │ Sentence-BERT Embeddings │
                 └────────────┬─────────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │   FAISS Index   │
                     │ Semantic Search │
                     └────────┬────────┘
                              │
                    User Question
                              │
                              ▼
                    ┌─────────────────┐
                    │  Planner Agent  │
                    └────────┬────────┘
                             ▼
                   ┌──────────────────┐
                   │ Researcher Agent │
                   │ Semantic Retrieval│
                   └────────┬─────────┘
                            ▼
                    ┌────────────────┐
                    │  Analyst Agent │
                    └────────┬───────┘
                             ▼
                    ┌────────────────┐
                    │  Critic Agent  │
                    └────────┬───────┘
                             ▼
                 ┌──────────────────────┐
                 │ Synthesizer Agent    │
                 │ Evidence-grounded    │
                 │ final response       │
                 └──────────┬───────────┘
                            ▼
                 Answer + Evidence + Trace
```

---

# 3. Technology Stack

## AI / NLP

- Python
- Large Language Models
- Sentence-BERT
- spaCy
- Retrieval-Augmented Generation (RAG)
- Prompt engineering
- Semantic search
- LangGraph

## Retrieval

- Sentence-Transformer embeddings
- FAISS
- Metadata-aware document chunks

## Backend

- FastAPI
- Pydantic
- PostgreSQL
- Redis

## Frontend

- Streamlit

## Deployment

- Docker
- Docker Compose

## Testing and Evaluation

- Pytest
- Custom answer relevance evaluation
- Evidence coverage evaluation
- Citation-rate evaluation

---

# 4. Document Ingestion Pipeline

The system supports:

- PDF
- DOCX
- TXT
- Markdown
- LOG
- CSV

The ingestion flow is:

```text
File
 ↓
Text Extraction
 ↓
Cleaning
 ↓
spaCy Preprocessing
 ↓
Chunking
 ↓
Sentence-BERT Embedding
 ↓
FAISS Index
```

PDF files are processed using `pypdf`, while DOCX files are processed using `python-docx`.

The extracted text is normalized before being converted into chunks.

---

# 5. NLP Preprocessing

The preprocessing layer uses **spaCy** to normalize technical text before indexing.

The system performs:

- Whitespace normalization
- Null-character removal
- Token-level processing
- Text normalization
- Chunk generation

Chunking is configurable through:

```text
CHUNK_SIZE
CHUNK_OVERLAP
```

This is important for RAG because extremely large documents cannot simply be sent directly to an LLM. Smaller chunks make retrieval more targeted and keep the context manageable.

---

# 6. Semantic Encoding with Sentence-BERT

Instead of relying only on keyword matching, the system converts text into semantic embeddings using a Sentence-Transformer model.

The default model is:

```text
sentence-transformers/all-MiniLM-L6-v2
```

Conceptually:

```text
"setup timing violation"
             ↓
      Sentence-BERT
             ↓
       Vector embedding
             ↓
          FAISS
```

A user query is embedded using the same model and compared against the indexed document vectors.

This allows the system to retrieve conceptually related text even when the exact words differ.

---

# 7. FAISS Vector Retrieval

The project uses **FAISS** for efficient vector similarity search.

Embeddings are normalized and indexed using an inner-product FAISS index.

The retrieval process is:

```text
User Question
      ↓
Sentence-BERT
      ↓
Query Vector
      ↓
FAISS Similarity Search
      ↓
Top-K Relevant Chunks
```

Each indexed chunk retains metadata including:

- Document ID
- Filename
- Chunk ID
- Original chunk text

The retrieved evidence is then passed to the agent workflow.

---

# 8. Agentic RAG with LangGraph

The most important part of the project is that it is not simply:

```text
Question → Vector Search → LLM → Answer
```

Instead, the project implements a multi-step **Agentic RAG workflow** using LangGraph.

The graph contains five specialized stages.

---

## Planner Agent

The Planner receives the user's question and creates an investigation plan.

Example:

```text
Question:
Why did the timing analysis fail?

Plan:
1. Identify the reported timing violation.
2. Examine slack and failing paths.
3. Check clock-related factors.
4. Compare the evidence with known failure patterns.
5. Recommend validation steps.
```

The Planner does not directly answer the question.

---

## Researcher Agent

The Researcher performs semantic retrieval from the FAISS knowledge base.

It retrieves the most relevant technical chunks for the investigation.

Example:

```text
[EDA-2026-001:0]
STATUS=FAILED
VIOLATION=SETUP
SLACK=-0.143ns

[EDA-2026-001:1]
CHECKS=clock_uncertainty, setup_margin, path_delay
```

---

## Analyst Agent

The Analyst receives:

- User question
- Investigation plan
- Retrieved evidence

It then performs evidence-grounded reasoning.

The prompt explicitly instructs the model to distinguish:

- Facts
- Hypotheses
- Supporting evidence
- Missing information
- Recommended checks

This reduces the risk of treating an unsupported LLM-generated explanation as a confirmed fact.

---

# 9. Critic Agent

The Critic is responsible for reviewing the generated analysis.

It checks:

- Whether claims are supported by retrieved evidence
- Whether unsupported assumptions were introduced
- Whether important investigation steps are missing

This creates an additional verification layer before the final response.

The workflow therefore becomes:

```text
Retrieve
   ↓
Analyze
   ↓
Critique
   ↓
Synthesize
```

rather than immediately returning the first generated answer.

---

# 10. Synthesizer Agent

The final agent combines:

- Original question
- Investigation plan
- Retrieved evidence
- Analyst findings
- Critic feedback

It generates a structured response containing:

1. Executive conclusion
2. Evidence-backed findings
3. Likely causes/explanations
4. Recommended next checks
5. Evidence references

The response is instructed to use chunk identifiers such as:

```text
[document_id:chunk_id]
```

This provides traceability back to the retrieved evidence.

---

# 11. Example Engineering Use Case

The repository includes a sample timing-analysis log.

Example information:

```text
STAGE=STA
STATUS=FAILED
CLOCK=core_clk
SLACK=-0.143ns
VIOLATION=SETUP
```

A user can ask:

> Why did this timing analysis fail and what should I check next?

The system retrieves the relevant log chunks and runs the agent workflow.

The resulting analysis can identify the reported setup violation, point to the negative slack and relevant checks, and recommend areas such as clock uncertainty, setup margin, path delay, and input transition for further investigation.

The important point is that the response is generated from the indexed evidence rather than relying exclusively on the LLM's general knowledge.

---

# 12. FastAPI Backend

The application exposes REST APIs.

### Health

```http
GET /health
```

### Document Upload

```http
POST /v1/documents/upload
```

### Engineering Query

```http
POST /v1/query
```

Example request:

```json
{
  "question": "What are the likely causes of the timing failure?",
  "top_k": 6
}
```

The API returns:

```json
{
  "answer": "...",
  "citations": [],
  "trace": [
    "planner",
    "researcher",
    "analyst",
    "critic",
    "synthesizer"
  ],
  "metadata": {}
}
```

This separation between the AI workflow and application interface makes the system easier to integrate with other enterprise applications.

---

# 13. Streamlit User Interface

The Streamlit interface provides two basic workflows:

### Document ingestion

Users can upload supported technical files directly through the UI.

### Engineering investigation

Users can submit a technical question and view:

- Final answer
- Retrieved evidence
- Source document
- Chunk ID
- Retrieval score
- Agent execution trace

This makes the multi-agent workflow observable instead of presenting only a final black-box response.

---

# 14. Redis Caching

Redis is used to cache query responses.

The application creates a deterministic hash from the question and uses it as the cache key.

This avoids repeating the complete LLM workflow for identical queries within the configured cache lifetime.

Conceptually:

```text
Question
   ↓
SHA-256
   ↓
Redis Cache
   ├── Hit → Return cached response
   └── Miss → Run Agentic RAG → Cache response
```

---

# 15. PostgreSQL Persistence

PostgreSQL stores document metadata such as:

- Document ID
- Filename
- Number of chunks
- Creation timestamp

The vector index itself is persisted separately through FAISS.

This creates a separation between:

```text
PostgreSQL
→ application/document metadata

FAISS
→ semantic vector index
```

---

# 16. Evaluation

The repository includes a lightweight evaluation framework.

It evaluates generated responses using three signals:

### Answer Relevance

Measures overlap between important terms in the question and generated answer.

### Evidence Coverage

Measures how much of the generated answer is supported by retrieved evidence.

### Citation Rate

Measures the presence of evidence references in the final answer.

The project also includes a small golden-question dataset under:

```text
evals/golden_set.json
```

The evaluation script runs the questions through the complete agent workflow and reports average metrics.

For a larger production deployment, this evaluation layer could be extended with dedicated LLM evaluation frameworks, human-labeled datasets, retrieval metrics, faithfulness evaluation, regression testing, and experiment tracking.

---

# 17. Testing

The repository includes Pytest tests for:

- API health endpoint
- Text preprocessing
- Chunking
- Evaluation metric bounds

Run:

```bash
pytest
```

This provides a basic automated validation layer for the application.

---

# 18. Docker Deployment

The project includes:

```text
Dockerfile
docker-compose.yml
```

The Compose environment provides:

```text
PostgreSQL
Redis
FastAPI
Streamlit
```

The architecture can therefore be started as a containerized application rather than requiring every service to be manually installed.

```bash
docker compose up -d
```

---

# 19. Project Structure

```text
engineering_intelligence_agent/
│
├── app/
│   ├── agents.py
│   ├── cache.py
│   ├── config.py
│   ├── db.py
│   ├── evaluation.py
│   ├── ingestion.py
│   ├── llm.py
│   ├── main.py
│   ├── preprocessing.py
│   ├── schemas.py
│   ├── tools.py
│   └── vector_store.py
│
├── ui/
│   └── streamlit_app.py
│
├── tests/
│   ├── test_api.py
│   ├── test_evaluation.py
│   └── test_preprocessing.py
│
├── evals/
│   └── golden_set.json
│
├── scripts/
│   ├── ingest_folder.py
│   └── evaluate.py
│
├── data/
│   └── sample_timing_failure.log
│
├── storage/
│   ├── uploads/
│   └── vector/
│
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
├── .env.example
└── README.md
```

---

# 20. Running the Project

Create the environment:

```bash
python -m venv .venv
```

Activate it and install dependencies:

```bash
pip install -r requirements.txt
python -m spacy download en_core_web_sm
```

Configure the LLM in `.env`:

```env
LLM_BASE_URL=https://api.openai.com/v1
LLM_API_KEY=your_api_key
LLM_MODEL=your_model
```

Start the API:

```bash
uvicorn app.main:app --reload --port 8000
```

Start the Streamlit application:

```bash
streamlit run ui/streamlit_app.py
```

The application then provides:

```text
FastAPI → http://localhost:8000
API Docs → http://localhost:8000/docs
Streamlit → http://localhost:8501
```

---

# 21. Why This Architecture Matters

The key engineering idea is that **LLMs are not being used as the only source of truth**.

The system combines:

```text
Semantic Retrieval
        +
Relevant Evidence
        +
Multi-Agent Reasoning
        +
Critique
        +
LLM Generation
```

This is particularly useful for enterprise engineering applications where answers need to be grounded in internal technical information.

The architecture also separates responsibilities:

- **Sentence-BERT** → semantic representation
- **FAISS** → vector retrieval
- **RAG** → contextual grounding
- **LangGraph** → workflow/orchestration
- **LLM** → reasoning and generation
- **Critic Agent** → response review
- **FastAPI** → service layer
- **Streamlit** → user interface
- **Redis** → caching
- **PostgreSQL** → metadata persistence
- **Docker** → deployment packaging

---

# 22. Extending the System

The current architecture can be extended into a more advanced engineering copilot.

Possible next capabilities include:

### Codebase Intelligence

Index an entire Git repository and retrieve relevant source files based on a stack trace.

```text
Error
 ↓
Stack Trace Parser
 ↓
Code Retrieval
 ↓
RAG
 ↓
LLM Analysis
 ↓
Suggested Fix
```

### Automated Test Generation

Generate unit tests for the proposed fix and execute them in an isolated environment.

### Git Integration

Connect the system to GitHub/GitLab and provide:

- Pull-request analysis
- Code review
- Bug investigation
- Change-impact analysis

### VLSI/EDA Intelligence

The same architecture can be applied to:

- RTL documentation
- Verilog/SystemVerilog
- Synthesis logs
- STA reports
- Simulation logs
- Verification reports
- Design specifications

For example:

```text
EDA Error
   ↓
Log Retrieval
   ↓
Relevant Design Documentation
   ↓
Agentic Analysis
   ↓
Root Cause
   ↓
Recommended Investigation
```

---

# 23. Key Takeaway

Engineering Intelligence Agent demonstrates how **RAG, semantic embeddings, vector search, LLMs and multi-agent orchestration** can be combined into an end-to-end engineering intelligence system.

Rather than building a simple chatbot, the project treats an engineering question as an **investigation workflow**:

```text
Plan → Retrieve → Analyze → Critique → Synthesize
```

This architecture provides a foundation for building AI copilots for software engineering, semiconductor/VLSI workflows, technical support, incident management, and other knowledge-intensive engineering environments.

---

## Tech Stack

**Python | LLM | LangGraph | RAG | Sentence-BERT | FAISS | spaCy | FastAPI | Streamlit | PostgreSQL | Redis | Docker | Pytest**

---

## Repository

[Engineering Intelligence Agent](https://github.com/VishnupriyaPSheejan/engineering_intelligence_agent)
