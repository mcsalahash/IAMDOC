# Plan d'action — 2026

Priorisation des actions de remédiation basée sur les breaking changes et bonnes pratiques.

---

## Matrice de priorité

| Priorité | Critère |
|---|---|
| 🔴 Immédiat | Échéance dépassée ou dans moins de 30 jours |
| 🟠 Court terme | Échéance dans 30 à 90 jours |
| 🟡 Moyen terme | Échéance dans 90 à 180 jours |
| 🟢 Continu | Bonnes pratiques sans date limite |

---

## Actions par priorité

### 🔴 Immédiat

| Action | Référence | Commande de diagnostic |
|---|---|---|
| Vérifier les apps sans service principal | [Breaking changes confirmés](breaking-changes-confirmes.md) | `Get-MgApplication \| Where-Object { -not (Get-MgServicePrincipal -Filter "appId eq '$($_.AppId)'")}` |
| Identifier les politiques CA avec "Require approved client app" | [Breaking changes confirmés](breaking-changes-confirmes.md) | Portail Entra → CA → filtrer par grant control |

### 🟠 Court terme

| Action | Référence | Commande de diagnostic |
|---|---|---|
| Inventorier les expéditeurs SMTP Basic Auth | [Breaking changes confirmés](breaking-changes-confirmes.md) | Exchange Admin Center → Reports → SMTP AUTH clients |
| Recenser les Custom Controls CA | [Breaking changes confirmés](breaking-changes-confirmes.md) | Portail Entra → CA → Custom Controls |
| Lancer campagne enregistrement SSPR/MFA | [Breaking changes confirmés](breaking-changes-confirmes.md) | Portail Entra → Users → Registration campaign |

### 🟡 Moyen terme

| Action | Référence | Commande de diagnostic |
|---|---|---|
| Déployer Token Protection (report-only) | [Token Protection](../entra-id/token-protection.md) | KQL : `SigninLogs \| where ConditionalAccessStatus == "reportOnly"` |
| Activer CAE sur les applications MSAL | [CAE](../entra-id/cae.md) | Vérifier `xms_cc` dans les tokens |
| Configurer les Access Reviews pour les rôles admin | [Access Reviews](../gouvernance/access-reviews.md) | Identity Governance → Access Reviews |

### 🟢 Continu

| Action | Fréquence | Référence |
|---|---|---|
| Revue du Message Center M365 | Hebdomadaire | [admin.microsoft.com](https://admin.microsoft.com) |
| Suivi de l'Identity Secure Score | Mensuel | [Identity Secure Score](../entra-id/secure-score.md) |
| Audit des secrets d'application expirant | Mensuel | `Get-MgApplication \| Select PasswordCredentials` |
| Revue des rôles admin actifs (PIM) | Trimestriel | [PIM](../entra-id/pim.md) |
| Access Reviews — groupes et applications | Trimestriel | [Access Reviews](../gouvernance/access-reviews.md) |

---

## Template de suivi (à copier dans ton outil de gestion)

```
ID      : MC-YYYYMMDD-XXX
Titre   : [Description du changement]
Source  : Message Center / Microsoft Learn / Blog
Échéance: YYYY-MM-DD
Impact  : [Faible / Moyen / Élevé]
Statut  : [À évaluer / En cours / Résolu / Non applicable]
Action  : [Description de l'action]
Resp.   : [Responsable]
Testé   : Oui / Non
Date résolution : YYYY-MM-DD
```
