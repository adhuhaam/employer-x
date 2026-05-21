# MiFPS API Calls & Auth Flow Analysis

Deep research into the API endpoints, authentication flow, and data exchange patterns for the **MiFPS Employer** and **MiFPS Worker** apps.

---

## 1. Authentication Architecture

### Worker App — OAuth 2.0 / OpenID Connect

The Worker app uses a **standard OAuth 2.0 Authorization Code flow with PKCE** via the `flutter_appauth` library (backed by `net.openid.appauth`).

```
┌──────────────┐     ┌─────────────────────────┐     ┌──────────────────┐
│  Worker App  │     │  mifps.bestinet.com     │     │  MiFPS API       │
│  (Flutter)   │     │  (OAuth/OpenID Server)  │     │  Backend         │
└──────┬───────┘     └────────────┬────────────┘     └────────┬─────────┘
       │                          │                            │
       │  1. Discover endpoints   │                            │
       │─────────────────────────>│                            │
       │  GET /.well-known/       │                            │
       │  openid-configuration    │                            │
       │<─────────────────────────│                            │
       │  {authorization_endpoint,│                            │
       │   token_endpoint,        │                            │
       │   issuer, jwks_uri, ...} │                            │
       │                          │                            │
       │  2. Authorization Request│                            │
       │─────────────────────────>│                            │
       │  GET /authorize?         │                            │
       │    client_id=...         │                            │
       │    redirect_uri=https:// │                            │
       │      mifps.bestinet.com/ │                            │
       │      oauthredirect       │                            │
       │    response_type=code    │                            │
       │    scope=openid ...      │                            │
       │    state=...             │                            │
       │    nonce=...             │                            │
       │    code_challenge=...    │                            │
       │    code_challenge_method │                            │
       │      =S256              │                            │
       │                          │                            │
       │  3. User logs in         │                            │
       │  (browser/webview)       │                            │
       │<─────────────────────────│                            │
       │                          │                            │
       │  4. Auth callback        │                            │
       │<─────────────────────────│                            │
       │  redirect_uri?           │                            │
       │    code=AUTH_CODE         │                            │
       │    state=...             │                            │
       │    (or implicit:         │                            │
       │     access_token,        │                            │
       │     id_token,            │                            │
       │     token_type,          │                            │
       │     expires_in,          │                            │
       │     scope)               │                            │
       │                          │                            │
       │  5. Token Exchange       │                            │
       │─────────────────────────>│                            │
       │  POST /token             │                            │
       │    grant_type=           │                            │
       │      authorization_code  │                            │
       │    code=AUTH_CODE        │                            │
       │    redirect_uri=...      │                            │
       │    client_id=...         │                            │
       │    code_verifier=...     │                            │
       │<─────────────────────────│                            │
       │  {access_token,          │                            │
       │   token_type: "Bearer",  │                            │
       │   expires_in,            │                            │
       │   refresh_token,         │                            │
       │   id_token,              │                            │
       │   scope}                 │                            │
       │                          │                            │
       │  6. API calls with token │                            │
       │─────────────────────────────────────────────────────>│
       │  Authorization: Bearer <access_token>                │
       │<─────────────────────────────────────────────────────│
       │                          │                            │
       │  7. Token Refresh        │                            │
       │─────────────────────────>│                            │
       │  POST /token             │                            │
       │    grant_type=           │                            │
       │      refresh_token       │                            │
       │    refresh_token=...     │                            │
       │<─────────────────────────│                            │
       │  {new access_token, ...} │                            │
       │                          │                            │
       │  8. End Session          │                            │
       │─────────────────────────>│                            │
       │  id_token_hint=...       │                            │
       │  post_logout_redirect_uri│                            │
       │  state=...               │                            │
```

### Employer App — WebView-Based Auth

The Employer app does **NOT** use standard OAuth/AppAuth. Instead, it uses `flutter_inappwebview` (InAppBrowser, Chrome Custom Tabs, Trusted Web Activity) for authentication. This is a more opaque flow where:

