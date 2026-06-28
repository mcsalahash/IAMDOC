# Web serveur — Microsoft.Identity.Web

Pour ASP.NET Core, Node.js, Python Flask/Django — toute application avec un backend confidentiel.

---

## Avantage vs SPA

Un web serveur peut stocker un `client_secret` ou un certificat de façon sécurisée. Il utilise le flow **Authorization Code confidentiel** — plus sécurisé que PKCE seul car le serveur s'authentifie lors de l'échange du code.

---

## Configuration ASP.NET Core

### Installation

```bash
dotnet add package Microsoft.Identity.Web
dotnet add package Microsoft.Identity.Web.MicrosoftGraph
```

### Program.cs

```csharp
using Microsoft.Identity.Web;
using Microsoft.Identity.Web.UI;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication(OpenIdConnectDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApp(builder.Configuration.GetSection("AzureAd"))
    .EnableTokenAcquisitionToCallDownstreamApi()
    .AddMicrosoftGraph(builder.Configuration.GetSection("MicrosoftGraph"))
    .AddDistributedTokenCaches();

builder.Services.AddStackExchangeRedisCache(options => {
    options.Configuration = builder.Configuration["Redis:Connection"];
});

builder.Services.AddRazorPages()
    .AddMicrosoftIdentityUI();

var app = builder.Build();
app.UseAuthentication();
app.UseAuthorization();
app.MapRazorPages();
app.Run();
```

### appsettings.json

```json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "YOUR_TENANT_ID",
    "ClientId": "YOUR_CLIENT_ID",
    "ClientSecret": "YOUR_SECRET",
    "CallbackPath": "/signin-oidc"
  },
  "MicrosoftGraph": {
    "BaseUrl": "https://graph.microsoft.com/v1.0",
    "Scopes": "User.Read Mail.Read"
  }
}
```

---

## Appeler Microsoft Graph (OBO automatique)

```csharp
// Controller — appel Graph au nom de l'utilisateur connecté
[Authorize]
public class ProfileController : Controller
{
    private readonly GraphServiceClient _graphClient;

    public ProfileController(GraphServiceClient graphClient)
        => _graphClient = graphClient;

    public async Task<IActionResult> Index()
    {
        var user = await _graphClient.Me.GetAsync();
        return View(user);
    }
}
```

---

## Token cache distribué

En production, le cache de tokens doit être **partagé entre instances** (load balancer).

```csharp
// Redis (recommandé)
.AddStackExchangeRedisCache(options =>
    options.Configuration = "mon-redis.cache.windows.net:6380,password=..."
)

// SQL Server
.AddSqlServerCache(options => {
    options.ConnectionString = "Server=...";
    options.SchemaName = "dbo";
    options.TableName = "TokenCache";
})

// Cosmos DB
.AddCosmosCache(options => {
    options.ContainerName = "TokenCache";
    options.DatabaseName = "AuthCache";
    options.ClientBuilder = new CosmosClientBuilder("...");
    options.CreateIfNotExists = true;
})
```

---

## BFF — Backend for Frontend

Pattern de sécurité maximal pour les SPAs : le backend gère tous les tokens, le frontend ne les voit jamais.

```
SPA (React)                  BFF (ASP.NET Core)           API Ressource
    │                              │                            │
    │  Cookie HttpOnly (session)   │                            │
    │◄────────────────────────────│                            │
    │                              │  Bearer access_token       │
    │  GET /api/data               │───────────────────────────►│
    │─────────────────────────────►│                            │
    │                              │◄───────────────────────────│
    │◄────────────────────────────│  Données                   │
```

```csharp
// BFF — endpoint proxy qui injecte le token
app.MapGet("/api/me", [Authorize] async (GraphServiceClient graph) => {
    var user = await graph.Me.GetAsync();
    return Results.Ok(user);
});

// CORS minimal — autoriser uniquement l'origine SPA
app.UseCors(policy => policy
    .WithOrigins("https://monapp.com")
    .AllowCredentials()
    .AllowAnyMethod());
```

---

## Node.js avec MSAL Node

```javascript
const { ConfidentialClientApplication } = require("@azure/msal-node");
const session = require("express-session");

const cca = new ConfidentialClientApplication({
    auth: {
        clientId: "CLIENT_ID",
        clientSecret: "CLIENT_SECRET",
        authority: "https://login.microsoftonline.com/TENANT_ID"
    }
});

// Route de callback
app.get("/callback", async (req, res) => {
    const tokenResponse = await cca.acquireTokenByCode({
        code: req.query.code,
        scopes: ["User.Read"],
        redirectUri: "http://localhost:3000/callback"
    });
    req.session.accessToken = tokenResponse.accessToken;
    res.redirect("/profile");
});
```
