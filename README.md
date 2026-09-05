# DataSentinel: Secure Autonomous BI Agent 🛡️📊

> **Autonomous BI Agent (LangGraph, FastAPI, Docker). Acts as a secure Data Analyst querying structured datasets (CSV) while adhering to corporate policies (PDFs) via PGVector RAG. Features sandboxed code execution, regex defensive parsing, and self-healing LLM loops.**

## 1. Project Overview
This project is an **Autonomous Business Intelligence (BI) Agent** designed to automate data analysis workflows for non-technical stakeholders (e.g., Managers, Executives). The agent intelligently queries structured tabular data (CSV/DBs) while dynamically adhering to unstructured enterprise policies (PDFs) using an advanced RAG (Retrieval-Augmented Generation) pipeline.

**The Core Value Proposition:**
Unlike standard LLM chatbots that hallucinate SQL or Python, this agent writes Pandas code, safely executes it in an isolated Docker container, and returns deterministic results. It acts as an autonomous Data Analyst, guided by strict corporate compliance rules.

## 2. Architecture & Workflow
The system follows a strict **Microservices-oriented architecture** with a clear separation of concerns between the API Gateway, Orchestrator, RAG Engine, and Execution Environment.

### How it Works (The Execution Flow)
1. **Data Ingestion:** The user uploads a CSV dataset and a Company Policy PDF via the Frontend.
2. **Policy Indexing:** The API parses the PDF using Layout-Aware OCR (`unstructured`), chunks it semantically, and stores the embeddings in a PostgreSQL vector database (`pgvector`).
3. **Query Reception:** The Manager submits a natural language request (e.g., "Which sales violated our discount policy?").
4. **Context Gathering (RAG):** The system searches the vector database for the specific policy clause (e.g., "Maximum discount is 15%").
5. **Code Generation:** The LangGraph Orchestrator prompts the LLM (Cohere Command R+) with the Database Schema, the retrieved legal policy, and strict environmental instructions.
6. **Code Parsing:** A robust regex-based parser strips out LLM conversational text, isolating pure Python code.
7. **Secure Execution:** The generated Pandas script is passed to a sandboxed Docker container (`python:3.11-slim`), where the dataset is mounted securely (`/mnt_data/`).
8. **Result Delivery:** The executed standard output (stdout) or error trace is returned. If an error occurs, the Orchestrator initiates a self-healing loop, feeding the error back to the LLM to fix its own code.

## 3. Technology Stack & Architectural Decisions

### A. Orchestration: LangGraph vs. LangChain Agents
*   **Selected:** `LangGraph`
*   **Why?** Standard LangChain Agents (`AgentExecutor`) behave like black boxes; they decide their own loops, which is catastrophic in enterprise environments where deterministic state machines are required. LangGraph provides cyclical graphs, allowing us to explicitly control retry limits, state transitions, and error-handling loops.

### B. Secure Execution: Docker Sandbox vs. Local `exec()`
*   **Selected:** `Docker API (docker SDK)`
*   **Why?** Using Python's native `exec()` or `eval()` to run LLM-generated code is a severe security vulnerability (Arbitrary Code Execution). By utilizing ephemeral Docker containers, we isolate the execution environment. The container lacks internet access (network isolation) and relies on read-only mounted volumes to process the data, ensuring zero system compromise even if the LLM generates malicious commands.

### C. Enterprise RAG: PGVector vs. Pinecone/ChromaDB
*   **Selected:** `PostgreSQL + pgvector` + `SQLAlchemy`
*   **Why?** In a production enterprise setting, data governance is key. SaaS vector stores (Pinecone) introduce data privacy concerns, while in-memory stores (Chroma) are not horizontally scalable. `pgvector` allows us to keep relational metadata (Tenant IDs) and vector embeddings in the exact same ACID-compliant database, simplifying infrastructure and enabling robust Multi-Tenancy isolation.

