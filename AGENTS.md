# AGENTS.md — Acumatica MCP Server (Python)

## 0) Purpose
This document instructs an AI developer tool to generate a working **MCP server** that exposes **Acumatica ERP** screens as **tools**, starting with **Cases (CR306015)**.  
The design must be:
- Contract-aware: leverage Acumatica’s REST API, OData, and Swagger/OpenAPI contracts.  
- Extensible: new screens can be added with adapters and schemas.  
- Production-ready: safe, typed, idempotent, with RBAC, observability, and Dockerized deployment.  
- Developer-friendly: include OpenAPI files, unit tests, and policies to guide codegen.

---
```text
## 1) Deliverables (repo structure)

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
  contracts/
    acumatica-openapi.json      # Swagger / OpenAPI example
  settings.py                   # Pydantic BaseSettings for env vars
  policies/
    rbac.yaml                   # Role-based tool restrictions
    tenants/
      default.fields.json       # Field mapping per endpoint/tenant
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

- ACU_URL: base Acumatica URL (e.g., https://erp/entity)
- ACU_ENDPOINT: endpoint name (default: Default)
- ACU_VER: endpoint version (e.g., 24.200.001)
- ACU_USER, ACU_PWD
- ACU_COMPANY, ACU_BRANCH
- ACU_TENANT: optional explicit tenant
- ACU_USE_OAUTH (bool), ACU_OAUTH_TOKEN (optional)
- ACU_OPENAPI_URL: optional direct URL to swagger.json
- MCP_SERVER_PORT: default 7331
- LOG_LEVEL: default INFO
- ROLE_DEFAULT: default Support
- TENANT: default tenant id

Provide .env.example with all placeholders.

---

## 4) Acumatica client contract

acumatica/client.py must implement:

- Authentication  
  - Session login:  
    POST {ACU_URL}/auth/login with body:  
    { "name": "user", "password": "pwd", "tenant": "Company", "branch": "Branch", "locale": "EN-US" }
  - Logout: POST {ACU_URL}/auth/logout
  - OAuth2 flows supported: Authorization Code, Implicit, ROPC, Hybrid. Use Authorization: Bearer <token> if ACU_USE_OAUTH.

- Endpoint pattern  
  {ACU_URL}/{Endpoint}/{Version}/  
  Example: https://site/entity/Default/24.200.001/

- Contracts  
  - Instance-level Swagger: {ACU_URL}/swagger.json
  - Endpoint-level Swagger: {ACU_URL}/{Endpoint}/{Version}/swagger.json?company=<Tenant>

- HTTP utilities  
  - Methods: get, post, put, delete
  - Retry + exponential backoff with tenacity
  - Error taxonomy: AcuAuthError, AcuNotFound, AcuConflict, AcuRateLimited, AcuValidationError, AcuUnknown
  - Redact secrets in logs

---

## 5) Screen adapter pattern

Each adapter lives under acumatica/screens/{screen}.py and must:
- Import AcumaticaClient
- Expose methods: create, update, search, get, etc.
- Map friendly args → Acumatica REST JSON (fields as { "value": "..." })
- Support linked entities and custom fields using Acumatica’s custom block
- Normalize outputs (dict with key fields only)
- Raise structured errors if status >=400

Template _template.py should:
- Define an Adapter class with __init__(settings)
- Include commented examples of tool methods
- Use schemas for args/results

---

## 6) Schemas

All tool arguments and results must be defined in Pydantic under schemas/{screen}.py.

- Example for Cases: CreateCaseArgs, AssignCaseArgs, HoldCaseArgs, AddNoteArgs, GetCaseArgs, SearchCasesArgs.
- Each field must include descriptions.
- Ensure consistency with Acumatica JSON format:
  "CustomerID": { "value": "JOHNGOOD" }
- Support nested/linked entities.
- Custom fields use:
  "custom": {
    "Entity": {
      "UsrField": { "type": "CustomStringField", "value": "ABC123" }
    }
  }

---

## 6b) API Contracts (Swagger / OpenAPI)

- Location: contracts/acumatica-openapi.json (or .yaml).
- Optional remote: ACU_OPENAPI_URL.
- The AI tool should:
  1. Parse the OpenAPI doc.
  2. Surface available operations for entities (Cases, SalesOrders, etc.).
  3. Generate/update adapter stubs (acumatica/screens/*.py) and Pydantic models (schemas/*.py).
  4. Overlay tenant field mappings (policies/tenants/*.fields.json).
  5. Add discovery tools in server.py:
     - list_endpoints() → wraps GET {ACU_URL}/entity
     - get_contract() → returns loaded Swagger JSON

---

## 7) Tools for Cases (MVP)

Expose in server.py with @mcp.tool:

- create_case(subject, customer?, caseClass?, owner?, description?, hold?) → dict
- assign_case(caseNbr, owner) → dict
- hold_case(caseNbr, hold: bool) → dict
- add_case_note(caseNbr, text) → dict
- get_case(caseNbr) → dict
- search_cases(status?, owner?, subject_contains?) → [dict]

Return normalized dicts with at least CaseNbr, Subject, Status, Owner.

---

## 8) RBAC & tenants

- policies/rbac.yaml: restrict tool visibility per role (Support, Accounting).
- policies/tenants/{tenant}.fields.json: map friendly args → Acumatica field names.
- Server must dynamically load tenant configs.

---

## 9) Deployment

- ops/Dockerfile: slim Python base, install requirements, expose port.
- ops/requirements.txt: list all dependencies.
- README.md must show:
  docker build -t acumatica-mcp .
  docker run --rm -e ACU_URL=... -e ACU_USER=... acumatica-mcp
- ops/compose.yaml: optional integration with MCP Gateway.

---

## 10) Logging & observability

- ops/logging.yaml: configure structured logs (JSON).
- Include request id, role, and tool name in logs.
- Redact sensitive fields.
- Support dry-run mode via simulate: true.

---

## 11) Testing

- tests/test_cases.py must:
  - Mock Acumatica REST with responses.
  - Validate happy paths (create_case, get_case).
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

- Never log passwords, tokens, or tenant names.
- Provide SECURITY.md with disclosure instructions.
- Ensure idempotency for create_* tools (include optional externalRef).
- Implement retries/backoff for rate limits.
- Honor Acumatica API limits and paging.

