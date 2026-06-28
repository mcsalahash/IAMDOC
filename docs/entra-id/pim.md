# PIM — Privileged Identity Management

Accès juste-à-temps (Just-In-Time) aux rôles privilégiés — les droits n'existent que quand ils sont nécessaires.

---

## Eligible vs Active

| | Eligible | Active |
|---|---|---|
| **État** | Peut activer le rôle | A le rôle en permanence |
| **Activation** | Requiert une action explicite | Immédiat |
| **Durée** | Limitée (ex: 1h, 8h max) | Permanente ou limitée |
| **MFA** | Requise à l'activation | Optionnelle |
| **Justification** | Requise | Non |
| **Approbation** | Possible | Non applicable |

!!! tip "Règle d'or"
    Aucun administrateur ne doit avoir de rôle **Active** permanent. Tout passe par des assignations **Eligible** activées à la demande.

---

## Activer un rôle éligible

```
Portail Entra → Identity Governance → PIM
→ Mes rôles → Rôles Entra ID éligibles
→ Activer → Justification → Durée → Activer
```

```powershell
# PowerShell — activer un rôle éligible
$roleId = "62e90394-69f5-4237-9190-012177145e10"  # Global Administrator
New-MgRoleManagementDirectoryRoleAssignmentScheduleRequest `
    -Action "selfActivate" `
    -PrincipalId (Get-MgContext).Account `
    -RoleDefinitionId $roleId `
    -DirectoryScopeId "/" `
    -Justification "Mise à jour configuration CA urgente" `
    -ScheduleInfo @{
        StartDateTime = (Get-Date)
        Expiration    = @{ Type = "AfterDuration"; Duration = "PT2H" }
    }
```

---

## Hiérarchie des rôles Entra ID

| Rôle | Périmètre | Usage |
|---|---|---|
| **Global Administrator** | Tout le tenant | Break glass uniquement |
| **Privileged Role Administrator** | Gestion des rôles | Configuration PIM |
| **Security Administrator** | Sécurité | CA, Identity Protection |
| **User Administrator** | Comptes utilisateurs | Helpdesk niveau 2 |
| **Application Administrator** | App Registrations | Développeurs |
| **Exchange Administrator** | Exchange Online | Messagerie |
| **Intune Administrator** | Intune | Endpoint management |

---

## Break Glass Accounts

Comptes d'accès d'urgence en cas de lockout du tenant.

```
Règles obligatoires :
✓ 2 comptes minimum (redondance)
✓ Exclure de TOUTES les politiques Conditional Access
✓ MFA avec clé physique FIDO2 (pas d'Authenticator app)
✓ Mots de passe longs (>30 chars), stockés hors ligne
✓ Alertes sur chaque utilisation (Log Analytics)
✓ Test de connexion mensuel
```

```kql
// Alerte sur utilisation des comptes Break Glass
SigninLogs
| where UserPrincipalName in ("breakglass1@dgfla.com", "breakglass2@dgfla.com")
| project TimeGenerated, UserPrincipalName, IPAddress, AppDisplayName, ResultType
```

---

## Audit PIM

```kql
// Activations de rôles PIM sur 7 jours
AuditLogs
| where TimeGenerated > ago(7d)
| where Category == "RoleManagement"
| where OperationName == "Add member to role (eligible)"
   or OperationName == "Add eligible member to role completed (PIM activation)"
| extend
    Actor  = tostring(InitiatedBy.user.userPrincipalName),
    Role   = tostring(TargetResources[0].displayName)
| project TimeGenerated, Actor, Role, OperationName
| order by TimeGenerated desc
```
