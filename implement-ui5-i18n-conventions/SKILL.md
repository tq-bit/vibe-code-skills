---
name: implement-ui5-i18n-conventions
description: Load when creating or modifying UI5 views, fragments, dialogs, or controls that display text. Defines the mandatory i18n key naming convention and translation requirements.
---

# UI5 i18n Conventions

All user-visible text in views, fragments, dialogs, and controls must be externalized to the component i18n resource bundle. Never hardcode display strings. NEVER internationalize Logs.

## Key naming convention

- Keys are PascalCase, for example `DocumentContentInfo`, `InstructionBlockInfo`, `HeaderData`, `PositionData`.
- Names describe the content or purpose, not the control type or placement.
- Keys must be unique across the application bundle.
- Keys describe WHERE texts are used with the following pattern: `<view/control where it's used>.<ui-control where it's used>.<first three words in PascalCase of the english translation>`
- If an i18n key is used over multiple views/controls, the first part of the key must be equal to "App"
- If an i18n key is used over multiple UI-Controls, the second part of the key must be equal to "Shared"
- If an i18n value has more than 8 words, you may simplify the third part of the key to maintain readability

## Examples

- "App.Shared.AreYouSure=Are you sure you want to delete this item?" describes a **generic message** used in **multiple components and UI-controls** over the app
- "App.Button.Back=Back" describes a **Back text** on a **button UI-control** that's **used over multiple views**
- "DocumentExtractionDetail.Panel.DocumentPreview" describes a specific **Panel** title that's used in the **DocumentExtractionDetail** - view
- "DocumentSchemaCreate.Wizard.Step1Title=Upload Document" describes a specific **Wizard Step** in the **DocumentSchemaCreate** view that describes the first step of the Wizard

## Required workflow

1. When adding any new text element, add the corresponding key and translation to the i18n properties file(s) in the same change.
2. Retrieve translated text in controllers via the BaseController helper `getAppResourceBundleText(identifier)` rather than direct bundle access.
3. In XML views, bind text through the i18n model (e.g. `{i18n>Key}`).
4. When renaming or removing a text element, update or remove its i18n key in the same change to avoid orphaned entries.
5. Search the codebase for an existing suitable key before adding a new one.