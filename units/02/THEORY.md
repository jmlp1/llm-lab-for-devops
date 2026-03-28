# Unit 2 Theory: .NET LLM Client + Prompt Library

---

## What is an HTTP Client?

**HTTP Client** = a piece of code that sends web requests and reads responses

You already did this in Unit 1 with PowerShell (`Invoke-RestMethod`). In Unit 2, you do the same thing from C# code — the concept is identical, just in a different language.

```
Your C# App
   |
   | HttpClient.PostAsJsonAsync(...)
   v
POST http://localhost:11434/api/generate
   |
   v
Ollama processes the request
   |
   v
JSON response returned
   |
   v
Your app extracts the answer text
```

**Why move from PowerShell to C#?**
- Scripts are fine for one-off use
- A compiled app is reusable, testable, and structured
- Units 3 and 4 build directly on top of this client

---

## What is JSON Serialization?

**Serialization** = converting a C# object into JSON text to send over HTTP
**Deserialization** = converting JSON text back into a C# object

```csharp
// You send this C# object:
var request = new { prompt = "Hello", model = "llama3.1:8b", stream = false };

// .NET serializes it to:
// {"prompt":"Hello","model":"llama3.1:8b","stream":false}

// Ollama returns:
// {"response":"Hi there!","done":true}

// You deserialize it back to access:
result.Response  // "Hi there!"
```

**Why it matters:** Ollama's API speaks JSON. You need to serialize your request and deserialize the response to use it from code.

---

## What is a CLI (Command-Line Interface)?

**CLI** = a program you control by typing commands and arguments

```powershell
dotnet run -- summarize-log --input ./samples/errors.log --output ./out/summary.md
```

Breaking this down:
- `dotnet run` — runs your C# app
- `--` — separates dotnet's own arguments from your app's arguments
- `summarize-log` — the command name (which operation to run)
- `--input` / `--output` — named arguments

**Why structure it this way?**
- Each command maps to one prompt template
- Easy to call from scripts or pipelines later
- Clear and self-documenting

---

## What is Configuration Management?

**Configuration** = settings that can change without modifying code

```json
{
  "Ollama": {
    "BaseUrl": "http://localhost:11434",
    "Model": "llama3.1:8b"
  }
}
```

**Why not just hardcode these values?**
- If you switch models, you change one config line — no code change
- If you deploy to a different machine, you change the URL — no code change
- Keeps sensitive values out of source code

In .NET, `appsettings.json` is the standard approach. You read it with `ConfigurationBuilder`.

---

## How Unit 2 Maps to Unit 1

Everything you did manually in Unit 1 becomes structured code in Unit 2:

| Unit 1 (PowerShell) | Unit 2 (C#) |
|---------------------|-------------|
| `Get-Content template.md` | `PromptEngine.LoadPrompt()` |
| `-replace "{{INPUT_LOG}}", $log` | `PromptEngine.InjectVariables()` |
| `Invoke-RestMethod` | `OllamaClient.GenerateAsync()` |
| `Write-Host $response.response` | `ReportGenerator.SaveReport()` |

Same logic — just reusable, testable, and extendable.
