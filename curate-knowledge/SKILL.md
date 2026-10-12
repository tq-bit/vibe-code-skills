---
name: curate-knowledge
description: Load when the user asks to write or update documentation, keep project knowledge, or curate the knowledge base. Also load automatically after a large change - one that produced more than 80,000 output tokens or touched more than 3 files at the same time - to capture what changed in .vibe/knowledge.
---

# Curate Knowledge

Keep the project's curated knowledge in `.vibe/knowledge/` accurate and useful. Curate on explicit user request (writing docs, updating knowledge) and after any large change: one that included more than 80,000 output tokens or touched more than 3 files at the same time.

## Target structure

`.vibe/knowledge/` uses exactly this layout:

1. **`index.md`** - Entry point. A table of the project's components (frontend, backend, jobs, stores, ...) with a brief explanation of each and a link to the file that documents it.
2. **`frontend/`** - Folder with multiple files, one per frontend component or concern (views, models, utilities, ...). Only present if the project contains frontend components.
3. **`backend/`** - Folder with multiple files, one per backend component or concern (handlers, services, factories, jobs, utilities, ...). Only present if the project contains backend components.
4. **`entities.md`** - The persistent data objects of the project, ideally described using a mermaid `erDiagram`. Add brief per-entity notes only where the diagram cannot express them.
5. **`faq.md`** - Low-effort, high-impact questions and answers about the project. Every entry is a brief question and a brief answer - nothing else. Add an entry whenever you learn something a future developer will search for.

Omit `frontend/` or `backend/` when the project has no such components; never keep empty folders.

## Curation workflow

1. Read the existing `.vibe/knowledge/` files and the changed code before writing - never curate from memory alone.
2. Update the affected files in place; do not duplicate information that another knowledge file already holds. Cross-reference between files instead of copying.
3. A curation pass after a large change is a delta update: capture what changed (new files, changed contracts, new conventions, gotchas) in the matching files and update `index.md`. Do not rewrite unaffected sections.
4. Keep entries brief and factual. Reference source files by path so readers can jump to the code. Do not explain every function - that is what the code and these references are for.
5. If `.vibe/knowledge/` exists in a different layout, migrate it: move content into the target structure without losing information, then update any references to the old paths (for example in `AGENTS.md` or `README.md`).
6. When a new knowledge file is added or renamed, update the knowledge table in the project's `AGENTS.md` if it maintains one.
7. Never commit; follow the repository's own rules (`AGENTS.md`).

## FAQ rules

`faq.md` may only ever contain brief questions and brief answers. No long-form sections, no code walkthroughs - move those into `frontend/`, `backend/`, or `entities.md` instead.
