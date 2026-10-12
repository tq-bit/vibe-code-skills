# AGENTS.md template — knowledge sections
<!-- Merge into the project's AGENTS.md. Replace every {{...}} placeholder with the interview results; delete rows and bullets that do not apply to the project. Never delete unrelated existing content. -->

## Knowledge Directory

Domain knowledge is maintained in separate files inside `.vibe/knowledge/`. Read the file matching your task's domain before answering questions or making changes:

| File | Scope | Read before working on |
|------|-------|------------------------|
| `.vibe/knowledge/index.md` | Entry point: mermaid mindmap of the knowledge structure, a flowchart of the cross-references between all knowledge files, and a table of when to read each | Start of any knowledge-driven task |
| `.vibe/knowledge/{{domain}}/rules.md` | {{Domain}} rules: MCP server consultation, verification requirements, change guidelines, pointers | Any file in {{code directories}} |
| `.vibe/knowledge/{{domain}}/{{component}}.md` | {{Component scope, one row per component file}} | Any task touching {{code paths}} |
| `.vibe/knowledge/{{domain}}/recent-changes.md` | {{Domain}} changes not yet documented in the other files | {{Domain}} work context |
| `.vibe/knowledge/entities.md` | Domain model: every entity with its properties and associations | Any task requiring the project's entities |
| `.vibe/knowledge/deployment.md` | {{Deployment scope: variants, roles, secrets, build/deploy commands}} | Any deployment/production task |
| `.vibe/knowledge/database-deploy.md` | {{Database scope: plans, options, sizing}} | Any task changing the database plan or options |
| `.vibe/knowledge/faq.md` | Brief questions and brief answers | Quick lookups; extend with new brief Q&As instead of long-form knowledge sections |

The domain rules files ({{list of rules.md paths}}) contain the mandatory MCP server consultation rules:

- **{{Domain}} ({{mcp-server}})**: required for non-trivial {{domain}} question or change — {{tool list, e.g. search_docs, search_model}}
- Hint: Non-trivial means either: 1. A new file is created OR 2. The planned change consists of 50 or more LoC OR 3. Two or more files are changed

If the relevant MCP server is unavailable, state this limitation explicitly and request permission to proceed.

## Maintenance

Extend the knowledge files (not this file) whenever:

- a new {{component type, e.g. service or service binding}} is added → the matching file in `.vibe/knowledge/{{domain}}/`
- a new {{shared base class}} method is added → `.vibe/knowledge/{{domain}}/{{component file}}.md`
- a new {{pattern, e.g. view pattern}} is implemented → the matching domain file
- a new development rule is added or an existing rule is modified → the matching domain `rules.md`
- a new entity is added or an existing entity's properties/associations change → `.vibe/knowledge/entities.md`
- low-effort, high-impact changes are made → `.vibe/knowledge/faq.md`

Keep this file limited to general instructions and references to the knowledge directory.
