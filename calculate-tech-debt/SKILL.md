---
name: calculate-tech-debt
description: Load this skill when the user asks for a code review, sanity check, technical debt assessment, or a per-file quality/debt rating with a final score for a codebase, backend, or module. Produces a read-only review that rates every source file 0-10 (0 = best practice, 10 = spaghetti) against the user's coding guidelines and computes a line-weighted final score.
---

# Calculate Technical Debt

Perform a **read-only** review. Do not modify, create, or delete any file in the target project. Never commit. If the user asks for fixes afterwards, that is a separate task after the review is delivered.

## Process

1. **Load the guidelines first.** Read the user-level `~/.vibe/AGENTS.md` and every project `AGENTS.md` on the path from the working directory to the repo root. These define the scoring criteria (naming conventions, error handling, style, architecture rules). Score against what is actually written there, not generic taste.
2. **Build the inventory.** List all hand-written source files in the requested scope (e.g. `find . -name "*.ts" -not -path "*/node_modules/*"`) with line counts via `wc -l`. Explicitly exclude generated, vendored, and build output (e.g. `@cds-models/`, `dist/`, `gen/`) and state that you excluded them and why.
3. **Read every file end to end.** Do not sample. Use batched `cat` commands to read files in groups. If a file's output is dominated by embedded blobs (base64 fixtures), read head and tail excerpts and note the blob.
4. **Check the safety net.** Run the project's typecheck/lint read-only (e.g. `npx tsc --noEmit -p tsconfig.json`). Also inspect the build config (`tsconfig.json` `include`/`exclude`, `skipLibCheck`, test runner config): which files are compiled, which are only transpiled, which are never touched. Blind spots in build coverage are findings, and claims about compile status must come from this step — never assume.
5. **Score each file from 0 to 10**, where 0 is a best-practice implementation and 10 is pure spaghetti:
   - 0-2: exemplary — clean, typed, documented, follows guidelines.
   - 3-4: solid with minor issues (typos, log noise, small style drift, missing tests).
   - 5-6: meaningful debt (type lies/casts, fragile access patterns, N+1 queries, misleading or dead stubs, heuristic control flow).
   - 7-8: real bugs or structural rot — only assign with concrete evidence (named the broken line, the dead path, or the wrong behavior). Placeholder test suites masquerading as coverage belong here.
   - 9-10: essentially unmaintainable; reserve for genuine spaghetti.
6. **Weigh criteria by impact:** runtime correctness bugs first, then guideline violations the user explicitly mandated (custom Error classes, naming conventions), then duplication/dead code, then style. Do not inflate scores for cosmetic issues.

## Output format

Produce exactly this structure:

1. **Method note** — one short paragraph: what was reviewed, what was excluded and why, how scoring was computed.
2. **Per-file tables**, grouped by area/module. Columns: file, score, one-line rationale. Bold the scores of 6 and above.
3. **Verification evidence** — the typecheck/lint result, the build-coverage blind spots, and which high-severity findings no tooling catches.
4. **Final score** — a small table with the line-weighted average (weight = lines per file), the unweighted mean, and a breakdown per top-level area. Compute these arithmetically (script or calculator), never eyeball them.
5. **Verdict** — 2-4 sentences: overall state, the top issues by severity, and a one-line estimate of what fixing the worst files would move the score to.

Additional rules:

- Cite evidence as `path/to/file.ts:line` or quote the exact broken expression.
- Flag claims you could not verify as "Unverified" (per project guideline requirements).
- Do not write the review to a file in the repo; the review is the response. Scratch files for computation go to the session scratchpad only.
- If the user's guidelines require consulting an MCP server for the technology in scope (e.g. CAP/UI5), do so before making claims about framework behavior, and cite it.
