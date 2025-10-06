# Security Remediation Recommendations
## PolarionMcpServers - Actionable Security Improvements

**Priority-Based Implementation Guide**  
**Based on IEC 27017:2015 Compliance Assessment**

---

## Quick Reference: Attention Levels

- 🔴 **HIGH** - Critical security risk requiring immediate action
- 🟡 **MEDIUM** - Significant security concern requiring planned remediation  
- 🟢 **LOW** - Minor security improvement or best practice enhancement

---

## Phase 1: Critical Security Fixes (0-30 Days)

### 1. Credential and Secrets Management 🔴 HIGH
**Current Score:** 2/10 | **Target Score:** 8/10  
**Effort:** 16-24 hours | **Impact:** Critical

#### Problem Statement
Hardcoded passwords in configuration files pose immediate security risk. Seven instances of plaintext credentials found across appsettings.json files.

#### Specific Actions

**Action 1.1: Remove Hardcoded Passwords**
```bash
# Files to modify:
- PolarionRemoteMcpServer/appsettings.json
- PolarionMcpServer/appsettings.json
- README.md (example passwords)
- CHANGELOG.md (example passwords)
```

**Implementation:**
1. Replace all password values with environment variable references
2. Update configuration loading to support environment variables
3. Document the new configuration method

**Code Changes:**
```csharp
// In appsettings.json - REMOVE passwords
{
  "SessionConfig": {
    "ServerUrl": "https://polarion.int.mycompany.com/",
    "Username": "shared_user_read_only",
    "Password": "", // REMOVE - use environment variable
    "ProjectId": "Starlight_Main"
  }
}

// In Program.cs - ADD environment variable support
var password = Environment.GetEnvironmentVariable("POLARION_PASSWORD_STARLIGHT") 
               ?? builder.Configuration["PolarionProjects:0:SessionConfig:Password"];
```

**Action 1.2: Implement Azure Key Vault Integration**

**Prerequisites:**
- Azure subscription with Key Vault created
- Managed Identity or Service Principal configured

**Implementation:**
```bash
# Install package
dotnet add package Azure.Extensions.AspNetCore.Configuration.Secrets
dotnet add package Azure.Identity
```

**Code Changes:**
```csharp
// In Program.cs
using Azure.Identity;

var builder = WebApplication.CreateBuilder(args);

// Add Azure Key Vault
var keyVaultEndpoint = new Uri(Environment.GetEnvironmentVariable("KEY_VAULT_ENDPOINT"));
builder.Configuration.AddAzureKeyVault(keyVaultEndpoint, new DefaultAzureCredential());
```

**Key Vault Secret Names:**
```
polarion-starlight-password
polarion-octopus-password
polarion-grogu-password
```

**Action 1.3: Update Documentation**
- Add security best practices section to README
- Remove example passwords from all documentation
- Add Key Vault configuration guide

**Verification:**
```bash
# Test that application starts without hardcoded passwords
dotnet run --no-build
# Verify logs show successful credential retrieval
grep "Creating Polarion client" logs/*.log
```

**Acceptance Criteria:**
- ✅ No passwords in any committed files
- ✅ Application runs with environment variables
- ✅ Documentation updated
- ✅ CI/CD pipeline supports secure configuration

---

### 2. API Authentication Layer 🔴 HIGH
**Current Score:** 5/10 | **Target Score:** 8/10  
**Effort:** 20-30 hours | **Impact:** High

#### Problem Statement
MCP endpoints have no authentication, allowing unrestricted access to anyone with network connectivity.

#### Specific Actions

**Action 2.1: Implement API Key Authentication**

**Implementation:**
```bash
# Install package
dotnet add package Microsoft.AspNetCore.Authentication
```

**Code Changes:**
```csharp
// Create new file: Middleware/ApiKeyAuthenticationMiddleware.cs
public class ApiKeyAuthenticationMiddleware
{
    private readonly RequestDelegate _next;
    private readonly IConfiguration _configuration;
    private const string API_KEY_HEADER = "X-API-Key";

    public ApiKeyAuthenticationMiddleware(RequestDelegate next, IConfiguration configuration)
    {
        _next = next;
        _configuration = configuration;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        if (!context.Request.Headers.TryGetValue(API_KEY_HEADER, out var extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("API Key missing");
            return;
        }

        var apiKey = _configuration.GetValue<string>("ApiKey");
        if (!apiKey.Equals(extractedApiKey))
        {
            context.Response.StatusCode = 401;
            await context.Response.WriteAsync("Invalid API Key");
            return;
        }

        await _next(context);
    }
}

// In Program.cs
app.UseMiddleware<ApiKeyAuthenticationMiddleware>();
```

