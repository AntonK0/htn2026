# System Architecture Specification: Trades Worker Factory Onboarding AI Engine

## 1. System Overview & Core Philosophy

This specification outlines the technical architecture for an automated onboarding generator designed for factory and trades environments. The system ingests unstructured plant documentation (EHS policies, equipment manuals, chemical datasheets) along with custom site-specific instructions, and automatically transforms them into structured training modules containing three mediums:
1. **Simplified Text Overviews:** Written at a 6th-to-8th grade reading level for rapid scanning.
2. **Video Scripts & Render Jobs:** Async video creation prompts with visual directions and audio transcripts.
3. **Interactive Assessments:** Practical, scenario-driven multiple-choice questions.

### Infrastructure Foundations
* **Orchestration Core:** Built on top of **openJiuwen Core** (`openjiuwen`), leveraging asynchronous graph execution, `ReActAgent` nodes, and `WorkflowGraph` execution models.
* **Database & Memory Engine:** **Elasticsearch** is used for hybrid retrieval (BM25 keyword search + dense vector embeddings via Reciprocal Rank Fusion) and persistent execution audit trails.
* **Interactivity & Approvals:** Built-in Human-in-the-Loop (HITL) checkpoints to allow EHS (Environmental Health and Safety) officers to verify compliance prior to asset generation.

---

## 2. High-Level Architecture Diagram

```
[ User Input Payload ]
  ├── PDF Safety Manuals / MSDS
  └── Custom Site Instructions (e.g., "Line 4 LOTO Priority")
           │
           ▼
[ Stage 1: Document Processing & Vector Ingestion ]
  ├── Vision-Aware PDF Extractor (Unstructured / LlamaParse)
  ├── Embedding Model (text-embedding-3-small)
  └── Elasticsearch Hybrid Storage (`factory_safety_knowledge`)
           │
           ▼
[ Stage 2: openJiuwen Curriculum Orchestrator ]
  ├── Ingestion & Extraction ReActAgent
  ├── Module Topic Deduplication & Hierarchy Mapping
  └── JSON Module Blueprint Output
           │
           ▼
[ Human-in-the-Loop Approval Checkpoint (HITL) ]
  └── EHS Safety Manager Approves or Adjusts Blueprint
           │
           ▼
[ Stage 3: Parallel Sub-Agent Generation Cluster ]
  ├── Agent 3A: Concise Text Overview Generator
  ├── Agent 3B: Video Scripting & Async Render Trigger
  └── Agent 3C: Practical Quiz & Distractor Generator
           │
           ▼
[ Stage 4: Compliance Guardrail & QA Agent ]
  ├── Cross-Reference Generated Content vs. Source Elasticsearch Chunks
  └── Hallucination & OSHA Rules Safety Gate
           │
           ▼
[ Stage 5: Output Packaging & Storage ]
  ├── Save Assets & Logs to Elasticsearch (`generated_training_modules`)
  └── SCORM 1.2 / Mobile LMS Webhook Payload Output
```

---

## 3. Storage & Indexing Requirements (Elasticsearch)

The system requires three distinct Elasticsearch indices to manage raw knowledge, agent states, and finalized LMS assets.

### Index 1: `factory_safety_knowledge`
* **Purpose:** Stores parsed PDF chunks, regulatory standards, and machinery rules for RAG retrieval.
* **Fields & Types:**
  * `chunk_id` (`keyword`): Unique hash of the chunk.
  * `content` (`text`, Standard Analyzer): Full text content of the chunk.
  * `dense_vector` (`dense_vector`, cosine similarity, 1536 dims): Embedding vector generated from the chunk text.
  * `source_document` (`keyword`): Filename or URI of the uploaded manual.
  * `page_number` (`integer`): Physical page location in the source PDF.
  * `hazard_level` (`keyword`): Tagged severity level (`HIGH`, `MEDIUM`, `LOW`, `CRITICAL`).
  * `category` (`keyword`): Standardized safety category (e.g., `PPE`, `LOTO`, `Chemical`, `Emergency`).

