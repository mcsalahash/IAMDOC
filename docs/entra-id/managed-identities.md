# Managed Identities

Identités gérées par Azure — zéro secret à stocker, zéro rotation manuelle.

---

## System-assigned vs User-assigned

| | System-assigned | User-assigned |
|---|---|---|
| **Cycle de vie** | Lié à la ressource | Indépendant |
| **Suppression** | Automatique avec la ressource | Manuelle |
| **Partage** | Non — une par ressource | Oui — plusieurs ressources |
| **Cas d'usage** | Ressource unique, identité dédiée | Pool de VMs, identité partagée |

---

## Activation

```bash
# VM Azure
az vm identity assign --name ma-vm --resource-group mon-rg

# App Service
az webapp identity assign --name mon-app --resource-group mon-rg

# Azure Function
az functionapp identity assign --name ma-function --resource-group mon-rg

# User-assigned — créer d'abord l'identité
az identity create --name mi-mon-app --resource-group mon-rg
az webapp identity assign --name mon-app --resource-group mon-rg \
    --identities /subscriptions/.../resourceGroups/.../providers/Microsoft.ManagedIdentity/userAssignedIdentities/mi-mon-app
```

---

## Utilisation dans le code

```csharp
// Azure.Identity — DefaultAzureCredential
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;
using Azure.Storage.Blobs;

// Key Vault
var kvClient = new SecretClient(
    new Uri("https://mon-kv.vault.azure.net/"),
    new DefaultAzureCredential()
);
var secret = await kvClient.GetSecretAsync("mon-secret");

// Blob Storage
var blobClient = new BlobServiceClient(
    new Uri("https://monstorage.blob.core.windows.net/"),
    new DefaultAzureCredential()
);
```

```python
# Python — azure-identity
from azure.identity import DefaultAzureCredential, ManagedIdentityCredential
from azure.keyvault.secrets import SecretClient

# En production — utiliser ManagedIdentityCredential directement
credential = ManagedIdentityCredential()
client = SecretClient("https://mon-kv.vault.azure.net/", credential)
secret = client.get_secret("mon-secret")
```

---

## Assigner des permissions RBAC

```bash
# Donner accès à un Key Vault
az role assignment create \
    --assignee-object-id $(az vm identity show --name ma-vm --resource-group mon-rg --query principalId -o tsv) \
    --role "Key Vault Secrets User" \
    --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.KeyVault/vaults/{kv}

# Donner accès à un Storage Account
az role assignment create \
    --assignee-object-id $principalId \
    --role "Storage Blob Data Reader" \
    --scope /subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Storage/storageAccounts/{sa}
```

---

## Microsoft Graph avec Managed Identity

```powershell
# Azure Automation Runbook — connexion avec Managed Identity
Connect-MgGraph -Identity -Scopes "User.Read.All", "Group.Read.All"

$users = Get-MgUser -Filter "accountEnabled eq true" -All
```

!!! warning "Scopes et Managed Identity"
    Les Managed Identities utilisent des **permissions d'application** (pas déléguées). Il faut assigner les permissions via `New-MgServicePrincipalAppRoleAssignment`, pas via le consentement utilisateur.
