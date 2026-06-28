# Breaking Changes à vérifier

!!! danger "Sources non officielles"
    Ces dates proviennent de blogs, contenus de veille ou communications tierces. Elles **n'ont pas été confirmées par Microsoft Learn** à la date de rédaction. À vérifier impérativement avant toute action.

**Source officielle de référence :**
- [Message Center M365](https://admin.microsoft.com) → Menu Messages
- [What's new in Microsoft Entra](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new)
- [Microsoft 365 Roadmap](https://www.microsoft.com/en-us/microsoft-365/roadmap)

---

## Dates rapportées

| Échéance rapportée | Changement annoncé | Niveau de confiance |
|---|---|---|
| **15 janvier 2026** | Désactivation de l'authentification legacy au niveau tenant | Non corroboré officiellement |
| **1er février 2026** | MFA obligatoire pour les rôles d'administration sur le portail Azure | Partiellement recoupé — initiative Microsoft confirmée, date précise non vérifiée |
| **31 mars 2026** | Retrait global du Basic Auth (au-delà du périmètre SMTP confirmé) | Non corroboré — la retraite SMTP officielle est fixée à fin déc. 2026, pas mars |
| **14 avril 2026** | Changements dans les templates de politiques Conditional Access | Non vérifié |
| **1er octobre 2026** | CAE rendue obligatoire pour tous les tenants | Non corroboré — aucune mention officielle d'un mandat généralisé à cette date |
| **1er octobre 2026** | Retrait des politiques de risque basées uniquement sur l'adresse IP | Non vérifié |

---

## Comment vérifier une date

```powershell
# Vérifier les annonces qui concernent ton tenant via Graph
Connect-MgGraph -Scopes "ServiceMessage.Read.All"

# Messages du Message Center des 90 derniers jours
Get-MgServiceAnnouncementMessage -Filter "lastModifiedDateTime gt 2026-04-01" |
    Where-Object { $_.Tags -contains "Updated" -or $_.Tags -contains "Feature update" } |
    Select-Object Id, Title, StartDateTime, EndDateTime |
    Sort-Object StartDateTime
```

---

## Checklist de vigilance

- [ ] Vérifier le Message Center M365 toutes les 2 semaines
- [ ] S'abonner aux notifications du [Microsoft Tech Community — Entra ID](https://techcommunity.microsoft.com/category/microsoft-entra-id)
- [ ] Tester l'impact de chaque breaking change annoncé en environnement pilote avant la date d'application
- [ ] Maintenir un tableau de suivi des MC (Message Center) avec statut de remédiation
