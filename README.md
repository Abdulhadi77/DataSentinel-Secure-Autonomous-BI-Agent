# DataSentinel: Secure Autonomous BI Agent 🛡️📊

> **Autonomous BI Agent (LangGraph, FastAPI, Docker). Acts as a secure Data Analyst querying structured datasets (CSV) while adhering to corporate policies (PDFs) via PGVector RAG. Features sandboxed code execution, regex defensive parsing, and self-healing LLM loops.**

## 1. Project Overview
This project is an **Enterprise-Grade Autonomous Business Intelligence (BI) Agent** designed to automate complex data analysis workflows. Unlike standard LLM chatbots that frequently hallucinate SQL or Python code, this agent acts as an autonomous Data Analyst. It writes deterministic Pandas code, executes it securely in an isolated environment, and dynamically adheres to unstructured enterprise legal policies (PDFs) using an advanced Retrieval-Augmented Generation (RAG) pipeline.

## 2. System Architecture & Execution Workflow
The system follows a strict **Microservices-oriented architecture** with a clear separation of concerns between the API Gateway, Orchestrator, RAG Engine, and Execution Environment.

**The Execution Flow:**
1. **Data Ingestion:** Upload structured data (CSV/DB) and unstructured Company Policy (PDF) via the decoupled Frontend.
2. **Layout-Aware Indexing:** The API parses the PDF using Computer Vision (`unstructured` + `YOLOX`), chunks it semantically, and stores the embeddings in a PostgreSQL vector database (`pgvector`).
3. **Context Gathering (RAG):** When a user asks a query, the system retrieves the specific legal constraint (e.g., "Maximum discount is 15%").
4. **Code Generation:** The LangGraph Orchestrator prompts the LLM (Cohere Command-R+) with the Database Schema, retrieved policies, and strict environmental constraints.
5. **Defensive Parsing:** A custom Regex parser strips out LLM conversational text, isolating pure executable Python code.
6. **Air-Gapped Execution:** The script is passed to a sandboxed Docker container where the dataset is mounted read-only.
7. **Self-Healing Loop:** If an execution error occurs (e.g., `SyntaxError`), the Orchestrator catches the traceback and feeds it back to the LLM to auto-fix the code (capped at 3 retries).

## 3. Key Architectural Features & Enterprise Security

### 🔐 Zero-Data-Retention Paradigm
**Challenge:** Enterprises cannot send highly sensitive tabular data to external LLMs due to compliance laws (GDPR, HIPAA).
**Solution:** Only metadata (Schema) and natural language queries are sent to the LLM API. The actual CSV/Database rows **never leave the secure Docker container** on the host machine. The AI writes the script, and the script executes locally.

### 🛑 Handling the "Chatty LLM" & Path Hallucinations
**Challenge:** LLMs often output markdown formatting (`### Explanation`) or hallucinate fake system paths, crashing standard execution engines.
**Solution:** 
* **Strict Regex Parsing:** Implemented a robust output parser (`re.search(r"```python\s*(.*?)\s*```")`) at the orchestration layer to strip all non-code chatter.
* **Prompt Guardrails:** Engineered the System Prompt to forbid file creation on disk, forcing the LLM to output results strictly via `stdout` (`print()`).

### 💾 State Management & Human-in-the-Loop (HITL)
**Challenge:** Autonomous systems executing code require checkpointing for safety and auditing.
**Solution:** Replaced in-memory state with **PostgreSQL Checkpointers (`AsyncPostgresSaver`)** inside the LangGraph workflow. This persists the exact state of every thread, laying the foundation for HITL approval processes (pausing execution for a human manager to approve the generated code).

### 👁️ LLMOps & Observability
Integrated **LangSmith** at the orchestrator level. Every LLM trace, retrieval latency, token consumption, and graph execution depth is logged, enabling strict performance monitoring and cost auditing per Tenant ID.

## 4. Technology Stack & Architectural Trade-offs

* **Orchestration (LangGraph vs. Standard Agents):** Standard agents behave like unpredictable black boxes. We selected `LangGraph` to provide cyclical state machines, allowing explicit control over retry limits and error-handling loops.
* **Execution (Docker Sandbox vs. Native `exec()`):** Using Python's native `exec()` is a critical security vulnerability. We utilize ephemeral Docker containers (`python:3.11-slim`) with **Network disabled** and **Read-Only volume mounts** to prevent arbitrary code execution attacks.
* **Vector DB (PGVector vs. SaaS Pinecone):** `pgvector` allows us to keep relational metadata (Tenant IDs) and vector embeddings in the exact same ACID-compliant PostgreSQL database, ensuring robust Multi-Tenancy isolation.
* **Document Parsing (`unstructured` vs. PyPDF2):** Standard parsers destroy table structures. We opted for a Layout-Aware strategy (`hi_res` with Tesseract OCR) to accurately detect headings and tables, ensuring semantic chunking retains context.

## 5. Deep Dive: Codebase Anatomy

* `core/api/production_gateway.py`: The async API layer using **FastAPI**. Implements `AsyncPostgresSaver` for state persistence and strict `Tenant ID` dependency injection for multi-tenancy.
* `core/agents/graph_orchestrator.py`: The AI Brain. Contains the compiled `StateGraph`, the deterministic Self-Healing loops, and the strict Regex Output Parser.
* `core/sandbox/docker_executor.py`: The Secure Execution Layer using **Docker SDK**. Enforces air-gapped isolation, timeouts, and resource limits.
* `core/rag/enterprise_pdf_parser.py`: The RAG engine utilizing `unstructured` and `PGVector` for semantic search grounding.
* `core/data/schema_extractor.py`: Extracts zero-data-retention schemas via **SQLAlchemy Reflection** to provide structural context to the LLM.
* `frontend/app.py`: A decoupled **Streamlit** UI operating on the Microservices pattern, interacting strictly via RESTful HTTP `POST` APIs.

## 6. Roadmap & Scalability
To scale this architecture to thousands of concurrent enterprise users:
1. **Kubernetes (K8s):** Transition from local Docker Daemons to K8s Jobs for distributed, concurrent code execution across a cluster.
2. **Message Queues:** Implement `Celery` + `Redis` to handle the asynchronous queueing of PDF indexing and Sandbox tasks.
3. **Authentication:** Replace mock API headers with robust OAuth2/JWT middleware.

---

## 7. How to Run Locally (Developer Setup)

### Prerequisites
* Python 3.11+
* Docker Desktop (Running and configured)
* PostgreSQL (v15+) with the `pgvector` extension installed.
* Tesseract OCR (Installed on your OS and added to PATH).

### Step 1: Environment Setup
```bash
git clone [https://github.com/your-username/autonomous-bi-agent.git](https://github.com/your-username/autonomous-bi-agent.git)
cd autonomous-bi-agent

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install core dependencies
pip install -r requirements.txt

# Install specific dependencies for Layout-Aware PDF parsing
pip install unstructured unstructured-inference unstructured[pdf]