### D. Document Parsing: Layout-Aware Parsing (`unstructured`) vs. PyPDF2
*   **Selected:** `unstructured` with Tesseract OCR fallback.
*   **Why?** Standard PDF parsers extract raw text, destroying the structure of complex tables or multi-column enterprise policies. We opted for a Layout-Aware strategy (using CV models like YOLOX) to accurately detect headings, paragraphs, and tables, ensuring that semantic chunking retains context.

### E. LLM Provider: Cohere Command vs. OpenAI GPT-4
*   **Selected:** `Cohere Command-R+` (via Langchain Cohere Integration).
*   **Why?** Cohere's latest Command models are specifically fine-tuned for RAG architectures and code generation. They excel in enterprise compliance tasks and provide highly accurate document-grounded responses with a smaller latency footprint.

## 4. Enterprise-Grade Security & LLMOps

### A. Zero-Data-Retention Paradigm
**The Challenge:** Enterprises cannot send highly sensitive tabular data (e.g., PII, payroll, trade secrets) to external LLM APIs (OpenAI, Cohere) due to strict compliance laws (GDPR, HIPAA).
**The Solution:** This architecture implements a **Zero-Data-Retention** policy. 
*   **What goes to the LLM:** Only the *metadata* (Database Schema/Column names) and the natural language query.
*   **Where the data lives:** The actual CSV/Database rows never leave the secure, isolated Docker container on the host machine. The LLM generates the script, and the script executes locally against the data.

### B. LLMOps & Observability: LangSmith Integration
**The Challenge:** Managing AI agents in production often feels like a black box. If an agent fails or hallucinates, debugging the exact prompt and LLM response is nearly impossible without tooling.
**The Solution:** Integrated **LangSmith** at the orchestrator level. 
*   Every LLM trace, retrieval latency, and token consumption is logged.
*   Enables strict performance monitoring and cost auditing per Tenant ID, a critical requirement for scaling SaaS BI tools.

### C. State Management & Human-in-the-Loop (HITL)
**The Challenge:** Autonomous systems executing code can be dangerous if left entirely unchecked, especially for write operations or high-stakes BI reporting.
**The Solution:** Replaced in-memory state with **PostgreSQL Checkpointers (`AsyncPostgresSaver`)** inside the LangGraph workflow.
*   This persists the exact state of every thread (`thread_id`).
*   It lays the architectural foundation for a **Human-in-the-Loop (HITL)** approval process. The execution graph can pause, wait for a human manager to approve the generated Pandas code via the UI, and resume execution seamlessly.

### D. Handling the "Chatty LLM" & Path Hallucinations
**The Challenge:** LLMs are trained to be helpful assistants, meaning they often output markdown formatting, explanatory text (`### Explanation`), or hallucinate fake system paths (e.g., trying to save outputs to `/mnt/writable_dir/`). If passed directly to an execution engine, this causes catastrophic `SyntaxErrors` or `FileNotFound` exceptions.
**The Solution:** 
1.  **Strict Regex Parsing:** Implemented a defensive output parser (`re.search(r"```python\s*(.*?)\s*```")`) at the orchestration layer to strip out all non-code chatter.
2.  **Prompt Guardrails:** Engineered the System Prompt to explicitly forbid file creation on disk, forcing the LLM to output results strictly via `stdout` (`print()`), which the Docker Sandbox captures safely.

## 5. Deep Dive: File-by-File Architectural Breakdown

To understand the enterprise-grade nature of this system, here is a detailed breakdown of the codebase architecture, highlighting the specific best practices and technologies implemented in each microservice.

### 1. `core/api/production_gateway.py` (The API & State Management Layer)
*   **Technologies:** `FastAPI`, `uvicorn`, `langgraph.checkpoint.postgres`, `SQLAlchemy`.
*   **Best Practices Implemented:**
    *   **Asynchronous Gateway:** Built fully async to handle multiple concurrent Manager requests without blocking the main event loop.
    *   **State Persistence (Checkpointers):** Utilized `AsyncPostgresSaver` to persist the LangGraph thread states into a PostgreSQL database. This is critical for **Fault Tolerance** and enables **Human-in-the-Loop (HITL)** capabilities (pausing execution for human approval and resuming without state loss).
    *   **Dependency Injection:** Used FastAPI's `Depends(get_tenant_id)` to enforce strict multi-tenancy auth checks at the gateway level before any graph execution begins.

