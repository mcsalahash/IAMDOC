# Application Proxy

Accès distant sécurisé aux applications web on-premises — sans VPN, sans port entrant.

---

## Architecture

```
Utilisateur distant
    │
    ▼ (HTTPS)
Microsoft Entra ID
(préauthentification + CA)
    │
    ▼
Service Application Proxy (cloud Microsoft)
    │
    ▼ (connexion sortante uniquement)
Connecteur Application Proxy (on-premises)
    │
    ▼
Application interne (intranet, LOB, SharePoint...)
```

Le connecteur n'ouvre **aucun port entrant** — il établit uniquement des connexions sortantes vers le cloud Microsoft.

---

## Modes de préauthentification

| Mode | Sécurité | Usage |
|---|---|---|
| **Microsoft Entra ID** | Haute — CA + MFA avant tout accès réseau | Applications web standard — recommandé |
| **Passthrough** | Faible — pas de préauth | Applications avec auth propre forte déjà en place |

---

## SSO vers l'application interne

| Méthode | Prérequis | Usage |
|---|---|---|
| **Kerberos (KCD)** | AD on-prem, SPN configuré | Applications IIS / Windows Auth |
| **Header-based** | Agent PingAccess | Applications lisant des headers HTTP |
| **Password-based** | Formulaire de login simple | Applications legacy sans SSO |
| **SAML** | App supportant SAML | Applications SAML modernes |

```powershell
# Configurer KCD sur le compte de service du connecteur
Set-ADUser -Identity "svc-appproxy" -ServicePrincipalNames @{Add="HTTP/monapp.dgfla.local"}

# Déléguer Kerberos au compte de service
Set-ADComputer -Identity "CONNECTORSERVER$" -PrincipalsAllowedToDelegateToAccount "svc-appproxy"
```

---

## Publication d'une application

```powershell
# Créer une application publiée via Application Proxy
New-MgApplication -DisplayName "Intranet RH" `
    -Web @{
        RedirectUris = @("https://intranet-rh.dgfla.com/callback")
    }

# Via le portail Entra :
# Applications → Enterprise applications → New application
# → Add an on-premises application
# → Internal URL : http://intranet-rh.dgfla.local
# → External URL : https://intranet-rh-TENANT.msappproxy.net
# → Pre-authentication : Microsoft Entra ID
```

---

## Avantages de sécurité

- **Conditional Access** appliqué avant tout accès réseau — même pour des apps legacy qui ne le supporteraient pas nativement
- **CAE** étendu aux apps on-premises via Application Proxy
- **Protection DDoS** au niveau du edge Microsoft
- **Zéro port entrant** — surface d'attaque réseau réduite

---

## Haute disponibilité

Déployer **minimum 2 connecteurs** dans le même groupe de connecteurs.

```powershell
# Vérifier l'état des connecteurs
Get-MgServicePrincipalSynchronizationJob -ServicePrincipalId $spId

# Portail Entra → Application proxy → Connector groups
# → Vérifier le statut Active/Inactive de chaque connecteur
```
