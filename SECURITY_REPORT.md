# Security Inspection Report

This security audit evaluates the repository across 10 core security vectors:
1. Cross-Site Scripting (XSS)
2. SQL Injection
3. NoSQL Injection
4. Cross-Site Request Forgery (CSRF)
5. Authentication Weaknesses
6. Authorization Bypass
7. Exposed Secrets
8. Unsafe Storage
9. Token Leaks
10. Insecure API Endpoints

This report assesses risks based on current configuration and architectural specifications. All proposed recommendations strictly preserve existing user behavior and do not modify application logic.

---

## Security Vector Analysis

### 1. Cross-Site Scripting (XSS)
- **Risk Level:** Medium
- **Vector Assessment:**
  - Client-side React components render dynamic inputs from users and Gemini AI model responses. If untrusted raw HTML or unsanitized strings are injected directly into DOM elements (e.g., using `dangerouslySetInnerHTML`), stored or reflected XSS attacks could execute arbitrary client script within the context of the user session.
- **System Impact:** Potential session hijack, unauthorized state modification, or client-side data leakage.
- **Minimal Non-Intrusive Security Fixes:**
  - Rely on React's default JSX string escaping for standard UI text elements.
  - If dynamic HTML or SVG attributes must be parsed from user/AI inputs, sanitize payloads using a lightweight DOM sanitization library (e.g., `DOMPurify`) prior to insertion.
  - Implement a basic Content Security Policy (`CSP`) header in response headers restricting script sources to trusted origins.

### 2. SQL Injection
- **Risk Level:** Low / Not Applicable
- **Vector Assessment:**
  - The application relies on Firebase Firestore (a NoSQL document database) and REST/gRPC API abstractions; no relational SQL engines or raw SQL query strings are used in the codebase.
- **System Impact:** None under current architecture.
- **Minimal Non-Intrusive Security Fixes:**
  - Maintain parameterized query patterns and ORM abstractions if SQL database engines are introduced in future features.

### 3. NoSQL Injection
- **Risk Level:** Low
- **Vector Assessment:**
  - Firestore queries utilize the official Firebase SDK method calls (`doc()`, `collection()`, `where()`), which sanitize and parameterize argument values natively. Unstructured operator injection (such as MongoDB-style `$gt` parameter pollution) is prevented by SDK abstractions.
- **System Impact:** Minimal risk of query operator manipulation or unintended data retrieval.
- **Minimal Non-Intrusive Security Fixes:**
  - Validate and sanitize input string types (e.g., ensuring `userId` parameters match expected alphanumeric string patterns) before passing them to Firestore path builders or filter clauses.

### 4. Cross-Site Request Forgery (CSRF)
- **Risk Level:** Medium
- **Vector Assessment:**
  - Primary API authentication uses Firebase Auth Bearer tokens (`Authorization: Bearer <token>`) in request headers rather than ambient browser-managed session cookies, mitigating standard CSRF.
  - However, Cloud Run handlers or custom HTTP endpoints accepting cross-origin requests could be susceptible if origin headers are unvalidated.
- **System Impact:** Unauthorized state modification if cross-origin endpoints accept unvalidated browser requests.
- **Minimal Non-Intrusive Security Fixes:**
  - Enforce strict CORS policies on custom Cloud Run and serverless endpoints, restricting allowed origins to `APP_URL`.
  - Validate `Origin` and `Referer` headers on all state-changing write endpoints.

### 5. Authentication Weaknesses
- **Risk Level:** Medium
- **Vector Assessment:**
  - User identity lifecycle is delegated to Firebase Auth, which handles token generation, signing, and short-lived ID tokens with hourly refreshes.
  - Potential weaknesses involve unhandled auth state transitions, such as expired session tokens causing silent request failures without user notification.
- **System Impact:** Disrupted user sessions or permission errors during network requests upon token expiration.
- **Minimal Non-Intrusive Security Fixes:**
  - Register an `onIdTokenChanged` auth state listener to seamlessly handle credential refreshes before dispatching network requests.
  - Provide clear, user-friendly authentication error messaging on sign-in failures without exposing system details.

### 6. Authorization Bypass
- **Risk Level:** Low / Medium
- **Vector Assessment:**
  - Security rules defined in `firestore.rules` enforce access control:
    ```rules
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == userId;
    }
    match /personality_tests/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
    ```
  - `/users/{userId}` permits authenticated users (`request.auth != null`) to read user documents, supporting collaborative and social discovery features.
  - Write permissions for `/users/{userId}` and `/personality_tests/{userId}` strictly enforce identity ownership (`request.auth.uid == userId`).
- **System Impact:** Modifying other users' data is prevented by rules. Profile reads are restricted to authenticated users to preserve collaborative functionality.
- **Minimal Non-Intrusive Security Fixes:**
  - Maintain the existing `/users/{userId}` read rule to preserve collaborative features while keeping write permissions strictly scoped to `request.auth.uid == userId`.
  - Add schema field validation rules in `firestore.rules` to enforce required field types without altering access permissions.

