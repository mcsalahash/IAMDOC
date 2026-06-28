# Lifecycle Workflows

Automatisation du provisioning et déprovisioning d'identités piloté par des **événements RH**.

---

## Principe

Lifecycle Workflows déclenche des tâches automatiques basées sur des attributs de l'utilisateur (`employeeHireDate`, `employeeLeaveDateTime`) ou des changements de propriétés (changement de département, de manager...).

```
Événement RH (embauche, départ, mutation)
    │
    ▼
Synchronisation RH → Entra ID (via SCIM ou Entra Connect)
    │
    ▼
Lifecycle Workflow déclenché
    │
    ▼
Tâches automatiques :
    ├── Créer le compte utilisateur
    ├── Ajouter aux groupes / access packages
    ├── Envoyer un email de bienvenue avec les credentials
    ├── Assigner une licence M365
    ├── Créer un ticket ServiceNow
    └── Ou à l'inverse : désactiver, révoquer, archiver
```

---

## Types de workflows

| Workflow | Déclencheur | Cas d'usage |
|---|---|---|
| **Joiner** | `employeeHireDate - X jours` | Onboarding avant l'arrivée |
| **Mover** | Changement d'attribut (département, manager, poste) | Mutation interne |
| **Leaver** | `employeeLeaveDateTime` ou désactivation | Offboarding |

---

## Tâches disponibles

| Tâche | Description |
|---|---|
| Add user to groups | Ajouter l'utilisateur à des groupes |
| Assign licenses | Attribuer des licences M365 |
| Enable user account | Activer le compte |
| Disable user account | Désactiver le compte |
| Remove user from all groups | Retirer de tous les groupes |
| Revoke user sign-in sessions | Révoquer toutes les sessions actives |
| Delete user | Suppression définitive (leaver) |
| Send welcome email | Email de bienvenue avec TAP (Temporary Access Pass) |
| Run custom extension | Appeler un Logic App ou une Azure Function |

---

## Créer un workflow via PowerShell

```powershell
Connect-MgGraph -Scopes "LifecycleWorkflows.ReadWrite.All"

$workflow = New-MgIdentityGovernanceLifecycleWorkflow -BodyParameter @{
    displayName = "Onboarding — Nouvel employé"
    category    = "joiner"
    isEnabled   = $true
    isSchedulingEnabled = $true
    executionConditions = @{
        "@odata.type" = "#microsoft.graph.identityGovernance.triggerAndScopeBasedConditions"
        scope = @{
            "@odata.type" = "#microsoft.graph.identityGovernance.ruleBasedSubjectSet"
            rule = "department eq 'DSI'"
        }
        trigger = @{
            "@odata.type" = "#microsoft.graph.identityGovernance.timeBasedAttributeTrigger"
            timeBasedAttribute = "employeeHireDate"
            offsetInDays = -2  # 2 jours avant l'arrivée
        }
    }
    tasks = @(
        @{
            displayName   = "Activer le compte"
            category      = "joiner"
            taskDefinitionId = "6fc52c9d-398b-4305-9763-15f42c1676fc"
            isEnabled     = $true
            arguments     = @()
        },
        @{
            displayName   = "Envoyer email de bienvenue"
            category      = "joiner"
            taskDefinitionId = "70b29d51-b59a-4773-9280-8841dfd3f2ea"
            isEnabled     = $true
            arguments     = @(@{ name = "cc"; value = "it-helpdesk@dgfla.com" })
        }
    )
}
```
