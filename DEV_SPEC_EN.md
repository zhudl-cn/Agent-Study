# RAG Assistant Developer Specification

> Version: 0.1 — English Edition

## Table of Contents

- Project Overview
- Core Features
- Technology Stack
- Testing Strategy
- System Architecture & Module Design
- Project Schedule
- Extensibility & Future Roadmap

---

## 1. Project Overview

### Design Philosophy

> **Core Positioning: Intelligent Document Retrieval Assistant**
> 
> RAG Assistant is an intelligent document retrieval assistant based on RAG technology, focused on helping developers quickly and accurately retrieve technical documents from private knowledge bases. Through semantic understanding and hybrid retrieval technology, document search becomes more intelligent and efficient.

This project is based on **RAG (Retrieval-Augmented Generation)** technology, using **C/S architecture**: the backend deploys an independent REST API server, while the frontend provides two interaction modes through **VS Code Extension** — an independent **Webview chat interface** or **Copilot Chat tool calls**.

### Hardware Support

> **Deployment Mode: Hybrid Architecture**
> 
> - **LLM**: Provided via **GitHub Copilot** (enterprise subscription), no local deployment required
> - **Embedding / Rerank**: Runs locally, independent of external APIs (enterprise network isolation)

**Hardware Requirements**:

| Deployment Mode | Hardware Requirements | Use Case | Notes |
|----------------|----------------------|----------|-------|
| **CPU Only** | CPU (≥4 cores), ≥16GB RAM | Development/Testing | Embedding/Rerank use ONNX acceleration |
| **CPU + GPU** | NVIDIA GPU (≥8GB VRAM) | Production | GPU-accelerated Embedding/Rerank |

**Local Model Selection** (Embedding and Rerank only):

| Model Type | Recommended Model | Language Support |
|------------|-------------------|-----------------|
| **Embedding** | `intfloat/multilingual-e5-base` | 🌐 Multilingual |
| **Embedding** | `BAAI/bge-m3` | 🌐 Multilingual |
| **Rerank** | `BAAI/bge-reranker-v2-m3` | 🌐 Multilingual |

### Core Capabilities

| Capability | Description | Typical Scenarios |
|-----------|-------------|-------------------|
| 📚 **Semantic Retrieval** | Document search based on semantic understanding | Technical docs, API specs |
| 🔄 **Hybrid Retrieval** | Dense + Sparse dual-path recall | Fuzzy search + exact matching |
| 🎯 **Reranking** | Cross-Encoder fine-grained ranking | High-precision matching |
| 🖼️ **Multimodal** | Understanding images and tables in documents | Flowcharts, architecture diagrams |

### Dual Interaction Modes

**Mode A: Webview Chat Interface**
- Independent chat window, ChatGPT-like experience
- Support for file upload, history, multi-turn conversation

**Mode B: Copilot Chat Participant**
- Call via `@rag` command in Copilot Chat
- Seamless integration with Copilot conversations

---

## 2. Core Features

### 2.1 RAG Strategy Design

RAG Assistant adopts a **multi-stage RAG architecture** with a "rough recall → fusion ranking → fine reranking" pipeline.

#### Retrieval Pipeline

```
User Query → Query Processing → Hybrid Retrieval → Fusion → Rerank → Answer Generation
                                    ↓
                      ┌─────────────┴─────────────┐
                      ▼                           ▼
                Dense Retrieval             Sparse Retrieval
                (Semantic Vectors)          (BM25 Keywords)
                      └─────────────┬─────────────┘
                                    ▼
                                RRF Fusion
```

#### Stage Strategies

| Stage | Strategy | Description |
|-------|----------|-------------|
| **Chunking** | Recursive Character Splitting | Split by paragraph/sentence boundaries |
| **Embedding** | Multilingual E5 / BGE-M3 | 768-dim vectors, multilingual |
| **Sparse Retrieval** | BM25 | Keyword matching, terminology optimization |
| **Fusion** | RRF (Reciprocal Rank Fusion) | k=60 for balanced weighting |
| **Reranking** | Cross-Encoder | Fine-grained relevance scoring |

