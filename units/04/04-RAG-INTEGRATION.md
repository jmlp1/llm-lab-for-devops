# Unit 4: RAG over Markdown Runbooks + End-to-End Demo

**Week:** 4  
**Duration:** 5 days  
**Goal:** Implement retrieval-augmented generation (RAG) using a vector database to query Markdown runbooks with citations, optional skill calls, and create an end-to-end demo.

---

## Learning Objectives

By the end of this unit, you will:
- [ ] Set up a vector database (Qdrant) on lab server or VM
- [ ] Implement document ingestion pipeline (chunk, embed, store)
- [ ] Build query pipeline with semantic retrieval
- [ ] Generate answers with citations and source references
- [ ] Integrate with Skills API from Unit 3
- [ ] Create and demonstrate end-to-end incident response workflow

---

## Prerequisites

- Unit 1–3 completed
- Docker on lab server or VM
- Python 3.10+ installed and on PATH (see [LAPTOP-SETUP.md](../../resources/LAPTOP-SETUP.md))
- Understanding of vector databases and embeddings (see [THEORY.md](THEORY.md))

---

## Daily Tasks

### Day 1: Set Up Vector Database (Qdrant)
**Objectives:**
- Deploy Qdrant on lab server or VM
- Understand vector database concepts
- Plan document structure

**Tasks:**
1. Deploy Qdrant on lab server or VM:
   ```powershell
   docker run -d \
     -p 6333:6333 \
     --name qdrant \
     --restart unless-stopped \
     qdrant/qdrant:latest
   ```

2. Verify connectivity:
   ```powershell
   curl http://localhost:6333/health
   ```

3. Create collection schema:
   ```json
   {
     "name": "runbooks",
     "vectors": {
       "size": 384,
       "distance": "Cosine"
     },
     "payload_schema": {
       "filename": { "type": "text" },
       "section": { "type": "text" },
       "source_chunk": { "type": "text" }
     }
   }
   ```

4. Design document structure:
   - **Source files:** `C:\ai-lab\docs\*.md` (5+ runbooks)
   - **Chunks:** Split by heading level (H2 sections)
   - **Metadata:** filename, section title, word count

5. Create runbooks in `./ai-lab/docs/` (sample topics):
   - `01-incident-response.md` — general incident workflow
   - `02-database-troubleshooting.md` — DB-specific issues
   - `03-network-diagnostics.md` — network problems
   - `04-log-analysis.md` — log interpretation
   - `05-recovery-procedures.md` — recovery steps

**Deliverable:** Qdrant running, collection created, 5 sample runbook templates

---

### Day 2: Build Document Ingestion Pipeline
**Objectives:**
- Read and parse Markdown documents
- Chunk by logical sections
- Create embeddings
- Store in Qdrant

**Tasks:**
1. Create Python script `ingest.py`:
   ```python
   import os
   import uuid
   from pathlib import Path
   from sentence_transformers import SentenceTransformer
   from qdrant_client import QdrantClient
   from qdrant_client.models import PointStruct, VectorParams, Distance

   COLLECTION_NAME = "runbooks"
   VECTOR_SIZE = 384  # all-MiniLM-L6-v2 output size

   class DocumentIngester:
       def __init__(self, qdrant_url="http://localhost:6333"):
           self.client = QdrantClient(url=qdrant_url)
           self.model = SentenceTransformer("all-MiniLM-L6-v2")
           self._ensure_collection()

       def _ensure_collection(self):
           """Create collection if it doesn't exist"""
           existing = [c.name for c in self.client.get_collections().collections]
           if COLLECTION_NAME not in existing:
               self.client.create_collection(
                   collection_name=COLLECTION_NAME,
                   vectors_config=VectorParams(size=VECTOR_SIZE, distance=Distance.COSINE)
               )

       def chunk_markdown(self, content, filename):
           """Split markdown by H2 headings"""
           chunks = []
           current_chunk = ""
           current_section = ""

           for line in content.split("\n"):
               if line.startswith("## "):
                   if current_chunk:
                       chunks.append({
                           "text": current_chunk,
                           "section": current_section,
                           "filename": filename
                       })
                   current_section = line.replace("## ", "")
                   current_chunk = ""
               else:
                   current_chunk += line + "\n"

           if current_chunk:
               chunks.append({"text": current_chunk, "section": current_section, "filename": filename})

           return chunks

       def ingest_documents(self, docs_path):
           """Load all .md files and store in Qdrant"""
           for md_file in Path(docs_path).glob("*.md"):
               with open(md_file, "r") as f:
                   content = f.read()

               chunks = self.chunk_markdown(content, md_file.name)
               points = []

               for chunk in chunks:
                   embedding = self.model.encode(chunk["text"]).tolist()
                   points.append(PointStruct(
                       id=str(uuid.uuid4()),
                       vector=embedding,
                       payload={
                           "text": chunk["text"],
                           "section": chunk["section"],
                           "filename": chunk["filename"]
                       }
                   ))

               if points:
                   self.client.upsert(collection_name=COLLECTION_NAME, points=points)
                   print(f"Ingested {md_file.name} — {len(points)} chunk(s)")

   if __name__ == "__main__":
       ingester = DocumentIngester()
       ingester.ingest_documents(".\\ai-lab\\docs")
   ```