1. The app opens a WebView to a login URL (likely on the Bestinet/MiFPS backend)
2. The user logs in through the web interface
3. The WebView intercepts a callback URL or JavaScript bridge to capture auth tokens
4. Tokens are stored in Flutter Secure Storage

---

## 2. OAuth Configuration Details (Worker App)

### Endpoints (from AndroidManifest + AppAuth source)

| Endpoint | Value |
|----------|-------|
| **OAuth Host** | `https://mifps.bestinet.com` |
| **Redirect URI** | `https://mifps.bestinet.com/oauthredirect` |
| **Discovery URL** | `https://mifps.bestinet.com/.well-known/openid-configuration` |
| **Authorization Endpoint** | Discovered via `.well-known` (likely `/authorize` or `/oauth/authorize`) |
| **Token Endpoint** | Discovered via `.well-known` (likely `/token` or `/oauth/token`) |
| **End Session Endpoint** | Discovered via `.well-known` (optional) |

### OAuth Parameters (from decompiled `AppAuth` source)

| Parameter | Value/Source |
|-----------|-------------|
| `client_id` | Configured in Dart code (not extractable from AOT binary) |
| `response_type` | `code` (Authorization Code flow) |
| `redirect_uri` | `https://mifps.bestinet.com/oauthredirect` |
| `scope` | Includes `openid` (likely also `profile`, `email`) |
| `code_challenge_method` | `S256` (PKCE) |
| `state` | Random per-request |
| `nonce` | Random per-request |

### ID Token (JWT) Claims

From the decompiled `IdToken` class (file `s.java`), the JWT `id_token` contains:

| Claim | Type | Description |
|-------|------|-------------|
| `iss` | String | Issuer (must match discovery issuer) |
| `sub` | String | **Subject — likely the NID (National ID) or user ID** |
| `aud` | String/Array | Audience (must match client_id) |
| `exp` | Long | Expiration time (Unix timestamp) |
| `iat` | Long | Issued-at time (Unix timestamp) |
| `nonce` | String | Must match request nonce |
| `azp` | String | Authorized party |

> **Note**: The `sub` claim in the ID token is the **user identifier** — for MiFPS this is very likely the **NID (National ID Number)**. The email would typically come from a `userinfo` endpoint or additional claims in the id_token beyond the standard set.

### Token Response Fields

From the decompiled `TokenResponse` class (file `w.java`):

```json
{
  "token_type": "Bearer",
  "access_token": "<JWT or opaque token>",
  "expires_in": 3600,
  "refresh_token": "<refresh token for renewal>",
  "id_token": "<JWT with user claims (iss, sub, aud, exp, iat, nonce, azp)>",
  "scope": "openid profile email ..."
}
```

### Authorization Response Fields

From the callback URL query parameters (file `AuthorizationManagementActivity.java`):

```
https://mifps.bestinet.com/oauthredirect?
  code=<authorization_code>
  state=<must match request state>
  token_type=<e.g. Bearer>
  access_token=<if implicit/hybrid flow>
  expires_in=<seconds>
  id_token=<JWT>
  scope=<granted scopes>
```

---

## 3. API Communication Layer

### HTTP Client Stack