### 2.2 Document Processing

| Document Type | Loader | Features |
|--------------|--------|----------|
| **PDF** | PyMuPDF | Text extraction, image extraction |
| **Markdown** | Mistune | Heading hierarchy preservation |
| **Word** | python-docx | Paragraph and table extraction |
| **Excel** | openpyxl | Sheet-by-sheet processing |

### 2.3 Vector Storage

| Component | Technology | Description |
|-----------|------------|-------------|
| **Vector Store** | Chroma | Persistent local storage |
| **Sparse Index** | BM25 (rank_bm25) | In-memory keyword index |

### 2.4 Observability & Evaluation

| Capability | Description |
|-----------|-------------|
| **Trace Logging** | Full pipeline tracing with unique trace_id |
| **Dashboard** | Streamlit UI for visualizing traces |
| **Evaluation** | Ragas integration for RAG quality metrics |

---

## 3. Technology Stack

### Server Side

| Layer | Technology |
|-------|------------|
| **Framework** | FastAPI |
| **Language** | Python 3.11+ |
| **Embedding** | Sentence-Transformers |
| **Vector Store** | Chroma |
| **Sparse Retrieval** | rank_bm25 |
| **Reranking** | Cross-Encoder |

### Client Side (VS Code Extension)

| Component | Technology |
|-----------|------------|
| **Language** | TypeScript |
| **Build** | esbuild |
| **UI** | WebView (HTML/CSS/JS) |
| **LLM** | VS Code Language Model API (Copilot GPT-4o) |

---

## 4. Testing Strategy

### Test Pyramid

```
        /\
       /E2E\         <- Few: Verify key business flows
      /------\
     /Integration\   <- Medium: Verify module collaboration
    /------------\
   /  Unit Tests  \  <- Many: Verify individual functions/classes
  /________________\
```

### Quality Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| **Recall@K** | Relevant docs in Top-K | ≥0.8 |
| **MRR** | Mean Reciprocal Rank | ≥0.7 |
| **Faithfulness** | Answer-context consistency | ≥0.85 |
| **Relevance** | Answer-question relevance | ≥0.8 |

### Performance Targets

| Test Type | Target |
|-----------|--------|
| **Retrieval Latency** | ≤500ms |
| **Ingestion Throughput** | ≥10 docs/min |
| **Concurrent Requests** | ≥10 requests |

---

## 5. System Architecture

### Module Structure

```
rag-server/
├── src/
│   ├── api/           # REST API (FastAPI)
│   ├── core/          # Configuration management
│   └── rag/
│       ├── ingestion/     # Document loaders & splitters
│       ├── embedding/     # Vector embedding
│       ├── vectorstore/   # Chroma integration
│       ├── sparse/        # BM25 retrieval
│       ├── fusion/        # RRF fusion
│       ├── rerank/        # Cross-Encoder reranking
│       ├── cache/         # LRU caching
│       ├── observability/ # Trace logging
│       ├── evaluation/    # Ragas evaluation
│       └── dashboard/     # Streamlit UI
│
├── cloud_sources/         # Cloud data source integration
│   ├── jira/              # JIRA REST API client
│   ├── confluence/        # Confluence REST API client
│   ├── scheduler/         # Sync scheduler
│   ├── summarizer/        # BERT summarization
│   └── indexer/           # Directory index builder
```

### 5.5 Cloud Data Source Design

Design a unified module for fetching, processing, and indexing cloud data sources like JIRA and Confluence, enabling automated data synchronization and intelligent summarization.

