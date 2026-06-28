# App Registrations

---

## App Registration vs Enterprise Application

| | App Registration | Enterprise Application |
|---|---|---|
| **Quoi** | La définition de l'application | La représentation dans un tenant |
| **Où** | Un seul tenant (l'éditeur) | Chaque tenant qui utilise l'app |
| **Contient** | Client ID, secrets, redirect URIs, permissions déclarées | Assignations, politiques CA, consentement |
| **Portail** | Entra ID → Applications → App registrations | Entra ID → Applications → Enterprise applications |

---

## Paramètres clés

### Supported account types

| Option | Usage |
|---|---|
| Single tenant | Application interne uniquement — comptes de ton tenant |
| Multitenant | Application SaaS — comptes de n'importe quel tenant Entra |
| Multitenant + comptes Microsoft | Apps grand public (Xbox, Outlook.com...) |

### Redirect URIs

```
# Web (ASP.NET, Node.js)
https://monapp.com/callback

# SPA (React, Angular)
http://localhost:3000  (dev)
https://monapp.com    (prod)

# Mobile & Desktop
ms-appx-web://Microsoft.AAD.BrokerPlugin/{client_id}  (WAM)
myapp://callback                                        (custom scheme)
```

---

## Permissions et consentement

### Ajouter des permissions Microsoft Graph

```powershell
# PowerShell — ajouter User.Read en permission déléguée
$graphId = "00000003-0000-0000-c000-000000000000"  # Microsoft Graph App ID
$userReadId = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"  # User.Read scope ID

Update-MgApplication -ApplicationId $appObjectId -RequiredResourceAccess @{
    ResourceAppId = $graphId
    ResourceAccess = @(@{
        Id   = $userReadId
        Type = "Scope"  # "Scope" = déléguée, "Role" = application
    })
}
```

### Admin consent — accorder le consentement au tenant entier

```
https://login.microsoftonline.com/{tenant}/adminconsent
  ?client_id={client_id}
  &redirect_uri={redirect_uri}
```

---

## Secrets et certificats

!!! warning "Préférer les certificats aux secrets"
    Un secret client est une chaîne — elle peut être copiée, partagée accidentellement, et n'est liée à aucune machine. Un certificat lie l'authentification à la détention de la clé privée.

```powershell
# Créer un secret client (expiration 1 an max recommandée)
$secret = Add-MgApplicationPassword -ApplicationId $appObjectId `
    -PasswordCredential @{ DisplayName = "prod-2026"; EndDateTime = "2027-01-01" }
$secret.SecretText  # Copier immédiatement — non récupérable ensuite

# Ajouter un certificat
$cert = New-SelfSignedCertificate -Subject "CN=MonApp" -CertStoreLocation "Cert:\CurrentUser\My"
$certBase64 = [Convert]::ToBase64String($cert.RawData)
Update-MgApplication -ApplicationId $appObjectId -KeyCredentials @{
    Type  = "AsymmetricX509Cert"
    Usage = "Verify"
    Key   = [Convert]::FromBase64String($certBase64)
}
```

---

## App Roles

Les App Roles permettent de définir des rôles métier assignables aux utilisateurs ou à d'autres applications.

```json
// Manifeste — définir un App Role
"appRoles": [
  {
    "allowedMemberTypes": ["User", "Application"],
    "description": "Accès complet à l'API",
    "displayName": "API.FullAccess",
    "id": "generate-a-guid-here",
    "isEnabled": true,
    "value": "API.FullAccess"
  }
]
```

```csharp
// Vérifier un App Role côté API (ASP.NET Core)
[Authorize(Roles = "API.FullAccess")]
app.MapGet("/admin", () => "Accès admin");
```

---

## Federated Identity Credentials

Permet à une application externe (GitHub Actions, AKS...) de s'authentifier sans secret.

```powershell
# Configurer GitHub Actions comme identité fédérée
New-MgApplicationFederatedIdentityCredential -ApplicationId $appObjectId `
    -BodyParameter @{
        Name     = "github-actions-prod"
        Issuer   = "https://token.actions.githubusercontent.com"
        Subject  = "repo:mon-org/mon-repo:ref:refs/heads/main"
        Audiences = @("api://AzureADTokenExchange")
    }
```
