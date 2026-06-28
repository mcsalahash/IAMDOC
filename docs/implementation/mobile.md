# Mobile — iOS / Android

---

## Brokered Authentication

Sur mobile, un **broker** (Microsoft Authenticator, Intune Company Portal) centralise l'authentification et fournit :

- SSO entre toutes les applications du device
- Intégration du PRT (device-level token)
- Conformité aux politiques Conditional Access
- Token binding au device

| | iOS | Android |
|---|---|---|
| Broker unique | Microsoft Authenticator | Authenticator ou Company Portal |
| Mécanisme IPC | URL schemes (msauthv2/v3) | Bound Service / AccountManager |
| Stockage tokens | Shared Keychain | AccountManager (TEE/StrongBox) |
| MAM-only broker | Authenticator | Company Portal |
| Fallback sans broker | ASWebAuthenticationSession | Chrome Custom Tabs |

---

## iOS — Configuration

### Info.plist

```xml
<key>CFBundleURLTypes</key>
<array>
    <dict>
        <key>CFBundleURLSchemes</key>
        <array>
            <string>msauth.$(PRODUCT_BUNDLE_IDENTIFIER)</string>
        </array>
    </dict>
</array>
<key>LSApplicationQueriesSchemes</key>
<array>
    <string>msauthv2</string>
    <string>msauthv3</string>
</array>
```

### AppDelegate.swift

```swift
import MSAL

@main
class AppDelegate: UIResponder, UIApplicationDelegate {
    func application(_ app: UIApplication,
                     open url: URL,
                     options: [UIApplication.OpenURLOptionsKey: Any] = [:]) -> Bool {
        return MSALPublicClientApplication.handleMSALResponse(url, sourceApplication: nil)
    }
}
```

### Authentification

```swift
let config = MSALPublicClientApplicationConfig(
    clientId: "CLIENT_ID",
    redirectUri: "msauth.com.monapp://auth",
    authority: try MSALAADAuthority(url: URL(string:
        "https://login.microsoftonline.com/TENANT_ID")!)
)
config.brokerEnabled = true  // Activer le broker

let application = try MSALPublicClientApplication(configuration: config)

let parameters = MSALInteractiveTokenParameters(
    scopes: ["User.Read"],
    webviewParameters: MSALWebviewParameters(authPresentationViewController: self)
)

application.acquireToken(with: parameters) { result, error in
    guard let result = result, error == nil else { return }
    let accessToken = result.accessToken
}
```

---

## Android — Configuration

### AndroidManifest.xml

```xml
<activity android:name="com.microsoft.identity.client.BrowserTabActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="msauth"
            android:host="com.monapp"
            android:path="/SIGNATURE_HASH" />
    </intent-filter>
</activity>
```

### auth_config.json

```json
{
  "client_id": "CLIENT_ID",
  "authorization_user_agent": "DEFAULT",
  "redirect_uri": "msauth://com.monapp/HASH",
  "broker_redirect_uri_registered": true,
  "authorities": [{
    "type": "AAD",
    "audience": { "type": "AzureADMyOrg", "tenant_id": "TENANT_ID" }
  }]
}
```

### Kotlin

```kotlin
val app = PublicClientApplication.createSingleAccountPublicClientApplication(
    context,
    R.raw.auth_config
)

val parameters = AcquireTokenParameters.Builder()
    .startAuthorizationFromActivity(this)
    .withScopes(listOf("User.Read"))
    .withCallback(object : AuthenticationCallback {
        override fun onSuccess(result: IAuthenticationResult) {
            val token = result.accessToken
        }
        override fun onError(exception: MsalException) { }
        override fun onCancel() { }
    })
    .build()

app.acquireToken(parameters)
```

---

## SSO Extension iOS (Enterprise SSO)

Déployée via MDM, intercepte les requêtes d'auth au niveau OS — étend le SSO Entra aux apps sans MSAL.

```xml
<!-- Profil MDM — Configuration SSO Extension -->
<key>ExtensionIdentifier</key>
<string>com.microsoft.azureauthenticator.ssoextension</string>
<key>TeamIdentifier</key>
<string>UBF8T346G9</string>
<key>URLs</key>
<array>
    <string>https://login.microsoftonline.com</string>
    <string>https://login.microsoft.com</string>
</array>
<key>Configurations</key>
<array>
    <dict>
        <key>browser_sso_interaction_enabled</key>
        <integer>1</integer>
        <key>disable_explicit_app_prompt</key>
        <integer>1</integer>
    </dict>
</array>
```

!!! warning "SSO Extension nécessite MDM"
    Sans profil MDM déployé, l'extension n'est pas active même si Authenticator est installé.
