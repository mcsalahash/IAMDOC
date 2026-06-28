# Tokens — JWT, Opaque, Validation

## Les trois types de tokens

| Token | Format | Durée | Usage |
|---|---|---|---|
| **Access Token** | JWT ou Opaque | 1 heure | Appeler une API |
| **Refresh Token** | Opaque | 90 jours glissants | Renouveler l'access token silencieusement |
| **ID Token** | JWT uniquement | Session | Identité de l'utilisateur (OIDC) |

!!! warning "Ne jamais appeler une API avec l'ID Token"
    L'`id_token` contient l'identité de l'utilisateur — il n'est **pas** destiné à être envoyé à une API. Seul l'`access_token` s'utilise dans le header `Authorization: Bearer`.

---

## Anatomie d'un JWT

Un JWT est composé de trois parties séparées par des points : `header.payload.signature`

=== "Header"
    ```json
    {
      "typ": "JWT",
      "alg": "RS256",
      "kid": "abc123"   // ID de la clé publique dans le JWKS
    }
    ```

=== "Payload (Claims)"
    ```json
    {
      "iss": "https://login.microsoftonline.com/{tenant}/v2.0",
      "aud": "api://my-api-client-id",
      "sub": "Aa1Bb2Cc3...",
      "oid": "object-id-de-l-utilisateur",
      "upn": "salah@dgfla.com",
      "scp": "User.Read Mail.Read",
      "roles": ["Admin", "Writer"],
      "iat": 1718000000,
      "exp": 1718003600,
      "nbf": 1718000000,
      "tid": "tenant-id",
      "ver": "2.0"
    }
    ```

=== "Signature"
    ```
    RSASHA256(
      base64url(header) + "." + base64url(payload),
      clé_privée_Microsoft
    )
    ```
    Vérifiable avec la clé publique disponible sur le JWKS endpoint.

---

## Claims importants à valider côté API

| Claim | Vérification obligatoire |
|---|---|
| `iss` | Doit correspondre à l'issuer Entra ID de ton tenant |
| `aud` | Doit correspondre à l'Application ID URI de ton API |
| `exp` | Timestamp actuel doit être inférieur à `exp` |
| `nbf` | Timestamp actuel doit être supérieur à `nbf` (si présent) |
| `scp` | Permissions déléguées — vérifier le scope attendu |
| `roles` | Permissions d'application — vérifier le rôle attendu |

---

## JWT vs Opaque Token

| | JWT | Opaque |
|---|---|---|
| **Contenu** | Claims lisibles (base64) | Référence opaque |
| **Validation** | Locale — signature + claims | Requête vers l'introspection endpoint |
| **Révocation** | Non immédiate (valable jusqu'à `exp`) | Immédiate |
| **Performances** | Rapide — pas d'appel réseau | Appel réseau à chaque validation |
| **Entra ID** | Access tokens pour Graph / APIs | Refresh tokens toujours opaques |

---

## Validation locale d'un JWT (côté API)

```python
# Python — validation avec la bibliothèque PyJWT
import jwt
from cryptography.hazmat.primitives import serialization

# 1. Récupérer les clés publiques Entra ID
jwks_uri = "https://login.microsoftonline.com/{tenant}/discovery/v2.0/keys"

# 2. Valider le token
decoded = jwt.decode(
    token,
    algorithms=["RS256"],
    audience="api://mon-api-client-id",
    issuer=f"https://login.microsoftonline.com/{tenant}/v2.0",
    options={"verify_exp": True}
)

# 3. Vérifier le scope ou le rôle
assert "User.Read" in decoded.get("scp", "").split()
```

```csharp
// ASP.NET Core — validation automatique avec Microsoft.Identity.Web
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddMicrosoftIdentityWebApi(builder.Configuration, "AzureAd");

// Dans appsettings.json
{
  "AzureAd": {
    "Instance": "https://login.microsoftonline.com/",
    "TenantId": "{tenant-id}",
    "ClientId": "{api-client-id}",
    "Audience": "api://{api-client-id}"
  }
}
```

---

## Durées de vie — Entra ID

| Token | Durée par défaut | Configurable |
|---|---|---|
| Access Token | 1 heure | 10 min — 1 jour (Token Lifetime Policy) |
| Refresh Token (multi-session) | 90 jours glissants | Oui |
| Refresh Token (single-session) | Jusqu'à révocation | Non |
| PRT | 14 jours (renouvelé à l'usage) | Non |
| CAE Access Token | Jusqu'à 24-28h | Non (géré par CAE) |
| ID Token | 1 heure | Non |
