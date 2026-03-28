# LLM Lab for DevOps

A hands-on learning path for DevOps and SysAdmin engineers who want to understand how LLMs work — without cloud accounts, Azure credits, or vendor lock-in.

Everything runs locally on your own machine.

---

## Who This Is For

You are a DevOps or SysAdmin engineer. You work with logs, incidents, runbooks, and infrastructure every day. You keep hearing about AI and LLMs but every guide starts with a cloud account, a credit card, or assumes you're a data scientist.

This is the guide you were looking for.

By the end of the 4 units you will have built a local tool that:
- Reads an error log and gives you a structured summary, root cause, and next steps
- Builds an incident timeline from raw log output
- Searches your own runbook library and surfaces the relevant procedure automatically

You will understand *how* it works, not just *that* it works — which is what you need before stepping into Azure AI, AWS Bedrock, or any cloud AI service.

---

## What You Need

- A Windows 11 or Linux laptop with 16GB+ RAM
- No cloud account required for Units 1–2
- A Linux server or VM on your LAN for Units 3–4 (can be a VM on your own machine)
- Basic comfort with the command line

---

## What This Covers

| Unit | Topic | Duration | Focus |
|------|-------|----------|-------|
| [Unit 1](units/01/01-LOCAL-LLM-RUNTIME.md) | Local LLM Runtime | Week 1 | Run a local LLM, learn prompt patterns, generate structured outputs |
| [Unit 2](units/02/02-DOTNET-LLM-CLIENT.md) | .NET LLM Client | Week 2 | Build a C# tool to run prompts from Markdown templates |
| [Unit 3](units/03/03-SKILLS-API.md) | Skills API | Week 3 | Build a read-only .NET API with audit logging and allowlists |
| [Unit 4](units/04/04-RAG-INTEGRATION.md) | RAG Integration | Week 4 | Add a vector database, embeddings, and citation-backed answers |

Each unit has a theory page, a step-by-step guide, and a checklist to verify you're ready to move on.

---

## How to Use This

1. Start with [Laptop Setup](resources/LAPTOP-SETUP.md)
2. Work through each unit in order — each one builds on the last
3. Do the tasks yourself. The guides explain *what* and *why*, not just *how*.
4. Check off the checklist at the end of each unit before moving on

---

## Lab Environment

- **Laptop:** Windows 11 or Linux — runs Ollama and .NET tooling
- **Server (Units 3–4):** Any Linux server or VM on your LAN — hosts Docker containers
- **Network:** LAN-only. No production credentials, no public endpoints.

See [Server Setup](resources/SERVER-SETUP.md) for details.

---

## Safety Rules

- All Skills API endpoints are read-only in Units 1–3
- Allowlist all log sources and paths
- Audit log every skill call
- No production credentials or endpoints — lab data only

See [SAFETY.md](resources/SAFETY.md) for details.

---

## Where This Leads

After completing these 4 units you will have a solid mental model of:
- How LLMs process prompts and generate responses
- What RAG is and why it matters for grounding answers in your own data
- How to expose LLM capabilities safely through an API

That foundation maps directly to the Microsoft Azure AI Engineer Associate (AI-102) curriculum — Azure OpenAI, Azure AI Search, prompt engineering, and orchestration patterns.

---

## Resources

- [Ollama](https://ollama.com)
- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Qdrant Vector DB](https://qdrant.tech)
- [Additional Resources](resources/RESOURCES.md)
