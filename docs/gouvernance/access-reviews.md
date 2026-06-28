# Access Reviews

Processus récurrent de validation que les accès accordés sont toujours justifiés — pièce maîtresse de la conformité SOX, ISO 27001, et RGPD.

---

## Pourquoi les Access Reviews ?

Sans revue périodique, les accès s'accumulent : un utilisateur change de poste, garde ses anciens accès ; un prestataire part, son accès reste actif ; un administrateur reste admin alors qu'il ne gère plus le périmètre concerné.

---

## Types de revues

| Type | Portée | Reviewers |
|---|---|---|
| **Groupes et applications** | Membres d'un groupe / utilisateurs d'une app | Manager, membres eux-mêmes, ou reviewers désignés |
| **Rôles Entra ID** | Assignations de rôles admin | Admins, propriétaires de rôles |
| **Rôles Azure** | Assignations RBAC Azure | Propriétaires de la ressource |
| **Packages d'accès** | Assignments Entitlement Management | Manager, propriétaire du package |

---

## Créer une revue

```
Portail Entra → Identity Governance → Access Reviews → New access review

Scope        : Groupes, apps, rôles ou packages
Reviewers    : Manager, members (self-review), ou groupe spécifique
Récurrence   : Hebdomadaire, mensuelle, trimestrielle, semestrielle, annuelle
Durée        : Nombre de jours pour compléter la revue
Si aucune réponse : Approuver automatiquement ou Refuser automatiquement
```

---

## Bonnes pratiques

| Pratique | Pourquoi |
|---|---|
| **Auto-apply results** | Automatise la suppression des accès non confirmés — pas de ticket manuel |
| **Si aucune réponse → Refuser** | Principe de moindre privilège — le silence = l'accès n'est plus justifié |
| **Revue trimestrielle pour les admins** | Les rôles privilégiés doivent être revus plus souvent |
| **Justification obligatoire** | Crée une piste d'audit et force la réflexion avant d'approuver |
| **Notifications d'escalade** | Relancer les reviewers qui n'ont pas répondu à J-3 |

---

## PowerShell — créer une revue par script

```powershell
Connect-MgGraph -Scopes "AccessReview.ReadWrite.All"

$review = New-MgIdentityGovernanceAccessReviewDefinition -BodyParameter @{
    displayName           = "Revue trimestrielle — Groupe Admins IT"
    descriptionForAdmins  = "Valider les membres du groupe GS-ENTRA-T1"
    scope = @{
        "@odata.type" = "#microsoft.graph.accessReviewQueryScope"
        query         = "/groups/GROUP_ID/members"
        queryType     = "MicrosoftGraph"
    }
    reviewers = @(@{
        query     = "./manager"
        queryType = "MicrosoftGraph"
    })
    settings = @{
        mailNotificationsEnabled    = $true
        reminderNotificationsEnabled = $true
        justificationRequiredOnApproval = $true
        defaultDecision              = "Deny"
        autoApplyDecisionsEnabled    = $true
        recurrence = @{
            pattern = @{ type = "absoluteMonthly"; interval = 3 }
            range   = @{ type = "noEnd"; startDate = "2026-07-01" }
        }
    }
}
```

---

## Suivi et audit

```kql
// Décisions de revue d'accès dans les logs d'audit
AuditLogs
| where TimeGenerated > ago(30d)
| where Category == "AccessReviews"
| extend
    Reviewer = tostring(InitiatedBy.user.userPrincipalName),
    Decision = tostring(TargetResources[0].modifiedProperties[0].newValue),
    Target   = tostring(TargetResources[0].userPrincipalName)
| project TimeGenerated, Reviewer, Target, Decision, OperationName
| where Decision in ("Approve", "Deny")
| summarize Approved=countif(Decision=="Approve"),
            Denied=countif(Decision=="Deny")
            by Reviewer
| order by Denied desc
```