### Index 2: `openjiuwen_agent_memory`
* **Purpose:** Backing store for openJiuwen's short and long-term memory system across workflow runs.
* **Fields & Types:**
  * `session_id` (`keyword`): Workflow execution UUID.
  * `agent_name` (`keyword`): Name of the executing agent (`IngestionAgent`, `QAAgent`).
  * `turn_type` (`keyword`): Type of entry (`system`, `user`, `assistant`, `tool_call`).
  * `message_payload` (`text`): Raw content string.
  * `timestamp` (`date`): Execution timestamp.

### Index 3: `generated_training_modules`
* **Purpose:** Production archive of approved onboarding assets for LMS sync and Kibana audit dashboards.
* **Fields & Types:**
  * `module_id` (`keyword`): Unique identifier (e.g., `MOD-LOTO-001`).
  * `facility_id` (`keyword`): Plant or site location code.
  * `title` (`text`): Human-readable module title.
  * `overview_text` (`text`): Low-reading-grade summary text.
  * `video_asset` (`object`):
    * `script_text` (`text`): Audio voiceover and visual directives.
    * `video_url` (`keyword`): CDN link to the rendered `.mp4`.
    * `status` (`keyword`): Render status (`PENDING`, `PROCESSING`, `READY`, `FAILED`).
  * `quiz_set` (`nested`): Array of question objects containing options, correct key, and safety rationale.
  * `approval_status` (`keyword`): `APPROVED`, `REJECTED`, or `PENDING_REVIEW`.

---

## 4. Multi-Agent Pipeline Workflow Specification

### Agent 1: Ingestion & Extraction Agent (`ReActAgent`)
* **Framework Type:** `openjiuwen.core.ReActAgent`
* **Model Configuration:** DeepSeek-R1 or GPT-4o (High-reasoning capability required)
* **Required Tools:**
  * `es_hybrid_safety_search`: Custom tool wrapper executing Elasticsearch Reciprocal Rank Fusion (RRF) search across `factory_safety_knowledge`.
* **Execution Logic:**
  1. Extract structured hazards, mandatory rules, and regulatory standard references from input PDFs.
  2. Map incoming custom instructions against extracted document chunks.
  3. Annotate each extracted rule with its associated hazard level and page citation.

### Agent 2: Curriculum Orchestrator (`WorkflowAgent` / Leader Node)
* **Framework Type:** `openjiuwen.core.WorkflowAgent`
* **Model Configuration:** GPT-4o or Claude 3.5 Sonnet
* **Execution Logic:**
  1. Receive the extracted hazards and rules from Agent 1.
  2. Group rules into logical, non-overlapping learning modules based on target trade roles (e.g., *Module 1: General Floor PPE*, *Module 2: Machine Guarding & Emergency Stops*).
  3. Sequence modules based on dependency rules: Foundational floor rules must precede specialized high-risk protocols (LOTO, Chemical handling).
  4. Enforce structural caps: Maximum 3 to 5 core learning objectives per module.
  5. Halt execution at the HITL approval boundary and emit the blueprint JSON.

### Parallel Sub-Agent Generation Cluster

#### Sub-Agent 3A: Concise Text Overview Generator
* **Role:** Produce visual, high-contrast, scannable reading material.
* **Style Guidelines:** 6th-to-8th grade reading level, short active-voice sentences, bold safety warnings, zero corporate jargon.

#### Sub-Agent 3B: Video Scripting & Render Trigger
* **Role:** Produce visual storyboards and trigger asynchronous video rendering jobs.
* **Execution Pattern:**
  1. Generate structured script split into `[VISUAL_DIRECTIVE]` and `[AUDIO_NARRATION]` pairs.
  2. Post script payload to external Video Rendering API (e.g., HeyGen, Shotstack, or internal programmatic generator).
  3. Receive asynchronous `job_id` and poll/register webhooks for task completion.

#### Sub-Agent 3C: Practical Quiz & Assessment Generator
* **Role:** Produce practical, situational assessments.
* **Requirements:**
  1. Generate 3 to 5 questions per module.
  2. Require at least one scenario-based practical question (e.g., *"You see line 3 displaying a red light with a low-pressure warning. What is your first step?"*).
  3. Provide exact distractor options and clear, rule-backed explanations for correct answers.

### Agent 4: Compliance & Safety QA Reviewer (`ReActAgent`)
* **Role:** Automated safety gate before module distribution.
* **Execution Logic:**
  1. Receive generated overviews, video scripts, and quizzes.
  2. Query Elasticsearch (`factory_safety_knowledge`) using RRF to fetch the original source citations for every stated rule.
  3. Validate that generated text does not contradict source PDF guidelines or omit critical PPE steps.
  4. If a contradiction or hallucination is detected, trigger a node retry with feedback flags sent back to the offending sub-agent.

