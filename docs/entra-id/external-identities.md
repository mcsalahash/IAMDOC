# External Identities — B2B / B2C

---

## B2B Collaboration (invités)

Les utilisateurs externes gardent **leur propre identité** et accèdent aux ressources partagées.

```
Flux d'invitation :
1. Admin/utilisateur invite partenaire@externe.com
2. Entra ID crée un compte Guest dans le tenant
3. L'invité reçoit un email avec un lien de rédemption
4. À la connexion, l'invité s'authentifie via son IdP d'origine
5. Entra ID émet un token d'accès pour les ressources partagées
```

```powershell
# Inviter un utilisateur externe
New-MgInvitation `
    -InvitedUserEmailAddress "partenaire@cabinet-externe.fr" `
    -InviteRedirectUrl "https://myapps.microsoft.com" `
    -InvitedUserDisplayName "Jean Dupont (Cabinet Externe)" `
    -SendInvitationMessage:$true `
    -InvitedUserMessageInfo @{
        MessageLanguage = "fr-FR"
        CustomizedMessageBody = "Bonjour, vous êtes invité à accéder à notre espace collaboratif."
    }
```

### Méthodes d'authentification des invités

| Méthode | Quand |
|---|---|
| Entra ID de l'organisation externe | L'invité a un compte professionnel Entra |
| Compte Microsoft | L'invité a un compte @outlook.com / @hotmail.com |
| Google Federation | L'invité a un compte Gmail |
| Email OTP | L'invité n'a aucun des comptes ci-dessus |
| SAML/WS-Fed (fédération directe) | L'invité vient d'une org avec IdP tiers |

---

## Paramètres d'accès externe (Cross-tenant access)

```
Portail Entra → External Identities → Cross-tenant access settings
```

| Paramètre | Description |
|---|---|
| Inbound defaults | Qui peut être invité dans ton tenant |
| Outbound defaults | Tes utilisateurs peuvent-ils être invités ailleurs |
| Tenant-specific settings | Règles par tenant partenaire |
| Microsoft cloud settings | Accès vers/depuis Azure Government, China |

---

## Conditional Access pour les invités

```json
{
  "conditions": {
    "users": {
      "includeGuestsOrExternalUsers": {
        "guestOrExternalUserTypes": "internalGuest,b2bCollaborationGuest",
        "externalTenants": { "membershipKind": "all" }
      }
    }
  },
  "grantControls": {
    "builtInControls": ["mfa"]
  }
}
```

---

## B2B vs B2C

| | B2B Collaboration | Azure AD B2C |
|---|---|---|
| **Cible** | Partenaires professionnels | Clients grand public |
| **Identité** | Gardée chez l'invité | Gérée dans B2C |
| **IdPs supportés** | Entra ID, Google, fédération SAML | 50+ IdPs sociaux |
| **Cas d'usage** | Portail partenaires, accès fournisseurs | Application mobile, site e-commerce |
| **Licence** | Entra ID P1/P2 | Tarification à l'authentification |
