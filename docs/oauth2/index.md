# OAuth 2.0 & OpenID Connect

Protocoles d'authentification et d'autorisation — des fondamentaux aux extensions de sécurité avancées.

## Dans ce module

- [Protocoles fondamentaux](protocoles.md) — OAuth2 vs OIDC, rôles, scopes
- [Flows utilisateur](flows-utilisateur.md) — Implicit (déprécié), Auth Code, PKCE
- [Flows machine-to-machine](flows-m2m.md) — Client Credentials, Device Code, ROPC, Assertion flows
- [Sécurité avancée](securite-avancee.md) — JAR, PAR, JWE, mTLS, DPoP
- [Tokens](tokens.md) — JWT anatomy, opaque tokens, introspection, validation

!!! warning "OAuth2 vs OIDC en une ligne"
    **OAuth2** résout l'**autorisation** (accès à une ressource). **OIDC** ajoute l'**authentification** (identité de l'utilisateur) en standardisant un `id_token` JWT au-dessus d'OAuth2.
