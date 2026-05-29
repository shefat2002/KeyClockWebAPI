# Keycloak Multi-Tenant Social Authentication with ASP.NET Core Web API

## Complete Implementation Guide

This guide provides a comprehensive, production-ready approach to integrating Keycloak with an ASP.NET Core Web API for multi-tenant social authentication (Google, Facebook, GitHub, etc.).

---

## Table of Contents

1. [Architecture Overview](#1-architecture-overview)
2. [Prerequisites](#2-prerequisites)
3. [Keycloak Setup & Configuration](#3-keycloak-setup--configuration)
4. [Social Provider Configuration](#4-social-provider-configuration)
5. [ASP.NET Core Web API Setup](#5-aspnet-core-web-api-setup)
6. [Database Design & User Synchronization](#6-database-design--user-synchronization)
7. [Frontend Integration](#7-frontend-integration)
8. [Testing & Validation](#8-testing--validation)
9. [Production Deployment](#9-production-deployment)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. Architecture Overview

### System Flow

```
┌────────────────────────────────────────────────────────┐
│                      Client / Frontend                 │
│              (Angular, React, Vue, Mobile)             │
└──────────────────────────┬─────────────────────────────┘
                           │
       1. Auth Request     │  5. Send Bearer Token
       (Authorization Code)│     (JWT)
                           ▼
┌────────────────────────────────────────────────────────┐
│                 Keycloak Identity Server               │
│   ┌────────────────────────────────────────────────┐   │
│   │  User Federation & Social Login Mappers        │   │
│   └──────┬───────────────┬────────────────┬────────┘   │
└──────────┼───────────────┼────────────────┼────────────┘
           │               │                │
           ▼               ▼                ▼
     ┌───────────┐   ┌────────────┐   ┌────────────┐
     │ Google IdP│   │Facebook IdP│   │ GitHub IdP │
     └───────────┘   └────────────┘   └────────────┘
                           │
                           │ 6. Cryptographic Validation
                           │    via JWKS endpoint
                           ▼
┌────────────────────────────────────────────────────────┐
│                ASP.NET Core Web API                    │
│   ┌────────────────────────────────────────────────┐   │
│   │ Microsoft.AspNetCore.Authentication.JwtBearer   │   │
│   └────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────┘
```

### Authentication Lifecycle

1. **Login Request**: Frontend redirects user to Keycloak
2. **Social Provider Selection**: User chooses social auth method
3. **Provider Handshake**: Keycloak manages OAuth flow with selected provider
4. **Token Generation**: Keycloak issues JWT tokens upon successful authentication
5. **API Access**: Frontend presents JWT to ASP.NET Core API
6. **Token Validation**: API cryptographically verifies JWT signature locally
7. **User Provisioning**: API syncs user data to local database on first access

---

## 2. Prerequisites

### Development Environment

- **.NET 8.0 SDK** or later
- **Docker Desktop** (for Keycloak container)
- **SQL Server** or **PostgreSQL** (for application database)
- **Postman** or similar API testing tool
- **Frontend Framework** (React/Angular/Vue) for testing auth flow

### Developer Accounts Required

- **Google Cloud Console** (for Google OAuth)
- **Facebook Developer Portal** (for Facebook Login)
- **GitHub Developer Settings** (for GitHub OAuth)
- **Keycloak Admin** (will be created during setup)

### Network Requirements

- **localhost**: 8080 (Keycloak default)
- **localhost**: 5000-5100 (ASP.NET Core API)
- **SSL/TLS**: Required for production social providers

---

## 3. Keycloak Setup & Configuration

### 3.1 Deploy Keycloak using Docker

Create `docker-compose.yml`:

```yaml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.0
    container_name: keycloak-server
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin123
      KC_DB: postgres
      KC_DB_URL: jdbc:postgresql://keycloak-db:5432/keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloak_password
      KC_HOSTNAME: localhost
      KC_HOSTNAME_PORT: 8080
      KC_HTTP_ENABLED: "true"
    ports:
      - "8080:8080"
    command: start
    depends_on:
      - keycloak-db
  
  keycloak-db:
    image: postgres:16-alpine
    container_name: keycloak-db
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloak_password
    volumes:
      - keycloak_pgdata:/var/lib/postgresql/data

volumes:
  keycloak_pgdata:
```

**Start Keycloak:**

```bash
docker-compose up -d
docker-compose logs -f keycloak
```

Wait for "Keycloak 25.0.0 started" message.

### 3.2 Create Realm and Client

**Access Keycloak Admin Console:**
- Navigate to: http://localhost:8080/admin
- Login with: `admin` / `admin123`

**Create Application Realm:**

1. Click the realm dropdown (top-left corner)
2. Click "Create Realm"
3. Name: `CompanyApp`
4. Click "Create"

**Create Frontend Client:**

1. Navigate to: **Clients** → **Create Client**
2. **Client Type**: OpenID Connect
3. **Client ID**: `company-app-frontend`
4. Click **Next**

**Configure Client Settings:**

```
Client Authentication: OFF (Public Client)
Authorization: OFF
Standard Flow Enabled: ON
Standard Flow Login Redirect URIs: 
  - http://localhost:3000/*
  - http://localhost:4200/*
  - https://yourdomain.com/*
Web Origins: 
  - http://localhost:3000
  - http://localhost:4200
  - https://yourdomain.com
```

**Create API Client (Bearer-only):**

1. Navigate to: **Clients** → **Create Client**
2. **Client ID**: `company-app-api`
3. **Client Authentication**: ON (Confidential)
4. **Authorization Enabled**: OFF

### 3.3 Configure User Roles

**Create Roles:**

1. Navigate to: **Realm Roles** → **Create Role**
2. Create roles: `Admin`, `User`, `Manager`
3. Assign default roles to users as needed

**Configure Client Scopes:**

1. Navigate to: **Client Scopes** → **company-app-api-dedicated**
2. Add **realm_access.roles** mapper to include roles in tokens

---

## 4. Social Provider Configuration

### 4.1 Google OAuth Setup

**Step 1: Create Google OAuth 2.0 Credentials**

1. Go to [Google Cloud Console](https://console.cloud.google.com/)
2. Create a new project or select existing one
3. Navigate to: **APIs & Services** → **Credentials**
4. Click **Create Credentials** → **OAuth 2.0 Client ID**
5. Application type: **Web application**
6. Name: `Keycloak CompanyApp`

**Authorized Redirect URIs:**
```
http://localhost:8080/realms/CompanyApp/broker/google/endpoint
https://your-keycloak-domain.com/realms/CompanyApp/broker/google/endpoint
```

**Step 2: Configure Keycloak Google Identity Provider**

1. Navigate to: **Identity Providers** → **Add Provider** → **Google**
2. **Alias**: `google`
3. **Client ID**: `[Your Google Client ID]`
4. **Client Secret**: `[Your Google Client Secret]`
5. **Scopes**: `email, profile`
6. Click **Add**

**Step 3: Configure Google Mapper**

1. Navigate to the created Google provider
2. Click **Mappers** → **Create Mapper**
3. **Mapper Type**: User Attribute
4. **Name**: `google_email`
5. **User Attribute**: `email`
6. **Token Claim Name**: `email`
7. **Claim JSON Type**: String
8. **Add to ID token**: ON
9. Click **Add**

### 4.2 Facebook OAuth Setup

**Step 1: Create Facebook App**

1. Go to [Facebook Developer Portal](https://developers.facebook.com/)
2. Create a new app: **Consumer**
3. Navigate to: **App Review** → **App Settings** → **Basic**

**Step 2: Configure Facebook Login**

1. Add **Facebook Login** product
2. In **Facebook Login** → **Settings**:
   - **Redirect OAuth URIs**: `http://localhost:8080/realms/CompanyApp/broker/facebook/endpoint`
   - **Cancel OAuth URI**: `http://localhost:8080/realms/CompanyApp`

**Step 3: Configure Keycloak Facebook Identity Provider**

1. Navigate to: **Identity Providers** → **Add Provider** → **Facebook**
2. **Client ID**: `[Your Facebook App ID]`
3. **Client Secret**: `[Your Facebook App Secret]`
4. **Default Scopes**: `email`
5. Click **Add**

### 4.3 GitHub OAuth Setup

**Step 1: Create GitHub OAuth App**

1. Go to GitHub → **Settings** → **Developer settings** → **OAuth Apps**
2. Click **New OAuth App**
3. **Application name**: `Keycloak CompanyApp`
4. **Authorization callback URL**: `http://localhost:8080/realms/CompanyApp/broker/github/endpoint`
5. Click **Register Application**

**Step 2: Configure Keycloak GitHub Identity Provider**

1. Navigate to: **Identity Providers** → **Add Provider** → **GitHub**
2. **Client ID**: `[Your GitHub Client ID]`
3. **Client Secret**: `[Your GitHub Client Secret]`
4. Click **Add**

---

## 5. ASP.NET Core Web API Setup

### 5.1 Create ASP.NET Core Project

```bash
dotnet new webapi -n CompanyApp.Api
cd CompanyApp.Api
```

### 5.2 Install Required Packages

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet add package System.IdentityModel.Tokens.Jwt
```

### 5.3 Configure appsettings.json

```json
{
  "Keycloak": {
    "Realm": "CompanyApp",
    "AuthServerUrl": "http://localhost:8080",
    "ClientId": "company-app-api",
    "ClientSecret": "your-client-secret-here",
    "UseHttps": false
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=localhost;Database=CompanyAppDb;Trusted_Connection=True;TrustServerCertificate=True;"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

### 5.4 Configure Keycloak Options Class

Create `Configuration/KeycloakOptions.cs`:

```csharp
namespace CompanyApp.Api.Configuration;

public class KeycloakOptions
{
    public const string SectionName = "Keycloak";
    
    public string Realm { get; set; } = string.Empty;
    public string AuthServerUrl { get; set; } = string.Empty;
    public string ClientId { get; set; } = string.Empty;
    public string ClientSecret { get; set; } = string.Empty;
    public bool UseHttps { get; set; }
    
    public string GetAuthority()
    {
        var protocol = UseHttps ? "https" : "http";
        return $"{protocol}://{AuthServerUrl}/realms/{Realm}";
    }
    
    public string GetIssuer()
    {
        return $"{GetAuthority()}";
    }
    
    public string GetAudience()
    {
        return ClientId;
    }
}
```

### 5.5 Configure JWT Authentication

Create `Extensions/AuthenticationExtensions.cs`:

```csharp
using CompanyApp.Api.Configuration;
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;

namespace CompanyApp.Api.Extensions;

public static class AuthenticationExtensions
{
    public static IServiceCollection AddKeycloakAuthentication(
        this IServiceCollection services,
        IConfiguration configuration)
    {
        var keycloakOptions = new KeycloakOptions();
        configuration.Bind(KeycloakOptions.SectionName, keycloakOptions);
        
        services.AddSingleton(keycloakOptions);
        
        services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
            .AddJwtBearer(options =>
            {
                options.Authority = keycloakOptions.GetAuthority();
                options.Audience = keycloakOptions.GetAudience();
                options.RequireHttpsMetadata = keycloakOptions.UseHttps;
                
                options.TokenValidationParameters = new TokenValidationParameters
                {
                    ValidateIssuer = true,
                    ValidIssuer = keycloakOptions.GetIssuer(),
                    
                    ValidateAudience = true,
                    ValidAudience = keycloakOptions.GetAudience(),
                    
                    ValidateLifetime = true,
                    
                    ValidateIssuerSigningKey = true,
                    IssuerSigningKeyResolver = (token, securityToken, identifier, validationParameters) =>
                    {
                        var issuer = securityToken.Issuer;
                        var authority = issuer.Replace($"/realms/{keycloakOptions.Realm}", "");
                        
                        // Keycloak's JWKS endpoint will be automatically discovered
                        // from the .well-known/openid-configuration endpoint
                        return validationParameters.IssuerSigningKeys ?? Array.Empty<SecurityKey>();
                    },
                    
                    RoleClaimType = "roles", // Custom claim type for Keycloak roles
                    NameClaimType = "preferred_username"
                };
                
                // Add automatic token refresh and metadata refresh
                options.Events = new JwtBearerEvents
                {
                    OnAuthenticationFailed = context =>
                    {
                        Console.WriteLine($"Authentication failed: {context.Exception.Message}");
                        return Task.CompletedTask;
                    },
                    
                    OnTokenValidated = context =>
                    {
                        Console.WriteLine($"Token validated for user: {context.Principal?.Identity?.Name}");
                        return Task.CompletedTask;
                    },
                    
                    OnMessageReceived = context =>
                    {
                        // Allow token from query string for WebSocket connections
                        var accessToken = context.Request.Query["access_token"];
                        if (!string.IsNullOrEmpty(accessToken))
                        {
                            context.Token = accessToken;
                        }
                        return Task.CompletedTask;
                    }
                };
            });
        
        return services;
    }
}
```

### 5.6 Configure Keycloak Role Mapper

Create `Extensions/ClaimsPrincipalExtensions.cs`:

```csharp
using System.Security.Claims;

namespace CompanyApp.Api.Extensions;

public static class ClaimsPrincipalExtensions
{
    public static string? GetKeycloakId(this ClaimsPrincipal user)
    {
        return user.FindFirst("sub")?.Value;
    }
    
    public static string? GetEmail(this ClaimsPrincipal user)
    {
        return user.FindFirst("email")?.Value;
    }
    
    public static string? GetFirstName(this ClaimsPrincipal user)
    {
        return user.FindFirst("given_name")?.Value;
    }
    
    public static string? GetLastName(this ClaimsPrincipal user)
    {
        return user.FindFirst("family_name")?.Value;
    }
    
    public static IEnumerable<string> GetRoles(this ClaimsPrincipal user)
    {
        // Keycloak stores roles in realm_access.roles claim
        var rolesClaim = user.FindFirst("roles")?.Value;
        
        if (!string.IsNullOrEmpty(rolesClaim))
        {
            // Try to parse as JSON array
            try
            {
                var roles = System.Text.Json.JsonSerializer.Deserialize<string[]>(rolesClaim);
                return roles ?? Array.Empty<string>();
            }
            catch
            {
                // If parsing fails, return empty array
                return Array.Empty<string>();
            }
        }
        
        // Alternative: Check for role claims
        return user.FindAll("role").Select(c => c.Value);
    }
    
    public static bool IsInRole(this ClaimsPrincipal user, string role)
    {
        return user.GetRoles().Contains(role);
    }
}
```

### 5.7 Program.cs Configuration

```csharp
using CompanyApp.Api.Configuration;
using CompanyApp.Api.Extensions;
using Microsoft.AspNetCore.Authentication.JwtBearer;

var builder = WebApplication.CreateBuilder(args);

// Add Keycloak Authentication
builder.Services.AddKeycloakAuthentication(builder.Configuration);

// Add Authorization
builder.Services.AddAuthorization(options =>
{
    // Default policy - authenticated users only
    options.AddPolicy("Authenticated", policy =>
        policy.RequireAuthenticatedUser());
    
    // Role-based policies
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireAuthenticatedUser()
              .RequireRole("Admin"));
    
    options.AddPolicy("UserOrAdmin", policy =>
        policy.RequireAuthenticatedUser()
              .RequireRole("Admin", "User"));
});

builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Configure Swagger with JWT Authentication
builder.Services.ConfigureSwaggerGen();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c =>
    {
        c.SwaggerEndpoint("/swagger/v1/swagger.json", "CompanyApp API V1");
        c.OAuthClientId("company-app-frontend");
        c.OAuthUsePkce();
    });
}

app.UseHttpsRedirection();
app.UseAuthentication();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### 5.8 Create Test Controller

Create `Controllers/AuthController.cs`:

```csharp
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using System.Security.Claims;

namespace CompanyApp.Api.Controllers;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    [HttpGet("test")]
    [Authorize]
    public IActionResult TestAuth()
    {
        var userInfo = new
        {
            IsAuthenticated = User.Identity?.IsAuthenticated ?? false,
            Username = User.Identity?.Name,
            KeycloakId = User.GetKeycloakId(),
            Email = User.GetEmail(),
            FirstName = User.GetFirstName(),
            LastName = User.GetLastName(),
            Roles = User.GetRoles().ToList()
        };
        
        return Ok(userInfo);
    }
    
    [HttpGet("admin")]
    [Authorize(Policy = "AdminOnly")]
    public IActionResult AdminEndpoint()
    {
        return Ok(new
        {
            Message = "Welcome, Admin!",
            Username = User.Identity?.Name
        });
    }
    
    [HttpGet("user")]
    [Authorize(Policy = "UserOrAdmin")]
    public IActionResult UserEndpoint()
    {
        return Ok(new
        {
            Message = "Welcome, User!",
            Username = User.Identity?.Name,
            Roles = User.GetRoles().ToList()
        });
    }
    
    [HttpGet("public")]
    public IActionResult PublicEndpoint()
    {
        return Ok(new
        {
            Message = "This is a public endpoint - no authentication required"
        });
    }
}
```

---

## 6. Database Design & User Synchronization

### 6.1 Database Schema Design

Create `Models/AppUser.cs`:

```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

namespace CompanyApp.Api.Models;

[Table("Users")]
public class AppUser
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    
    [Required]
    [MaxLength(255)]
    public string KeycloakUserId { get; set; } = string.Empty; // 'sub' claim from JWT
    
    [Required]
    [MaxLength(255)]
    public string Email { get; set; } = string.Empty;
    
    [MaxLength(100)]
    public string? FirstName { get; set; }
    
    [MaxLength(100)]
    public string? LastName { get; set; }
    
    [MaxLength(100)]
    public string? Username { get; set; }
    
    [MaxLength(50)]
    public string? Provider { get; set; } // 'google', 'facebook', 'github'
    
    public bool IsActive { get; set; } = true;
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    public DateTime? LastLoginAt { get; set; }
    
    public DateTime UpdatedAt { get; set; } = DateTime.UtcNow;
}
```

### 6.2 Configure Entity Framework

Create `Data/AppDbContext.cs`:

```csharp
using CompanyApp.Api.Models;
using Microsoft.EntityFrameworkCore;

namespace CompanyApp.Api.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options)
    {
    }
    
    public DbSet<AppUser> Users { get; set; }
    
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        modelBuilder.Entity<AppUser>(entity =>
        {
            entity.HasIndex(u => u.KeycloakUserId).IsUnique();
            entity.HasIndex(u => u.Email).IsUnique();
            
            entity.Property(u => u.KeycloakUserId)
                  .IsRequired()
                  .HasMaxLength(255);
            
            entity.Property(u => u.Email)
                  .IsRequired()
                  .HasMaxLength(255);
            
            entity.Property(u => u.FirstName)
                  .HasMaxLength(100);
            
            entity.Property(u => u.LastName)
                  .HasMaxLength(100);
            
            entity.Property(u => u.Username)
                  .HasMaxLength(100);
            
            entity.Property(u => u.Provider)
                  .HasMaxLength(50);
        });
    }
}
```

### 6.3 Configure Database Service

Update `Program.cs`:

```csharp
// Add Database Context
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));
```

### 6.4 Create User Synchronization Middleware

Create `Middleware/UserSyncMiddleware.cs`:

```csharp
using CompanyApp.Api.Data;
using CompanyApp.Api.Extensions;
using CompanyApp.Api.Models;
using Microsoft.EntityFrameworkCore;

namespace CompanyApp.Api.Middleware;

public class UserSyncMiddleware
{
    private readonly RequestDelegate _next;
    
    public UserSyncMiddleware(RequestDelegate next)
    {
        _next = next;
    }
    
    public async Task InvokeAsync(HttpContext context, AppDbContext dbContext)
    {
        // Skip user sync for non-authenticated requests
        if (!context.User.Identity?.IsAuthenticated ?? true)
        {
            await _next(context);
            return;
        }
        
        var keycloakUserId = context.User.GetKeycloakId();
        var email = context.User.GetEmail();
        var firstName = context.User.GetFirstName();
        var lastName = context.User.GetLastName();
        var username = context.User.Identity?.Name;
        
        // Check if user exists in database
        if (!string.IsNullOrEmpty(keycloakUserId) && !string.IsNullOrEmpty(email))
        {
            var user = await dbContext.Users
                .FirstOrDefaultAsync(u => u.KeycloakUserId == keycloakUserId);
            
            if (user == null)
            {
                // Create new user (lazy provisioning)
                user = new AppUser
                {
                    KeycloakUserId = keycloakUserId,
                    Email = email,
                    FirstName = firstName,
                    LastName = lastName,
                    Username = username,
                    Provider = DetectProvider(context.User),
                    CreatedAt = DateTime.UtcNow,
                    LastLoginAt = DateTime.UtcNow,
                    IsActive = true
                };
                
                dbContext.Users.Add(user);
                await dbContext.SaveChangesAsync();
                
                Console.WriteLine($"Created new user: {user.Email} via {user.Provider}");
            }
            else
            {
                // Update last login and user info
                user.LastLoginAt = DateTime.UtcNow;
                user.UpdatedAt = DateTime.UtcNow;
                
                // Update user info if changed
                if (!string.IsNullOrEmpty(firstName) && user.FirstName != firstName)
                    user.FirstName = firstName;
                
                if (!string.IsNullOrEmpty(lastName) && user.LastName != lastName)
                    user.LastName = lastName;
                
                await dbContext.SaveChangesAsync();
            }
            
            // Add user to HttpContext items for later use
            context.Items["AppUser"] = user;
        }
        
        await _next(context);
    }
    
    private static string? DetectProvider(System.Security.Claims.ClaimsPrincipal user)
    {
        // Try to detect the authentication provider from claims
        var identityProvider = user.FindFirst("identity_provider")?.Value;
        
        if (!string.IsNullOrEmpty(identityProvider))
        {
            return identityProvider switch
            {
                "google" => "Google",
                "facebook" => "Facebook", 
                "github" => "GitHub",
                _ => "Keycloak"
            };
        }
        
        // Alternative: Check for provider-specific claims
        if (user.HasClaim(c => c.Type == "google_email"))
            return "Google";
        
        if (user.HasClaim(c => c.Type == "facebook_email"))
            return "Facebook";
        
        return null;
    }
}
```

### 6.5 Register Middleware

Update `Program.cs`:

```csharp
app.UseMiddleware<CompanyApp.Api.Middleware.UserSyncMiddleware>();
app.UseAuthentication();
app.UseAuthorization();
```

---

## 7. Frontend Integration

### 7.1 Frontend Authentication Flow

**Step 1: Redirect to Keycloak for Authentication**

```typescript
const keycloakUrl = 'http://localhost:8080';
const realm = 'CompanyApp';
const clientId = 'company-app-frontend';
const redirectUri = window.location.origin + '/callback';

function login() {
    const state = generateRandomState();
    const codeVerifier = generateCodeVerifier();
    const codeChallenge = generateCodeChallenge(codeVerifier);
    
    sessionStorage.setItem('pkce_state', state);
    sessionStorage.setItem('code_verifier', codeVerifier);
    
    const authUrl = `${keycloakUrl}/realms/${realm}/protocol/openid-connect/auth` +
        `?client_id=${encodeURIComponent(clientId)}` +
        `&redirect_uri=${encodeURIComponent(redirectUri)}` +
        `&response_type=code` +
        `&scope=openid profile email` +
        `&state=${state}` +
        `&code_challenge=${codeChallenge}` +
        `&code_challenge_method=S256`;
    
    window.location.href = authUrl;
}
```

**Step 2: Handle OAuth Callback**

```typescript
async function handleCallback() {
    const urlParams = new URLSearchParams(window.location.search);
    const code = urlParams.get('code');
    const state = urlParams.get('state');
    const storedState = sessionStorage.getItem('pkce_state');
    
    if (state !== storedState) {
        throw new Error('Invalid state parameter');
    }
    
    const codeVerifier = sessionStorage.getItem('code_verifier');
    
    const response = await fetch(`${keycloakUrl}/realms/${realm}/protocol/openid-connect/token`, {
        method: 'POST',
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded',
        },
        body: new URLSearchParams({
            grant_type: 'authorization_code',
            code: code,
            redirect_uri: redirectUri,
            client_id: clientId,
            code_verifier: codeVerifier,
        }),
    });
    
    const tokens = await response.json();
    
    sessionStorage.setItem('access_token', tokens.access_token);
    sessionStorage.setItem('refresh_token', tokens.refresh_token);
    
    window.location.href = '/dashboard';
}
```

**Step 3: Make Authenticated API Calls**

```typescript
async function callApi() {
    const accessToken = sessionStorage.getItem('access_token');
    
    const response = await fetch('http://localhost:5000/api/auth/test', {
        headers: {
            'Authorization': `Bearer ${accessToken}`,
            'Content-Type': 'application/json',
        },
    });
    
    if (response.ok) {
        const userData = await response.json();
        console.log('User data:', userData);
    } else {
        console.error('API call failed');
    }
}
```

### 7.2 Token Refresh Logic

```typescript
async function refreshToken() {
    const refreshToken = sessionStorage.getItem('refresh_token');
    
    if (!refreshToken) {
        logout();
        return;
    }
    
    try {
        const response = await fetch(`${keycloakUrl}/realms/${realm}/protocol/openid-connect/token`, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded',
            },
            body: new URLSearchParams({
                grant_type: 'refresh_token',
                refresh_token: refreshToken,
                client_id: clientId,
            }),
        });
        
        if (response.ok) {
            const tokens = await response.json();
            sessionStorage.setItem('access_token', tokens.access_token);
            sessionStorage.setItem('refresh_token', tokens.refresh_token);
            return tokens.access_token;
        } else {
            logout();
        }
    } catch (error) {
        console.error('Token refresh failed:', error);
        logout();
    }
}

function logout() {
    sessionStorage.clear();
    window.location.href = keycloakUrl + `/realms/${realm}/protocol/openid-connect/logout?redirect_uri=${encodeURIComponent(window.location.origin)}`;
}
```

---

## 8. Testing & Validation

### 8.1 Test Keycloak Configuration

**Test 1: Verify Keycloak is Running**

```bash
curl http://localhost:8080/realms/CompanyApp/.well-known/openid-configuration
```

Expected response: JSON configuration with Keycloak endpoints

**Test 2: Test Direct Keycloak Login**

1. Navigate to: `http://localhost:8080/realms/CompanyApp/protocol/openid-connect/auth?client_id=company-app-frontend&redirect_uri=http://localhost:3000/callback&response_type=code&scope=openid+profile+email`
2. Select social provider (Google/Facebook/GitHub)
3. Complete social login
4. Verify redirect with authorization code

### 8.2 Test ASP.NET Core API

**Test 1: Public Endpoint (No Auth)**

```bash
curl http://localhost:5000/api/auth/public
```

Expected response: Success message

**Test 2: Protected Endpoint (With Auth)**

```bash
# First, get a token from Keycloak
TOKEN=$(curl -X POST "http://localhost:8080/realms/CompanyApp/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=company-app-api" \
  -d "grant_type=password" \
  -d "username=admin" \
  -d "password=admin123" | jq -r '.access_token')

# Then test the protected endpoint
curl -H "Authorization: Bearer $TOKEN" http://localhost:5000/api/auth/test
```

**Test 3: Role-Based Authorization**

```bash
# Test admin endpoint with admin user
curl -H "Authorization: Bearer $ADMIN_TOKEN" http://localhost:5000/api/auth/admin

# Test admin endpoint with regular user (should fail)
curl -H "Authorization: Bearer $USER_TOKEN" http://localhost:5000/api/auth/admin
```

### 8.3 Test Social Provider Integration

**Test Google OAuth:**

1. Navigate to Keycloak login page
2. Select "Google" provider
3. Complete Google OAuth flow
4. Verify user is created in Keycloak
5. Get token and test API access
6. Verify user is created in application database

**Test Facebook OAuth:**

1. Follow same steps as Google
2. Use Facebook provider
3. Verify Facebook user claims are mapped correctly

**Test GitHub OAuth:**

1. Follow same steps as Google
2. Use GitHub provider
3. Verify GitHub user data is properly mapped

### 8.4 Test User Synchronization

**Test 1: New User Provisioning**

```sql
-- Check database before authentication
SELECT * FROM Users WHERE Email = 'newuser@example.com';
-- Should return empty

-- Authenticate as new user via social provider
-- Check database after authentication
SELECT * FROM Users WHERE Email = 'newuser@example.com';
-- Should return user record
```

**Test 2: Existing User Update**

```sql
-- Update user profile in Keycloak
-- Authenticate again
-- Verify LastLoginAt and UpdatedAt are updated
SELECT * FROM Users WHERE KeycloakUserId = 'user-sub-claim';
```

### 8.5 Performance Testing

**Test 1: Token Validation Performance**

```bash
# Measure API response time with valid token
time curl -H "Authorization: Bearer $TOKEN" http://localhost:5000/api/auth/test

# Should be fast (local validation, no network call to Keycloak)
```

**Test 2: Concurrent Requests**

```bash
# Test multiple concurrent requests
for i in {1..100}; do
    curl -H "Authorization: Bearer $TOKEN" http://localhost:5000/api/auth/test &
done
wait
```

---

## 9. Production Deployment

### 9.1 Security Hardening

**Keycloak Security:**

1. **Enable HTTPS**: Set `KC_HTTP_ENABLED=false` and configure SSL certificates
2. **Change Default Admin Password**: Immediately after first deployment
3. **Limit Realms**: Create dedicated realms for each application
4. **Configure Password Policies**: Set strong password requirements
5. **Enable Brute Force Protection**: Configure authentication flow limits
6. **Set Appropriate Token Lifetimes**:
   - Access tokens: 5-15 minutes
   - Refresh tokens: 30 days
   - Remember me tokens: 30 days

**ASP.NET Core Security:**

1. **Enable HTTPS Only**: Set `RequireHttpsMetadata = true`
2. **Validate Audience**: Ensure audience validation is enabled
3. **Implement Rate Limiting**: Add middleware to prevent abuse
4. **Enable CORS**: Configure allowed origins properly
5. **Security Headers**: Add security headers middleware

### 9.2 Production Configuration

**Production appsettings.json:**

```json
{
  "Keycloak": {
    "Realm": "CompanyApp",
    "AuthServerUrl": "https://auth.yourdomain.com",
    "ClientId": "company-app-api",
    "ClientSecret": "your-production-secret",
    "UseHttps": true
  },
  "ConnectionStrings": {
    "DefaultConnection": "Server=production-db;Database=CompanyAppDb;User Id=appuser;Password=strong-password;MultipleActiveResultSets=true;"
  }
}
```

**Production Docker Compose:**

```yaml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:25.0.0
    container_name: keycloak-prod
    environment:
      KC_HTTP_ENABLED: "false"
      KC_HOSTNAME_STRICT: "true"
      KC_HOSTNAME_STRICT_HTTPS: "true"
      KC_PROXY: "edge"
    ports:
      - "443:8443"
    volumes:
      - ./certs:/etc/x509/https
```

### 9.3 Monitoring & Logging

**Application Performance Monitoring:**

1. **Health Checks**: Implement health check endpoints
2. **Structured Logging**: Use Serilog or similar
3. **Token Usage Metrics**: Track token validation success/failure
4. **User Synchronization Metrics**: Monitor user creation rates

**Keycloak Monitoring:**

1. **Admin Events**: Enable and monitor admin events
2. **User Events**: Track login/logout events
3. **Failed Login Attempts**: Monitor authentication failures

### 9.4 Backup & Recovery

**Database Backup:**

```bash
# Daily backup of application database
sqlcmd -S localhost -d CompanyAppDb -Q "BACKUP DATABASE CompanyAppDb TO DISK='C:\Backups\CompanyAppDb.bak'"

# Keycloak database backup
docker exec keycloak-db pg_dump -U keycloak keycloak > keycloak_backup.sql
```

**Keycloak Configuration Export:**

```bash
# Export realm configuration
docker exec keycloak-server /opt/keycloak/bin/kc.sh export --realm CompanyApp --dir /tmp/export
```

---

## 10. Troubleshooting

### 10.1 Common Issues

**Issue: "Invalid Token" or "401 Unauthorized"**

**Solution:**
1. Verify token hasn't expired
2. Check issuer URL matches Keycloak realm URL
3. Validate audience matches client ID
4. Ensure JWT signature validation is working

**Issue: "Role-based authorization not working"**

**Solution:**
1. Check Keycloak role mappers are configured
2. Verify roles are included in token claims
3. Ensure custom claim mapping is correctly configured
4. Test with token introspection endpoint

**Issue: "User not created in database"**

**Solution:**
1. Check UserSyncMiddleware is registered in Program.cs
2. Verify user claims are properly mapped from Keycloak
3. Ensure database connection is working
4. Check middleware execution order

**Issue: "Social provider redirect fails"**

**Solution:**
1. Verify redirect URIs match in Keycloak and provider settings
2. Check provider credentials are correct
3. Ensure HTTPS is properly configured for production
4. Verify CORS settings if using browser-based auth

**Issue: "Token validation is slow"**

**Solution:**
1. Check Keycloak JWKS endpoint caching
2. Verify no network issues with Keycloak server
3. Monitor JWKS refresh rate
4. Consider increasing token lifetime to reduce refresh frequency

### 10.2 Debugging Tools

**View JWT Token Contents:**

```bash
# Decode JWT token
echo $TOKEN | cut -d. -f2 | base64 -d | jq
```

**Test Keycloak Configuration:**

```bash
# Get OpenID configuration
curl http://localhost:8080/realms/CompanyApp/.well-known/openid-configuration

# Get JWKS (public keys)
curl http://localhost:8080/realms/CompanyApp/protocol/openid-connect/certs

# Introspect token
curl -X POST "http://localhost:8080/realms/CompanyApp/protocol/openid-connect/introspect" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=company-app-api" \
  -d "client_secret=your-secret" \
  -d "token=$TOKEN"
```

**Database Queries:**

```sql
-- Check recently created users
SELECT * FROM Users ORDER BY CreatedAt DESC;

-- Users by provider
SELECT Provider, COUNT(*) FROM Users GROUP BY Provider;

-- Stale users (no recent login)
SELECT * FROM Users WHERE LastLoginAt < DATEADD(day, -30, GETDATE());
```

### 10.3 Performance Optimization

**Optimize Token Validation:**

1. **Cache JWKS Response**: Configure appropriate cache duration
2. **Reduce Token Size**: Remove unnecessary claims from tokens
3. **Database Indexing**: Ensure proper indexes on user lookup fields
4. **Connection Pooling**: Optimize database connection settings

**Scale Considerations:**

1. **Load Balancing**: Configure multiple Keycloak instances
2. **Database Scaling**: Plan for database read replicas
3. **Caching Layer**: Consider Redis for session/token caching
4. **Monitoring**: Set up comprehensive monitoring

---

## 11. Next Steps

### 11.1 Advanced Features

1. **Multi-factor Authentication**: Configure MFA in Keycloak
2. **User Consent Management**: Implement consent tracking
3. **Advanced Token Customization**: Add custom claims and mappers
4. **Event Sourcing**: Implement event-based user sync
5. **API Gateway Integration**: Integrate with API Gateway for routing

### 11.2 Additional Providers

1. **Microsoft OAuth**: Add Azure AD/Microsoft login
2. **LinkedIn OAuth**: Add LinkedIn login
3. **Apple Sign In**: Add Apple authentication
4. **SAML Integration**: Support enterprise SAML providers

### 11.3 Mobile App Support

1. **Native App Flows**: Configure mobile clients
2. **Deep Linking**: Set up deep linking for mobile apps
3. **Biometric Authentication**: Integrate with device biometrics
4. **Offline Token Support**: Configure token refresh for offline scenarios

---

## Conclusion

This implementation provides a production-ready, stateless authentication system using Keycloak as an identity broker. The ASP.NET Core API validates JWT tokens locally without making network calls to Keycloak for each request, ensuring high performance while maintaining security.

The system supports multiple social providers, automatic user provisioning, and role-based authorization, making it suitable for modern multi-tenant applications requiring federated authentication.

For questions or issues, refer to the troubleshooting section or consult the official Keycloak and ASP.NET Core documentation.