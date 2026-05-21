# MiFPS App Structure & Workflow Analysis

Deep analysis of the **MiFPS Employer** and **MiFPS Worker** Android APKs via decompilation (apktool + jadx).

---

## 1. App Overview

**MiFPS** = **Maldives Integrated Foreign Placement System** — a suite of mobile apps by **Bestinet Sdn Bhd** for managing foreign worker placement in the Maldives. The system connects employers, workers, and registration agents.

| Property | Employer App | Worker App |
|----------|-------------|------------|
| Package ID | `com.bestinet.mifps_employer` | `com.bestinet.mifps_worker` |
| Version | 1.0.0 (build 2026010103) | 1.0.0 (build 2026010103) |
| Min SDK | API 24 (Android 7.0) | API 23 (Android 6.0) |
| Target SDK | API 36 | API 36 |
| Framework | Flutter (Dart AOT) | Flutter (Dart AOT) |
| Size | ~41 MB | ~39 MB |
| OTA Updates | Shorebird (app_id: `7f964a04-...`) | Not present |

---

## 2. Tech Stack

### Common to Both Apps

| Component | Technology |
|-----------|-----------|
| **UI Framework** | Flutter (Dart AOT-compiled, no kernel_blob.bin — release mode) |
| **Fonts** | Inter (all weights) + Material Icons + Font Awesome 7 + Iconsax + NunitoSans (Employer only) |
| **Biometrics SDK** | Identy SDK — fingerprint (4F, 2T, thumb) + face recognition + OCR/document scanning + NFC |
| **Firebase** | Cloud Messaging (FCM), Firebase Installations, Analytics |
| **Camera** | CameraX (camera_android_camerax) |
| **ML** | ML Kit Barcode Scanning (TFLite models bundled) |
| **Storage** | Flutter Secure Storage, Shared Preferences, DataStore, Room DB (Identy face users) |
| **Networking** | OkHttp3 |
| **File handling** | open_file, image_picker, printing (PDF), url_launcher |
| **Notifications** | flutter_local_notifications |
| **Permissions** | permission_handler |
| **Crypto** | BouncyCastle, Apache Tika (MIME detection) |

### Employer-Only

| Component | Technology |
|-----------|-----------|
| **Auth** | In-app WebView (flutter_inappwebview_android v6+) — no AppAuth/OpenID |
| **OTA** | Shorebird code push |
| **WebView** | Full InAppWebView with Chrome Custom Tabs, Trusted Web Activity, In-App Browser |
| **Stepper** | easy_stepper (multi-step workflows) |
| **Phone formatting** | flutter_multi_formatter (country flags bundled) |

### Worker-Only

| Component | Technology |
|-----------|-----------|
| **Auth** | OAuth 2.0 / OpenID Connect via `flutter_appauth` + `net.openid.appauth` |
| **OAuth Host** | `https://mifps.bestinet.com` |
| **OAuth Redirect** | `https://mifps.bestinet.com/oauthredirect` |
| **WebView** | flutter_inappwebview (older single-activity version) |

---

## 3. Identy Biometric SDK (Both Apps)

Both apps embed a comprehensive biometric verification suite from **Identy** (`licensemgr.identy.io`):

### Fingerprint Activities
| Activity | Purpose |
|----------|---------|
| `Enroll4FActivity` | Enroll 4 fingers |
| `Enroll2TActivity` | Enroll 2 thumbs |
| `EnrollFingersActivity` | General finger enrollment |
| `EnrollThumbActivity` | Single thumb enrollment |
| `Verify4FActivity` | Verify 4 fingers |
| `Verify2TActivity` | Verify 2 thumbs |
| `VerifyFingersActivity` | General finger verification |
| `VerifyThumbActivity` | Single thumb verification |
| `Capture4FActivity` | Capture 4 finger images |
| `CaptureFingersActivity` | General finger capture |
| `CaptureThumbActivity` | Thumb capture |
| `Capture2TActivity` | Capture 2 thumbs |

### Face Activities
| Activity | Purpose |
|----------|---------|
| `CaptureFaceActivity` | Capture face photo |
| `EnrollFaceActivity` | Enroll face (1:1) |
| `EnrollFace1toNActivity` | Enroll face (1:N database) |
| `VerifyFaceActivity` | Verify face (1:1) |
| `VerifyFace1toNActivity` | Verify face (1:N database) |
| `FaceIntroActivity` | Face capture instructions |
| `GCTestActivity` | Face quality/GC test |

### Document/OCR Activities
| Activity | Purpose |
|----------|---------|
| `CaptureOCRActivity` | Scan document via camera |
| `CaptureOCRGalleryActivity` | Scan document from gallery |
| `NFCReadActivity` | Read NFC chip (e-passport) |
| `OcrIntroActivity` | Document scan instructions |

### Identy License Files
- **Employer**: 6 license files (dev + production, expiring 31-Dec-2025)
- **Worker**: 3 license files (production only, expiring 31-Dec-2025)

