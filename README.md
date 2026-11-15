# AI Support Desk

A multi-role AI customer support system where learners, support agents, and administrators use a single unified chat interface. Instead of hiding buttons in the UI or relying on prompt instructions for security, the system enforces permissions at the protocol, middleware, and data layers using end-to-end JWT identity propagation.

---

## Architecture Overview

The application runs as **three separate processes across two ports**:

1. **SQLite Database (`database/support.db`)**: Zero-dependency relational store initialized from `schema.sql` and `seed.sql`.
2. **MCP Tool Server (`127.0.0.1:8001/mcp`)**: Exposes 25 domain tools over `streamable-http` transport, protected by JWT verification and scope-filtering middleware.
3. **FastAPI Backend & Web UI (`127.0.0.1:8000`)**: Handles login, issues signed JWTs, runs a bounded LLM tool-calling loop, and serves the single-page web client.

```text
Browser (UI)
  │
  ├── POST /api/login ──────────────────► FastAPI Backend (:8000)
  │   ◄── JWT (sub, role, scope) ───────────┘ (verifies bcrypt hash)
  │
  └── POST /api/chat + Bearer JWT ──────► FastAPI Backend (:8000)
                                            │
                                            ├── tools/list + Same JWT ──► MCP Server (:8001/mcp)
                                            │   ◄── Filtered Tool List ─────┘ (JWTTokenVerifier + check_access)
                                            │
                                            └── tools/call + Same JWT ──► MCP Server (:8001/mcp)
                                                ◄── Result / Access Denied ──┘ (Row-level & business guards)
```

### Key Engineering Decisions

* **Single Identity End-to-End**: The browser never talks directly to port `8001`, and the backend holds no privileged service account. On every `/api/chat` request, FastAPI opens a per-request MCP client session forwarding the user's exact `Authorization: Bearer <jwt>` header.
* **Three-Layer Security Enforcement**:
  1. **Authentication (`JWTTokenVerifier`)**: Validates the HS256 signature, expiration, issuer (`iss`), and audience (`aud`) before any MCP message is processed.
  2. **Authorization Middleware (`check_access`)**: Filters `tools/list` so the LLM only sees permitted tools (reducing prompt bloat and tool-selection errors), and gates `tools/call` to block direct or forged requests with a structured `CallToolResult(is_error=True)`.
  3. **Row-Level & Business-Rule Guards (Inside Tools)**: Reads caller identity directly from `get_access_token()` (never trusting model-supplied user IDs), enforces ticket ownership, caps support course extensions at 30 days, and requires matching `course_id` + `course_title` pairs on financial or destructive operations.
* **Deterministic Execution Trace**: Every chat response returns a structured `tool_calls` trace (`name`, `arguments`, `is_error`, `result`) rendered directly below the assistant message for transparent debugging.

---

## Project Structure

```text
├── common/
│   ├── config.py          # Environment variables, ports, JWT claims, model config
│   ├── permissions.py     # Role-to-scope mappings & per-user custom scope resolution
│   ├── auth.py            # PyJWT encoding/decoding with strict claim verification
│   └── prompts.py         # Dynamic per-user system prompt & server instructions
├── database/
│   ├── schema.sql         # 7 relational tables with foreign keys & CHECK constraints
│   ├── seed.sql         # Relative-date dataset (enrollments, tickets, FAQs)
│   ├── db.py              # SQLite connection manager & database reset utility
│   └── models.py          # Dataclass ORM layer (all SQL queries isolated here)
├── mcp_server/
│   └── server.py          # MCPServer (mcp >= 2.3), TOOL_SCOPES, middleware, 25 tools
├── backend/
│   └── app.py             # FastAPI endpoints, per-request MCP client, 8-round chat loop
├── frontend/
│   └── index.html         # Single-file chat interface with live tool-call trace inspector
├── .env.example           # Required environment variables template
└── pyproject.toml         # Dependencies managed via uv
```

---

## Roles & Permission Scopes

Permissions use `resource:action` scope strings embedded in the user's JWT at login (with optional per-user overrides via `users.scopes`):

