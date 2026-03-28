# Unit 4 Checklist: RAG Integration

Use this at the end of the unit to verify you're ready to move on.

- [ ] Qdrant running on lab server or VM (`/health` endpoint returns OK)
- [ ] `runbooks` collection created with correct vector size
- [ ] 5+ Markdown runbook files created in `./ai-lab/docs/`
- [ ] `ingest.py` ingests all runbooks without errors
- [ ] Vector count in Qdrant matches expected number of chunks
- [ ] `query.py` returns top-3 results with citations for a test query
- [ ] `incident-response.py` workflow runs end-to-end with a sample log
- [ ] Skills API called at least once from the workflow
- [ ] Audit log shows skill calls triggered by the workflow
- [ ] `demo.md` — demo script written and tested
- [ ] `ARCHITECTURE.md` — component overview and data flow documented
