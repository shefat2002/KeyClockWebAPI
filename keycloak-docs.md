# Keycloak Authentication in ASP.NET Core 10 Web API

> **Complete guide** for integrating Keycloak as an Identity Provider with ASP.NET Core 10 Web APIs using OAuth 2.0 / OpenID Connect (OIDC).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Prerequisites](#2-prerequisites)
3. [Keycloak Setup](#3-keycloak-setup)
4. [Project Setup](#4-project-setup)
5. [Configuration](#5-configuration)
6. [Authentication & Authorization Middleware](#6-authentication--authorization-middleware)
7. [JWT Bearer Token Validation](#7-jwt-bearer-token-validation)
8. [Role-Based Authorization](#8-role-based-authorization)
9. [Policy-Based Authorization](#9-policy-based-authorization)
10. [Protecting API Endpoints](#10-protecting-api-endpoints)
11. [Token Introspection](#11-token-introspection)
12. [Refresh Token Handling](#12-refresh-token-handling)
13. [Multi-Tenant Support](#13-multi-tenant-support)
14. [Error Handling](#14-error-handling)
15. [Testing with Swagger / Scalar UI](#15-testing-with-swagger--scalar-ui)
16. [Security Best Practices](#16-security-best-practices)
17. [Troubleshooting](#17-troubleshooting)

---

## 1. Overview

### What Is Keycloak?

Keycloak is an open-source Identity and Access Management (IAM) solution that provides:

- **Single Sign-On (SSO)** across applications
- **OAuth 2.0 / OpenID Connect** protocol support
- **User federation** (LDAP, Active Directory)
- **Social login** (Google, GitHub, Facebook, etc.)
- **Fine-grained authorization** (roles, groups, policies)
- **Token management** (access tokens, refresh tokens, ID tokens)

### How It Works with ASP.NET Core 10

```
┌─────────────┐        (1) Login Request         ┌──────────────┐
│   Client    │ ───────────────────────────────► │   Keycloak   │
│  (Browser/  │ ◄─────────────────────────────── │   (IdP)      │
│   Mobile)   │        (2) JWT Access Token       └──────────────┘
└─────────────┘                                         │
       │                                                │ JWKS / OIDC Discovery
       │ (3) API Call + Bearer Token                    │
       ▼                                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                  ASP.NET Core 10 Web API                        │
│                                                                 │
│  Middleware validates JWT ──► Controller / Endpoint executes    │
└─────────────────────────────────────────────────────────────────┘
```

**Flow summary:**
1. The client authenticates against Keycloak and receives a signed JWT access token.
2. The client attaches that token to every API request as a `Bearer` header.
3. ASP.NET Core validates the token signature using Keycloak's public keys (JWKS endpoint) — **no database round-trip required**.

---

## 2. Prerequisites

| Requirement | Version / Notes |
|---|---|
| .NET SDK | 10.0 or later |
| Keycloak Server | 24+ (or Keycloak in Docker) |
| IDE | Visual Studio 2022+, VS Code, or Rider |
| Docker *(optional)* | For running Keycloak locally |

### Run Keycloak Locally via Docker

```bash
docker run -d \
  --name keycloak \
  -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  quay.io/keycloak/keycloak:latest \
  start-dev
```

Access the Admin Console at: `http://localhost:8080`

---

## 3. Keycloak Setup

### 3.1 Create a Realm

1. Log in to the Keycloak Admin Console (`http://localhost:8080`).
2. Click **Create Realm**.
3. Set the Realm Name to `my-app` and click **Create**.

> A **Realm** is an isolated namespace — it manages its own clients, users, and roles.

### 3.2 Create a Client

1. Navigate to **Clients** → **Create client**.
2. Fill in the details:

| Field | Value |
|---|---|
| Client Type | `OpenID Connect` |
| Client ID | `my-api-client` |
| Name | `My API Client` |

3. Click **Next**.
4. Under **Capability config**:

| Setting | Value |
|---|---|
| Client authentication | `On` *(confidential client)* |
| Authorization | `On` *(if using resource-based auth)* |
| Standard flow | `On` |
| Direct access grants | `On` *(for testing with password grant)* |

5. Click **Next** → **Save**.

### 3.3 Get the Client Secret

1. Go to the **Credentials** tab of your client.
2. Copy the **Client secret** — you'll need this for server-side flows.

### 3.4 Create Realm Roles

1. Go to **Realm roles** → **Create role**.
2. Create the following roles:

| Role Name | Description |
|---|---|
| `admin` | Full system access |
| `user` | Standard user access |
| `moderator` | Content moderation access |

### 3.5 Create a Test User

1. Go to **Users** → **Create new user**.
2. Set username to `testuser`, enable **Email verified**, click **Create**.
3. Go to the **Credentials** tab → set a password (uncheck **Temporary**).
4. Go to **Role mapping** → **Assign role** → assign `user` and/or `admin`.

### 3.6 Locate Important URLs

Your Keycloak realm exposes an OIDC Discovery endpoint:

```
http://localhost:8080/realms/my-app/.well-known/openid-configuration
```

Key URLs derived from this:

| URL | Purpose |
|---|---|
| `http://localhost:8080/realms/my-app` | Issuer / Authority |
| `/protocol/openid-connect/token` | Token endpoint |
| `/protocol/openid-connect/certs` | JWKS (public keys) |
| `/protocol/openid-connect/userinfo` | UserInfo endpoint |
| `/protocol/openid-connect/logout` | Logout endpoint |

---

## 4. Project Setup

### 4.1 Create the Web API Project

```bash
dotnet new webapi -n KeycloakAuthDemo --use-minimal-apis
cd KeycloakAuthDemo
```

### 4.2 Install NuGet Packages

```bash
# Core authentication packages (included in .NET 10 by default)
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

# Optional: for advanced JWT handling
dotnet add package System.IdentityModel.Tokens.Jwt

# Optional: Swagger / OpenAPI for testing
dotnet add package Scalar.AspNetCore
```

Your `.csproj` should include:

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.*" />
    <PackageReference Include="Scalar.AspNetCore" Version="*" />
  </ItemGroup>
</Project>
```

---

## 5. Configuration

### 5.1 appsettings.json

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "Keycloak": {
    "Authority": "http://localhost:8080/realms/my-app",
    "Audience": "my-api-client",
    "ClientId": "my-api-client",
    "ClientSecret": "YOUR_CLIENT_SECRET_HERE",
    "RequireHttpsMetadata": false,
    "ValidateIssuer": true,
    "ValidateAudience": true,
    "ValidateLifetime": true,
    "RolesClaimType": "realm_access",
    "NameClaimType": "preferred_username"
  },
  "AllowedHosts": "*"
}
```

> **Note:** Set `RequireHttpsMetadata` to `true` in production. Never commit secrets to source control — use environment variables, Azure Key Vault, or AWS Secrets Manager.

### 5.2 appsettings.Development.json

```json
{
  "Keycloak": {
    "Authority": "http://localhost:8080/realms/my-app",
    "RequireHttpsMetadata": false
  }
}
```

### 5.3 KeycloakOptions Model

Create a strongly-typed options class to bind configuration:

```csharp
// Options/KeycloakOptions.cs

namespace KeycloakAuthDemo.Options;

public sealed class KeycloakOptions
{
    public const string SectionName = "Keycloak";

    public string Authority { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public string ClientId { get; set; } = string.Empty;
    public string ClientSecret { get; set; } = string.Empty;
    public bool RequireHttpsMetadata { get; set; } = true;
    public bool ValidateIssuer { get; set; } = true;
    public bool ValidateAudience { get; set; } = true;
    public bool ValidateLifetime { get; set; } = true;
    public string RolesClaimType { get; set; } = "realm_access";
    public string NameClaimType { get; set; } = "preferred_username";
}
```

---

## 6. Authentication & Authorization Middleware

### 6.1 Program.cs — Full Setup

```csharp
// Program.cs

using System.Security.Claims;
using System.Text.Json;
using KeycloakAuthDemo.Extensions;
using KeycloakAuthDemo.Options;
using Scalar.AspNetCore;

var builder = WebApplication.CreateBuilder(args);

// ── Bind Keycloak options ───────────────────────────────────────────────────
builder.Services.Configure<KeycloakOptions>(
    builder.Configuration.GetSection(KeycloakOptions.SectionName));

// ── Add OpenAPI / Swagger ───────────────────────────────────────────────────
builder.Services.AddOpenApi();

// ── Add Keycloak Authentication ─────────────────────────────────────────────
builder.Services.AddKeycloakAuthentication(builder.Configuration);

// ── Add Authorization Policies ──────────────────────────────────────────────
builder.Services.AddKeycloakAuthorization();

// ── Add Controllers (if using controller-based API) ─────────────────────────
builder.Services.AddControllers();

var app = builder.Build();

// ── Middleware Pipeline ─────────────────────────────────────────────────────
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference(options =>
    {
        options.WithTitle("Keycloak Auth Demo API")
               .WithTheme(ScalarTheme.Moon)
               .WithOAuth2Authentication(oauth =>
               {
                   oauth.ClientId = "my-api-client";
               });
    });
}

app.UseHttpsRedirection();
app.UseAuthentication();  // ← Must come before UseAuthorization
app.UseAuthorization();

app.MapControllers();

// ── Minimal API endpoints ───────────────────────────────────────────────────
app.MapGet("/api/public", () => Results.Ok(new { message = "This is a public endpoint." }))
   .WithName("PublicEndpoint")
   .WithSummary("No authentication required");

app.MapGet("/api/protected", (ClaimsPrincipal user) =>
    Results.Ok(new
    {
        message = "You are authenticated!",
        username = user.FindFirst("preferred_username")?.Value,
        email = user.FindFirst(ClaimTypes.Email)?.Value,
        roles = user.FindAll("realm_access").Select(c => c.Value).ToList()
    }))
   .RequireAuthorization()
   .WithName("ProtectedEndpoint")
   .WithSummary("Requires authentication");

app.MapGet("/api/admin", () => Results.Ok(new { message = "Admin access granted." }))
   .RequireAuthorization("RequireAdminRole")
   .WithName("AdminEndpoint")
   .WithSummary("Requires admin role");

app.Run();
```

---

## 7. JWT Bearer Token Validation

### 7.1 Authentication Extension

```csharp
// Extensions/AuthenticationExtensions.cs

using System.Security.Claims;
using System.Text.Json;
using KeycloakAuthDemo.Options;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

namespace KeycloakAuthDemo.Extensions;

public static class AuthenticationExtensions
{
    public static IServiceCollection AddKeycloakAuthentication(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var keycloakOptions = configuration
            .GetSection(KeycloakOptions.SectionName)
            .Get<KeycloakOptions>()
            ?? throw new InvalidOperationException("Keycloak configuration is missing.");

        services
            .AddAuthentication(options =>
            {
                options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
                options.DefaultChallengeScheme    = JwtBearerDefaults.AuthenticationScheme;
            })
            .AddJwtBearer(options =>
            {
                // ── OIDC Discovery ────────────────────────────────────────
                // ASP.NET Core will automatically fetch the JWKS keys from:
                // {Authority}/.well-known/openid-configuration
                options.Authority            = keycloakOptions.Authority;
                options.RequireHttpsMetadata = keycloakOptions.RequireHttpsMetadata;
                options.Audience             = keycloakOptions.Audience;

                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer           = keycloakOptions.ValidateIssuer,
                    ValidIssuer              = keycloakOptions.Authority,

                    ValidateAudience         = keycloakOptions.ValidateAudience,
                    ValidAudience            = keycloakOptions.Audience,

                    ValidateLifetime         = keycloakOptions.ValidateLifetime,
                    ClockSkew                = TimeSpan.FromSeconds(30),

                    // Map Keycloak claim names to .NET claim types
                    NameClaimType            = keycloakOptions.NameClaimType,
                    RoleClaimType            = ClaimTypes.Role,

                    ValidateIssuerSigningKey  = true,
                    // Keys are fetched automatically from JWKS endpoint
                };

                // ── Custom Claims Transformation ──────────────────────────
                // Keycloak stores roles inside realm_access.roles (JSON object)
                // We need to extract and map them to standard ClaimTypes.Role
                options.Events = new JwtBearerEvents
                {
                    OnTokenValidated = context =>
                    {
                        MapKeycloakRolesToClaims(context);
                        return Task.CompletedTask;
                    },

                    OnAuthenticationFailed = context =>
                    {
                        var logger = context.HttpContext.RequestServices
                            .GetRequiredService<ILogger<Program>>();
                        logger.LogWarning("Authentication failed: {Error}",
                            context.Exception.Message);
                        return Task.CompletedTask;
                    },

                    OnChallenge = context =>
                    {
                        // Custom 401 response body
                        context.HandleResponse();
                        context.Response.StatusCode = StatusCodes.Status401Unauthorized;
                        context.Response.ContentType = "application/json";
                        var body = JsonSerializer.Serialize(new
                        {
                            error = "unauthorized",
                            message = "Authentication token is missing or invalid."
                        });
                        return context.Response.WriteAsync(body);
                    },

                    OnForbidden = context =>
                    {
                        context.Response.StatusCode = StatusCodes.Status403Forbidden;
                        context.Response.ContentType = "application/json";
                        var body = JsonSerializer.Serialize(new
                        {
                            error = "forbidden",
                            message = "You do not have permission to access this resource."
                        });
                        return context.Response.WriteAsync(body);
                    }
                };
            });

        return services;
    }

    /// <summary>
    /// Keycloak JWT structure for realm roles:
    /// {
    ///   "realm_access": {
    ///     "roles": ["admin", "user", "offline_access"]
    ///   }
    /// }
    /// This method extracts those roles and adds them as ClaimTypes.Role claims
    /// so that [Authorize(Roles = "admin")] works correctly.
    /// </summary>
    private static void MapKeycloakRolesToClaims(TokenValidatedContext context)
    {
        var claimsIdentity = context.Principal?.Identity as ClaimsIdentity;
        if (claimsIdentity is null) return;

        // Extract realm_access claim (it's a raw JSON object)
        var realmAccessClaim = claimsIdentity.FindFirst("realm_access");
        if (realmAccessClaim is null) return;

        try
        {
            using var doc = JsonDocument.Parse(realmAccessClaim.Value);
            if (doc.RootElement.TryGetProperty("roles", out var rolesElement))
            {
                foreach (var role in rolesElement.EnumerateArray())
                {
                    var roleName = role.GetString();
                    if (!string.IsNullOrWhiteSpace(roleName))
                    {
                        claimsIdentity.AddClaim(
                            new Claim(ClaimTypes.Role, roleName));
                    }
                }
            }
        }
        catch (JsonException ex)
        {
            var logger = context.HttpContext.RequestServices
                .GetRequiredService<ILogger<Program>>();
            logger.LogError(ex, "Failed to parse realm_access roles from Keycloak token.");
        }
    }
}
```

> **Why extract roles manually?**
> Keycloak embeds roles inside a nested JSON object (`realm_access.roles`). ASP.NET Core's default JWT handler doesn't know this structure, so we parse it manually in `OnTokenValidated` and add standard `ClaimTypes.Role` claims.

---

## 8. Role-Based Authorization

### 8.1 Authorization Extension

```csharp
// Extensions/AuthorizationExtensions.cs

using Microsoft.AspNetCore.Authorization;

namespace KeycloakAuthDemo.Extensions;

public static class AuthorizationExtensions
{
    public static IServiceCollection AddKeycloakAuthorization(
        this IServiceCollection services)
    {
        services.AddAuthorizationBuilder()
            // ── Basic role policies ─────────────────────────────────
            .AddPolicy("RequireAdminRole", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireRole("admin"))

            .AddPolicy("RequireUserRole", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireRole("user"))

            .AddPolicy("RequireModeratorRole", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireRole("moderator"))

            // ── Combined role policy ────────────────────────────────
            .AddPolicy("RequireAdminOrModerator", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireRole("admin", "moderator"))

            // ── Claim-based policy ──────────────────────────────────
            .AddPolicy("RequireVerifiedEmail", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireClaim("email_verified", "true"))

            // ── Custom composite policy ─────────────────────────────
            .AddPolicy("RequireActiveAdminUser", policy =>
                policy.RequireAuthenticatedUser()
                      .RequireRole("admin")
                      .RequireClaim("email_verified", "true"));

        return services;
    }
}
```

### 8.2 Using Role Authorization on Controllers

```csharp
// Controllers/ProductsController.cs

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KeycloakAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]  // All actions require authentication by default
public class ProductsController : ControllerBase
{
    [HttpGet]
    [AllowAnonymous]  // Override: anyone can list products
    public IActionResult GetAll()
    {
        return Ok(new[] { "Product A", "Product B", "Product C" });
    }

    [HttpGet("{id:int}")]
    // Inherits [Authorize] from the controller
    public IActionResult GetById(int id)
    {
        return Ok(new { Id = id, Name = $"Product {id}" });
    }

    [HttpPost]
    [Authorize(Roles = "admin,moderator")]  // Must have one of these roles
    public IActionResult Create([FromBody] CreateProductRequest request)
    {
        return CreatedAtAction(nameof(GetById), new { id = 1 },
            new { Id = 1, Name = request.Name });
    }

    [HttpDelete("{id:int}")]
    [Authorize(Policy = "RequireAdminRole")]  // Must satisfy the named policy
    public IActionResult Delete(int id)
    {
        return NoContent();
    }
}

public record CreateProductRequest(string Name, decimal Price);
```

---

## 9. Policy-Based Authorization

### 9.1 Custom Requirement and Handler

For complex authorization logic, implement `IAuthorizationRequirement` and `AuthorizationHandler<T>`.

```csharp
// Authorization/Requirements/MinimumAgeRequirement.cs

using Microsoft.AspNetCore.Authorization;

namespace KeycloakAuthDemo.Authorization.Requirements;

/// <summary>
/// Custom requirement: user's age claim must meet a minimum value.
/// </summary>
public sealed class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}
```

```csharp
// Authorization/Handlers/MinimumAgeHandler.cs

using System.Security.Claims;
using KeycloakAuthDemo.Authorization.Requirements;
using Microsoft.AspNetCore.Authorization;

namespace KeycloakAuthDemo.Authorization.Handlers;

public sealed class MinimumAgeHandler
    : AuthorizationHandler<MinimumAgeRequirement>
{
    private readonly ILogger<MinimumAgeHandler> _logger;

    public MinimumAgeHandler(ILogger<MinimumAgeHandler> logger)
        => _logger = logger;

    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var birthDateClaim = context.User.FindFirst("birthdate");

        if (birthDateClaim is null)
        {
            _logger.LogWarning("Birthdate claim is missing from token.");
            context.Fail();
            return Task.CompletedTask;
        }

        if (!DateOnly.TryParse(birthDateClaim.Value, out var birthDate))
        {
            _logger.LogWarning("Invalid birthdate format: {Value}", birthDateClaim.Value);
            context.Fail();
            return Task.CompletedTask;
        }

        var today = DateOnly.FromDateTime(DateTime.UtcNow);
        var age   = today.Year - birthDate.Year;

        if (birthDate > today.AddYears(-age)) age--;

        if (age >= requirement.MinimumAge)
        {
            context.Succeed(requirement);
        }
        else
        {
            _logger.LogWarning(
                "User age {Age} does not meet minimum requirement {Min}.",
                age, requirement.MinimumAge);
            context.Fail();
        }

        return Task.CompletedTask;
    }
}
```

### 9.2 Register the Custom Handler

```csharp
// In Program.cs or AuthorizationExtensions.cs

services.AddSingleton<IAuthorizationHandler, MinimumAgeHandler>();

services.AddAuthorizationBuilder()
    .AddPolicy("RequireAdult", policy =>
        policy.Requirements.Add(new MinimumAgeRequirement(18)));
```

### 9.3 Resource-Based Authorization

```csharp
// Authorization/Requirements/DocumentOwnerRequirement.cs

using Microsoft.AspNetCore.Authorization;

namespace KeycloakAuthDemo.Authorization.Requirements;

public sealed class DocumentOwnerRequirement : IAuthorizationRequirement { }
```

```csharp
// Authorization/Handlers/DocumentOwnerHandler.cs

using KeycloakAuthDemo.Authorization.Requirements;
using KeycloakAuthDemo.Models;
using Microsoft.AspNetCore.Authorization;

namespace KeycloakAuthDemo.Authorization.Handlers;

public sealed class DocumentOwnerHandler
    : AuthorizationHandler<DocumentOwnerRequirement, Document>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        DocumentOwnerRequirement requirement,
        Document resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;

        // Admins can access any document
        if (context.User.IsInRole("admin"))
        {
            context.Succeed(requirement);
            return Task.CompletedTask;
        }

        if (userId == resource.OwnerId)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

```csharp
// Controllers/DocumentsController.cs

using KeycloakAuthDemo.Authorization.Requirements;
using KeycloakAuthDemo.Models;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace KeycloakAuthDemo.Controllers;

[ApiController]
[Route("api/[controller]")]
[Authorize]
public class DocumentsController : ControllerBase
{
    private readonly IAuthorizationService _authorizationService;

    public DocumentsController(IAuthorizationService authorizationService)
        => _authorizationService = authorizationService;

    [HttpGet("{id:int}")]
    public async Task<IActionResult> Get(int id)
    {
        // Simulated document fetch
        var document = new Document { Id = id, OwnerId = "user-123", Title = "My Doc" };

        var result = await _authorizationService.AuthorizeAsync(
            User, document, new DocumentOwnerRequirement());

        if (!result.Succeeded)
            return Forbid();

        return Ok(document);
    }
}
```

---

## 10. Protecting API Endpoints

### 10.1 Attribute-Based (Controller)

```csharp
// Protect the entire controller
[Authorize]
public class SecureController : ControllerBase { }

// Protect with specific role
[Authorize(Roles = "admin")]
public IActionResult AdminOnly() => Ok();

// Protect with named policy
[Authorize(Policy = "RequireAdminRole")]
public IActionResult PolicyProtected() => Ok();

// Allow unauthenticated access to specific action
[AllowAnonymous]
public IActionResult PublicAction() => Ok();

// Multiple policies — ALL must be satisfied
[Authorize(Policy = "RequireUserRole")]
[Authorize(Policy = "RequireVerifiedEmail")]
public IActionResult MultiPolicy() => Ok();
```

### 10.2 Minimal API (Program.cs)

```csharp
// No authentication needed
app.MapGet("/public", () => "Public")
   .AllowAnonymous();

// Requires any valid token
app.MapGet("/protected", () => "Protected")
   .RequireAuthorization();

// Requires specific policy
app.MapGet("/admin", () => "Admin only")
   .RequireAuthorization("RequireAdminRole");

// Requires specific role
app.MapGet("/users-only", () => "Users only")
   .RequireAuthorization(policy => policy.RequireRole("user"));

// Group with shared authorization
var adminGroup = app.MapGroup("/api/admin")
    .RequireAuthorization("RequireAdminRole");

adminGroup.MapGet("/dashboard", () => "Admin Dashboard");
adminGroup.MapGet("/users",     () => "User Management");
adminGroup.MapPost("/settings", () => "Save Settings");
```

### 10.3 Accessing User Claims in Endpoints

```csharp
app.MapGet("/api/me", (ClaimsPrincipal user) =>
{
    return Results.Ok(new
    {
        Id       = user.FindFirst("sub")?.Value,
        Username = user.FindFirst("preferred_username")?.Value,
        Email    = user.FindFirst("email")?.Value,
        Roles    = user.FindAll(ClaimTypes.Role).Select(c => c.Value).ToArray(),
        IsAdmin  = user.IsInRole("admin")
    });
})
.RequireAuthorization()
.WithName("GetCurrentUser");
```

---

## 11. Token Introspection

For stateful token validation (e.g., checking if a token has been revoked before expiry), you can call Keycloak's introspection endpoint.

```csharp
// Services/KeycloakTokenIntrospectionService.cs

using System.Net.Http.Headers;
using System.Text.Json;
using KeycloakAuthDemo.Options;
using Microsoft.Extensions.Options;

namespace KeycloakAuthDemo.Services;

public sealed class KeycloakTokenIntrospectionService
{
    private readonly HttpClient _httpClient;
    private readonly KeycloakOptions _options;
    private readonly ILogger<KeycloakTokenIntrospectionService> _logger;

    public KeycloakTokenIntrospectionService(
        HttpClient httpClient,
        IOptions<KeycloakOptions> options,
        ILogger<KeycloakTokenIntrospectionService> logger)
    {
        _httpClient = httpClient;
        _options    = options.Value;
        _logger     = logger;
    }

    /// <summary>
    /// Validates the token against the Keycloak introspection endpoint.
    /// Useful for detecting revoked tokens before their natural expiry.
    /// </summary>
    public async Task<bool> IsTokenActiveAsync(string accessToken,
        CancellationToken cancellationToken = default)
    {
        var introspectionUrl =
            $"{_options.Authority}/protocol/openid-connect/token/introspect";

        var formData = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["token"]         = accessToken,
            ["client_id"]     = _options.ClientId,
            ["client_secret"] = _options.ClientSecret,
        });

        try
        {
            var response = await _httpClient.PostAsync(
                introspectionUrl, formData, cancellationToken);

            if (!response.IsSuccessStatusCode)
            {
                _logger.LogWarning("Introspection returned {Status}",
                    response.StatusCode);
                return false;
            }

            var content = await response.Content.ReadAsStringAsync(cancellationToken);
            using var doc = JsonDocument.Parse(content);

            return doc.RootElement.TryGetProperty("active", out var active)
                   && active.GetBoolean();
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Token introspection failed.");
            return false;
        }
    }
}
```

**Register the service:**

```csharp
// Program.cs
builder.Services.AddHttpClient<KeycloakTokenIntrospectionService>();
```

**Use in an endpoint:**

```csharp
app.MapGet("/api/introspect-check", async (
    HttpContext context,
    KeycloakTokenIntrospectionService introspection) =>
{
    var token = context.Request.Headers.Authorization
        .ToString().Replace("Bearer ", "", StringComparison.OrdinalIgnoreCase);

    var isActive = await introspection.IsTokenActiveAsync(token);

    return isActive
        ? Results.Ok(new { active = true })
        : Results.Unauthorized();
})
.RequireAuthorization();
```

> **Performance Note:** Token introspection adds a network round-trip on every request. Use it only for sensitive operations or implement a short-lived local cache.

---

## 12. Refresh Token Handling

Refresh tokens are typically used on the **client side** (SPA, mobile app) to obtain new access tokens. The API itself does not handle refresh tokens — it only validates access tokens.

The following example shows how a backend-for-frontend (BFF) or a service-to-service scenario might exchange a refresh token.

```csharp
// Services/KeycloakTokenService.cs

using System.Text.Json;
using KeycloakAuthDemo.Options;
using Microsoft.Extensions.Options;

namespace KeycloakAuthDemo.Services;

public sealed record TokenResponse(
    string AccessToken,
    string RefreshToken,
    int ExpiresIn,
    string TokenType);

public sealed class KeycloakTokenService
{
    private readonly HttpClient _httpClient;
    private readonly KeycloakOptions _options;

    public KeycloakTokenService(HttpClient httpClient, IOptions<KeycloakOptions> options)
    {
        _httpClient = httpClient;
        _options    = options.Value;
    }

    /// <summary>
    /// Exchanges credentials for tokens (Resource Owner Password Grant).
    /// Use this ONLY in trusted, internal service-to-service flows.
    /// For user-facing applications, always use the Authorization Code flow.
    /// </summary>
    public async Task<TokenResponse?> GetTokenAsync(
        string username, string password,
        CancellationToken cancellationToken = default)
    {
        var tokenUrl = $"{_options.Authority}/protocol/openid-connect/token";

        var form = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"]    = "password",
            ["client_id"]     = _options.ClientId,
            ["client_secret"] = _options.ClientSecret,
            ["username"]      = username,
            ["password"]      = password,
            ["scope"]         = "openid profile email",
        });

        var response = await _httpClient.PostAsync(tokenUrl, form, cancellationToken);
        if (!response.IsSuccessStatusCode) return null;

        var json = await response.Content.ReadAsStringAsync(cancellationToken);
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        return new TokenResponse(
            AccessToken:  root.GetProperty("access_token").GetString()!,
            RefreshToken: root.GetProperty("refresh_token").GetString()!,
            ExpiresIn:    root.GetProperty("expires_in").GetInt32(),
            TokenType:    root.GetProperty("token_type").GetString()!);
    }

    /// <summary>
    /// Refreshes an expired access token using a refresh token.
    /// </summary>
    public async Task<TokenResponse?> RefreshTokenAsync(
        string refreshToken,
        CancellationToken cancellationToken = default)
    {
        var tokenUrl = $"{_options.Authority}/protocol/openid-connect/token";

        var form = new FormUrlEncodedContent(new Dictionary<string, string>
        {
            ["grant_type"]    = "refresh_token",
            ["client_id"]     = _options.ClientId,
            ["client_secret"] = _options.ClientSecret,
            ["refresh_token"] = refreshToken,
        });

        var response = await _httpClient.PostAsync(tokenUrl, form, cancellationToken);
        if (!response.IsSuccessStatusCode) return null;

        var json = await response.Content.ReadAsStringAsync(cancellationToken);
        using var doc = JsonDocument.Parse(json);
        var root = doc.RootElement;

        return new TokenResponse(
            AccessToken:  root.GetProperty("access_token").GetString()!,
            RefreshToken: root.GetProperty("refresh_token").GetString()!,
            ExpiresIn:    root.GetProperty("expires_in").GetInt32(),
            TokenType:    root.GetProperty("token_type").GetString()!);
    }
}
```

---

## 13. Multi-Tenant Support

If your API must support multiple Keycloak realms (multi-tenancy), you can dynamically resolve the authority per request.

```csharp
// Services/KeycloakTenantResolver.cs

namespace KeycloakAuthDemo.Services;

public sealed class KeycloakTenantResolver
{
    private static readonly Dictionary<string, string> TenantAuthorities = new()
    {
        ["tenant-a"] = "http://localhost:8080/realms/tenant-a",
        ["tenant-b"] = "http://localhost:8080/realms/tenant-b",
    };

    public string? ResolveAuthority(string tenantId)
        => TenantAuthorities.GetValueOrDefault(tenantId);
}
```

```csharp
// In JwtBearerEvents.OnMessageReceived, resolve tenant from header/JWT
options.Events = new JwtBearerEvents
{
    OnMessageReceived = context =>
    {
        // Read tenant from custom header
        var tenantId = context.Request.Headers["X-Tenant-Id"].FirstOrDefault();
        if (!string.IsNullOrWhiteSpace(tenantId))
        {
            context.HttpContext.Items["TenantId"] = tenantId;
        }
        return Task.CompletedTask;
    },

    OnTokenValidated = context =>
    {
        // Verify the issuer matches the expected tenant authority
        var tenantId  = context.HttpContext.Items["TenantId"] as string;
        var resolver  = context.HttpContext.RequestServices
                               .GetRequiredService<KeycloakTenantResolver>();
        var authority = resolver.ResolveAuthority(tenantId ?? string.Empty);

        if (authority is null)
        {
            context.Fail("Unknown tenant.");
            return Task.CompletedTask;
        }

        var issuer = context.Principal?.FindFirst("iss")?.Value;
        if (issuer != authority)
            context.Fail($"Token issuer mismatch. Expected: {authority}");

        return Task.CompletedTask;
    }
};
```

---

## 14. Error Handling

### 14.1 Global Exception Handler

```csharp
// Middleware/GlobalExceptionHandler.cs

using System.Text.Json;
using Microsoft.AspNetCore.Diagnostics;
using Microsoft.AspNetCore.Mvc;

namespace KeycloakAuthDemo.Middleware;

public sealed class GlobalExceptionHandler : IExceptionHandler
{
    private readonly ILogger<GlobalExceptionHandler> _logger;

    public GlobalExceptionHandler(ILogger<GlobalExceptionHandler> logger)
        => _logger = logger;

    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        _logger.LogError(exception, "Unhandled exception: {Message}", exception.Message);

        var problem = new ProblemDetails
        {
            Status   = StatusCodes.Status500InternalServerError,
            Title    = "An unexpected error occurred.",
            Detail   = exception.Message,
            Instance = context.Request.Path
        };

        context.Response.StatusCode  = problem.Status!.Value;
        context.Response.ContentType = "application/problem+json";

        await context.Response.WriteAsJsonAsync(problem, cancellationToken);
        return true;
    }
}
```

```csharp
// Register in Program.cs
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

// In middleware pipeline
app.UseExceptionHandler();
```

### 14.2 Authorization Error Responses

The custom `OnChallenge` and `OnForbidden` handlers in Section 7 produce consistent JSON error responses:

```json
// 401 Unauthorized
{
  "error": "unauthorized",
  "message": "Authentication token is missing or invalid."
}

// 403 Forbidden
{
  "error": "forbidden",
  "message": "You do not have permission to access this resource."
}
```

---

## 15. Testing with Swagger / Scalar UI

### 15.1 Configure OpenAPI with OAuth2

```csharp
// Program.cs

builder.Services.AddOpenApi(options =>
{
    options.AddDocumentTransformer((document, context, cancellationToken) =>
    {
        document.Info = new()
        {
            Title       = "Keycloak Auth Demo API",
            Version     = "v1",
            Description = "ASP.NET Core 10 Web API secured with Keycloak"
        };

        // Add Bearer token security scheme
        document.Components ??= new();
        document.Components.SecuritySchemes = new Dictionary<string, OpenApiSecurityScheme>
        {
            ["Bearer"] = new OpenApiSecurityScheme
            {
                Type        = SecuritySchemeType.Http,
                Scheme      = "bearer",
                BearerFormat = "JWT",
                Description = "Enter your Keycloak access token"
            }
        };

        return Task.CompletedTask;
    });
});
```

### 15.2 Get a Token for Testing

Use `curl` or Postman to obtain a token from Keycloak:

```bash
curl -X POST http://localhost:8080/realms/my-app/protocol/openid-connect/token \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=password" \
  -d "client_id=my-api-client" \
  -d "client_secret=YOUR_CLIENT_SECRET" \
  -d "username=testuser" \
  -d "password=testpassword" \
  -d "scope=openid"
```

**Response:**
```json
{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6...",
  "expires_in": 300,
  "refresh_expires_in": 1800,
  "refresh_token": "eyJhbGciOiJIUzUxMiIsInR5...",
  "token_type": "Bearer",
  "not-before-policy": 0,
  "session_state": "abc123",
  "scope": "openid profile email"
}
```

Use the `access_token` value in the `Authorization: Bearer <token>` header of your API calls.

### 15.3 Test API Endpoint with Token

```bash
curl -X GET http://localhost:5000/api/protected \
  -H "Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCI6..."
```

---

## 16. Security Best Practices

| Practice | Description |
|---|---|
| **Use HTTPS** | Always use HTTPS in production. Set `RequireHttpsMetadata = true`. |
| **Short token lifetime** | Configure Keycloak access tokens to expire in 5 minutes or less. |
| **Validate all claims** | Always validate `iss`, `aud`, `exp`, and signature. |
| **Never log tokens** | Tokens are credentials — never log `Authorization` header values. |
| **Rotate client secrets** | Rotate Keycloak client secrets regularly. |
| **Use environment variables** | Store `ClientSecret` in environment variables, not `appsettings.json`. |
| **Least-privilege roles** | Assign the minimum roles necessary to each user. |
| **Token introspection for sensitive ops** | For high-risk operations, verify the token hasn't been revoked. |
| **CORS configuration** | Restrict `AllowedOrigins` to known client origins. |
| **Audit logging** | Log authentication events (login success/failure, role changes). |

### 16.1 Environment Variable Configuration

```bash
# Set secrets via environment variables (recommended for production)
export Keycloak__Authority="https://keycloak.example.com/realms/my-app"
export Keycloak__ClientSecret="super-secret-value"
```

ASP.NET Core automatically maps `__` (double underscore) to `:` in configuration keys, so `Keycloak__ClientSecret` binds to `Keycloak:ClientSecret`.

### 16.2 CORS Configuration

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowedOrigins", policy =>
    {
        policy.WithOrigins(
                "https://my-frontend.example.com",
                "http://localhost:3000")
              .AllowAnyHeader()
              .AllowAnyMethod()
              .AllowCredentials();
    });
});

// Middleware pipeline
app.UseCors("AllowedOrigins");
```

---

## 17. Troubleshooting

### Common Errors and Fixes

| Error | Cause | Solution |
|---|---|---|
| `401 Unauthorized` | Token missing or malformed | Check `Authorization: Bearer <token>` header format |
| `401 Unauthorized` | Token expired | Request a new token using refresh token flow |
| `401 Unauthorized` | Wrong `Authority` URL | Verify `Keycloak:Authority` in config matches realm URL exactly |
| `401 Unauthorized` | `ValidateAudience` mismatch | Ensure `Audience` matches the client ID or `aud` claim in token |
| `403 Forbidden` | Role not mapped | Check `OnTokenValidated` is correctly mapping `realm_access.roles` |
| `403 Forbidden` | Wrong role name | Role names are case-sensitive in both Keycloak and `[Authorize(Roles)]` |
| JWKS fetch fails | Keycloak not reachable | Check network connectivity and `Authority` URL |
| `SSL/TLS` error | HTTPS mismatch in dev | Set `RequireHttpsMetadata = false` in development only |

### 17.1 Inspect a JWT Token

Use `jwt.io` or decode manually to inspect the token claims:

```bash
# Decode JWT payload (base64)
echo "eyJhbGci..." | cut -d '.' -f 2 | base64 --decode 2>/dev/null | python3 -m json.tool
```

**Expected Keycloak JWT payload:**

```json
{
  "exp": 1700000000,
  "iat": 1699999700,
  "iss": "http://localhost:8080/realms/my-app",
  "aud": "my-api-client",
  "sub": "user-uuid-here",
  "typ": "Bearer",
  "preferred_username": "testuser",
  "email": "testuser@example.com",
  "email_verified": true,
  "realm_access": {
    "roles": [
      "admin",
      "user",
      "offline_access",
      "uma_authorization"
    ]
  },
  "scope": "openid profile email"
}
```

### 17.2 Enable Detailed Auth Logging

```json
// appsettings.Development.json
{
  "Logging": {
    "LogLevel": {
      "Microsoft.AspNetCore.Authentication": "Debug",
      "Microsoft.AspNetCore.Authorization":  "Debug"
    }
  }
}
```

### 17.3 Verify OIDC Discovery Endpoint

```bash
curl http://localhost:8080/realms/my-app/.well-known/openid-configuration | python3 -m json.tool
```

This shows all endpoints, supported algorithms, and claims — useful for verifying your configuration.

---

## Complete Project Structure

```
KeycloakAuthDemo/
├── Authorization/
│   ├── Handlers/
│   │   ├── DocumentOwnerHandler.cs
│   │   └── MinimumAgeHandler.cs
│   └── Requirements/
│       ├── DocumentOwnerRequirement.cs
│       └── MinimumAgeRequirement.cs
├── Controllers/
│   ├── DocumentsController.cs
│   └── ProductsController.cs
├── Extensions/
│   ├── AuthenticationExtensions.cs
│   └── AuthorizationExtensions.cs
├── Middleware/
│   └── GlobalExceptionHandler.cs
├── Models/
│   └── Document.cs
├── Options/
│   └── KeycloakOptions.cs
├── Services/
│   ├── KeycloakTokenIntrospectionService.cs
│   ├── KeycloakTokenService.cs
│   └── KeycloakTenantResolver.cs
├── appsettings.json
├── appsettings.Development.json
├── KeycloakAuthDemo.csproj
└── Program.cs
```

---

## Quick Reference

### Token Grant Types

| Grant Type | Use Case |
|---|---|
| `authorization_code` | Browser-based SPA / web app *(recommended)* |
| `client_credentials` | Service-to-service (no user context) |
| `password` | Trusted internal tools only *(deprecated in OAuth 2.1)* |
| `refresh_token` | Renew access token silently |

### Key Keycloak Endpoints

| Endpoint | Path |
|---|---|
| Token | `/realms/{realm}/protocol/openid-connect/token` |
| JWKS | `/realms/{realm}/protocol/openid-connect/certs` |
| UserInfo | `/realms/{realm}/protocol/openid-connect/userinfo` |
| Introspect | `/realms/{realm}/protocol/openid-connect/token/introspect` |
| Logout | `/realms/{realm}/protocol/openid-connect/logout` |
| Discovery | `/realms/{realm}/.well-known/openid-configuration` |

---

*Documentation generated for ASP.NET Core 10 + Keycloak 24+. Always refer to the [official Keycloak documentation](https://www.keycloak.org/documentation) and [Microsoft ASP.NET Core security docs](https://learn.microsoft.com/en-us/aspnet/core/security/) for the latest updates.*