**Configuration:**
```json
// appsettings.json
{
  "ApiKey": "", // Load from environment or Key Vault
  "ApiKeyValidationEnabled": true
}
```

**Action 2.2: Add Rate Limiting**

```bash
dotnet add package AspNetCoreRateLimit
```

**Code Changes:**
```csharp
// In Program.cs
builder.Services.AddMemoryCache();
builder.Services.Configure<IpRateLimitOptions>(builder.Configuration.GetSection("IpRateLimiting"));
builder.Services.AddInMemoryRateLimiting();
builder.Services.AddSingleton<IRateLimitConfiguration, RateLimitConfiguration>();

// Add middleware
app.UseIpRateLimiting();
```

**Configuration:**
```json
// appsettings.json
{
  "IpRateLimiting": {
    "EnableEndpointRateLimiting": true,
    "StackBlockedRequests": false,
    "GeneralRules": [
      {
        "Endpoint": "*",
        "Period": "1m",
        "Limit": 100
      }
    ]
  }
}
```

**Verification:**
```bash
# Test without API key
curl http://localhost:8080/starlight/sse
# Expected: 401 Unauthorized

# Test with valid API key
curl -H "X-API-Key: your-api-key" http://localhost:8080/starlight/sse
# Expected: 200 OK or appropriate response
```

**Acceptance Criteria:**
- ✅ API key required for all MCP endpoints
- ✅ Invalid keys rejected with 401
- ✅ Rate limiting prevents abuse
- ✅ API keys stored securely (not hardcoded)

---

### 3. Enforce HTTPS in Production 🔴 HIGH
**Current Score:** 6/10 | **Target Score:** 9/10  
**Effort:** 8-12 hours | **Impact:** High

#### Problem Statement
HTTPS is available but not enforced. Docker deployment examples use HTTP, creating man-in-the-middle risks.

#### Specific Actions

**Action 3.1: Add HTTPS Redirection**

**Code Changes:**
```csharp
// In Program.cs
app.UseHttpsRedirection();

// Add HSTS for production
if (app.Environment.IsProduction())
{
    app.UseHsts();
}

// Configure HSTS
builder.Services.AddHsts(options =>
{
    options.Preload = true;
    options.IncludeSubDomains = true;
    options.MaxAge = TimeSpan.FromDays(365);
});
```

**Action 3.2: Update Docker Configuration**

**Create Dockerfile (if not exists):**
```dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS base
WORKDIR /app
EXPOSE 8080
EXPOSE 8443

# Run as non-root user
RUN adduser --disabled-password --gecos '' appuser && chown -R appuser /app
USER appuser

FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src
COPY ["PolarionRemoteMcpServer/PolarionRemoteMcpServer.csproj", "PolarionRemoteMcpServer/"]
RUN dotnet restore "PolarionRemoteMcpServer/PolarionRemoteMcpServer.csproj"
COPY . .
WORKDIR "/src/PolarionRemoteMcpServer"
RUN dotnet build "PolarionRemoteMcpServer.csproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "PolarionRemoteMcpServer.csproj" -c Release -o /app/publish

FROM base AS final
WORKDIR /app
COPY --from=publish /app/publish .
ENTRYPOINT ["dotnet", "PolarionRemoteMcpServer.dll"]
```

**Update docker-compose.yml:**
```yaml
version: '3.8'
services:
  polarion-mcp:
    image: peakflames/polarion-remote-mcp-server:latest
    ports:
      - "8443:8443"  # HTTPS only
    environment:
      - ASPNETCORE_URLS=https://+:8443
      - ASPNETCORE_Kestrel__Certificates__Default__Path=/certs/cert.pfx
      - ASPNETCORE_Kestrel__Certificates__Default__Password=${CERT_PASSWORD}
    volumes:
      - ./appsettings.json:/app/appsettings.json:ro
      - ./certs:/certs:ro
    restart: unless-stopped
```

