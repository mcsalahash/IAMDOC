# Desktop .NET + WAM Broker

Pour WPF, WinForms, WinUI 3, applications console Windows.

---

## Pourquoi le WAM Broker ?

Le Web Account Manager (WAM) est le broker d'authentification intégré à Windows 10/11. MSAL.NET peut y déléguer l'authentification au lieu d'ouvrir un navigateur embarqué.

| | Navigateur embarqué | WAM Broker |
|---|---|---|
| SSO avec le compte Windows | Non | Oui |
| Intégration PRT | Non | Oui |
| Conditional Access device | Limité | Complet |
| Windows Hello | Non | Oui |
| Expérience utilisateur | Navigateur dans l'app | Native OS |

---

## Configuration

```bash
dotnet add package Microsoft.Identity.Client
dotnet add package Microsoft.Identity.Client.Broker
```

```csharp
using Microsoft.Identity.Client;
using Microsoft.Identity.Client.Broker;

var brokerOptions = new BrokerOptions(BrokerOptions.OperatingSystems.Windows)
{
    Title = "Mon Application"
};

var app = PublicClientApplicationBuilder
    .Create("YOUR_CLIENT_ID")
    .WithAuthority("https://login.microsoftonline.com/YOUR_TENANT_ID")
    .WithDefaultRedirectUri()  // Génère automatiquement le redirect URI WAM
    .WithBroker(brokerOptions)
    .WithParentActivityOrWindow(GetWindowHandle)
    .Build();
```

---

## Pattern d'authentification recommandé

```csharp
private static async Task<AuthenticationResult> GetTokenAsync(string[] scopes)
{
    var accounts = await app.GetAccountsAsync();

    // 1. Essayer le compte WAM (compte Windows connecté)
    var account = accounts.FirstOrDefault()
                ?? PublicClientApplication.OperatingSystemAccount;

    try
    {
        // 2. Tentative silencieuse — PRT + cache
        return await app.AcquireTokenSilent(scopes, account)
            .ExecuteAsync();
    }
    catch (MsalUiRequiredException ex)
    {
        // MsalUiRequiredException signifie :
        // - Aucun token valide en cache
        // - Refresh token expiré (>90 jours)
        // - MDP changé / session révoquée
        // - Step-up MFA requis par une politique CA
        // - Premier lancement de l'application

        return await app.AcquireTokenInteractive(scopes)
            .WithParentActivityOrWindow(GetWindowHandle())
            .ExecuteAsync();
    }
}

// WPF
private static IntPtr GetWindowHandle()
    => new WindowInteropHelper(Application.Current.MainWindow).Handle;

// WinForms
// private static IntPtr GetWindowHandle() => this.Handle;
```

---

## OperatingSystemAccount

`PublicClientApplication.OperatingSystemAccount` représente le **compte Windows actuellement connecté** à l'OS. Son usage :

- Tente un SSO totalement silencieux via le PRT (Primary Refresh Token)
- Ne fonctionne que si l'appareil est Entra joined ou Hybrid joined
- Si le PRT est absent ou invalide → fallback vers l'interaction

```csharp
// Pattern optimal pour un device corporate Entra joined
var accounts = await app.GetAccountsAsync();

// Priorité 1 : compte déjà connu de MSAL (sessions précédentes)
// Priorité 2 : compte Windows OS (SSO via PRT)
var account = accounts.FirstOrDefault()
           ?? PublicClientApplication.OperatingSystemAccount;
```

---

## Cache persistant

Sans sérialisation, le cache est perdu à chaque fermeture. Sur Windows, MSAL utilise DPAPI.

```csharp
using Microsoft.Identity.Client.Extensions.Msal;

var storageProperties = new StorageCreationPropertiesBuilder(
    "msal_token_cache.bin",
    Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData))
    .WithLinuxKeyring("com.monapp.msal", "default", "MSALCache", null, null)
    .WithMacKeychain("MSALCache", "MSALCache")
    .Build();

var cacheHelper = await MsalCacheHelper.CreateAsync(storageProperties);
cacheHelper.RegisterCache(app.UserTokenCache);
```

---

## Application console / CLI

```csharp
// Console — pas de fenêtre, utiliser le handle de la console
[DllImport("kernel32.dll")]
static extern IntPtr GetConsoleWindow();

var app = PublicClientApplicationBuilder
    .Create("CLIENT_ID")
    .WithDefaultRedirectUri()
    .WithBroker(new BrokerOptions(BrokerOptions.OperatingSystems.Windows))
    .WithParentActivityOrWindow(GetConsoleWindow)
    .Build();
```

!!! tip "Console sans broker"
    Si l'application CLI tourne dans un contexte sans UI (pipeline CI, tâche planifiée), utiliser **Device Code Flow** ou **Managed Identity** plutôt que WAM.