### 7. Exposed Secrets
- **Risk Level:** Low
- **Vector Assessment:**
  - Sensitive environment variables such as `GEMINI_API_KEY` are injected at runtime via AI Studio injection rules and specified in `.env.example`.
  - Local environment files (`.env.local`) are excluded in `.gitignore`.
  - Public Firebase client configurations in `firebase-applet-config.json` contain public identifiers (`apiKey`, `projectId`, `appId`), which are safe to expose in web app clients per Firebase architecture.
- **System Impact:** Minimal risk of secret leakage under current runtime configuration.
- **Minimal Non-Intrusive Security Fixes:**
  - Ensure API calls to Gemini are routed through serverless/Cloud Run endpoints so `GEMINI_API_KEY` is never bundled into client-side JS assets.
  - Keep `.env.local` explicitly listed in `.gitignore`.

### 8. Unsafe Storage
- **Risk Level:** Medium
- **Vector Assessment:**
  - Storing user architecture state or authorization tokens in unencrypted client storage (`localStorage` or `sessionStorage`) risks exposure to client-side scripts or XSS vulnerabilities.
- **System Impact:** Token theft or data exposure if client-side storage is read by unauthorized scripts.
- **Minimal Non-Intrusive Security Fixes:**
  - Rely on Firebase Auth SDK's secure in-memory or `IndexedDB` token management rather than storing raw tokens in `localStorage`.
  - Flush cached local application state upon user logout.

### 9. Token Leaks
- **Risk Level:** Low / Medium
- **Vector Assessment:**
  - Auth tokens or keys could potentially leak through HTTP `Referrer` headers, URL parameters, or client console logs.
- **System Impact:** Intercepted tokens could enable temporary unauthorized API calls until expiration.
- **Minimal Non-Intrusive Security Fixes:**
  - Include `Referrer-Policy: strict-origin-when-cross-origin` headers on responses.
  - Pass authentication credentials strictly via HTTP `Authorization` headers, never via URL query parameters.

### 10. Insecure API Endpoints
- **Risk Level:** Medium
- **Vector Assessment:**
  - Cloud Run services and serverless API endpoints communicating with Gemini AI must validate request signatures and payload boundaries.
- **System Impact:** Unprotected endpoints could lead to unauthorized API usage or resource exhaustion.
- **Minimal Non-Intrusive Security Fixes:**
  - Verify Firebase ID tokens server-side using Firebase Admin SDK (`verifyIdToken`) for all protected endpoints.
  - Enforce payload body size limits (e.g., max 2MB) and schema validation on API routes to prevent service abuse.

---

## Summary Matrix

| Security Vector | Risk Level | Current Posture / Vector Assessment | Minimal Security Recommendation | Impact |
| :--- | :--- | :--- | :--- | :--- |
| **1. XSS** | Medium | Dynamic text rendering from user/AI inputs | Standard JSX escaping; sanitize dynamic HTML with DOMPurify; set CSP headers | Prevents malicious client script execution |
| **2. SQL Injection** | Low / N/A | Uses Firebase Firestore (NoSQL); no SQL engines present | Maintain parameterization if SQL components are added | N/A (Preventative) |
| **3. NoSQL Injection** | Low | Firestore SDK parameterizes query filters | Validate path parameter string formats before building queries | Prevents parameter pollution |
| **4. CSRF** | Medium | Header-based Bearer auth; endpoints need origin validation | Enforce strict CORS and validate `Origin`/`Referer` headers on Cloud Run endpoints | Blocks cross-origin request forgery |
| **5. Authentication Weaknesses** | Medium | Managed by Firebase Auth; short-lived tokens | Attach `onIdTokenChanged` listener to handle expired session transitions gracefully | Ensures smooth re-authentication |
| **6. Authorization Bypass** | Low / Medium | `firestore.rules` restricts writes; `/users/{userId}` allows auth read for social features | Preserve existing rules; add schema field type checks | Preserves collaborative reads while blocking unauthorized writes |
| **7. Exposed Secrets** | Low | Secrets injected at runtime; `.env.local` gitignored | Route `GEMINI_API_KEY` calls through serverless backend endpoints | Prevents secret exposure in client bundles |
| **8. Unsafe Storage** | Medium | Client-side cached state | Rely on Firebase SDK `IndexedDB` token storage; avoid `localStorage` for tokens | Protects tokens from local storage extraction |
| **9. Token Leaks** | Low / Medium | Potential token leak via query params or referrers | Pass tokens in `Authorization` headers only; set `Referrer-Policy: strict-origin-when-cross-origin` | Prevents token leakage in logs and referrers |
| **10. Insecure API Endpoints** | Medium | Cloud Run endpoints invoking Gemini AI APIs | Verify Firebase ID tokens server-side with `verifyIdToken`; enforce payload limits | Prevents API abuse and unauthorized billing |
