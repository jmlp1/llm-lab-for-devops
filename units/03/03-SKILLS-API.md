# Unit 3: Skills API (Read-only) + Audit Logging

**Week:** 3  
**Duration:** 5 days  
**Goal:** Create a controlled .NET Minimal API that allows the LLM to safely call operational "skills" with read-only access, audit logging, and allowlisting.

---

## Learning Objectives

By the end of this unit, you will:
- [ ] Build a .NET Minimal API with read-only endpoints
- [ ] Implement allowlisting for resources and operations
- [ ] Create audit logging for all API calls
- [ ] Deploy on a Linux server or VM using Docker
- [ ] Integrate skills calls from the Unit 2 LLM client

---

## Prerequisites

- .NET 8.0+ SDK
- Docker installed on lab server or VM
- Network connectivity between laptop and server
- Understanding of REST APIs
- Unit 1 & 2 completed

---

## Daily Tasks

### Day 1: Project Setup & API Design
**Objectives:**
- Create Minimal API project
- Design endpoint structure
- Plan allowlist and audit schema

**Tasks:**
1. Create new project — run from your `AI_Learning` folder:
   ```powershell
   dotnet new web -n skills-api
   cd skills-api
   ```

2. No additional packages needed — both `System.Diagnostics` and `Microsoft.AspNetCore.RateLimiting` are built into .NET 8's ASP.NET Core SDK. No `dotnet add package` required.

3. Create configuration schema:
   ```json
   {
     "allowlist": {
       "logSources": ["/var/log/app/*.log", "C:\\logs\\*.log"],
       "services": ["docker", "networking", "storage"],
       "maxTailLines": 1000,
       "maxResponseSize": 1048576
     },
     "audit": {
       "logPath": "/var/log/skills-api-audit.log",
       "enabled": true
     }
   }
   ```

4. Create the folder structure — **folders only**, files get created as you work through each day:
   ```powershell
   mkdir src, config
   ```
   Expected layout:
   ```
   skills-api/
   ├── src/
   │   ├── Program.cs
   │   ├── AllowlistValidator.cs
   │   ├── AuditLogger.cs
   │   ├── SystemSkills.cs
   │   ├── ServiceSkills.cs
   │   └── LogSkills.cs
   ├── config/
   │   └── allowlist.json
   └── Dockerfile
   ```

**Deliverable:** Project structure created, allowlist schema in `appsettings.json`

---

### Day 2–3: Implement Endpoints & Validators
**Objectives:**
- Implement all 4 endpoint categories
- Add allowlist validation
- Add audit logging

**Tasks:**

All files go in the `src/` folder you created on Day 1. Copy each file then read the comments to understand what it does.

1. Create `src/AllowlistValidator.cs` — reads the allowlist from `appsettings.json` and validates incoming requests:
   - `IsServiceAllowed(name)` — checks the service name against `allowlist:services`
   - `IsLogSourceAllowed(source)` — matches the file path against `allowlist:logSources` patterns (supports `*` wildcards)
   - `ValidateTailLines(lines)` — enforces the `allowlist:maxTailLines` limit

2. Create `src/AuditLogger.cs` — appends one line per request to `audit.log`:
   ```
   [2026-03-27T10:30:25Z] caller=llm-client endpoint=/api/logs/tail parameters=[source=/var/log/blocked.log] success=false error=ALLOWLIST_VIOLATION
   ```

3. Create `src/SystemSkills.cs` — maps two endpoints:
   - `GET /api/system/info` — returns OS, machine name, CPU count, memory, .NET version
   - `GET /api/system/disk` — returns name, type, total/free GB and used % for every ready drive

4. Create `src/ServiceSkills.cs` — maps one endpoint:
   - `GET /api/services/status?name=docker` — validates against allowlist, then runs `sc query` (Windows) or `systemctl is-active` (Linux) and returns the status

5. Create `src/LogSkills.cs` — maps one endpoint:
   - `GET /api/logs/tail?source=...&lines=50` — validates source against allowlist and enforces the max lines limit, then returns the last N lines of the file

6. Replace `Program.cs` with the wired-up version that:
   - Registers `AllowlistValidator` and `AuditLogger` as singletons
   - Configures rate limiting (100 req/min)
   - Adds middleware to default `X-Caller` to `"unknown"` when not provided
   - Calls `MapSystemEndpoints`, `MapServiceEndpoints`, `MapLogEndpoints`

   Verify it builds:
   ```powershell
   dotnet build
   ```

**Deliverable:** `dotnet build` succeeds with 0 errors, all 4 endpoints registered

---

### Day 4: Error Handling, Testing & Documentation
**Objectives:**
- Add comprehensive error handling
- Test all endpoints
- Document API contracts

