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
   │   ├── files to be added in next steps
   │ 
   ├── config/
   │   └── files to be added in next steps
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

   // Print all registered endpoints at startup so you can verify they loaded
   app.Lifetime.ApplicationStarted.Register(() =>
   {
       var endpoints = app.Services.GetRequiredService<EndpointDataSource>().Endpoints;
       var logger = app.Services.GetRequiredService<ILoggerFactory>().CreateLogger("Startup");
       logger.LogInformation("Registered {Count} endpoints:", endpoints.Count);
       foreach (var ep in endpoints)
           logger.LogInformation("  {Endpoint}", ep.DisplayName);
   });

   app.Run();
   ```

   Build and run to verify all endpoints are registered:
   ```powershell
   dotnet build
   dotnet run
   ```

   You should see this in the console output:
   ```
   info: Startup[0] Registered 4 endpoints:
   info: Startup[0]   HTTP: GET /api/system/info
   info: Startup[0]   HTTP: GET /api/system/disk
   info: Startup[0]   HTTP: GET /api/services/status
   info: Startup[0]   HTTP: GET /api/logs/tail
   ```
   Press `Ctrl+C` to stop.

**Deliverable:** `dotnet run` shows all 4 endpoints registered in the console

---

### Day 4: Testing & Documentation
**Objectives:**
- Test all endpoints including error cases
- Document the API

> Error handling and rate limiting are already built into the code from Days 2–3. Day 4 is about proving they work and documenting what you built.

**Tasks:**

1. Start the API:
   ```powershell
   dotnet run
   ```

2. Create `test-skills.ps1` in the `skills-api/` folder and run it — it tests both happy path and error cases:
   ```powershell
   $base = "http://localhost:5230"

   Write-Host "`n--- System Info ---"
   curl "$base/api/system/info"

   Write-Host "`n--- Disk Usage ---"
   curl "$base/api/system/disk"

   Write-Host "`n--- Service: docker (allowed) ---"
   curl "$base/api/services/status?name=docker"

   Write-Host "`n--- Service: spooler (blocked - not in allowlist) ---"
   curl "$base/api/services/status?name=spooler"

   Write-Host "`n--- Logs: invalid line count (blocked) ---"
   curl "$base/api/logs/tail?source=C:\logs\app.log&lines=9999"

   Write-Host "`n--- audit.log ---"
   Get-Content ./audit.log
   ```
   Run it:
   ```powershell
   .\test-skills.ps1
   ```
   Expected: two blocked requests with `ALLOWLIST_VIOLATION` and `INVALID_LINES`, visible in the audit log at the end.

3. Create `API.md` in the `skills-api/` folder — fill in your actual responses from the test run above:
   ````markdown
   # Skills API

   Base URL: `http://localhost:5230` (dev) / `http://VM-IP:5000` (VM)
   Rate limit: 100 requests/min per endpoint
   Auth: pass `X-Caller: your-app-name` header to identify the caller in audit logs

   ## Endpoints

   ### GET /api/system/info
   Returns OS, machine name, CPU count, memory, and .NET version.

   **Response:**
   ```json
   { "os": "...", "machine": "...", "processors": 8, ... }
   ```

   ### GET /api/system/disk
   Returns disk usage for all ready drives.

   **Response:**
   ```json
   [{ "name": "C:\\", "totalGb": 476.84, "freeGb": 120.5, "usedPercent": 74.7 }]
   ```

   ### GET /api/services/status?name=docker
   Returns service status. Name must be in the allowlist.

   **Allowed values:** docker, networking, storage
   **Response:** `{ "service": "docker", "status": "running" }`
   **Error (403):** `{ "code": "ALLOWLIST_VIOLATION" }`

   ### GET /api/logs/tail?source=PATH&lines=N
   Returns the last N lines of a log file. Source must match an allowlist pattern. Max lines: 1000.

   **Error (403):** `{ "code": "ALLOWLIST_VIOLATION" }`
   **Error (400):** `{ "code": "INVALID_LINES" }`
   **Error (404):** `{ "code": "FILE_NOT_FOUND" }`

   ## Audit Log
   Every request is logged to `audit.log`:
   ```
   [2026-04-04T18:31:30Z] caller=unknown endpoint=/api/services/status parameters=[name=spooler] success=false error=ALLOWLIST_VIOLATION
   ```
   ````

**Deliverable:** `test-skills.ps1` runs with expected blocked responses, `API.md` filled in with your actual output

---

### Day 5: Publish & Deploy to VM
**Objectives:**
- Publish a self-contained Linux binary from Windows
- Deploy and run it on the VM as a systemd service
- Create a reusable deploy script

> Each step is labelled **[Windows]** or **[VM]** so it's clear where to run it.

**Tasks:**

1. **[Windows]** Publish a self-contained Linux binary — run from your `skills-api/` folder:
   ```powershell
   dotnet publish -c Release -r linux-x64 --self-contained -o ./publish
   ```
   This creates a `skills-api/publish/` folder with everything needed to run on Linux — no .NET runtime required on the VM.

