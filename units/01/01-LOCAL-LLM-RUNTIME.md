# Unit 1: Local LLM Runtime + Baseline Prompting

**Week:** 1  
**Duration:** 5 days  
**Goal:** Run a local LLM and establish prompt patterns for DevOps outputs (checklists, summaries, JSON).

---

## Learning Objectives

By the end of this unit, you will:
- [x] Install and configure Ollama on Windows 11
- [x] Pull and run a local LLM model (`llama3.1:8b` or similar)
- [x] Create reusable prompt templates for DevOps tasks
- [ ] Generate baseline outputs: log summaries, incident timelines, troubleshooting checklists
- [ ] Document your setup and processes

---

## Prerequisites

- Windows 11 Pro with 16GB+ RAM
- GPU recommended (NVIDIA or AMD) but CPU-only works
- Folder structure: `./ai-lab/{prompts, docs, samples, out}` (in learning folder)

---

## Daily Tasks

### Day 1: Install & Run Ollama
**Status:** ✅ COMPLETE (Ollama/Llama already installed)

**Verification Steps:**
1. Open PowerShell and run:
   ```powershell
   ollama serve
   ```
2. In another terminal, test the endpoint:
   ```powershell
   curl http://localhost:11434/api/tags
   ```
   Expected response should list your installed models.

3. If Ollama doesn't start, reinstall or update to latest version.

**Status Check:** If both commands work, proceed to **Day 2**.

**Deliverable:** Working Ollama endpoint confirmed

---

### Day 2–3: Baseline Prompting & Style Guide
**Objectives:**
- Understand LLM prompting fundamentals
- Create reusable prompt patterns
- Establish response style guide

**Tasks:**
1. Review `./ai-lab/prompts/00-style.md` — this file is already in the repo and defines the response style used across all prompts:
   - Headings + bullets, **no paragraphs**
   - Ask clarifying questions when input is missing
   - Structured formats: JSON, tables, checklists

   Feel free to adjust it to suit your preferences — it is guidance, not a fixed rule.

2. Review the 3 prompt templates already provided in `./ai-lab/prompts/`:
   - `01-log-summary.md` — summarizes error logs
   - `02-incident-timeline.md` — builds an incident timeline from logs
   - `03-troubleshooting-checklist.md` — generates a diagnostic checklist

   Read through each one, understand what it does, and modify or improve them as you see fit. They are starting points, not final answers.

3. **Smoke Test** each prompt using the steps below.

#### How to Run the Smoke Test

The prompt templates use a `{{INPUT_LOG}}` placeholder. These steps show how to load a template, inject the log, and send it to Ollama.

**Step 1: Load the template**
```powershell
$template = Get-Content "./ai-lab/prompts/01-log-summary.md" -Raw
```
> Reads the entire file into a variable as one block of text. `-Raw` means keep it as-is, don't split into lines.

**Step 2: Load the log**
```powershell
$log = Get-Content "./ai-lab/samples/sample-errors.log" -Raw
```
> Same as Step 1 — reads the log file into a variable so you can inject it into the prompt.

**Step 3: Replace the placeholder**
```powershell
$prompt = $template -replace "{{INPUT_LOG}}", $log
```
> Finds `{{INPUT_LOG}}` in the template text and replaces it with the actual log content. The LLM receives one complete prompt with the log already embedded in it.

