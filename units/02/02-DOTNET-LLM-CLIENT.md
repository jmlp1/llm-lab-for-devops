# Unit 2: .NET LLM Client + Prompt Library

**Week:** 2
**Duration:** 5 days
**Goal:** Build a repeatable .NET console app that runs prompts from `.md` files and generates Markdown reports.

---

## Learning Objectives

By the end of this unit, you will:
- [ ] Create a C# console application to interact with Ollama
- [ ] Understand how a .NET app is structured and how the pieces connect
- [ ] Load prompt templates from files and inject variables into them
- [ ] Call the Ollama API from C# and get a response
- [ ] Save LLM output as formatted Markdown reports
- [ ] Run 3 working CLI commands end-to-end

---

## Prerequisites

- .NET 8.0+ SDK installed
- Visual Studio Code (you are already using it)
- Ollama running from Unit 1
- Git

---

## Daily Tasks

### Day 1: Project Setup & Structure
**Objectives:**
- Create the .NET console project
- Install dependencies
- Set up the folder structure and configuration file

**Tasks:**
1. Create new project — run from your `AI_Learning` folder:
   ```powershell
   dotnet new console -n ops-llm-client
   cd ops-llm-client
   ```

2. Add NuGet packages:

   > **First-time setup:** If you get "no versions available" errors, your machine has no NuGet source configured. Add the official one first:
   > ```powershell
   > dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org
   > ```
   > Then retry the commands below.

   ```powershell
   dotnet add package System.Net.Http.Json
   dotnet add package System.Text.Json
   dotnet add package Microsoft.Extensions.Configuration.Json
   ```
   > `System.Net.Http.Json` — sends HTTP requests and handles JSON automatically.
   > `System.Text.Json` — parses JSON responses from Ollama.
   > `Microsoft.Extensions.Configuration.Json` — loads settings from `appsettings.json`.

3. Create the `ops-llm-client` folder structure — **folders only**, files get created as you work through each day:
   ```powershell
   mkdir src, prompts, samples, out
   ```
   Expected layout:
   ```
   ops-llm-client/
   ├── src/           (C# source files — created in Days 2–4)
   ├── prompts/       (copy from ../ai-lab/prompts/ — see step 5)
   ├── samples/       (copy from ../ai-lab/samples/ — see step 5)
   └── out/           (generated reports — starts empty)
   ```

4. Create `appsettings.json` in the project root (`ops-llm-client/appsettings.json`):
   ```json
   {
     "Ollama": {
       "BaseUrl": "http://localhost:11434",
       "Model": "llama3.1:8b"
     },
     "Paths": {
       "PromptsDir": "./prompts",
       "SamplesDir": "./samples",
       "OutputDir": "./out"
     }
   }
   ```
   > Paths are relative to where you run `dotnet run` from. Think of this like `values.yaml` in Helm — defaults live here, CLI flags override them at runtime.

5. Tell .NET to copy `appsettings.json` to the build output — open `ops-llm-client.csproj` and add inside `<Project>`:
   ```xml
   <ItemGroup>
     <Content Include="appsettings.json">
       <CopyToOutputDirectory>Always</CopyToOutputDirectory>
     </Content>
   </ItemGroup>
   ```
   > Without this, .NET looks for `appsettings.json` in the `bin/Debug/` folder and can't find it.

6. Copy prompt templates and sample log into the project:
   ```powershell
   Copy-Item ..\ai-lab\prompts\* .\prompts\
   Copy-Item ..\ai-lab\samples\* .\samples\
   ```

**Deliverable:** Project compiles without errors — run `dotnet build` to verify.

---

### Day 2–3: Implement LLM Client & Prompt Engine
**Objectives:**
- Understand how the app is structured — 3 helper classes, each with one responsibility
- Implement the classes that handle API calls, prompt loading, and report saving

#### How the app is structured

Before writing code, understand what each piece does:

