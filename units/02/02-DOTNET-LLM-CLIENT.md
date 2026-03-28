# Unit 2: .NET LLM Client + Prompt Library

**Week:** 2  
**Duration:** 5 days  
**Goal:** Build a repeatable .NET console app that runs prompts from `.md` files and generates Markdown reports.

---

## Learning Objectives

By the end of this unit, you will:
- [ ] Create a C# console application to interact with Ollama
- [ ] Implement a prompt library system using Markdown templates
- [ ] Inject input data (logs, configurations) into prompts
- [ ] Generate formatted Markdown reports
- [ ] Document commands and workflows

---

## Prerequisites

- .NET 8.0+ SDK installed
- Visual Studio Code or Visual Studio 2022
- Ollama running from Unit 1
- Git (optional but recommended)

---

## Daily Tasks

### Day 1: Project Setup & Structure
**Objectives:**
- Initialize .NET console project
- Set up configuration and dependencies
- Plan project structure

**Tasks:**
1. Create new project:
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
   dotnet add package System.CommandLine
   ```
   > `Microsoft.Extensions.Configuration.Json` is required for `ConfigurationBuilder` to load `appsettings.json`.
   > `System.CommandLine` provides the `--input`/`--output` flag parsing used in Day 4.

3. Create the `ops-llm-client` folder structure — **folders only**, files get created as you work through each day:
   ```powershell
   mkdir src, prompts, samples, out
   ```
   Expected layout:
   ```
   ops-llm-client/
   ├── src/           (C# source files — created in Days 2–4)
   ├── prompts/       (copy from ./ai-lab/prompts when ready)
   ├── samples/       (test data — copy from ./ai-lab/samples)
   └── out/           (generated reports — starts empty)
   ```

4. Create `appsettings.json` in the project root (`ops-llm-client/appsettings.json`):
   ```json
   {
     "Ollama": {
       "BaseUrl": "http://localhost:11434",
       "Model": "llama3.1:8b"
     }
   }
   ```

**Deliverable:** Project structure set up, compiles without errors

---

### Day 2–3: Implement LLM Client & Prompt Engine
**Objectives:**
- Call Ollama API from C#
- Load prompts from Markdown files
- Inject variables into prompts

**Tasks:**
1. Create `src/OllamaClient.cs`:
   - Constructor: takes base URL and model name
   - Method: `async Task<string> GenerateAsync(string prompt)`
   - Handle retries and timeouts
   ```csharp
   using System.Net.Http.Json;
   using System.Text.Json;

   public class OllamaClient
   {
       private readonly HttpClient _httpClient;
       private readonly string _model;

       public OllamaClient(string baseUrl, string model)
       {
           _httpClient = new HttpClient { BaseAddress = new Uri(baseUrl) };
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
   - Load prompt from `prompts/*.md`
   - Replace placeholders: `{{INPUT}}`, `{{DATE}}`, `{{CONTEXT}}`
   - Example:
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

3. Create `src/ReportGenerator.cs`:
   > **Exercise:** implement this yourself — no code provided.
   - Format output as Markdown
   - Add metadata (timestamp, model, input file)
   - Save to `out/*.md`

   **Hints:**
   - You need a public method that accepts the LLM output, the model name, and the input filename as parameters
   - Use `DateTime.Now` for the timestamp
   - Build the Markdown string first, then write it to a file — look at how `PromptEngine` uses `File.ReadAllText`, you need the write equivalent
   - Generate the output filename dynamically so each run produces a new file (e.g. include the timestamp in the name)
   - The `Code Example` section at the bottom of this guide shows how `ReportGenerator` is called — use that as a clue for what your method signature should look like

**Deliverable:** OllamaClient works, prompts load, variables inject correctly

---

### Day 4: Implement CLI Commands
**Objectives:**
- Create command interface
- Implement 3 main commands
- Add help/documentation

> All `dotnet` commands in this section must be run from the `ops-llm-client/` project root.

**Tasks:**
1. Implement 3 commands in `Program.cs`:
   > **Exercise:** implement this yourself — the expected commands and their flags are shown below, but the implementation is yours.

   **Command 1: summarize-log**
   ```powershell
   dotnet run -- summarize-log --input ./ai-lab/samples/errors.log --output ./ai-lab/out/summary.md
   ```

   **Command 2: draft-runbook-steps**
   ```powershell
   dotnet run -- draft-runbook-steps --incident "Database connection failure" --output ./ai-lab/out/runbook.md
   ```

   **Command 3: generate-checklist**
   ```powershell
   dotnet run -- generate-checklist --topic "Network troubleshooting" --output ./ai-lab/out/checklist.md
   ```

2. Add `--help` support — running `dotnet run -- --help` should print available commands and their flags. The `System.CommandLine` package handles this automatically once commands are wired up correctly.

3. Add basic error handling — at minimum:
   - If `--input` file does not exist, print a clear error message and exit
   - If Ollama is not reachable, catch the exception and tell the user rather than crashing

**Deliverable:** Run each command with a real input and confirm a `.md` file is created in `out/` with a metadata header and the LLM response body

---

### Day 5: Testing & Documentation
**Objectives:**
- Test all commands end-to-end
- Create comprehensive README
- Verify output quality

**Tasks:**
1. Create `README.md`:
   > **Exercise:** write this yourself — document what you built in your own words.
   - Overview of the tool
   - Installation steps
   - 3 example commands with sample output
   - Troubleshooting section

2. Test all 3 commands with different inputs
3. Verify output Markdown is valid
4. Create test results log
5. Document any manual adjustments needed

**Deliverable:** README + test results + all 3 sample outputs

---

## Deliverables Checklist

- [ ] Working C# console application
- [ ] OllamaClient implementation
- [ ] PromptEngine with variable injection
- [ ] ReportGenerator with Markdown formatting
- [ ] 3 CLI commands functional
- [ ] Comprehensive README with examples
- [ ] 3 sample generated reports
- [ ] appsettings.json configuration file
- [ ] Test results documentation

---

## Code Example: Simple Main Program

```csharp
var config = new ConfigurationBuilder()
    .AddJsonFile("appsettings.json")
    .Build();

var baseUrl = config["Ollama:BaseUrl"];
var model = config["Ollama:Model"];

var client = new OllamaClient(baseUrl, model);
var engine = new PromptEngine();
var generator = new ReportGenerator();

// Example: summarize log
var prompt = engine.LoadPrompt("prompts/01-log-summary.md");
var logContent = File.ReadAllText("samples/errors.log");
var filled = engine.InjectVariables(prompt, new Dictionary<string, string> { ["INPUT"] = logContent });
var result = await client.GenerateAsync(filled);
generator.SaveReport(result, "out/summary.md");
```

---

## Resources

- [Ollama API Documentation](https://github.com/ollama/ollama/blob/main/docs/api.md)
- [.NET Console Apps](https://learn.microsoft.com/en-us/dotnet/core/tutorials/console-teleprompter)
- [Command-line parsing libraries](../resources/RESOURCES.md#csharp)

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Cannot connect to Ollama | Verify Ollama is running on `http://localhost:11434` |
| Prompts not found | Check file paths; use absolute paths if relative fails |
| JSON parsing errors | Use `System.Text.Json.JsonDocument` for flexible parsing |
| Slow generation | Reduce model size or check system resources |

---

## Next Step

Once completed, proceed to **[Unit 3: Skills API](03-SKILLS-API.md)**
