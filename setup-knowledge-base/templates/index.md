# Project Knowledge Index

{{One-sentence entry point: what this directory holds and that AGENTS.md defines when consultation is mandatory.}} The mindmap below shows the knowledge structure at a glance; the flowchart below it maps the cross-references between the files.

<!-- mindmap gotchas: node text must not contain "/", "(", ")" or ":" — the parser treats them as shape delimiters. Write "db, srv" instead of "db/, srv/" and "Q and As" instead of "Q&As". One leaf per knowledge file; branch per domain. -->

```mermaid
mindmap
  root(({{Project Name}} Knowledge))
    {{Domain}} — {{code dirs, comma separated}}
      rules.md — dev rules, MCP consultation, {{pointers}}
      {{component}}.md — {{scope}}
      recent-changes.md — {{undocumented changes}}
    Domain docs
      entities.md — data model
      deployment.md — {{deployment scope}}
      database-deploy.md — {{database scope}}
      faq.md — brief Q and As
    Entry points
      index.md — this file
      AGENTS.md — consultation rules
```

## Knowledge Graph (cross-references)

An arrow means *"document A references document B"*; every edge was extracted from the actual cross-references in the files.

```mermaid
flowchart LR
    %% ── Entry points ──
    AGENTS["AGENTS.md<br/>session rules"]
    README["README.md"]
    INDEX["index.md<br/>(this graph)"]

    %% ── {{Domain}} ──
    DR["{{domain}}/rules.md"]
    DC1["{{domain}}/{{component}}.md"]
    DRC["{{domain}}/recent-changes.md"]

    %% ── Shared domain docs ──
    ENT["entities.md"]
    DEP["deployment.md"]
    DB["database-deploy.md"]
    FAQ["faq.md"]

    %% ── References (extracted from the finished files — grep, never invent) ──
    AGENTS --> INDEX
    README --> INDEX
    AGENTS --> DR

    DR --> DC1
    DR --> DEP
    DC1 --> DR
    DRC --> DC1
    DRC --> DEP

    %% ── Styling (one classDef and color per domain) ──
    classDef root fill:#efe3fa,stroke:#6c32a9,color:#32363a;
    classDef {{domain}} fill:#e3f0ff,stroke:#0a6ed1,color:#32363a;
    classDef docs fill:#e2f3e8,stroke:#107e3e,color:#32363a;
    class AGENTS,README,INDEX root;
    class DR,DC1,DRC {{domain}};
    class ENT,DEP,DB,FAQ docs;
```

## Knowledge Files

| Component | File | Read before working on |
|-----------|------|------------------------|
| {{Domain}} rules | `{{domain}}/rules.md` | Any file in {{code dirs}} |
| {{Component}} | `{{domain}}/{{component}}.md` | {{trigger, e.g. any task touching the component's code paths}} |
| Recent changes | `{{domain}}/recent-changes.md` | {{Domain}} work context (undocumented fixes & known debt) |
| Domain model | `entities.md` | Entities and their properties/associations |
| Deployment | `deployment.md` | {{Deployment triggers}} |
| Database | `database-deploy.md` | {{Database triggers}} |
| FAQ | `faq.md` | Quick lookups |
