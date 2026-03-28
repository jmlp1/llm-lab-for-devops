# Safety & Security Guidelines

These rules apply to all units. They are non-negotiable.

---

## Core Principles

### 1. Read-Only in Phase 1 (Units 1–3)
- All Skills API endpoints must be GET requests only
- No write, delete, create, or modify operations
- No file writes, process kills, or config changes via the API

### 2. Allowlist Everything
- Log sources: only explicitly listed paths are accessible
- Services: only named services can be queried
- Enforce at the API layer — do not trust client input

### 3. Audit Every Call
- Log: timestamp, caller, endpoint, parameters, result size, success/failure
- Store logs outside the application (e.g., separate volume)
- Never delete audit logs during development

### 4. No Production Credentials or Endpoints
- Use only your lab environment (local server or VM)
- Never connect to production databases, APIs, or infrastructure
- Use placeholder/dummy credentials in config examples

### 5. Network Isolation
- Lab server accessible only on LAN (no public internet exposure)
- Document firewall rules before deploying anything
- No port forwarding to the internet

---

## Threat Scenarios to Consider

| Scenario | Mitigation |
|----------|-----------|
| Prompt injection via log content | Sanitize input; treat all LLM input as untrusted |
| Path traversal in log tail endpoint | Strict allowlist; reject relative paths (`../`) |
| Runaway LLM calls (resource usage) | Timeout on Ollama calls; max tokens if supported |
| Sensitive data in outputs | Only use lab/sample data; never real production logs |
| Over-privileged API | Keep all endpoints read-only; no admin operations |

---

## Before Each Unit

- [ ] Confirm you are working with sample/lab data only
- [ ] Confirm Ollama is running locally (not a cloud service)
- [ ] Confirm lab server is on LAN only (Unit 3+)
- [ ] Confirm audit logging is active before making skill calls (Unit 3+)