**Tasks:**
1. Add error handling:
   - 403 Forbidden (allowlist violation)
   - 400 Bad Request (invalid parameters)
   - 500 Internal Server Error
   - Return structured error responses:
   ```json
   {
     "error": "Service not in allowlist",
     "code": "ALLOWLIST_VIOLATION",
     "status": 403,
     "timestamp": "2026-03-27T10:30:00Z"
   }
   ```

2. Add rate limiting middleware in `Program.cs`:
   ```csharp
   builder.Services.AddRateLimiter(options =>
   {
       options.AddFixedWindowLimiter("default", limiter =>
       {
           limiter.PermitLimit = 100;
           limiter.Window = TimeSpan.FromMinutes(1);
           limiter.QueueProcessingOrder = QueueProcessingOrder.OldestFirst;
           limiter.QueueLimit = 0;
       });
       options.RejectionStatusCode = 429;
   });

   // After app.Build():
   app.UseRateLimiter();

   // On each endpoint:
   app.MapGet("/api/system/info", ...).RequireRateLimiting("default");
   ```
   > This enforces the 100 req/min limit from the Safety Checklist using the built-in ASP.NET Core rate limiter (no external library needed beyond the package added on Day 1).

3. Create test script (`test-skills.ps1`):
   ```powershell
   # Test system/info
   curl -X GET "http://localhost:5000/api/system/info"

   # Test services/status
   curl -X GET "http://localhost:5000/api/services/status?name=docker"

   # Test logs/tail
   curl -X GET "http://localhost:5000/api/logs/tail?source=/var/log/app.log&lines=50"
   ```

4. Create `API.md` documentation:
   - Endpoint specifications
   - Request/response examples
   - Error codes
   - Rate limiting policy

5. Verify `audit.log` is being written correctly

**Deliverable:** Test results, API documentation, audit log sample

---

### Day 5: Docker Deployment & Integration
**Objectives:**
- Containerize the API
- Deploy on lab server or VM
- Test from Unit 2 LLM client

**Tasks:**
1. Create `Dockerfile`:
   ```dockerfile
   FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
   WORKDIR /app
   COPY . .
   RUN dotnet publish -c Release -o out
   
   FROM mcr.microsoft.com/dotnet/aspnet:8.0
   WORKDIR /app
   COPY --from=build /app/out .
   EXPOSE 5000
   CMD ["dotnet", "skills-api.dll"]
   ```

2. Build and deploy:
   ```powershell
   # On lab server or VM
   docker build -t skills-api:latest .
   docker run -d -p 5000:5000 \
     -v /var/log:/logs \
     -e ASPNETCORE_URLS=http://+:5000 \
     --name skills-api \
     skills-api:latest
   ```

3. Test from laptop:
   ```powershell
   # Replace LAB-SERVER-IP with actual IP
   curl -X GET "http://LAB-SERVER-IP:5000/api/system/info"
   ```

4. Integrate with Unit 2 LLM client:
   - Add method to OllamaClient to call skills endpoint
   - Update prompts to reference skill calls
   - Test end-to-end workflow

**Deliverable:** Docker container running, API accessible from network, integration tested

---

## Deliverables Checklist

- [ ] .NET Minimal API created
- [ ] 4 endpoint categories implemented
- [ ] AllowlistValidator enforcing rules
- [ ] AuditLogger writing audit.log
- [ ] All endpoints tested and working
- [ ] Error handling comprehensive
- [ ] API.md documentation complete
- [ ] Dockerfile created and tested
- [ ] Docker image deployed on lab server or VM
- [ ] Network connectivity verified
- [ ] Integration with Unit 2 client tested
- [ ] Sample audit.log and allowlist.json provided

---

## Audit Log Example

```
[2026-03-27T10:30:15Z] caller=llm-client endpoint=/api/system/info parameters=[] success=true
[2026-03-27T10:30:20Z] caller=llm-client endpoint=/api/services/status parameters=[name=docker] success=true
[2026-03-27T10:30:25Z] caller=llm-client endpoint=/api/logs/tail parameters=[source=/var/log/blocked.log] success=false error=ALLOWLIST_VIOLATION
```

---

## Safety Checklist

- [ ] All endpoints are read-only (no write, delete, modify)
- [ ] Allowlist is enforced for all operations
- [ ] Audit logging captures all requests
- [ ] Error messages don't leak sensitive information
- [ ] API requires authentication header (X-Caller)
- [ ] Rate limiting in place (recommended: 100 req/min)
- [ ] Max response sizes enforced
- [ ] Firewall rules restrict to lab network only

---

## Resources

- [.NET Minimal APIs](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/minimal-apis)
- [Docker for .NET](https://learn.microsoft.com/en-us/dotnet/architecture/containerized-lifecycle/)
- [Security Best Practices](../resources/SAFETY.md)

---

## Next Step

Once completed, proceed to **[Unit 4: RAG Integration](04-RAG-INTEGRATION.md)**
