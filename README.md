# Enterprise HR Policy & Employee Support Agentic RAG Copilot

An end-to-end Forward Deployed Engineer (FDE) project that converts an Agentic RAG workflow into a deployable internal HR product using LangGraph, FastAPI, Pinecone, OpenAI, Tavily, HTML, CSS, and JavaScript.

## 1. Business Problem

### Customer
NovaRetail, a fictional 3,000-employee retail company.

### Problem
The HR team maintains many internal documents: leave policies, remote-work rules, payroll guidance, benefits information, onboarding procedures, conduct policies, and HR operations runbooks.

Employees still send repetitive HR questions because they do not know where the correct policy lives, keyword search returns too many documents, generic chatbots may invent policy details, internal documents may not cover current public regulations, and some questions require fresh external information.

### Example
An employee asks:

> “How many annual leave days do employees receive?”

The answer exists in the private company HR knowledge base, so the system should answer from internal policy without searching the public internet.

Another employee asks:

> “What are the latest public holiday rules in Bangladesh?”

The internal KB may not contain current public information. The system should recognize weak private evidence, use external search, grade the evidence, and clearly identify the answer as external information requiring HR validation.

### Business Goal
Build a secure HR Policy Copilot that:

1. Searches trusted private HR knowledge first.
2. Checks whether retrieved evidence is sufficient.
3. Uses web search only when private knowledge is insufficient.
4. Rewrites weak queries and retries.
5. Generates grounded answers.
6. Shows the LangGraph decision path for transparency and debugging.
7. Lets authorized HR staff add new company documents.

## 2. Why This Is an FDE Project

A Forward Deployed Engineer does more than build an LLM notebook. The FDE translates a customer problem into a usable product:

```text
Customer Problem
      ↓
Discovery & Requirements
      ↓
Solution Architecture
      ↓
Data / Knowledge Integration
      ↓
Agentic RAG Development
      ↓
API Development
      ↓
User Interface
      ↓
Security + Audit + Testing
      ↓
Deployment
      ↓
Observe + Improve
```

## 3. Simple Architecture

![System architecture](docs/architecture.png)

The picture above shows the system components and how a request moves between them: the web UI sends a question to FastAPI, the LangGraph agent coordinates retrieval and decisions, and OpenAI, Pinecone, or Tavily are called as needed. The private HR knowledge base is always checked before public web search.

```text
Employee / HR User
        ↓
HTML/CSS/JavaScript Web UI
        ↓ POST /api/chat
FastAPI
        ↓
LangGraph Agentic RAG Controller
        ↓
 ┌───────────────┬─────────────────┐
 ↓               ↓
Private HR KB    Tavily Web Search
Pinecone         (fallback only)
 └───────┬───────┘
         ↓
OpenAI LLM
Grounded Answer
```

## 4. Agentic RAG Workflow

```mermaid
flowchart TD
            START([Question received]) --> ROUTE[Route question with OpenAI]
            ROUTE -->|Greeting or casual chat| DIRECT[Answer directly]
            DIRECT --> END1([Done])
            ROUTE -->|HR or policy question| KB[Retrieve relevant chunks from private Pinecone KB]
            KB --> GRADEKB[Grade private evidence]
            GRADEKB -->|Good| ANSWERKB[Generate answer using private KB only]
            ANSWERKB --> END2([Done])
            GRADEKB -->|Weak| WEB[Search public web with Tavily]
            WEB --> GRADEWEB[Grade web evidence with OpenAI]
            GRADEWEB -->|Good| ANSWERWEB[Generate answer from web evidence; flag as external]
            ANSWERWEB --> END3([Done])
            GRADEWEB -->|Weak and retries remain| REWRITE[Rewrite the query with OpenAI]
            REWRITE --> KB
            GRADEWEB -->|Weak and retry limit reached| STOP[Return insufficient-evidence response]
            STOP --> END4([Done])
```

### How to read the graph

1. **Route:** The agent uses the OpenAI model to classify the message. Greetings skip retrieval; HR and policy questions continue to the private knowledge base.
2. **Search company knowledge first:** Pinecone returns the most relevant document chunks. OpenAI grades whether those chunks fully support an answer.
3. **Use internal evidence when it is enough:** A good grade leads to an answer grounded only in company documents, with private-source citations.
4. **Fall back to the web only when needed:** A weak private-KB grade triggers Tavily search. A good web grade produces an answer clearly identified as external information, not company policy.
5. **Retry carefully:** If web evidence is weak and retries remain, OpenAI rewrites the query and the graph searches the private KB again. If evidence is still weak after the configured retry limit, the agent stops and says it cannot answer confidently.

The visible trace returned by `/api/chat` follows the path taken through this graph. The default `MAX_RETRIES` value is `1`.

### API endpoints

| Method | Endpoint | Purpose | Request / access |
|---|---|---|---|
| `GET` | `/` | Serves the employee web interface. | No authentication. |
| `GET` | `/api/health` | Returns service health and the configured application name. | No authentication. |
| `POST` | `/api/chat` | Runs the Agentic RAG workflow. Returns `answer`, `source_used`, `trace`, `citations`, and `rewritten_query`. | JSON body: `{"question":"How many annual leave days do employees receive?"}`. No API-key header. |
| `POST` | `/api/ingest` | Uploads and indexes an HR document in Pinecone. Supports `.pdf`, `.txt`, `.md`, and `.docx`. | Multipart form field `file`; header `X-Admin-Key` must match `ADMIN_API_KEY`. |
| `GET` | `/docs` | Interactive Swagger UI for the API. | Available while the app is running. |
| `GET` | `/redoc` | ReDoc API reference. | Available while the app is running. |
| `GET` | `/openapi.json` | OpenAPI schema used by the interactive API docs. | Available while the app is running. |