**Action 3.3: Update Documentation**

Update README.md:
```markdown
## Security Requirements

### HTTPS Configuration (Required for Production)

1. Generate or obtain SSL certificate
2. Configure certificate in appsettings.json or environment variables
3. Update docker run command to use HTTPS port

**DO NOT deploy to production without HTTPS enabled.**
```

**Verification:**
```bash
# Test HTTP redirect
curl -I http://localhost:8080/starlight/sse
# Expected: 308 Permanent Redirect to HTTPS

# Test HTTPS
curl -k https://localhost:8443/starlight/sse
# Expected: Successful connection
```

**Acceptance Criteria:**
- ✅ HTTP automatically redirects to HTTPS
- ✅ HSTS header present in responses
- ✅ Docker deployment uses HTTPS
- ✅ Documentation updated with HTTPS requirements

---

### 4. Container Security Hardening 🔴 HIGH
**Current Score:** 4/10 | **Target Score:** 8/10  
**Effort:** 12-16 hours | **Impact:** Medium-High

#### Problem Statement
Container likely runs as root, no security scanning, base image unknown, creating container escape risks.

#### Specific Actions

**Action 4.1: Run as Non-Root User**

Already included in Dockerfile above. Key points:
```dockerfile
RUN adduser --disabled-password --gecos '' appuser && chown -R appuser /app
USER appuser
```

**Action 4.2: Add Security Scanning**

```bash
# Install Trivy for vulnerability scanning
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image peakflames/polarion-remote-mcp-server:latest

# Add to CI/CD pipeline
```

**GitHub Actions Workflow (.github/workflows/security-scan.yml):**
```yaml
name: Security Scan

on:
  push:
    branches: [ develop, main ]
  pull_request:
    branches: [ develop, main ]
  schedule:
    - cron: '0 0 * * 0'  # Weekly

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Build Docker image
        run: docker build -t polarion-mcp:scan .
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'polarion-mcp:scan'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

**Action 4.3: Add Resource Limits**

**docker-compose.yml:**
```yaml
services:
  polarion-mcp:
    # ... existing config ...
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M
    security_opt:
      - no-new-privileges:true
    read_only: true
    tmpfs:
      - /tmp
      - /app/logs
```

**Action 4.4: Add Health Checks**

**Code Changes:**
```csharp
// In Program.cs
app.MapHealthChecks("/health");

builder.Services.AddHealthChecks()
    .AddCheck("self", () => HealthCheckResult.Healthy());
```

**Dockerfile:**
```dockerfile
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f https://localhost:8443/health || exit 1
```

**Verification:**
```bash
# Check container user
docker exec <container-id> whoami
# Expected: appuser (not root)

# Run security scan
trivy image peakflames/polarion-remote-mcp-server:latest
# Expected: No CRITICAL vulnerabilities

# Check resource limits
docker stats <container-id>
# Expected: Memory capped at 512M
```

**Acceptance Criteria:**
- ✅ Container runs as non-root user
- ✅ Automated vulnerability scanning in CI/CD
- ✅ Resource limits configured
- ✅ Health checks implemented
- ✅ Security options configured

---

### 5. Access Control Implementation 🔴 HIGH
**Current Score:** 3/10 | **Target Score:** 7/10  
**Effort:** 24-32 hours | **Impact:** High

#### Problem Statement
No user-based access control. All clients with network access can use any configured project.

#### Specific Actions

**Action 5.1: Implement Project-Level Access Control**

**Create new configuration structure:**
```json
// appsettings.json
{
  "PolarionProjects": [
    {
      "ProjectUrlAlias": "starlight",
      "AccessControl": {
        "Enabled": true,
        "AllowedApiKeys": [
          "key-starlight-team-alpha",
          "key-starlight-team-beta"
        ]
      }
    }
  ]
}
```

**Code Changes:**
```csharp
// Create new file: Models/ProjectAccessControl.cs
public class ProjectAccessControl
{
    public bool Enabled { get; set; }
    public List<string> AllowedApiKeys { get; set; } = new();
}