2. **[VM]** Create a dedicated service account and deployment folder, then give your login user write access via group membership:
   ```bash
   # Create the service account — no shell, no login
   # You can name it anything, e.g. svc-skillsapi — keep it consistent below
   sudo useradd -r -s /bin/false skills-api

   # Create the deployment folder owned by the service account
   sudo mkdir -p /opt/skills-api
   sudo chown skills-api:skills-api /opt/skills-api

   # Add your login user to the skills-api group and grant group write access
   # -R applies to existing files too — without it SCP will fail with Permission denied
   sudo usermod -aG skills-api YOUR-LOGIN-USER
   sudo chmod -R g+w /opt/skills-api

   # Log out and back in for the group change to take effect
   ```

3. **[Windows]** SCP as your login user — group membership gives you write access to `/opt/skills-api`:
   ```powershell
   # Replace YOUR-LOGIN-USER and VM-IP with yours
   scp -r ./publish/* YOUR-LOGIN-USER@VM-IP:/opt/skills-api/
   ```

4. **[VM]** Make the binary executable and verify it starts:
   ```bash
   sudo chmod +x /opt/skills-api/skills-api
   sudo -u skills-api ASPNETCORE_URLS=http://+:5000 /opt/skills-api/skills-api
   ```
   You should see the registered endpoints in the output. Press `Ctrl+C` to stop.

   > By default ASP.NET binds to `localhost` only — `ASPNETCORE_URLS=http://+:5000` tells it to listen on all interfaces. The systemd service sets this automatically via the `Environment=` line.

5. **[VM]** Create a systemd service so it runs automatically and restarts on failure:
   ```bash
   sudo nano /etc/systemd/system/skills-api.service
   ```
   Paste this as-is — no personalisation needed:
   ```ini
   [Unit]
   Description=Skills API
   After=network.target

   [Service]
   WorkingDirectory=/opt/skills-api
   ExecStart=/opt/skills-api/skills-api
   Restart=always
   RestartSec=5
   User=skills-api
   Environment=ASPNETCORE_URLS=http://+:5000

   [Install]
   WantedBy=multi-user.target
   ```
   Enable and start it:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable skills-api
   sudo systemctl start skills-api
   sudo systemctl status skills-api
   ```

6. **[Windows]** Test the VM's API from your laptop:
   ```powershell
   curl "http://VM-IP:5000/api/system/info"
   curl "http://VM-IP:5000/api/system/disk"
   curl "http://VM-IP:5000/api/services/status?name=docker"
   ```
   The responses should show the **VM's** OS, disk, and service info — not your laptop's.

7. **[Windows]** Create `deploy.ps1` so future deploys are one command — replace `YOUR-LOGIN-USER` and `VM-IP` with yours:
   ```powershell
   # deploy.ps1 — run from the skills-api/ folder
   param(
       [string]$User = "YOUR-LOGIN-USER",  # your SSH login user
       [string]$VMip = "VM-IP"             # your VM's IP address
   )

   Write-Host "Publishing..."
   dotnet publish -c Release -r linux-x64 --self-contained -o ./publish

   # Stop the service before copying — binary can't be overwritten while running
   Write-Host "Stopping service..."
   ssh "${User}@${VMip}" "sudo systemctl stop skills-api"

   # SCP as your login user — group membership grants write access to /opt/skills-api
   Write-Host "Copying to VM..."
   scp -r ./publish/* "${User}@${VMip}:/opt/skills-api/"

   # Start and show status to confirm it's running
   Write-Host "Starting service..."
   ssh "${User}@${VMip}" "sudo systemctl start skills-api && sudo systemctl status skills-api"

   Write-Host "Done. Testing endpoint..."
   curl "http://${VMip}:5000/api/system/info"
   ```
   Run it:
   ```powershell
   .\deploy.ps1 -User youruser -VMip 1.2.3.4
   ```

**Deliverable:** `systemctl status skills-api` shows active on the VM, curl from Windows returns the VM's system info

---

### Stretch Goal: Docker Hub Deploy (Option B)

Once the basic deploy works, try the proper registry-based workflow used in real CI/CD pipelines.

> Requires: Docker Desktop on Windows, a free [Docker Hub](https://hub.docker.com) account.

1. **[Windows]** Create `Dockerfile` in the `skills-api/` folder:
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

2. **[Windows]** Build and push the image to Docker Hub:
   ```powershell
   docker build -t YOUR-DOCKERHUB-USERNAME/skills-api:latest .
   docker push YOUR-DOCKERHUB-USERNAME/skills-api:latest
   ```

3. **[VM]** Pull and run the image — no build step needed on the server:
   ```bash
   docker pull YOUR-DOCKERHUB-USERNAME/skills-api:latest
   docker run -d -p 5000:5000 \
     -v /var/log:/var/log:ro \
     -e ASPNETCORE_URLS=http://+:5000 \
     --name skills-api-docker \
     YOUR-DOCKERHUB-USERNAME/skills-api:latest
   ```

4. **[Windows]** Test it the same way:
   ```powershell
   curl "http://VM-IP:5000/api/system/info"
   ```

> Notice the difference: with Option A the server runs a binary you compiled. With Option B the server runs a container image you built and shipped. The server never touches the source code either way — that's the point.

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
