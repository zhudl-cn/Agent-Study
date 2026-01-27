# RAG Tool Architecture Document

> 简易 RAG 工具架构设计文档
> Version: 1.2
> Date: 2025-01
> Last Updated: 2025-01-27

---

## Table of Contents

1. [Project Overview](#1-project-overview)
   - [1.1 Purpose](#11-purpose)
   - [1.2 Core Requirements](#12-core-requirements)
   - [1.3 Key Features](#13-key-features)
2. [System Architecture](#2-system-architecture)
   - [2.1 High-Level Architecture](#21-high-level-architecture)
   - [2.2 Data Flow](#22-data-flow)
3. [Technology Stack](#3-technology-stack)
   - [3.1 Complete Technology Stack](#31-complete-technology-stack)
   - [3.2 Model Storage Layout](#32-model-storage-layout)
   - [3.3 Model Download Script](#33-model-download-script)
4. [Module Design](#4-module-design)
   - [4.1 VS Code Extension](#41-vs-code-extension)
   - [4.2 Backend Service](#42-backend-service)
   - [4.3 AI Models Service](#43-ai-models-service)
   - [4.4 Document Processing](#44-document-processing)
   - [4.5 Confluence Integration](#45-confluence-integration)
   - [4.6 Scheduler Tasks](#46-scheduler-tasks)
   - [4.7 Vector Store](#47-vector-store)
5. [Data Structures](#5-data-structures)
6. [Deployment](#6-deployment)
   - [6.1 Directory Structure](#61-directory-structure)
   - [6.2 Environment Configuration](#62-environment-configuration)
   - [6.3 Systemd Service](#63-systemd-service)
   - [6.4 Installation Script](#64-installation-script)
   - [6.5 Requirements](#65-requirements)
7. [Development Guide](#7-development-guide)
8. [Performance Considerations](#8-performance-considerations)
9. [Security Considerations](#9-security-considerations)
10. [Error Handling Strategy](#10-error-handling-strategy)
11. [Future Enhancements](#11-future-enhancements)
- [Appendix A: Quick Start Guide](#appendix-a-quick-start-guide)
- [Appendix B: Skill Command Reference](#appendix-b-skill-command-reference)
- [Appendix C: Troubleshooting](#appendix-c-troubleshooting)
- [Appendix D: Complete Configuration Reference](#appendix-d-complete-configuration-reference)
- [Appendix E: API Reference](#appendix-e-api-reference)

---

## 1. Project Overview

### 1.1 Purpose

构建一个本地化部署的 RAG (Retrieval-Augmented Generation) 工具，支持多种文档格式和图片内容的检索，通过 VS Code 插件形式提供用户交互界面，使用 Copilot API 作为唯一的大语言模型接口。

### 1.2 Core Requirements

| Requirement | Description |
|-------------|-------------|
| Data Sources | PDF, Word, Excel, PPT, Confluence, Images (text/table/UML) |
| Languages | English, Japanese (primary), Chinese (secondary) |
| Deployment | Fully local, no external API calls (except Copilot) |
| Hardware | Linux server, Intel Xeon CPU, No GPU |
| Interface | VS Code Extension with Webview |
| LLM | Copilot API only |

### 1.3 Key Features

- Multi-format document parsing and indexing
- Image understanding (OCR, table recognition, UML diagram analysis)
- Skill-based interaction system
- Confluence real-time integration with index caching
- User image upload for analysis (max 2 images)

---

## 2. System Architecture

### 2.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           VS Code Extension                             │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                        Webview Chat UI                            │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │  Command Bar: /search /confluence /explain-uml /compare ... │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │  Chat History (Markdown + Inline Images + Source Links)     │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  │  ┌─────────────────────────────────────────────────────────────┐  │  │
│  │  │  Input: [Text] [📎 Upload max 2 images] [Send]              │  │  │
│  │  └─────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         Skill Router                              │  │
│  │            (Command Parsing + Intent Recognition)                 │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                    │                                    │
│                                    ▼                                    │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │              Skill Context + User Query ──► Copilot API           │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────┬────────────────────────────────────┘
                                     │ HTTP
                                     ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        Backend Service (FastAPI)                        │
│                                                                         │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                         Skill Handlers                            │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │  │
│  │  │ search   │ │confluence│ │  docs    │ │ analyze  │              │  │
│  │  │ (全源)   │ │ (wiki)   │ │ (本地)   │ │ (图片)   │              │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐              │  │
│  │  │explain   │ │ compare  │ │ summary  │ │  index   │              │  │
│  │  │-uml      │ │ (对比)   │ │ (摘要)   │ │ (刷新)   │              │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘              │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│                                                                         │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────┐  │
│  │ Confluence      │  │ Document        │  │ Local AI Models         │  │
│  │ Index Manager   │  │ Vector Store    │  │ (BGE/CLIP/Qwen2-VL/     │  │
│  │ + Realtime API  │  │ (Qdrant)        │  │  PaddleOCR)             │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────────────┘  │
│                                                                         │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │                    Scheduler (APScheduler)                      │    │
│  │                  - Confluence Index Sync                        │    │
│  │                  - Cleanup Tasks                                │    │
│  └─────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
                                     │
           ┌─────────────────────────┼─────────────────────────┐
           ▼                         ▼                         ▼
    ┌─────────────┐          ┌─────────────┐          ┌─────────────┐
    │ Confluence  │          │ Local Files │          │ Image Store │
    │ REST API    │          │ (PDF/Office)│          │ /opt/rag/   │
    └─────────────┘          └─────────────┘          └─────────────┘
```

### 2.2 Data Flow

#### 2.2.1 Document Ingestion Flow

```
Local Document (PDF/Word/Excel/PPT)
              │
              ▼
┌─────────────────────────────────┐
│      Document Parser            │
│  - PyMuPDF (PDF)                │
│  - python-docx (Word)           │
│  - openpyxl (Excel)             │
│  - python-pptx (PPT)            │
└─────────────────────────────────┘
              │
              ├──── Text Content ────┐
              │                      │
              ▼                      ▼
┌─────────────────────────┐  ┌─────────────────────────┐
│    Image Extraction     │  │    Text Chunking        │
└─────────────────────────┘  │    (512-1024 chars)     │
              │              └─────────────────────────┘
              ▼                      │
┌─────────────────────────┐          │
│   Image Classification  │          │
│   (CLIP zero-shot)      │          │
└─────────────────────────┘          │
              │                      │
    ┌─────────┼─────────┐            │
    ▼         ▼         ▼            │
┌───────┐ ┌───────┐ ┌───────┐        │
│ Text  │ │ Table │ │  UML  │        │
│ Image │ │ Image │ │ Image │        │
└───┬───┘ └───┬───┘ └───┬───┘        │
    │         │         │            │
    ▼         ▼         ▼            │
┌───────┐ ┌───────┐ ┌───────┐        │
│Paddle │ │  PP-  │ │Qwen2  │        │
│  OCR  │ │Struct │ │  VL   │        │
└───┬───┘ └───┬───┘ └───┬───┘        │
    │         │         │            │
    └─────────┴────┬────┘            │
                   │                 │
                   ▼                 ▼
          ┌─────────────────────────────┐
          │    Text Description         │
          │    + Image Path             │
          │    + Source Metadata        │
          └─────────────────────────────┘
                        │
                        ▼
          ┌─────────────────────────────┐
          │      BGE Embedding          │
          │      + Store in Qdrant      │
          └─────────────────────────────┘
```

#### 2.2.2 Query Flow

```
User Input (Text + Optional Images)
              │
              ▼
┌─────────────────────────────────┐
│         Skill Router            │
│  1. Parse command (/xxx)        │
│  2. Intent recognition          │
│  3. Extract parameters          │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│      Selected Skill Handler     │
│  - Execute skill logic          │
│  - Retrieve relevant context    │
│  - Process user images (if any) │
└─────────────────────────────────┘
              │
              ├──── Has User Images? ────┐
              │              Yes         │
              │                          ▼
              │           ┌─────────────────────────┐
              │           │  VLM Image Description  │
              │           │  (Qwen2-VL, ~15-30s)    │
              │           └─────────────────────────┘
              │                          │
              ▼                          ▼
┌─────────────────────────────────────────────────┐
│              Compose Context                    │
│  - Retrieved documents/chunks                   │
│  - Image descriptions                           │
│  - Source references                            │
└─────────────────────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│         Copilot API             │
│  Context + User Question        │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│      Response to User           │
│  - Answer text (Markdown)       │
│  - Referenced images            │
│  - Source links                 │
└─────────────────────────────────┘
```

#### 2.2.3 Confluence Integration Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   Confluence Index Mode                         │
│                                                                 │
│   Storage: Index only (metadata)                                │
│   Content: Fetched on-demand via API                            │
└─────────────────────────────────────────────────────────────────┘

Scheduled Sync (Every N hours)
              │
              ▼
┌─────────────────────────────────┐
│  1. Fetch Space/Page List       │
│     GET /wiki/rest/api/content  │
│     ?expand=version,ancestors,  │
│      metadata.labels            │
│     (No body content)           │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  2. Update Local Index          │
│     - Page ID, Title, Path      │
│     - Labels, Last Modified     │
│     - Parent relationships      │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  3. Detect Deleted Pages        │
│     Remove from index           │
└─────────────────────────────────┘


Query Time
              │
              ▼
┌─────────────────────────────────┐
│  1. Search Index (Local, Fast)  │
│     - Title matching            │
│     - Path matching             │
│     - Label matching            │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  2. Fetch Content (Realtime)    │
│     GET /wiki/rest/api/content/ │
│         {pageId}?expand=body    │
│     (Parallel requests)         │
└─────────────────────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│  3. Parse & Extract             │
│     - HTML to Markdown          │
│     - Extract relevant sections │
│     - Preserve image URLs       │
└─────────────────────────────────┘
```

---

## 3. Technology Stack

### 3.1 Complete Technology Stack

| Layer | Technology | Version | Purpose | Local Deployment |
|-------|------------|---------|---------|------------------|
| **VS Code Extension** |
| Runtime | Node.js | 18+ | Extension host | ✅ |
| Language | TypeScript | 5.x | Extension development | ✅ |
| UI | Webview API | - | Chat interface | ✅ |
| LLM | vscode.lm API | - | Copilot integration | ✅ (API only) |
| **Backend Service** |
| Framework | FastAPI | 0.100+ | REST API server | ✅ |
| Server | Uvicorn | 0.23+ | ASGI server | ✅ |
| Scheduler | APScheduler | 3.10+ | Scheduled tasks | ✅ |
| **Document Processing** |
| PDF | PyMuPDF | 1.23+ | PDF parsing | ✅ |
| Word | python-docx | 0.8+ | DOCX parsing | ✅ |
| Excel | openpyxl | 3.1+ | XLSX parsing | ✅ |
| PowerPoint | python-pptx | 0.6+ | PPTX parsing | ✅ |
| **AI Models (All Local)** |
| Text Embedding | BAAI/bge-base-en-v1.5 | - | Vector embedding | ✅ ~400MB |
| Image Classification | openai/clip-vit-base-patch32 | - | Zero-shot classification | ✅ ~600MB |
| Vision Language | Qwen/Qwen2-VL-2B-Instruct | transformers | Image understanding | ✅ ~4GB |
| OCR | PaddleOCR | 2.7+ | Text extraction | ✅ ~150MB |
| Table Recognition | PP-Structure | 2.7+ | Table extraction | ✅ (included) |
| **Storage** |
| Vector DB | Qdrant | 1.7+ | Vector storage | ✅ Embedded mode |
| File Storage | Local FS | - | Images and documents | ✅ |
| Index Storage | JSON/SQLite | - | Confluence index | ✅ |

### 3.2 Model Storage Layout

```
/opt/rag-models/
├── bge-base-en-v1.5/              # ~400MB
│   ├── config.json
│   ├── model.safetensors
│   ├── tokenizer.json
│   └── tokenizer_config.json
│
├── clip-vit-base-patch32/          # ~600MB
│   ├── config.json
│   ├── model.safetensors
│   └── preprocessor_config.json
│
├── vlm/                            # ~2-4GB (choose one)
│   ├── config.json
│   ├── model.safetensors
│   └── tokenizer.json
│   # Recommended: Qwen/Qwen2-VL-2B-Instruct or vikhyatk/moondream2
│
└── paddleocr/                      # ~400MB (multi-language)
    ├── det/                        # Detection model (language-agnostic)
    │   └── en_PP-OCRv4_det_infer/
    ├── rec/                        # Recognition models (per language)
    │   ├── en/                     # English
    │   │   └── en_PP-OCRv4_rec_infer/
    │   ├── japan/                  # Japanese
    │   │   └── japan_PP-OCRv3_rec_infer/
    │   └── ch/                     # Chinese
    │       └── ch_PP-OCRv4_rec_infer/
    ├── cls/                        # Classification model
    │   └── ch_ppocr_mobile_v2.0_cls_infer/
    └── table/                      # Table structure model
        └── en_ppstructure_mobile_v2.0_SLANet_infer/
```

### 3.3 Model Download Script

```python
# scripts/download_models.py
"""
Run this script on a machine with internet access,
then copy /opt/rag-models to the target server.
"""

from huggingface_hub import snapshot_download
import os

MODEL_DIR = "/opt/rag-models"

models = {
    "bge-base-en-v1.5": "BAAI/bge-base-en-v1.5",
    "clip-vit-base-patch32": "openai/clip-vit-base-patch32",
    "vlm": "Qwen/Qwen2-VL-2B-Instruct",  # or "vikhyatk/moondream2" for faster inference
}

for local_name, repo_id in models.items():
    print(f"Downloading {repo_id}...")
    snapshot_download(
        repo_id=repo_id,
        local_dir=os.path.join(MODEL_DIR, local_name),
        local_dir_use_symlinks=False
    )

# PaddleOCR models - download manually for multi-language support
paddleocr_models = {
    # Detection (language-agnostic)
    "det": "https://paddleocr.bj.bcebos.com/PP-OCRv4/english/en_PP-OCRv4_det_infer.tar",
    # Recognition (per language)
    "rec_en": "https://paddleocr.bj.bcebos.com/PP-OCRv4/english/en_PP-OCRv4_rec_infer.tar",
    "rec_japan": "https://paddleocr.bj.bcebos.com/PP-OCRv3/multilingual/japan_PP-OCRv3_rec_infer.tar",
    "rec_ch": "https://paddleocr.bj.bcebos.com/PP-OCRv4/chinese/ch_PP-OCRv4_rec_infer.tar",
    # Classification
    "cls": "https://paddleocr.bj.bcebos.com/dygraph_v2.0/ch/ch_ppocr_mobile_v2.0_cls_infer.tar",
    # Table structure
    "table": "https://paddleocr.bj.bcebos.com/ppstructure/models/slanet/en_ppstructure_mobile_v2.0_SLANet_infer.tar"
}

import urllib.request
import tarfile

paddleocr_dir = os.path.join(MODEL_DIR, "paddleocr")
os.makedirs(paddleocr_dir, exist_ok=True)

for name, url in paddleocr_models.items():
    print(f"Downloading PaddleOCR {name}...")
    tar_path = os.path.join(paddleocr_dir, f"{name}.tar")
    urllib.request.urlretrieve(url, tar_path)

    with tarfile.open(tar_path, 'r') as tar:
        tar.extractall(paddleocr_dir)
    os.remove(tar_path)

print("All models downloaded successfully!")
```

---

## 4. Module Design

### 4.1 VS Code Extension

#### 4.1.0 Extension Package Configuration

```json
// package.json
{
  "name": "rag-assistant",
  "displayName": "RAG Assistant",
  "description": "RAG-based document retrieval with Copilot integration",
  "version": "1.0.0",
  "publisher": "your-company",
  "engines": {
    "vscode": "^1.85.0"
  },
  "categories": ["Other"],
  "activationEvents": [
    "onCommand:rag-assistant.openChat"
  ],
  "main": "./out/extension.js",
  "contributes": {
    "commands": [
      {
        "command": "rag-assistant.openChat",
        "title": "Open RAG Chat",
        "category": "RAG"
      }
    ],
    "configuration": {
      "title": "RAG Assistant",
      "properties": {
        "rag-assistant.backendUrl": {
          "type": "string",
          "default": "http://localhost:8000",
          "description": "URL of the RAG backend service"
        },
        "rag-assistant.maxImages": {
          "type": "number",
          "default": 2,
          "description": "Maximum number of images to upload per message"
        }
      }
    },
    "viewsContainers": {
      "activitybar": [
        {
          "id": "rag-assistant",
          "title": "RAG Assistant",
          "icon": "media/icon.svg"
        }
      ]
    },
    "views": {
      "rag-assistant": [
        {
          "type": "webview",
          "id": "rag-assistant.chatView",
          "name": "Chat"
        }
      ]
    }
  },
  "scripts": {
    "vscode:prepublish": "npm run compile",
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "lint": "eslint src --ext ts",
    "test": "node ./out/test/runTest.js"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/vscode": "^1.85.0",
    "@typescript-eslint/eslint-plugin": "^6.0.0",
    "@typescript-eslint/parser": "^6.0.0",
    "eslint": "^8.0.0",
    "typescript": "^5.3.0"
  },
  "dependencies": {
    "marked": "^11.0.0"
  }
}
```

#### 4.1.1 Extension Structure

```
vscode-rag-extension/
├── package.json
├── tsconfig.json
├── src/
│   ├── extension.ts              # Extension entry point
│   ├── webview/
│   │   ├── ChatPanel.ts          # Webview panel management
│   │   ├── index.html            # Chat UI template
│   │   ├── styles.css            # UI styles
│   │   └── main.js               # Frontend logic
│   ├── services/
│   │   ├── BackendService.ts     # HTTP client for backend
│   │   ├── CopilotService.ts     # vscode.lm API wrapper
│   │   └── SkillRouter.ts        # Command/intent routing
│   ├── skills/
│   │   ├── types.ts              # Skill interfaces
│   │   └── registry.ts           # Skill definitions
│   └── utils/
│       ├── config.ts             # Configuration
│       └── logger.ts             # Logging
├── media/
│   └── icons/                    # Extension icons
└── README.md
```

#### 4.1.2 Backend Service Client

```typescript
// src/services/BackendService.ts

export interface SkillResult {
    text: string;
    images: string[];
    sources: Array<{
        title: string;
        path: string;
        type: string;
        url?: string;
    }>;
    metadata?: Record<string, any>;
}

export class BackendService {
    constructor(private baseUrl: string) {}

    async executeSkill(
        text: string,
        images?: Uint8Array[]
    ): Promise<SkillResult> {
        const formData = new FormData();

        // Parse command from text
        let skill = 'search';  // default
        let query = text;

        if (text.startsWith('/')) {
            const parts = text.split(/\s+/, 2);
            const cmd = parts[0].toLowerCase();
            query = parts[1] || '';

            // Map command to skill name
            const cmdMap: Record<string, string> = {
                '/search': 'search',
                '/s': 'search',
                '/confluence': 'confluence',
                '/cf': 'confluence',
                '/wiki': 'confluence',
                '/docs': 'docs',
                '/d': 'docs',
                '/analyze': 'analyze',
                '/a': 'analyze',
                '/explain-uml': 'explain-uml',
                '/uml': 'explain-uml',
                '/compare': 'compare',
                '/diff': 'compare',
                '/summary': 'summary',
                '/sum': 'summary',
                '/index': 'index',
                '/idx': 'index'
            };

            skill = cmdMap[cmd] || 'search';
        }

        formData.append('skill', skill);
        formData.append('query', query);

        // Add images if present
        if (images && images.length > 0) {
            images.forEach((imgBytes, i) => {
                const blob = new Blob([imgBytes], { type: 'image/png' });
                formData.append('images', blob, `image_${i}.png`);
            });
        }

        const response = await fetch(`${this.baseUrl}/api/skill`, {
            method: 'POST',
            body: formData
        });

        if (!response.ok) {
            const error = await response.json();
            throw new Error(error.detail || 'Backend request failed');
        }

        return await response.json();
    }

    async getCommands(): Promise<Array<{
        name: string;
        triggers: string[];
        description: string;
        requires_image: boolean;
    }>> {
        const response = await fetch(`${this.baseUrl}/api/commands`);
        if (!response.ok) {
            throw new Error('Failed to fetch commands');
        }
        return await response.json();
    }

    async getStatus(): Promise<{
        status: string;
        version: string;
        models_loaded: string[];
        document_count: number;
    }> {
        const response = await fetch(`${this.baseUrl}/api/status`);
        if (!response.ok) {
            throw new Error('Failed to fetch status');
        }
        return await response.json();
    }
}
```

#### 4.1.3 Key Interfaces

```typescript
// src/skills/types.ts

export interface SkillInput {
  query?: string;
  images?: Uint8Array[];
  options?: Record<string, unknown>;
}

export interface SkillOutput {
  text: string;                    // Context for Copilot
  images: string[];                // Image URLs for display
  sources: SourceReference[];      // Source links
  metadata?: Record<string, unknown>;
}

export interface SourceReference {
  title: string;
  path: string;
  type: 'confluence' | 'document' | 'image';
  url?: string;
}

export interface Skill {
  name: string;
  description: string;
  triggers: string[];              // e.g., ["/search", "/s"]
  keywords: string[];              // Natural language triggers
  requiresImage?: boolean;
  maxImages?: number;
}
```

#### 4.1.4 Copilot Integration

```typescript
// src/services/CopilotService.ts

import * as vscode from 'vscode';

export class CopilotService {
  private model: vscode.LanguageModelChat | null = null;

  async initialize(): Promise<void> {
    const models = await vscode.lm.selectChatModels({
      vendor: 'copilot',
      family: 'gpt-4'
    });

    if (models.length > 0) {
      this.model = models[0];
    } else {
      throw new Error('Copilot model not available');
    }
  }

  async chat(
    userQuery: string,
    ragContext: string,
    imageDescriptions?: string[]
  ): Promise<string> {
    if (!this.model) {
      throw new Error('Model not initialized');
    }

    // Compose prompt with RAG context
    let systemContext = `You are a helpful assistant with access to the following retrieved information:\n\n${ragContext}`;

    if (imageDescriptions?.length) {
      systemContext += `\n\nUser uploaded images with the following content:\n`;
      imageDescriptions.forEach((desc, i) => {
        systemContext += `\nImage ${i + 1}: ${desc}`;
      });
    }

    systemContext += `\n\nAnswer the user's question based on the above context. If the context doesn't contain relevant information, say so.`;

    const messages = [
      vscode.LanguageModelChatMessage.User(systemContext),
      vscode.LanguageModelChatMessage.User(userQuery)
    ];

    const response = await this.model.sendRequest(messages, {});

    let result = '';
    for await (const chunk of response.text) {
      result += chunk;
    }

    return result;
  }
}
```

#### 4.1.5 Extension Entry Point

```typescript
// src/extension.ts

import * as vscode from 'vscode';
import { ChatPanel } from './webview/ChatPanel';
import { CopilotService } from './services/CopilotService';
import { BackendService } from './services/BackendService';

let chatPanel: ChatPanel | undefined;

export async function activate(context: vscode.ExtensionContext) {
    console.log('RAG Assistant extension is now active');

    // Initialize services
    const config = vscode.workspace.getConfiguration('rag-assistant');
    const backendUrl = config.get<string>('backendUrl', 'http://localhost:8000');

    const backendService = new BackendService(backendUrl);
    const copilotService = new CopilotService();

    // Initialize Copilot
    try {
        await copilotService.initialize();
    } catch (e) {
        vscode.window.showWarningMessage(
            'Copilot not available. Some features may be limited.'
        );
    }

    // Register command to open chat
    const openChatCommand = vscode.commands.registerCommand(
        'rag-assistant.openChat',
        () => {
            if (chatPanel) {
                chatPanel.reveal();
            } else {
                chatPanel = new ChatPanel(
                    context.extensionUri,
                    backendService,
                    copilotService
                );

                chatPanel.onDidDispose(() => {
                    chatPanel = undefined;
                });
            }
        }
    );

    context.subscriptions.push(openChatCommand);

    // Auto-open chat panel on activation
    vscode.commands.executeCommand('rag-assistant.openChat');
}

export function deactivate() {
    if (chatPanel) {
        chatPanel.dispose();
    }
}
```

```typescript
// src/webview/ChatPanel.ts

import * as vscode from 'vscode';
import { BackendService } from '../services/BackendService';
import { CopilotService } from '../services/CopilotService';

export class ChatPanel {
    public static readonly viewType = 'ragChat';
    private readonly panel: vscode.WebviewPanel;
    private readonly extensionUri: vscode.Uri;
    private disposables: vscode.Disposable[] = [];

    constructor(
        extensionUri: vscode.Uri,
        private backendService: BackendService,
        private copilotService: CopilotService
    ) {
        this.extensionUri = extensionUri;

        this.panel = vscode.window.createWebviewPanel(
            ChatPanel.viewType,
            'RAG Chat',
            vscode.ViewColumn.Two,
            {
                enableScripts: true,
                retainContextWhenHidden: true,
                localResourceRoots: [extensionUri]
            }
        );

        this.panel.webview.html = this.getHtmlContent();

        // Handle messages from webview
        this.panel.webview.onDidReceiveMessage(
            async (message) => {
                switch (message.type) {
                    case 'chat':
                        await this.handleChatMessage(message);
                        break;
                    case 'getCommands':
                        await this.sendAvailableCommands();
                        break;
                }
            },
            null,
            this.disposables
        );

        this.panel.onDidDispose(() => this.dispose(), null, this.disposables);
    }

    private async handleChatMessage(message: {
        text: string;
        images?: string[];  // Base64 encoded
    }) {
        try {
            // Update UI: show loading
            this.panel.webview.postMessage({
                type: 'status',
                status: 'processing',
                text: 'Analyzing your request...'
            });

            // Convert base64 images to bytes for backend
            const imageBytes = message.images?.map(b64 =>
                Uint8Array.from(atob(b64), c => c.charCodeAt(0))
            );

            // Call backend skill
            this.panel.webview.postMessage({
                type: 'status',
                status: 'processing',
                text: 'Searching knowledge base...'
            });

            const skillResult = await this.backendService.executeSkill(
                message.text,
                imageBytes
            );

            // If VLM processing needed, update status
            if (skillResult.metadata?.processingImages) {
                this.panel.webview.postMessage({
                    type: 'status',
                    status: 'processing',
                    text: 'Analyzing images (this may take 15-30 seconds)...'
                });
            }

            // Call Copilot with context
            this.panel.webview.postMessage({
                type: 'status',
                status: 'processing',
                text: 'Generating response...'
            });

            const response = await this.copilotService.chat(
                message.text,
                skillResult.text,
                skillResult.metadata?.imageDescriptions
            );

            // Send response to webview
            this.panel.webview.postMessage({
                type: 'response',
                text: response,
                images: skillResult.images,
                sources: skillResult.sources
            });

        } catch (error) {
            this.panel.webview.postMessage({
                type: 'error',
                text: `Error: ${error instanceof Error ? error.message : 'Unknown error'}`
            });
        } finally {
            this.panel.webview.postMessage({
                type: 'status',
                status: 'idle'
            });
        }
    }

    private async sendAvailableCommands() {
        try {
            const commands = await this.backendService.getCommands();
            this.panel.webview.postMessage({
                type: 'commands',
                commands
            });
        } catch (e) {
            console.error('Failed to fetch commands:', e);
        }
    }

    private getHtmlContent(): string {
        const styleUri = this.panel.webview.asWebviewUri(
            vscode.Uri.joinPath(this.extensionUri, 'media', 'styles.css')
        );
        const scriptUri = this.panel.webview.asWebviewUri(
            vscode.Uri.joinPath(this.extensionUri, 'media', 'main.js')
        );

        return `<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="Content-Security-Policy" content="default-src 'none'; style-src ${this.panel.webview.cspSource} 'unsafe-inline'; script-src ${this.panel.webview.cspSource}; img-src ${this.panel.webview.cspSource} data: https:;">
    <link rel="stylesheet" href="${styleUri}">
    <title>RAG Chat</title>
</head>
<body>
    <div id="app">
        <div id="command-bar" class="hidden"></div>
        <div id="chat-history"></div>
        <div id="image-preview" class="hidden">
            <div id="image-list"></div>
            <span class="image-hint">Max 2 images</span>
        </div>
        <div id="input-area">
            <textarea id="user-input" placeholder="Ask a question or type / for commands..." rows="1"></textarea>
            <button id="attach-btn" title="Attach image">📎</button>
            <button id="send-btn" title="Send">➤</button>
            <input type="file" id="file-input" accept="image/*" multiple hidden>
        </div>
        <div id="status" class="hidden">
            <span class="spinner"></span>
            <span id="status-text">Processing...</span>
        </div>
    </div>
    <script src="${scriptUri}"></script>
</body>
</html>`;
    }

    public reveal() {
        this.panel.reveal();
    }

    public dispose() {
        this.panel.dispose();
        this.disposables.forEach(d => d.dispose());
    }

    public onDidDispose(callback: () => void) {
        this.panel.onDidDispose(callback);
    }
}
```

#### 4.1.6 Webview Frontend Logic

```javascript
// media/main.js

(function() {
    const vscode = acquireVsCodeApi();

    // State
    let attachedImages = [];
    let commands = [];
    let isProcessing = false;

    // DOM Elements
    const chatHistory = document.getElementById('chat-history');
    const userInput = document.getElementById('user-input');
    const sendBtn = document.getElementById('send-btn');
    const attachBtn = document.getElementById('attach-btn');
    const fileInput = document.getElementById('file-input');
    const imagePreview = document.getElementById('image-preview');
    const imageList = document.getElementById('image-list');
    const commandBar = document.getElementById('command-bar');
    const statusDiv = document.getElementById('status');
    const statusText = document.getElementById('status-text');

    // Initialize
    vscode.postMessage({ type: 'getCommands' });

    // Event Listeners
    sendBtn.addEventListener('click', sendMessage);
    userInput.addEventListener('keydown', (e) => {
        if (e.key === 'Enter' && !e.shiftKey) {
            e.preventDefault();
            sendMessage();
        }
    });

    userInput.addEventListener('input', () => {
        const value = userInput.value;
        if (value.startsWith('/') && value.length > 0) {
            showCommandSuggestions(value);
        } else {
            hideCommandSuggestions();
        }
        autoResizeTextarea();
    });

    attachBtn.addEventListener('click', () => fileInput.click());

    fileInput.addEventListener('change', handleFileSelect);

    // Handle messages from extension
    window.addEventListener('message', (event) => {
        const message = event.data;

        switch (message.type) {
            case 'response':
                addMessage('assistant', message.text, message.images, message.sources);
                break;
            case 'error':
                addMessage('error', message.text);
                break;
            case 'status':
                updateStatus(message.status, message.text);
                break;
            case 'commands':
                commands = message.commands;
                break;
        }
    });

    function sendMessage() {
        const text = userInput.value.trim();
        if (!text && attachedImages.length === 0) return;
        if (isProcessing) return;

        // Add user message to chat
        addMessage('user', text, attachedImages.map(img => img.preview));

        // Send to extension
        vscode.postMessage({
            type: 'chat',
            text: text,
            images: attachedImages.map(img => img.base64)
        });

        // Clear input
        userInput.value = '';
        clearAttachedImages();
        autoResizeTextarea();
    }

    function addMessage(role, text, images = [], sources = []) {
        const messageDiv = document.createElement('div');
        messageDiv.className = `message ${role}`;

        // Text content (render markdown for assistant)
        const textDiv = document.createElement('div');
        textDiv.className = 'message-text';
        if (role === 'assistant') {
            textDiv.innerHTML = marked.parse(text);
        } else {
            textDiv.textContent = text;
        }
        messageDiv.appendChild(textDiv);

        // Images
        if (images && images.length > 0) {
            const imagesDiv = document.createElement('div');
            imagesDiv.className = 'message-images';
            images.forEach(src => {
                const img = document.createElement('img');
                img.src = src;
                img.className = 'message-image';
                img.addEventListener('click', () => {
                    // Open image in new tab or modal
                    window.open(src, '_blank');
                });
                imagesDiv.appendChild(img);
            });
            messageDiv.appendChild(imagesDiv);
        }

        // Sources
        if (sources && sources.length > 0) {
            const sourcesDiv = document.createElement('div');
            sourcesDiv.className = 'message-sources';
            sourcesDiv.innerHTML = '<strong>Sources:</strong>';
            const sourceList = document.createElement('ul');
            sources.forEach(source => {
                const li = document.createElement('li');
                if (source.url) {
                    const a = document.createElement('a');
                    a.href = source.url;
                    a.textContent = source.title;
                    a.target = '_blank';
                    li.appendChild(a);
                } else {
                    li.textContent = `${source.title} (${source.path})`;
                }
                sourceList.appendChild(li);
            });
            sourcesDiv.appendChild(sourceList);
            messageDiv.appendChild(sourcesDiv);
        }

        chatHistory.appendChild(messageDiv);
        chatHistory.scrollTop = chatHistory.scrollHeight;
    }

    function handleFileSelect(e) {
        const files = Array.from(e.target.files);
        const maxImages = 2;

        if (attachedImages.length + files.length > maxImages) {
            alert(`Maximum ${maxImages} images allowed`);
            return;
        }

        files.forEach(file => {
            if (!file.type.startsWith('image/')) return;
            if (file.size > 5 * 1024 * 1024) {
                alert('Image must be smaller than 5MB');
                return;
            }

            const reader = new FileReader();
            reader.onload = (e) => {
                const base64 = e.target.result.split(',')[1];
                attachedImages.push({
                    name: file.name,
                    preview: e.target.result,
                    base64: base64
                });
                updateImagePreview();
            };
            reader.readAsDataURL(file);
        });

        fileInput.value = '';
    }

    function updateImagePreview() {
        if (attachedImages.length === 0) {
            imagePreview.classList.add('hidden');
            return;
        }

        imagePreview.classList.remove('hidden');
        imageList.innerHTML = '';

        attachedImages.forEach((img, index) => {
            const container = document.createElement('div');
            container.className = 'preview-image-container';

            const imgEl = document.createElement('img');
            imgEl.src = img.preview;
            imgEl.className = 'preview-image';

            const removeBtn = document.createElement('button');
            removeBtn.className = 'remove-image';
            removeBtn.textContent = '×';
            removeBtn.onclick = () => {
                attachedImages.splice(index, 1);
                updateImagePreview();
            };

            container.appendChild(imgEl);
            container.appendChild(removeBtn);
            imageList.appendChild(container);
        });
    }

    function clearAttachedImages() {
        attachedImages = [];
        updateImagePreview();
    }

    function showCommandSuggestions(input) {
        const query = input.slice(1).toLowerCase();
        const filtered = commands.filter(cmd =>
            cmd.triggers.some(t => t.slice(1).startsWith(query))
        );

        if (filtered.length === 0) {
            hideCommandSuggestions();
            return;
        }

        commandBar.innerHTML = '';
        filtered.forEach(cmd => {
            const item = document.createElement('div');
            item.className = 'command-item';
            item.innerHTML = `
                <span class="cmd">${cmd.triggers[0]}</span>
                <span class="desc">${cmd.description}</span>
                ${cmd.requires_image ? '<span class="badge">📎</span>' : ''}
            `;
            item.onclick = () => {
                userInput.value = cmd.triggers[0] + ' ';
                hideCommandSuggestions();
                userInput.focus();
            };
            commandBar.appendChild(item);
        });

        commandBar.classList.remove('hidden');
    }

    function hideCommandSuggestions() {
        commandBar.classList.add('hidden');
    }

    function updateStatus(status, text) {
        if (status === 'processing') {
            isProcessing = true;
            statusDiv.classList.remove('hidden');
            statusText.textContent = text;
            sendBtn.disabled = true;
        } else {
            isProcessing = false;
            statusDiv.classList.add('hidden');
            sendBtn.disabled = false;
        }
    }

    function autoResizeTextarea() {
        userInput.style.height = 'auto';
        userInput.style.height = Math.min(userInput.scrollHeight, 150) + 'px';
    }
})();
```

#### 4.1.7 Webview Chat UI HTML Template

```html
<!-- src/webview/index.html (alternative standalone template) -->
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>RAG Chat</title>
  <link rel="stylesheet" href="${styleUri}">
</head>
<body>
  <div id="app">
    <!-- Command Suggestions -->
    <div id="command-bar" class="hidden">
      <div class="command-item" data-cmd="/search">
        <span class="cmd">/search</span>
        <span class="desc">Search all sources</span>
      </div>
      <div class="command-item" data-cmd="/confluence">
        <span class="cmd">/confluence</span>
        <span class="desc">Search Confluence only</span>
      </div>
      <div class="command-item" data-cmd="/explain-uml">
        <span class="cmd">/explain-uml</span>
        <span class="desc">Explain UML diagram</span>
      </div>
      <div class="command-item" data-cmd="/compare">
        <span class="cmd">/compare</span>
        <span class="desc">Compare two images</span>
      </div>
      <div class="command-item" data-cmd="/summary">
        <span class="cmd">/summary</span>
        <span class="desc">Summarize a page</span>
      </div>
    </div>

    <!-- Chat History -->
    <div id="chat-history">
      <!-- Messages will be inserted here -->
    </div>

    <!-- Image Preview Area -->
    <div id="image-preview" class="hidden">
      <div id="image-list"></div>
      <span class="image-hint">Max 2 images</span>
    </div>

    <!-- Input Area -->
    <div id="input-area">
      <textarea
        id="user-input"
        placeholder="Ask a question or type / for commands..."
        rows="1"
      ></textarea>
      <button id="attach-btn" title="Attach image">📎</button>
      <button id="send-btn" title="Send">➤</button>
      <input type="file" id="file-input" accept="image/*" multiple hidden>
    </div>

    <!-- Processing Status -->
    <div id="status" class="hidden">
      <span class="spinner"></span>
      <span id="status-text">Processing...</span>
    </div>
  </div>

  <script src="${scriptUri}"></script>
</body>
</html>
```

#### 4.1.8 Webview Styles

```css
/* media/styles.css */

:root {
    --bg-primary: var(--vscode-editor-background);
    --bg-secondary: var(--vscode-sideBar-background);
    --text-primary: var(--vscode-editor-foreground);
    --text-secondary: var(--vscode-descriptionForeground);
    --border-color: var(--vscode-panel-border);
    --accent-color: var(--vscode-button-background);
    --accent-hover: var(--vscode-button-hoverBackground);
    --user-msg-bg: var(--vscode-button-background);
    --assistant-msg-bg: var(--vscode-editor-inactiveSelectionBackground);
    --error-color: var(--vscode-errorForeground);
}

* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

body {
    font-family: var(--vscode-font-family);
    font-size: var(--vscode-font-size);
    background: var(--bg-primary);
    color: var(--text-primary);
    height: 100vh;
    overflow: hidden;
}

#app {
    display: flex;
    flex-direction: column;
    height: 100%;
    padding: 10px;
}

/* Command Bar */
#command-bar {
    position: absolute;
    bottom: 80px;
    left: 10px;
    right: 10px;
    background: var(--bg-secondary);
    border: 1px solid var(--border-color);
    border-radius: 6px;
    max-height: 200px;
    overflow-y: auto;
    z-index: 100;
}

#command-bar.hidden {
    display: none;
}

.command-item {
    padding: 8px 12px;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 10px;
}

.command-item:hover {
    background: var(--vscode-list-hoverBackground);
}

.command-item .cmd {
    font-family: monospace;
    color: var(--accent-color);
    min-width: 120px;
}

.command-item .desc {
    color: var(--text-secondary);
    flex: 1;
}

.command-item .badge {
    font-size: 12px;
}

/* Chat History */
#chat-history {
    flex: 1;
    overflow-y: auto;
    padding: 10px 0;
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.message {
    max-width: 85%;
    padding: 10px 14px;
    border-radius: 12px;
    line-height: 1.5;
}

.message.user {
    align-self: flex-end;
    background: var(--user-msg-bg);
    color: var(--vscode-button-foreground);
    border-bottom-right-radius: 4px;
}

.message.assistant {
    align-self: flex-start;
    background: var(--assistant-msg-bg);
    border-bottom-left-radius: 4px;
}

.message.error {
    align-self: center;
    background: var(--vscode-inputValidation-errorBackground);
    color: var(--error-color);
    border: 1px solid var(--error-color);
}

.message-text {
    word-wrap: break-word;
}

.message-text code {
    background: rgba(0,0,0,0.2);
    padding: 2px 6px;
    border-radius: 4px;
    font-family: var(--vscode-editor-font-family);
}

.message-text pre {
    background: rgba(0,0,0,0.3);
    padding: 10px;
    border-radius: 6px;
    overflow-x: auto;
    margin: 8px 0;
}

.message-images {
    display: flex;
    gap: 8px;
    margin-top: 8px;
    flex-wrap: wrap;
}

.message-image {
    max-width: 200px;
    max-height: 150px;
    border-radius: 6px;
    cursor: pointer;
    transition: transform 0.2s;
}

.message-image:hover {
    transform: scale(1.05);
}

.message-sources {
    margin-top: 10px;
    padding-top: 8px;
    border-top: 1px solid var(--border-color);
    font-size: 0.9em;
    color: var(--text-secondary);
}

.message-sources ul {
    margin-top: 4px;
    padding-left: 20px;
}

.message-sources a {
    color: var(--vscode-textLink-foreground);
}

/* Image Preview */
#image-preview {
    padding: 8px;
    background: var(--bg-secondary);
    border-radius: 6px;
    margin-bottom: 8px;
}

#image-preview.hidden {
    display: none;
}

#image-list {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
}

.preview-image-container {
    position: relative;
    display: inline-block;
}

.preview-image {
    width: 60px;
    height: 60px;
    object-fit: cover;
    border-radius: 6px;
}

.remove-image {
    position: absolute;
    top: -6px;
    right: -6px;
    width: 20px;
    height: 20px;
    border-radius: 50%;
    background: var(--error-color);
    color: white;
    border: none;
    cursor: pointer;
    font-size: 14px;
    line-height: 1;
}

.image-hint {
    color: var(--text-secondary);
    font-size: 0.85em;
    margin-left: 8px;
}

/* Input Area */
#input-area {
    display: flex;
    gap: 8px;
    align-items: flex-end;
    padding: 10px;
    background: var(--bg-secondary);
    border-radius: 8px;
}

#user-input {
    flex: 1;
    background: var(--vscode-input-background);
    color: var(--vscode-input-foreground);
    border: 1px solid var(--vscode-input-border);
    border-radius: 6px;
    padding: 8px 12px;
    font-family: inherit;
    font-size: inherit;
    resize: none;
    min-height: 36px;
    max-height: 150px;
}

#user-input:focus {
    outline: none;
    border-color: var(--accent-color);
}

#attach-btn, #send-btn {
    background: var(--accent-color);
    color: var(--vscode-button-foreground);
    border: none;
    border-radius: 6px;
    width: 36px;
    height: 36px;
    cursor: pointer;
    font-size: 16px;
    transition: background 0.2s;
}

#attach-btn:hover, #send-btn:hover {
    background: var(--accent-hover);
}

#send-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
}

/* Status */
#status {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 8px 12px;
    background: var(--bg-secondary);
    border-radius: 6px;
    margin-top: 8px;
    color: var(--text-secondary);
}

#status.hidden {
    display: none;
}

.spinner {
    width: 16px;
    height: 16px;
    border: 2px solid var(--border-color);
    border-top-color: var(--accent-color);
    border-radius: 50%;
    animation: spin 1s linear infinite;
}

@keyframes spin {
    to { transform: rotate(360deg); }
}
```

---

### 4.2 Backend Service

#### 4.2.1 Project Structure

```
rag-backend/
├── pyproject.toml
├── requirements.txt
├── config/
│   ├── __init__.py
│   └── settings.py               # Configuration management
├── app/
│   ├── __init__.py
│   ├── main.py                   # FastAPI application
│   ├── api/
│   │   ├── __init__.py
│   │   ├── routes.py             # API endpoints
│   │   └── dependencies.py       # Dependency injection
│   ├── skills/
│   │   ├── __init__.py
│   │   ├── base.py               # Base skill class
│   │   ├── router.py             # Skill routing
│   │   ├── search_skill.py       # Global search
│   │   ├── confluence_skill.py   # Confluence search
│   │   ├── docs_skill.py         # Local docs search
│   │   ├── analyze_skill.py      # Image analysis
│   │   ├── explain_uml_skill.py  # UML explanation
│   │   ├── compare_skill.py      # Image comparison
│   │   ├── summary_skill.py      # Page summary
│   │   └── index_skill.py        # Index refresh
│   ├── services/
│   │   ├── __init__.py
│   │   ├── document_parser.py    # Document parsing
│   │   ├── image_processor.py    # Image processing pipeline
│   │   ├── embedding_service.py  # Text embedding
│   │   ├── vector_store.py       # Qdrant operations
│   │   ├── confluence_client.py  # Confluence API client
│   │   └── confluence_index.py   # Confluence index manager
│   ├── models/
│   │   ├── __init__.py
│   │   ├── vlm.py                # Vision language model
│   │   ├── ocr.py                # OCR service
│   │   └── clip.py               # CLIP classifier
│   └── scheduler/
│       ├── __init__.py
│       └── tasks.py              # Scheduled tasks
├── scripts/
│   ├── download_models.py        # Model download script
│   └── init_db.py                # Database initialization
└── tests/
    └── ...
```

#### 4.2.2 Configuration

```python
# config/settings.py

from pydantic_settings import BaseSettings
from pathlib import Path

class Settings(BaseSettings):
    # Server
    host: str = "0.0.0.0"
    port: int = 8000
    debug: bool = False

    # Paths
    model_dir: Path = Path("/opt/rag-models")
    data_dir: Path = Path("/opt/rag-data")
    upload_dir: Path = Path("/opt/rag-data/uploads")

    # Confluence
    confluence_base_url: str = ""
    confluence_username: str = ""
    confluence_api_token: str = ""
    confluence_sync_interval_hours: int = 4

    # Models
    embedding_model: str = "bge-base-en-v1.5"
    vlm_model_name: str = "Qwen/Qwen2-VL-2B-Instruct"  # or "vikhyatk/moondream2"

    # Processing
    max_upload_images: int = 2
    max_image_size_mb: int = 5
    chunk_size: int = 512
    chunk_overlap: int = 50

    # Vector Store
    qdrant_path: Path = Path("/opt/rag-data/qdrant")
    collection_name: str = "documents"

    class Config:
        env_file = ".env"
        env_prefix = "RAG_"

settings = Settings()
```

#### 4.2.3 FastAPI Application Entry Point

```python
# app/main.py

from contextlib import asynccontextmanager
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from fastapi.staticfiles import StaticFiles
import logging

from config.settings import settings
from app.api.routes import router
from app.services.embedding_service import EmbeddingService
from app.services.vector_store import VectorStore
from app.services.confluence_client import ConfluenceClient
from app.services.confluence_index import ConfluenceIndex
from app.models.clip import CLIPClassifier
from app.models.vlm import VisionLanguageModel
from app.models.ocr import OCRService
from app.skills.router import SkillRouter
from app.skills import register_skills
from app.scheduler.tasks import setup_scheduler

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/opt/rag/logs/app.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

# Global services container
services = {}

@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan manager for startup/shutdown."""
    # Startup
    logger.info("Starting RAG Backend Service...")

    # Initialize AI models
    logger.info("Loading embedding model...")
    services['embedding_service'] = EmbeddingService()

    logger.info("Loading CLIP classifier...")
    services['clip_classifier'] = CLIPClassifier()

    logger.info("Loading VLM (this may take a while on CPU)...")
    services['vlm'] = VisionLanguageModel()

    logger.info("Loading OCR service...")
    services['ocr_service'] = OCRService()

    # Initialize vector store
    logger.info("Initializing vector store...")
    services['vector_store'] = VectorStore(services['embedding_service'])

    # Initialize image store and processor
    logger.info("Initializing image services...")
    from app.services.image_store import ImageStore
    from app.services.image_processor import ImageProcessor
    from app.services.document_parser import DocumentParser
    from app.services.document_service import DocumentService

    services['image_store'] = ImageStore()
    services['image_processor'] = ImageProcessor(
        services['clip_classifier'],
        services['ocr_service'],
        services['vlm']
    )
    services['document_parser'] = DocumentParser()
    services['document_service'] = DocumentService(
        services['document_parser'],
        services['image_processor'],
        services['embedding_service'],
        services['vector_store'],
        services['image_store']
    )

    # Initialize Confluence
    logger.info("Initializing Confluence client...")
    services['confluence_client'] = ConfluenceClient()
    services['confluence_index'] = ConfluenceIndex(services['confluence_client'])

    # Initialize skill router
    logger.info("Registering skills...")
    services['skill_router'] = SkillRouter()
    register_skills(services['skill_router'], services)

    # Setup scheduler for periodic tasks
    scheduler = setup_scheduler(services)
    scheduler.start()
    services['scheduler'] = scheduler

    logger.info("RAG Backend Service started successfully!")

    yield

    # Shutdown
    logger.info("Shutting down RAG Backend Service...")
    services['scheduler'].shutdown()
    logger.info("Shutdown complete.")


# Create FastAPI application
app = FastAPI(
    title="RAG Tool Backend",
    description="Backend service for RAG-based document retrieval",
    version="1.0.0",
    lifespan=lifespan
)

# CORS middleware for VS Code extension
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],  # VS Code extension
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Static files for serving images
app.mount("/static", StaticFiles(directory=str(settings.data_dir / "images")), name="static")

# Include API routes
app.include_router(router)

# Dependency injection helper
def get_services():
    return services
```

#### 4.2.4 API Endpoints

```python
# app/api/routes.py

from fastapi import APIRouter, UploadFile, File, Form, HTTPException
from typing import List, Optional
from pydantic import BaseModel

router = APIRouter()

# Request/Response Models
class SkillRequest(BaseModel):
    skill: str
    query: Optional[str] = None
    options: dict = {}

class SkillResponse(BaseModel):
    text: str
    images: List[str] = []
    sources: List[dict] = []
    metadata: dict = {}

class IngestRequest(BaseModel):
    file_path: str
    source_type: str  # 'pdf', 'word', 'excel', 'ppt'

class StatusResponse(BaseModel):
    status: str
    version: str
    models_loaded: List[str]
    confluence_last_sync: Optional[str]
    document_count: int
    image_count: int


# Endpoints
@router.post("/api/skill", response_model=SkillResponse)
async def execute_skill(
    request: SkillRequest,
    images: Optional[List[UploadFile]] = File(None)
):
    """Execute a skill with optional images."""
    # Validate image count
    if images and len(images) > settings.max_upload_images:
        raise HTTPException(
            status_code=400,
            detail=f"Maximum {settings.max_upload_images} images allowed"
        )

    # Route to skill handler
    skill_handler = skill_router.get_skill(request.skill)
    if not skill_handler:
        raise HTTPException(status_code=404, detail=f"Skill '{request.skill}' not found")

    # Read image bytes
    image_bytes = []
    if images:
        for img in images:
            content = await img.read()
            if len(content) > settings.max_image_size_mb * 1024 * 1024:
                raise HTTPException(status_code=400, detail="Image too large")
            image_bytes.append(content)

    # Execute skill
    result = await skill_handler.execute(
        query=request.query,
        images=image_bytes,
        options=request.options
    )

    return SkillResponse(**result.dict())


@router.post("/api/ingest")
async def ingest_document(request: IngestRequest):
    """Ingest a document into the vector store."""
    return await document_service.ingest(request.file_path, request.source_type)


@router.post("/api/ingest/upload")
async def upload_and_ingest(
    file: UploadFile = File(...),
    source_type: str = Form(...)
):
    """Upload and ingest a document."""
    # Save uploaded file
    file_path = settings.upload_dir / file.filename
    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)

    # Ingest
    return await document_service.ingest(str(file_path), source_type)


@router.post("/api/confluence/sync")
async def trigger_confluence_sync():
    """Manually trigger Confluence index sync."""
    await confluence_index.sync()
    return {"status": "sync_completed"}


@router.get("/api/status", response_model=StatusResponse)
async def get_status():
    """Get system status."""
    from app.main import services
    return StatusResponse(
        status="healthy",
        version="1.0.0",
        models_loaded=["embedding", "clip", "vlm", "ocr"],
        confluence_last_sync=services['confluence_index'].last_sync_time,
        document_count=services['vector_store'].count(),
        image_count=services['image_store'].count()
    )


@router.get("/api/images/{image_id}")
async def get_image(image_id: str):
    """Serve stored images."""
    from fastapi.responses import FileResponse
    image_path = settings.data_dir / "images" / image_id
    if not image_path.exists():
        raise HTTPException(status_code=404, detail="Image not found")
    return FileResponse(image_path)


@router.get("/api/commands")
async def get_commands():
    """Get available skill commands for UI."""
    return skill_router.get_available_commands()


# Error Handlers
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """Global exception handler for unhandled errors."""
    import traceback
    logger.error(f"Unhandled error: {exc}\n{traceback.format_exc()}")
    return JSONResponse(
        status_code=500,
        content={"detail": "Internal server error", "error": str(exc)}
    )
```

#### 4.2.5 Skill Base Class

```python
# app/skills/base.py

from abc import ABC, abstractmethod
from typing import List, Optional
from pydantic import BaseModel

class SkillInput(BaseModel):
    query: Optional[str] = None
    images: Optional[List[bytes]] = None
    options: dict = {}

class SourceReference(BaseModel):
    title: str
    path: str
    type: str  # 'confluence', 'document', 'image'
    url: Optional[str] = None

class SkillOutput(BaseModel):
    text: str
    images: List[str] = []
    sources: List[SourceReference] = []
    metadata: dict = {}

class BaseSkill(ABC):
    """Base class for all skills."""

    name: str
    description: str
    triggers: List[str]       # Command triggers e.g., ["/search", "/s"]
    keywords: List[str]       # Natural language keywords
    requires_image: bool = False
    max_images: int = 0

    def __init__(self, services: dict):
        """
        Args:
            services: Dict containing shared services like
                      embedding_service, vector_store, vlm, etc.
        """
        self.services = services

    @abstractmethod
    async def execute(self, input: SkillInput) -> SkillOutput:
        """Execute the skill logic."""
        pass

    def validate_input(self, input: SkillInput) -> None:
        """Validate skill input."""
        if self.requires_image and not input.images:
            raise ValueError(f"Skill '{self.name}' requires at least one image")

        if input.images and len(input.images) > self.max_images:
            raise ValueError(f"Skill '{self.name}' accepts max {self.max_images} images")
```

#### 4.2.6 Skill Implementations

```python
# app/skills/search_skill.py

from .base import BaseSkill, SkillInput, SkillOutput, SourceReference

class SearchSkill(BaseSkill):
    """Global search across all sources."""

    name = "search"
    description = "Search all document sources"
    triggers = ["/search", "/s"]
    keywords = ["search", "find", "look for", "搜索", "查找"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        embedding_service = self.services['embedding_service']
        vector_store = self.services['vector_store']
        confluence_skill = self.services['confluence_skill']

        # 1. Search local vector store
        query_embedding = embedding_service.embed(input.query)
        local_results = vector_store.search(
            embedding=query_embedding,
            limit=5
        )

        # 2. Search Confluence index
        confluence_results = await confluence_skill.search_index(input.query, limit=3)

        # 3. Fetch Confluence content for matched pages
        confluence_contents = await confluence_skill.fetch_contents(
            [r.page_id for r in confluence_results]
        )

        # 4. Combine results
        context_parts = []
        sources = []
        images = []

        for result in local_results:
            context_parts.append(f"[Document: {result.metadata['title']}]\n{result.text}")
            sources.append(SourceReference(
                title=result.metadata['title'],
                path=result.metadata['path'],
                type='document'
            ))
            if result.metadata.get('image_path'):
                images.append(result.metadata['image_path'])

        for page_id, content in confluence_contents.items():
            context_parts.append(f"[Confluence: {content['title']}]\n{content['body']}")
            sources.append(SourceReference(
                title=content['title'],
                path=content['path'],
                type='confluence',
                url=content['url']
            ))

        return SkillOutput(
            text="\n\n---\n\n".join(context_parts),
            images=images,
            sources=sources
        )
```

```python
# app/skills/confluence_skill.py

from .base import BaseSkill, SkillInput, SkillOutput, SourceReference
import asyncio

class ConfluenceSkill(BaseSkill):
    """Search Confluence with index + realtime fetch."""

    name = "confluence"
    description = "Search Confluence wiki"
    triggers = ["/confluence", "/cf", "/wiki"]
    keywords = ["confluence", "wiki", "文档"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        confluence_index = self.services['confluence_index']
        confluence_client = self.services['confluence_client']

        # 1. Search in local index
        matched_pages = confluence_index.search(
            query=input.query,
            fields=['title', 'path', 'labels'],
            limit=5
        )

        if not matched_pages:
            return SkillOutput(
                text="No matching Confluence pages found.",
                sources=[]
            )

        # 2. Fetch content from Confluence API (parallel)
        async def fetch_page(page):
            content = await confluence_client.get_page_content(page['id'])
            return page['id'], content

        tasks = [fetch_page(p) for p in matched_pages[:3]]  # Limit to top 3
        results = await asyncio.gather(*tasks)

        # 3. Process and format
        context_parts = []
        sources = []

        for page_id, content in results:
            page_meta = next(p for p in matched_pages if p['id'] == page_id)

            # Extract relevant sections based on query
            relevant_text = self._extract_relevant(content['body'], input.query)

            context_parts.append(
                f"[{page_meta['title']}]\n"
                f"Path: {page_meta['path']}\n\n"
                f"{relevant_text}"
            )

            sources.append(SourceReference(
                title=page_meta['title'],
                path=page_meta['path'],
                type='confluence',
                url=f"{confluence_client.base_url}/wiki{page_meta['path']}"
            ))

        return SkillOutput(
            text="\n\n---\n\n".join(context_parts),
            images=[],  # Confluence images are inline in content
            sources=sources
        )

    def _extract_relevant(self, body: str, query: str, max_chars: int = 2000) -> str:
        """Extract most relevant sections from page body."""
        # Simple implementation: find paragraphs containing query terms
        paragraphs = body.split('\n\n')
        query_terms = query.lower().split()

        scored = []
        for p in paragraphs:
            score = sum(1 for term in query_terms if term in p.lower())
            if score > 0:
                scored.append((score, p))

        scored.sort(reverse=True)

        result = []
        total_chars = 0
        for _, p in scored:
            if total_chars + len(p) > max_chars:
                break
            result.append(p)
            total_chars += len(p)

        return '\n\n'.join(result) if result else body[:max_chars]
```

```python
# app/skills/explain_uml_skill.py

from .base import BaseSkill, SkillInput, SkillOutput, SourceReference

class ExplainUMLSkill(BaseSkill):
    """Explain UML diagrams using VLM."""

    name = "explain-uml"
    description = "Explain UML diagram content"
    triggers = ["/explain-uml", "/uml"]
    keywords = ["UML", "diagram", "class diagram", "sequence diagram",
                "flowchart", "类图", "时序图", "流程图"]
    requires_image = True
    max_images = 1

    UML_PROMPT = """Analyze this UML diagram in detail. Provide:

1. **Diagram Type**: Identify if this is a class diagram, sequence diagram,
   activity diagram, state diagram, component diagram, or other type.

2. **Elements**: List all major elements:
   - For class diagrams: classes, interfaces, attributes, methods
   - For sequence diagrams: actors, objects, lifelines
   - For activity diagrams: activities, decisions, forks/joins
   - For component diagrams: components, interfaces, dependencies

3. **Relationships**: Describe all relationships:
   - Inheritance, implementation, association, aggregation, composition
   - Message flows, dependencies, transitions

4. **Purpose**: Explain what this diagram represents in terms of:
   - System architecture or design
   - Business process or workflow
   - Data flow or state transitions

5. **Key Insights**: Note any design patterns, potential issues,
   or important architectural decisions visible in the diagram.

Format your response in clear sections with headers."""

    async def execute(self, input: SkillInput) -> SkillOutput:
        vlm = self.services['vlm']

        if not input.images:
            return SkillOutput(
                text="Please upload a UML diagram image to analyze.",
                sources=[]
            )

        # Process image with VLM
        description = await vlm.describe(
            image=input.images[0],
            prompt=self.UML_PROMPT
        )

        # Add user's specific question context if provided
        if input.query and input.query.strip():
            context = f"""[UML Diagram Analysis]

{description}

---
User's specific question: {input.query}
"""
        else:
            context = f"[UML Diagram Analysis]\n\n{description}"

        return SkillOutput(
            text=context,
            images=[],
            sources=[],
            metadata={"diagram_analyzed": True}
        )
```

```python
# app/skills/compare_skill.py

from .base import BaseSkill, SkillInput, SkillOutput

class CompareSkill(BaseSkill):
    """Compare two images."""

    name = "compare"
    description = "Compare two diagrams or images"
    triggers = ["/compare", "/diff"]
    keywords = ["compare", "difference", "对比", "区别", "变化"]
    requires_image = True
    max_images = 2

    COMPARE_PROMPT = """Compare these two images and describe:

1. **Similarities**: What elements, structures, or content are the same?

2. **Differences**: What has changed between the two images?
   - Added elements
   - Removed elements
   - Modified elements
   - Structural changes

3. **Summary**: Provide a brief summary of how the two images differ.

Be specific and reference concrete elements in each image."""

    async def execute(self, input: SkillInput) -> SkillOutput:
        vlm = self.services['vlm']

        if not input.images or len(input.images) < 2:
            return SkillOutput(
                text="Please upload exactly 2 images to compare.",
                sources=[]
            )

        # Describe each image first
        desc1 = await vlm.describe(input.images[0], "Describe this image in detail.")
        desc2 = await vlm.describe(input.images[1], "Describe this image in detail.")

        # Compare
        comparison_context = f"""[Image Comparison]

**Image 1 Description:**
{desc1}

**Image 2 Description:**
{desc2}

---
Please compare these two images based on the descriptions above.
"""

        return SkillOutput(
            text=comparison_context,
            images=[],
            sources=[],
            metadata={"images_compared": 2}
        )
```

#### 4.2.7 Additional Skill Implementations

```python
# app/skills/analyze_skill.py

from .base import BaseSkill, SkillInput, SkillOutput

class AnalyzeSkill(BaseSkill):
    """General image analysis skill."""

    name = "analyze"
    description = "Analyze uploaded image content"
    triggers = ["/analyze", "/a"]
    keywords = ["analyze", "what is", "explain", "describe", "分析", "解释"]
    requires_image = True
    max_images = 2

    async def execute(self, input: SkillInput) -> SkillOutput:
        vlm = self.services['vlm']
        ocr = self.services['ocr_service']
        clip = self.services['clip_classifier']

        if not input.images:
            return SkillOutput(
                text="Please upload an image to analyze.",
                sources=[]
            )

        descriptions = []
        for i, img_bytes in enumerate(input.images):
            # Classify image type first
            img_type, confidence = clip.classify(img_bytes)

            if img_type == 'text':
                # Use OCR for text-heavy images
                text = ocr.extract_text(img_bytes, lang='auto')
                desc = f"[Text Content from Image {i+1}]\n{text}"
            elif img_type == 'table':
                # Use table extraction
                markdown, _ = ocr.extract_table(img_bytes)
                desc = f"[Table from Image {i+1}]\n{markdown}" if markdown else ocr.extract_text(img_bytes)
            else:
                # Use VLM for complex images
                prompt = input.query if input.query else "Describe this image in detail."
                desc = await vlm.describe(img_bytes, prompt)
                desc = f"[Image {i+1} Analysis]\n{desc}"

            descriptions.append(desc)

        return SkillOutput(
            text="\n\n---\n\n".join(descriptions),
            images=[],
            sources=[],
            metadata={"images_analyzed": len(input.images)}
        )
```

```python
# app/skills/docs_skill.py

from .base import BaseSkill, SkillInput, SkillOutput, SourceReference

class DocsSkill(BaseSkill):
    """Search local documents only (exclude Confluence)."""

    name = "docs"
    description = "Search local documents (PDF, Word, Excel, PPT)"
    triggers = ["/docs", "/d", "/local"]
    keywords = ["document", "file", "local", "pdf", "文件"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        embedding_service = self.services['embedding_service']
        vector_store = self.services['vector_store']

        if not input.query:
            return SkillOutput(
                text="Please provide a search query.",
                sources=[]
            )

        # Search local vector store only
        results = vector_store.search(
            query=input.query,
            limit=10,
            filters=None  # No filter = all local documents
        )

        if not results:
            return SkillOutput(
                text="No matching documents found in local storage.",
                sources=[]
            )

        context_parts = []
        sources = []
        images = []

        for result in results:
            meta = result['metadata']
            context_parts.append(
                f"[{meta.get('title', 'Unknown')} - {meta.get('type', 'document').upper()}]\n"
                f"Source: {meta.get('source', 'Unknown')}\n"
                f"{result['text']}"
            )

            sources.append(SourceReference(
                title=meta.get('title', 'Unknown'),
                path=meta.get('source', ''),
                type='document'
            ))

            if meta.get('image_path'):
                images.append(meta['image_path'])

        return SkillOutput(
            text="\n\n---\n\n".join(context_parts),
            images=images[:5],  # Limit to 5 images
            sources=sources
        )
```

```python
# app/skills/summary_skill.py

from .base import BaseSkill, SkillInput, SkillOutput, SourceReference

class SummarySkill(BaseSkill):
    """Summarize a specific Confluence page."""

    name = "summary"
    description = "Summarize a Confluence page by URL or title"
    triggers = ["/summary", "/sum"]
    keywords = ["summarize", "summary", "tldr", "概要", "总结"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        confluence_index = self.services['confluence_index']
        confluence_client = self.services['confluence_client']

        if not input.query:
            return SkillOutput(
                text="Please provide a Confluence page URL or title to summarize.",
                sources=[]
            )

        query = input.query.strip()

        # Check if it's a URL
        page_id = None
        if 'confluence' in query.lower() or '/wiki/' in query:
            # Extract page ID from URL
            import re
            match = re.search(r'/pages/(\d+)', query)
            if match:
                page_id = match.group(1)

        # If not URL, search by title
        if not page_id:
            matched = confluence_index.search(query, fields=['title'], limit=1)
            if matched:
                page_id = matched[0]['id']

        if not page_id:
            return SkillOutput(
                text=f"Could not find Confluence page matching: {query}",
                sources=[]
            )

        # Fetch full page content
        try:
            content = await confluence_client.get_page_content(page_id)
        except Exception as e:
            return SkillOutput(
                text=f"Error fetching page: {str(e)}",
                sources=[]
            )

        # Create summary prompt for Copilot
        summary_context = f"""[Confluence Page Summary Request]

Page Title: {content['title']}
URL: {content['url']}

Full Content:
{content['body'][:8000]}  # Limit content length

---
Please provide a concise summary of this page, highlighting:
1. Main topic and purpose
2. Key points and decisions
3. Action items or next steps (if any)
"""

        return SkillOutput(
            text=summary_context,
            images=[],
            sources=[SourceReference(
                title=content['title'],
                path=content['url'],
                type='confluence',
                url=content['url']
            )]
        )
```

```python
# app/skills/index_skill.py

from .base import BaseSkill, SkillInput, SkillOutput

class IndexSkill(BaseSkill):
    """Manage Confluence index."""

    name = "index"
    description = "Manage Confluence index (refresh, status)"
    triggers = ["/index", "/idx"]
    keywords = ["index", "refresh", "sync", "索引", "刷新"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        confluence_index = self.services['confluence_index']

        query = (input.query or '').lower().strip()

        if query in ['refresh', 'sync', 'update']:
            # Trigger sync
            stats = await confluence_index.sync()
            return SkillOutput(
                text=f"""[Confluence Index Refresh Complete]

Statistics:
- Pages added: {stats['added']}
- Pages updated: {stats['updated']}
- Pages removed: {stats['removed']}
- Last sync: {confluence_index.last_sync_time}
""",
                sources=[]
            )

        elif query in ['status', 'info', '']:
            # Show status
            total_pages = sum(
                len(space['pages'])
                for space in confluence_index.index['spaces'].values()
            )
            spaces = list(confluence_index.index['spaces'].keys())

            return SkillOutput(
                text=f"""[Confluence Index Status]

- Total pages indexed: {total_pages}
- Spaces: {', '.join(spaces) if spaces else 'None'}
- Last sync: {confluence_index.last_sync_time or 'Never'}

Use `/index refresh` to update the index.
""",
                sources=[]
            )

        else:
            return SkillOutput(
                text="Unknown index command. Use `/index status` or `/index refresh`.",
                sources=[]
            )
```

#### 4.2.8 Skill Router

```python
# app/skills/router.py

from typing import Dict, List, Optional, Tuple
from .base import BaseSkill, SkillInput

class SkillRouter:
    """Route user input to appropriate skill."""

    def __init__(self):
        self.skills: Dict[str, BaseSkill] = {}
        self.command_map: Dict[str, str] = {}

    def register(self, skill: BaseSkill) -> None:
        """Register a skill."""
        self.skills[skill.name] = skill
        for trigger in skill.triggers:
            self.command_map[trigger.lower()] = skill.name

    def get_skill(self, name: str) -> Optional[BaseSkill]:
        """Get skill by name."""
        return self.skills.get(name)

    def route(
        self,
        user_input: str,
        has_images: bool = False,
        image_count: int = 0
    ) -> Tuple[BaseSkill, SkillInput]:
        """
        Route user input to appropriate skill.

        Returns:
            Tuple of (skill, skill_input)
        """
        user_input = user_input.strip()

        # 1. Check for explicit command
        if user_input.startswith('/'):
            parts = user_input.split(maxsplit=1)
            command = parts[0].lower()
            query = parts[1] if len(parts) > 1 else None

            if command in self.command_map:
                skill_name = self.command_map[command]
                skill = self.skills[skill_name]
                return skill, SkillInput(query=query)

        # 2. Intent recognition based on keywords and context
        input_lower = user_input.lower()

        # If user has images, prioritize image-related skills
        if has_images:
            # UML-related keywords
            uml_keywords = ['uml', 'class diagram', 'sequence', 'flowchart',
                          'activity', 'state diagram', '类图', '时序图', '流程图']
            if any(kw in input_lower for kw in uml_keywords):
                return self.skills['explain-uml'], SkillInput(query=user_input)

            # Comparison (requires 2 images)
            if image_count >= 2:
                compare_keywords = ['compare', 'difference', 'diff', '对比', '区别']
                if any(kw in input_lower for kw in compare_keywords):
                    return self.skills['compare'], SkillInput(query=user_input)

            # Default for images: analyze
            return self.skills['analyze'], SkillInput(query=user_input)

        # 3. Text-only queries
        confluence_keywords = ['confluence', 'wiki', 'page']
        if any(kw in input_lower for kw in confluence_keywords):
            return self.skills['confluence'], SkillInput(query=user_input)

        # 4. Default: global search
        return self.skills['search'], SkillInput(query=user_input)

    def get_available_commands(self) -> List[dict]:
        """Get list of available commands for UI."""
        commands = []
        for skill in self.skills.values():
            commands.append({
                'name': skill.name,
                'triggers': skill.triggers,
                'description': skill.description,
                'requires_image': skill.requires_image
            })
        return commands
```

### 4.3 AI Models Service

#### 4.3.1 Embedding Service

```python
# app/services/embedding_service.py

from sentence_transformers import SentenceTransformer
from typing import List, Union
import numpy as np
from config.settings import settings

class EmbeddingService:
    """Text embedding using BGE model."""

    def __init__(self):
        model_path = settings.model_dir / settings.embedding_model
        self.model = SentenceTransformer(str(model_path))
        self.dimension = self.model.get_sentence_embedding_dimension()

    def embed(self, text: Union[str, List[str]]) -> np.ndarray:
        """
        Embed text(s) into vector(s).

        Args:
            text: Single text or list of texts

        Returns:
            Numpy array of shape (dim,) for single text
            or (n, dim) for list of texts
        """
        return self.model.encode(text, normalize_embeddings=True)

    def embed_batch(self, texts: List[str], batch_size: int = 32) -> np.ndarray:
        """Embed texts in batches for memory efficiency."""
        return self.model.encode(
            texts,
            normalize_embeddings=True,
            batch_size=batch_size,
            show_progress_bar=True
        )
```

#### 4.3.2 CLIP Classifier

```python
# app/models/clip.py

import torch
from PIL import Image
from transformers import CLIPProcessor, CLIPModel
from typing import List, Tuple
import io
from config.settings import settings

class CLIPClassifier:
    """Zero-shot image classification using CLIP."""

    # Predefined categories for image classification
    CATEGORIES = [
        "a screenshot with text",
        "a table or spreadsheet",
        "a UML class diagram",
        "a UML sequence diagram",
        "a flowchart or activity diagram",
        "an architecture diagram",
        "a state diagram",
        "a chart or graph",
        "a photo or screenshot",
    ]

    CATEGORY_MAPPING = {
        "a screenshot with text": "text",
        "a table or spreadsheet": "table",
        "a UML class diagram": "uml",
        "a UML sequence diagram": "uml",
        "a flowchart or activity diagram": "uml",
        "an architecture diagram": "uml",
        "a state diagram": "uml",
        "a chart or graph": "table",
        "a photo or screenshot": "general",
    }

    def __init__(self):
        model_path = settings.model_dir / "clip-vit-base-patch32"
        self.model = CLIPModel.from_pretrained(str(model_path))
        self.processor = CLIPProcessor.from_pretrained(str(model_path))
        self.model.eval()

        # Precompute text embeddings for categories
        text_inputs = self.processor(
            text=self.CATEGORIES,
            return_tensors="pt",
            padding=True
        )
        with torch.no_grad():
            self.text_features = self.model.get_text_features(**text_inputs)
            self.text_features = self.text_features / self.text_features.norm(dim=-1, keepdim=True)

    def classify(self, image_bytes: bytes) -> Tuple[str, float]:
        """
        Classify image into predefined categories.

        Returns:
            Tuple of (category, confidence)
        """
        image = Image.open(io.BytesIO(image_bytes)).convert("RGB")

        image_inputs = self.processor(images=image, return_tensors="pt")

        with torch.no_grad():
            image_features = self.model.get_image_features(**image_inputs)
            image_features = image_features / image_features.norm(dim=-1, keepdim=True)

            similarity = (image_features @ self.text_features.T).squeeze()
            best_idx = similarity.argmax().item()
            confidence = similarity[best_idx].item()

        category = self.CATEGORIES[best_idx]
        mapped_type = self.CATEGORY_MAPPING[category]

        return mapped_type, confidence

    def get_embedding(self, image_bytes: bytes) -> torch.Tensor:
        """Get CLIP embedding for an image."""
        image = Image.open(io.BytesIO(image_bytes)).convert("RGB")
        image_inputs = self.processor(images=image, return_tensors="pt")

        with torch.no_grad():
            image_features = self.model.get_image_features(**image_inputs)
            image_features = image_features / image_features.norm(dim=-1, keepdim=True)

        return image_features.squeeze()
```

#### 4.3.3 Vision Language Model

**Important Note**: For CPU-only deployment, we recommend using **Qwen2-VL-2B-Instruct** or **moondream2** as they have better CPU inference support. The architecture uses transformers library instead of llama.cpp for broader model compatibility.

```python
# app/models/vlm.py

from transformers import AutoModelForCausalLM, AutoTokenizer, AutoProcessor
from PIL import Image
import torch
import io
import asyncio
from functools import partial
from config.settings import settings
import logging

logger = logging.getLogger(__name__)

class VisionLanguageModel:
    """
    Vision Language Model for image understanding.

    Recommended models for CPU deployment:
    - Qwen/Qwen2-VL-2B-Instruct (best quality/speed balance)
    - openbmb/MiniCPM-V-2 (smaller, faster)
    - vikhyatk/moondream2 (fastest, smallest)
    """

    def __init__(self):
        model_name = settings.vlm_model_name  # e.g., "Qwen/Qwen2-VL-2B-Instruct"
        model_path = settings.model_dir / "vlm"

        logger.info(f"Loading VLM model: {model_name}")

        # Load with CPU optimizations
        self.processor = AutoProcessor.from_pretrained(
            str(model_path),
            trust_remote_code=True
        )

        self.model = AutoModelForCausalLM.from_pretrained(
            str(model_path),
            torch_dtype=torch.float32,  # CPU doesn't support float16 well
            device_map="cpu",
            trust_remote_code=True,
            low_cpu_mem_usage=True
        )
        self.model.eval()

        logger.info("VLM model loaded successfully")

    def _preprocess_image(self, image_bytes: bytes) -> Image.Image:
        """Preprocess image for model input."""
        img = Image.open(io.BytesIO(image_bytes)).convert("RGB")

        # Resize if too large (save memory and speed up inference)
        max_size = 512  # Smaller for CPU efficiency
        if max(img.size) > max_size:
            ratio = max_size / max(img.size)
            new_size = (int(img.size[0] * ratio), int(img.size[1] * ratio))
            img = img.resize(new_size, Image.Resampling.LANCZOS)

        return img

    def _generate_sync(self, image: Image.Image, prompt: str) -> str:
        """Synchronous generation (runs in thread pool)."""
        # Prepare inputs
        messages = [
            {
                "role": "user",
                "content": [
                    {"type": "image", "image": image},
                    {"type": "text", "text": prompt}
                ]
            }
        ]

        # Process inputs
        text = self.processor.apply_chat_template(
            messages, tokenize=False, add_generation_prompt=True
        )
        inputs = self.processor(
            text=[text],
            images=[image],
            return_tensors="pt",
            padding=True
        )

        # Generate
        with torch.no_grad():
            generated_ids = self.model.generate(
                **inputs,
                max_new_tokens=512,
                do_sample=False,
                num_beams=1,  # Greedy for speed
                pad_token_id=self.processor.tokenizer.pad_token_id
            )

        # Decode
        generated_ids_trimmed = [
            out_ids[len(in_ids):]
            for in_ids, out_ids in zip(inputs.input_ids, generated_ids)
        ]
        output_text = self.processor.batch_decode(
            generated_ids_trimmed,
            skip_special_tokens=True,
            clean_up_tokenization_spaces=False
        )[0]

        return output_text

    async def describe(self, image: bytes, prompt: str) -> str:
        """
        Generate description for an image (async wrapper).

        Args:
            image: Image bytes
            prompt: Instruction prompt

        Returns:
            Generated description
        """
        img = self._preprocess_image(image)

        # Run CPU-intensive work in thread pool
        loop = asyncio.get_event_loop()
        result = await loop.run_in_executor(
            None,
            partial(self._generate_sync, img, prompt)
        )

        return result


# Alternative: Moondream2 (faster, smaller)
class MoondreamVLM:
    """Lightweight VLM using Moondream2 - best for CPU deployment."""

    def __init__(self):
        from transformers import AutoModelForCausalLM, AutoTokenizer

        model_path = settings.model_dir / "moondream2"

        self.model = AutoModelForCausalLM.from_pretrained(
            str(model_path),
            trust_remote_code=True,
            torch_dtype=torch.float32,
            device_map="cpu"
        )
        self.tokenizer = AutoTokenizer.from_pretrained(
            str(model_path),
            trust_remote_code=True
        )
        self.model.eval()

    async def describe(self, image: bytes, prompt: str) -> str:
        img = Image.open(io.BytesIO(image)).convert("RGB")

        # Resize for efficiency
        max_size = 384
        if max(img.size) > max_size:
            ratio = max_size / max(img.size)
            new_size = (int(img.size[0] * ratio), int(img.size[1] * ratio))
            img = img.resize(new_size, Image.Resampling.LANCZOS)

        loop = asyncio.get_event_loop()

        def _generate():
            enc_image = self.model.encode_image(img)
            return self.model.answer_question(enc_image, prompt, self.tokenizer)

        return await loop.run_in_executor(None, _generate)
```

#### 4.3.4 OCR Service

```python
# app/models/ocr.py

from paddleocr import PaddleOCR, PPStructure
from PIL import Image
import io
from typing import Tuple
from config.settings import settings

class OCRService:
    """OCR and table recognition using PaddleOCR."""

    def __init__(self):
        model_dir = settings.model_dir / "paddleocr"

        # Standard OCR - Initialize multiple language engines
        # Primary: English, Secondary: Japanese
        self.ocr_engines = {
            'en': PaddleOCR(
                use_angle_cls=True,
                lang='en',
                det_model_dir=str(model_dir / "det"),
                rec_model_dir=str(model_dir / "rec" / "en"),
                cls_model_dir=str(model_dir / "cls"),
                use_gpu=False,
                show_log=False
            ),
            'japan': PaddleOCR(
                use_angle_cls=True,
                lang='japan',
                det_model_dir=str(model_dir / "det"),
                rec_model_dir=str(model_dir / "rec" / "japan"),
                cls_model_dir=str(model_dir / "cls"),
                use_gpu=False,
                show_log=False
            ),
            'ch': PaddleOCR(
                use_angle_cls=True,
                lang='ch',
                det_model_dir=str(model_dir / "det"),
                rec_model_dir=str(model_dir / "rec" / "ch"),
                cls_model_dir=str(model_dir / "cls"),
                use_gpu=False,
                show_log=False
            )
        }
        self.default_ocr = self.ocr_engines['en']

        # Table structure recognition
        self.table_engine = PPStructure(
            table=True,
            ocr=True,
            show_log=False,
            use_gpu=False,
            table_model_dir=str(model_dir / "table")
        )

    def extract_text(self, image_bytes: bytes, lang: str = 'auto') -> str:
        """
        Extract text from image using OCR.

        Args:
            image_bytes: Raw image bytes
            lang: Language hint ('en', 'japan', 'ch', 'auto')
                  'auto' will try English first, then Japanese if low confidence

        Returns:
            Extracted text
        """
        img = Image.open(io.BytesIO(image_bytes))

        if lang == 'auto':
            # Try English first (most common for UML)
            result = self.ocr_engines['en'].ocr(img, cls=True)
            if result and result[0]:
                # Check average confidence
                confidences = [line[1][1] for line in result[0]]
                avg_confidence = sum(confidences) / len(confidences)

                # If low confidence, try Japanese
                if avg_confidence < 0.7:
                    result_jp = self.ocr_engines['japan'].ocr(img, cls=True)
                    if result_jp and result_jp[0]:
                        jp_confidences = [line[1][1] for line in result_jp[0]]
                        jp_avg = sum(jp_confidences) / len(jp_confidences)
                        if jp_avg > avg_confidence:
                            result = result_jp
        else:
            ocr = self.ocr_engines.get(lang, self.default_ocr)
            result = ocr.ocr(img, cls=True)

        if not result or not result[0]:
            return ""

        # Combine all detected text
        lines = []
        for line in result[0]:
            text = line[1][0]  # (bbox, (text, confidence))
            lines.append(text)

        return '\n'.join(lines)

    def extract_table(self, image_bytes: bytes) -> Tuple[str, str]:
        """
        Extract table from image.

        Returns:
            Tuple of (markdown_table, html_table)
        """
        img = Image.open(io.BytesIO(image_bytes))

        result = self.table_engine(img)

        if not result:
            return "", ""

        # Find table results
        for item in result:
            if item.get('type') == 'table':
                html = item.get('res', {}).get('html', '')
                markdown = self._html_table_to_markdown(html)
                return markdown, html

        return "", ""

    def _html_table_to_markdown(self, html: str) -> str:
        """Convert HTML table to Markdown format."""
        # Simple conversion - can be enhanced
        import re
        from html import unescape

        # Extract rows
        rows = re.findall(r'<tr[^>]*>(.*?)</tr>', html, re.DOTALL)
        if not rows:
            return ""

        md_rows = []
        for i, row in enumerate(rows):
            cells = re.findall(r'<t[hd][^>]*>(.*?)</t[hd]>', row, re.DOTALL)
            cells = [unescape(re.sub(r'<[^>]+>', '', c)).strip() for c in cells]
            md_rows.append('| ' + ' | '.join(cells) + ' |')

            # Add header separator after first row
            if i == 0:
                md_rows.append('| ' + ' | '.join(['---'] * len(cells)) + ' |')

        return '\n'.join(md_rows)
```

### 4.4 Document Processing

#### 4.4.1 Document Parser

```python
# app/services/document_parser.py

from pathlib import Path
from typing import List, Dict, Any, Generator
from dataclasses import dataclass
import fitz  # PyMuPDF
from docx import Document
from openpyxl import load_workbook
from pptx import Presentation
from config.settings import settings

@dataclass
class Chunk:
    """A chunk of text with metadata."""
    text: str
    metadata: Dict[str, Any]
    images: List[bytes] = None

class DocumentParser:
    """Parse various document formats."""

    def __init__(self, chunk_size: int = None, chunk_overlap: int = None):
        self.chunk_size = chunk_size or settings.chunk_size
        self.chunk_overlap = chunk_overlap or settings.chunk_overlap

    def parse(self, file_path: str) -> Generator[Chunk, None, None]:
        """
        Parse document and yield chunks.

        Args:
            file_path: Path to document

        Yields:
            Chunk objects
        """
        path = Path(file_path)
        suffix = path.suffix.lower()

        if suffix == '.pdf':
            yield from self._parse_pdf(path)
        elif suffix in ['.docx', '.doc']:
            yield from self._parse_word(path)
        elif suffix in ['.xlsx', '.xls']:
            yield from self._parse_excel(path)
        elif suffix in ['.pptx', '.ppt']:
            yield from self._parse_ppt(path)
        else:
            raise ValueError(f"Unsupported file format: {suffix}")

    def _parse_pdf(self, path: Path) -> Generator[Chunk, None, None]:
        """Parse PDF document."""
        doc = fitz.open(path)

        for page_num, page in enumerate(doc):
            # Extract text
            text = page.get_text()

            # Extract images
            images = []
            for img_index, img in enumerate(page.get_images(full=True)):
                xref = img[0]
                base_image = doc.extract_image(xref)
                images.append(base_image["image"])

            # Chunk the text
            for chunk_text in self._chunk_text(text):
                yield Chunk(
                    text=chunk_text,
                    metadata={
                        'source': str(path),
                        'title': path.stem,
                        'page': page_num + 1,
                        'type': 'pdf'
                    },
                    images=images if images else None
                )

            # Clear images for subsequent chunks from same page
            images = []

        doc.close()

    def _parse_word(self, path: Path) -> Generator[Chunk, None, None]:
        """Parse Word document."""
        doc = Document(path)

        full_text = []
        images = []

        for para in doc.paragraphs:
            if para.text.strip():
                full_text.append(para.text)

        # Extract images from document
        for rel in doc.part.rels.values():
            if "image" in rel.reltype:
                images.append(rel.target_part.blob)

        # Chunk the combined text
        combined_text = '\n'.join(full_text)
        for chunk_text in self._chunk_text(combined_text):
            yield Chunk(
                text=chunk_text,
                metadata={
                    'source': str(path),
                    'title': path.stem,
                    'type': 'word'
                },
                images=images if images else None
            )
            images = []  # Only attach images to first chunk

    def _parse_excel(self, path: Path) -> Generator[Chunk, None, None]:
        """Parse Excel document."""
        wb = load_workbook(path, data_only=True)

        for sheet_name in wb.sheetnames:
            sheet = wb[sheet_name]

            # Convert sheet to text representation
            rows = []
            for row in sheet.iter_rows(values_only=True):
                row_text = ' | '.join(str(cell) if cell else '' for cell in row)
                if row_text.strip(' |'):
                    rows.append(row_text)

            if rows:
                text = f"Sheet: {sheet_name}\n" + '\n'.join(rows)

                for chunk_text in self._chunk_text(text):
                    yield Chunk(
                        text=chunk_text,
                        metadata={
                            'source': str(path),
                            'title': path.stem,
                            'sheet': sheet_name,
                            'type': 'excel'
                        }
                    )

    def _parse_ppt(self, path: Path) -> Generator[Chunk, None, None]:
        """Parse PowerPoint document."""
        prs = Presentation(path)

        for slide_num, slide in enumerate(prs.slides):
            texts = []
            images = []

            for shape in slide.shapes:
                # Extract text
                if hasattr(shape, "text") and shape.text.strip():
                    texts.append(shape.text)

                # Extract images
                if shape.shape_type == 13:  # Picture
                    image = shape.image
                    images.append(image.blob)

            if texts:
                text = f"Slide {slide_num + 1}:\n" + '\n'.join(texts)

                for chunk_text in self._chunk_text(text):
                    yield Chunk(
                        text=chunk_text,
                        metadata={
                            'source': str(path),
                            'title': path.stem,
                            'slide': slide_num + 1,
                            'type': 'ppt'
                        },
                        images=images if images else None
                    )
                    images = []

    def _chunk_text(self, text: str) -> Generator[str, None, None]:
        """Split text into overlapping chunks."""
        if len(text) <= self.chunk_size:
            yield text
            return

        start = 0
        while start < len(text):
            end = start + self.chunk_size

            # Try to break at sentence boundary
            if end < len(text):
                # Look for sentence end within last 20% of chunk
                search_start = end - int(self.chunk_size * 0.2)
                for sep in ['. ', '。', '\n\n', '\n']:
                    pos = text.rfind(sep, search_start, end)
                    if pos != -1:
                        end = pos + len(sep)
                        break

            yield text[start:end].strip()
            start = end - self.chunk_overlap
```

#### 4.4.2 Image Processor

```python
# app/services/image_processor.py

from typing import Tuple
from dataclasses import dataclass
from config.settings import settings

@dataclass
class ProcessedImage:
    """Result of image processing."""
    image_type: str           # 'text', 'table', 'uml', 'general'
    description: str          # Text description for embedding
    original_bytes: bytes     # Original image
    confidence: float         # Classification confidence

class ImageProcessor:
    """Process images through classification and description pipeline."""

    def __init__(self, clip_classifier, ocr_service, vlm):
        self.clip = clip_classifier
        self.ocr = ocr_service
        self.vlm = vlm

    async def process(self, image_bytes: bytes) -> ProcessedImage:
        """
        Process image through appropriate pipeline based on classification.

        Args:
            image_bytes: Raw image bytes

        Returns:
            ProcessedImage with description
        """
        # 1. Classify image
        image_type, confidence = self.clip.classify(image_bytes)

        # 2. Process based on type
        if image_type == 'text':
            description = self.ocr.extract_text(image_bytes)

        elif image_type == 'table':
            markdown_table, _ = self.ocr.extract_table(image_bytes)
            if markdown_table:
                description = f"[Table Content]\n{markdown_table}"
            else:
                # Fallback to OCR if table extraction fails
                description = self.ocr.extract_text(image_bytes)

        elif image_type == 'uml':
            # Use VLM for UML diagrams
            description = await self.vlm.describe(
                image_bytes,
                prompt=self._get_uml_prompt()
            )

        else:  # 'general'
            # Use VLM for general images
            description = await self.vlm.describe(
                image_bytes,
                prompt="Describe this image in detail."
            )

        return ProcessedImage(
            image_type=image_type,
            description=description,
            original_bytes=image_bytes,
            confidence=confidence
        )

    def _get_uml_prompt(self) -> str:
        return """Analyze this diagram and provide:
1. Diagram type (class/sequence/activity/state/component/flowchart)
2. All elements (classes, actors, components, etc.)
3. Relationships between elements
4. Overall purpose and flow

Be detailed and structured."""
```

#### 4.4.3 Document Service

```python
# app/services/document_service.py

from typing import Dict, Any, Optional
from pathlib import Path
import hashlib
import logging
from dataclasses import dataclass

from config.settings import settings
from app.services.document_parser import DocumentParser
from app.services.image_processor import ImageProcessor
from app.services.embedding_service import EmbeddingService
from app.services.vector_store import VectorStore

logger = logging.getLogger(__name__)

@dataclass
class IngestResult:
    """Result of document ingestion."""
    success: bool
    document_id: str
    chunks_count: int
    images_count: int
    error: Optional[str] = None

class DocumentService:
    """Service for document ingestion and management."""

    def __init__(
        self,
        parser: DocumentParser,
        image_processor: ImageProcessor,
        embedding_service: EmbeddingService,
        vector_store: VectorStore,
        image_store: 'ImageStore'
    ):
        self.parser = parser
        self.image_processor = image_processor
        self.embedding_service = embedding_service
        self.vector_store = vector_store
        self.image_store = image_store

    async def ingest(self, file_path: str, source_type: str) -> IngestResult:
        """
        Ingest a document into the vector store.

        Args:
            file_path: Path to the document
            source_type: Type of document (pdf, docx, xlsx, pptx)

        Returns:
            IngestResult with ingestion statistics
        """
        path = Path(file_path)
        document_id = self._generate_document_id(path)

        try:
            logger.info(f"Starting ingestion: {path.name}")

            # 1. Parse document
            content, images = self.parser.parse(str(path))

            # 2. Process images
            processed_images = []
            for i, img_bytes in enumerate(images):
                processed = await self.image_processor.process(img_bytes)
                image_id = f"{document_id}_img_{i}"
                self.image_store.save(image_id, img_bytes)
                processed_images.append({
                    'id': image_id,
                    'description': processed.description,
                    'type': processed.image_type
                })

            # 3. Chunk text content
            chunks = list(self.parser.chunk_text(content))

            # 4. Create embeddings and store
            for i, chunk in enumerate(chunks):
                embedding = self.embedding_service.embed(chunk)
                self.vector_store.add(
                    id=f"{document_id}_chunk_{i}",
                    embedding=embedding,
                    metadata={
                        'document_id': document_id,
                        'source': str(path),
                        'source_type': source_type,
                        'chunk_index': i,
                        'content': chunk
                    }
                )

            # 5. Store image descriptions as separate embeddings
            for img in processed_images:
                embedding = self.embedding_service.embed(img['description'])
                self.vector_store.add(
                    id=img['id'],
                    embedding=embedding,
                    metadata={
                        'document_id': document_id,
                        'source': str(path),
                        'source_type': 'image',
                        'image_type': img['type'],
                        'content': img['description'],
                        'image_id': img['id']
                    }
                )

            logger.info(f"Ingestion complete: {len(chunks)} chunks, {len(processed_images)} images")

            return IngestResult(
                success=True,
                document_id=document_id,
                chunks_count=len(chunks),
                images_count=len(processed_images)
            )

        except Exception as e:
            logger.error(f"Ingestion failed: {e}")
            return IngestResult(
                success=False,
                document_id=document_id,
                chunks_count=0,
                images_count=0,
                error=str(e)
            )

    def _generate_document_id(self, path: Path) -> str:
        """Generate unique document ID from file path and content hash."""
        content_hash = hashlib.md5(path.read_bytes()).hexdigest()[:8]
        return f"{path.stem}_{content_hash}"

    async def delete(self, document_id: str) -> bool:
        """Delete a document and all its chunks/images from the store."""
        try:
            self.vector_store.delete_by_document(document_id)
            self.image_store.delete_by_prefix(document_id)
            return True
        except Exception as e:
            logger.error(f"Delete failed: {e}")
            return False
```

#### 4.4.4 Image Store

```python
# app/services/image_store.py

from pathlib import Path
from typing import Optional
import shutil
import logging

from config.settings import settings

logger = logging.getLogger(__name__)

class ImageStore:
    """Simple file-based image storage."""

    def __init__(self, base_path: Optional[Path] = None):
        self.base_path = base_path or settings.data_dir / "images"
        self.base_path.mkdir(parents=True, exist_ok=True)

    def save(self, image_id: str, data: bytes) -> str:
        """Save image and return path."""
        # Determine extension from magic bytes
        ext = self._detect_extension(data)
        file_path = self.base_path / f"{image_id}{ext}"
        file_path.write_bytes(data)
        return str(file_path)

    def get(self, image_id: str) -> Optional[bytes]:
        """Get image by ID."""
        for ext in ['.png', '.jpg', '.jpeg', '.gif', '.webp']:
            path = self.base_path / f"{image_id}{ext}"
            if path.exists():
                return path.read_bytes()
        return None

    def get_path(self, image_id: str) -> Optional[Path]:
        """Get image file path by ID."""
        for ext in ['.png', '.jpg', '.jpeg', '.gif', '.webp']:
            path = self.base_path / f"{image_id}{ext}"
            if path.exists():
                return path
        return None

    def delete(self, image_id: str) -> bool:
        """Delete image by ID."""
        for ext in ['.png', '.jpg', '.jpeg', '.gif', '.webp']:
            path = self.base_path / f"{image_id}{ext}"
            if path.exists():
                path.unlink()
                return True
        return False

    def delete_by_prefix(self, prefix: str) -> int:
        """Delete all images with given prefix."""
        count = 0
        for path in self.base_path.glob(f"{prefix}*"):
            path.unlink()
            count += 1
        return count

    def count(self) -> int:
        """Count total stored images."""
        return len(list(self.base_path.glob("*")))

    def _detect_extension(self, data: bytes) -> str:
        """Detect image format from magic bytes."""
        if data[:8] == b'\x89PNG\r\n\x1a\n':
            return '.png'
        elif data[:2] == b'\xff\xd8':
            return '.jpg'
        elif data[:6] in (b'GIF87a', b'GIF89a'):
            return '.gif'
        elif data[:4] == b'RIFF' and data[8:12] == b'WEBP':
            return '.webp'
        return '.bin'
```

### 4.5 Confluence Integration

#### 4.5.1 Confluence Client

```python
# app/services/confluence_client.py

import aiohttp
from typing import List, Dict, Optional
from config.settings import settings

class ConfluenceClient:
    """Async client for Confluence REST API with caching and retry."""

    def __init__(self):
        self.base_url = settings.confluence_base_url.rstrip('/')
        self.auth = aiohttp.BasicAuth(
            settings.confluence_username,
            settings.confluence_api_token
        )
        # Short-term cache for page content (5 minutes)
        self._cache: Dict[str, tuple] = {}  # {page_id: (content, timestamp)}
        self._cache_ttl = 300  # 5 minutes

    def _get_cached(self, key: str) -> Optional[Dict]:
        """Get cached content if not expired."""
        import time
        if key in self._cache:
            content, timestamp = self._cache[key]
            if time.time() - timestamp < self._cache_ttl:
                return content
            del self._cache[key]
        return None

    def _set_cache(self, key: str, content: Dict) -> None:
        """Cache content with timestamp."""
        import time
        self._cache[key] = (content, time.time())
        # Clean old entries if cache too large
        if len(self._cache) > 100:
            cutoff = time.time() - self._cache_ttl
            self._cache = {
                k: v for k, v in self._cache.items()
                if v[1] > cutoff
            }

    async def _request_with_retry(
        self,
        session: aiohttp.ClientSession,
        url: str,
        max_retries: int = 3
    ) -> Dict:
        """Make request with exponential backoff retry."""
        import asyncio
        last_error = None

        for attempt in range(max_retries):
            try:
                async with session.get(url) as response:
                    if response.status == 200:
                        return await response.json()
                    elif response.status == 429:  # Rate limited
                        wait_time = int(response.headers.get('Retry-After', 5))
                        await asyncio.sleep(wait_time)
                        continue
                    else:
                        response.raise_for_status()
            except aiohttp.ClientError as e:
                last_error = e
                wait_time = (2 ** attempt) * 0.5  # Exponential backoff
                await asyncio.sleep(wait_time)

        raise last_error or Exception("Request failed after retries")

    async def get_spaces(self) -> List[Dict]:
        """Get all accessible spaces."""
        async with aiohttp.ClientSession(auth=self.auth) as session:
            url = f"{self.base_url}/wiki/rest/api/space"
            async with session.get(url) as response:
                data = await response.json()
                return data.get('results', [])

    async def get_pages(
        self,
        space_key: str,
        expand: str = "version,ancestors,metadata.labels",
        limit: int = 500
    ) -> List[Dict]:
        """Get all pages in a space (metadata only)."""
        pages = []
        start = 0

        async with aiohttp.ClientSession(auth=self.auth) as session:
            while True:
                url = (
                    f"{self.base_url}/wiki/rest/api/content"
                    f"?spaceKey={space_key}"
                    f"&expand={expand}"
                    f"&limit={limit}"
                    f"&start={start}"
                )

                async with session.get(url) as response:
                    data = await response.json()
                    results = data.get('results', [])
                    pages.extend(results)

                    if len(results) < limit:
                        break
                    start += limit

        return pages

    async def get_page_content(self, page_id: str, use_cache: bool = True) -> Dict:
        """Get full page content with optional caching."""
        # Check cache first
        if use_cache:
            cached = self._get_cached(f"page_{page_id}")
            if cached:
                return cached

        async with aiohttp.ClientSession(auth=self.auth) as session:
            url = (
                f"{self.base_url}/wiki/rest/api/content/{page_id}"
                f"?expand=body.storage,version"
            )

            data = await self._request_with_retry(session, url)

            result = {
                'id': data['id'],
                'title': data['title'],
                'body': self._html_to_text(data['body']['storage']['value']),
                'version': data['version']['number'],
                'url': f"{self.base_url}/wiki{data['_links']['webui']}"
            }

            # Cache the result
            self._set_cache(f"page_{page_id}", result)

            return result

    async def get_page_attachments(self, page_id: str) -> List[Dict]:
        """Get attachments (including images) for a page."""
        async with aiohttp.ClientSession(auth=self.auth) as session:
            url = (
                f"{self.base_url}/wiki/rest/api/content/{page_id}/child/attachment"
            )

            async with session.get(url) as response:
                data = await response.json()
                return data.get('results', [])

    async def download_attachment(self, download_url: str) -> bytes:
        """Download attachment content."""
        async with aiohttp.ClientSession(auth=self.auth) as session:
            full_url = f"{self.base_url}{download_url}"
            async with session.get(full_url) as response:
                return await response.read()

    def _html_to_text(self, html: str) -> str:
        """Convert Confluence storage format HTML to plain text."""
        import re
        from html import unescape

        # Remove script and style elements
        html = re.sub(r'<(script|style)[^>]*>.*?</\1>', '', html, flags=re.DOTALL)

        # Convert common elements
        html = re.sub(r'<br\s*/?>', '\n', html)
        html = re.sub(r'<p[^>]*>', '\n', html)
        html = re.sub(r'</p>', '\n', html)
        html = re.sub(r'<li[^>]*>', '\n- ', html)
        html = re.sub(r'<h[1-6][^>]*>', '\n## ', html)

        # Remove remaining tags
        html = re.sub(r'<[^>]+>', '', html)

        # Decode HTML entities
        text = unescape(html)

        # Clean up whitespace
        text = re.sub(r'\n{3,}', '\n\n', text)

        return text.strip()
```

#### 4.5.2 Confluence Index Manager

```python
# app/services/confluence_index.py

import json
import asyncio
from pathlib import Path
from datetime import datetime
from typing import List, Dict, Optional
from config.settings import settings

class ConfluenceIndex:
    """Manage Confluence page index for fast searching."""

    def __init__(self, confluence_client):
        self.client = confluence_client
        self.index_path = settings.data_dir / "confluence_index.json"
        self.index = self._load_index()

    def _load_index(self) -> Dict:
        """Load index from file."""
        if self.index_path.exists():
            with open(self.index_path, 'r') as f:
                return json.load(f)
        return {
            'spaces': {},
            'last_sync': None
        }

    def _save_index(self) -> None:
        """Save index to file."""
        self.index_path.parent.mkdir(parents=True, exist_ok=True)
        with open(self.index_path, 'w') as f:
            json.dump(self.index, f, indent=2, default=str)

    @property
    def last_sync_time(self) -> Optional[str]:
        """Get last sync timestamp."""
        return self.index.get('last_sync')

    async def sync(self) -> Dict:
        """
        Sync index with Confluence.

        Returns:
            Sync statistics
        """
        stats = {'added': 0, 'updated': 0, 'removed': 0}
        current_page_ids = set()

        # Get all spaces
        spaces = await self.client.get_spaces()

        for space in spaces:
            space_key = space['key']

            if space_key not in self.index['spaces']:
                self.index['spaces'][space_key] = {
                    'name': space['name'],
                    'pages': {}
                }

            # Get pages (metadata only)
            pages = await self.client.get_pages(space_key)

            for page in pages:
                page_id = page['id']
                current_page_ids.add(page_id)

                page_data = {
                    'title': page['title'],
                    'path': self._build_path(page),
                    'labels': [l['name'] for l in page.get('metadata', {}).get('labels', {}).get('results', [])],
                    'last_modified': page['version']['when'],
                    'version': page['version']['number']
                }

                existing = self.index['spaces'][space_key]['pages'].get(page_id)

                if not existing:
                    stats['added'] += 1
                elif existing['version'] != page_data['version']:
                    stats['updated'] += 1

                self.index['spaces'][space_key]['pages'][page_id] = page_data

        # Remove deleted pages
        for space_key, space_data in self.index['spaces'].items():
            to_remove = [
                pid for pid in space_data['pages']
                if pid not in current_page_ids
            ]
            for pid in to_remove:
                del space_data['pages'][pid]
                stats['removed'] += 1

        self.index['last_sync'] = datetime.utcnow().isoformat()
        self._save_index()

        return stats

    def search(
        self,
        query: str,
        fields: List[str] = None,
        limit: int = 10
    ) -> List[Dict]:
        """
        Search index for matching pages.

        Args:
            query: Search query
            fields: Fields to search in ['title', 'path', 'labels']
            limit: Maximum results

        Returns:
            List of matching page metadata
        """
        fields = fields or ['title', 'path', 'labels']
        query_lower = query.lower()
        query_terms = query_lower.split()

        results = []

        for space_key, space_data in self.index['spaces'].items():
            for page_id, page in space_data['pages'].items():
                score = 0

                # Title matching
                if 'title' in fields:
                    title_lower = page['title'].lower()
                    if query_lower in title_lower:
                        score += 10
                    else:
                        score += sum(2 for term in query_terms if term in title_lower)

                # Path matching
                if 'path' in fields:
                    path_lower = page['path'].lower()
                    score += sum(1 for term in query_terms if term in path_lower)

                # Label matching
                if 'labels' in fields:
                    labels_lower = [l.lower() for l in page['labels']]
                    score += sum(3 for term in query_terms if any(term in l for l in labels_lower))

                if score > 0:
                    results.append({
                        'id': page_id,
                        'space': space_key,
                        'score': score,
                        **page
                    })

        # Sort by score and limit
        results.sort(key=lambda x: x['score'], reverse=True)
        return results[:limit]

    def _build_path(self, page: Dict) -> str:
        """Build page path from ancestors."""
        ancestors = page.get('ancestors', [])
        path_parts = [a['title'] for a in ancestors]
        path_parts.append(page['title'])
        return '/' + '/'.join(path_parts)
```

### 4.6 Scheduler Tasks

```python
# app/scheduler/tasks.py

from apscheduler.schedulers.asyncio import AsyncIOScheduler
from apscheduler.triggers.interval import IntervalTrigger
import logging
from config.settings import settings

logger = logging.getLogger(__name__)

def setup_scheduler(services: dict) -> AsyncIOScheduler:
    """Setup scheduled tasks."""

    scheduler = AsyncIOScheduler()

    # Confluence index sync task
    async def sync_confluence_index():
        logger.info("Starting scheduled Confluence index sync...")
        try:
            confluence_index = services['confluence_index']
            stats = await confluence_index.sync()
            logger.info(
                f"Confluence sync complete: "
                f"added={stats['added']}, "
                f"updated={stats['updated']}, "
                f"removed={stats['removed']}"
            )
        except Exception as e:
            logger.error(f"Confluence sync failed: {e}")

    scheduler.add_job(
        sync_confluence_index,
        trigger=IntervalTrigger(hours=settings.confluence_sync_interval_hours),
        id='confluence_sync',
        name='Sync Confluence Index',
        replace_existing=True
    )

    # Cleanup old uploaded files task
    async def cleanup_uploads():
        import os
        from pathlib import Path
        from datetime import datetime, timedelta

        logger.info("Starting upload cleanup...")
        upload_dir = Path(settings.upload_dir)
        cutoff = datetime.now() - timedelta(days=7)
        removed = 0

        for f in upload_dir.iterdir():
            if f.is_file():
                mtime = datetime.fromtimestamp(f.stat().st_mtime)
                if mtime < cutoff:
                    f.unlink()
                    removed += 1

        logger.info(f"Cleanup complete: removed {removed} old files")

    scheduler.add_job(
        cleanup_uploads,
        trigger=IntervalTrigger(days=1),
        id='cleanup_uploads',
        name='Cleanup Old Uploads',
        replace_existing=True
    )

    # Health check logging
    async def log_health_status():
        vector_store = services['vector_store']
        confluence_index = services['confluence_index']

        doc_count = vector_store.count()
        page_count = sum(
            len(s['pages']) for s in confluence_index.index['spaces'].values()
        )

        logger.info(
            f"Health check: docs={doc_count}, "
            f"confluence_pages={page_count}, "
            f"status=healthy"
        )

    scheduler.add_job(
        log_health_status,
        trigger=IntervalTrigger(hours=1),
        id='health_check',
        name='Health Check Logging',
        replace_existing=True
    )

    return scheduler
```

### 4.7 Vector Store

```python
# app/services/vector_store.py

from qdrant_client import QdrantClient
from qdrant_client.models import (
    VectorParams, Distance, PointStruct,
    Filter, FieldCondition, MatchValue
)
from typing import List, Dict, Optional
import uuid
from config.settings import settings

class VectorStore:
    """Vector storage using Qdrant."""

    def __init__(self, embedding_service):
        self.embedding_service = embedding_service
        self.collection_name = settings.collection_name

        # Use embedded mode (no Docker needed)
        self.client = QdrantClient(path=str(settings.qdrant_path))

        self._ensure_collection()

    def _ensure_collection(self) -> None:
        """Create collection if not exists."""
        collections = self.client.get_collections().collections
        exists = any(c.name == self.collection_name for c in collections)

        if not exists:
            self.client.create_collection(
                collection_name=self.collection_name,
                vectors_config=VectorParams(
                    size=self.embedding_service.dimension,
                    distance=Distance.COSINE
                )
            )

    def add(
        self,
        texts: List[str],
        metadatas: List[Dict],
        ids: Optional[List[str]] = None
    ) -> List[str]:
        """
        Add texts to vector store.

        Args:
            texts: List of text strings
            metadatas: List of metadata dicts
            ids: Optional list of IDs

        Returns:
            List of assigned IDs
        """
        if ids is None:
            ids = [str(uuid.uuid4()) for _ in texts]

        embeddings = self.embedding_service.embed_batch(texts)

        points = [
            PointStruct(
                id=id,
                vector=embedding.tolist(),
                payload={**metadata, 'text': text}
            )
            for id, embedding, text, metadata in zip(ids, embeddings, texts, metadatas)
        ]

        self.client.upsert(
            collection_name=self.collection_name,
            points=points
        )

        return ids

    def search(
        self,
        query: str = None,
        embedding: List[float] = None,
        limit: int = 5,
        filters: Dict = None
    ) -> List[Dict]:
        """
        Search for similar documents.

        Args:
            query: Text query (will be embedded)
            embedding: Pre-computed embedding
            limit: Number of results
            filters: Optional metadata filters

        Returns:
            List of results with text, metadata, and score
        """
        if embedding is None:
            if query is None:
                raise ValueError("Either query or embedding must be provided")
            embedding = self.embedding_service.embed(query).tolist()

        # Build filter if provided
        qdrant_filter = None
        if filters:
            conditions = [
                FieldCondition(key=k, match=MatchValue(value=v))
                for k, v in filters.items()
            ]
            qdrant_filter = Filter(must=conditions)

        results = self.client.search(
            collection_name=self.collection_name,
            query_vector=embedding,
            limit=limit,
            query_filter=qdrant_filter
        )

        return [
            {
                'id': r.id,
                'text': r.payload.get('text', ''),
                'metadata': {k: v for k, v in r.payload.items() if k != 'text'},
                'score': r.score
            }
            for r in results
        ]

    def delete(self, ids: List[str]) -> None:
        """Delete documents by ID."""
        self.client.delete(
            collection_name=self.collection_name,
            points_selector=ids
        )

    def count(self) -> int:
        """Get total document count."""
        info = self.client.get_collection(self.collection_name)
        return info.points_count
```

---

## 5. Data Structures

### 5.1 Confluence Index Schema

```json
{
  "spaces": {
    "SPACE_KEY": {
      "name": "Space Name",
      "pages": {
        "page_id": {
          "title": "Page Title",
          "path": "/Space/Parent/Page Title",
          "labels": ["label1", "label2"],
          "last_modified": "2024-01-15T10:30:00Z",
          "version": 5
        }
      }
    }
  },
  "last_sync": "2024-01-20T08:00:00Z"
}
```

### 5.2 Vector Store Document Schema

```json
{
  "id": "uuid-string",
  "vector": [0.1, 0.2, ...],
  "payload": {
    "text": "Chunk content...",
    "source": "/path/to/document.pdf",
    "title": "Document Title",
    "type": "pdf",
    "page": 5,
    "image_path": "/opt/rag-data/images/uuid.png",
    "image_type": "uml",
    "image_description": "UML class diagram showing..."
  }
}
```

### 5.3 API Request/Response Schemas

```typescript
// Skill Request
interface SkillRequest {
  skill: string;
  query?: string;
  options?: Record<string, any>;
  // Images sent as multipart form data
}

// Skill Response
interface SkillResponse {
  text: string;
  images: string[];  // Image URLs
  sources: SourceReference[];
  metadata?: Record<string, any>;
}

interface SourceReference {
  title: string;
  path: string;
  type: 'confluence' | 'document' | 'image';
  url?: string;
}

// Ingest Request
interface IngestRequest {
  file_path: string;
  source_type: 'pdf' | 'word' | 'excel' | 'ppt';
}

// Status Response
interface StatusResponse {
  status: 'healthy' | 'degraded' | 'unhealthy';
  version: string;
  models_loaded: string[];
  confluence_last_sync?: string;
  document_count: number;
  image_count: number;
}
```

---

## 6. Deployment

### 6.1 Directory Structure

```
/opt/rag/
├── models/                    # AI models
│   ├── bge-base-en-v1.5/
│   ├── clip-vit-base-patch32/
│   ├── vlm/                   # Qwen2-VL or moondream2
│   └── paddleocr/
├── data/                      # Runtime data
│   ├── qdrant/               # Vector database
│   ├── images/               # Stored images
│   ├── uploads/              # Uploaded documents
│   └── confluence_index.json # Confluence index
├── backend/                   # Backend service
│   ├── app/
│   ├── config/
│   ├── requirements.txt
│   └── .env
└── logs/                      # Log files
```

### 6.2 Environment Configuration

```bash
# /opt/rag/backend/.env

# Server
RAG_HOST=0.0.0.0
RAG_PORT=8000
RAG_DEBUG=false

# Paths
RAG_MODEL_DIR=/opt/rag/models
RAG_DATA_DIR=/opt/rag/data
RAG_UPLOAD_DIR=/opt/rag/data/uploads

# Confluence
RAG_CONFLUENCE_BASE_URL=https://your-company.atlassian.net
RAG_CONFLUENCE_USERNAME=your-email@company.com
RAG_CONFLUENCE_API_TOKEN=your-api-token
RAG_CONFLUENCE_SYNC_INTERVAL_HOURS=4

# Models
RAG_EMBEDDING_MODEL=bge-base-en-v1.5
RAG_VLM_MODEL_NAME=Qwen/Qwen2-VL-2B-Instruct

# Processing
RAG_MAX_UPLOAD_IMAGES=2
RAG_MAX_IMAGE_SIZE_MB=5
RAG_CHUNK_SIZE=512
RAG_CHUNK_OVERLAP=50
```

### 6.3 Systemd Service

```ini
# /etc/systemd/system/rag-backend.service

[Unit]
Description=RAG Backend Service
After=network.target

[Service]
Type=simple
User=rag
Group=rag
WorkingDirectory=/opt/rag/backend
Environment="PATH=/opt/rag/venv/bin"
ExecStart=/opt/rag/venv/bin/uvicorn app.main:app --host 0.0.0.0 --port 8000
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 6.4 Installation Script

```bash
#!/bin/bash
# install.sh

set -e

RAG_DIR=/opt/rag
VENV_DIR=$RAG_DIR/venv

# Create directories
sudo mkdir -p $RAG_DIR/{models,data,backend,logs}
sudo mkdir -p $RAG_DIR/data/{qdrant,images,uploads}

# Create user
sudo useradd -r -s /bin/false rag || true

# Setup Python environment
python3 -m venv $VENV_DIR
source $VENV_DIR/bin/activate

# Install dependencies
pip install --upgrade pip
pip install -r requirements.txt

# Download models (run separately on machine with internet)
# python scripts/download_models.py

# Set permissions
sudo chown -R rag:rag $RAG_DIR

# Install systemd service
sudo cp rag-backend.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable rag-backend
sudo systemctl start rag-backend

echo "Installation complete!"
```

### 6.5 Requirements

```
# requirements.txt

# Web framework
fastapi>=0.100.0
uvicorn[standard]>=0.23.0
python-multipart>=0.0.6

# Scheduling
apscheduler>=3.10.0

# Document processing
PyMuPDF>=1.23.0
python-docx>=0.8.11
openpyxl>=3.1.0
python-pptx>=0.6.21

# AI/ML
sentence-transformers>=2.2.0
transformers>=4.35.0
torch>=2.0.0
Pillow>=10.0.0

# Vector store
qdrant-client>=1.7.0

# OCR
paddlepaddle>=2.5.0
paddleocr>=2.7.0

# HTTP client
aiohttp>=3.9.0

# Configuration
pydantic-settings>=2.0.0
python-dotenv>=1.0.0

# Utilities
numpy>=1.24.0
```

#### 6.5.2 Development Dependencies

```text
# requirements-dev.txt

# Testing
pytest>=7.4.0
pytest-asyncio>=0.21.0
pytest-cov>=4.1.0

# Code quality
black>=23.0.0
isort>=5.12.0
flake8>=6.1.0
mypy>=1.5.0

# Debugging
ipython>=8.0.0
ipdb>=0.13.0

# Documentation
mkdocs>=1.5.0
mkdocs-material>=9.0.0
```

---

## 7. Development Guide

### 7.1 Local Development Setup

```bash
# Clone repository
git clone <repo-url>
cd rag-tool

# Backend setup
cd backend
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install -r requirements-dev.txt

# Create .env file
cp .env.example .env
# Edit .env with your settings

# Run backend
uvicorn app.main:app --reload --port 8000

# VS Code extension setup
cd ../vscode-extension
npm install
npm run compile

# Press F5 in VS Code to launch extension development host
```

### 7.2 Testing

```bash
# Backend tests
cd backend
pytest tests/ -v

# Extension tests
cd vscode-extension
npm test
```

#### 7.2.1 Backend Test Examples

```python
# tests/test_embedding_service.py
import pytest
from app.services.embedding_service import EmbeddingService

@pytest.fixture(scope="module")
def embedding_service():
    return EmbeddingService()

def test_embed_single_text(embedding_service):
    text = "This is a test document about RAG systems."
    embedding = embedding_service.embed(text)
    assert embedding is not None
    assert len(embedding) == 768  # BGE base dimension

def test_embed_batch(embedding_service):
    texts = [
        "First document",
        "Second document",
        "Third document"
    ]
    embeddings = embedding_service.embed_batch(texts)
    assert len(embeddings) == 3
    assert all(len(e) == 768 for e in embeddings)

def test_embed_empty_text(embedding_service):
    embedding = embedding_service.embed("")
    assert embedding is not None
```

```python
# tests/test_skill_router.py
import pytest
from app.skills.router import SkillRouter
from app.skills.search import SearchSkill

@pytest.fixture
def router(services):
    router = SkillRouter()
    router.register(SearchSkill(services))
    return router

def test_route_command(router):
    skill = router.route("/search test query")
    assert skill is not None
    assert skill.name == "search"

def test_route_keyword(router):
    skill = router.route("find documents about architecture")
    assert skill is not None
    assert skill.name == "search"

def test_route_unknown_returns_search(router):
    skill = router.route("random question")
    assert skill.name == "search"  # default skill
```

```python
# tests/test_ocr_service.py
import pytest
from PIL import Image
import io
from app.models.ocr import OCRService

@pytest.fixture(scope="module")
def ocr_service():
    return OCRService()

def test_detect_language_english(ocr_service):
    # Create test image with English text
    img = Image.new('RGB', (200, 50), color='white')
    buffer = io.BytesIO()
    img.save(buffer, format='PNG')
    # Language detection test
    lang = ocr_service._detect_language(Image.open(io.BytesIO(buffer.getvalue())))
    assert lang in ['en', 'japan', 'ch']

def test_ocr_japanese(ocr_service):
    # Test with Japanese text image
    pass  # Add actual test with Japanese text image
```

```python
# tests/conftest.py
import pytest
from unittest.mock import MagicMock, AsyncMock

@pytest.fixture
def services():
    """Mock services for testing."""
    return {
        'embedding_service': MagicMock(),
        'vector_store': MagicMock(),
        'clip_classifier': MagicMock(),
        'vlm': MagicMock(),
        'ocr_service': MagicMock(),
        'confluence_client': AsyncMock(),
        'confluence_index': MagicMock(),
    }
```

### 7.3 Adding a New Skill

1. Create skill class in `app/skills/`:

```python
# app/skills/my_skill.py

from .base import BaseSkill, SkillInput, SkillOutput

class MySkill(BaseSkill):
    name = "my-skill"
    description = "Description of what this skill does"
    triggers = ["/myskill", "/ms"]
    keywords = ["keyword1", "keyword2"]
    requires_image = False
    max_images = 0

    async def execute(self, input: SkillInput) -> SkillOutput:
        # Implement skill logic
        return SkillOutput(
            text="Result context for Copilot",
            sources=[]
        )
```

2. Register in `app/skills/__init__.py`:

```python
from .my_skill import MySkill

def register_skills(router, services):
    """Register all skills with the skill router."""
    from .search import SearchSkill
    from .confluence import ConfluenceSkill
    from .explain_uml import ExplainUMLSkill
    from .compare import CompareSkill
    from .analyze import AnalyzeSkill
    from .docs import DocsSkill
    from .summary import SummarySkill
    from .index import IndexSkill

    router.register(SearchSkill(services))
    router.register(ConfluenceSkill(services))
    router.register(ExplainUMLSkill(services))
    router.register(CompareSkill(services))
    router.register(AnalyzeSkill(services))
    router.register(DocsSkill(services))
    router.register(SummarySkill(services))
    router.register(IndexSkill(services))

    # Add custom skills here
    # router.register(MySkill(services))
```

3. Add command to VS Code extension UI if needed.

---

## 8. Performance Considerations

### 8.1 CPU Optimization

| Component | Optimization |
|-----------|-------------|
| BGE Embedding | Batch processing, ONNX runtime |
| CLIP | Precomputed text embeddings |
| Qwen2-VL | float32 with low_cpu_mem_usage, image resize to 512px |
| PaddleOCR | MKL/OpenBLAS acceleration |

### 8.2 Expected Performance (Intel Xeon, 32 cores)

| Operation | Time |
|-----------|------|
| Text embedding (single) | ~50ms |
| Text embedding (batch 100) | ~2s |
| CLIP classification | ~500ms |
| OCR (single image) | ~1-2s |
| VLM description | ~15-30s |
| Vector search | ~10ms |
| Confluence API call | ~200-500ms |

### 8.3 Capacity Planning

| Metric | Recommended Limit |
|--------|-------------------|
| Total documents | 10,000 |
| Total images (with VLM descriptions) | 200 |
| Confluence pages in index | Unlimited |
| Concurrent users | 10 |
| User uploaded images per request | 2 |

---

## 9. Security Considerations

### 9.1 Authentication & Authorization
- Confluence API tokens stored in environment variables (never in code)
- No external API calls except Copilot (via VS Code)
- All AI models run locally
- Internal tool assumption - no user authentication required
- Consider adding basic auth if exposed beyond localhost

### 9.2 Input Validation
- File uploads validated for type (image/*, pdf, docx, xlsx, pptx)
- Maximum file size enforced (5MB for images, configurable for documents)
- Filename sanitization to prevent path traversal attacks
- Query input length limits to prevent DoS

### 9.3 Data Protection
- No sensitive data logged (queries may contain confidential info)
- Uploaded files stored temporarily and cleaned up periodically
- Vector embeddings don't expose original content directly
- Confluence content fetched on-demand, not stored permanently

### 9.4 Network Security
- Backend should only listen on localhost or internal network
- Use HTTPS if exposed beyond localhost
- Rate limiting on API endpoints recommended for production

---

## 10. Error Handling Strategy

### 10.1 Error Categories

| Category | Examples | Handling |
|----------|----------|----------|
| **User Error** | Invalid file type, query too long | Return 400 with clear message |
| **Service Unavailable** | Confluence down, VLM timeout | Return 503, suggest retry |
| **Internal Error** | Model loading failed, disk full | Return 500, log details |
| **Configuration Error** | Missing API token, invalid path | Fail startup with clear message |

### 10.2 Error Response Format

```json
{
  "error": {
    "code": "CONFLUENCE_UNAVAILABLE",
    "message": "Unable to connect to Confluence. Please try again later.",
    "details": "Connection timeout after 30s",
    "retry_after": 60
  }
}
```

### 10.3 Graceful Degradation

```python
# Example: Fallback when VLM is unavailable
async def process_image_with_fallback(image_bytes: bytes) -> str:
    try:
        # Try VLM first
        return await vlm.describe(image_bytes, prompt)
    except VLMTimeoutError:
        logger.warning("VLM timeout, falling back to OCR")
        # Fallback to OCR
        return ocr_service.extract_text(image_bytes)
    except VLMUnavailableError:
        logger.error("VLM unavailable")
        return "[Image analysis unavailable - VLM service not responding]"
```

### 10.4 Retry Strategy

| Operation | Max Retries | Backoff | Timeout |
|-----------|-------------|---------|---------|
| Confluence API | 3 | Exponential (1s, 2s, 4s) | 30s |
| VLM Inference | 1 | None | 60s |
| Vector Search | 2 | Linear (0.5s) | 5s |
| Embedding | 2 | Linear (0.5s) | 10s |

---

## 11. Future Enhancements

- [ ] Support for more document formats (HTML, Markdown, TXT)
- [ ] Incremental document updates (hash-based change detection)
- [ ] User feedback loop for retrieval quality improvement
- [x] ~~Caching layer for frequent queries~~ (Implemented)
- [x] ~~Multi-language OCR support (English, Japanese)~~ (Implemented)
- [ ] Confluence attachment indexing (images in pages)
- [ ] Admin UI for system management
- [ ] WebSocket support for real-time progress updates
- [ ] Batch document ingestion with progress tracking
- [ ] Export/import of vector database
- [ ] Query analytics and popular searches
- [ ] GPU acceleration support (optional, for faster VLM)

---

## Appendix A: Quick Start Guide

### Step 1: Download Models (On a machine with internet)

```bash
# Create virtual environment
python3 -m venv venv
source venv/bin/activate
pip install huggingface_hub

# Run download script
python scripts/download_models.py

# Copy models to target server
scp -r /opt/rag-models user@server:/opt/rag-models
```

### Step 2: Setup Backend Server

```bash
# On the Linux server
cd /opt/rag/backend
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt

# Configure environment
cp .env.example .env
vi .env  # Edit with your Confluence credentials

# Start server
uvicorn app.main:app --host 0.0.0.0 --port 8000
```

### Step 3: Install VS Code Extension

```bash
# On development machine
cd vscode-rag-extension
npm install
npm run compile

# Package extension
npx vsce package

# Install the .vsix file in VS Code
# Or press F5 to run in development mode
```

### Step 4: Configure Extension

1. Open VS Code Settings
2. Search for "RAG Assistant"
3. Set `Backend URL` to your server address
4. Press `Ctrl+Shift+P` → "Open RAG Chat"

---

## Appendix B: Skill Command Reference

| Command | Description | Requires Image |
|---------|-------------|----------------|
| `/search <query>` | Search all sources | No |
| `/confluence <query>` | Search Confluence only | No |
| `/docs <query>` | Search local documents only | No |
| `/analyze` | Analyze uploaded image | Yes (1) |
| `/explain-uml` | Explain UML diagram | Yes (1) |
| `/compare` | Compare two images | Yes (2) |
| `/summary <url>` | Summarize Confluence page | No |
| `/index refresh` | Refresh Confluence index | No |

---

## Appendix C: Troubleshooting

### Model Loading Errors

```bash
# Check model files exist
ls -la /opt/rag/models/

# Verify model integrity
python -c "from sentence_transformers import SentenceTransformer; SentenceTransformer('/opt/rag/models/bge-base-en-v1.5')"
```

### Confluence Connection Issues

```bash
# Test API connection
curl -u "email:token" "https://your-company.atlassian.net/wiki/rest/api/space"
```

### Performance Issues

```bash
# Check CPU usage
htop

# Monitor memory
free -h

# Check disk I/O
iotop
```

---

## Appendix D: Complete Configuration Reference

### Backend Environment Variables

| Variable | Required | Default | Description |
|----------|----------|---------|-------------|
| `RAG_HOST` | No | 0.0.0.0 | Server bind address |
| `RAG_PORT` | No | 8000 | Server port |
| `RAG_DEBUG` | No | false | Enable debug mode |
| `RAG_MODEL_DIR` | Yes | /opt/rag/models | Path to AI models |
| `RAG_DATA_DIR` | Yes | /opt/rag/data | Path to runtime data |
| `RAG_CONFLUENCE_BASE_URL` | Yes | - | Confluence instance URL |
| `RAG_CONFLUENCE_USERNAME` | Yes | - | Confluence username/email |
| `RAG_CONFLUENCE_API_TOKEN` | Yes | - | Confluence API token |
| `RAG_CONFLUENCE_SYNC_INTERVAL_HOURS` | No | 4 | Index sync frequency |
| `RAG_MAX_UPLOAD_IMAGES` | No | 2 | Max images per request |
| `RAG_MAX_IMAGE_SIZE_MB` | No | 5 | Max single image size |
| `RAG_CHUNK_SIZE` | No | 512 | Text chunk size |
| `RAG_CHUNK_OVERLAP` | No | 50 | Chunk overlap size |

### VS Code Extension Settings

| Setting | Default | Description |
|---------|---------|-------------|
| `rag-assistant.backendUrl` | http://localhost:8000 | Backend service URL |
| `rag-assistant.maxImages` | 2 | Max images to upload |

---

## Appendix E: API Reference

### POST /api/skill

Execute a skill with optional images.

**Request:**
```
Content-Type: multipart/form-data

skill: string (required) - Skill name
query: string (optional) - User query
images: File[] (optional) - Image files
```

**Response:**
```json
{
  "text": "Context text for LLM",
  "images": ["http://server/static/img1.png"],
  "sources": [
    {"title": "Doc Title", "path": "/path", "type": "document"}
  ],
  "metadata": {}
}
```

### POST /api/ingest

Ingest a document into the vector store.

**Request:**
```json
{
  "file_path": "/path/to/document.pdf",
  "source_type": "pdf"
}
```

### GET /api/status

Get system health status.

**Response:**
```json
{
  "status": "healthy",
  "version": "1.0.0",
  "models_loaded": ["bge", "clip", "vlm", "ocr"],
  "confluence_last_sync": "2025-01-27T10:00:00Z",
  "document_count": 1500,
  "image_count": 150
}
```

### POST /api/confluence/sync

Trigger Confluence index synchronization.

**Response:**
```json
{
  "status": "sync_completed",
  "stats": {"added": 10, "updated": 5, "removed": 2}
}
```

---

*Document Version: 1.1*
*Last Updated: 2025-01-27*
*Generated for RAG Tool project implementation.*