// Update PolarionProjectConfig.cs
public class PolarionProjectConfig
{
    // ... existing properties ...
    public ProjectAccessControl? AccessControl { get; set; }
}

// Create new file: Middleware/ProjectAccessControlMiddleware.cs
public class ProjectAccessControlMiddleware
{
    private readonly RequestDelegate _next;
    private readonly List<PolarionProjectConfig> _projectConfigs;

    public async Task InvokeAsync(HttpContext context)
    {
        var projectId = context.GetRouteValue("projectId")?.ToString();
        var config = _projectConfigs.FirstOrDefault(p => 
            p.ProjectUrlAlias.Equals(projectId, StringComparison.OrdinalIgnoreCase));

        if (config?.AccessControl?.Enabled == true)
        {
            var apiKey = context.Request.Headers["X-API-Key"].FirstOrDefault();
            if (!config.AccessControl.AllowedApiKeys.Contains(apiKey))
            {
                context.Response.StatusCode = 403;
                await context.Response.WriteAsync($"Access denied to project '{projectId}'");
                return;
            }
        }

        await _next(context);
    }
}
```

**Action 5.2: Add Access Logging**

```csharp
// In middleware
_logger.LogWarning(
    "Access attempt to project {ProjectId} from {IpAddress} with API key {ApiKeyPrefix}",
    projectId,
    context.Connection.RemoteIpAddress,
    apiKey?.Substring(0, Math.Min(8, apiKey.Length)) + "***"
);
```

**Verification:**
```bash
# Test access control
curl -H "X-API-Key: unauthorized-key" http://localhost:8080/starlight/sse
# Expected: 403 Forbidden

curl -H "X-API-Key: key-starlight-team-alpha" http://localhost:8080/starlight/sse
# Expected: 200 OK
```

**Acceptance Criteria:**
- ✅ Project-level access control implemented
- ✅ API keys validated per project
- ✅ Access attempts logged
- ✅ 403 status for unauthorized access

---

## Phase 2: High Priority Improvements (30-90 Days)

### 6. Data Encryption at Rest 🟡 MEDIUM
**Current Score:** 4/10 | **Target Score:** 8/10  
**Effort:** 16-24 hours

#### Actions:
1. Encrypt configuration files using Data Protection API
2. Implement log file encryption
3. Add database encryption (if applicable)
4. Document encryption key management

**Implementation Guide:**
```csharp
// Use ASP.NET Core Data Protection
builder.Services.AddDataProtection()
    .PersistKeysToAzureBlobStorage(/* connection */)
    .ProtectKeysWithAzureKeyVault(/* key vault */)
    .SetApplicationName("PolarionMcpServer");
```

---

### 7. Input Sanitization for Lucene Queries 🟡 MEDIUM
**Current Score:** 6/10 | **Target Score:** 9/10  
**Effort:** 8-12 hours

#### Actions:
1. Create Lucene query sanitization utility
2. Escape special characters in user input
3. Validate query syntax before execution
4. Add query complexity limits

**Implementation:**
```csharp
// Create new file: Utils/LuceneQuerySanitizer.cs
public static class LuceneQuerySanitizer
{
    private static readonly char[] SpecialChars = { '+', '-', '&', '|', '!', '(', ')', '{', '}', 
                                                     '[', ']', '^', '"', '~', '*', '?', ':', '\\' };
    
    public static string Sanitize(string input)
    {
        if (string.IsNullOrWhiteSpace(input))
            return string.Empty;
        
        var sb = new StringBuilder(input.Length * 2);
        foreach (var c in input)
        {
            if (SpecialChars.Contains(c))
                sb.Append('\\');
            sb.Append(c);
        }
        return sb.ToString();
    }
}

// In McpTools_SearchWorkitemsInDocument.cs
var sanitizedTerms = LuceneQuerySanitizer.Sanitize(textSearchTerms);
var descriptionQuery = $"description:({sanitizedTerms.Trim()})";
```

---

### 8. Dependency Vulnerability Scanning 🟡 MEDIUM
**Current Score:** 6/10 | **Target Score:** 9/10  
**Effort:** 4-8 hours

#### Actions:
1. Add Dependabot configuration
2. Add OWASP Dependency Check to CI/CD
3. Configure automated security updates
4. Create dependency review process

**GitHub Configuration (.github/dependabot.yml):**
```yaml
version: 2
updates:
  - package-ecosystem: "nuget"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
    reviewers:
      - "security-team"
    labels:
      - "dependencies"
      - "security"