#### 5.5.1 Architecture Overview

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Cloud Data Source Module                            │
│                                                                             │
│  ┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐          │
│  │   JIRA Client   │    │ Confluence Client│    │  Other Sources  │          │
│  └────────┬────────┘    └────────┬────────┘    └────────┬────────┘          │
│           └──────────────────────┼──────────────────────┘                   │
│                                  ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Sync Scheduler                                │  │
│  │  • Full/Incremental Sync    • Cron Jobs    • Checkpoint Resume        │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                  ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Data Cleaning & Preprocessing                 │  │
│  │  • HTML/Confluence Format Parsing    • Content Cleaning    • Metadata │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                  ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         BERT Summarization & Indexing                 │  │
│  │  • Document Summary    • Keyword Extraction    • Directory Tree       │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
│                                  ▼                                          │
│  ┌───────────────────────────────────────────────────────────────────────┐  │
│  │                         Vectorization & Storage                       │  │
│  │  • Reuse Existing Embedding Pipeline    • Dual Index (Vector + BM25) │  │
│  └───────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 5.5.2 Module Responsibilities

| Module | Responsibility | Description |
|--------|----------------|-------------|
| **JIRA Client** | JIRA Data Fetching | Get projects, issues, comments, attachments |
| **Confluence Client** | Confluence Data Fetching | Get spaces, pages, child pages, attachments |
| **Scheduler** | Sync Strategy Management | Full/incremental sync, cron jobs, state management |
| **Data Cleaning** | Content Preprocessing | HTML parsing, format conversion, noise removal |
| **BERT Summarizer** | Intelligent Summarization | Document summaries, keyword extraction |
| **Index Builder** | Directory Tree Construction | Project/space directory, metadata indexing |
| **Vectorization** | RAG Pipeline Integration | Reuse existing Embedding and storage modules |

#### 5.5.3 Sync Strategies

| Strategy | Description | Use Case |
|----------|-------------|----------|
| **Full Sync** | Fetch all data on first run | Initial indexing |
| **Incremental Sync** | Fetch changes based on `lastModified` timestamp | Daily updates |
| **Scheduled Sync** | Execute via Cron expressions | Automated maintenance |

#### 5.5.4 BERT Summarization Model Selection

| Model | Size | Language | Features |
|-------|------|----------|----------|
| `bert-extractive-summarizer` | Base | Multilingual | Extractive summary, fast |
| `facebook/bart-large-cnn` | Large | English | Generative summary, high quality |
| `csebuetnlp/mT5_multilingual_XLSum` | Base | Multilingual | Multilingual generative summary |

#### 5.5.5 Directory Index Structure

```
cloud_index/
├── JIRA/
│   ├── PROJECT-A/
│   │   ├── _index.json          # Project metadata + summary
│   │   └── issues/              # Issue details
│   └── PROJECT-B/
└── Confluence/
    ├── Space-A/
    │   ├── _index.json          # Space metadata + summary
    │   └── pages/               # Page content
    └── Space-B/
```

#### 5.5.6 Metadata Fields

| Field | Description |
|-------|-------------|
| `source` | Data source type (jira/confluence) |
| `source_url` | Original link |
| `project` / `space` | Parent project/space |
| `summary` | BERT-generated summary |
| `keywords` | Extracted keywords |
| `last_synced` | Last synchronization time |

---

## 6. Project Schedule

### Phase Overview

| Phase | Name | Goal |
|-------|------|------|
| A | Infrastructure | Project skeleton, config, basic API |
| B | RAG Core Pipeline | Ingestion & retrieval implementation |
| C | Advanced Features | Hybrid retrieval, Rerank |
| D | Observability | Dashboard, evaluation framework |
| E | Optimization | Performance, documentation |
| F | VS Code Extension | WebView, file upload, Copilot integration |
| G | JIRA/Confluence Automation | Cloud data sync, BERT summarization, indexing |

### Phase A: Infrastructure ✅

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| A1: Project Init ✅ | Project structure, dependencies | `pip install -e .` succeeds |
| A2: Config ✅ | Configuration loading & validation | Config loads correctly |
| A3: REST API ✅ | FastAPI basic routes | `/api/health` returns 200 |