| Class | File | Responsibility |
|-------|------|----------------|
| `OllamaClient` | `src/OllamaClient.cs` | Sends a prompt to Ollama over HTTP, returns the response text |
| `PromptEngine` | `src/PromptEngine.cs` | Loads a prompt `.md` file, replaces `{{PLACEHOLDERS}}` with real values |
| `ReportGenerator` | `src/ReportGenerator.cs` | Takes LLM output and saves it as a Markdown file with a metadata header |

`Program.cs` is the orchestrator — it reads CLI flags, then calls these three classes in order. You build the three helpers first, then wire them up in Day 4.

**Tasks:**

1. Create `src/OllamaClient.cs`:
   ```csharp
   using System.Net.Http.Json;
   using System.Text.Json;

   public class OllamaClient
   {
       private readonly HttpClient _httpClient;
       private readonly string _model;

       public OllamaClient(string baseUrl, string model)
       {
           // Timeout set to 10 minutes — LLMs can be slow on CPU-only machines (no dedicated GPU)
           // If you have an NVIDIA or AMD GPU, Ollama will use it automatically and responses will be faster
           _httpClient = new HttpClient { BaseAddress = new Uri(baseUrl), Timeout = TimeSpan.FromMinutes(10) };
           _model = model;
       }

       public async Task<string> GenerateAsync(string prompt)
       {
           var request = new { prompt = prompt, model = _model, stream = false };
           var response = await _httpClient.PostAsJsonAsync("/api/generate", request);
           response.EnsureSuccessStatusCode();
           var json = await response.Content.ReadFromJsonAsync<JsonDocument>();
           return json!.RootElement.GetProperty("response").GetString() ?? "";
       }
   }
   ```

2. Create `src/PromptEngine.cs`:
   ```csharp
   using System.Collections.Generic;
   using System.IO;

   public class PromptEngine
   {
       public string LoadPrompt(string filePath)
       {
           return File.ReadAllText(filePath);
       }

       public string InjectVariables(string prompt, Dictionary<string, string> vars)
       {
           foreach (var (key, value) in vars)
           {
               prompt = prompt.Replace($"{{{{{key}}}}}", value);
           }
           return prompt;
       }
   }
   ```
   > `InjectVariables` finds `{{KEY}}` in the prompt text and replaces it with the value. This is how the prompt templates from Unit 1 get their real data injected at runtime.

3. Create `src/ReportGenerator.cs`:
   > **Exercise:** implement this yourself — no code provided.
   - Write a public method called `SaveReport` that accepts: the LLM output string, the output file path, the model name, and the input filename
   - Build a Markdown string with a metadata header (timestamp, model, input file) followed by the LLM output
   - Use `File.WriteAllText` to save it — the write equivalent of `File.ReadAllText` used in `PromptEngine`
   - Make sure the output directory exists before writing — use `Directory.CreateDirectory`

   **Hints:**
   - `DateTime.Now.ToString("yyyy-MM-dd HH:mm:ss")` gives you a formatted timestamp
   - Look at how `Program.cs` calls `generator.SaveReport(result, output, model, input)` — that tells you the method signature
   - The `using System.IO;` directive is required

**Deliverable:** Run `dotnet build` — zero errors before moving to Day 4.

---

### Day 4: Wire Up Program.cs
**Objectives:**
- Understand what `Program.cs` does and how it connects all three helper classes
- Replace the default "Hello, World!" with working CLI commands
- Run all 3 commands and verify output files are created

> All `dotnet run` commands must be run from the `ops-llm-client/` project root.

#### What is Program.cs?

`Program.cs` is the **entry point** — the first thing that runs when you type `dotnet run`. Right now it just prints "Hello, World!" because that is the default .NET template.

Think of it like a **Bash script that orchestrates other tools** — it does not do the work itself, it reads the inputs, calls the right tool, and saves the output. The three classes you built in Days 2–3 are those tools.

