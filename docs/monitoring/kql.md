# KQL — Requêtes de référence

Requêtes KQL prêtes à l'emploi pour l'investigation et le monitoring Entra ID.

---

## Connexions échouées

```kql
// Top 20 des erreurs d'authentification sur 7 jours
SigninLogs
| where TimeGenerated > ago(7d)
| where ResultType != "0"
| summarize Count = count() by ResultType, ResultDescription, AppDisplayName
| order by Count desc
| take 20
```

```kql
// Connexions bloquées par Conditional Access
SigninLogs
| where TimeGenerated > ago(24h)
| where ConditionalAccessStatus == "failure"
| where ResultType == "53003"
| project TimeGenerated, UserPrincipalName, AppDisplayName,
          IPAddress, ConditionalAccessPolicies
| order by TimeGenerated desc
```

---

## Authentification héritée (Legacy Auth)

```kql
// Détecter l'usage de Legacy Authentication
SigninLogs
| where TimeGenerated > ago(30d)
| where ClientAppUsed in (
    "Exchange ActiveSync", "IMAP4", "MAPI", "POP3",
    "SMTP", "Authenticated SMTP", "Exchange Web Services",
    "Other clients", "Exchange Online PowerShell"
  )
| summarize Count = count() by UserPrincipalName, ClientAppUsed, AppDisplayName
| order by Count desc
```

---

## MFA

```kql
// Taux de réussite MFA par méthode
SigninLogs
| where TimeGenerated > ago(7d)
| where AuthenticationRequirement == "multiFactorAuthentication"
| summarize
    Total = count(),
    Success = countif(ResultType == "0")
  by tostring(MfaDetail.authMethod)
| extend TauxReussite = round(100.0 * Success / Total, 1)
| order by Total desc
```

```kql
// Utilisateurs sans MFA enregistrée
AADUserRegistrationDetails
| where IsMfaRegistered == false
| where IsAdmin == false
| project UserPrincipalName, IsPasswordlessCapable, IsMfaRegistered
```

---

## Comptes à risque (Identity Protection)

```kql
// Utilisateurs avec risque élevé actif
AADRiskyUsers
| where RiskLevel == "high"
| where RiskState == "atRisk"
| project UserPrincipalName, RiskLevel, RiskLastUpdatedDateTime, RiskDetail
| order by RiskLastUpdatedDateTime desc
```

```kql
// Événements de risque récents
AADUserRiskEvents
| where TimeGenerated > ago(7d)
| where RiskLevel in ("high", "medium")
| project TimeGenerated, UserPrincipalName, RiskEventType,
          RiskLevel, RiskState, IPAddress, Location
| order by TimeGenerated desc
```

---

## Audit des rôles privilégiés

```kql
// Attributions de rôles admin sur 30 jours
AuditLogs
| where TimeGenerated > ago(30d)
| where Category == "RoleManagement"
| where OperationName in (
    "Add member to role",
    "Add eligible member to role"
  )
| extend
    Actor = tostring(InitiatedBy.user.userPrincipalName),
    Target = tostring(TargetResources[0].userPrincipalName),
    Role = tostring(TargetResources[0].modifiedProperties[1].newValue)
| project TimeGenerated, Actor, Target, Role, OperationName
| order by TimeGenerated desc
```

---

## App Registrations — secrets expirant

```kql
// Secrets et certificats expirant dans les 30 prochains jours
AADServicePrincipalSignInLogs
| join kind=inner (
    MicrosoftGraphActivityLogs
    | where RequestUri has "keyCredentials"
  ) on $left.ServicePrincipalId == $right.UniqueTokenIdentifier
```

```kql
// Via Microsoft Graph (PowerShell) — plus fiable pour les secrets
// Get-MgApplication | foreach {
//   $_.PasswordCredentials | where EndDateTime -lt (Get-Date).AddDays(30)
// }
```

---

## Token Protection

```kql
// Échecs Token Protection
SigninLogs
| where TimeGenerated > ago(7d)
| where ConditionalAccessStatus == "failure"
| mv-expand ConditionalAccessPolicies
| where ConditionalAccessPolicies.enforcedSessionControls has "SignInTokenProtection"
| project TimeGenerated, UserPrincipalName, AppDisplayName,
          DeviceDetail, IPAddress
| order by TimeGenerated desc
```

---

## Codes d'erreur fréquents

| Code | Signification | Action |
|---|---|---|
| `50058` | Cookie de session silencieuse expiré | Normal — ré-auth requise |
| `50076` | MFA requise par policy | Vérifier la policy CA |
| `50079` | MFA forcée (état sécurité changé) | Vérifier Identity Protection |
| `50105` | Utilisateur non assigné à l'app | Assigner l'utilisateur dans Enterprise App |
| `53003` | Bloqué par Conditional Access | Analyser la policy CA applicable |
| `65001` | Consentement requis | Admin consent ou consentement utilisateur manquant |
| `700016` | Application introuvable dans le tenant | Vérifier le client_id |
| `7000215` | Client secret invalide ou expiré | Renouveler le secret |
