# Token Protection — anti-AiTM

Lie cryptographiquement un token à l'appareil qui l'a demandé, rendant son rejeu impossible depuis une autre machine même en cas de vol.

---

## La menace : Adversary-in-the-Middle (AiTM)

Une attaque AiTM fonctionne ainsi :

1. L'attaquant crée un proxy de phishing identique au portail de connexion
2. La victime saisit ses identifiants **et complète la MFA** sur le site frauduleux
3. Le proxy transmet tout en temps réel à Entra ID et récupère le cookie/token de session
4. L'attaquant **rejoue ce token depuis sa propre machine** — la MFA a déjà été satisfaite

!!! danger "MFA ne protège pas contre AiTM"
    L'attaquant ne contourne pas la MFA — il laisse la victime la compléter pour lui. Le token volé est pleinement valide. C'est exactement ce que Token Protection résout.

---

## Comment Token Protection y répond

```
Device de l'utilisateur
├── TPM / Secure Enclave génère une paire de clés
├── Clé privée → jamais exportable, jamais accessible en dehors du TPM
└── Clé publique → communiquée à Entra ID

Émission du token
├── Entra ID lie l'access_token à la clé publique du device
└── Toute requête doit être signée avec la clé privée correspondante

Sur le Resource Provider
├── Vérifie la preuve cryptographique (Proof-of-Possession)
└── Sans clé privée → token rejeté, même si le token lui-même est valide
```

---

## Prérequis

| Élément | Détail |
|---|---|
| Système d'exploitation | Windows 10/11 |
| Enregistrement device | Entra joined ou Hybrid Entra joined |
| Matériel | TPM 2.0 fonctionnel |
| Client | Applications modernes avec WAM broker (Outlook, Teams, Office) |
| Licence | Microsoft Entra ID P1 minimum |

---

## Activation via Conditional Access

Token Protection s'active comme **contrôle de session** dans une politique CA.

**Portail Entra → Protection → Conditional Access → Nouvelle politique**

```
Utilisateurs        : groupe pilote (ex: DSI)
Applications        : Exchange Online, SharePoint Online
Session             : Require token protection for sign-in sessions ✓
Activer la policy   : Report-only (d'abord)
```

### Déploiement recommandé

| Étape | Action |
|---|---|
| 1 | Déployer en **Report-only** sur le groupe pilote |
| 2 | Analyser les Sign-in Logs — filtre : `Token Protection = Failure` |
| 3 | Identifier les applications incompatibles et les exclure si nécessaire |
| 4 | Basculer en mode **On** une fois le périmètre validé |

---

## KQL — Surveiller les échecs Token Protection

```kql
SigninLogs
| where TimeGenerated > ago(7d)
| where ConditionalAccessStatus == "failure"
| where ConditionalAccessPolicies has "token protection"
| project TimeGenerated, UserPrincipalName, AppDisplayName,
          DeviceDetail, IPAddress, ResultDescription
| order by TimeGenerated desc
```

---

## Complémentarité avec CAE

| | Token Protection | CAE |
|---|---|---|
| **Protège contre** | Rejeu d'un token volé (autre device) | Token utilisé après événement critique |
| **Mécanisme** | Liaison cryptographique au TPM | Révocation événementielle |
| **Licence** | Entra ID P1 | Free (événements critiques) / P1 (CA policies) |
| **Couvre** | Pendant toute la durée de vie du token | Après un événement détecté |

Voir aussi : [CAE — Continuous Access Evaluation](cae.md)