Flow for each command:
```
User runs: dotnet run -- summarize-log --input ./samples/sample-errors.log --output ./out/summary.md
                │
                ▼
        Program.cs reads the command name and flags
                │
                ▼
        PromptEngine loads the right prompt template
        and injects the log content into {{INPUT_LOG}}
                │
                ▼
        OllamaClient sends the filled prompt to Ollama
        and waits for the response (may take 1–3 minutes)
                │
                ▼
        ReportGenerator saves the response to ./out/summary.md
        with a metadata header (timestamp, model, input file)
```

#### Program.cs — full working code

Replace the entire content of `Program.cs` with this:

```csharp
using Microsoft.Extensions.Configuration;

// Load config from appsettings.json
var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var baseUrl = config["Ollama:BaseUrl"] ?? "http://localhost:11434";
var model   = config["Ollama:Model"]   ?? "llama3.1:8b";

// Initialise the three helper classes built in Days 2-3
var client    = new OllamaClient(baseUrl, model);
var engine    = new PromptEngine();
var generator = new ReportGenerator();

// -------------------------------------------------------
// Parse arguments — args[0] = command, rest = --flag value pairs
// -------------------------------------------------------
if (args.Length == 0 || args[0] == "--help")
{
    Console.WriteLine("ops-llm-client — LLM tools for DevOps");
    Console.WriteLine();
    Console.WriteLine("Commands:");
    Console.WriteLine("  summarize-log       --input <file> --output <file>");
    Console.WriteLine("  draft-runbook-steps --incident \"description\" --output <file>");
    Console.WriteLine("  generate-checklist  --topic \"topic\" --output <file>");
    return;
}

// Helper: extract a named flag value from the argument list
string GetArg(string[] a, string flag)
{
    for (int i = 0; i < a.Length - 1; i++)
        if (a[i] == flag) return a[i + 1];
    return "";
}

var command = args[0];
var rest    = args[1..];

switch (command)
{
    // ---------------------------------------------------
    case "summarize-log":
    {
        var input  = GetArg(rest, "--input");
        var output = GetArg(rest, "--output");

        if (!File.Exists(input)) { Console.WriteLine($"Error: file not found — {input}"); return; }
        if (string.IsNullOrEmpty(output)) { Console.WriteLine("Error: --output is required"); return; }

        var prompt  = engine.LoadPrompt("prompts/01-log-summary.md");
        var content = File.ReadAllText(input);
        var filled  = engine.InjectVariables(prompt, new Dictionary<string, string> { ["INPUT_LOG"] = content });

        Console.WriteLine("Sending to Ollama...");
        var result = await client.GenerateAsync(filled);
        generator.SaveReport(result, output, model, input);
        Console.WriteLine($"Report saved to {output}");
        break;
    }

    // ---------------------------------------------------
    case "draft-runbook-steps":
    {
        var incident = GetArg(rest, "--incident");
        var output   = GetArg(rest, "--output");

        if (string.IsNullOrEmpty(incident)) { Console.WriteLine("Error: --incident is required"); return; }
        if (string.IsNullOrEmpty(output))   { Console.WriteLine("Error: --output is required"); return; }

        var prompt = engine.LoadPrompt("prompts/02-incident-timeline.md");
        var filled = engine.InjectVariables(prompt, new Dictionary<string, string>
        {
            ["INPUT_DESCRIPTION"] = incident,
            ["INPUT_LOG"]         = ""
        });

        Console.WriteLine("Sending to Ollama...");
        var result = await client.GenerateAsync(filled);
        generator.SaveReport(result, output, model, incident);
        Console.WriteLine($"Runbook saved to {output}");
        break;
    }

    // ---------------------------------------------------
    case "generate-checklist":
    {
        var topic  = GetArg(rest, "--topic");
        var output = GetArg(rest, "--output");

        if (string.IsNullOrEmpty(topic))  { Console.WriteLine("Error: --topic is required"); return; }
        if (string.IsNullOrEmpty(output)) { Console.WriteLine("Error: --output is required"); return; }

        var prompt = engine.LoadPrompt("prompts/03-troubleshooting-checklist.md");
        var filled = engine.InjectVariables(prompt, new Dictionary<string, string>
        {
            ["INPUT_ISSUE"]  = topic,
            ["ENVIRONMENT"]  = "Production",
            ["OS"]           = "Windows Server",
            ["SERVICE_TYPE"] = "Web API"
        });

        Console.WriteLine("Sending to Ollama...");
        var result = await client.GenerateAsync(filled);
        generator.SaveReport(result, output, model, topic);
        Console.WriteLine($"Checklist saved to {output}");
        break;
    }

    // ---------------------------------------------------
    default:
        Console.WriteLine($"Unknown command: {command}. Run without arguments to see help.");
        break;
}
```

