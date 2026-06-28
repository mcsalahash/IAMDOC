# Identity Secure Score

Voir la page dédiée dans la section Entra ID → [Identity Secure Score](../entra-id/secure-score.md)

Cette page couvre l'accès et le suivi du score depuis un contexte monitoring / opérations.

---

## Accès rapide

| Accès | URL |
|---|---|
| Portail Entra | [entra.microsoft.com → Identity → Overview](https://entra.microsoft.com/#view/Microsoft_AAD_IAM/TenantOverview.ReactView) |
| Microsoft Secure Score (global) | [security.microsoft.com](https://security.microsoft.com/securescore) |

---

## Suivi via Graph API

```powershell
# Score actuel
Connect-MgGraph -Scopes "SecurityEvents.Read.All"
$scores = Invoke-MgGraphRequest -Method GET `
    -Uri "https://graph.microsoft.com/v1.0/security/identitySecureScores?`$top=1"
$scores.value[0] | Select-Object currentScore, maxScore, createdDateTime
```

---

## Alerte sur dégradation du score

```kql
// Log Analytics — alerte si le score baisse de plus de 5 points
SecurityResources
| where type == "microsoft.security/securescores"
| extend Score = toreal(properties.score.current)
| order by TimeGenerated desc
| serialize
| extend PreviousScore = prev(Score, 1)
| where Score < PreviousScore - 5
| project TimeGenerated, Score, PreviousScore, Drop = PreviousScore - Score
```
