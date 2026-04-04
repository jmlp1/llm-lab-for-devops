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

3. Add the allowlist and audit config to `appsettings.json` (already in your `skills-api` folder):
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

All files go in the `src/` folder. Copy each file as-is, then read it to understand what it does before moving on.

1. Create `src/AllowlistValidator.cs` — reads the allowlist from `appsettings.json` and validates incoming requests. Notice how `MatchesPattern` handles the `*` wildcard so `/var/log/app/*.log` matches any `.log` file in that folder:
   ```csharp
   public class AllowlistValidator(IConfiguration config)
   {
       public bool IsServiceAllowed(string service)
       {
           var allowed = config.GetSection("allowlist:services").Get<string[]>() ?? [];
           return allowed.Contains(service, StringComparer.OrdinalIgnoreCase);
       }

       public bool IsLogSourceAllowed(string source)
       {
           var patterns = config.GetSection("allowlist:logSources").Get<string[]>() ?? [];
           return patterns.Any(p => MatchesPattern(source, p));
       }

       public bool ValidateTailLines(int lines)
       {
           var max = config.GetValue<int>("allowlist:maxTailLines", 1000);
           return lines > 0 && lines <= max;
       }

       private static bool MatchesPattern(string path, string pattern)
       {
           path = path.Replace('\\', '/');
           pattern = pattern.Replace('\\', '/');

           if (!pattern.Contains('*'))
               return string.Equals(path, pattern, StringComparison.OrdinalIgnoreCase);

           var parts = pattern.Split('*');
           return path.StartsWith(parts[0], StringComparison.OrdinalIgnoreCase)
               && path.EndsWith(parts[1], StringComparison.OrdinalIgnoreCase);
       }
   }
   ```

2. Create `src/AuditLogger.cs` — appends one structured line per request to `audit.log`. The path comes from `appsettings.json` so you can change it without touching code:
   ```csharp
   public class AuditLogger(IConfiguration config)
   {
       private readonly string _logPath = config.GetValue<string>("audit:logPath", "./audit.log")!;
       private readonly bool _enabled = config.GetValue<bool>("audit:enabled", true);

       public void LogOperation(string caller, string endpoint, Dictionary<string, string> parameters, bool success, string? error = null)
       {
           if (!_enabled) return;

           var paramStr = string.Join(", ", parameters.Select(p => $"{p.Key}={p.Value}"));
           var line = $"[{DateTime.UtcNow:O}] caller={caller} endpoint={endpoint} parameters=[{paramStr}] success={success.ToString().ToLower()}";
           if (error != null) line += $" error={error}";

           File.AppendAllText(_logPath, line + Environment.NewLine);
       }
   }
   ```

3. Create `src/SystemSkills.cs` — maps `/api/system/info` and `/api/system/disk`. Notice that every endpoint reads `X-Caller` from the request header and passes it to `AuditLogger` — that's how we know which caller made the request:
   ```csharp
   public static class SystemSkills
   {
       public static void MapSystemEndpoints(WebApplication app)
       {
           app.MapGet("/api/system/info", (AuditLogger audit, HttpContext ctx) =>
           {
               var caller = GetCaller(ctx);
               var info = new
               {
                   os = Environment.OSVersion.ToString(),
                   machine = Environment.MachineName,
                   processors = Environment.ProcessorCount,
                   memory = GetMemoryInfo(),
                   dotnetVersion = Environment.Version.ToString()
               };
               audit.LogOperation(caller, "/api/system/info", [], true);
               return Results.Ok(info);
           }).RequireRateLimiting("default");

           app.MapGet("/api/system/disk", (AuditLogger audit, HttpContext ctx) =>
           {
               var caller = GetCaller(ctx);
               var drives = DriveInfo.GetDrives()
                   .Where(d => d.IsReady)
                   .Select(d => new
                   {
                       name = d.Name,
                       type = d.DriveType.ToString(),
                       totalGb = Math.Round(d.TotalSize / 1_073_741_824.0, 2),
                       freeGb = Math.Round(d.AvailableFreeSpace / 1_073_741_824.0, 2),
                       usedPercent = Math.Round((1.0 - (double)d.AvailableFreeSpace / d.TotalSize) * 100, 1)
                   });
               audit.LogOperation(caller, "/api/system/disk", [], true);
               return Results.Ok(drives);
           }).RequireRateLimiting("default");
       }

       private static object GetMemoryInfo()
       {
           var info = GC.GetGCMemoryInfo();
           return new
           {
               totalMb = Math.Round(info.TotalAvailableMemoryBytes / 1_048_576.0, 0),
               heapMb = Math.Round(GC.GetTotalMemory(false) / 1_048_576.0, 2)
           };
       }

       internal static string GetCaller(HttpContext ctx) =>
           ctx.Request.Headers["X-Caller"].ToString() is { Length: > 0 } c ? c : "unknown";
   }
   ```

