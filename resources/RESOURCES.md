# Resources & References

---

## Unit 1: Local LLM Runtime

### Ollama
- [Ollama GitHub](https://github.com/ollama/ollama) — installation, model list, API reference
- [Ollama API Reference](https://github.com/ollama/ollama/blob/main/docs/api.md) — `/api/generate`, `/api/tags`, streaming

### Models
- [Llama 3.1 (Meta)](https://huggingface.co/meta-llama/Llama-3.1-8B) — model card
- Pull command: `ollama pull llama3.1:8b`

### Prompting
- [Prompt Engineering Guide](https://www.promptingguide.ai/) — general techniques
- Principle: structured prompts (role, task, format, constraints) produce consistent results

---

## Unit 2: .NET LLM Client

### .NET / C#
- [.NET 8.0 SDK](https://dotnet.microsoft.com/download/dotnet/8.0)
- [Console App Tutorial](https://learn.microsoft.com/en-us/dotnet/core/tutorials/console-teleprompter)
- [HttpClient in .NET](https://learn.microsoft.com/en-us/dotnet/fundamentals/networking/http/httpclient)
- [System.Text.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview)

### CLI Argument Parsing
- [System.CommandLine (Microsoft)](https://learn.microsoft.com/en-us/dotnet/standard/commandline/) — recommended for the 3-command interface

---

## Unit 3: Skills API

### ASP.NET Core Minimal APIs
- [Minimal APIs Overview](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Minimal API Tutorial](https://learn.microsoft.com/en-us/aspnet/core/tutorials/min-web-api)

### Docker
- [Docker Desktop for Windows](https://docs.docker.com/desktop/install/windows-install/)
- [Dockerfile Reference](https://docs.docker.com/reference/dockerfile/)
- [.NET Docker Images](https://hub.docker.com/_/microsoft-dotnet)

---

## Unit 4: RAG Integration

### Qdrant
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Qdrant Quick Start](https://qdrant.tech/documentation/quickstart/)
- [Qdrant Python Client](https://github.com/qdrant/qdrant-client)

### Embeddings
- [Sentence Transformers](https://www.sbert.net/) — `all-MiniLM-L6-v2`, produces 384-dim vectors
- [Ollama Embeddings API](https://github.com/ollama/ollama/blob/main/docs/api.md#generate-embeddings) — alternative using `nomic-embed-text`, keeps everything on Ollama
- Pull the Ollama embedding model: `ollama pull nomic-embed-text`

### RAG Concepts
- [RAG Overview (Microsoft)](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/retrieval-augmented-generation)

---

## AI-102 Certification

- [AI-102 Exam Page](https://learn.microsoft.com/en-us/credentials/certifications/exams/ai-102/)
- [AI-102 Study Guide](https://learn.microsoft.com/en-us/credentials/certifications/resources/study-guides/ai-102)
- [Microsoft Learn AI-102 Path](https://learn.microsoft.com/en-us/training/paths/prepare-for-ai-engineering/)
- [AI-900 (Optional warm-up)](https://learn.microsoft.com/en-us/credentials/certifications/exams/ai-900/) — shorter foundation cert, good confidence builder
