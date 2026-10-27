# Mobile checks (iOS, Android, React Native, Flutter, Expo)

Read this for any mobile app. Apply it in addition to the web-oriented references (secrets, access, data, injection).

Based on OWASP Mobile Top 10 and MASVS. See Sources at the end for links.

## Secrets in the binary

Anything shipped in the app can be extracted in minutes (`apktool`, `strings`, unzip an IPA). Never embed server API keys, signing secrets, or admin credentials. Obfuscation is not protection.

**React Native / Expo specifics:** Hermes bytecode does not hide strings. Values from `react-native-config`, `EXPO_PUBLIC_*`, and anything in `eas.json` or `app.config.js` that ends up in the JS bundle are baked in at build time and are not secret. The only safe pattern is a **backend proxy**: the app sends the user's session token to your server, and your server calls the third-party API with the real key.

```ts
// Bad: provider key shipped in the app
fetch('https://api.openai.com/v1/chat/completions', { headers: { Authorization: `Bearer ${OPENAI_API_KEY}` } });

// Good: your backend holds the key
fetch('https://your-api.com/ai/chat', {
  method: 'POST',
  headers: { Authorization: `Bearer ${userSessionToken}` },
  body: JSON.stringify({ message: userInput }),
});
```

## Insecure local storage
- Tokens, passwords, PII, or PINs in `SharedPreferences`, `UserDefaults`, `AsyncStorage`, plain SQLite/Realm, `localStorage` in a WebView, or plain files
- `AsyncStorage` is unencrypted plaintext on disk; on a rooted or jailbroken device tokens are trivially readable. Use `expo-secure-store` (Expo) or `react-native-keychain` (bare RN) for auth tokens.
- Use Keychain (iOS), Keystore with `EncryptedSharedPreferences` (Android), `expo-secure-store`, or `flutter_secure_storage`
- Sensitive data in app backups (`android:allowBackup="true"`), clipboard, screenshots/app-switcher previews, keyboard cache, or external storage

## Network
- Cleartext HTTP allowed (`usesCleartextTraffic`, `NSAllowsArbitraryLoads`), disabled certificate validation, trust-all `TrustManager`
- No certificate pinning for high-risk apps (finance, health); note it as a hardening item, not a blocker, for low-risk apps
- Tokens or PII in query strings

## Platform misuse
- Android: exported activities/services/receivers/providers with no permission, `debuggable="true"` in release, implicit intents carrying sensitive data, overly broad permissions, WebView with `setJavaScriptEnabled(true)` plus `addJavascriptInterface` or `allowFileAccess`
- iOS: overly broad entitlements, unprotected URL schemes, unvalidated Universal Link parameters, pasteboard leaks
- **Deep links:** parameters trusted without validation or auth (account takeover, open redirect); OAuth redirect handled by a hijackable custom scheme (prefer App Links / Universal Links). Any app or web page can fire `myapp://path?param=value`, so also check that: no tokens, passwords, or access-granting ids travel in the URL, and no destructive action (delete, pay, transfer) runs straight from link parameters without an in-app confirmation.

## Auth and authorization

All authorization must be enforced on the backend. Client checks (biometric, jailbreak/root detection, "premium" flags) are UX only. Check biometric prompts actually gate a server-validated secret.

A plain `isAuthenticated = true` result from a biometric API can be hooked (for example with Frida on a jailbroken or rooted device). Robust pattern: the server sends a random challenge, the app signs it with a hardware-backed key (Secure Enclave on iOS, StrongBox/Keystore on Android) that is unlocked by the biometric, and the server verifies the signature. Bypassing the prompt then yields nothing the attacker can forge.

## Release hygiene

Debug logging stripped (`Log.d`, `print`, `console.log`), test endpoints and feature flags removed, no source maps or `.env` bundled, ProGuard/R8 or Hermes bytecode as basic hardening, crash reporting configured to scrub PII, app store privacy labels/Data Safety form match what the app really collects.

## OTA updates

For CodePush and EAS Update, ensure the update channel is access-controlled and signed.

## Sources

- [OWASP MASVS](https://mas.owasp.org/MASVS/)
- [OWASP Mobile Top 10](https://owasp.org/www-project-mobile-top-10/)
- [expo-secure-store](https://docs.expo.dev/versions/latest/sdk/securestore/)
- [react-native-keychain](https://github.com/oblador/react-native-keychain)