### 2. `core/agents/graph_orchestrator.py` (The AI Brain & Logic Layer)
*   **Technologies:** `LangGraph`, `LangChain`, `Cohere Command-R+`, Python `re` (Regex).
*   **Best Practices Implemented:**
    *   **Deterministic Workflows:** Replaced unpredictable standard Agents with a compiled `StateGraph`. The graph explicitly defines conditional edges (`condition_check_success`) to route the flow between the LLM and the Sandbox.
    *   **Self-Healing AI Loop:** If the Docker sandbox throws an error (e.g., `SyntaxError` or missing column), the graph loops back to the LLM (`coder_agent`), injecting the exact error traceback into the prompt for autonomous code correction (capped at 3 `MAX_RETRIES` to prevent infinite loops).
    *   **Defensive Parsing (Regex):** Built a strict Regex parser to extract pure Python code from the LLM's response, completely neutralizing the "Chatty LLM" problem and preventing hallucinated conversational text from crashing the execution engine.
    *   **LLMOps Observability:** Integrated `LangSmith` traces natively to monitor prompt performance, token usage, and graph execution depth.

### 3. `core/sandbox/docker_executor.py` (The Secure Execution Layer)
*   **Technologies:** `Docker SDK for Python`, `asyncio`.
*   **Best Practices Implemented:**
    *   **Air-Gapped Isolation:** The generated code runs inside an ephemeral `python:3.11-slim` container with **Network disabled** (`network_mode="none"`), preventing data exfiltration or malicious API calls.
    *   **Read-Only Volume Mounts:** The enterprise CSV data is mounted using strictly `ro` (Read-Only) permissions (`/mnt_data/:ro`). The AI can analyze the data but is physically prevented from altering or corrupting the source file.
    *   **Timeout & Resource Limits:** Implemented strict asynchronous timeouts to kill the container if the LLM writes an infinite `while` loop or highly inefficient Pandas operations, protecting host CPU/RAM resources.

### 4. `core/rag/enterprise_pdf_parser.py` (The Retrieval-Augmented Generation Layer)
*   **Technologies:** `unstructured` (Layout-Aware), `Tesseract OCR`, `PGVector`, `sentence-transformers`.
*   **Best Practices Implemented:**
    *   **Layout-Aware Parsing vs. Naive Text:** Instead of blindly extracting text (which destroys tables and headers), we use computer vision (`hi_res` strategy via YOLOX) to understand the visual structure of the PDF. This ensures that legal rules in tables or bullet points are chunked contextually.
    *   **Multi-Tenant Vector DB:** Embedded policies are saved in PostgreSQL (`pgvector`) alongside a `tenant_id` metadata tag. The retrieval strictly filters by tenant, ensuring zero data leakage between different enterprise clients.
    *   **Semantic Search Grounding:** Prevents LLM hallucinations by injecting the *exact* retrieved legal bounds (e.g., "Max discount 15%") directly into the System Prompt before code generation.

### 5. `core/data/schema_extractor.py` (The Data Profiling Layer)
*   **Technologies:** `SQLAlchemy Reflection`, `Pandas`.
*   **Best Practices Implemented:**
    *   **Zero-Data-Retention Exposure:** Automatically extracts column names and data types (e.g., `int64`, `O`) without sending any actual row data to the LLM. This provides the AI with perfect structural context while strictly adhering to Data Privacy compliance.

### 6. `frontend/app.py` (The Decoupled User Interface)
*   **Technologies:** `Streamlit`, `requests`.
*   **Best Practices Implemented:**
    *   **Dumb UI (Microservices Pattern):** The frontend contains zero business logic or database connections. It strictly interacts with the Backend via RESTful HTTP `POST` APIs (`/upload-policy`, `/analyze`).
    *   **Responsive State Management:** Uses Streamlit's `session_state` to handle mock authentication tokens and display execution progress fluidly to the end-user.

