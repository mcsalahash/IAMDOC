# Protocoles fondamentaux

## OAuth 2.0 vs OpenID Connect

| | OAuth 2.0 | OpenID Connect (OIDC) |
|---|---|---|
| **Objectif** | Autorisation — accès à une ressource | Authentification — identité de l'utilisateur |
| **Ce qu'il émet** | `access_token` | `access_token` + `id_token` |
| **Standard** | RFC 6749 | Couche au-dessus d'OAuth 2.0 |
| **Question répondue** | *Que peut faire cette application ?* | *Qui est l'utilisateur ?* |

!!! tip "En une phrase"
    **OAuth 2.0** résout l'**autorisation**. **OIDC** ajoute l'**authentification** en standardisant un `id_token` JWT par-dessus OAuth 2.0.

---

## Rôles OAuth 2.0

| Rôle | Définition | Exemple Entra ID |
|---|---|---|
| **Resource Owner** | L'utilisateur qui possède les données | L'employé qui se connecte |
| **Client** | L'application qui veut accéder aux données | App React, script PowerShell |
| **Authorization Server** | Émet les tokens après authentification | `login.microsoftonline.com/{tenant}` |
| **Resource Server** | L'API protégée qui consomme le token | Microsoft Graph, ton API Azure |

---

## Public vs Confidential Client

| | Public Client | Confidential Client |
|---|---|---|
| **Peut garder un secret ?** | Non | Oui |
| **Exemples** | SPA, app mobile, desktop | Web server, daemon, Azure Function |
| **Credential** | PKCE uniquement | Secret ou certificat |
| **Flows autorisés** | Auth Code + PKCE, Device Code | Auth Code, Client Credentials, OBO |

!!! warning "SPA = toujours Public Client"
    Le code JavaScript d'une SPA est visible dans le navigateur. Un client secret dans une SPA n'est **pas un secret** — n'importe qui peut l'extraire avec les DevTools.

---

## Scopes

=== "Permissions déléguées"
    L'application agit **au nom de l'utilisateur**. L'utilisateur doit être connecté.
    ```
    User.Read              → lire le profil de l'utilisateur connecté
    Mail.ReadWrite         → lire et écrire ses emails
    Files.Read.All         → lire tous ses fichiers OneDrive
    ```

=== "Permissions d'application"
    L'application agit **en son propre nom** — daemon, service. Pas d'utilisateur.
    ```
    User.Read.All          → lire tous les utilisateurs du tenant
    Mail.Send              → envoyer des emails sans utilisateur
    Directory.ReadWrite.All → lire/écrire l'annuaire
    ```

### Scope `.default`

```
https://graph.microsoft.com/.default
```

Demande toutes les permissions déjà consenties pour cette application. Utilisé principalement dans le Client Credentials Flow.

---

## Endpoints Entra ID

```bash
# Authorization
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize

# Token
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token

# Device Code
https://login.microsoftonline.com/{tenant}/oauth2/v2.0/devicecode

# JWKS — clés publiques pour valider les JWT
https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys

# OpenID Configuration
https://login.microsoftonline.com/{tenant}/v2.0/.well-known/openid-configuration
```

!!! info "Remplace `{tenant}` par"
    - `common` — comptes Microsoft + comptes professionnels
    - `organizations` — comptes professionnels uniquement
    - `{tenant-id}` — ton tenant spécifique **(recommandé en production)**