### Phase B: RAG Core Pipeline ✅

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| B1: Loader ✅ | PDF/Markdown/Word loading | Documents parse correctly |
| B2: Splitter ✅ | Recursive chunking | Chunk size matches config |
| B3: Embedding ✅ | Local model integration | Vector dimensions correct |
| B4: VectorStore ✅ | Chroma integration | Documents store and query |
| B5: Retrieval API ✅ | `/api/query` implementation | Retrieval returns results |

### Phase C: Advanced Features ✅

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| C1: BM25 ✅ | Sparse retrieval index | Sparse retrieval works |
| C2: RRF ✅ | Hybrid retrieval fusion | Fusion outperforms single strategy |
| C3: Rerank ✅ | Cross-Encoder integration | Reranked order is better |
| C4: Multimodal ❌ | Vision model integration | Removed (slow model loading) |

### Phase D: Observability ✅

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| D1: Structured Logging ✅ | Trace logging | All stages logged |
| D2: Dashboard ✅ | Streamlit UI | Request details viewable |
| D3: Evaluation ✅ | Ragas integration | Metrics calculable |

### Phase E: Optimization ✅

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| E1: Performance ✅ | Batch processing, caching | Latency meets target |
| E2: README ✅ | Usage documentation | New users can get started |
| E3: API Docs ✅ | OpenAPI documentation | Swagger UI accessible |

### Phase F: VS Code Extension

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| F1: Project Init | TypeScript project setup | Extension loads |
| F2: WebView UI | Chat interface implementation | Messages send/receive |
| F3: RAG API Integration | REST client | Retrieval results display |
| F4: File Upload | Upload to RAG Server | Files upload successfully |
| F5: Copilot Integration | GPT-4o answer generation | RAG-enhanced answers work |
| F6: Testing & Packaging | VSIX packaging | Can install and use |

### Phase G: JIRA/Confluence Automation

| Task | Description | Acceptance Criteria |
|------|-------------|---------------------|
| G1: JIRA API Client | Implement JIRA REST API client | Can fetch ticket list, details, comments, attachments |
| G2: Confluence API Client | Implement Confluence REST API client | Can fetch page list, content, child pages, attachments |
| G3: Sync Scheduler | Scheduled/incremental sync mechanism | Supports cron jobs and incremental update detection |
| G4: BERT Summarizer | Integrate BERT summarization model | Can generate document summaries |
| G5: Directory Generator | Build directory tree from content structure | Clear hierarchy, navigable |
| G6: Data Preprocessing | HTML/Markdown parsing and cleaning | Clean, structured content |
| G7: Vectorization & Import | Import via existing pipeline | Documents searchable, sources traceable |
| G8: Configuration | API credentials, sync rules config | Flexible config, secure storage |
| G9: CLI Tool | Provide CLI for triggering sync | Can be called manually or via scripts |

---

## 7. Extensibility & Future Roadmap

### 7.1 New Data Sources

**Currently Supported**:
- Local documents: PDF, Excel, PPT, Word, Images
- Cloud documents: JIRA Ticket, Confluence (via REST API)

**Extension Directions**:
- More cloud sources: Notion, SharePoint
- Enterprise knowledge bases

### 7.2 More Client Integrations

Currently supported: VS Code Extension, REST API

**Extension Directions**:
- Web UI: Independent web interface
- CLI Tool: Command-line retrieval tool
- MCP Protocol: Optional support for more clients

### 7.3 Advanced RAG Strategies

**Extension Directions**:
- Agentic RAG: Multi-turn conversation with agents
- Graph RAG: Knowledge graph enhanced retrieval
- Self-RAG: Self-reflection and iterative retrieval

### 7.4 Enterprise Capabilities

**Extension Directions**:
- Access Control: Document-level permissions
- Distributed Deployment: Multi-node scaling
- Monitoring & Alerting: Prometheus/Grafana integration
