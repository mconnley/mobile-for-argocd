Fixes ISSUE_URL

Since Android 7 (API 24), apps trust only system CAs unless they opt in with a network security config. The Android build has none, so a server whose certificate chains to a user-installed CA fails the TLS handshake. On v1.3.0 the app reports "Could not reach the server"; on `main` it silently drops the SSO option.

### Change

Adds a local Expo config plugin, `plugins/withUserCaTrust.js`, registered in `app.json` (`expo-build-properties` has no option for this). The plugin:

- writes `res/xml/network_security_config.xml`, trusting `system` and `user` CAs;
- sets `android:networkSecurityConfig` on `<application>`;
- keeps cleartext as it is today:
  - **`main`:** `false`, matching the targetSdk ≥ 28 default the release build already uses.
  - **`debug` and `debugOptimized`:** `true`, so Metro keeps working. The RN template sets `usesCleartextTraffic="true"` in those source sets, and a network security config would otherwise override it.

iOS is unchanged.

### Testing

- `yarn lint`, `yarn typecheck` and `yarn format:check` pass.
- `npx expo prebuild --platform android --clean`: the manifest carries the attribute, and the three source sets get the XML with the expected cleartext values.
- A/B on an Android 16 (API 36) emulator. Release APKs from `main@fe3e05f` with and without this change; server certificate chained to a private root installed as a user CA:
  - **Without the change:** no SSO button, and no request reaches the server (checked in the server's access log).
  - **With the change, before the CA was installed (control):** same result.
  - **With the change and the user CA:** `GET /api/v1/settings` returns 200, and the "Continue with &lt;provider&gt;" SSO button appears.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