Both apps use **OkHttp3** (via Dart's HTTP layer or directly) for API communication.

| Component | Technology |
|-----------|-----------|
| HTTP Client | OkHttp3 (Kotlin/Java, called from Dart via platform channels) |
| Interceptors | `ConnectInterceptor` (connection management) |
| Cache | `DiskLruCache` (response caching) |
| TLS | BouncyCastle TLS provider |
| HTTP/2 | Supported (Http2Connection, Http2Stream) |

### API Call Pattern (Inferred)

Based on the architecture, all API calls follow this pattern:

```http
GET/POST https://mifps.bestinet.com/api/<endpoint>
Authorization: Bearer <access_token>
Content-Type: application/json
Accept: application/json
```

### Likely API Endpoints (Inferred from App Features)

Since the actual Dart code is AOT-compiled and the API paths are not extractable as plain strings, these are **inferred** from the app's feature set:

#### Worker App Endpoints

| Method | Endpoint (Inferred) | Purpose | Evidence |
|--------|---------------------|---------|----------|
| GET | `/api/worker/profile` | Get worker profile | `profile-circle.png`, `avatar.png` |
| PUT | `/api/worker/profile` | Update worker profile | `Edit.png` |
| GET | `/api/worker/jobs` | Get job listings | `bag.png` |
| GET | `/api/worker/employer` | Get employer info | `buildings.png` |
| GET | `/api/worker/payments` | Get salary/payment info | `credit_card.png` |
| GET | `/api/worker/tasks` | Get assigned tasks | `task.png` |
| GET | `/api/worker/notifications` | Get notifications | `bell.png` |
| GET/POST | `/api/worker/chats` | Chat messages | `Chats.png` |
| GET | `/api/worker/id-card` | Digital ID card data | `id-card.png` |
| POST | `/api/worker/biometric/enroll` | Submit biometric data | Identy SDK |
| POST | `/api/worker/biometric/verify` | Verify biometric data | Identy SDK |
| POST | `/api/worker/document/scan` | Submit document scan | OCR activities |

#### Employer App Endpoints

| Method | Endpoint (Inferred) | Purpose | Evidence |
|--------|---------------------|---------|----------|
| GET | `/api/employer/dashboard` | Dashboard data | `home-tab.png` |
| GET | `/api/employer/candidates` | Candidate list | `candidate-icon.png` |
| POST | `/api/employer/candidates` | Add candidate | - |
| GET | `/api/employer/candidates/{id}` | Candidate detail | - |
| PUT | `/api/employer/candidates/{id}/blacklist` | Blacklist worker | `blacklist-flag.png` |
| PUT | `/api/employer/candidates/{id}/whitelist` | Whitelist worker | `whitelist-flag.png` |
| GET | `/api/employer/legalization` | Legalization status | `legalization-icon.png`, `easy_stepper` |
| POST | `/api/employer/legalization` | Submit documents | - |
| GET | `/api/employer/payments` | Payment history | `payment-icon.png` |
| POST | `/api/employer/payments` | Process payment | - |
| GET | `/api/employer/profile` | Employer profile | `profile-tab.png` |
| POST | `/api/employer/qr/scan` | QR code scan result | `qr_code.png` |
| POST | `/api/employer/biometric/enroll` | Biometric enrollment | Identy SDK |
| POST | `/api/employer/biometric/verify` | Biometric verification | Identy SDK |

---

## 4. Biometric API Calls (Identy SDK)

Both apps make external API calls to the Identy license server:

| Method | URL | Purpose |
|--------|-----|---------|
| POST | `https://licensemgr.identy.io/nverify/v1` | Verify biometric license (v1) |
| POST | `https://licensemgr.identy.io/nverify/v2` | Verify biometric license (v2) |

The biometric data (fingerprint templates, face embeddings, OCR results) is:
1. Captured locally on-device using the Identy SDK
2. Processed using bundled ONNX Runtime ML models (`.ort` files)
3. Sent to the MiFPS backend for storage/verification

---

## 5. Firebase API Calls

| Service | URL | Purpose |
|---------|-----|---------|
| Firebase Installations | `https://firebaseinstallations.googleapis.com/v1/` | Device registration |
| FCM (push) | `projects/{project_id}/installations/{id}/authTokens:generate` | Generate FCM auth tokens |
| Firebase Storage (Employer) | `https://mifps-employer---production.firebasestorage.app` | File upload/download |
| Firebase Storage (Worker) | `https://mifps-worker---production.firebasestorage.app` | File upload/download |

---

## 6. Backend Server Status

| Domain | DNS | Status |
|--------|-----|--------|
| `mifps.bestinet.com` | **NXDOMAIN** (not resolving) | Server appears **offline or decommissioned** |
| `bestinet.com` | Resolves (Cloudflare) | Returns 404 |
| `www.bestinet.com` | Does not resolve | Offline |
| `mifps.com.mv` | Resolves (Cloudflare) | Returns Cloudflare challenge page (403) |

> **Critical finding**: The primary backend `mifps.bestinet.com` does not resolve in DNS. The apps cannot authenticate or make API calls to a live server. The Maldives domain `mifps.com.mv` is behind Cloudflare protection.

---

## 7. Strategy for Removing Auth / Building a Mock Server

### Option A: Mock OAuth Server + Mock API Backend

Since the backend is offline, we can build a **complete mock server** that:

1. **Serves OpenID Discovery** at `/.well-known/openid-configuration`
2. **Handles Authorization** — immediately redirects with a code
3. **Issues Tokens** — returns mock JWT tokens with user data
4. **Serves API endpoints** — returns mock data for all API calls

#### Mock OpenID Discovery Response

```json
{
  "issuer": "https://mifps.bestinet.com",
  "authorization_endpoint": "https://mifps.bestinet.com/authorize",
  "token_endpoint": "https://mifps.bestinet.com/token",
  "userinfo_endpoint": "https://mifps.bestinet.com/userinfo",
  "end_session_endpoint": "https://mifps.bestinet.com/logout",
  "jwks_uri": "https://mifps.bestinet.com/.well-known/jwks.json",
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"],
  "scopes_supported": ["openid", "profile", "email"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"],
  "claims_supported": ["sub", "iss", "aud", "exp", "iat", "nonce", "email", "name", "nid"]
}
```

#### Mock Token Response (Bypassing Auth)

```json
{
  "access_token": "mock_access_token_for_testing",
  "token_type": "Bearer",
  "expires_in": 86400,
  "refresh_token": "mock_refresh_token",
  "id_token": "<JWT with claims below>",
  "scope": "openid profile email"
}
```

#### Mock ID Token JWT Claims

```json
{
  "iss": "https://mifps.bestinet.com",
  "sub": "A123456",
  "aud": "<client_id>",
  "exp": 1735689600,
  "iat": 1735603200,
  "nonce": "<from request>",
  "email": "worker@example.com",
  "name": "Test Worker",
  "nid": "A123456"
}
```

### Option B: DNS Override + Local Mock Server

To intercept the app's traffic:

1. **Set up DNS** to point `mifps.bestinet.com` to a local server
2. **Generate a self-signed TLS cert** for `mifps.bestinet.com`
3. **Run a mock server** (Node.js/Python) that handles all endpoints
4. **Install the APK** on an Android device/emulator with the CA cert trusted

#### `/etc/hosts` entry:
```
127.0.0.1 mifps.bestinet.com
```

### Option C: Repackage APK with Modified Auth

For testing without the real server:

1. **Decompile** the APK with apktool
2. **Modify the smali** to skip auth checks or hardcode tokens
3. **Repackage and re-sign** the APK
4. **Install** the modified APK

This is complex because the Dart code is AOT-compiled, but the AppAuth Java layer can be modified.

### Recommendation

**Option A (Mock Server) + Option B (DNS Override)** is the most practical approach:

- No need to modify the APK
- Can test the full auth flow including token exchange
- Can mock any API endpoint to return test data
- The NID number and email are in the ID token JWT `sub` and `email` claims
- The callback URL `https://mifps.bestinet.com/oauthredirect` is already known

---

## 8. Data Flow Summary

```
User opens app
    │
    ▼
OAuth Discovery (/.well-known/openid-configuration)
    │
    ▼
Login page in browser/webview
    │ User enters credentials
    ▼
Authorization server issues code → callback URL
    │ https://mifps.bestinet.com/oauthredirect?code=...&state=...
    ▼
App exchanges code for tokens (POST /token)
    │
    ▼
Receives: access_token, refresh_token, id_token (JWT)
    │
    │ id_token contains: { sub: "NID_NUMBER", email: "user@email.com", ... }
    │
    ▼
All API calls use: Authorization: Bearer <access_token>
    │
    ├─→ GET /api/worker/profile  →  { name, nid, email, dob, gender, phone, ... }
    ├─→ GET /api/worker/jobs     →  { employer, position, start_date, ... }
    ├─→ GET /api/worker/payments →  { salary, history, credit_card, ... }
    ├─→ POST /api/biometric/*    →  { fingerprint_data, face_data, ... }
    └─→ etc.
```