## 6. Roadmap & Scalability (Future Improvements)
While the core logic is production-ready, scaling to thousands of concurrent users would require:
1.  **Kubernetes Jobs:** Transitioning from the local Docker Daemon to Kubernetes (K8s) Jobs for the sandbox environment to handle distributed, concurrent code execution across a cluster.
2.  **Message Queues:** Implementing `Celery` + `Redis` or `RabbitMQ` to handle the asynchronous queueing of PDF indexing and code execution tasks, freeing up the FastAPI Gateway.
3.  **Authentication:** Replacing the mock API headers with robust OAuth2/JWT middleware for secure tenant validation.
4.  **UI Data Visualization:** Parsing the resulting DataFrames generated by the container and charting them dynamically using Streamlit's native plotting tools or a frontend framework (React).

---

## 7. How to Run Locally (Developer Setup)

### Prerequisites
Before you begin, ensure you have the following installed:
*   **Python 3.11+**
*   **Docker Desktop** (Running and configured for WSL2 if on Windows).
*   **PostgreSQL** (v15+) with the `pgvector` extension installed.
*   **Tesseract OCR** (For Layout-Aware PDF parsing).

### Step 1: Clone and Environment Setup
```bash
# Clone the repository
git clone [https://github.com/your-username/autonomous-bi-agent.git](https://github.com/your-username/autonomous-bi-agent.git)
cd autonomous-bi-agent

# Create and activate a virtual environment
python -m venv venv
# On Windows:
venv\Scripts\activate
# On macOS/Linux:
source venv/bin/activate

# Install the required dependencies
pip install -r requirements.txt

# Install specific unstructured dependencies for PDF parsing
pip install unstructured unstructured-inference unstructured[pdf]
```

### Step 2: Environment Variables
Create a `.env` file in the root directory of the project and add your API keys and Database URI:

```ini
# .env file
cohere_api_key="your_cohere_api_key_here"
LANGCHAIN_API_KEY="your_langsmith_api_key_here"
LANGCHAIN_TRACING_V2="true"
LANGCHAIN_PROJECT="autonomous_bi_project"

# PostgreSQL Connection String (Ensure pgvector is enabled in this DB)
DB_URI_RAG="postgresql+psycopg://postgres:password@localhost:5433/autonomous_bi"
```

### Step 3: Database Preparation
Ensure your PostgreSQL server is running and the database exists. The system uses SQLAlchemy to auto-generate the necessary tables (for LangGraph Checkpointers and Vector Embeddings) on the first run, but you must ensure the `pgvector` extension is active:
```sql
-- Run this in your PostgreSQL CLI tool (e.g., pgAdmin or psql)
CREATE EXTENSION IF NOT EXISTS vector;
```

### Step 4: Run the Backend Microservice (FastAPI)
The backend manages the LangGraph orchestration, RAG indexing, and Docker sandbox execution.
```bash
# Open a new Terminal (ensure venv is activated)
# Start the production API gateway
uvicorn core.api.production_gateway:app --reload --port 8000
```
*The API will be available at: `http://127.0.0.1:8000/docs`*

### Step 5: Run the Frontend UI (Streamlit)
The frontend serves as the interaction layer for the Manager to upload policies and query data.
```bash
# Open a second Terminal (ensure venv is activated)
# Run the Streamlit app from the root directory to ensure correct path resolution
python -m streamlit run frontend/app.py
```
*The UI will be available at: `http://localhost:8501`*

### Step 6: Testing the System
1.  Open the Streamlit UI.
2.  Upload the sample `uploaded_data.csv` (contains sales data).
3.  Upload the `policy.pdf` (contains the "15% max discount" rule).
4.  Wait for the green success message indicating the RAG engine has chunked and indexed the PDF.
5.  Type the query: *"Which sales violated the discount policy?"*
6.  Watch the backend terminal as the LangGraph agent generates code, safely executes it in Docker, and returns the strictly formatted result to the UI.
