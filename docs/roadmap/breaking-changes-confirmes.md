# Breaking Changes confirmés — 2026

Échéances corroborées par la documentation officielle Microsoft Learn / Tech Community.

!!! warning "Toujours vérifier avant d'agir"
    Les calendriers Microsoft font l'objet de reports fréquents. Vérifier systématiquement le [Message Center M365](https://admin.microsoft.com) et [What's new in Entra](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new) avant toute planification ferme.

---

## Calendrier

| Échéance | Changement | Action requise |
|---|---|---|
| **31 mars 2026** | Applications sans service principal associé voient leur flux de connexion échouer | Identifier via les Sign-in Logs, provisionner un service principal |
| **16 mars 2026** | Retrait Identity Protection pour Azure AD B2C | Migrer vers un fournisseur alternatif |
| **1 mai 2026** | Retrait des blades "Agent registry" et "Agent collections" — Agent 365 devient le registre unifié | Aucune action, fonctionnalité conservée ailleurs |
| **30 juin 2026** *(reporté depuis mars)* | Retrait du grant control CA "Require approved client app" — remplacé par "Require app protection policy" (Intune) | Migrer les politiques CA utilisant ce contrôle |
| **6 juillet 2026** | CA appliqué lors de l'enregistrement des infos de sécurité — inclut WHfB et macOS Platform SSO | Tester les policies CA ciblant "Register security information" en report-only |
| **7 septembre 2026** | SSPR n'accepte plus que les méthodes explicitement enregistrées (fin usage des attributs directory) | Lancer une campagne d'enregistrement des méthodes MFA (Microsoft démarre le 6 juillet) |
| **30 septembre 2026** | Retrait des Custom Controls CA (MFA tierce) — remplacés par External MFA. EOL complet mai 2027 | Recenser les policies avec Custom Controls, migrer vers External MFA |
| **Fin déc. 2026** | Désactivation par défaut Basic Auth SMTP AUTH (Client Submission) sur Exchange Online | Inventorier via rapport SMTP AUTH, migrer vers OAuth 2.0 SMTP ou Microsoft Graph |

---

## Actions par priorité

### Immédiat

```powershell
# 1. Identifier les apps sans service principal (avant 31 mars)
Get-MgApplication | ForEach-Object {
    $sp = Get-MgServicePrincipal -Filter "appId eq '$($_.AppId)'" -ErrorAction SilentlyContinue
    if (-not $sp) {
        [PSCustomObject]@{
            AppName = $_.DisplayName
            AppId = $_.AppId
            CreatedDateTime = $_.CreatedDateTime
        }
    }
} | Export-Csv "apps-sans-sp.csv"
```

### Court terme

```kql
// Identifier les politiques CA utilisant "Require approved client app"
// (à migrer avant 30 juin)
// → Portail Entra → Protection → Conditional Access → filtrer par "Require approved client app"
```

```powershell
# Identifier les expéditeurs Basic Auth SMTP (avant fin déc. 2026)
# Exchange Admin Center → Reports → Mail flow → SMTP AUTH clients report
```

### Moyen terme

```powershell
# Lancer la campagne d'enregistrement SSPR/MFA (avant 7 sept.)
# Portail Entra → Identity → Users → Registration campaign
```

---

## Retrait Basic Auth SMTP — Impact et migration

| Protocole | Statut | Alternative |
|---|---|---|
| SMTP AUTH (Basic) | Désactivé par défaut fin 2026, retiré H2 2027 | OAuth 2.0 SMTP, High Volume Email, Microsoft Graph API |
| EWS (Basic) | Déjà retiré | Microsoft Graph |
| ActiveSync (Basic) | Déjà retiré | Modern Auth |
| IMAP/POP3 (Basic) | Déjà retiré | IMAP/POP3 OAuth |

```powershell
# Désactiver Basic Auth SMTP sur un mailbox spécifique (test)
Set-CASMailbox -Identity "user@dgfla.com" -SmtpClientAuthenticationDisabled $true

# Réactiver temporairement si nécessaire pendant migration
Set-CASMailbox -Identity "user@dgfla.com" -SmtpClientAuthenticationDisabled $false
```
