---
name: implement-cap-cf-deployment
description: Load when the user wants to deploy a CAP application to SAP BTP Cloud Foundry and/or SAP Build Work Zone, or asks to check, create or adjust deployment files (MTA descriptors, xs-security role models, destination service config, approuter routing, PostgreSQL/object-store instance options) so the deployment succeeds and the app is importable into Work Zone. The skill compares the project's files against full reference files and applies the minimal adjustments (HTML5Runtime_enabled, grant-types, redirect-uris, websockets, ServiceInstanceName, …).
---

# Implement CAP Cloud Foundry Deployment

Goal: check a CAP project's deployment files against proven, complete references and adjust them for a successful Cloud Foundry deployment whose UI5 app is servable via a standalone approuter AND importable into SAP Build Work Zone.

Hard rules for every adjustment:

- **Minimal diff**: adjust only what blocks a successful deployment; no unrelated refactoring.
- **Placeholders stay placeholders**: references use `com.example.cap.myapp` / `<mta-id>` / `<app>` — map them onto the project's real IDs, never the other way around.
- **Secrets never go into the MTA** or any deployment file — they are set post-deploy via `cf set-env <srv-app> <SECRET_ENV> <value> && cf restart <srv-app>`.
- Do not rename the project's existing service instances; bind or create per the project's strategy (binding variant vs. full variant).

## Relevant project files and their references

Read this table first. Column 1 lists the files to locate in the project; column 2 the reference file to load (path relative to this skill directory) **before** adjusting that file.

| # | Project file (check / adjust) | Reference file (load on demand) |
|---|---|---|
| 1 | `mta.yaml` — binding variant (binds existing instances) | `references/mta-binding-variant-resources.yaml` |
| 2 | `mta-premium.yaml` — full variant (creates all instances) | `references/mta-premium.yaml` |
| 3 | `.deploy/xs-security.json` — role model (with Job Scheduler) | `references/xs-security.json` |
| 4 | `.deploy/xs-security-minimal.json` — role model without Job Scheduler (binding variant) | `references/xs-security-minimal.json` |
| 5 | `.deploy/pg-options.json` — PostgreSQL instance parameters | `references/pg-options.json` |
| 6 | `.deploy/objectstore-options.json` — Object Store instance parameters | `references/objectstore-options.json` |
| 7 | `.deploy/app-router/xs-app.json` — approuter routing | `references/app-router-xs-app.json` |
| 8 | `.deploy/app-router/package.json` — approuter runtime | `references/app-router-package.json` |
| 9 | — (no project file) Work Zone / HTML5 runtime / XSUAA gotchas | `references/work-zone-gotchas.md` |

Paths and names vary per project: MTA descriptors usually sit at the project root; the `.deploy/` files wherever the descriptors' `path:` entries point. Locate the actual files; if a file from the table is missing, create it from its reference (adapted to the project's IDs).

## Workflow

1. **Read all files that are relevant.** Inventory the project against the table above: which files exist, which are missing, which variant(s) the project uses (binding, full, or both). Use the table as the checklist; do not skip a file just because it looks fine.
2. **For each file, read the respective reference file** (`read_file` on the skill-relative path). Load a reference even when the project file looks close — the differences that matter (see step 3) are easy to miss.
3. **View the difference and make the necessary adjustments.** Compare each project file against its reference and adjust for a successful deployment with Work Zone integration. The points most commonly missing — verify each explicitly:
   - **`HTML5Runtime_enabled: true` in the destination service resource config** plus the instance-level `srv-api` destination in `init_data` (`HTML5.ForwardAuthToken: true`) — without it the app is NOT importable into Work Zone's content provider, and the managed approuter cannot route the app's `/odata` / `/ws` calls (it resolves destinations from the instance, not from the app metadata registered by the app-deployer).
   - **Explicit `grant-types` list** on the xsuaa resource: `authorization_code` + `client_credentials` + `refresh_token` + `user_token` — a partial list silently disables the respective flow (approuter login, technical clients, Work Zone token exchange).
   - **`redirect-uris` + `requires: app-api`** on the xsuaa resource — only when a standalone approuter module exists (its login needs the redirect; pure Work Zone deployments don't).
   - **`"websockets": { "enabled": true }`** in both xs-app.json files (standalone approuter AND the app's own xs-app.json shipped in the UI5 zip).
   - **`ServiceInstanceName` in the destination content module = the real CF instance name** (resolved against the space), NOT the MTA resource name.
   - **`$XSSERVICENAME(...)` placeholder must match the MTA resource name exactly** (watch ID prefixes of parallel stacks).
   - **`role-template-references` as prefixed strings** — `["$XSAPPNAME.admin"]`, never bare names or objects.
   - **xsappname rules**: max 100 chars, no dots, unique per subaccount (key on the space when the org name does not fit).
   - **Parallel stacks**: a second app-host instance with the same app ID fails with CODE 1001 — bind the existing repo-host instead; the app-deployer updates the app in place.
   - **Instance options match the plan**: PostgreSQL `memory` only 2 or 4 for `standard` (8+ = premium), no config body for the free plan; storage is immutable after creation.
4. **If the user asks a follow-up question the references cannot answer**, conduct a web search (SAP Help Portal, CAP docs) before answering. Cite what was found; mark anything unconfirmed.
5. **Summarize the files and the adjustments made** — per file: checked ✓ / adjusted (what exactly and why) / created (from which reference); plus anything deliberately left untouched. Include the deploy follow-ups that remain manual (role collection assignment, secrets via `cf set-env`, site assignment in Work Zone).
6. **If a web search was conducted**, end the summary with a suggestion how to adjust this skill for future usage: which reference file should absorb the finding (usually `references/work-zone-gotchas.md`, or the reference file of the affected project file), so the same question is answered from the references alone next time.

## Summary requirements

Every summary must state: variant(s) detected, per-file verdict (compliant / adjusted / created / missing), the Work Zone-critical flags verified (`HTML5Runtime_enabled`, `init_data` srv-api, grant-types, websockets, ServiceInstanceName), and the remaining manual steps.
