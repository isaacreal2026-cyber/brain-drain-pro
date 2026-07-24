## 2026-07-23 - Empty Repository Context
**Learning:** The repository contains only workspace configuration files (such as package.json, pnpm-workspace.yaml, tsconfig.json, firestore.rules, and firebase-applet-config.json) and lacks any application source code or directories like 'artifacts/*' or 'scripts'. Running 'pnpm install' alters pnpm-lock.yaml by stripping missing workspace dependencies.
**Action:** In metadata-only workspaces with no source code, do not attempt to install packages or perform code/performance changes. Document this state in .jules/bolt.md and stop.
