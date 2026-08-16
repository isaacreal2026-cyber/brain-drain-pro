## 2026-07-21 - Empty Repository Context
**Learning:** The repository contains only workspace configuration files (such as package.json, pnpm-workspace.yaml, tsconfig.json, firestore.rules, and firebase-applet-config.json) and lacks any application source code or directories like 'artifacts/*' or 'scripts'. Running 'pnpm install' alters pnpm-lock.yaml by stripping missing workspace dependencies.
**Action:** In metadata-only workspaces with no source code, do not attempt to install packages or perform code/performance changes. Document this state in .jules/bolt.md and stop.

## 2026-08-16 - Stress Testing Audit Evaluation
**Learning:** Performing stress testing simulations on Brain Builder's architectural parameters highlights critical bottleneck vectors across 9 key areas: high traffic, rapid API requests, concurrent users, network failures, database delays, server restarts, large datasets, memory pressure, and CPU pressure.
**Action:** Detailed stress testing findings and behavior-preserving mitigation recommendations (e.g., Firestore transactions, debouncing, `enableIndexedDbPersistence`, D3 cleanup hooks, Web Workers) are preserved in `STRESS_TEST_REPORT.md`.