| Role | Default Scopes | Capabilities |
| :--- | :--- | :--- |
| **`learner`** | `faq:read`, `courses:read`, `courses:purchase`, `me:read`, `tickets:read`, `tickets:create`, `tickets:reply` | View own enrollments, payments, and certificates; purchase courses (simulated); raise and reply to own tickets. |
| **`support`** | `faq:read`, `courses:read`, `tickets:read`, `tickets:read_all`, `tickets:manage`, `learners:read`, `access:extend`, `enrollments:remove` | Inspect all tickets and internal notes, manage/assign the support queue, search learners, extend access (max 30 days), remove enrollments. |
| **`admin`** | All `support` scopes + `access:extend_unlimited`, `certificates:issue`, `faq:write`, `stats:read`, `users:manage` | Unlimited access extensions, issue 100%-progress certificates, publish FAQs, view support metrics, manage user roles and scopes. |

---

## Getting Started

### 1. Prerequisites & Installation
Make sure Python 3.11+ and [`uv`](https://docs.astral.sh/uv/) are installed, then sync dependencies:

```bash
uv sync
```

### 2. Configure Environment Variables
Copy `.env.example` to `.env` and set your OpenAI API key along with a strong `JWT_SECRET` (minimum 32 characters; the application refuses to start otherwise):

```bash
cp .env.example .env
python -c "import secrets; print(secrets.token_urlsafe(48))"
```

Example `.env`:
```env
OPENAI_API_KEY=sk-...
JWT_SECRET=<paste-generated-secret-here>
JWT_EXPIRY_MINUTES=60
LLM_MODEL=gpt-4o-mini
BACKEND_PORT=8000
MCP_PORT=8001
```

### 3. Initialize the Database
Generate `database/support.db` from the schema and relative-date seed data (re-run anytime to reset state):

```bash
uv run python -m database.db
```

### 4. Start the Servers
Open two terminals and start the MCP tool server first, followed by the FastAPI backend:

**Terminal 1: MCP Server (`127.0.0.1:8001/mcp`)**
```bash
uv run python -m mcp_server.server
```

**Terminal 2: FastAPI Backend & UI (`127.0.0.1:8000`)**
```bash
uv run python -m backend.app
```

Open `http://127.0.0.1:8000` in your browser.

---

## Seeded Accounts

All seeded accounts use the password **`password123`**:

| Email | Role | Pre-Seeded Scenario |
| :--- | :--- | :--- |
| `rahul@learner.com` | `learner` | Java course at 100% progress (no certificate issued yet); failed payment for Agentic AI yesterday. |
| `priya@learner.com` | `learner` | Spring Boot access expired 3 days ago; Java course at 25% progress. |
| `aman@learner.com` | `learner` | DevOps course purchased 4 days ago. |
| `neha@support.com` | `support` | Support agent working the open ticket queue. |
| `vikram@support.com` | `support` | Secondary support agent for ticket assignment. |
| `akshay@admin.com` | `admin` | Full administrative access across all 25 tools. |

### Suggested Test Flows
* **Dynamic Tool Visibility**: Log in as `rahul@learner.com` vs. `akshay@admin.com` and compare the sidebar tool lists (`tools/list` exposes 10 tools to learners vs. 25 to admins).
* **Autonomous Ticket Triage**: As Rahul, send *"I want the certificate for the Java complete course"*. The assistant verifies 100% completion and automatically creates a categorized support ticket.
* **Permission Boundary Checks**: As Neha (`support`), try issuing a certificate or extending course access by 60 days to inspect middleware and tool-level `Access denied` responses in the execution trace.

---

## Roadmap
* **Dynamic RBAC & Schema Config**: Move `ROLE_SCOPES`, `TOOL_SCOPES`, ticket categories, and priorities out of static dicts and SQL `CHECK` constraints into database tables or external config.
* **Hybrid RAG Knowledge Base**: Replace SQL-based `search_faq` with a vector + keyword retrieval pipeline over markdown documentation.
* **Production Hardening**: Add domain guardrails to `SYSTEM_PROMPT`, server-side conversation history persistence, idempotency keys on mutating tools, and PostgreSQL support.