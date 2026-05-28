# Cease & Desist Document Processing — Agentic AI System

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/YOUR-USERNAME/YOUR-REPO/blob/main/Capstone_Project_Rajendra_Raju_final.ipynb)
[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://python.org)
[![LangGraph](https://img.shields.io/badge/LangGraph-Agentic-green.svg)](https://github.com/langchain-ai/langgraph)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

## The Problem

Enterprises receive hundreds of Cease & Desist PDFs — customers formally requesting all communication stop. Someone has to manually read each one to decide if it is legitimate.

That process is slow, inconsistent, and does not scale.

This project automates it using a multi-agent LangGraph pipeline with RAG-augmented classification.

---

## What Makes This Different From a Basic LLM Classifier

Two things were added after finding that a plain LLM prompt was not reliable enough:

**1. RAG before classification**
Before the model reads any document, it retrieves the most relevant legal rules from a ChromaDB knowledge base. The model classifies with legal context, not just pattern matching. This made a noticeable difference on edge cases like Letters of Authority, which a plain prompt often misclassified as Cease requests.

**2. Confidence threshold**
If the model's confidence is below 0.60, the document is automatically escalated to human review — regardless of what label the model chose. A wrong auto-classification in production is far more expensive than a manual review.

---

## System Architecture

```
PDF Upload
    │
    ▼
Document Loader Agent       ← extracts text from PDF
    │
    ▼
Classification Agent        ← RAG context + LLM + Pydantic structured output
    │                         (if confidence < 0.60, label is overridden to Uncertain)
    │
    ├── label = Cease      → Database Agent   → SQLite
    ├── label = Irrelevant → Archive Agent    → JSONL
    └── label = Uncertain  → HITL Agent       → human review
         (low confidence        │
          or ambiguous)         └── human decides → Database or Archive
                                                      │
                                                      ▼
                                               Audit Agent       ← logs every decision
                                                      │
                                                     END
```

LangGraph compiles these as a proper state machine — routing is explicit, not buried in if-else chains.

---

## Agents

| Agent | Job |
|---|---|
| Document Loader | Reads the PDF, extracts text via `pypdf` |
| Classification Agent | Retrieves RAG context, calls Groq LLM, applies confidence threshold |
| Database Agent | Inserts Cease records into SQLite with full metadata |
| Archive Agent | Appends Irrelevant records to a JSONL flat file |
| HITL Agent | Presents Uncertain documents for human review, captures decision + notes |
| Audit Agent | Writes one audit record per document to `audit_log.jsonl` |

---

## Classification Schema

The LLM returns a typed Pydantic object — not free text:

```python
class ClassificationResult(BaseModel):
    label:            Literal["Cease", "Uncertain", "Irrelevant"]
    confidence:       float          # 0.0 – 1.0
    explanation:      str            # model's rationale
    customer_name:    str            # extracted from document
    request_summary:  str            # one-line summary
    key_details:      str            # key facts and dates
```

---

## Output Files

| File | Format | Contains |
|---|---|---|
| `outputs/cease_documents.db` | SQLite | All Cease records with metadata |
| `outputs/irrelevant_archive.jsonl` | JSONL | Irrelevant document records |
| `outputs/audit_log.jsonl` | JSONL | Every decision logged here, one line per document |
| `outputs/hitl_queue.jsonl` | JSONL | Manual review decisions — who reviewed, what they decided, their notes |
| `outputs/processing_errors.jsonl` | JSONL | Per-file errors (only if errors occur) |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Workflow orchestration | LangGraph |
| LLM | Groq — LLaMA 3.3 70B |
| Structured output | Pydantic v2 |
| RAG embeddings | HuggingFace — all-MiniLM-L6-v2 |
| Vector store | ChromaDB (in-memory) |
| PDF parsing | pypdf |
| Storage | SQLite3 + JSONL |
| Observability | LangSmith, Arize Phoenix (optional) |
| Runtime | Google Colab / Python 3.10+ |

---

## How to Run

### 1. Open in Google Colab
Click the badge at the top of this README.

### 2. Add your Groq API key to Colab Secrets
Click the 🔑 icon in the left sidebar and add:

| Secret | Required |
|---|---|
| `GROQ_API_KEY` | ✅ Yes |
| `LANGSMITH_API_KEY` | Optional |
| `ARIZE_API_KEY` | Optional |

Get a free Groq API key at [console.groq.com](https://console.groq.com)

### 3. Run cells top to bottom

| Cell | What it does |
|---|---|
| 1 — Install | pip installs |
| 2 — Secrets | reads Groq key from Colab Secrets |
| 3 — Upload | upload your PDFs here |
| 4 — Imports | imports + output paths |
| 5 — Helpers | PDF reader, JSONL writer, dedup |
| 6 — Database | creates SQLite table |
| 7 — RAG + LLM | RAG setup + Groq LLM |
| 8 — Agents | all 6 agent functions |
| 9 — Graph | builds and compiles the graph |
| 10 — Config | set `TEST_MODE` before running |
| 11 — Execute | runs the pipeline |
| 12 — Review | print results + pipeline summary |

### 4. Test modes

```python
TEST_MODE = "one"   # quick test on first PDF — interactive HITL enabled
TEST_MODE = "two"   # first two PDFs — interactive HITL enabled
TEST_MODE = "all"   # all uploaded PDFs — non-interactive batch mode
```

---

## Engineering Decisions

**Why LangGraph over a simple loop?**
The routing logic (Cease → DB, Irrelevant → archive, Uncertain → human) is a state machine. LangGraph makes that explicit and inspectable. A for-loop with if-else chains buries the logic and makes it hard to extend.

**Why RAG for classification?**
Without RAG, the LLM had no grounding in what legally distinguishes a Cease & Desist from a Letter of Authority. Injecting the top-matching rules from the knowledge base before classification significantly reduced false positives on ambiguous documents.

**Why Pydantic structured output?**
Free-text LLM responses require parsing and validation. Pydantic enforces the schema at the model level — if the LLM returns a malformed response, the retry logic catches it before it reaches downstream agents.

**Why a confidence threshold?**
High confidence does not mean correct. Setting a floor at 0.60 ensures borderline cases always reach a human reviewer rather than being silently auto-classified.

---

## Project Structure

```
📦 cease-desist-agent/
├── 📓 Capstone_Project_Rajendra_Raju.ipynb
├── 📁 uploaded_pdfs/          ← PDFs go here (auto-created)
├── 📁 outputs/
│   ├── cease_documents.db
│   ├── irrelevant_archive.jsonl
│   ├── audit_log.jsonl
│   ├── hitl_queue.jsonl
│   └── processing_errors.jsonl
└── 📄 README.md
```

---

## Author

**Rajendra Raju**

[LinkedIn](https://www.linkedin.com/in/YOUR-PROFILE) · [GitHub](https://github.com/YOUR-USERNAME)
