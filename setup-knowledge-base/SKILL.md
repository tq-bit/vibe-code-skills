---
name: setup-knowledge-base
description: "Load when the user wants to set up, scaffold, migrate, or restructure a project's curated knowledge base in .vibe/knowledge/ — an index.md that must contain a mermaid mindmap, a cross-reference flowchart and a file table, per-domain folders whose files mirror the actual code components, a rules.md and recent-changes.md per domain, and the AGENTS.md knowledge table. Interview-driven: inspects the codebase first, asks which domains and components exist, then derives the matching structure. Pairs with the curate-knowledge skill for day-to-day maintenance."
---

# Setup Knowledge Base

Scaffolds a reusable knowledge-base structure under `.vibe/knowledge/` for any repository, and migrates existing flat knowledge files into it. The structure mirrors the actual implementation: every knowledge file corresponds to a real component or concern in the code. Once set up, day-to-day maintenance (delta updates, curation after large changes) is governed by the `curate-knowledge` skill — load it for follow-up curation tasks.

## Non-negotiables

Every knowledge base created by this skill must have, regardless of stack:

1. **`index.md`** — the entry point, always containing all three, in this order:
   - a mermaid **mindmap** of the knowledge structure (one branch per domain, one leaf per file, plus shared docs and entry points)
   - a mermaid **flowchart** of the cross-references between all knowledge files — an arrow means *"document A references document B"*; extract every edge from the actual files (grep), **never invent edges**
   - a **table** with columns: Component | File | Read before working on
