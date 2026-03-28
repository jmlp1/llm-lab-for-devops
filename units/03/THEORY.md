# Unit 3 Theory: Skills API + Audit Logging

---

## What is a "Skill"?

**Skill** = a specific, controlled action the LLM can trigger on your behalf

In Unit 3 you build an API that exposes skills like:
- "What is the current disk usage?" → calls `/api/system/disk`
- "Is the docker service running?" → calls `/api/services/status?name=docker`
- "Show the last 50 lines of this log" → calls `/api/logs/tail`

The LLM does not run these directly — it sends a request to your API, and your API decides whether it is allowed.

```
LLM client
   | (HTTP GET)
   v
Skills API (.NET Minimal API)
   | checks allowlist
   | runs operation if allowed
   | writes to audit log
   v
Returns result to LLM client
```

This pattern — giving an LLM access to controlled tools — is called **tool calling** or **function calling**. It is a core concept in AI-102.

---

## What is a REST API?

**REST API** = a web service that uses standard HTTP methods to perform operations

For the Skills API, you only use **GET** because everything is read-only:

```
GET /api/system/info                          -> CPU, RAM, OS info
GET /api/system/disk                          -> disk usage per drive
GET /api/services/status?name=docker          -> service running status
GET /api/logs/tail?source=app.log&lines=50    -> last N lines of a log
```

**.NET Minimal API** is the simplest way to build this in C# — a few `app.MapGet(...)` calls in `Program.cs`.

---

## What is Allowlisting?

**Allowlist** = an explicit list of what is permitted (everything else is denied by default)

```json
{
  "services": ["docker", "networking"],
  "logSources": ["./ai-lab/samples/sample-errors.log"]
}
```

If the LLM requests a service not on the list → `403 Forbidden`.
If it requests a log file not on the list → `403 Forbidden`.

**Why not just trust the LLM?**
LLMs can be manipulated via **prompt injection** — malicious content in a log file could trick the model into requesting something it should not access. The allowlist stops that at the API layer, regardless of what the LLM asks for.

---

## What is Audit Logging?

**Audit log** = a persistent record of everything that was called and what happened

```
[2026-03-27T10:30:15Z] caller=llm-client endpoint=/api/system/info success=true
[2026-03-27T10:30:25Z] caller=llm-client endpoint=/api/logs/tail source=./app.log success=false error=ALLOWLIST_VIOLATION
```

**Why it matters:**
- You can see exactly what the LLM called vs. what you expected
- You can detect unexpected or suspicious patterns
- Required for any serious AI system
- Maps directly to AI-102 content safety and governance requirements

---

## What is Docker?

**Docker** = a way to package an application and all its dependencies so it runs consistently anywhere

```
Your .NET app + runtime + config
        |
        v
   Docker image  (the package)
        |
        v
Docker container  (a running instance of the package)
```

**Why use Docker for the Skills API?**
- Deploy to any lab server or VM without "it works on my machine" problems
- Easy to stop, restart, update, or roll back
- Same approach used for Qdrant in Unit 4

**Key commands you'll use:**
```bash
docker build -t skills-api:latest .    # build the image from Dockerfile
docker run -d -p 5000:5000 skills-api  # run a container in the background
docker ps                               # list running containers
docker logs skills-api                  # view container output
docker stop skills-api                  # stop the container
```