---

## 5. Technical Implementation Requirements for Cursor

When implementing this codebase, ensure the project layout cleanly separates backend multi-agent services, storage tools, and frontend administrative interfaces:

### Full Directory Structure
```text
factory_onboarding_ai/
├── backend/
│   ├── config/
│   │   ├── elasticsearch.yaml      # Cluster endpoints and index mapping specs
│   │   └── agents.yaml             # Model targets, temperature settings, system prompts
│   ├── src/
│   │   ├── agents/
│   │   │   ├── ingestion_agent.py  # ReActAgent for parsing and indexing
│   │   │   ├── orchestrator.py     # Curriculum Orchestrator logic
│   │   │   ├── text_agent.py       # Overview generator
│   │   │   ├── video_agent.py      # Scripting and async rendering wrapper
│   │   │   ├── quiz_agent.py       # Assessment generator
│   │   │   └── qa_agent.py         # Guardrail and compliance reviewer
│   │   ├── storage/
│   │   │   ├── es_client.py        # Elastic connection manager
│   │   │   ├── retrieval_tools.py  # Hybrid BM25 + Vector search implementation
│   │   │   └── memory_store.py     # openJiuwen custom memory provider for ES
│   │   ├── services/
│   │   │   ├── video_api.py        # Async polling client for video rendering APIs
│   │   │   └── pdf_processor.py    # Document parsing and chunking utility
│   │   ├── api/
│   │   │   ├── routes_workflow.py  # FastAPI endpoints for running pipeline
│   │   │   ├── routes_hitl.py      # Endpoint for EHS manager approvals
│   │   │   └── webhooks.py         # Async callback receiver for video API completion
│   │   └── workflow/
│   │       ├── pipeline.py         # openJiuwen WorkflowGraph orchestrator
│   │       └── hitl.py             # Human-in-the-Loop review state handlers
│   ├── tests/
│   │   ├── test_es_retriever.py
│   │   └── test_pipeline_graph.py
│   ├── main.py                     # Backend server entry point (FastAPI / ASGI)
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   ├── assets/                 # Icons, branding assets, global styles
│   │   ├── components/
│   │   │   ├── DocumentUploader.jsx# PDF and instruction submitter
│   │   │   ├── ModuleBlueprint.jsx # HITL module approval editor
│   │   │   ├── VideoPlayer.jsx     # Video preview player with script captions
│   │   │   └── QuizViewer.jsx      # Assessment verification widget
│   │   ├── pages/
│   │   │   ├── Dashboard.jsx       # Pipeline monitoring & status view
│   │   │   ├── ModuleEditor.jsx    # EHS review & editing interface
│   │   │   └── WorkerOnboarding.jsx# Mobile-friendly trade worker training viewer
│   │   ├── services/
│   │   │   └── api.js              # Axios / Fetch client connecting to FastAPI
│   │   ├── App.jsx                 # Main application router
│   │   └── main.jsx                # Frontend mounting point
│   ├── public/
│   ├── package.json
│   └── vite.config.js
├── ARCHITECTURE.md
└── docker-compose.yml              # Local setup for Elasticsearch and backend runtime
```

### Key Python Dependencies (Backend)
* `openjiuwen` (Core Agent & Workflow Engine)
* `elasticsearch` (Official Elasticsearch Python Client)
* `fastapi` & `uvicorn` (REST API server for frontend interaction)
* `pydantic` (Data Validation and JSON Schema generation)
* `asyncio` & `aiohttp` (Asynchronous I/O execution)
* `unstructured` or `pypdf` (Document ingestion)

### System Execution Guardrails
1. **Async Engine Native:** All agent nodes must be implemented asynchronously using `asyncio` and openJiuwen's `execute_async` interfaces to prevent blocking during API calls and video generation polling.
2. **Strict Schema Validation:** All inter-agent data passing must rely on Pydantic models to guarantee zero key-error failures across node transitions.
3. **Resilient Retry Policy:** External API calls (Elasticsearch queries, LLM completions, Video API webhooks) must be wrapped in exponential backoff policies.