#### Run and verify

Run all 3 commands and confirm output files are created in `out/`:

```powershell
dotnet run -- summarize-log --input ./samples/sample-errors.log --output ./out/summary.md
```
Expected: `Report saved to ./out/summary.md`

```powershell
dotnet run -- draft-runbook-steps --incident "Database connection failure" --output ./out/runbook.md
```
Expected: `Runbook saved to ./out/runbook.md`

```powershell
dotnet run -- generate-checklist --topic "Network troubleshooting" --output ./out/checklist.md
```
Expected: `Checklist saved to ./out/checklist.md`

Then check the files were created:
```powershell
ls ./out/
```

Open each file and verify it contains a metadata header and a structured LLM response.

> **Note:** Each command sends a prompt to your local Ollama instance. Response time depends on your hardware:
> - **Dedicated GPU (NVIDIA/AMD):** 10–30 seconds
> - **CPU only (including Intel integrated/Iris):** 3–10 minutes
>
> Ollama does not support Intel integrated graphics — if you have Intel Iris or similar, it will run on CPU. This is normal. If you hit a timeout error, increase the timeout in `OllamaClient.cs` to `TimeSpan.FromMinutes(10)` or higher.

**Deliverable:** 3 `.md` files in `out/`, each with a metadata header and LLM response body.

---

### Day 5: Testing & Documentation
**Objectives:**
- Test all commands end-to-end with different inputs
- Write a README for the tool

**Tasks:**
1. Run each command a second time with different inputs to confirm they are repeatable
2. Check `out/` — you should have at least 3 files
3. Create `README.md` in the project root:
   > **Exercise:** write this yourself — document what you built in your own words.
   - What the tool does
   - How to install and run it
   - One example per command with the expected output
   - A troubleshooting section for common errors

**Deliverable:** `README.md` written + 3 sample output files in `out/`

---

## Deliverables Checklist

- [ ] `dotnet build` succeeds with zero errors
- [ ] `src/OllamaClient.cs` — HTTP client with 5-minute timeout
- [ ] `src/PromptEngine.cs` — loads and injects prompt variables
- [ ] `src/ReportGenerator.cs` — saves Markdown report with metadata header
- [ ] `appsettings.json` — Ollama config + default paths
- [ ] `ops-llm-client.csproj` — `appsettings.json` set to copy to output
- [ ] `Program.cs` — 3 CLI commands wired up
- [ ] 3 output files in `out/` — one per command
- [ ] `README.md` — written in your own words

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| `appsettings.json` not found | Check `ops-llm-client.csproj` has `CopyToOutputDirectory` set for `appsettings.json` |
| Prompts not found | Run `Copy-Item ..\ai-lab\prompts\* .\prompts\` from the project root |
| Timeout error | Increase timeout in `OllamaClient.cs` — set `Timeout = TimeSpan.FromMinutes(10)`. On CPU-only machines (Intel Iris etc.) responses can take 5–10 minutes |
| No versions available (NuGet) | Run `dotnet nuget add source https://api.nuget.org/v3/index.json -n nuget.org` |
| Cannot connect to Ollama | Verify Ollama is running: `curl http://localhost:11434/api/tags` |
| Output file not created | Check the `out/` folder exists in the project root |

---

## Resources

- [Ollama API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [.NET Console Apps](https://learn.microsoft.com/en-us/dotnet/core/tutorials/console-teleprompter)

---

## Next Step

Once completed, proceed to **[Unit 3: Skills API](../03/03-SKILLS-API.md)**
