# README.md — Acumatica MCP Server

## Overview
This repository provides an MCP server that exposes Acumatica ERP functionality as AI-callable tools.  
The server connects to your Acumatica instance via its REST API, using credentials and endpoint details provided in environment variables.  
MCP clients (like AI assistants, Codex CLI, or Docker MCP Gateway) can then discover and call tools such as create_case, assign_case, or get_case.

---

## Features
- Connects to Acumatica ERP using session login or OAuth2.
- Exposes Cases (CR306015) as a first-class set of MCP tools.
- Contract-aware: can consume Swagger/OpenAPI (contracts/acumatica-openapi.json).
- Safe by design: Pydantic validation, RBAC policies, dry-run simulation.
- Docker-first: easy to build and run in a container.
- Extensible: add new screen adapters and schemas.

---

## Quickstart

### 1. Clone the repo
git clone https://github.com/YOURUSER/acumatica-mcp.git
cd acumatica-mcp

### 2. Prepare environment
Copy .env.example and fill in your Acumatica details:
cp .env.example .env

Required variables:
ACU_URL=https://your-site/entity
ACU_ENDPOINT=Default
ACU_VER=24.200.001
ACU_USER=admin
ACU_PWD=secret
ACU_COMPANY=MyCompany
ACU_BRANCH=HQ
MCP_SERVER_PORT=7331

Optional:
ACU_OPENAPI_URL=https://your-site/entity/Default/24.200.001/swagger.json
ACU_USE_OAUTH=true
ACU_OAUTH_TOKEN=yourtoken

### 3. Build and run with Docker
docker build -t acumatica-mcp .
docker run --rm --env-file .env -p 7331:7331 acumatica-mcp

### 4. Connect with MCP Toolkit
If using Docker Desktop + MCP Toolkit:
- Add this server under Settings → MCP Servers.
- The server will be auto-discovered if you label the container correctly (--label mcp.server=acumatica).

Or with Codex CLI:
codex connect http://localhost:7331

---

## Available Tools (MVP)

- create_case(subject, customer?, caseClass?, owner?, description?, hold?)
- assign_case(caseNbr, owner)
- hold_case(caseNbr, hold: bool)
- add_case_note(caseNbr, text)
- get_case(caseNbr)
- search_cases(status?, owner?, subject_contains?)

Each returns normalized JSON with at least CaseNbr, Subject, Status, Owner.

---

## Development

### Install locally
python -m venv .venv
source .venv/bin/activate
pip install -r ops/requirements.txt

### Run server
uvicorn server:app --host 0.0.0.0 --port 7331 --reload

### Run tests
pytest -v

---

## Extending to new screens
1. Create a schema in schemas/{screen}.py.
2. Create an adapter in acumatica/screens/{screen}.py.
3. Register tools in server.py.
4. Add tests in tests/test_{screen}.py.
5. Update README with new tools.

---

## Security
- Never commit real credentials.
- Use .env for secrets (ignored by .gitignore).
- Tokens and passwords are never logged.
- For disclosure instructions, see SECURITY.md.

---

## References
- Acumatica Integration Development Guide (REST API, OData, Swagger).
- MCP Toolkit (Docker + AI integration).
