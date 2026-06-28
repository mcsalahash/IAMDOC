# Identity Secure Score

Indicateur chiffré de la posture de sécurité identitaire du tenant — consultable et actionnable.

---

## Accès

```
Portail Entra → Identity → Overview → Recommendations
```

Ou directement : [entra.microsoft.com](https://entra.microsoft.com) → **Identity Secure Score**

---

## Calcul du score

```
Score = Points obtenus / Points maximum atteignables × 100
```

Chaque recommandation a un **poids** et peut être dans l'un des états suivants :

| État | Points |
|---|---|
| Completed | Points pleins |
| Partiellement réalisée | Score proportionnel à la couverture |
| To address | 0 |
| Risk accepted | Exclue du calcul |
| Resolved through third party | Points accordés |

---

## Recommandations types (classées par impact)

| Recommandation | Poids indicatif |
|---|---|
| MFA activée pour tous les utilisateurs | Élevé |
| Blocage de l'authentification legacy | Élevé |
| Activation de l'Identity Protection | Élevé |
| Nombre d'admins globaux entre 2 et 4 | Moyen |
| SSPR activé | Moyen |
| Expiration des secrets d'application | Moyen |
| Sign-in risk policy activée | Moyen |
| Activation de PIM pour les rôles privilégiés | Élevé |

---

## Graph API — récupérer le score

```powershell
# PowerShell — score actuel
$token = (Get-AzAccessToken -ResourceUrl "https://graph.microsoft.com").Token
$headers = @{ Authorization = "Bearer $token" }

$score = Invoke-RestMethod `
    -Uri "https://graph.microsoft.com/v1.0/security/identitySecureScores?`$top=1" `
    -Headers $headers

$score.value[0].currentScore
$score.value[0].maxScore
```

```python
# Python — score et détail des recommandations
import requests

headers = {"Authorization": f"Bearer {access_token}"}

# Score actuel
r = requests.get(
    "https://graph.microsoft.com/v1.0/security/identitySecureScores?$top=1",
    headers=headers
)
data = r.json()["value"][0]
print(f"Score : {data['currentScore']} / {data['maxScore']}")

# Catalogue des recommandations
r2 = requests.get(
    "https://graph.microsoft.com/v1.0/security/secureScoreControlProfiles",
    headers=headers
)
for ctrl in r2.json()["value"]:
    print(f"{ctrl['controlName']} — {ctrl['maxScore']} pts")
```

---

## KQL — suivi de l'évolution

```kql
// Évolution du score sur 90 jours (si connecté à Sentinel)
SecurityResources
| where type == "microsoft.security/securescores"
| where name == "ascScore"
| extend Score = toreal(properties.score.current)
| summarize Score=max(Score) by bin(TimeGenerated, 1d)
| render timechart
```