### Identy ML Models (bundled as `.ort` / `.mobile` files)
- `identy_vdata*.ort` — fingerprint verification models (14 files)
- `identy_vfdata*.ort` — face verification models (10 files)
- `identy_vodata*.m` — OCR/document models (11 files)
- `qclaplace.all.ort` — quality check model
- `sdk_template.pub` — template public key

---

## 4. App Structure & Navigation

### Employer App — Bottom Tab Navigation

Inferred from tab icon assets and image naming:

| Tab | Icon Asset | Purpose |
|-----|-----------|---------|
| **Home** | `home-tab.png` | Dashboard / overview |
| **Features** | `features-tab.png` | Core business features |
| **Profile** | `profile-tab.png` | User/employer profile |
| **Settings** | `settings-tab.png` | App settings |

### Employer App — Core Feature Icons

| Feature | Icon Asset | Purpose |
|---------|-----------|---------|
| **Candidates** | `candidate-icon.png` | Worker candidate management |
| **Legalization** | `legalization-icon.png` | Document legalization workflow |
| **Payments** | `payment-icon.png` | Payment processing |
| **QR Code** | `qr_code.png`, `qr_code_frame.png` | QR code scanning/display |
| **Blacklist** | `blacklist-flag.png` | Blacklisted worker flagging |
| **Whitelist** | `whitelist-flag.png` | Whitelisted worker flagging |

### Worker App — Navigation & Features

Inferred from image assets:

| Feature | Icon Asset | Purpose |
|---------|-----------|---------|
| **Home** | `home-2.png` | Worker dashboard |
| **Jobs/Employment** | `bag.png` | Job listings/current employment |
| **Chats** | `Chats.png` | Messaging system |
| **ID Card** | `id-card.png` | Digital worker ID |
| **Payments** | `credit_card.png` | Payment/salary info |
| **Employer** | `buildings.png` | Employer information |
| **Awards/Achievements** | `award.png` | Worker awards |
| **Tasks** | `task.png` | Assigned tasks |
| **Search** | `Search.png` | Search functionality |
| **Organization** | `TreeStructure.png` | Org structure view |
| **Grid View** | `GridFour.png` | Grid layout toggle |
| **Profile** | `profile-circle.png` | Worker profile |
| **Settings** | `setting-2.png` | App settings |
| **Notifications** | `bell.png` | Push notifications |
| **Calendar** | `calendar.png` | Schedule/dates |
| **Edit** | `Edit.png` | Edit functionality |
| **Delete** | `Trash.png` | Delete items |
| **Add** | `Plus.png` | Create new items |

---

## 5. Inferred App Workflows

### Employer App Workflow

```
┌─────────────────────────────────────────────────┐
│                  APP LAUNCH                       │
│  Splash → Privacy Policy / Terms of Use → Login   │
│  (InAppWebView-based auth, no standard OAuth)     │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              HOME DASHBOARD (Tab 1)               │
│  Overview cards, notifications, quick actions      │
└───────────────────────┬─────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        ▼               ▼               ▼
┌──────────────┐ ┌──────────────┐ ┌──────────────┐
│  CANDIDATES  │ │ LEGALIZATION │ │   PAYMENTS   │
│              │ │              │ │              │
│ • View list  │ │ • Document   │ │ • View       │
│ • Add new    │ │   upload     │ │   payment    │
│ • Blacklist/ │ │ • Status     │ │   history    │
│   Whitelist  │ │   tracking   │ │ • Process    │
│ • QR scan    │ │ • Approval   │ │   payments   │
│ • View       │ │   workflow   │ │              │
│   profile    │ │ (stepper)    │ │              │
└──────┬───────┘ └──────────────┘ └──────────────┘
       │
       ▼
┌─────────────────────────────────────────────────┐
│           BIOMETRIC VERIFICATION                  │
│                                                   │
│  ┌──────────────┐  ┌──────────────┐              │
│  │ FINGERPRINT  │  │    FACE      │              │
│  │ • Enroll 4F  │  │ • Enroll     │              │
│  │ • Enroll 2T  │  │ • Verify     │              │
│  │ • Verify     │  │ • 1:N Match  │              │
│  │ • Capture    │  │ • Liveness   │              │
│  └──────────────┘  └──────────────┘              │
│  ┌──────────────┐  ┌──────────────┐              │
│  │  DOCUMENT    │  │     NFC      │              │
│  │ • OCR scan   │  │ • e-Passport │              │
│  │ • Gallery    │  │   chip read  │              │
│  │ • ID verify  │  │              │              │
│  └──────────────┘  └──────────────┘              │
└─────────────────────────────────────────────────┘
```

### Worker App Workflow

