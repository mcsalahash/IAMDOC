# Logs Entra ID

---

## Types de logs

| Log | Contenu | Rétention |
|---|---|---|
| **SignInLogs** | Toutes les tentatives de connexion | 30 jours (P1/P2), 7 jours (Free) |
| **AuditLogs** | Changements de configuration (users, apps, rôles) | 30 jours |
| **ProvisioningLogs** | Provisioning SCIM vers apps tierces | 30 jours |
| **RiskyUsers** | Utilisateurs marqués à risque | Jusqu'à résolution |
| **RiskDetections** | Événements de risque individuels | 30 jours |

---

## Structure d'un SignInLog

```json
{
  "id": "...",
  "createdDateTime": "2026-06-28T09:15:00Z",
  "userDisplayName": "Salaheddine El Machkoury",
  "userPrincipalName": "seelmachkoury@dgfla.com",
  "appDisplayName": "Microsoft Teams",
  "appId": "1fec8e78-...",
  "ipAddress": "86.1.2.3",
  "clientAppUsed": "Browser",
  "resultType": "0",
  "resultDescription": "Successfully signed in",
  "conditionalAccessStatus": "success",
  "authenticationRequirement": "multiFactorAuthentication",
  "deviceDetail": {
    "deviceId": "...",
    "operatingSystem": "Windows10",
    "isCompliant": true,
    "isManaged": true
  },
  "location": {
    "city": "Paris",
    "countryOrRegion": "FR"
  },
  "mfaDetail": {
    "authMethod": "Phone app notification",
    "authDetail": "MFA completed in Azure AD"
  }
}
```

---

## Exporter vers Log Analytics

```
Portail Entra → Monitoring → Diagnostic settings
→ Add diagnostic setting
→ Cocher : SignInLogs, AuditLogs, RiskyUsers, RiskDetections
→ Destination : Send to Log Analytics workspace
```

!!! tip "Rétention étendue"
    Une fois dans Log Analytics, la rétention est configurable jusqu'à **2 ans** (vs 30 jours dans Entra ID seul).

---

## Accès via Graph API

```powershell
# Récupérer les 10 dernières connexions
$logs = Invoke-MgGraphRequest -Method GET `
    -Uri "https://graph.microsoft.com/v1.0/auditLogs/signIns?`$top=10&`$orderby=createdDateTime desc"

$logs.value | Select-Object userPrincipalName, appDisplayName, createdDateTime, resultType
```

```python
# Python
import requests

r = requests.get(
    "https://graph.microsoft.com/v1.0/auditLogs/signIns",
    headers={"Authorization": f"Bearer {token}"},
    params={"$top": 10, "$orderby": "createdDateTime desc"}
)
for log in r.json()["value"]:
    print(log["userPrincipalName"], log["resultType"], log["createdDateTime"])
```

---

## Codes de résultat fréquents

| ResultType | Signification |
|---|---|
| `0` | Succès |
| `50058` | Session silencieuse expirée |
| `50076` | MFA requise |
| `50079` | MFA forcée (état sécurité changé) |
| `50105` | Utilisateur non assigné à l'application |
| `53003` | Bloqué par Conditional Access |
| `65001` | Consentement manquant |
| `700016` | Application introuvable dans le tenant |
| `7000215` | Secret client invalide ou expiré |

Référence complète : [Codes AADSTS →](../index.md#codes-aadsts-fréquents)
