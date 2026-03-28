# Unit 1 Theory: Local LLM Runtime + Baseline Prompting

---

## What is an LLM?

**LLM = Large Language Model**

A machine learning model trained on massive amounts of text data that can:
- Understand questions/instructions (prompts)
- Generate human-like responses
- Follow patterns and complete tasks

**Examples:**
- GPT-4 (OpenAI) — requires internet, paid
- Llama 3.1 (Meta) — open source, runs locally (what you're using)
- Mistral — smaller, faster alternative

---

## Why "Local" LLM?

**Local LLM (what you have):**
- ✓ Runs on your own computer
- ✓ No internet needed after download
- ✓ Private data (logs stay private)
- ✓ Free to use
- ✗ Slower than cloud services
- ✗ Limited by your hardware

**Cloud LLM:**
- ✓ Very fast
- ✓ More powerful
- ✗ Requires internet
- ✗ Data sent to external servers
- ✗ Costs money per request

---

## What is Ollama?

**Ollama** is a tool that:
1. Downloads LLM models (like Llama 3.1)
2. Runs them locally on your computer
3. Provides an **API** (web interface) to talk to the model

**How it works:**
```
Your Code
   ↓
HTTP Request to http://localhost:11434/api/generate
   ↓
Ollama (running locally)
   ↓
Llama 3.1 Model (processes your request)
   ↓
HTTP Response (answer)
   ↓
Your Code Gets Answer
```

---

## What is a Prompt?

**A prompt** = Instructions you send to the LLM

**Bad prompt (too vague):**
```
Analyze this log
```

**Good prompt (clear, structured):**
```
Analyze this error log and provide:
1. Count of errors
2. Types of errors
3. Timeline (when they occurred)
4. Recommended next steps
```

---

## What is a Prompt Template?

**Prompt template** = A reusable prompt with placeholders for data

**Example:**
```
Analyze the following log and summarize errors:

LOG CONTENT:
{{INPUT_LOG}}

Provide:
- Error count
- Error types
- Root cause
```

When you use it:
1. You replace `{{INPUT_LOG}}` with actual log content
2. Send to LLM
3. Get back structured response

**Why use templates?**
- ✓ Consistent results every time
- ✓ Reusable across different logs
- ✓ Easy to modify for similar tasks
- ✓ Professional, structured outputs

---

## What is an API?

**API = Application Programming Interface**

It's a way for programs to talk to each other.

**REST API:** Uses HTTP requests (like a web browser)

**Example:**
```
Request:
POST http://localhost:11434/api/generate
{
  "prompt": "What is DevOps?",
  "model": "llama3.1:8b"
}

Response:
{
  "response": "DevOps is an approach that combines...",
  "done": true
}
```

**In this unit:**
- You'll use PowerShell to send HTTP requests to Ollama API
- No web browser needed—just raw API calls

---

## Key Concepts for Unit 1

### 1. Model = Brain
The trained AI that understands and generates text
- `llama3.1:8b` = Llama 3.1 with 8 billion parameters
- Larger = smarter but slower
- Smaller = faster but less capable

### 2. Prompt = Questions/Instructions
How you communicate with the model
- More clarity → better results
- Structured prompts → consistent results

### 3. Response = Answer
What the model generates based on your prompt
- Quality depends on prompt quality
- Often needs refinement/iteration

### 4. Endpoint = Where to Send Requests
`http://localhost:11434/api/generate` is where Ollama listens

### 5. JSON = Data Format
```json
{
  "prompt": "question here",
  "model": "llama3.1:8b",
  "stream": false
}
```

---

## Simple Flow: What Happens When You Ask a Question?

```
Step 1: You write a prompt
   "What is DevOps?"

Step 2: PowerShell sends it to Ollama API
   POST http://localhost:11434/api/generate

Step 3: Ollama receives it
   API parses the request

Step 4: Llama 3.1 processes it
   Model thinks: "DevOps... related to... development... operations..."

Step 5: Model generates response
   "DevOps is a practice that combines..."

Step 6: Response sent back to you
   PowerShell receives the answer

Step 7: You read/use the response
   Extract text, save to file, or display
```

---

## Why This Matters for DevOps

**DevOps tasks:** Log analysis, troubleshooting, documentation, checklists

**LLM helps with:**
- ✓ Analyzing logs quickly (patterns, errors)
- ✓ Creating incident timelines
- ✓ Generating troubleshooting steps
- ✓ Writing runbook procedures
- ✓ Automating repetitive analysis

**In this unit, you'll:**
1. Learn how to send questions to the LLM
2. Create template prompts for common DevOps tasks
3. Test them and see results
4. Understand how to make good prompts

---

## What You'll Learn

By end of Unit 1:
- ✓ How Ollama and LLMs work
- ✓ How to use APIs with PowerShell
- ✓ How to write effective prompts
- ✓ How to create reusable templates
- ✓ How to test and validate outputs

**Next Unit (Unit 2):** You'll build a C# tool to automate this process.

---

## Remember

You don't need to be an AI expert—just understand:
1. **Input** (prompt) → **Process** (LLM) → **Output** (response)
2. Better prompts = Better responses
3. Templates = Consistency and automation

Ready? Let's test it with a simple question!