```

---

### 9. Security Documentation 🟡 MEDIUM
**Current Score:** 6/10 | **Target Score:** 9/10  
**Effort:** 12-16 hours

#### Actions:
1. Create SECURITY.md with vulnerability reporting process
2. Add security configuration guide
3. Document secure deployment practices
4. Create security checklist for production

**Create SECURITY.md:**
```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 0.4.x   | :white_check_mark: |
| < 0.4   | :x:                |

## Reporting a Vulnerability

Please report security vulnerabilities to security@yourcompany.com

## Security Best Practices

1. Never use hardcoded credentials
2. Always enable HTTPS in production
3. Use API key authentication
4. Configure appropriate rate limiting
5. Run containers as non-root user
```

---

### 10. Session Security Enhancements 🟡 MEDIUM
**Current Score:** 5/10 | **Target Score:** 8/10  
**Effort:** 12-16 hours

#### Actions:
1. Implement session timeout for MCP connections
2. Add session invalidation on suspicious activity
3. Implement concurrent session limits
4. Add session monitoring

---

## Phase 3: Medium Priority (90-180 Days)

### 11. SIEM Integration 🟢 LOW
**Effort:** 16-24 hours

#### Actions:
1. Add Serilog sink for Elasticsearch/Splunk
2. Configure structured logging for security events
3. Create security dashboards
4. Set up alerting rules

---

### 12. Security Testing Automation 🟢 LOW
**Effort:** 20-30 hours

#### Actions:
1. Implement SAST with SonarQube
2. Add DAST with OWASP ZAP
3. Create security unit tests
4. Add penetration testing to release process

---

### 13. Advanced Monitoring 🟢 LOW
**Effort:** 16-24 hours

#### Actions:
1. Implement Application Insights
2. Add custom security metrics
3. Create anomaly detection rules
4. Set up security alerts

---

## Implementation Checklist

### Pre-Implementation
- [ ] Review and approve remediation plan
- [ ] Allocate resources and timeline
- [ ] Set up development/staging environment
- [ ] Create backup of current configuration

### Phase 1 Implementation
- [ ] Complete Action 1: Credential Management
- [ ] Complete Action 2: API Authentication
- [ ] Complete Action 3: HTTPS Enforcement
- [ ] Complete Action 4: Container Security
- [ ] Complete Action 5: Access Control
- [ ] Run security verification tests
- [ ] Update documentation
- [ ] Deploy to staging
- [ ] Conduct security review
- [ ] Deploy to production

### Post-Implementation
- [ ] Monitor logs for security events
- [ ] Verify metrics improvement
- [ ] Update security assessment
- [ ] Plan Phase 2 implementation
- [ ] Conduct team training

---

## Success Metrics

### Phase 1 Targets
- **Overall Security Score:** 5.2 → 7.8 (+50%)
- **Critical Vulnerabilities:** 15 → 2 (-87%)
- **Compliance:** 48% → 78% (+30%)
- **Risk Exposure:** $1.18M → $472K (-60%)

### Measurement Methods
1. Re-run security assessment after each phase
2. Track vulnerability count over time
3. Monitor security incidents
4. Measure compliance percentage
5. Review access logs for anomalies

---

## Support and Resources

### Tools Required
- Azure Key Vault or equivalent secrets manager
- Docker vulnerability scanner (Trivy)
- Dependency scanning tool (Dependabot)
- SSL certificates for HTTPS
- Rate limiting library (AspNetCoreRateLimit)

### Documentation Links
- IEC 27017:2015 Standard
- OWASP Top 10
- Microsoft Security Best Practices
- Docker Security Guide
- ASP.NET Core Security Documentation

### Training Needs
- Secure coding practices
- Secrets management
- Container security
- API security
- Incident response

---

**Document Version:** 1.0  
**Last Updated:** 2024  
**Next Review:** After Phase 1 completion  