### API keys and configuration

Keep real credentials in your local `.env` file. Do not commit that file or paste secrets into source code.

| Variable | Required for | Notes |
|---|---|---|
| `OPENAI_API_KEY` | Chat, routing, grading, query rewriting, answer generation, and embeddings. | Required for the assistant's AI and embedding calls. |
| `PINECONE_API_KEY` | Private HR knowledge retrieval and document ingestion. | Needed to connect to and manage the configured vector index. |
| `TAVILY_API_KEY` | Web fallback when private HR evidence is insufficient. | Not used for greetings or answers supported by the private KB. |
| `ADMIN_API_KEY` | `POST /api/ingest`. | Local default is `change-me-in-production`; replace it with a strong private value before deployment. Send it as `X-Admin-Key`. |
| `PINECONE_INDEX_NAME` | Pinecone index selection. | Optional; default: `fde-hr-policy-rag`. |
| `PINECONE_NAMESPACE` | Namespace within the Pinecone index. | Optional; default: `company-hr-kb`. |
| `OPENAI_MODEL` | Chat model selection. | Optional; default: `gpt-4o-mini`. |
| `EMBEDDING_MODEL` | Embedding model selection. | Optional; default: `text-embedding-3-small`. |
| `APP_ENV`, `TOP_K`, `MAX_RETRIES` | Application behavior. | Optional; defaults are `development`, `4`, and `1`, respectively. |

## 5. Technology Stack

| Layer | Technology | Purpose |
|---|---|---|
| Agent workflow | LangGraph | Stateful routing and conditional decisions |
| LLM | OpenAI | Routing, grading, rewriting, answer generation |
| Embeddings | OpenAI `text-embedding-3-small` | Vector embeddings |
| Vector DB | Pinecone | Private enterprise HR knowledge base |
| External search | Tavily | Fallback when company HR KB is insufficient |
| API | FastAPI | Backend and REST endpoints |
| Frontend | HTML/CSS/JavaScript | Employee-facing interface |
| Audit | SQLite | Decision-path logging |
| Packaging | Docker | Reproducible deployment |

## 6. Project Structure

```text
Enterprise-HR-Policy-Agentic-RAG-Copilot/
├── app/
│   ├── api/routes.py
│   ├── core/config.py
│   ├── core/logging.py
│   ├── rag/state.py
│   ├── rag/vectorstore.py
│   ├── rag/workflow.py
│   ├── services/audit.py
│   ├── services/ingestion.py
│   └── main.py
├── data/sample_kb/
│   ├── company_hr_handbook.md
│   └── hr_operations_runbook.md
├── static/
│   ├── css/style.css
│   └── js/app.js
├── templates/index.html
├── uploads/
├── Dockerfile
├── ingest_sample_kb.py
├── requirements.txt
├── run.py
└── README.md
```

## 7. Setup

### Step 1 — Create and activate a virtual environment

```bash
python -m venv venv
```

Windows:

```bash
venv\Scripts\activate
```

macOS/Linux:

```bash
source venv/bin/activate
```

### Step 2 — Install dependencies

```bash
pip install -r requirements.txt
```

### Step 3 — Configure environment

Create a `.env` file in the project root and add your provider credentials and settings. Use real keys from OpenAI, Pinecone, and Tavily; the admin key is a secret value you choose yourself.

```env
OPENAI_API_KEY=your_openai_api_key_here
TAVILY_API_KEY=your_tavily_api_key_here
PINECONE_API_KEY=your_pinecone_api_key_here
PINECONE_INDEX_NAME=fde-hr-policy-rag
PINECONE_NAMESPACE=company-hr-kb
OPENAI_MODEL=gpt-4o-mini
EMBEDDING_MODEL=text-embedding-3-small
ADMIN_API_KEY=change-me-in-production
APP_ENV=development
TOP_K=4
MAX_RETRIES=1
```

`OPENAI_API_KEY` and `PINECONE_API_KEY` are needed for private-KB question answering. `TAVILY_API_KEY` is needed only when the workflow falls back to web search. `ADMIN_API_KEY` protects document uploads. See [API keys and configuration](#api-keys-and-configuration) above for details and defaults.

### Step 4 — Load sample HR knowledge

```bash
python ingest_sample_kb.py
```

### Step 5 — Run the application

```bash
python run.py
```

Open `http://127.0.0.1:8080` and FastAPI docs at `http://127.0.0.1:8080/docs`.

## 8. Classroom Demo Scenarios

### Demo A — Private KB Success
Ask: **How many annual leave days do employees receive?**

Expected path:

```text
Router → KB
Private KB Retrieval
KB Grade → GOOD
Generate from Private KB
```

### Demo B — Company Policy Question
Ask: **How many days per week can I work remotely?**

Expected result: answer from the internal HR handbook, without web search.

### Demo C — External / Current Information
Ask: **What are the latest public holiday rules in Bangladesh?**

Expected path when internal HR documents are insufficient:

```text
Router → KB
Private KB Retrieval
KB Grade → WEAK
Tavily Search
Web Grade → GOOD
Web Answer
```

### Demo D — Weak Query Rewrite
Ask an ambiguous HR question such as: **What happens if mine is wrong?**

If neither private nor web evidence is sufficient, the workflow can rewrite the query, retry the KB, and eventually stop with insufficient evidence rather than hallucinating.

## 9. What Changed From the IT Support Reference

The application structure, graph topology, API shape, retrieval logic, ingestion layer, audit layer, Docker setup, and frontend behavior remain the same. Only domain-specific elements were changed: HR prompts, HR configuration names, UI wording, example questions, sample documents, and documentation.
