# Laptop Setup: Windows 11

---

## Required Tools

### Ollama (Unit 1)
1. Download from [ollama.com](https://ollama.com)
2. Install and verify:
   ```powershell
   ollama serve
   # In a separate terminal:
   curl http://localhost:11434/api/tags
   ```
3. Pull the model:
   ```powershell
   ollama pull llama3.1:8b
   ```
   *(~5GB download — allow time on first run)*

---

### .NET SDK (Unit 2+)
1. Download .NET 8.0 SDK from [dotnet.microsoft.com](https://dotnet.microsoft.com/download/dotnet/8.0)
2. Verify:
   ```powershell
   dotnet --version
   # Expected: 8.x.x
   ```
3. Add NuGet.org as package source — required for restoring packages and cross-platform publishing:
   ```powershell
   dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
   ```
   Verify:
   ```powershell
   dotnet nuget list source
   # Expected: nuget.org [Enabled] https://api.nuget.org/v3/index.json
   ```

---

### Python (Unit 4)
1. Download Python 3.10+ from [python.org](https://www.python.org/downloads/)
2. During install: check **"Add Python to PATH"**
3. Verify:
   ```powershell
   python --version
   pip --version
   ```
4. Install Unit 4 dependencies when you reach that unit:
   ```powershell
   pip install sentence-transformers qdrant-client requests
   ```

---

### Docker Desktop (Optional — needed for local testing before lab server deploy)
- Download from [docker.com](https://www.docker.com/products/docker-desktop/)
- Requires WSL2 — Docker Desktop will guide you through setup

---

## Folder Structure

All working files live under your learning folder:
```
AI_Learning/
├── units/          <- learning materials (read-only reference)
├── resources/      <- guides and references
├── templates/      <- starter files
└── ai-lab/         <- your hands-on working directory
    ├── prompts/    <- prompt template files
    ├── docs/       <- markdown runbooks (Unit 4)
    ├── samples/    <- test input data
    └── out/        <- generated outputs
```
