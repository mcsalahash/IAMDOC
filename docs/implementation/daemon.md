# Daemon & Managed Identity

Services, scripts, pipelines — authentification sans utilisateur.

---

## Choisir entre Client Credentials et Managed Identity

| Critère | Client Credentials | Managed Identity |
|---|---|---|
| Ressource Azure ? | Indifférent | Requis |
| Secret à gérer | Oui (secret ou cert.) | Non |
| Rotation | Manuelle ou automatisée | Automatique |
| Environnement on-prem | Oui | Non (sauf Arc) |
| Recommandation | Si pas de MI possible | **Toujours préférer** |

---

## Client Credentials — avec certificat (recommandé)

```csharp
using Microsoft.Identity.Client;
using System.Security.Cryptography.X509Certificates;

// Charger le certificat depuis le Key Vault (avec MI)
var certClient = new CertificateClient(
    new Uri("https://mon-kv.vault.azure.net/"),
    new DefaultAzureCredential()
);
var cert = await certClient.DownloadCertificateAsync("mon-cert");

// Configurer MSAL avec le certificat
var app = ConfidentialClientApplicationBuilder
    .Create("CLIENT_ID")
    .WithCertificate(cert)
    .WithAuthority("https://login.microsoftonline.com/TENANT_ID")
    .Build();

// Acquérir un token (mis en cache automatiquement)
var result = await app
    .AcquireTokenForClient(new[] { "https://graph.microsoft.com/.default" })
    .ExecuteAsync();

// Appeler l'API
using var http = new HttpClient();
http.DefaultRequestHeaders.Authorization =
    new AuthenticationHeaderValue("Bearer", result.AccessToken);
var response = await http.GetAsync("https://graph.microsoft.com/v1.0/users");
```

---

## Client Credentials — PowerShell (Azure Automation)

```powershell
# Avec certificat (recommandé)
Connect-MgGraph -ClientId "CLIENT_ID" `
                -TenantId "TENANT_ID" `
                -CertificateThumbprint "ABC123..."

# Avec secret (acceptable pour les scripts internes non exposés)
$body = @{
    grant_type    = "client_credentials"
    client_id     = $ClientId
    client_secret = $ClientSecret
    scope         = "https://graph.microsoft.com/.default"
}
$token = (Invoke-RestMethod -Method POST `
    -Uri "https://login.microsoftonline.com/$TenantId/oauth2/v2.0/token" `
    -Body $body).access_token
```

---

## Managed Identity

```csharp
// Azure.Identity — choisir le credential adapté à l'environnement

// Production sur Azure
var credential = new ManagedIdentityCredential();

// Développement local
var credential = new AzureCliCredential();  // après 'az login'

// Les deux en une ligne (DefaultAzureCredential)
var credential = new DefaultAzureCredential();

// Utilisation
var graphClient = new GraphServiceClient(credential);
var users = await graphClient.Users.GetAsync();
```

```powershell
# Azure Automation — Managed Identity
Connect-MgGraph -Identity

# Azure Function — Managed Identity
Connect-ExchangeOnline -ManagedIdentity -Organization "dgfla.onmicrosoft.com"
```

---

## Assigner les permissions à une Managed Identity

Les MI utilisent des **App Roles** (permissions d'application) — pas de consentement utilisateur.

```powershell
# Assigner User.Read.All à une Managed Identity
$miObjectId = (Get-AzUserAssignedIdentity -Name "mi-mon-app" -ResourceGroup "mon-rg").PrincipalId
$graphId    = (Get-MgServicePrincipal -Filter "AppId eq '00000003-0000-0000-c000-000000000000'").Id
$role       = (Get-MgServicePrincipal -Id $graphId).AppRoles |
                Where-Object { $_.Value -eq "User.Read.All" }

New-MgServicePrincipalAppRoleAssignment `
    -ServicePrincipalId $miObjectId `
    -PrincipalId        $miObjectId `
    -ResourceId         $graphId `
    -AppRoleId          $role.Id
```

---

## Azure DevOps — Federated Identity (sans secret)

```powershell
# App Registration — configurer GitHub Actions comme identité fédérée
New-MgApplicationFederatedIdentityCredential -ApplicationId $appObjectId `
    -BodyParameter @{
        Name      = "github-prod"
        Issuer    = "https://token.actions.githubusercontent.com"
        Subject   = "repo:mon-org/mon-repo:ref:refs/heads/main"
        Audiences = @("api://AzureADTokenExchange")
    }
```

```yaml
# GitHub Actions — connexion sans secret
- uses: azure/login@v2
  with:
    client-id: ${{ secrets.AZURE_CLIENT_ID }}
    tenant-id: ${{ secrets.AZURE_TENANT_ID }}
    subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}
    # Pas de client-secret — authentification par federated identity
```
