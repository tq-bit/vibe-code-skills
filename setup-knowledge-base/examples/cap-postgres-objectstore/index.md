# Project Knowledge Index

Minimal SAP CAP template: one service, one entity — attachments implemented by hand (no attachment plugin) on top of a provider-agnostic object store, with a one-page UI5 upload app and a Cloud Foundry MTA. AGENTS.md defines when consultation of the knowledge files is mandatory. The mindmap below shows the knowledge structure at a glance; the flowchart below it maps the cross-references between the files.

```mermaid
mindmap
  root((cds-attachments-demo Knowledge))
    Backend — db, srv, scripts
      rules.md — dev rules, MCP consultation, deployment pointers
      ticket-service.md — service handlers, content lifecycle
      object-store.md — store factory, drivers, BTP binding facts
      cli.md — dev CLI menu and commands
      recent-changes.md — open debt
    Frontend — app ticket-upload
      rules.md — UI5 MCP consultation, OData v4 rules
      ticket-upload.md — app structure, upload and delete flows
      recent-changes.md — settled fixes
    Domain docs
      entities.md — data model
      deployment.md — Cloud Foundry MTA, roles, Work Zone
      database-deploy.md — PostgreSQL plans and schema deployer
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

    %% ── Backend ──
    BR["backend/rules.md"]
    BTS["backend/ticket-service.md"]
    BOS["backend/object-store.md"]
    BCLI["backend/cli.md"]
    BRC["backend/recent-changes.md"]

    %% ── Frontend ──
    FR["frontend/rules.md"]
    FTU["frontend/ticket-upload.md"]
    FRC["frontend/recent-changes.md"]

    %% ── Shared domain docs ──
    ENT["entities.md"]
    DEP["deployment.md"]
    DB["database-deploy.md"]
    FAQ["faq.md"]

    %% ── References (extracted from the finished files) ──
    AGENTS --> INDEX
    README --> INDEX
    AGENTS --> BR
    AGENTS --> FR

    BR --> BTS
    BR --> BOS
    BR --> BCLI
    BR --> BRC
    BR --> DEP
    BR --> DB
    BOS --> BTS
    BOS --> BR
    BCLI --> DEP
    BRC --> DEP

    FR --> FTU
    FR --> FRC
    FR --> DEP
    FTU --> FR
    FRC --> FR
    FRC --> FTU

    ENT --> BTS
    DEP --> BOS
    DEP --> BRC
    DEP --> DB
    DB --> DEP
    FAQ --> DEP
    FAQ --> BRC
    FAQ --> FR

    %% ── Styling (one classDef and color per domain) ──
    classDef root fill:#efe3fa,stroke:#6c32a9,color:#32363a;
    classDef backend fill:#e3f0ff,stroke:#0a6ed1,color:#32363a;
    classDef frontend fill:#fff2e8,stroke:#d04343,color:#32363a;
    classDef docs fill:#e2f3e8,stroke:#107e3e,color:#32363a;
    class AGENTS,README,INDEX root;
    class BR,BTS,BOS,BCLI,BRC backend;
    class FR,FTU,FRC frontend;
    class ENT,DEP,DB,FAQ docs;
```

## Knowledge Files

| Component | File | Read before working on |
|-----------|------|------------------------|
| Backend rules | `backend/rules.md` | Any file in `db/`, `srv/`, `scripts/` |
| TicketService | `backend/ticket-service.md` | Any task touching `srv/ticket-service.cds` or the handlers in `srv/ticket-service.ts` |
| Object store | `backend/object-store.md` | Any task touching `srv/store/`, `srv/util/env.ts` or the objectstore binding |
| Dev CLI | `backend/cli.md` | Any task touching `scripts/` or the interactive menu commands |
| Backend recent changes | `backend/recent-changes.md` | Backend work context (open debt) |
| Frontend rules | `frontend/rules.md` | Any file in `app/ticket-upload/webapp/` |
| UI app | `frontend/ticket-upload.md` | Any task touching the views, controllers, i18n or manifest of `app/ticket-upload` |
| Frontend recent changes | `frontend/recent-changes.md` | Frontend work context (settled fixes) |
| Domain model | `entities.md` | Entities and their properties/associations |
| Deployment | `deployment.md` | Any deployment, mta.yaml, .deploy/, binding or production task |
| Database | `database-deploy.md` | Any task changing the PostgreSQL plan, options or schema deployment |
| FAQ | `faq.md` | Quick lookups |
