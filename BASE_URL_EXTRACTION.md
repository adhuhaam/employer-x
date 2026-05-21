# MiFPS APK Base URL Extraction

Analysis of `MiFPS Employer_1.0.0.apk`, `MiFPS RA_1.0.0.apk`, and `MiFPS Worker_1.0.0.apk` (strings, resources, AndroidManifest, jadx decompile).

## Summary

| APK | Package | API / OAuth host | OAuth redirect | Firebase project | Firebase storage |
|-----|---------|------------------|----------------|------------------|------------------|
| Employer | `com.bestinet.mifps_employer` | **Not embedded** in APK | N/A (no AppAuth intent filters) | `mifps-employer---production` | `mifps-employer---production.firebasestorage.app` |
| RA | `com.bestinet.mifps_ra` | `https://mifps.bestinet.com` | `https://mifps.bestinet.com/oauthredirect` | `mifps-ra---production` | `mifps-ra---production.firebasestorage.app` |
| Worker | `com.bestinet.mifps_worker` | `https://mifps.bestinet.com` | `https://mifps.bestinet.com/oauthredirect` | `mifps-worker---production` | `mifps-worker---production.firebasestorage.app` |

## Primary backend base URL (RA & Worker)

**Base URL:** `https://mifps.bestinet.com`

Evidence: `AndroidManifest.xml` intent filters for `flutter_appauth` / `net.openid.appauth`:

- `android:host="mifps.bestinet.com"`
- `android:path="/oauthredirect"`
- `android:scheme="https://mifps.bestinet.com/oauthredirect"`

The REST API base is typically the same origin as the OAuth issuer; the exact API path prefix (e.g. `/api`) is not stored as a plain string in these APKs and is likely configured in compiled Flutter code or fetched via OpenID discovery at runtime.

## Employer app note

The Employer APK does **not** register OAuth redirect handlers for `mifps.bestinet.com`. No plaintext `mifps.bestinet.com` string was found in the Employer binary. The backend base URL may be supplied at runtime (remote config), built into Dart AOT code not present as readable strings in this build, or use a different auth flow. Corporate site referenced in privacy policy: `https://www.bestinet.com`.

## Firebase (all three — production)

| App | Project ID | Storage bucket URL |
|-----|------------|-------------------|
| Employer | `mifps-employer---production` | `https://mifps-employer---production.firebasestorage.app` |
| RA | `mifps-ra---production` | `https://mifps-ra---production.firebasestorage.app` |
| Worker | `mifps-worker---production` | `https://mifps-worker---production.firebasestorage.app` |

## Other service URLs (all three)

- **Identy license verification:** `https://licensemgr.identy.io/nverify/v1` and `https://licensemgr.identy.io/nverify/v2`
- **Firebase Installations API:** `https://firebaseinstallations.googleapis.com/v1/`

## Package IDs

- Employer: `com.bestinet.mifps_employer`
- RA: `com.bestinet.mifps_ra`
- Worker: `com.bestinet.mifps_worker`
