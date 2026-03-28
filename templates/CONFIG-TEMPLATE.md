# Configuration Templates

Reference configs for each unit. Copy and adapt as needed.

---

## Unit 2: .NET LLM Client — `appsettings.json`

```json
{
  "Ollama": {
    "BaseUrl": "http://localhost:11434",
    "Model": "llama3.1:8b",
    "TimeoutSeconds": 120
  },
  "Paths": {
    "PromptsDir": "./prompts",
    "SamplesDir": "./samples",
    "OutputDir": "./out"
  }
}
```

---

## Unit 3: Skills API — `allowlist.json`

```json
{
  "allowlist": {
    "services": [
      "docker",
      "networking",
      "storage"
    ],
    "logSources": [
      "./ai-lab/samples/sample-errors.log"
    ],
    "maxTailLines": 200,
    "maxResponseSizeBytes": 102400
  },
  "audit": {
    "logPath": "./logs/audit.log",
    "enabled": true
  }
}
```

---

## Unit 3: Skills API — `Dockerfile`

```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /app
COPY . .
RUN dotnet publish -c Release -o out

FROM mcr.microsoft.com/dotnet/aspnet:8.0
WORKDIR /app
COPY --from=build /app/out .
EXPOSE 5000
ENV ASPNETCORE_URLS=http://+:5000
CMD ["dotnet", "skills-api.dll"]
```

---

## Unit 4: Qdrant Collection — vector size by embedding model

| Embedding Model | How to use | Vector size |
|----------------|-----------|-------------|
| `all-MiniLM-L6-v2` | `pip install sentence-transformers` | **384** |
| `nomic-embed-text` | `ollama pull nomic-embed-text` | **768** |

Use the matching size in your Qdrant collection config:
```json
{
  "name": "runbooks",
  "vectors": {
    "size": 384,
    "distance": "Cosine"
  }
}
```

---

## Unit 4: Python Dependencies — `requirements.txt`

```
sentence-transformers==2.7.0
qdrant-client==1.9.1
requests==2.32.0
```

Install with:
```powershell
pip install -r requirements.txt
```
