# Conditional Access

Moteur de politiques de sécurité Zero Trust d'Entra ID — évalue chaque tentative d'accès selon des signaux contextuels et applique les contrôles appropriés.

---

## Modèle If/Then

```
SI   (Conditions)        ALORS   (Contrôles)
─────────────────────────────────────────────
Utilisateurs              Accorder l'accès
Applications                → Nécessite MFA
Plateforme                  → Nécessite device conforme
Localisation réseau         → Nécessite app protection policy
Risque (user/sign-in)       → Nécessite changement de MDP
État du device            Bloquer l'accès
Heure d'accès             Limiter la session
```

---

## Structure d'une politique CA

```json
{
  "displayName": "Require MFA for All Users",
  "state": "enabled",
  "conditions": {
    "users": {
      "includeUsers": ["All"],
      "excludeGroups": ["GS-CA-BreakGlass"]
    },
    "applications": {
      "includeApplications": ["All"]
    },
    "locations": {
      "excludeLocations": ["AllTrustedLocations"]
    }
  },
  "grantControls": {
    "operator": "OR",
    "builtInControls": ["mfa"]
  }
}
```

---

## Signaux évalués

| Signal | Exemples |
|---|---|
| **Identité** | Rôle admin, groupe, type de compte (invité/membre) |
| **Application** | Exchange Online, SharePoint, app personnalisée |
| **Localisation** | Named Location (IP/pays), réseau de confiance |
| **Device** | Compliant (Intune), Hybrid Entra joined, plateforme (iOS/Android/Windows) |
| **Risque** | Risque sign-in (faible/moyen/élevé), risque utilisateur (Identity Protection) |
| **Session** | Fréquence de connexion, persistance de session |

---

## Contrôles disponibles

=== "Grant Controls (accorder ou bloquer)"
    | Contrôle | Description |
    |---|---|
    | **Require MFA** | Authentification forte obligatoire |
    | **Require compliant device** | Device doit être conforme dans Intune |
    | **Require Hybrid Entra joined** | Device joint à l'AD on-prem et Entra |
    | **Require approved client app** | Retiré — remplacé par App Protection Policy |
    | **Require app protection policy** | Policy Intune MAM requise |
    | **Require password change** | Déclenché sur risque utilisateur élevé |
    | **Block access** | Blocage total |

=== "Session Controls (contrôler la session)"
    | Contrôle | Description |
    |---|---|
    | **Sign-in frequency** | Forcer une ré-authentification toutes les X heures |
    | **Persistent browser session** | Contrôle du "Rester connecté" |
    | **App enforced restrictions** | Délègue le contrôle à l'app (SharePoint, Exchange) |
    | **Conditional Access App Control** | Proxy MCAS pour contrôle fin |
    | **Token protection** | Lie le token au device (anti-AiTM) |

---

## Authentication Strengths

Définit le **niveau de MFA requis** au-delà du simple "MFA oui/non".

| Strength | Méthodes acceptées |
|---|---|
| **Multifactor authentication** | Toutes les méthodes MFA (SMS inclus) |
| **Passwordless MFA** | Authenticator, FIDO2, Windows Hello |
| **Phishing-resistant MFA** | FIDO2 et Windows Hello uniquement |

!!! tip "Phishing-resistant MFA"
    FIDO2 et Windows Hello sont liés au domaine — ils ne peuvent pas être utilisés sur un site de phishing. C'est la protection la plus forte contre les attaques AiTM au niveau de la MFA.

---

## Bonnes pratiques

!!! danger "Toujours exclure les Break Glass accounts"
    Avant d'activer une politique, vérifier que les comptes Break Glass sont dans le groupe d'exclusion. Une politique mal configurée qui bloque les admins = lockout total du tenant.

!!! warning "Report-only avant activation"
    Activer toute nouvelle politique en **Report-only** et analyser l'impact dans les Sign-in Logs pendant 24-48h avant de passer en **On**.

```kql
-- KQL : Simuler l'impact d'une politique CA en Report-only
SigninLogs
| where TimeGenerated > ago(24h)
| where ConditionalAccessStatus == "reportOnly"
| where ConditionalAccessPolicies has "MA POLITIQUE"
| summarize Count=count() by ResultDescription, UserPrincipalName
| order by Count desc
```

---

## KQL — Audit des politiques CA

```kql
// Connexions bloquées par CA
SigninLogs
| where TimeGenerated > ago(7d)
| where ConditionalAccessStatus == "failure"
| where ResultType == "53003"  // BlockedByConditionalAccess
| project TimeGenerated, UserPrincipalName, AppDisplayName,
          IPAddress, DeviceDetail, ConditionalAccessPolicies
| order by TimeGenerated desc
```
