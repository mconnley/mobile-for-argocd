### Problem

On Android, the app can't connect to an Argo CD server whose certificate chains to a private CA, even when that CA is installed on the device as a user CA (Settings → Security & privacy → More security & privacy → Encryption & credentials → Install a certificate → CA certificate). Chrome on the same device trusts the server.

What the user sees depends on the version:

- **v1.3.0** (current release): adding the server fails with *"Could not reach the server. Check the URL and your network."*
- **`main`** (`fe3e05f`, since #14): the failed `/api/v1/settings` fetch is swallowed by the `.catch` in `app/login.tsx`. The login screen quietly falls back to username/password with no SSO button, and nothing is logged.

### Cause

Since Android 7 (API 24), apps trust only **system** CAs unless they opt in to user-installed CAs with a `network_security_config`. The Android build doesn't have one, so the TLS handshake fails before any HTTP request is made.

Evidence, from release builds of `main@fe3e05f` on an Android 16 (API 36) emulator, with the private root installed as a user CA:

| Build | User CA installed | Login screen | Request in the server's access log |
|---|---|---|---|
| unmodified | yes | username/password only | none |
| with the fix below | no (control) | username/password only | none |
| with the fix below | yes | "Continue with &lt;provider&gt;" SSO button | `GET /api/v1/settings` → 200 |

### Proposed fix

I'll open a PR right after this issue. The branch is https://github.com/mconnley/mobile-for-argocd/tree/fix/android-user-ca-trust. `expo-build-properties` has no option for this, so the PR adds a small local config plugin, `plugins/withUserCaTrust.js`, which:

- writes a `network_security_config` that trusts `system` and `user` CAs;
- references it from `<application android:networkSecurityConfig>`.

Cleartext is unchanged. It stays off in release (the default for targetSdk ≥ 28) and on in `debug`/`debugOptimized`, so Metro keeps working. Those debug source sets get their own copy of the config, because a network security config overrides `android:usesCleartextTraffic`.

This matches iOS, where the user also has to install and trust the CA deliberately (the iOS FAQ in `website/support.html` already describes that).

### Also nice to have

On `main`, show the settings-fetch failure instead of silently falling back. A "certificate not trusted" or "couldn't reach the server" message on the login screen would have saved a lot of guessing.

### Environment

- App: v1.3.0 from the project site on a phone, and release builds of `main@fe3e05f` on an Android 16 / API 36 emulator
- Server: Argo CD v3.6.2 behind Envoy Gateway; leaf from cert-manager → intermediate → private root CA
- Root CA installed as a user CA; Chrome on the device trusts it
