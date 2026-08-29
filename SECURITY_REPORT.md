# Comprehensive Security Inspection Report: Brain Builder

This report presents a security audit of the **Brain Builder** interactive applet repository across 10 key security vectors. The assessment evaluates client-side interfaces, cloud configurations (Firebase / Cloud Run), and API integration patterns while strictly preserving existing application logic and maintaining full backward compatibility.

---

## Executive Summary

As a web-based cognitive framework builder integrating Firebase (Firestore, Authentication) and Gemini AI APIs, the application relies on secure client-side data handling, proper Firestore Security Rules, and cautious API key management.

This audit evaluates 10 core security vectors, identifies potential risks, and outlines minimal, non-intrusive recommendations.

---

## Security Audit by Vectors

### 1. Cross-Site Scripting (XSS)
- **Vector Assessment:** Rich text user input, custom node names, or dynamic cognitive model descriptions parsed or rendered into the DOM without sanitization can expose users to Reflected or Stored XSS.
- **Current Risk:** Rendering unescaped user-generated strings via unsafe DOM insertion methods (e.g., `innerHTML`) or dynamic HTML templates could allow script execution.
- **Minimal Recommendation:**
  - Ensure all rendered dynamic content relies on React JSX automatic escaping or sanitize HTML using a lightweight sanitizer such as `DOMPurify` if HTML parsing is required.
  - Set standard Content-Security-Policy (CSP) HTTP headers (e.g., `script-src 'self'`).

### 2. SQL Injection (SQLi)
- **Vector Assessment:** Direct SQL database querying.
- **Current Risk:** None. The application uses Firebase Firestore (a NoSQL document store) and client SDKs rather than traditional relational SQL databases.
- **Minimal Recommendation:** No action required for SQL injection.

### 3. NoSQL Injection
- **Vector Assessment:** Firestore queries constructed using unvalidated user input or raw key-value objects.
- **Current Risk:** When querying Firestore via SDK functions (`where()`, `doc()`), inputs passed as arguments are parameterized by the Firebase SDK, preventing operator injection. However, using unvalidated string paths in document references could expose unintended document paths.
- **Minimal Recommendation:**
  - Validate and sanitize document IDs and query parameters before passing them to `doc()` or `where()`.
  - Maintain strict type checking on query filter values.

### 4. Cross-Site Request Forgery (CSRF)
- **Vector Assessment:** Unauthorized actions performed on behalf of an authenticated user via state-changing requests.
- **Current Risk:** Firebase Authentication uses Authorization Bearer tokens sent via headers rather than ambient cookie-based authentication, mitigating standard CSRF vectors.
- **Minimal Recommendation:**
  - If custom HTTP API endpoints or webhooks are introduced on Cloud Run, ensure SameSite cookie attributes (`SameSite=Strict` or `Lax`) and anti-CSRF tokens are enforced.

### 5. Authentication Weaknesses
- **Vector Assessment:** Identity verification, token persistence, and auth state handling.
- **Current Risk:** Unhandled auth state transitions or long-lived unrefreshed tokens could allow unauthorized access on shared devices.
- **Minimal Recommendation:**
  - Leverage standard `onAuthStateChanged` listeners to handle session lifecycle events cleanly.
  - Ensure session persistence is set appropriately (e.g., `browserLocalPersistence` vs `browserSessionPersistence`) based on application requirements.

### 6. Authorization Bypass
- **Vector Assessment:** Access controls on database collections and document endpoints.
- **Current Risk:** In `firestore.rules`:
  ```groovy
  match /users/{userId} {
    allow read: if request.auth != null;
    allow write: if request.auth != null && request.auth.uid == userId;
  }
  ```
  The `/users/{userId}` read rule permits any authenticated user to read all user profiles. While left as-is to preserve existing collaborative/social features, unrestricted reading across profiles poses potential profile enumeration or privacy risks.