2. **One folder per code domain** (e.g. `backend/`, `frontend/`) with **one file per component or concern, mirroring the actual code** (e.g. `handlers.md` for `srv/documents/`, `models.md` for `webapp/model/`). Never invent components the code does not have; never document code that does not exist.
3. **`rules.md` per domain**: MCP server consultation rules, verification requirements, change guidelines, and pointers to cross-cutting docs.
4. **`recent-changes.md` per domain**: only changes NOT documented in the other files. Entries whose behavior is fully documented elsewhere are removed with a link to the file that documents them.
5. **AGENTS.md knowledge table** — merged into the project's AGENTS.md (never replacing unrelated content): the `index.md` row first, then one row per knowledge file.
6. Conventions: kebab-case filenames, English, brief and factual entries that reference source files by path, cross-reference between files instead of copying, never commit (follow the repository's own rules).

## Workflow

### 1. Inspect before asking

Never ask what the repository can answer. List the top-level directories, read the manifests (`package.json`, `mta.yaml`, `pom.xml`, `requirements.txt`, `go.mod`, ...), detect frameworks, services, and build tooling. The interview only covers what inspection cannot tell you.

### 2. Interview the user

Use `ask_user_question` (one widget, a few tabs — the user may answer several at once). Skip every question the inspection or the user already answered:

1. **Domains**: Which parts of the codebase get their own knowledge folder? (backend only / frontend only / both / CLI / infra / mobile / more)
2. **Components per domain**: For each domain, which components or concerns deserve their own file? Offer the discovered candidates as multiple choice (e.g. a CAP backend: handlers, AI factory, object store, jobs; a UI5 frontend: views-and-controllers, models).
3. **MCP consultation rules**: Which MCP server must be consulted per domain, with which tools, and what counts as non-trivial? (Default definition: a new file is created OR the change is ≥50 LoC OR two or more files are changed.)
4. **Cross-cutting docs**: Which exist or should exist — domain model (`entities.md`), deployment (`deployment.md`), database (`database-deploy.md`), FAQ (`faq.md`)? Only create files that apply.
5. **Mandatory rules**: Which coding rules must be captured in each domain's `rules.md` (error handling, naming, validation patterns, authorization-change approval)?

### 3. Derive and confirm the structure

Map each domain → folder and each component/concern → file, mirroring the code directories. Present the resulting structure tree to the user in one short block before creating anything.

### 4. Create the knowledge base

- `index.md` from `templates/index.md` (same directory as this SKILL.md)
- the domain folders with `rules.md`, the component files, `recent-changes.md`
- the AGENTS.md sections from `templates/agents.md` — if an AGENTS.md already exists, merge: add the knowledge table and maintenance section, update prose that claims knowledge lives in a single file, never delete unrelated content
- write the domain files first, then extract the flowchart edges by grepping the finished files for their cross-references

### 5. Report

Structure tree, which file to read for which task, and the pointer to `curate-knowledge` for maintenance.

## Generic structure

```
.vibe/knowledge/
├── index.md                     # mindmap + cross-reference flowchart + table (mandatory)
├── <domain>/                    # one folder per code domain (e.g. backend/, frontend/)
│   ├── rules.md                 # MCP consultation + verification + change guidelines (mandatory per domain)
│   ├── <component>.md           # one per component/concern, mirroring the code
│   └── recent-changes.md        # only undocumented changes (mandatory per domain)
├── entities.md                  # persistent data model (if applicable)
├── deployment.md                # deployment/production wiring (if applicable)
├── <domain|component>-deploy.md           # domain or component specifics (if applicable)
└── faq.md                       # brief Q&As (optional but recommended)
```

## Worked example (SAP CAP repository: backend + frontend)

A complete rendered example lives in `examples/cap-postgres-objectstore/` (relative to this skill directory): a CAP + PostgreSQL + Object Store + UI5 project with `index.md` (mindmap, extracted cross-reference flowchart, file table), both domain folders with `rules.md`/component files/`recent-changes.md`, all cross-cutting docs (`entities.md`, `deployment.md`, `<domain|component>-deploy.md`, `faq.md`) and the merged project `AGENTS.md`. Read files there with `read_file` when a concrete shape of a target artifact is needed.

Reference implementation:

```
.vibe/knowledge/
├── index.md                     # mindmap + flowchart + table
├── backend/
│   ├── rules.md                 # CAP MCP consultation, verification, change guidelines, authorization summary
│   ├── handlers.md              # srv/documents/ handler tree, extraction pipeline, locking, websocket, srv/util/, @types/
│   ├── ai-factory.md            # srv/ai/ provider factory (OCR/LLM)
│   ├── ai-prompt-assembly.md    # prompt building (component deep-dive)
│   ├── object-store.md          # srv/store/ factory + archiving flow
│   ├── jobs.md                  # srv/jobs/ actions, internal scheduler, dev sidecar
│   └── recent-changes.md        # dev CLI tooling only
├── frontend/
│   ├── rules.md                 # UI5 MCP consultation, OData v4 config, element IDs, view guidelines
│   ├── views-and-controllers.md # BaseController, controller hierarchy, routing, i18n, websocket client, utilities, types
│   ├── models.md                # webapp/model/ hierarchy, OData v4 entity creation
│   └── recent-changes.md        # settled fixes & known debt only
├── entities.md
├── deployment.md
├── <domain|component>-deploy.md
└── faq.md
```

A project without a frontend omits `frontend/`; a CLI-only project gets a `cli/` domain folder instead — the folders always mirror what the code actually contains.

## index.md rules

Use `templates/index.md`. Beyond the template:

- Mindmap node text must not contain `/`, `(`, `)`, or `:` — the mindmap parser treats them as shape delimiters. Write `db, srv` instead of `db/, srv/`, and `Q and As` instead of `Q&As:`.
- Flowchart: group nodes per domain with `subgraph` or `classDef` coloring; add `AGENTS.md` and `README.md` as root nodes pointing to `index.md`; extract every edge from the finished files by grepping for cross-file references (both full paths like `knowledge/backend/rules.md` and sibling references like `` `rules.md` ``).
- Keep the order: mindmap → "Knowledge Graph (cross-references)" section → "Knowledge Files" table.

## Migration mode (existing flat knowledge files)

When the project already keeps knowledge in single flat files (e.g. one `backend.md`):

1. Split them **verbatim by `##` headings via script** — never retype content; assert the expected heading order before splitting so nothing is lost.
2. Distill each `recent-changes` section: keep only entries not documented in the other files, link the removed entries to their new home.
3. Update every reference afterwards: the AGENTS.md knowledge table and prose, README.md, and cross-references in all knowledge files.
4. Delete the originals only after a repo-wide grep confirms no stale references remain.

## After setup

- NEVER do a full setup of the knowledge base if the knowledge folder exists already - always migrate or restructure instead
- Update the AGENTS.md knowledge table whenever a knowledge file is added or renamed.
- Load `curate-knowledge` for maintenance: delta updates after changes, FAQ brevity rules, curation passes after large changes.
