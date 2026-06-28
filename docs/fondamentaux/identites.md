# Types d'identités

Entra ID gère plusieurs catégories d'identités, chacune avec ses caractéristiques et son modèle de sécurité.

---

## Vue d'ensemble

```
Identités dans Entra ID
├── Identités humaines
│   ├── Membres (comptes internes)
│   └── Invités / Guests (B2B)
└── Identités non-humaines (NHI)
    ├── Service Principals (applications)
    └── Managed Identities (ressources Azure)
```

---

## Comptes utilisateurs membres

Comptes créés directement dans le tenant ou synchronisés depuis Active Directory.

| Attribut | Description |
|---|---|
| `UserPrincipalName` | Identifiant de connexion — `salah@dgfla.com` |
| `ObjectId` | Identifiant unique immuable dans Entra ID |
| `UserType` | `Member` pour les comptes internes |
| `AccountEnabled` | `true` / `false` |
| `OnPremisesSyncEnabled` | `true` si synchronisé depuis AD on-prem |

---

## Invités B2B (Guest)

Utilisateurs externes invités à collaborer — ils gardent leur identité d'origine.

| Aspect | Détail |
|---|---|
| `UserType` | `Guest` |
| Authentification | Via leur IdP d'origine (autre tenant, Google, email OTP) |
| Accès | Limité aux ressources explicitement partagées |
| Rédemption | L'invité doit accepter l'invitation au premier accès |

```powershell
# Inviter un utilisateur externe
New-MgInvitation -InvitedUserEmailAddress "partenaire@externe.com" `
                 -InviteRedirectUrl "https://myapps.microsoft.com" `
                 -SendInvitationMessage:$true
```

---

## Service Principals (Applications)

Une application dans Entra ID a deux représentations :

| | App Registration | Enterprise App / Service Principal |
|---|---|---|
| **Quoi** | La définition de l'application | La représentation dans un tenant |
| **Où** | Un seul tenant (l'éditeur) | Chaque tenant qui utilise l'app |
| **Contient** | Client ID, secrets, redirect URIs, permissions déclarées | Assignations, politiques CA, consentement |
| **Analogie** | Le moule | La pièce coulée |

```powershell
# Créer une App Registration
$app = New-MgApplication -DisplayName "Mon API" `
    -SignInAudience "AzureADMyOrg"

# Créer le Service Principal associé
New-MgServicePrincipal -AppId $app.AppId
```

---

## Managed Identities

Identités gérées par Azure pour les ressources Azure — **aucun credential à gérer**.

| Type | Durée de vie | Partage entre ressources |
|---|---|---|
| **System-assigned** | Liée à la ressource — supprimée avec elle | Non |
| **User-assigned** | Indépendante — cycle de vie géré manuellement | Oui |

```bash
# Activer une Managed Identity sur une VM Azure
az vm identity assign --name ma-vm --resource-group mon-rg

# Activer sur un App Service
az webapp identity assign --name mon-app --resource-group mon-rg
```

!!! tip "Quand utiliser User-assigned vs System-assigned"
    - **System-assigned** : quand la ressource est unique et ne partage pas son identité
    - **User-assigned** : quand plusieurs ressources ont besoin des mêmes permissions (ex: plusieurs App Services accédant au même Key Vault)

---

## Groupes

| Type | Membres | Assignation dynamique |
|---|---|---|
| **Security** | Utilisateurs, devices, SPs | Oui (règles dynamiques) |
| **Microsoft 365** | Utilisateurs uniquement | Non |
| **Distribution** | Utilisateurs (héritage Exchange) | Non |

```powershell
# Groupe dynamique — tous les utilisateurs du département IT
New-MgGroup -DisplayName "GS-IT-Dept" `
    -GroupTypes @("DynamicMembership") `
    -MembershipRule "(user.department -eq ""IT"")" `
    -MembershipRuleProcessingState "On" `
    -SecurityEnabled:$true `
    -MailEnabled:$false `
    -MailNickname "GS-IT-Dept"
```
