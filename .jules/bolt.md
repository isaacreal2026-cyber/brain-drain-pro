## 2026-07-21 - Empty Repository Context
**Learning:** The repository contains only workspace configuration files (such as package.json, pnpm-workspace.yaml, tsconfig.json, firestore.rules, and firebase-applet-config.json) and lacks any application source code or directories like 'artifacts/*' or 'scripts'. Running 'pnpm install' alters pnpm-lock.yaml by stripping missing workspace dependencies.
**Action:** In metadata-only workspaces with no source code, do not attempt to install packages or perform code/performance changes. Document this state in .jules/bolt.md and stop.

## 2026-08-11 - Security Audit & Analysis of Brain Builder
**Learning:** Conducted a comprehensive security audit across 10 areas. Confirmed that as a configuration-only repo, the attack surface is minimal. No application code, server routing, or databases other than Google Cloud Firestore are utilized. The Firebase Client API key in config is public by design, which is not a secret exposure.
**Action:** Drafted `SECURITY_AUDIT.md` covering all 10 areas. Documented the permissive read rules for `/users/{userId}` in Firestore rules as an intentional trade-off to support collaborative/social profile sharing. Disallowed any breaking security rule edits to keep functionality intact. Recommending GCP-level API key restrictions and proper data categorization instead.
