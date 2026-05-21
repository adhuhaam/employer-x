# MiFPS APK base URL extraction (Worker, Employer, RA v1.0.0)

Artifacts analyzed:

- `MiFPS Worker_1.0.0.apk` — Android package `com.bestinet.mifps_worker`
- `MiFPS Employer_1.0.0.apk` — Android package `com.bestinet.mifps_employer`
- `MiFPS RA_1.0.0.apk` — Android package `com.bestinet.mifps_ra`

## Summary

**A dedicated REST or GraphQL “API base URL” (for example `https://api.example.com/`) does not appear as a contiguous string in DEX, `resources.arsc`, or Flutter `assets/flutter_assets` for any of the three builds.**

The applications are **Flutter**-based (Flutter WebView pigeon classes, `assets/flutter_assets`, Font Awesome package assets, and so on). In a normal release Android build, application strings such as environment base URLs are usually embedded in **Dart AOT** inside native libraries (commonly `lib/arm64-v8a/libapp.so` plus `libflutter.so`) or in rare cases in other blobs.

**These APK zips do not contain any `lib/<abi>/*.so` entries** (no `libapp.so`, no `libflutter.so`, no `icudtl.dat` in the archive listing checked via `unzip -l`). Without those binaries, the compiled Dart layer that would typically hold the backend base URL **is not present in these files**, so static extraction from the three APKs alone cannot recover that value.

## What was found (non-backend URLs and metadata)

Across all three APKs, HTTP(S) strings are overwhelmingly **SDK, XML namespace, and documentation** noise (Apache XML, Adobe XMP, SLF4J, Google issue tracker links, Tika mime catalog links, and so on).

**Third-party endpoints that do appear as real URLs:**

| URL / host | Role |
|------------|------|
| `https://licensemgr.identy.io/nverify/v1` | Identy license / verification service |
| `https://licensemgr.identy.io/nverify/v2` | Identy license / verification service |
| `https://firebaseinstallations.googleapis.com/v1/` | Firebase Installations (Google) |
| Various `https://www.googleapis.com/auth/...` | Google Sign-In / Play Games style OAuth scope strings |

**Employer-only:**

- `assets/flutter_assets/shorebird.yaml` — Shorebird updater config (`app_id: 7f964a04-899c-43d4-bdef-0d8e61771e28`). This identifies the app to Shorebird’s patch servers; it is **not** the MiFPS business API base URL.
- `assets/flutter_assets/assets/files/privacy_policy.html` — mentions `www.bestinet.com` in legal copy (corporate site, not shown as an API base).

**Identified application IDs (from DEX / assets):**

- Worker / Employer / RA: `com.bestinet.mifps_worker`, `com.bestinet.mifps_employer`, `com.bestinet.mifps_ra`
- Employer privacy HTML: “Bestinet Sdn Bhd”, reference to `www.bestinet.com`

## Method used

1. Unzip each APK and run `strings` on `classes*.dex`, `resources.arsc`, and Flutter asset files; filter `http://` / `https://` patterns.
2. Raw-byte scan of every ZIP member for substrings such as `bestinet`, `mifps`, `BASE_URL`, `baseUrl`, and `https://` — confirms there is **no** full backend URL string containing those markers inside the archive bytes.
3. Confirmed absence of `lib/` native libraries in the APK table of contents.

## Conclusion

**You cannot extract the MiFPS backend base URL from these three APK files alone** with static analysis, because the likely carrier of that string (Flutter AOT in `libapp.so` / engine libs) is **missing from the supplied APK archives**.

To obtain the base URL you would typically need: the **full installable build** (universal APK or split APKs / AAB with native `lib` splits), **remote config / bootstrap** traffic capture at runtime, or the **source / CI env** where the URL is defined.
