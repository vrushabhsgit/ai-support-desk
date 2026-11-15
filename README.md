# AI Support Desk

A multi-role AI customer support system where learners, support agents, and administrators use a single unified chat interface[cite: 1, 2]. Instead of hiding buttons in the UI or relying on prompt instructions for security, the system enforces permissions at the protocol, middleware, and data layers using end-to-end JWT identity propagation[cite: 2, 4, 17, 20].

---

## Architecture Overview

The application runs as **three separate processes across two ports**[cite: 4]:

1. **SQLite Database (`database/support.db`)**: Zero-dependency relational store initialized from `schema.sql` and `seed.sql`[cite: 3, 4, 5].
2. **MCP Tool Server (`127.0.0.1:8001/mcp`)**: Exposes 25 domain tools over `streamable-http` transport, protected by JWT verification and scope-filtering middleware[cite: 4, 15, 18, 19].
3. **FastAPI Backend & Web UI (`127.0.0.1:8000`)**: Handles login, issues signed JWTs, runs a bounded LLM tool-calling loop, and serves the single-page web client[cite: 4, 5, 22, 25].

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

* **Single Identity End-to-End**: The browser never talks directly to port `8001`, and the backend holds no privileged service account[cite: 4, 24]. On every `/api/chat` request, FastAPI opens a per-request MCP client session forwarding the user's exact `Authorization: Bearer <jwt>` header[cite: 23, 24].
* **Three-Layer Security Enforcement**:
  1. **Authentication (`JWTTokenVerifier`)**: Validates the HS256 signature, expiration, issuer (`iss`), and audience (`aud`) before any MCP message is processed[cite: 9, 10, 15].
  2. **Authorization Middleware (`check_access`)**: Filters `tools/list` so the LLM only sees permitted tools (reducing prompt bloat and tool-selection errors), and gates `tools/call` to block direct or forged requests with a structured `CallToolResult(is_error=True)`[cite: 17, 18].
  3. **Row-Level & Business-Rule Guards (Inside Tools)**: Reads caller identity directly from `get_access_token()` (never trusting model-supplied user IDs), enforces ticket ownership, caps support course extensions at 30 days, and requires matching `course_id` + `course_title` pairs on financial or destructive operations[cite: 16, 20, 21, 22].
* **Deterministic Execution Trace**: Every chat response returns a structured `tool_calls` trace (`name`, `arguments`, `is_error`, `result`) rendered directly below the assistant message for transparent debugging[cite: 25, 27].

---

## Project Structure

```text
├── common/
│   ├── config.py          # Environment variables, ports, JWT claims, model config[cite: 5, 10]
│   ├── permissions.py     # Role-to-scope mappings & per-user custom scope resolution[cite: 5, 7, 8]
│   ├── auth.py            # PyJWT encoding/decoding with strict claim verification[cite: 5, 8, 9]
│   └── prompts.py         # Dynamic per-user system prompt & server instructions[cite: 5, 27, 28]
├── database/
│   ├── schema.sql         # 7 relational tables with foreign keys & CHECK constraints[cite: 5, 10, 11]
│   ├── seed.sql         # Relative-date dataset (enrollments, tickets, FAQs)[cite: 5, 12]
│   ├── db.py              # SQLite connection manager & database reset utility[cite: 5, 13]
│   └── models.py          # Dataclass ORM layer (all SQL queries isolated here)[cite: 5, 12]
├── mcp_server/
│   └── server.py          # MCPServer (mcp >= 2.3), TOOL_SCOPES, middleware, 25 tools[cite: 5, 14, 18, 19]
├── backend/
│   └── app.py             # FastAPI endpoints, per-request MCP client, 8-round chat loop[cite: 5, 24, 25]
├── frontend/
│   └── index.html         # Single-file chat interface with live tool-call trace inspector[cite: 5, 27]
├── .env.example           # Required environment variables template[cite: 9]
└── pyproject.toml         # Dependencies managed via uv[cite: 19, 33]
```

---

## Roles & Permission Scopes

Permissions use `resource:action` scope strings embedded in the user's JWT at login (with optional per-user overrides via `users.scopes`)[cite: 6, 8]:

