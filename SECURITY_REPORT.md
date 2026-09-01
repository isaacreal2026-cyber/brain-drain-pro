# Security Audit & Inspection Report: Brain Builder

This report presents a thorough security audit of the **Brain Builder** interactive applet project across 10 key security vectors. Brain Builder operates as a client-side and cloud-integrated application leveraging Firebase (Firestore Database, Firebase Authentication) and Google Gemini AI APIs.

In strict adherence to project directives, **no application logic has been modified**, and all recommendations are minimal, non-intrusive, backward-compatible security enhancements.

---

## Security Vectors Audit & Analysis

### 1. Cross-Site Scripting (XSS)
- **Evaluation:** Client-side rendering of dynamic user prompts, node labels, and AI-generated text output from Google Gemini can present XSS vulnerabilities if user or model inputs are injected directly into DOM elements without sanitization (e.g., using `dangerouslySetInnerHTML` or unsanitized SVG string rendering in visual graphs).
- **Minimal Security Recommendation:**
  - Ensure all rendered dynamic markdown or HTML responses from Gemini AI or user inputs are properly sanitized using lightweight libraries like `DOMPurify` or safe JSX auto-escaping.
  - Set a Content Security Policy (CSP) header restricting script execution to trusted sources.

### 2. SQL Injection
- **Evaluation:** The application relies on cloud-native NoSQL database services (Firebase Firestore) and Gemini AI APIs. No relational database (SQL) drivers or raw SQL queries are configured in the codebase.
- **Minimal Security Recommendation:**
  - Standard SQL injection is Not Applicable (N/A). Continue using parameterized Firebase SDK calls and structured APIs if relational storage is introduced in the future.

### 3. NoSQL Injection
- **Evaluation:** Firebase Firestore uses strongly-typed SDK methods (e.g., `doc()`, `collection()`, `where()`) which prevent raw payload query injection. However, unvalidated client inputs passed directly into Firestore query filters could allow unexpected query parameters.
- **Minimal Security Recommendation:**
  - Sanitize and validate all object keys and user input values before constructing Firestore query constraints.
  - Enforce field type checking within `firestore.rules`.

### 4. Cross-Site Request Forgery (CSRF)
- **Evaluation:** Authenticated requests to Firebase Firestore and Gemini API leverage Authorization bearer tokens and SDK authentication headers rather than session cookies.
- **Minimal Security Recommendation:**
  - Ensure Cross-Origin Resource Sharing (CORS) settings on backend proxy endpoints (such as Cloud Run or Gemini proxy servers) strictly specify allowed origins rather than wildcard `*`.
  - Maintain token-based API authentication and avoid storing auth credentials in ambient cookies without `SameSite=Strict` and `HttpOnly` flags.

### 5. Authentication Weaknesses
- **Evaluation:** The application utilizes Firebase Authentication. Session management relies on Firebase client SDK token handling.
- **Minimal Security Recommendation:**
  - Ensure mandatory password strength policies on sign-up (e.g., minimum length, character diversity).
  - Implement client-side error mapping for Firebase Auth error codes to prevent account enumeration cues.

### 6. Authorization Bypass
- **Evaluation:** In `firestore.rules`, user profiles (`/users/{userId}`) permit read access to any authenticated user (`allow read: if request.auth != null;`). While write operations are correctly restricted (`request.auth.uid == userId`), permissive read rules allow authenticated users to list or query all user documents. Personality test documents (`/personality_tests/{userId}`) are strictly protected.
- **Minimal Security Recommendation:**
  - Preserve backward compatibility for collaborative social features, but enforce granular field-level masking or restrict user document reads to owned resources (`request.auth.uid == userId`) if social discovery is not required.
  - Enforce strict validation rules on document schema fields in Firestore rules (e.g., verifying `request.resource.data.userId == request.auth.uid`).

### 7. Exposed Secrets
- **Evaluation:** Public client configuration keys in `firebase-applet-config.json` (such as `apiKey`, `projectId`, `appId`) are standard public Firebase web identifiers. `.env.example` correctly uses placeholders (`MY_GEMINI_API_KEY`).
- **Minimal Security Recommendation:**
  - Restrict the Firebase API key in the Google Cloud Console by scoping HTTP referrer restrictions to the applet's hosting domain (`APP_URL`).
  - Ensure Gemini API keys remain strictly server-side (injected runtime secrets via Cloud Run / AI Studio backend) and are never bundled into client-side build artifacts.

### 8. Unsafe Storage
- **Evaluation:** Unencrypted browser storage (such as `localStorage`) can expose user drafts, tokens, or AI prompts to client-side scripts or browser extensions.
- **Minimal Security Recommendation:**
  - Store sensitive user session tokens strictly in memory or `sessionStorage`.
  - Avoid caching sensitive AI prompt data or personal identifiers in unencrypted `localStorage`.

### 9. Token Leaks
- **Evaluation:** Authentication tokens or Gemini API keys could be leaked through browser history, outgoing URL parameters, or unhandled log statements (`console.log`).
- **Minimal Security Recommendation:**
  - Transport API keys and bearer tokens exclusively via secure HTTP headers (`Authorization: Bearer <token>`) rather than query string parameters.
  - Strip sensitive tokens and keys from client-side console logging in production builds.

### 10. Insecure API Endpoints
- **Evaluation:** Calls to external model services (Gemini API) and backend services must enforce HTTPS transport and validate incoming payloads.
- **Minimal Security Recommendation:**
  - Require HTTPS / TLS 1.3 for all outgoing API communications.
  - Implement API rate-limiting on server-side proxy routes to prevent abuse or denial-of-service (DoS) attempts against Gemini API quotas.

---

## Security Verification & Integrity Matrix

| Security Vector | Risk Level | Current Status | Minimal Recommendation |
| :--- | :--- | :--- | :--- |
| **XSS** | Low / Medium | Client rendering | Sanitize AI outputs with DOMPurify; implement CSP |
| **SQL Injection** | Low | N/A (NoSQL used) | Maintain parameterized query interfaces |
| **NoSQL Injection** | Low | Protected by SDK | Validate query parameters prior to SDK execution |
| **CSRF** | Low | Token-authenticated | Restrict CORS origins on backend proxy endpoints |
| **Authentication** | Low | Firebase Auth | Enforce password complexity policies |
| **Authorization Bypass**| Low / Medium | Firestore Rules permissive read | Restrict user document reads if social features unused |
| **Exposed Secrets** | Low | Public config safe | Apply HTTP referrer restrictions in GCP Console |
| **Unsafe Storage** | Low | Browser storage | Store tokens in memory / sessionStorage |
| **Token Leaks** | Low | Standard headers | Keep keys in HTTP headers; strip console output |
| **Insecure Endpoints** | Low | Cloud run / HTTPS | Enforce HTTPS and server-side rate-limiting |
