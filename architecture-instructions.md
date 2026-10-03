# Architecture Instructions
## Standard Patterns and Practices for .NET Applications

**Version:** 1.0  
**Last Updated:** August 2026  
**Applies To:** All .NET API and Web Applications

---

## Table of Contents

1. [Architecture Overview](#architecture-overview)
2. [Mandatory Patterns](#mandatory-patterns)
3. [Project Structure Standards](#project-structure-standards)
4. [CQRS Implementation Guide](#cqrs-implementation-guide)
5. [Dependency Injection Guidelines](#dependency-injection-guidelines)
6. [Error Handling Standards](#error-handling-standards)
7. [Logging Standards](#logging-standards)
8. [Secrets Management](#secrets-management)
9. [Package Management](#package-management)
10. [Testing Requirements](#testing-requirements)
11. [Performance Standards](#performance-standards)
12. [Security Standards](#security-standards)
13. [Appendix: Reference Implementations](#appendix-reference-implementations)

---

## Architecture Overview

### Core Principles

Applications **MUST** follow these architectural principles:

1. **Vertical Slice Architecture** - Features organized by business capability, not technical layer
2. **CQRS** - Clear separation between Commands (writes) and Queries (reads)
3. **Dependency Inversion** - Depend on abstractions, not concretions
4. **Single Responsibility** - Each class/method has one reason to change
5. **DRY (Don't Repeat Yourself)** - Avoid code duplication through abstraction

### Architecture Decision Framework

When choosing between patterns, evaluate:

| Criteria | Simple Approach | Advanced Approach |
|----------|----------------|-------------------|
| **Team Size** | < 5 developers | 5+ developers |
| **Codebase Size** | < 20 features | 20+ features |
| **Team Experience** | Junior/Mid | Senior/Expert |
| **Project Timeline** | < 6 months | 6+ months |
| **Maintenance Window** | Short-term | Long-term |

**Recommendation:**
- **New/Small Projects:** Use Simple Approach (AMS-style)
- **Large/Complex Projects:** Use Advanced Approach (Insured Portal-style)

---

## Mandatory Patterns

### 1. Vertical Slice Architecture ✅ REQUIRED

**Definition:** Organize code by feature/use-case, not by technical layer.

**Structure:**
```
Features/
  ├── Agencies/              ← One business feature
  │   ├── Commands/          ← All write operations
  │   ├── Queries/           ← All read operations
  │   ├── AgenciesController.cs
  │   ├── AgenciesMapper.cs
  │   └── AgenciesValidator.cs
  └── Policies/              ← Another business feature
      ├── Commands/
      ├── Queries/
      └── PoliciesController.cs
```

**Benefits:**
- Easy to locate all code for a feature
- Reduces merge conflicts (teams work on different features)
- Enables feature toggles and incremental rollout
- Clear ownership boundaries

**Anti-Pattern (AVOID):**
```
Controllers/
  ├── AgenciesController.cs
  ├── PoliciesController.cs
Services/
  ├── AgencyService.cs
  ├── PolicyService.cs
Repositories/
  ├── AgencyRepository.cs
  ├── PolicyRepository.cs
```

---

### 2. CQRS (Command Query Responsibility Segregation) ✅ REQUIRED

**Definition:** Separate read operations (Queries) from write operations (Commands).

#### Command Pattern

**Commands:**
- Change system state
- Return success/failure or created entity
- Can have side effects
- Should be idempotent when possible

**Example:**
```csharp
// Command definition
public record CreateAgencyCommand(AgencyForCreationOrUpdate Agency);

// Command handler
public class CreateAgencyHandler : ICommandHandler<CreateAgencyCommand, AgencyForCreationOrUpdate>
{
    private readonly DbContext _context;
    
    public async Task<AgencyForCreationOrUpdate> HandleAsync(
        CreateAgencyCommand command, 
        CancellationToken ct = default)
    {
        var entity = _mapper.Map(command.Agency);
        await _context.Agencies.AddAsync(entity, ct);
        await _context.SaveChangesAsync(ct);
        return _mapper.Map(entity);
    }
}
```

#### Query Pattern

**Queries:**
- Do NOT change system state
- Return data
- Should be cacheable
- Can use read-optimized models

**Example:**
```csharp
// Query definition
public record GetAgenciesQuery(AgencyParameters Parameters);

// Query handler
public class GetAgenciesHandler : IQueryHandler<GetAgenciesQuery, (IEnumerable<Agency>, PaginationMetadata)>
{
    private readonly DbContext _context;
    
    public async Task<(IEnumerable<Agency>, PaginationMetadata)> HandleAsync(
        GetAgenciesQuery query, 
        CancellationToken ct = default)
    {
        var agencies = await _context.Agencies
            .Where(a => a.IsActive)
            .ToListAsync(ct);
            
        return (agencies, new PaginationMetadata(...));
    }
}
```

---

### 3. Dependency Injection Standards

#### Service Lifetimes

**Choose the correct lifetime:**

```csharp
// TRANSIENT - New instance every time (default for stateless services)
builder.Services.AddTransient<IEmailService, EmailService>();

// SCOPED - One instance per HTTP request (default for DB contexts)
builder.Services.AddScoped<IAgencyRepository, AgencyRepository>();
builder.Services.AddScoped<DbContext>();

// SINGLETON - One instance for application lifetime (use sparingly)
builder.Services.AddSingleton<IApplicationConfiguration, ApplicationConfiguration>();
builder.Services.AddSingleton<ISecretManager>(secretManager);
```

**Rules:**
- ✅ **DO** use Scoped for database contexts and unit-of-work patterns
- ✅ **DO** use Transient for stateless services
- ✅ **DO** use Singleton for truly global state only
- ❌ **DON'T** inject Scoped services into Singletons
- ❌ **DON'T** inject Transient DbContexts

---

## Project Structure Standards

### Recommended Project Structure

```
Solution Root/
├── DJB.[Domain].Api/                    ← API Project
│   ├── Features/                        ← Vertical slices
│   │   ├── Agencies/
│   │   │   ├── Commands/
│   │   │   │   ├── CreateAgencyCommand.cs
│   │   │   │   └── CreateAgencyHandler.cs
│   │   │   ├── Queries/
│   │   │   │   ├── GetAgenciesQuery.cs
│   │   │   │   └── GetAgenciesHandler.cs
│   │   │   ├── AgenciesController.cs
│   │   │   ├── AgenciesMapper.cs
│   │   │   └── AgenciesValidator.cs
│   │   └── _Shared/                     ← Shared abstractions
│   │       ├── ICommandHandler.cs
│   │       ├── IQueryHandler.cs
│   │       └── Result.cs
│   ├── Models/                          ← Domain entities
│   ├── Configuration/                   ← App configuration
│   ├── Services/                        ← Infrastructure services
│   └── Program.cs                       ← Startup
├── DJB.[Domain].Models/                 ← Shared models
│   ├── Entities/                        ← Database entities
│   ├── DTOs/                           ← Data transfer objects
│   └── Configuration/                   ← Configuration models
└── DJB.[Domain].Tests/                  ← Test project
    ├── Features/                        ← Feature tests
    └── Integration/                     ← Integration tests
```

### Naming Conventions

**Commands:**
- Pattern: `{Verb}{Entity}Command`
- Examples: `CreateAgencyCommand`, `UpdatePolicyCommand`, `DeleteClaimCommand`

**Queries:**
- Pattern: `Get{Entity}Query` or `Get{Entity}{Modifier}Query`
- Examples: `GetAgenciesQuery`, `GetAgencyByIdQuery`, `GetActivePoliciesQuery`

**Handlers:**
- Pattern: `{CommandOrQuery}Handler`
- Examples: `CreateAgencyHandler`, `GetAgenciesHandler`

**Controllers:**
- Pattern: `{Entity}Controller` (plural)
- Examples: `AgenciesController`, `PoliciesController`, `ClaimsController`

---

## CQRS Implementation Guide

### Option A: Simple CQRS (Recommended for New Projects)

**When to use:**
- Small to medium projects (< 20 features)
- Team < 5 developers
- Project timeline < 6 months

**Implementation:**

1. **Define interfaces:**
```csharp
// Features/_Shared/ICommandHandler.cs
public interface ICommandHandler<TCommand, TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken ct = default);
}

// Features/_Shared/IQueryHandler.cs
public interface IQueryHandler<TQuery, TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken ct = default);
}
```

2. **Create registration extensions:**
```csharp
// Features/_Shared/HandlerRegistrationExtensions.cs
public static class HandlerRegistrationExtensions
{
    public static IServiceCollection AddQueryHandler<TQuery, TResult, THandler>(
        this IServiceCollection services)
        where THandler : class, IQueryHandler<TQuery, TResult>
    {
        services.AddScoped<THandler>();
        services.AddScoped<IQueryHandler<TQuery, TResult>>(sp =>
            new LoggingQueryHandler<TQuery, TResult>(
                sp.GetRequiredService<THandler>(),
                sp.GetRequiredService<ILogger<LoggingQueryHandler<TQuery, TResult>>>()));
        return services;
    }
    
    // Similar for AddCommandHandler
}
```

3. **Register handlers in Program.cs:**
```csharp
// Program.cs
builder.Services.AddQueryHandler<GetAgenciesQuery, IEnumerable<Agency>, GetAgenciesHandler>();
builder.Services.AddCommandHandler<CreateAgencyCommand, Agency, CreateAgencyHandler>();
```

**Pros:**
- ✅ Explicit and clear
- ✅ Easy to debug
- ✅ No "magic"
- ✅ IDE-friendly (easy to find registrations)

**Cons:**
- ⚠️ Manual registration (can forget)
- ⚠️ Verbose for large projects

---

### Option B: Advanced CQRS with Pipeline (Recommended for Large Projects)

**When to use:**
- Large projects (20+ features)
- Team 5+ developers
- Long-term maintenance (6+ months)
- Need for validation, logging, caching behaviors

**Implementation:**

1. **Define Result pattern:**
```csharp
// Features/_Shared/Result.cs
public sealed class Result<T>
{
    public bool IsSuccess => Status == ResultStatus.Ok;
    public T Value { get; }
    public string ErrorMessage { get; }
    public ResultStatus Status { get; }

    private Result(ResultStatus status, T value, string errorMessage)
    {
        Status = status;
        Value = value;
        ErrorMessage = errorMessage;
    }

    public static Result<T> Ok(T value) => new(ResultStatus.Ok, value, null);
    public static Result<T> NotFound(string message = null) => new(ResultStatus.NotFound, default, message);
    public static Result<T> BadRequest(string message = null) => new(ResultStatus.BadRequest, default, message);
    public static Result<T> Error(string message = null) => new(ResultStatus.Error, default, message);
}

public enum ResultStatus { Ok, NotFound, BadRequest, Unauthorized, Error }
```

2. **Define interfaces with Result:**
```csharp
// Features/_Shared/ICommandHandler.cs
public interface ICommandHandler<TRequest, TResponse>
{
    Task<Result<TResponse>> HandleAsync(TRequest request, CancellationToken ct = default);
}
```

3. **Create pipeline behaviors:**
```csharp
// Features/_Shared/IPipelineBehavior.cs
public interface IPipelineBehavior<TRequest, TResponse>
{
    Task<Result<TResponse>> HandleAsync(
        TRequest request,
        Func<Task<Result<TResponse>>> next,
        CancellationToken ct = default);
}

// Features/_Shared/Behaviors/LoggingBehavior.cs
public sealed class LoggingBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
{
    private readonly ILogger<LoggingBehavior<TRequest, TResponse>> _logger;

    public async Task<Result<TResponse>> HandleAsync(
        TRequest request,
        Func<Task<Result<TResponse>>> next,
        CancellationToken ct = default)
    {
        var requestName = typeof(TRequest).Name;
        _logger.LogInformation("Handling {RequestName}", requestName);
        
        var sw = Stopwatch.StartNew();
        var result = await next();
        sw.Stop();
        
        _logger.LogInformation("Handled {RequestName} in {ElapsedMs}ms", 
            requestName, sw.ElapsedMilliseconds);
        
        return result;
    }
}
```

4. **Auto-register handlers:**
```csharp
// Features/_Shared/ServiceCollectionExtensions.cs
public static IServiceCollection AddFeatureHandlers(this IServiceCollection services)
{
    var assembly = Assembly.GetExecutingAssembly();

    // Register pipeline behaviors
    services.AddScoped(typeof(IPipelineBehavior<,>), typeof(LoggingBehavior<,>));
    services.AddScoped(typeof(IPipelineBehavior<,>), typeof(ValidationBehavior<,>));

    // Auto-discover handlers
    RegisterHandlers(services, assembly, typeof(IQueryHandler<,>));
    RegisterHandlers(services, assembly, typeof(ICommandHandler<,>));

    return services;
}
```

5. **Register in Program.cs:**
```csharp
// Program.cs - ONE LINE!
builder.Services.AddFeatureHandlers();
```

**Pros:**
- ✅ Auto-registration (can't forget)
- ✅ Pipeline behaviors (logging, validation, etc.)
- ✅ Result pattern (no exceptions)
- ✅ Scales to 100+ handlers

**Cons:**
- ⚠️ More complex
- ⚠️ Harder to debug
- ⚠️ Steeper learning curve

---

## Dependency Injection Guidelines

### Registration Patterns

#### 1. Handler Registration

**Simple approach (AMS-style):**
```csharp
// Program.cs
builder.Services.AddQueryHandler<GetAgenciesQuery, IEnumerable<Agency>, GetAgenciesHandler>();
builder.Services.AddCommandHandler<CreateAgencyCommand, Agency, CreateAgencyHandler>();
```

**Advanced approach (Insured Portal-style):**
```csharp
// Program.cs
builder.Services.AddFeatureHandlers(); // Auto-discovers all handlers
```

#### 2. Service Registration

```csharp
// Infrastructure services
builder.Services.AddScoped<IAgencyRepository, AgencyRepository>();
builder.Services.AddScoped<IEmailService, EmailService>();
builder.Services.AddSingleton<IApplicationConfiguration, ApplicationConfiguration>();

// HttpClient factory pattern
builder.Services.AddHttpClient<IExternalService, ExternalService>(client =>
{
    client.BaseAddress = new Uri(configuration["ExternalService:Url"]);
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

#### 3. Database Context Registration

```csharp
// SQL Server with connection resiliency
builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(
        connectionString,
        sqlOptions =>
        {
            sqlOptions.EnableRetryOnFailure(
                maxRetryCount: 3,
                maxRetryDelay: TimeSpan.FromSeconds(5),
                errorNumbersToAdd: null);
        }));
```

---

## Error Handling Standards

### Option A: Exception-Based (Simple Projects)

**Use for:**
- Small APIs
- Internal services
- Prototype/MVP projects

**Pattern:**
```csharp
public class CreateAgencyHandler : ICommandHandler<CreateAgencyCommand, Agency>
{
    public async Task<Agency> HandleAsync(CreateAgencyCommand command, CancellationToken ct)
    {
        // Validate
        if (string.IsNullOrEmpty(command.Agency.Name))
            throw new ValidationException("Agency name is required");
        
        // Business logic
        var agency = await _repository.CreateAsync(command.Agency, ct);
        
        return agency;
    }
}

// Controller
[HttpPost]
public async Task<ActionResult<Agency>> Create(AgencyForCreation agency)
{
    try
    {
        var result = await _handler.HandleAsync(new CreateAgencyCommand(agency));
        return CreatedAtAction(nameof(Get), new { id = result.Id }, result);
    }
    catch (ValidationException ex)
    {
        return BadRequest(ex.Message);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Error creating agency");
        return StatusCode(500, "An error occurred");
    }
}
```

---

### Option B: Result Pattern (Large Projects) ✅ RECOMMENDED

**Use for:**
- Production APIs
- Customer-facing services
- Long-term projects

**Pattern:**
```csharp
public class CreateAgencyHandler : ICommandHandler<CreateAgencyCommand, Agency>
{
    public async Task<Result<Agency>> HandleAsync(CreateAgencyCommand command, CancellationToken ct)
    {
        // Validate
        if (string.IsNullOrEmpty(command.Agency.Name))
            return Result<Agency>.BadRequest("Agency name is required");
        
        // Check for duplicates
        if (await _repository.ExistsAsync(command.Agency.Name, ct))
            return Result<Agency>.BadRequest("Agency already exists");
        
        // Business logic
        try
        {
            var agency = await _repository.CreateAsync(command.Agency, ct);
            return Result<Agency>.Ok(agency);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Error creating agency");
            return Result<Agency>.Error("Failed to create agency");
        }
    }
}

// Controller extension
public static class ControllerExtensions
{
    public static ActionResult<T> ToActionResult<T>(this ControllerBase controller, Result<T> result)
    {
        return result.Status switch
        {
            ResultStatus.Ok => controller.Ok(result.Value),
            ResultStatus.NotFound => controller.NotFound(result.ErrorMessage),
            ResultStatus.BadRequest => controller.BadRequest(result.ErrorMessage),
            ResultStatus.Unauthorized => controller.Unauthorized(result.ErrorMessage),
            ResultStatus.Error => controller.StatusCode(500, result.ErrorMessage),
            _ => controller.StatusCode(500, "Unknown error")
        };
    }
}

// Controller
[HttpPost]
public async Task<ActionResult<Agency>> Create(AgencyForCreation agency)
{
    var result = await _handler.HandleAsync(new CreateAgencyCommand(agency));
    return this.ToActionResult(result);
}
```

**Benefits:**
- ✅ No exception-driven control flow
- ✅ Explicit error states
- ✅ Better performance (no stack unwinding)
- ✅ Easier to test
- ✅ Clear success/failure paths

---

## Logging Standards

### Overview

DJB applications **MUST** use the corporate NLog integration to send logs to DJB's centralized Error Log and Message Log SOAP services. This ensures consistent logging across all applications and centralized observability.

### Mandatory Requirements

- ✅ **Corporate NLog Integration** - Use `DJB.Helpers.Logging` for all production applications
- ✅ **Error Log** - All Error-level logs MUST be sent to corporate Error Log
- ✅ **Message Log** - Info/Warning logs MUST be sent to corporate Message Log
- ✅ **Early Configuration** - Logging MUST be configured early in Program.cs (before other services)
- ✅ **Standard ILogger** - Use Microsoft.Extensions.Logging.ILogger (not NLog directly)

### Implementation

#### 1. Add Project Reference

Add `DJB.Helpers.Logging` to your project:

```xml
<ItemGroup>
  <ProjectReference Include="..\DJB.Helpers.Logging\DJB.Helpers.Logging.csproj" />
</ItemGroup>
```

**Note:** If `DJB.Helpers.Logging` is not in your solution, copy it from the PortalCore repository:
- Source: `C:\source\repos\PortalCore\DJB.AgentPortal\DJB.Helpers.Logging`
- Destination: Your solution's `src` or root folder

#### 2. Configure appsettings.json

Add the `Logging:Corporate` section to all environment configuration files:

```json
{
  "Logging": {
    "Corporate": {
      "ServiceUrl": "https://DJBservicesdev.ga.afginc.com/WebServices/restErrorLog/Service1.asmx",
      "LogLevel": "Warning",
      "ApplicationName": "DJB.YourApplication"
    }
  }
}
```

**Configuration by Environment:**

| Environment | ServiceUrl |
|-------------|------------|
| **DEV/QA** | `https://DJBservicesdev.ga.afginc.com/WebServices/restErrorLog/Service1.asmx` |
| **PROD** | `https://DJBservices.ga.afginc.com/WebServices/restErrorLog/Service1.asmx` |

**LogLevel Options:**
- `"Information"` - Send Info, Warning, and Error logs to Message Log/Error Log
- `"Warning"` - Send Warning and Error logs (recommended for production)
- `"None"` - Send only Error logs to Error Log

**ApplicationName:**
- Use pattern: `DJB.{Domain}.{Component}`
- Examples: `DJB.AgentPortal.Api`, `DJB.BillingCenter`, `DJB.Ams.Api`

#### 3. Register in Program.cs

**CRITICAL:** Register logging **FIRST** in Program.cs, before any other services:

```csharp
using DJB.Helpers.Logging.DependencyInjection;
using Microsoft.Extensions.Logging;

var builder = WebApplication.CreateBuilder(args);

// ✅ FIRST: Configure corporate logging so startup failures are captured
builder.Services.AddCorporateLogging(builder.Configuration);

// ... other service registrations ...

var app = builder.Build();
app.Run();
```

**Why First?**
- Captures startup failures and configuration errors
- Ensures logging is available for all subsequent service registrations
- Prevents silent failures during application initialization

#### 4. Use Standard ILogger

**In Controllers/Handlers:**

```csharp
using Microsoft.Extensions.Logging;

public class CreateAgencyHandler : ICommandHandler<CreateAgencyCommand, Agency>
{
    private readonly ILogger<CreateAgencyHandler> _logger;
    
    public CreateAgencyHandler(ILogger<CreateAgencyHandler> logger)
    {
        _logger = logger;
    }
    
    public async Task<Result<Agency>> HandleAsync(CreateAgencyCommand command, CancellationToken ct)
    {
        _logger.LogInformation("Creating agency: {AgencyName}", command.Agency.Name);
        
        try
        {
            var agency = await _repository.CreateAsync(command.Agency, ct);
            _logger.LogInformation("Agency created successfully: {AgencyId}", agency.Id);
            return Result<Agency>.Ok(agency);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Failed to create agency: {AgencyName}", command.Agency.Name);
            return Result<Agency>.Error("Failed to create agency");
        }
    }
}
```

### Log Levels and Routing

| Level | Destination | When to Use |
|-------|-------------|-------------|
| **Error** | **Error Log** (always) | Exceptions, failures, data corruption |
| **Warning** | **Message Log** (if LogLevel ≤ Warning) | Degraded performance, retries, potential issues |
| **Information** | **Message Log** (if LogLevel = Information) | Key business events, audit trail |
| **Debug/Trace** | Suppressed (not sent) | Development debugging only |

### Log Message Best Practices

**✅ DO:**
```csharp
// Structured logging with parameters
_logger.LogInformation("User {UserId} created agency {AgencyId}", userId, agencyId);

// Include context in errors
_logger.LogError(ex, "Failed to process payment for policy {PolicyId}", policyId);

// Log business events
_logger.LogInformation("Policy {PolicyId} bound successfully", policyId);
```

**❌ DON'T:**
```csharp
// String interpolation (prevents structured logging)
_logger.LogInformation($"User {userId} created agency {agencyId}");

// Logging sensitive data
_logger.LogInformation("User password: {Password}", password);  // ❌ NEVER

// Excessive logging in tight loops
for (int i = 0; i < 10000; i++)
{
    _logger.LogInformation("Processing item {Index}", i);  // ❌ Too noisy
}
```

### Noise Suppression

The corporate logging integration automatically suppresses noisy Microsoft and HTTP client logs at Debug/Info/Trace levels:

- `Microsoft.*` - ASP.NET Core infrastructure logs
- `System.Net.Http.*` - HTTP client internal logs

Only Warning and Error logs from these namespaces reach the corporate logs.

### Error Log Parameters

When an error is logged, the following information is automatically captured and sent to the Error Log:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `systemSource` | `ApplicationName` config | Application identifier |
| `message` | Log message + exception | Formatted error message with inner exceptions |
| `stackTrace` | Exception | Full stack trace |
| `providerName` | Logger name | Class/namespace that logged the error |
| `serverName` | Machine name | Server hostname |
| `userId` | ASP.NET identity | Currently authenticated user |
| `eventType` | Log level | Error level |
| `errorGroup` | Context-specific | Allows grouping related errors |

### Message Log Parameters

When an info/warning is logged:

| Parameter | Source | Description |
|-----------|--------|-------------|
| `application` | `ApplicationName` config | Application identifier |
| `source` | Logger name | Class/namespace that logged the message |
| `message` | Formatted message | Log level + message text |
| `errorGroup` | Context-specific | Allows grouping related messages |

### Testing Logging Configuration

**Verify logging is working:**

```csharp
// In Program.cs or a test endpoint
var logger = app.Services.GetRequiredService<ILogger<Program>>();

// Test warning (should appear in Message Log)
logger.LogWarning("Test warning - logging configuration successful");

// Test error (should appear in Error Log)
logger.LogError(new Exception("Test exception"), "Test error - logging configuration successful");
```

**Check corporate logs:**
1. DEV Error Log: Query `ErrorLog` table for your `ApplicationName`
2. DEV Message Log: Query `MessageLog` table for your `ApplicationName`

### Local Development

For local development, you have two options:

**Option 1: Enable Corporate Logging (Recommended)**
```json
{
  "Logging": {
    "Corporate": {
      "ServiceUrl": "https://DJBservicesdev.ga.afginc.com/WebServices/restErrorLog/Service1.asmx",
      "LogLevel": "Warning",
      "ApplicationName": "DJB.YourApplication.LocalDev"
    }
  }
}
```

**Option 2: Disable Corporate Logging**
```json
{
  "Logging": {
    "Corporate": {
      "ServiceUrl": "",  // Empty = disabled
      "ApplicationName": "DJB.YourApplication.LocalDev"
    }
  }
}
```

When `ServiceUrl` is empty, corporate logging is silently disabled and standard console logging is used.

### Troubleshooting

**Logs not appearing in corporate system?**
1. ✅ Verify `ApplicationName` is set in configuration
2. ✅ Verify `ServiceUrl` is correct for your environment
3. ✅ Check network connectivity to the SOAP endpoint
4. ✅ Ensure logging is registered **before** other services in Program.cs
5. ✅ Verify log level (Info logs require `LogLevel: "Information"`)

**Application not starting?**
1. ✅ Check for configuration errors in `appsettings.json`
2. ✅ Verify `DJB.Helpers.Logging` project reference
3. ✅ Ensure NLog packages are installed (see Directory.Packages.props)

---

## Secrets Management

### Overview

DJB applications **MUST** use the corporate Secret Manager (Conjur/CyberArk) for all sensitive credentials. This ensures secrets are never committed to source control and are managed centrally.

### Mandatory Requirements

- ✅ **Conjur Integration** - Use `DJB.Helpers.Secrets` for vault access
- ✅ **Zero Secret Scrubbing** - Bootstrap secret is automatically removed from appsettings after first run
- ✅ **No Hardcoded Secrets** - NEVER commit credentials to source control
- ✅ **Eager Deployment** - Secrets are available immediately at startup
- ❌ **Never Use User Secrets for Production** - Microsoft User Secrets are development-only

### Implementation

#### 1. Add Project Reference

Add `DJB.Helpers.Secrets` to your project:

```xml
<ItemGroup>
  <ProjectReference Include="..\DJB.Helpers.Secrets\DJB.Helpers.Secrets.csproj" />
</ItemGroup>
```

Add the corporate Secret Manager package to `Directory.Packages.props`:

```xml
<PackageVersion Include="com.gaig.appsec.secretmanager" Version="1.3.2" />
```

**Note:** If `DJB.Helpers.Secrets` is not in your solution, copy it from the PortalCore repository:
- Source: `C:\source\repos\PortalCore\DJB.AgentPortal\DJB.Helpers.Secrets`
- Destination: Your solution's `src` or root folder

#### 2. Configure ConjurConfig

Add the `ConjurConfig` section to `appsettings.json`:

```json
{
  "ConjurConfig": {
    "appName": "YourApplicationName",
    "environment": "localdev",
    "vault": "conjur",
    "conjur": {
      "hostname": "conjurapi.gaig.com",
      "name": "ca_sync/LOB1/your-application-path",
      "providerVault": "GAPROD",
      "account": "host/conjur_0_your-account",
      "password": {
        "location": "file",
        "value": ""
      },
      "cachePolicy": "cache_first"
    },
    "storeConfig": {
      "secrets": [
        {
          "name": "DatabaseConnection",
          "location": "vault",
          "vault": "YourAppDbConnection",
          "type": "string"
        }
      ]
    }
  }
}
```

**Configuration Properties:**

| Property | Required | Description | Example |
|----------|----------|-------------|---------|
| `appName` | ✅ Yes | Application name registered with vault | `"DJB.AgentPortal"` |
| `environment` | ✅ Yes | Deployment environment | `"localdev"`, `"DEV"`, `"QA"`, `"PROD"` |
| `conjur.hostname` | ✅ Yes | Conjur API endpoint | `"conjurapi.gaig.com"` |
| `conjur.name` | ✅ Yes | Conjur policy path | `"ca_sync/LOB1/agent-portals"` |
| `conjur.account` | ✅ Yes | Conjur host identity | `"host/conjur_0_agent-portals"` |
| `conjur.password.value` | First run only | Bootstrap ("zero") secret | Obtained from CyberArk |
| `storeConfig.secrets` | ✅ Yes | Array of secrets to retrieve | See below |

**Secret Configuration:**

```json
{
  "name": "LocalSecretName",      // Name used in code with GetSecret()
  "location": "vault",             // Always "vault"
  "vault": "VaultVariableName",    // Variable name in Conjur
  "type": "string"                 // Usually "string"
}
```

#### 3. First Run: Add Zero Secret

**Obtain the zero secret:**
1. Log into CyberArk
2. Search for your application's Conjur account
3. For local development, use the **sandbox account** (ends with `_s`)
4. Copy the bootstrap secret

**Add to appsettings.json (first run only):**

```json
{
  "ConjurConfig": {
    "conjur": {
      "password": {
        "value": "your-bootstrap-secret-here"
      }
    }
  }
}
```

**After first run:**
- Zero secret is automatically scrubbed from `appsettings.json`
- `Secrets/config.json` is created (encrypted by Secret Manager)
- Subsequent runs use the encrypted config (no zero secret needed)

#### 4. Register in Program.cs

Register Secret Manager **AFTER** logging but **BEFORE** database/services that need secrets:

```csharp
using Microsoft.Extensions.DependencyInjection;

var builder = WebApplication.CreateBuilder(args);

// Corporate logging (first)
builder.Services.AddCorporateLogging(builder.Configuration);

// Corporate secrets (second)
builder.Services.AddCorporateSecrets(builder.Configuration, builder.Environment);

// Database connection using secret from vault
var secretManager = builder.Services.BuildServiceProvider().GetRequiredService<ISecretManager>();
var connectionString = secretManager.GetSecret("DatabaseConnection");

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(connectionString));

// ... other services ...

var app = builder.Build();
app.Run();
```

#### 5. Use ISecretManager

**In services/configuration:**

```csharp
using com.gaig.appsec.secretmanager;

public class ExternalApiClient
{
    private readonly HttpClient _httpClient;
    private readonly string _apiKey;

    public ExternalApiClient(HttpClient httpClient, ISecretManager secrets)
    {
        _httpClient = httpClient;
        _apiKey = secrets.GetSecret("ExternalApiKey");
    }

    public async Task<string> CallApiAsync()
    {
        _httpClient.DefaultRequestHeaders.Add("X-API-Key", _apiKey);
        // ... make request ...
    }
}
```

**In OAuth token providers:**

```csharp
public class OAuthTokenProvider
{
    private readonly ISecretManager _secrets;

    public OAuthTokenProvider(ISecretManager secrets)
    {
        _secrets = secrets;
    }

    public async Task<string> GetTokenAsync()
    {
        var clientId = _secrets.GetSecret("OAuthClientId");
        var clientSecret = _secrets.GetSecret("OAuthClientSecret");
        
        // Use credentials to acquire token...
    }
}
```

### Environment-Specific Configuration

**Local Development (Sandbox):**

```json
{
  "environment": "localdev",
  "conjur": {
    "account": "host/conjur_0_myapp_s"  // Note: '_s' suffix for sandbox
  }
}
```

**Production:**

```json
{
  "environment": "PROD",
  "conjur": {
    "account": "host/conjur_0_myapp"  // No '_s' suffix
  }
}
```

### Common Secret Patterns

**Database Connection String:**

```json
{
  "storeConfig": {
    "secrets": [
      {
        "name": "DatabaseConnection",
        "location": "vault",
        "vault": "MyAppDbConnectionString",
        "type": "string"
      }
    ]
  }
}
```

**OAuth Client Credentials:**

```json
{
  "storeConfig": {
    "secrets": [
      {
        "name": "OAuthClientId",
        "location": "vault",
        "vault": "MyAppOAuthClientId",
        "type": "string"
      },
      {
        "name": "OAuthClientSecret",
        "location": "vault",
        "vault": "MyAppOAuthClientSecret",
        "type": "string"
      }
    ]
  }
}
```

**API Keys:**

```json
{
  "storeConfig": {
    "secrets": [
      {
        "name": "StripeApiKey",
        "location": "vault",
        "vault": "MyAppStripeKey",
        "type": "string"
      }
    ]
  }
}
```

### Best Practices

**✅ DO:**
- Use descriptive, application-specific secret names (`MyAppDbPassword`, not `DbPassword`)
- Keep one secret per vault variable (atomic credentials)
- Rotate zero secrets according to DJB policies
- Test with sandbox accounts (`_s` suffix) in development
- Document required secrets for deployment

**❌ DON'T:**
- Commit zero secret to source control
- Use same credentials across environments
- Share vault accounts between applications
- Hardcode fallback credentials "just in case"
- Log secret values (even accidentally)

### Troubleshooting

**Missing Zero Secret Error:**
```
SecretManagerConfigurationException: Conjur error: the bootstrap ('zero') secret is missing.
```

**Fix:**
1. Delete `Secrets` and `Conjur` folders
2. Add zero secret to `appsettings.json` at `ConjurConfig:conjur:password:value`
3. Obtain from CyberArk (search for your Conjur account)
4. For local dev, use sandbox account (ends with `_s`)

**Secrets Not Found:**

**Check:**
1. Secret `name` in code matches `storeConfig.secrets[].name`
2. Secret `vault` variable exists in Conjur
3. Conjur account has read access to vault variable

**Connection Errors:**

**Verify:**
1. Network access to `conjurapi.gaig.com`
2. Conjur policy path is correct (`ConjurConfig:conjur:name`)
3. Host identity matches registration (`ConjurConfig:conjur:account`)

---

## Package Management

### Overview

DJB applications **MUST** use Central Package Management (CPM) to ensure consistent package versions across all projects in a solution. This prevents version conflicts and simplifies dependency updates.

### Mandatory Requirements

- ✅ **Directory.Packages.props** - Required for all multi-project solutions
- ✅ **Central Version Control** - Package versions defined once, used everywhere
- ✅ **No Version Attributes** - Individual projects MUST NOT specify version numbers
- ❌ **No packages.config** - Legacy NuGet format is forbidden

### Implementation

#### 1. Enable Central Package Management

Create `Directory.Packages.props` in your solution root:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <!-- Corporate packages -->
    <PackageVersion Include="com.gaig.appsec.secretmanager" Version="1.3.2" />
    
    <!-- Microsoft packages -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.0.6" />
    <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
    <PackageVersion Include="Microsoft.Extensions.Configuration" Version="10.0.9" />
    <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="10.0.9" />
    
    <!-- Third-party packages -->
    <PackageVersion Include="NLog" Version="5.4.0" />
    <PackageVersion Include="NLog.Extensions.Logging" Version="5.4.0" />
    
    <!-- Testing packages -->
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.4" />
    <PackageVersion Include="coverlet.collector" Version="6.0.4" />
  </ItemGroup>
</Project>
```

#### 2. Update Project Files

**Remove version numbers from all `.csproj` files:**

**❌ OLD (Individual project):**
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.0.6" />
  <PackageReference Include="NLog" Version="5.4.0" />
</ItemGroup>
```

**✅ NEW (Central management):**
```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" />
  <PackageReference Include="NLog" />
</ItemGroup>
```

### Package Organization

**Group packages logically in Directory.Packages.props:**

```xml
<ItemGroup>
  <!-- Corporate security & secrets -->
  <PackageVersion Include="com.gaig.appsec.secretmanager" Version="1.3.2" />
  
  <!-- ASP.NET Core -->
  <PackageVersion Include="Microsoft.AspNetCore.OpenApi" Version="10.0.10" />
  <PackageVersion Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.0" />
  
  <!-- Entity Framework Core -->
  <PackageVersion Include="Microsoft.EntityFrameworkCore.SqlServer" Version="10.0.6" />
  <PackageVersion Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.10" />
  
  <!-- Extensions -->
  <PackageVersion Include="Microsoft.Extensions.Configuration" Version="10.0.9" />
  <PackageVersion Include="Microsoft.Extensions.DependencyInjection" Version="10.0.9" />
  <PackageVersion Include="Microsoft.Extensions.Logging.Abstractions" Version="10.0.9" />
  
  <!-- Logging -->
  <PackageVersion Include="NLog" Version="5.4.0" />
  <PackageVersion Include="NLog.Extensions.Logging" Version="5.4.0" />
  
  <!-- Testing -->
  <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="17.14.1" />
  <PackageVersion Include="xunit" Version="2.9.3" />
  <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.4" />
  <PackageVersion Include="coverlet.collector" Version="6.0.4" />
</ItemGroup>
```

### Adding New Packages

**When adding a new package to any project:**

1. **First:** Add version to `Directory.Packages.props`
   ```xml
   <PackageVersion Include="Newtonsoft.Json" Version="13.0.3" />
   ```

2. **Then:** Reference without version in project
   ```xml
   <PackageReference Include="Newtonsoft.Json" />
   ```

### Updating Package Versions

**To update a package across the entire solution:**

1. Update version in `Directory.Packages.props` (one place)
2. Build solution to verify compatibility
3. All projects automatically use the new version

**Example:**
```xml
<!-- Before -->
<PackageVersion Include="NLog" Version="5.3.0" />

<!-- After -->
<PackageVersion Include="NLog" Version="5.4.0" />
```

### Benefits

**Why use Central Package Management:**

| Benefit | Description |
|---------|-------------|
| **Version Consistency** | All projects use same package versions |
| **Simplified Updates** | Update version once, apply everywhere |
| **No Conflicts** | Prevents transitive dependency version conflicts |
| **Easier Auditing** | One file to review for security/CVE checks |
| **Smaller Project Files** | Less clutter in individual `.csproj` files |
| **Better Tooling** | IDEs understand CPM and provide better IntelliSense |

### Best Practices

**✅ DO:**
- Keep `Directory.Packages.props` alphabetically sorted within groups
- Group packages by category (ASP.NET, EF Core, Testing, etc.)
- Document why specific versions are pinned (security, compatibility)
- Use same version for related packages (e.g., all `Microsoft.Extensions.*` packages)
- Commit `Directory.Packages.props` to source control

**❌ DON'T:**
- Mix central and individual versioning (choose one)
- Specify versions in individual projects when CPM is enabled
- Use different versions for tightly-coupled packages
- Forget to update all related packages together

### Migration from Individual Versions

**If you have an existing solution without CPM:**

1. Create `Directory.Packages.props` in solution root
2. Run this PowerShell script to extract all package versions:

```powershell
# Extract all package versions from solution
$packages = Get-ChildItem -Recurse -Filter "*.csproj" | 
    Select-String '<PackageReference Include="([^"]+)" Version="([^"]+)"' | 
    ForEach-Object {
        [PSCustomObject]@{
            Name = $_.Matches.Groups[1].Value
            Version = $_.Matches.Groups[2].Value
        }
    } | 
    Group-Object Name | 
    ForEach-Object {
        # Use highest version if conflicts exist
        $_.Group | Sort-Object Version -Descending | Select-Object -First 1
    }

# Output for Directory.Packages.props
$packages | ForEach-Object {
    "    <PackageVersion Include=`"$($_.Name)`" Version=`"$($_.Version)`" />"
}
```

3. Add extracted versions to `Directory.Packages.props`
4. Remove `Version` attributes from all `.csproj` files
5. Build and test

### Troubleshooting

**Build Error: "Version information is not allowed"**

```
error NU1008: Projects that use central package version management should not define the version on the PackageReference items but on the PackageVersion items
```

**Fix:** Remove `Version` attribute from `<PackageReference>` in `.csproj` file.

**Package Version Conflict:**

**Symptom:** Runtime errors like `FileLoadException` or `Method not found`

**Fix:**
1. Check `Directory.Packages.props` for inconsistent versions
2. Ensure related packages use compatible versions
3. Use `dotnet list package --vulnerable` to check for issues

---

## Testing Requirements

### Unit Testing Standards

**Requirements:**
- ✅ All handlers MUST have unit tests
- ✅ Minimum 80% code coverage for handlers
- ✅ Use AAA pattern (Arrange, Act, Assert)

**Example:**
```csharp
public class CreateAgencyHandlerTests
{
    [Fact]
    public async Task HandleAsync_ValidAgency_ReturnsCreatedAgency()
    {
        // Arrange
        var mockRepository = new Mock<IAgencyRepository>();
        var handler = new CreateAgencyHandler(mockRepository.Object);
        var command = new CreateAgencyCommand(new AgencyForCreation { Name = "Test" });
        
        mockRepository
            .Setup(r => r.CreateAsync(It.IsAny<Agency>(), It.IsAny<CancellationToken>()))
            .ReturnsAsync(new Agency { Id = 1, Name = "Test" });
        
        // Act
        var result = await handler.HandleAsync(command, CancellationToken.None);
        
        // Assert
        Assert.True(result.IsSuccess);
        Assert.Equal("Test", result.Value.Name);
        mockRepository.Verify(r => r.CreateAsync(It.IsAny<Agency>(), It.IsAny<CancellationToken>()), Times.Once);
    }
    
    [Fact]
    public async Task HandleAsync_EmptyName_ReturnsBadRequest()
    {
        // Arrange
        var handler = new CreateAgencyHandler(Mock.Of<IAgencyRepository>());
        var command = new CreateAgencyCommand(new AgencyForCreation { Name = "" });
        
        // Act
        var result = await handler.HandleAsync(command, CancellationToken.None);
        
        // Assert
        Assert.False(result.IsSuccess);
        Assert.Equal(ResultStatus.BadRequest, result.Status);
    }
}
```

### Integration Testing Standards

**Requirements:**
- ✅ All API endpoints MUST have integration tests
- ✅ Use WebApplicationFactory for testing
- ✅ Test happy path and error scenarios

**Example:**
```csharp
public class AgenciesControllerTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;
    
    public AgenciesControllerTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
    
    [Fact]
    public async Task GetAgencies_ReturnsOk()
    {
        // Act
        var response = await _client.GetAsync("/agencies");
        
        // Assert
        response.EnsureSuccessStatusCode();
        var agencies = await response.Content.ReadFromJsonAsync<List<Agency>>();
        Assert.NotNull(agencies);
    }
}
```

---

## Performance Standards

### 1. Database Query Optimization

**Requirements:**
- ✅ Use AsNoTracking() for read-only queries
- ✅ Use pagination for large result sets
- ✅ Include necessary related entities with Include()
- ❌ Avoid N+1 query problems

**Example:**
```csharp
public async Task<IEnumerable<Agency>> GetAgenciesAsync(AgencyParameters parameters, CancellationToken ct)
{
    return await _context.Agencies
        .AsNoTracking()                           // ✅ Read-only optimization
        .Include(a => a.Contacts)                 // ✅ Eager loading
        .Where(a => a.IsActive)
        .Skip((parameters.PageNumber - 1) * parameters.PageSize)  // ✅ Pagination
        .Take(parameters.PageSize)
        .ToListAsync(ct);
}
```

### 2. API Response Caching

**Requirements:**
- ✅ Cache GET endpoints that don't change frequently
- ✅ Use appropriate cache durations
- ✅ Implement cache invalidation strategies

**Example:**
```csharp
[HttpGet("{id}")]
[ResponseCache(Duration = 300, Location = ResponseCacheLocation.Any)] // 5 minutes
public async Task<ActionResult<Agency>> Get(int id)
{
    var result = await _handler.HandleAsync(new GetAgencyQuery(id));
    return this.ToActionResult(result);
}
```

### 3. Async/Await Best Practices

**Rules:**
- ✅ **DO** use async/await for I/O operations
- ✅ **DO** accept CancellationToken
- ✅ **DO** use ConfigureAwait(false) in library code
- ❌ **DON'T** use Task.Result or Task.Wait()
- ❌ **DON'T** use async void (except event handlers)

---

## Security Standards

### 1. Authentication & Authorization

**Requirements:**
- ✅ All API endpoints MUST require authentication (except health checks)
- ✅ Use [Authorize] attribute on controllers
- ✅ Use [AllowAnonymous] explicitly for public endpoints

**Example:**
```csharp
[Authorize] // ✅ Required authentication
[Route("api/[controller]")]
public class AgenciesController : ControllerBase
{
    [AllowAnonymous] // ✅ Explicitly public
    [HttpGet("health")]
    public IActionResult Health() => Ok("Healthy");
    
    [HttpGet]
    public async Task<ActionResult<IEnumerable<Agency>>> Get()
    {
        // Requires authentication
    }
}
```

### 2. Input Validation

**Requirements:**
- ✅ Validate all user input
- ✅ Use Data Annotations on DTOs
- ✅ Sanitize HTML content
- ✅ Use parameterized queries (never string concatenation)

**Example:**
```csharp
public class AgencyForCreation
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; }
    
    [Required]
    [EmailAddress]
    public string Email { get; set; }
    
    [Phone]
    public string Phone { get; set; }
}
```

### 3. Secrets Management

**Requirements:**
- ✅ Use Secret Manager for development
- ✅ Use Azure Key Vault or Conjur for production
- ❌ NEVER commit secrets to source control
- ❌ NEVER hardcode connection strings

**Example:**
```csharp
// Configuration
builder.Services.AddSingleton<ISecretManager>(sp =>
{
    var secretManager = new SecretManager(vaultPath);
    secretManager.Deploy();
    return secretManager;
});

// Usage
var connectionString = _secretManager.GetSecret("DatabaseConnection");
```

---

## Appendix: Reference Implementations

### AMS API (Simple Approach)

**Best for:**
- New projects
- Small teams (< 5 developers)
- Short timelines (< 6 months)
- Learning CQRS

**Repository:** `C:\source\repos\AMS\DJB.Ams.Api`

**Key Features:**
- ✅ Explicit handler registration
- ✅ Decorator pattern for logging
- ✅ Simple folder structure
- ✅ Easy to understand
- ✅ Minimal abstractions

**When to use:**
- Starting a new API
- Team unfamiliar with CQRS
- Need fast onboarding
- Prototype or MVP

---

### Insured Portal Service (Advanced Approach)

**Best for:**
- Large projects
- Large teams (5+ developers)
- Long-term maintenance
- Complex business logic

**Repository:** `C:\source\repos\InsuredPortal\DJB.Portal.Insured.Service`

**Key Features:**
- ✅ Auto-registration
- ✅ Pipeline behaviors (logging, validation)
- ✅ Result pattern
- ✅ Feature-per-folder organization
- ✅ Highly testable

**When to use:**
- Scaling existing API
- Need validation/logging/caching behaviors
- Want DRY code
- Long-term project

---

### Agent Portal (Advanced Approach + Corporate Infrastructure)

**Best for:**
- Enterprise applications
- Applications requiring corporate logging and secrets
- Long-term, mission-critical systems
- .NET 10+ modernization efforts

**Repository:** `C:\source\repos\PortalCore\DJB.AgentPortal`

**Key Features:**
- ✅ Corporate NLog integration (DJB.Helpers.Logging)
- ✅ Conjur/CyberArk secrets (DJB.Helpers.Secrets)
- ✅ Central Package Management (Directory.Packages.props)
- ✅ FastEndpoints (alternative to Controllers)
- ✅ Vertical slice architecture with pure use-case slicing
- ✅ Modular design (separate projects for Billing, Security, CMS)
- ✅ Architecture tests with NetArchTest
- ✅ Integration tests with WebApplicationFactory

**When to use:**
- Modernizing legacy .NET Framework applications
- Need corporate logging/secrets infrastructure out-of-the-box
- Building customer-facing portals or mission-critical APIs
- Want reusable helper libraries (Logging, Secrets) across multiple applications

**Reusable Components:**
- `DJB.Helpers.Logging` - Copy this to any project for corporate NLog integration
- `DJB.Helpers.Secrets` - Copy this to any project for Conjur secrets
- Both are **agnostic** and work in any .NET application

---

### What to Copy from Each Reference

**From AMS (Simple):**
- `AddQueryHandler` / `AddCommandHandler` extension methods
- Explicit registration pattern in `Program.cs`
- Simple folder structure (Features/{Domain}/Commands|Queries)

**From Insured Portal (Advanced):**
- `Result<T>` pattern and `ResultStatus` enum
- Auto-discovery registration via reflection
- Pipeline behaviors (Logging, Validation, Transaction)
- `IPipelineBehavior<TRequest, TResponse>` interface

**From Agent Portal (Infrastructure):**
- `DJB.Helpers.Logging` project (copy entire folder)
- `DJB.Helpers.Secrets` project (copy entire folder)
- `Directory.Packages.props` template
- Early logging configuration pattern (before other services)
- Secrets deployment pattern (after logging, before database)

---

## Migration Path

### From Traditional Layered Architecture to Vertical Slices

**Step 1: Create Features folder**
```
src/
├── Controllers/        ← OLD
├── Services/          ← OLD
├── Repositories/      ← OLD
└── Features/          ← NEW (start here)
    └── Agencies/
```

**Step 2: Move one feature at a time**
- Move controller, service, repository for ONE feature
- Create Commands and Queries
- Implement handlers
- Test thoroughly
- Repeat for next feature

**Step 3: Remove old layers**
- Once all features migrated, delete old folders
- Update documentation
- Train team on new structure

---

## Conclusion

These architecture instructions provide:

1. ✅ **Clear standards** for .NET applications
2. ✅ **Two approaches** (simple and advanced) for different needs
3. ✅ **Concrete examples** from real projects
4. ✅ **Migration path** from legacy patterns
5. ✅ **Testing requirements** for quality assurance
6. ✅ **Corporate logging integration** via DJB.Helpers.Logging
7. ✅ **Secrets management** via DJB.Helpers.Secrets with Conjur
8. ✅ **Central package management** for version consistency

**Choose the right approach based on your project needs, team size, and timeline.**

### Quick Start Checklist

When starting a new .NET application:

- [ ] ✅ Choose architecture approach (Simple or Advanced)
- [ ] ✅ Create `Directory.Packages.props` for central package management
- [ ] ✅ Add `DJB.Helpers.Logging` project reference
- [ ] ✅ Configure corporate logging in `appsettings.json` (Logging:Corporate section)
- [ ] ✅ Register corporate logging **FIRST** in `Program.cs`
- [ ] ✅ Add `DJB.Helpers.Secrets` project reference
- [ ] ✅ Configure Conjur in `appsettings.json` (ConjurConfig section)
- [ ] ✅ Obtain zero secret from CyberArk (sandbox for local dev)
- [ ] ✅ Register corporate secrets in `Program.cs`
- [ ] ✅ Use Vertical Slice Architecture (feature folders, not layers)
- [ ] ✅ Implement CQRS (Commands/ and Queries/ separation)
- [ ] ✅ Use Result<T> pattern for error handling (advanced projects)
- [ ] ✅ Write unit tests for all handlers (80% coverage minimum)
- [ ] ✅ Write integration tests for all endpoints
- [ ] ✅ Enable TreatWarningsAsErrors in .csproj
- [ ] ✅ Configure CI/CD pipeline
- [ ] ✅ Document API with Swagger/OpenAPI

**Questions?** Contact the Architecture Review Board.

---

**Document Control:**
- **Version:** 1.0
- **Status:** Approved
- **Next Review:** February 2027
- **Maintainer:** Architecture Team
