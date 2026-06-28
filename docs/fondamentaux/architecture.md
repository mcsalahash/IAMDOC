# Architecture Entra ID

---

## Composants fondamentaux

```
Microsoft Entra ID Tenant
├── Directory (annuaire)
│   ├── Utilisateurs & Groupes
│   ├── Applications (App Registrations + Enterprise Apps)
│   ├── Devices (enregistrés / joints)
│   └── Rôles & Permissions
├── Authentication (STS — Security Token Service)
│   ├── OAuth 2.0 / OIDC endpoint
│   ├── SAML endpoint
│   └── WS-Federation endpoint
├── Authorization
│   ├── Conditional Access Engine
│   ├── PIM (Privileged Identity Management)
│   └── Identity Protection
└── Synchronisation
    ├── Entra Connect (Sync depuis AD on-prem)
    └── SCIM (Provisioning vers apps tierces)
```

---

## Niveaux de licence

| Tier | Fonctionnalités clés |
|---|---|
| **Free** | SSO basique, MFA (Authenticator), CAE événements critiques, SSPR |
| **P1** | Conditional Access, PIM basique, Identity Protection, Hybrid Identity |
| **P2** | Identity Protection avancée (risques), PIM complet, Access Reviews |
| **Gouvernance** | Entitlement Management, Lifecycle Workflows avancés |

---

## Types d'enregistrement des devices

| Type | Description | Cas d'usage |
|---|---|---|
| **Entra Registered** | Device personnel enregistré (BYOD) | iOS/Android/Windows perso |
| **Entra Joined** | Device joint uniquement à Entra ID (cloud-only) | Postes Windows 10/11 modernes |
| **Hybrid Entra Joined** | Jointure simultanée AD on-prem + Entra ID | Migration progressive depuis AD |

```powershell
# Vérifier l'état d'enregistrement d'un device Windows
dsregcmd /status
# Voir : AzureAdJoined, DomainJoined, WorkplaceJoined
```

---

## Hiérarchie des tenants Microsoft

```
Azure Active Directory (Entra ID) Tenant
│
├── Management Groups
│   └── Subscriptions Azure
│       └── Resource Groups
│           └── Ressources Azure
│
└── Microsoft 365
    ├── Exchange Online
    ├── SharePoint Online
    ├── Teams
    └── Intune
```

!!! info "Un tenant = une instance Entra ID"
    Toutes les ressources Azure et M365 d'une organisation partagent le même tenant Entra ID. Les comptes utilisateurs et les politiques CA s'appliquent à l'ensemble.

---

## Endpoints de référence

```bash
# Remplacer {tenant} par le tenant ID ou le domaine (ex: dgfla.com)

# OpenID Configuration (metadata complet)
https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration

# Authorization
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize

# Token
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token

# Clés publiques JWKS
https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys

# Microsoft Graph
https://graph.microsoft.com/v1.0/

# Azure Resource Manager
https://management.azure.com/
```
