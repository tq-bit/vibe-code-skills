# Frontend: Recent Changes

Only changes NOT documented in the other knowledge files. Settled items move
to their home file with a link.

- **Settled 2026-10-09 — delete buttons did nothing on OK.** Root cause:
  `await MessageBox.confirm(...)` (returns undefined — no Promise) and a
  call to the nonexistent v4 `Context#deleted()`. Both API facts are now in
  `rules.md`; the delete flow itself is documented in `ticket-upload.md`.