2. Install Python dependencies:
   ```powershell
   pip install sentence-transformers qdrant-client
   ```

3. Run ingestion:
   ```powershell
   python ingest.py
   ```

4. Verify documents are stored:
   ```python
   from qdrant_client import QdrantClient
   client = QdrantClient(url="http://localhost:6333")
   collection_info = client.get_collection("runbooks")
   print(f"Vectors stored: {collection_info.points_count}")
   ```

**Deliverable:** Ingestion script working, documents stored in Qdrant, verification successful

---

### Day 3: Build Query Pipeline with Citations
**Objectives:**
- Implement semantic search
- Retrieve relevant chunks
- Format answers with citations
- Expose query engine as HTTP service for C# integration

**Tasks:**
1. Create Python script `query.py`:
   ```python
   from sentence_transformers import SentenceTransformer
   from qdrant_client import QdrantClient
   import json
   
   class RAGQueryEngine:
       def __init__(self, qdrant_url="http://localhost:6333"):
           self.client = QdrantClient(url=qdrant_url)
           self.model = SentenceTransformer("all-MiniLM-L6-v2")
       
       def search(self, query, top_k=3):
           """Retrieve top-k relevant chunks"""
           query_embedding = self.model.encode(query)
           results = self.client.search(
               collection_name="runbooks",
               query_vector=query_embedding,
               limit=top_k
           )
           
           context = []
           citations = []
           for result in results:
               context.append(result.payload["text"])
               citations.append({
                   "file": result.payload["filename"],
                   "section": result.payload["section"]
               })
           
           return context, citations
       
       def generate_answer(self, query, context, citations):
           """Format answer with LLM + citations"""
           # Call LLM with context
           prompt = f"""
           Question: {query}
           
           Context:
           {chr(10).join(context)}
           
           Answer based on context and cite sources.
           """
           
           # Call Ollama API directly
           import requests
           r = requests.post(
               "http://localhost:11434/api/generate",
               json={"prompt": prompt, "model": "llama3.1:8b", "stream": False}
           )
           llm_response = r.json()["response"]
           
           # Format answer with citations
           answer = {
               "answer": llm_response,
               "citations": citations,
               "confidence": "medium"
           }
           
           return answer
   
   if __name__ == "__main__":
       engine = RAGQueryEngine()
       results = engine.search("How do I troubleshoot database connection errors?")
       print(json.dumps(results, indent=2))
   ```

2. Expose the query engine as a lightweight HTTP endpoint so the C# client can call it:
   ```python
   # Add to query.py
   from http.server import HTTPServer, BaseHTTPRequestHandler
   import json

   class QueryHandler(BaseHTTPRequestHandler):
       def do_POST(self):
           length = int(self.headers["Content-Length"])
           body = json.loads(self.rfile.read(length))
           engine = RAGQueryEngine()
           context, citations = engine.search(body["query"])
           answer = engine.generate_answer(body["query"], context, citations)
           self.send_response(200)
           self.send_header("Content-Type", "application/json")
           self.end_headers()
           self.wfile.write(json.dumps(answer).encode())

   if __name__ == "__main__":
       HTTPServer(("localhost", 8080), QueryHandler).serve_forever()
   ```
   > Python cannot be imported from C#. Exposing it as an HTTP service on port 8080 lets the Unit 2 C# client call it via `HttpClient` like any other API.

3. In Unit 2 C# client, add a method to call the RAG service:
   ```csharp
   public async Task<string> QueryRagAsync(string query)
   {
       var request = new { query = query };
       var response = await _httpClient.PostAsJsonAsync("http://localhost:8080", request);
       response.EnsureSuccessStatusCode();
       var json = await response.Content.ReadFromJsonAsync<JsonDocument>();
       return json!.RootElement.GetProperty("answer").GetString() ?? "";
   }
   ```

3. Test with sample queries:
   - "How do I respond to a security incident?"
   - "What are the steps to diagnose network problems?"
   - "Where do I find error logs?"

**Deliverable:** Query engine working, citations included in responses, integration tested

---

### Day 4: Integrate Skills API & Build End-to-End Workflow
**Objectives:**
- Connect to Skills API from Unit 3
- Build complete incident response workflow
- Create decision logic (when to call skills)