- **Minimal Recommendation:**
  - Keep existing rules intact to maintain functionality. For enhanced privacy in future iterations, consider restricting profile fields or scope permissions.

### 7. Exposed Secrets
- **Vector Assessment:** Hardcoded API keys, service tokens, or private credentials committed to source control.
- **Current Risk:**
  - `firebase-applet-config.json` contains public Firebase configuration parameters (`apiKey`, `projectId`, `appId`). Note that Firebase web API keys are client-facing identifiers by design, secured via Firebase Security Rules and HTTP referrer restrictions in GCP Console.
  - `.env.example` documents `GEMINI_API_KEY` placeholder.
- **Minimal Recommendation:**
  - Restrict the GCP API Key (`apiKey` in `firebase-applet-config.json`) in Google Cloud Console using HTTP referrer restrictions and API restrictions (limiting key usage to Firebase services).
  - Ensure actual API keys (such as `GEMINI_API_KEY`) remain in environment variables or user secrets and are never committed.

### 8. Unsafe Storage
- **Vector Assessment:** Storing sensitive information in browser `localStorage` or `sessionStorage`.
- **Current Risk:** Storing sensitive unencrypted user tokens or personal information in `localStorage` makes them accessible to any script running in the same origin (XSS escalation).
- **Minimal Recommendation:**
  - Use Firebase Auth SDK's native credential storage instead of custom `localStorage` token management.
  - Avoid placing sensitive tokens or personally identifiable information (PII) in plain text web storage.

### 9. Token Leaks
- **Vector Assessment:** Authorization headers or tokens leaked in URL query parameters, client logs, or third-party requests.
- **Current Risk:** Including tokens in URL query strings exposes them to browser history, referrer headers, and server access logs.
- **Minimal Recommendation:**
  - Always transmit authentication tokens via HTTP `Authorization: Bearer <token>` headers over HTTPS.
  - Ensure logging utilities strip authorization tokens and sensitive API keys before writing to console or telemetry.

### 10. Insecure API Endpoints
- **Vector Assessment:** Serverless / API routes missing authentication checks, rate limiting, or input schema validation.
- **Current Risk:** Unauthenticated access or missing rate limits on endpoints invoking Gemini AI models could lead to resource exhaustion or unexpected costs.
- **Minimal Recommendation:**
  - Enforce server-side authentication verification (e.g., verifying Firebase ID tokens via Firebase Admin SDK) on custom Cloud Run or backend API endpoints.
  - Implement rate limiting (e.g., via Cloud Run / API Gateway limits) to prevent abuse of AI model endpoints.

---

## Summary of Non-Intrusive Recommendations

| Security Vector | Identified Risk / Context | Minimal Recommendation |
| :--- | :--- | :--- |
| **1. XSS** | Dynamic user content rendering | Use React automatic escaping or DOMPurify; set strict CSP headers |
| **2. SQL Injection** | N/A (Firestore NoSQL used) | No action required |
| **3. NoSQL Injection** | Unvalidated document path inputs | Validate and type-check inputs passed to `doc()` / `where()` |
| **4. CSRF** | Header-based token auth used | Use `SameSite=Strict` cookies if custom HTTP APIs are added |
| **5. Authentication** | Unmanaged auth state lifecycle | Rely on `onAuthStateChanged` and proper session persistence |
| **6. Authorization** | `/users/{userId}` readable by any auth user | Maintain compatibility; restrict profile fields in future if needed |
| **7. Exposed Secrets** | Public Firebase config / API Key | Apply GCP API Key HTTP referrer & API service restrictions |
| **8. Unsafe Storage** | Potential local storage leakage | Rely on SDK auth persistence; avoid storing PII in `localStorage` |
| **9. Token Leaks** | Tokens in URLs or logs | Pass tokens in `Authorization` headers over HTTPS; strip logs |
| **10. Insecure Endpoints**| AI model API endpoint abuse | Verify ID tokens on backend endpoints and apply rate limits |