4. Create `src/ServiceSkills.cs` — maps `/api/services/status?name=`. Notice the allowlist check happens before the OS command runs — if the service isn't allowed, we never touch the system:
   ```csharp
   public static class ServiceSkills
   {
       public static void MapServiceEndpoints(WebApplication app)
       {
           app.MapGet("/api/services/status", (string name, AllowlistValidator allowlist, AuditLogger audit, HttpContext ctx) =>
           {
               var caller = SystemSkills.GetCaller(ctx);
               var parameters = new Dictionary<string, string> { ["name"] = name };

               if (!allowlist.IsServiceAllowed(name))
               {
                   audit.LogOperation(caller, "/api/services/status", parameters, false, "ALLOWLIST_VIOLATION");
                   return Results.Json(new { error = "Service not in allowlist", code = "ALLOWLIST_VIOLATION", status = 403, timestamp = DateTime.UtcNow }, statusCode: 403);
               }

               var status = CheckServiceStatus(name);
               audit.LogOperation(caller, "/api/services/status", parameters, true);
               return Results.Ok(new { service = name, status, timestamp = DateTime.UtcNow });
           }).RequireRateLimiting("default");
       }

       private static string CheckServiceStatus(string name)
       {
           try
           {
               var (cmd, args) = Environment.OSVersion.Platform == PlatformID.Unix
                   ? ("systemctl", $"is-active {name}")
                   : ("sc", $"query {name}");

               using var process = new System.Diagnostics.Process
               {
                   StartInfo = new System.Diagnostics.ProcessStartInfo
                   {
                       FileName = cmd,
                       Arguments = args,
                       RedirectStandardOutput = true,
                       UseShellExecute = false,
                       CreateNoWindow = true
                   }
               };
               process.Start();
               var output = process.StandardOutput.ReadToEnd().Trim();
               process.WaitForExit();

               if (Environment.OSVersion.Platform == PlatformID.Unix)
                   return process.ExitCode == 0 ? "active" : "inactive";

               return output.Contains("RUNNING") ? "running"
                   : output.Contains("STOPPED") ? "stopped"
                   : "unknown";
           }
           catch
           {
               return "unknown";
           }
       }
   }
   ```

5. Create `src/LogSkills.cs` — maps `/api/logs/tail?source=...&lines=50`. Notice there are three separate validation checks before touching the file — allowlist, line count, then file existence:
   ```csharp
   public static class LogSkills
   {
       public static void MapLogEndpoints(WebApplication app)
       {
           app.MapGet("/api/logs/tail", (string source, int lines, AllowlistValidator allowlist, AuditLogger audit, HttpContext ctx) =>
           {
               var caller = SystemSkills.GetCaller(ctx);
               var parameters = new Dictionary<string, string> { ["source"] = source, ["lines"] = lines.ToString() };

               if (!allowlist.IsLogSourceAllowed(source))
               {
                   audit.LogOperation(caller, "/api/logs/tail", parameters, false, "ALLOWLIST_VIOLATION");
                   return Results.Json(new { error = "Log source not in allowlist", code = "ALLOWLIST_VIOLATION", status = 403, timestamp = DateTime.UtcNow }, statusCode: 403);
               }

               if (!allowlist.ValidateTailLines(lines))
               {
                   audit.LogOperation(caller, "/api/logs/tail", parameters, false, "INVALID_LINES");
                   return Results.Json(new { error = "Line count must be between 1 and 1000", code = "INVALID_LINES", status = 400, timestamp = DateTime.UtcNow }, statusCode: 400);
               }

               if (!File.Exists(source))
               {
                   audit.LogOperation(caller, "/api/logs/tail", parameters, false, "FILE_NOT_FOUND");
                   return Results.Json(new { error = "Log file not found", code = "FILE_NOT_FOUND", status = 404, timestamp = DateTime.UtcNow }, statusCode: 404);
               }

               var content = TailFile(source, lines);
               audit.LogOperation(caller, "/api/logs/tail", parameters, true);
               return Results.Ok(new { source, lines = content.Length, content });
           }).RequireRateLimiting("default");
       }

       private static string[] TailFile(string path, int lines)
       {
           var all = File.ReadAllLines(path);
           return all.Length <= lines ? all : all[^lines..];
       }
   }
   ```

6. Replace the entire content of `Program.cs` with this — it wires everything together: registers the two services, configures rate limiting, adds the `X-Caller` middleware, then maps all endpoints:
   ```csharp
   using Microsoft.AspNetCore.RateLimiting;
   using System.Threading.RateLimiting;

   var builder = WebApplication.CreateBuilder(args);

   builder.Services.AddSingleton<AllowlistValidator>();
   builder.Services.AddSingleton<AuditLogger>();

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

   var app = builder.Build();

   app.UseRateLimiter();

   // Ensure X-Caller header is always set — individual endpoints use it for audit logging
   app.Use(async (context, next) =>
   {
       if (!context.Request.Headers.ContainsKey("X-Caller"))
           context.Request.Headers.Append("X-Caller", "unknown");
       await next.Invoke();
   });

   SystemSkills.MapSystemEndpoints(app);
   ServiceSkills.MapServiceEndpoints(app);
   LogSkills.MapLogEndpoints(app);

   app.Run();
   ```

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
