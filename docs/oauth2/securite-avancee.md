# Sécurité avancée — JAR, PAR, DPoP

Extensions OAuth 2.0 qui renforcent la sécurité du protocole lui-même, au-delà de ce que PKCE couvre.

---

## Le problème : la requête d'autorisation est vulnérable

Dans un flow Auth Code standard, la requête d'autorisation passe dans l'URL du navigateur :

```
GET /authorize?client_id=...&redirect_uri=https://monapp.com/callback&scope=...&state=xyz
```

Un attaquant peut :
- **Modifier le `redirect_uri`** pour rediriger le code vers son propre serveur
- **Injecter des paramètres** via un lien malveillant
- **Capturer l'URL** dans les logs proxy / historique navigateur

JAR, PAR et JWE résolvent ces problèmes à différents niveaux.

---

## JAR — JWT Secured Authorization Request (RFC 9101)

La requête d'autorisation est encapsulée dans un **JWT signé** par le client.

```
GET /authorize?client_id=CLIENT_ID&request=eyJhbGciOiJSUzI1NiJ9...
```

```json
// Payload du JWT (request object)
{
  "iss": "CLIENT_ID",
  "aud": "https://login.microsoftonline.com/{tenant}/v2.0",
  "response_type": "code",
  "client_id": "CLIENT_ID",
  "redirect_uri": "https://monapp.com/callback",
  "scope": "openid User.Read",
  "state": "xyz123",
  "code_challenge": "E9Melhoa2...",
  "code_challenge_method": "S256",
  "exp": 1718003600
}
```

**Ce que ça protège :** la signature empêche la modification des paramètres — un attaquant ne peut pas altérer le `redirect_uri` ou injecter des scopes sans invalider la signature.

**Limite :** le JWT est toujours dans l'URL — un observateur peut le décoder (base64) et voir les paramètres. JAR protège l'**intégrité**, pas la **confidentialité**.

---

## JWE — JSON Web Encryption

Chiffre le request object JAR pour garantir la **confidentialité** de la requête.

```
request=eyJhbGciOiJSU0EtT0FFUC0yNTYiLCJlbmMiOiJBMjU2R0NNIn0...
```

Le JWT est maintenant **chiffré avec la clé publique de l'Authorization Server** — seul Entra ID peut le déchiffrer.

| | Signé (JAR) | Chiffré (JWE) | Signé + Chiffré (JAR + JWE) |
|---|---|---|---|
| Intégrité | ✓ | ✗ | ✓ |
| Confidentialité | ✗ | ✓ | ✓ |
| Protection maximale | Non | Non | Oui |

---

## PAR — Pushed Authorization Requests (RFC 9126)

Au lieu d'envoyer les paramètres dans l'URL, le client les **pousse directement au serveur** via un endpoint dédié avant la redirection.

```
# Étape 1 — Pousser la requête (serveur à serveur, hors du navigateur)
POST /oauth2/v2.0/par
Content-Type: application/x-www-form-urlencoded

client_id=CLIENT_ID
&client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
&client_assertion=eyJ...  (JWT signé)
&response_type=code
&redirect_uri=https://monapp.com/callback
&scope=openid User.Read
&code_challenge=E9Melhoa2...
&code_challenge_method=S256

# Réponse — request_uri temporaire
{
  "request_uri": "urn:ietf:params:oauth:request_uri:abc123",
  "expires_in": 90
}

# Étape 2 — Rediriger avec juste le request_uri
GET /authorize?client_id=CLIENT_ID&request_uri=urn:ietf:params:oauth:request_uri:abc123
```

**Avantages PAR :**
- Paramètres jamais exposés dans le navigateur / logs proxy
- Le client s'authentifie dès l'étape 1 (pas seulement lors de l'échange de code)
- URL de redirection très courte et propre

---

## DPoP — Demonstration of Proof of Possession (RFC 9449)

Lie l'`access_token` à la **clé cryptographique du client** — même concept que [Token Protection](../entra-id/token-protection.md) mais au niveau du protocole standard OAuth2.

```
# Le client génère une paire de clés (éphémère ou persistante)
# Lors de chaque appel API, il joint un JWT DPoP signé avec sa clé privée

GET /v1.0/me HTTP/1.1
Authorization: DPoP eyJhbGciOiJSUzI1NiJ9...access_token...
DPoP: eyJhbGciOiJSUzI1NiIsImp3ayI6eyJrdHkiOi...proof_jwt...

# Le JWT DPoP contient :
{
  "jti": "uuid-unique",     // Anti-replay
  "htm": "GET",             // Méthode HTTP
  "htu": "https://graph.microsoft.com/v1.0/me",  // URL exacte
  "iat": 1718000000,        // Timestamp (court TTL)
  "ath": "hash-du-access-token"
}
```

**Ce que ça empêche :** un attaquant qui vole l'`access_token` ne peut pas l'utiliser — sans la clé privée du client, il ne peut pas générer un proof JWT valide.

---

## Récapitulatif — Quand utiliser quoi

| Extension | Protège contre | Complexité | Support Entra ID |
|---|---|---|---|
| **PKCE** | Interception du code d'autorisation | Faible | Natif |
| **JAR** | Manipulation des paramètres d'autorisation | Moyenne | Partiel |
| **PAR** | Exposition des paramètres dans l'URL | Moyenne | Preview |
| **JWE** | Lecture des paramètres de la requête | Élevée | Partiel |
| **DPoP** | Rejeu du token sur une autre machine | Moyenne | Preview |
| **Token Protection** | Rejeu (via TPM, implémentation Microsoft) | Faible | GA (P1) |

!!! info "OAuth 2.1"
    Le draft OAuth 2.1 rendra PKCE obligatoire pour tous les flows Auth Code et retirera officiellement l'Implicit Flow et ROPC. PAR et DPoP sont inclus comme recommandations fortes.
