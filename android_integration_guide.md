# Android Studio Integration Guide & API Documentation
**SGOD Integrated System - DepEd SDO Biliran**

This guide provides the complete blueprint and API documentation to build the native Android version using Android Studio with assistance from Gemini.

---

## 1. Architecture Overview

```
┌────────────────────────────────────────────────────────┐
│            Native Android App (Kotlin Compose)         │
│               Retrofit + OkHttp + Coil                 │
└──────────────────────────┬─────────────────────────────┘
                           │ HTTPS REST JSON + Bearer Auth
                           ▼
┌────────────────────────────────────────────────────────┐
│             Cloudflare Worker (Hono Backend)           │
│             CORS Enabled for Native Clients            │
└──────────────┬──────────────────────────┬──────────────┘
               │                          │
               ▼                          ▼
┌──────────────────────────────┐  ┌──────────────────────────────┐
│  Cloudflare D1 Database      │  │  Cloudflare R2 Bucket        │
│  (sgod_biliran_db)           │  │  (sgod-documents)           │
└──────────────────────────────┘  └──────────────────────────────┘
```

---

## 2. Prompt for Gemini in Android Studio

Copy and paste this prompt to Gemini when starting your project in Android Studio:

```text
I am creating a native Android app in Kotlin using Jetpack Compose for the "SGOD Integrated System (DepEd Biliran)".
The backend is an active Cloudflare Worker (Hono) that binds to Cloudflare D1 (SQLite) and Cloudflare R2 (document/image storage).

Requirements:
1. Architecture: MVVM with Repository pattern.
2. UI: Modern Jetpack Compose with Material 3, Noto Sans font, and DepEd/SGOD color palette (Primary Orange #ea580c, Deep Navy, Slate).
3. Networking: Retrofit 2 + OkHttp 3 + Kotlinx Serialization.
4. Authentication: Bearer token authentication (Header: 'Authorization: Bearer <token>' or 'X-User-Email'). Token is returned from POST /api/auth/login.
5. Storage: Encrypted DataStore to persist user session, school ID, and token.
6. Modules:
   - Login Screen (supports principal, encoder, coordinator, SGOD chief, SDS, admin accounts).
   - School Dashboard: Displays school info, quarterly compliance, recent announcements, module cards (DMS, HRD, M&E, SocMob, etc.).
   - Module Submission Screen: Multi-field form with 'Save as Draft' and 'Submit to SGOD' actions.
   - DMS Document Management: List clearance documents, view status badges (Pending, Approved, Returned), pick and upload PDF/images via multipart/form-data.
   - Profile & Encoder Settings: Manage school profile and assign designated encoder credentials.

Base URL: https://<your-cloudflare-worker-domain> (or http://10.0.2.2:8787 for Android Emulator local testing)
```

---

## 3. Recommended Android Studio Project Setup

### `build.gradle.kts` (Module: app)
```kotlin
dependencies {
    // Jetpack Compose & Material 3
    implementation(platform("androidx.compose:compose-bom:2024.10.01"))
    implementation("androidx.compose.ui:ui")
    implementation("androidx.compose.material3:material3")
    implementation("androidx.compose.material:material-icons-extended")
    implementation("androidx.navigation:navigation-compose:2.8.3")
    implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.8.7")

    // Networking & Serialization
    implementation("com.squareup.retrofit2:retrofit:2.11.0")
    implementation("com.jakewharton.retrofit:retrofit2-kotlinx-serialization-converter:1.0.0")
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")
    implementation("com.squareup.okhttp3:okhttp:4.12.0")
    implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

    // Image loading (R2 logo and photos)
    implementation("io.coil-kt:coil-compose:2.7.0")

    // Encrypted DataStore for Session Token
    implementation("androidx.datastore:datastore-preferences:1.1.1")
}
```

---

## 4. API Endpoints Reference

All endpoints accept and return JSON. Endpoints that require authentication accept:
- `Authorization: Bearer <token>`
- OR `X-User-Email: <email>`

### Authentication
| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/auth/login` | `{ "email": "...", "password": "..." }` → Returns `token`, `user`, `school` |
| `POST` | `/api/auth/logout` | Clears server-side session |
| `GET` | `/api/auth/me` | Returns current authenticated user and school |

### System & Periods
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/periods` | Lists all School Years & Quarters, plus `active_period` |
| `GET` | `/api/modules` | Lists all 8 SGOD modules with metadata & colors |
| `GET` | `/api/schools` | Lists all division schools |
| `GET` | `/api/schools/:id` | Get specific school details |
| `GET` | `/api/announcements`| Division announcements & directives |

### School Dashboard & Submissions
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/school/dashboard` | Aggregated dashboard data (school, submissions, documents, periods) |
| `GET` | `/api/submissions` | List submissions (supports `?quarter=Q1&school_year=2024-2025`) |
| `GET` | `/api/submissions/:module_key` | Submissions for specific module (hrd, me, socmob, etc.) |
| `POST` | `/api/submissions/:module_key` | Submit quarterly report (`quarter`, `school_year`, `form_data`, `action_type: 'draft' \| 'submit'`) |
| `POST` | `/api/submissions/:id/approve` | Principal signs and endorses encoder submission |
| `POST` | `/api/submissions/:id/review` | SGOD Coordinator validates (`status: 'APPROVED_BY_SGOD' \| 'RETURNED'`) |

### Document Management (DMS) & R2 Storage
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/dms/documents` | List clearance documents for school |
| `GET` | `/api/dms/documents/:id`| Get document status, reviewer remarks, tracking no |
| `POST` | `/api/dms/upload` | Multipart form (`file`, `title`, `category`) → Stores into Cloudflare R2 |
| `POST` | `/api/dms/documents/:id/review` | SGOD reviews clearance (`status: 'APPROVED' \| 'RETURNED'`) |
| `DELETE` | `/api/dms/documents/:id`| Delete pending document from DMS |
| `GET` | `/documents/:id/view` | Direct PDF stream (from R2 or generated clearance slip) |
| `GET` | `/assets/logo.png` | Division SGOD logo from R2 |

### Division Overview & Settings
| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/api/division/overview` | Division-wide metrics & submission progress |
| `POST` | `/api/settings/profile` | Update principal name/designation/password |
| `POST` | `/api/settings/encoder` | Assign/update designated school encoder credentials |