**Step 4: Send to Ollama**
```powershell
$body = @{
    prompt = $prompt
    model  = "llama3.1:8b"
    stream = $false
} | ConvertTo-Json
```
> Builds the request payload — what model to use, what the prompt is, and `stream = $false` means wait for the full answer before returning (don't print it word by word). `ConvertTo-Json` converts it to JSON format because that's what the API expects.

```powershell
$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
$response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" -Method Post -Body $bytes -ContentType "application/json"
```
> `$bytes` converts the JSON to UTF-8 encoding — without this PowerShell sends it in a format Ollama rejects.
> `Invoke-RestMethod` sends the request to Ollama over HTTP and waits for the response. This is the same as calling any web API.

**Step 5: Save output**
```powershell
$response.response | Out-File "./ai-lab/out/01-output.txt" -Encoding UTF8
Write-Host "Saved to ./ai-lab/out/01-output.txt"
```
> `$response.response` contains just the answer text from Ollama. `Out-File` saves it to a file. `-Encoding UTF8` ensures the file is readable and not garbled.

Template 01 done. Open `./ai-lab/out/01-output.txt` to verify the output looks correct, then move to Template 02 below.

---

#### Template 02 — Incident Timeline

**Step 1: Load template and log**
```powershell
$template = Get-Content "./ai-lab/prompts/02-incident-timeline.md" -Raw
$log      = Get-Content "./ai-lab/samples/sample-errors.log" -Raw
```

**Step 2: Inject both placeholders**
```powershell
$prompt = $template -replace "{{INPUT_DESCRIPTION}}", "Database connection failure causing service outage"
$prompt = $prompt -replace "{{INPUT_LOG}}", $log
```

**Step 3: Send to Ollama**
```powershell
$body = @{
    prompt = $prompt
    model  = "llama3.1:8b"
    stream = $false
} | ConvertTo-Json

$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
$response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" -Method Post -Body $bytes -ContentType "application/json"
```

**Step 4: Save output**
```powershell
$response.response | Out-File "./ai-lab/out/02-output.txt" -Encoding UTF8
Write-Host "Saved to ./ai-lab/out/02-output.txt"
```

---

#### Template 03 — Troubleshooting Checklist

**Step 1: Load template**
```powershell
$template = Get-Content "./ai-lab/prompts/03-troubleshooting-checklist.md" -Raw
```

**Step 2: Inject placeholders**
```powershell
$prompt = $template -replace "{{INPUT_ISSUE}}", "Database connection timeouts causing HTTP 500 errors"
$prompt = $prompt -replace "{{ENVIRONMENT}}", "Production"
$prompt = $prompt -replace "{{OS}}", "Windows Server"
$prompt = $prompt -replace "{{SERVICE_TYPE}}", "Web API with SQL database backend"
```

**Step 3: Send to Ollama**
```powershell
$body = @{
    prompt = $prompt
    model  = "llama3.1:8b"
    stream = $false
} | ConvertTo-Json

$bytes = [System.Text.Encoding]::UTF8.GetBytes($body)
$response = Invoke-RestMethod -Uri "http://localhost:11434/api/generate" -Method Post -Body $bytes -ContentType "application/json"
```

**Step 4: Save output**
```powershell
$response.response | Out-File "./ai-lab/out/03-output.txt" -Encoding UTF8
Write-Host "Saved to ./ai-lab/out/03-output.txt"
```

**Deliverable:** 3 prompt templates + 3 generated sample outputs

---

### Day 4–5: Documentation & Testing
**Objectives:**
- Document how to use the local LLM
- Create reusable workflow guide
- Verify all outputs

**Tasks:**
1. Create `./ai-lab/ai-lab.md` with:
   - How to start Ollama
   - How to run a prompt (curl example)
   - How to add new prompts
   - How to stop/restart
   - Performance tips

2. Create sample outputs:
   - Log summary output
   - Incident timeline output
   - Troubleshooting checklist output
   - Save all to `./ai-lab/out/`

3. **Test Checklist:**
   - [ ] Ollama starts without errors
   - [ ] Model loads in < 2 minutes
   - [ ] Prompts generate in < 30 seconds
   - [ ] Outputs are readable and structured
   - [ ] Documentation is clear

**Deliverable:** `ai-lab.md` setup guide + test results

---

## Deliverables Checklist

- [x] Ollama installed and verified
- [x] Llama model pulled
- [x] `./ai-lab/ai-lab.md` — setup and usage guide
- [x] `./ai-lab/prompts/00-style.md` — style guide
- [x] `./ai-lab/prompts/01-log-summary.md` — prompt template
- [x] `./ai-lab/prompts/02-incident-timeline.md` — prompt template
- [x] `./ai-lab/prompts/03-troubleshooting-checklist.md` — prompt template
- [x] 3 sample outputs in `./ai-lab/out/` (one per prompt)
- [x] Test results documentation

---

## Resources

- [Ollama Documentation](https://github.com/ollama/ollama)
- [llama3.1 Model Card](https://huggingface.co/meta-llama/Llama-3.1-8B)
- [Prompting Best Practices](../resources/RESOURCES.md#prompting)

---

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Model download very slow | Use a faster internet connection; pre-pull model on another machine |
| Out of memory errors | Reduce model size (`llama3.1:7b` or `mistral`) or disable other apps |
| Endpoint not responding | Ensure Ollama is still running; try `ollama serve` again |
| Poor output quality | Review style guide; adjust prompt clarity; try another model |

---

## Next Step

Once completed, proceed to **[Unit 2: .NET LLM Client](02-DOTNET-LLM-CLIENT.md)**
