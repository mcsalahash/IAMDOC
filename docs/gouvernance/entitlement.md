# Entitlement Management

Gestion des accès à la demande via des **access packages** — groupes, applications et sites SharePoint regroupés en un bundle accessible via un workflow d'approbation.

---

## Concepts clés

| Terme | Définition |
|---|---|
| **Catalog** | Conteneur d'access packages — par département, projet ou domaine |
| **Access Package** | Bundle de ressources (groupes + apps + SharePoint) avec une politique d'accès |
| **Policy** | Qui peut demander, qui approuve, combien de temps l'accès dure |
| **Assignment** | Instance d'un access package accordé à un utilisateur |
| **Connected Organization** | Tenant externe autorisé à demander des access packages |

---

## Cas d'usage typiques

- **Onboarding projet** : un access package "Projet X" donne accès au groupe Teams, au site SharePoint et à l'application de gestion de projet — un seul clic pour l'utilisateur, une seule approbation pour le manager
- **Accès fournisseur** : un partenaire externe demande un accès limité à 6 mois, approuvé par le responsable métier, révoqué automatiquement à l'échéance
- **Accès temporaire escaladé** : accès à des données sensibles pour 48h, nécessitant une double approbation

---

## Créer un access package

```
Portail Entra → Identity Governance → Entitlement Management
→ Access packages → New access package

1. Basics        : nom, description, catalog
2. Resource roles : groupes, apps, sites SharePoint à inclure
3. Requests      : qui peut demander (membres, invités, organisation connectée)
4. Requestor info : questions à poser lors de la demande
5. Approval      : approbateurs, délai de réponse, approbation en cascade
6. Lifecycle     : durée d'accès, expiration, révisions d'accès périodiques
```

---

## Politiques d'accès

```json
{
  "requestorSettings": {
    "scopeType": "AllExistingDirectoryMemberUsers",
    "acceptRequests": true
  },
  "requestApprovalSettings": {
    "isApprovalRequired": true,
    "approvalStages": [{
      "approvalStageTimeOutInDays": 7,
      "isApproverJustificationRequired": true,
      "primaryApprovers": [{ "@odata.type": "#microsoft.graph.groupMembers",
                             "groupId": "MANAGERS_GROUP_ID" }]
    }]
  },
  "accessReviewSettings": {
    "isEnabled": true,
    "recurrenceType": "quarterly",
    "reviewerType": "Manager"
  }
}
```

---

## Portail My Access

Les utilisateurs accèdent à leurs packages via **myaccess.microsoft.com** :

```
https://myaccess.microsoft.com/@{tenant-domain}
```

---

## PowerShell — audit des assignments

```powershell
Connect-MgGraph -Scopes "EntitlementManagement.Read.All"

# Tous les assignments actifs
Get-MgEntitlementManagementAssignment -Filter "state eq 'delivered'" |
    Select-Object Id, @{N="User";E={$_.Principal.DisplayName}},
                      @{N="Package";E={$_.AccessPackage.DisplayName}},
                      ScheduledEndDateTime |
    Export-Csv "entitlement-assignments.csv"
```
