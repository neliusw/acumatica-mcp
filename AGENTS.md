# AGENTS.md — Acumatica MCP Server (Python)

## 0) Purpose
This document instructs an AI developer tool to generate a working **MCP server** that exposes **Acumatica ERP** screens as **tools**, starting with **Cases (CR306015)**. The server must be designed for expansion to additional screens (e.g., Sales Orders). The generated repo must be Docker-first, typed with Pydantic, and safe for production patterns (idempotency, RBAC, logging).

---

## 1) Deliverables (repo structure)

```text
acumatica-mcp/
  server.py                     # MCP entrypoint + tool registry
  acumatica/
    __init__.py
    client.py                   # Auth + HTTP wrapper for Acumatica
    screens/
      cases.py                  # Cases adapter (MVP screen)
      _template.py              # Boilerplate adapter template
  schemas/
    cases.py                    # Pydantic models for Cases tools
  settings.py                   # Pydantic BaseSettings for env vars
  policies/
    rbac.yaml                   # Role-based tool restrictions
    tenants/
      default.fields.json       # Field mapping for default endpoint
  tests/
    test_cases.py               # Unit tests with pytest + responses
  ops/
    Dockerfile
    requirements.txt
    compose.yaml                # Optional: local dev with MCP Gateway
    logging.yaml                # Structured logging config
    Makefile
  .env.example
  README.md
  AGENTS.md
  SECURITY.md
```

---

## 2) Runtime & dependencies

- Python 3.11+
- Libraries: mcp, pydantic>=2, requests, tenacity, uvicorn, pyyaml
- Testing: pytest, responses
- Dev/lint: ruff, black

---

## 3) Environment variables

- ACU_URL: base Acumatica URL (https://erp/entity)
- ACU_ENDPOINT: default endpoint name (Default)
- ACU_VER: endpoint version (e.g., 23.200.001)
- ACU_USER, ACU_PWD
- ACU_COMPANY, ACU_BRANCH
- ACU_USE_OAUTH (bool), ACU_OAUTH_TOKEN (optional)
- MCP_SERVER_PORT (default 7331)
- LOG_LEVEL (default INFO)
- ROLE_DEFAULT (default Support)
- TENANT (default default)

Provide `.env.example` with all placeholders.

---

## 4) Acumatica client contract

acumatica/client.py must implement:

- Session login: POST {ACU_URL}/auth/login with {name, password, company, branch}
- OAuth2 mode: use Authorization: Bearer {token} if ACU_USE_OAUTH
- Path builder: /entity/{Endpoint}/{Version}/{resource}
- Methods: get, post, put, delete
- Retry + backoff with tenacity
- Typed exceptions: AcuAuthError, AcuNotFound, AcuConflict, AcuRateLimited, AcuValidationError, AcuUnknown
- Redact secrets in logs

---

## 5) Screen adapter pattern

Each adapter lives under acumatica/screens/{screen}.py and must:
- Import AcumaticaClient
- Expose methods: create, update, search, get, etc.
- Map friendly args to Acumatica REST JSON
- Normalize outputs (dict with key fields only)
- Raise structured errors if status >=400

Template example (_template.py):
- Adapter class with init(settings)
- Example tool methods with comments
- Use schemas for arguments and outputs

---

## 6) Schemas

All tool arguments and results must be defined in Pydantic under schemas/{screen}.py. Example: CreateCaseArgs, AssignCaseArgs, HoldCaseArgs, AddNoteArgs, GetCaseArgs, SearchCasesArgs. Ensure descriptions are provided for every field.

---

## 7) Tools for Cases (MVP)

Expose in server.py with @mcp.tool decorators:

- create_case(subject, customer?, caseClass?, owner?, description?, hold?) → dict
- assign_case(caseNbr, owner) → dict
- hold_case(caseNbr, hold: bool) → dict
- add_case_note(caseNbr, text) → dict
- get_case(caseNbr) → dict
- search_cases(status?, owner?, subject_contains?) → [dict]

Return normalized dicts with at least CaseNbr, Subject, Status, Owner.

---

## 8) RBAC & tenants

- policies/rbac.yaml must allow restricting tools by role (Support, Accounting).
- policies/tenants/{tenant}.fields.json maps friendly args → Acumatica field names.
- server must load tenant config dynamically.

---

## 9) Deployment

- ops/Dockerfile: slim Python base, install requirements, expose port.
- ops/requirements.txt: list all libs.
- README.md must show:
  docker build -t acumatica-mcp .
  docker run --rm -e ACU_URL=... -e ACU_USER=... acumatica-mcp
- ops/compose.yaml optional for MCP Gateway integration.

---

## 10) Logging & observability

- ops/logging.yaml configures structured logs (JSON).
- Include request id and tool name in logs.
- Redact sensitive fields.
- Support dry-run mode with simulate: true flag in args.

---

## 11) Testing

- tests/test_cases.py must:
  - Mock Acumatica REST with responses.
  - Validate happy path (create_case, get_case).
  - Validate error handling (404, 401, 429).
- Run pytest in CI.

---

## 12) Expansion rules

To add a new screen:
1. Create schemas/{screen}.py
2. Create acumatica/screens/{screen}.py adapter
3. Register tools in server.py
4. Add tests/test_{screen}.py
5. Update README.md

---

## 13) Security

- Never log passwords or tokens.
- Provide SECURITY.md with disclosure instructions.
- Ensure idempotency for create_* tools (include optional externalRef).
- Add rate limit handling in client (retry with backoff).

