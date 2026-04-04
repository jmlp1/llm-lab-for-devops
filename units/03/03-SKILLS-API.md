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

3. Design endpoints (read-only only):
   ```
   GET  /api/system/info
   GET  /api/system/disk
   GET  /api/services/status?name=...
   GET  /api/logs/tail?source=...&lines=100
   ```

4. Create configuration schema:
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

5. Create folder structure:
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
   
   Log paths to allowlist:
   - /var/log/app/*.log (Linux)
   - ./ai-lab/samples/*.log (Windows)
   ```

**Deliverable:** Project structure, endpoints designed, allowlist schema created

---

### Day 2–3: Implement Endpoints & Validators
**Objectives:**
- Implement all 4 endpoint categories
- Add allowlist validation
- Add audit logging middleware

**Tasks:**
1. Create `AllowlistValidator.cs`:
   ```csharp
   public class AllowlistValidator
   {
       private readonly IConfiguration _config;

       public bool IsServiceAllowed(string service) { }
       public bool IsLogSourceAllowed(string source) { }
       public bool ValidateTailLines(int lines) { }
   }
   ```

2. Create `AuditLogger.cs`:
   ```csharp
   public class AuditLogger
   {
       public void LogOperation(string caller, string endpoint, 
           Dictionary<string, string> parameters, bool success, string? error = null)
       {
           var entry = new
           {
               timestamp = DateTime.UtcNow,
               caller = caller,
               endpoint = endpoint,
               parameters = parameters,
               success = success,
               error = error
           };
           // Write to audit.log
       }
   }
   ```

3. Implement `SystemSkills.cs`:
   - `/api/system/info` — CPU, memory, OS info
   - `/api/system/disk` — disk usage per mount point

4. Implement `ServiceSkills.cs`:
   - `/api/services/status?name=docker` — check service status
   - Validate service name against allowlist

5. Implement `LogSkills.cs`:
   - `/api/logs/tail?source=/var/log/app.log&lines=50`
   - Validate source against allowlist
   - Enforce max lines limit

6. Add middleware for audit logging:
   ```csharp
   app.Use(async (context, next) =>
   {
       var auditLogger = context.RequestServices.GetRequiredService<AuditLogger>();
       var caller = context.Request.Headers["X-Caller"].ToString() ?? "unknown";
       auditLogger.LogOperation(caller, context.Request.Path, ...);
       await next.Invoke();
   });
   ```

**Deliverable:** All 4 endpoint categories working, validators enforcing rules, audit logging active

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