**Tasks:**
1. Create `incident-response.py`:
   - Input: incident log or description
   - Output: analysis + recommendations + optional skill calls

   ```python
   class IncidentResponseWorkflow:
       def __init__(self, llm_client, rag_engine, skills_api):
           self.llm = llm_client
           self.rag = rag_engine
           self.skills = skills_api
       
       def analyze_incident(self, incident_log):
           # Step 1: Summarize incident
           summary = self.llm.summarize(incident_log)
           
           # Step 2: Search for related runbooks
           context, citations = self.rag.search(summary)
           
           # Step 3: Generate initial analysis
           analysis = self.llm.generate_answer(summary, context, citations)
           
           # Step 4: Suggest skill calls if needed
           skill_calls = self.suggest_skills(analysis)
           
           # Step 5: Execute read-only skills
           skill_results = {}
           for skill in skill_calls:
               result = self.skills.call(skill)
               skill_results[skill] = result
           
           # Step 6: Incorporate skill results
           final_answer = self.llm.refine_answer(analysis, skill_results)
           
           return {
               "summary": summary,
               "analysis": analysis,
               "citations": citations,
               "skill_calls": skill_calls,
               "skill_results": skill_results,
               "final_answer": final_answer
           }
       
       def suggest_skills(self, analysis):
           # Heuristic: extract keywords that suggest skill calls
           # E.g., "disk full" -> suggest /api/system/disk
           skills = []
           if "disk" in analysis.lower():
               skills.append("system/disk")
           if "service" in analysis.lower():
               skills.append("services/status")
           return skills
   ```

2. Create end-to-end test:
   - Provide sample incident log
   - Run complete workflow
   - Verify output includes citations and skill results

3. Document workflow as `INCIDENT-RESPONSE-WORKFLOW.md`

**Deliverable:** Workflow implemented, tested with sample incident, documentation complete

---

### Day 5: Demo, Architecture Diagram & Final Testing
**Objectives:**
- Create end-to-end demonstration
- Document architecture
- Finalize all deliverables

**Tasks:**
1. Create demonstration script (`demo.md`):
   - Scenario: Simulated database connection error incident
   - Steps:
     1. Provide incident log
     2. Run workflow
     3. Show analysis output
     4. Show citations
     5. Show skill calls and results
     6. Show final recommendations

2. Create architecture diagram:
   ```
   Incident Log
      ↓
   [Unit 2: LLM Client]
      ↓
   [Unit 4: RAG Query Engine]
      ↓ (retrieves context from)
   [Qdrant Vector DB] ← [Markdown Runbooks]
      ↓
   [Ollama Local LLM]
      ↓
   [Unit 3: Skills API]
      ↓ (calls if needed)
   [Read-only Skills] → Audit Log
      ↓
   [Final Answer + Citations + Skill Results]
   ```

3. Create `ARCHITECTURE.md`:
   - Component overview
   - Data flow
   - Security considerations
   - Deployment topology

4. Final testing checklist:
   - [ ] All 4 units working together
   - [ ] RAG returns accurate citations
   - [ ] Skills API called safely
   - [ ] Audit logging captures all operations
   - [ ] Output is formatted correctly
   - [ ] Demo runs without errors
   - [ ] Documentation is complete and clear

5. Create summary report: `COMPLETION-REPORT.md`

**Deliverable:** Working end-to-end demo, architecture diagram, completed documentation

---

## Deliverables Checklist

- [ ] Qdrant running on lab server or VM with runbooks collection
- [ ] `ingest.py` script ingesting documents successfully
- [ ] `query.py` script with semantic search
- [ ] 5+ markdown runbook templates in `C:\ai-lab\docs\`
- [ ] RAG query engine returning citations
- [ ] Skills API integration working
- [ ] Incident response workflow implemented
- [ ] End-to-end demo script (`demo.md`)
- [ ] Architecture diagram (`ARCHITECTURE.md`)
- [ ] Sample incident analysis output with citations
- [ ] Audit log showing all operations
- [ ] Completion report with test results

---

## Sample Query Output

```json
{
  "question": "How do I troubleshoot high CPU usage?",
  "answer": "High CPU usage can be caused by... Check running processes and review logs. Consider restarting services.",
  "citations": [
    {
      "file": "02-database-troubleshooting.md",
      "section": "Performance Issues"
    },
    {
      "file": "03-network-diagnostics.md",
      "section": "System Monitoring"
    }
  ],
  "suggested_skills": [
    "system/info",
    "system/disk"
  ],
  "confidence": "high"
}
```

---

## Resources

- [RAG Patterns](https://learn.microsoft.com/en-us/azure/ai-services/openai/concepts/retrieval-augmented-generation)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Sentence Transformers](https://www.sbert.net/)
- [Azure OpenAI Embeddings](https://learn.microsoft.com/en-us/azure/ai-services/openai/reference#embeddings)

---

## Completion & Next Steps

✅ **Unit 4 Complete:** You now have a fully functional RAG-enabled incident response system.

**What's Next:**
- (Optional) **Weeks 5–8:** Production hardening
  - Docker Compose for all services
  - GitHub Actions CI/CD pipeline
  - Observability and monitoring
  - Approval-gated write operations
  - Threat modeling and security audit

- **Certification Path:** Review your work against AI-102 exam domains and supplement as needed


