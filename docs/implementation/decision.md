# Arbre de décision — Quel flow ?

## Choisir en 3 questions

```
1. Y a-t-il un utilisateur humain qui se connecte ?
   │
   ├── NON → Client Credentials ou Managed Identity
   │         └── Ressource Azure ?  → Managed Identity (zéro credential)
   │             Sinon             → Client Credentials + certificat
   │
   └── OUI → Quel type d'application ?
             │
             ├── SPA (JavaScript dans le navigateur)
             │   └── Auth Code + PKCE avec MSAL.js
             │
             ├── Web serveur (ASP.NET, Node.js, Python)
             │   └── Auth Code confidentiel avec Microsoft.Identity.Web
             │
             ├── Desktop (.NET, WPF, WinForms)
             │   └── Auth Code + PKCE + WAM Broker (MSAL.NET)
             │
             ├── Mobile (iOS, Android)
             │   └── Auth Code + PKCE + Broker (Authenticator)
             │
             ├── Electron
             │   └── Auth Code + PKCE via navigateur système (MSAL Node)
             │
             └── Appareil sans navigateur (CLI, TV, IoT)
                 └── Device Code Flow
```

---

## Tableau de référence rapide

| Type d'app | Flow | SDK | Stockage tokens | PKCE | Broker |
|---|---|---|---|---|---|
| SPA | Auth Code + PKCE | MSAL.js | sessionStorage | Obligatoire | Non |
| SPA + BFF | Auth Code (confidentiel) | MI.Web | Serveur (cache) | Non | Non |
| Web serveur | Auth Code (confidentiel) | MI.Web | Cookie + cache | Non | Non |
| Desktop .NET | Auth Code + PKCE | MSAL.NET + WAM | DPAPI (disque) | Oui | WAM |
| Electron | Auth Code + PKCE | MSAL Node | safeStorage | Oui | Non |
| Mobile iOS | Auth Code + PKCE | MSAL.iOS | Keychain | Oui | Authenticator |
| Mobile Android | Auth Code + PKCE | MSAL.Android | Keystore | Oui | Authenticator |
| Daemon/Service | Client Credentials | MSAL + Cert/MI | Mémoire | N/A | Non |
| Azure Resource | Managed Identity | Azure.Identity | Mémoire | N/A | Non |
| CLI / IoT | Device Code | MSAL | Mémoire/fichier | N/A | Non |

---

## Règles à retenir

!!! danger "Ne jamais faire"
    - Stocker un `access_token` dans `localStorage` (vulnérable XSS)
    - Mettre un `client_secret` dans le code d'une SPA ou d'une app mobile
    - Utiliser l'Implicit Flow (déprécié depuis OAuth 2.1)
    - Utiliser ROPC (Resource Owner Password Credentials) — contourne MFA et CA

!!! success "Toujours faire"
    - Utiliser PKCE pour tout client public
    - Tenter `acquireTokenSilent()` avant tout appel interactif
    - Préférer Managed Identity au Client Credentials quand la ressource est sur Azure
    - Utiliser le WAM Broker sur Windows pour le SSO device-wide
