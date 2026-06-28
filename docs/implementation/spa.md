# SPA — Single Page Applications

Pour React, Angular, Vue, Svelte — tout JavaScript exécuté dans le navigateur.

---

## Configuration MSAL.js

```typescript
// src/authConfig.ts
import { Configuration, LogLevel } from "@azure/msal-browser";

export const msalConfig: Configuration = {
    auth: {
        clientId: "YOUR_CLIENT_ID",
        authority: "https://login.microsoftonline.com/YOUR_TENANT_ID",
        redirectUri: "http://localhost:3000",
        postLogoutRedirectUri: "/",
        navigateToLoginRequestUrl: false
    },
    cache: {
        cacheLocation: "sessionStorage", // Recommandé — isolation par onglet
        storeAuthStateInCookie: false
    },
    system: {
        loggerOptions: {
            logLevel: LogLevel.Warning
        }
    }
};

export const loginRequest = {
    scopes: ["openid", "profile", "User.Read"]
};
```

---

## Initialisation et hooks React

```typescript
// src/index.tsx
import { PublicClientApplication } from "@azure/msal-browser";
import { MsalProvider } from "@azure/msal-react";

const pca = new PublicClientApplication(msalConfig);

root.render(
    <MsalProvider instance={pca}>
        <App />
    </MsalProvider>
);
```

```typescript
// Composant protégé
import { useMsal, useIsAuthenticated } from "@azure/msal-react";

function ProfileButton() {
    const { instance, accounts } = useMsal();
    const isAuthenticated = useIsAuthenticated();

    const handleLogin = () => {
        instance.loginPopup(loginRequest);
    };

    return isAuthenticated
        ? <span>{accounts[0]?.username}</span>
        : <button onClick={handleLogin}>Se connecter</button>;
}
```

---

## Acquisition de token — pattern recommandé

```typescript
async function getToken(scopes: string[]): Promise<string> {
    const account = instance.getAllAccounts()[0];
    if (!account) throw new Error("Aucun compte connecté");

    try {
        // 1. Tentative silencieuse (cache → refresh token)
        const result = await instance.acquireTokenSilent({ scopes, account });
        return result.accessToken;

    } catch (error) {
        if (error instanceof InteractionRequiredAuthError) {
            // 2. Fallback interactif si le token silencieux échoue
            const result = await instance.acquireTokenPopup({ scopes, account });
            return result.accessToken;
        }
        throw error;
    }
}

// Appel API avec le token
async function callApi() {
    const token = await getToken(["https://graph.microsoft.com/User.Read"]);
    const response = await fetch("https://graph.microsoft.com/v1.0/me", {
        headers: { Authorization: `Bearer ${token}` }
    });
    return response.json();
}
```

---

## sessionStorage vs localStorage

| | sessionStorage | localStorage |
|---|---|---|
| Portée | Un seul onglet | Tous les onglets du domaine |
| SSO inter-onglets | Non | Oui |
| Sécurité XSS | Meilleure | Moins bonne |
| Recommandation | **Par défaut** | Uniquement si SSO inter-onglets requis |

!!! info "MSAL.js ne stocke jamais les refresh tokens dans localStorage"
    Même en mode `localStorage`, les refresh tokens restent dans un cookie sécurisé géré par Entra ID — pas dans le storage JavaScript.

---

## BFF Pattern (le plus sécurisé)

Pour les applications très sensibles, déplacer toute la gestion des tokens côté serveur.

```
SPA (navigateur)
    │  Cookie HttpOnly (pas de token exposé)
    ▼
BFF Backend (ASP.NET Core)
    │  access_token (jamais envoyé au navigateur)
    ▼
API Ressource
```

→ Voir [Web serveur — Microsoft.Identity.Web](web-server.md) pour l'implémentation BFF.