```
┌─────────────────────────────────────────────────┐
│                  APP LAUNCH                       │
│  Splash Screen → Login                            │
│  (OAuth 2.0 via mifps.bestinet.com)               │
│  Username/Password + possible CAPTCHA              │
└───────────────────────┬─────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────┐
│              HOME DASHBOARD                       │
│  Banner, quick action cards, notifications        │
└───────────────────────┬─────────────────────────┘
                        │
    ┌───────┬───────┬───┼───┬───────┬───────┐
    ▼       ▼       ▼       ▼       ▼       ▼
┌───────┐┌──────┐┌──────┐┌──────┐┌──────┐┌──────┐
│ JOBS  ││CHATS ││ ID   ││TASKS ││ PAY  ││PROFILE│
│       ││      ││ CARD ││      ││      ││      │
│• View ││• Msg ││• Dig ││• To  ││• Sal ││• Per │
│  curr ││  emp ││  ID  ││  do  ││  ary ││  son │
│  empl ││  loyer│  card││  list││  info││  al  │
│• Job  ││      ││• QR  ││• Com ││• Cre ││  info│
│  hist ││      ││      ││  plet││  dit ││• DOB │
│  ory  ││      ││      ││  ed  ││  card││• Gen │
└───┬───┘└──────┘└──────┘└──────┘└──────┘│  der │
    │                                     │• Mail│
    ▼                                     │• Phon│
┌─────────────────────────────────────┐   └──────┘
│      BIOMETRIC ENROLLMENT           │
│  (Same Identy SDK as Employer)      │
│  • Fingerprint enroll/verify        │
│  • Face enroll/verify               │
│  • Document OCR scan                │
│  • NFC e-passport read              │
└─────────────────────────────────────┘
```

---

## 6. Permissions Comparison

| Permission | Employer | Worker | Purpose |
|-----------|:--------:|:------:|---------|
| INTERNET | ✅ | ✅ | API communication |
| CAMERA | ✅ | ✅ | Biometrics, QR scanning |
| POST_NOTIFICATIONS | ✅ | ✅ | Push notifications |
| RECORD_AUDIO | ✅ | ✅ | Audio capture |
| NFC | ✅ | ✅ | e-Passport reading |
| READ_PHONE_STATE | ✅ | ✅ | Device identification |
| HIGH_SAMPLING_RATE_SENSORS | ✅ | ✅ | Biometric sensors |
| READ/WRITE_EXTERNAL_STORAGE | ✅ | ✅ | File access |
| WAKE_LOCK | ✅ | ✅ | Background processing |
| ACCESS_NETWORK_STATE | ✅ | ✅ | Network checks |
| ACCESS_WIFI_STATE | ✅ | ✅ | WiFi status |
| VIBRATE | ✅ | ✅ | Haptic feedback |
| READ_LOGS | ✅ | ✅ | Debugging |
| RECORD_VIDEO | ✅ | ✅ | Video capture |
| CHECK_LICENSE | ✅ | ✅ | Play Store license |

---

## 7. Firebase Configuration

| Property | Employer | Worker |
|----------|---------|--------|
| Project ID | `mifps-employer---production` | `mifps-worker---production` |
| Sender ID | `388620488767` | `590512783447` |
| Storage Bucket | `mifps-employer---production.firebasestorage.app` | `mifps-worker---production.firebasestorage.app` |

---

## 8. Key Differences Between Apps

| Aspect | Employer | Worker |
|--------|---------|--------|
| **Auth method** | WebView-based (no standard OAuth) | OAuth 2.0 / OpenID Connect |
| **Backend URL** | Not embedded (runtime/remote config) | `https://mifps.bestinet.com` |
| **OTA updates** | Shorebird enabled | Not present |
| **WebView** | InAppWebView v6+ (Chrome Custom Tabs) | InAppWebView (legacy single activity) |
| **Legal docs** | Privacy Policy + Terms of Use HTML bundled | Not bundled |
| **Stepper UI** | easy_stepper (multi-step forms) | Not present |
| **Country formatter** | flutter_multi_formatter (flags) | flutter_multi_formatter (flags) |
| **Chat feature** | Not evident from assets | Chat icon/feature present |
| **Min Android** | API 24 (7.0) | API 23 (6.0) |

---

## 9. External Services & Endpoints

| Service | URL | Purpose |
|---------|-----|---------|
| MiFPS Backend | `https://mifps.bestinet.com` | Main API + OAuth |
| Identy License | `https://licensemgr.identy.io/nverify/v1`, `/v2` | Biometric license verification |
| Firebase Installations | `https://firebaseinstallations.googleapis.com/v1/` | Firebase device registration |
| Firebase Storage (Emp) | `https://mifps-employer---production.firebasestorage.app` | File storage |
| Firebase Storage (Wkr) | `https://mifps-worker---production.firebasestorage.app` | File storage |
| Bestinet Corporate | `https://www.bestinet.com` | Company website |
| MiFPS Website | `https://www.mifps.com.mv` | Product website |

---

## 10. Security Notes

- Both apps use **BouncyCastle** cryptography library
- **Flutter Secure Storage** used for sensitive data (tokens, credentials)
- Identy SDK uses **public key verification** (`sdk_template.pub`)
- Employer uses **Shorebird OTA**, allowing code updates without app store release
- Worker uses **standard OAuth 2.0** flow with PKCE support (via `flutter_appauth`)
- Both apps target **SDK 36** (latest Android)