| Role | Default Scopes | Capabilities |
| :--- | :--- | :--- |
| **`learner`**[cite: 2] | `faq:read`, `courses:read`, `courses:purchase`, `me:read`, `tickets:read`, `tickets:create`, `tickets:reply`[cite: 7] | View own enrollments, payments, and certificates; purchase courses (simulated); raise and reply to own tickets[cite: 2, 7]. |
| **`support`**[cite: 2] | `faq:read`, `courses:read`, `tickets:read`, `tickets:read_all`, `tickets:manage`, `learners:read`, `access:extend`, `enrollments:remove`[cite: 7] | Inspect all tickets and internal notes, manage/assign the support queue, search learners, extend access (max 30 days), remove enrollments[cite: 2, 7]. |
| **`admin`**[cite: 2] | All `support` scopes + `access:extend_unlimited`, `certificates:issue`, `faq:write`, `stats:read`, `users:manage`[cite: 7] | Unlimited access extensions, issue 100%-progress certificates, publish FAQs, view support metrics, manage user roles and scopes[cite: 2, 7, 22]. |

---

## Getting Started

### 1. Prerequisites & Installation
Make sure Python 3.11+ and [`uv`](https://docs.astral.sh/uv/) are installed, then sync dependencies[cite: 4, 33]:

```bash
uv sync
```

### 2. Configure Environment Variables
Copy `.env.example` to `.env` and set your OpenAI API key along with a strong `JWT_SECRET` (minimum 32 characters; the application refuses to start otherwise)[cite: 9, 10]:

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
Generate `database/support.db` from the schema and relative-date seed data (re-run anytime to reset state)[cite: 3, 4, 12]:

```bash
uv run python -m database.db
```

### 4. Start the Servers
Open two terminals and start the MCP tool server first, followed by the FastAPI backend[cite: 4]:

**Terminal 1: MCP Server (`127.0.0.1:8001/mcp`)**[cite: 4]
```bash
uv run python -m mcp_server.server
```

**Terminal 2: FastAPI Backend & UI (`127.0.0.1:8000`)**[cite: 4]
```bash
uv run python -m backend.app
```

Open `http://127.0.0.1:8000` in your browser[cite: 4].

---

## Seeded Accounts

All seeded accounts use the password **`password123`**[cite: 12, 29]:

| Email | Role | Pre-Seeded Scenario |
| :--- | :--- | :--- |
| `rahul@learner.com`[cite: 29] | `learner`[cite: 29] | Java course at 100% progress (no certificate issued yet); failed payment for Agentic AI yesterday[cite: 12, 29]. |
| `priya@learner.com`[cite: 29] | `learner`[cite: 29] | Spring Boot access expired 3 days ago; Java course at 25% progress[cite: 12, 29]. |
| `aman@learner.com`[cite: 29] | `learner`[cite: 29] | DevOps course purchased 4 days ago[cite: 29]. |
| `neha@support.com` | `support`[cite: 29] | Support agent working the open ticket queue[cite: 29]. |
| `vikram@support.com` | `support`[cite: 29] | Secondary support agent for ticket assignment[cite: 29]. |
| `akshay@admin.com` | `admin`[cite: 29] | Full administrative access across all 25 tools[cite: 20, 29]. |

### Suggested Test Flows
* **Dynamic Tool Visibility**: Log in as `rahul@learner.com` vs. `akshay@admin.com` and compare the sidebar tool lists (`tools/list` exposes 10 tools to learners vs. 25 to admins)[cite: 26, 27, 29, 33].
* **Autonomous Ticket Triage**: As Rahul, send *"I want the certificate for the Java complete course"*. The assistant verifies 100% completion and automatically creates a categorized support ticket[cite: 29].
* **Permission Boundary Checks**: As Neha (`support`), try issuing a certificate or extending course access by 60 days to inspect middleware and tool-level `Access denied` responses in the execution trace[cite: 8, 21, 27, 29, 33].

---

## Roadmap
* **Dynamic RBAC & Schema Config**: Move `ROLE_SCOPES`, `TOOL_SCOPES`, ticket categories, and priorities out of static dicts and SQL `CHECK` constraints into database tables or external config[cite: 12, 31].
* **Hybrid RAG Knowledge Base**: Replace SQL-based `search_faq` with a vector + keyword retrieval pipeline over markdown documentation[cite: 33].
* **Production Hardening**: Add domain guardrails to `SYSTEM_PROMPT`, server-side conversation history persistence, idempotency keys on mutating tools, and PostgreSQL support[cite: 3, 4, 26, 28, 31, 32].