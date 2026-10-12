# Backend: Recent Changes

Only changes NOT documented in the other knowledge files. Settled items move
to their home file with a link.

- **Open debt — `mta.yaml` UI module `npm ci` prunes root node_modules**
  (reproduced 2026-10-09). `app/ticket-upload` is a workspace of the root
  `package.json`; an `npm ci` executed inside the workspace dir (which is what
  the UI module's build commands do during `mbt build`) rebuilds the install
  tree for that workspace only and deletes every root dependency
  (`@sap/cds`, `tsx`, `cds-plugin-ui5`, ...). Symptom afterwards:
  `npm start` fails with `cds-serve: not found`, `cds watch` (global cds-dk)
  serves OData but silently without the `.ts` handlers and without the UI
  mount. Recovery: `npm ci` in the project root. Candidate fix (unverified):
  change the UI module build command to `npm install --workspaces=false`.
  The general `mbt build` node_modules gotchas are in ../deployment.md.
