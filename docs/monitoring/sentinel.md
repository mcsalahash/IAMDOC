# Microsoft Sentinel — Intégration Entra ID

---

## Connecteurs de données

| Connecteur | Logs importés |
|---|---|
| **Microsoft Entra ID** | SignInLogs, AuditLogs, ProvisioningLogs, RiskyUsers |
| **Microsoft Entra ID Protection** | RiskDetections, RiskyUsers (temps réel) |
| **Microsoft Defender XDR** | IdentityInfo, IdentityLogonEvents, IdentityQueryEvents |

```
Sentinel → Data connectors → Microsoft Entra ID → Connect
→ Sélectionner les tables à importer
→ Verify last data received : vérifier que les logs arrivent
```

---

## Règles analytiques utiles (built-in)

| Règle | Description |
|---|---|
| Suspicious Sign-In Activity | Connexions depuis des pays inattendus |
| MFA Disabled for a User | Désactivation de MFA détectée |
| Bulk Changes to Privileged Accounts | Modifications massives de comptes admin |
| New Admin Account Created | Nouveau compte admin sans PIM |
| Sign-in from IPs flagged by Threat Intelligence | IP malveillantes connues |

---

## Workbook Identity & Access

```
Sentinel → Workbooks → Templates → Identity & Access
→ Save → View saved workbook
```

Contient :
- Carte des connexions par pays
- Top utilisateurs par volume d'authentification
- Taux d'échec MFA par méthode
- Évolution des risques Identity Protection

---

## Règle analytique personnalisée — détection legacy auth

```kql
// Règle : authentification legacy détectée
SigninLogs
| where ClientAppUsed in (
    "Exchange ActiveSync", "IMAP4", "MAPI",
    "POP3", "SMTP", "Authenticated SMTP"
  )
| where ResultType == "0"
| summarize Count = count(), Users = make_set(UserPrincipalName)
    by ClientAppUsed, bin(TimeGenerated, 1h)
| where Count > 0
// → Sévérité : Medium
// → Tactique MITRE : Initial Access
```

---

## Réponse automatique (Playbooks)

```
Incident généré par Sentinel
    → Logic App déclenché
        → Désactiver le compte (via Graph API)
        → Révoquer les sessions (RevokeSignInSessions)
        → Notification Teams / email SOC
```

```json
// Action Graph — révoquer les sessions d'un utilisateur
POST https://graph.microsoft.com/v1.0/users/{userId}/revokeSignInSessions
```
