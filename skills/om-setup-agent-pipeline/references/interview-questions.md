# Setup interview questions

The questions step 3 of `om-setup-agent-pipeline` asks the user (skipped with `--defaults`, which writes the auto-detected config without confirmation):

0. **Greenfield or existing-project setup?** Check for an existing agent instruction file (`AGENTS.md`, `CLAUDE.md`, or equivalent) and any existing process docs (`CONTRIBUTING.md`, an internal `SDLC.md`/`CODE_REVIEW.md` under a different name, etc.) before asking anything else. Two cases:
   - **Greenfield** — no such file. Proceed with the rest of the interview as normal; every doc in question 8 generates fresh from the repo scan.
   - **Existing project** — a file is present. Read it in full now, not just its existence, and tell the user what you found (its scope: does it cover process/SDLC, review rules, contract surfaces, or just architecture?). Then, for each of `SDLC.md`, `CODE_REVIEW.md`, and `BACKWARD_COMPATIBILITY.md` the user opts into generating in question 8, ask which **reconciliation mode** to use rather than assuming "generate independently": **link** (new doc points at the existing file's relevant section), **merge** (generate from the template, seeded with the existing file's stated conventions, flagging any detected disagreement), **offer-diff** (show generated-from-scratch vs. existing-file-implied side by side and let the user pick per section), or **skip** (existing coverage is already sufficient). `AGENTS.md`/`CLAUDE.md` itself is never overwritten regardless of answer — this question is only about the other three docs. Full mechanics: `references/project-docs.md` → Existing-project reconciliation. On an unattended `--defaults` run with an existing file detected, default every doc to **link** (never independent generation) and say so plainly in the final report.
1. Confirm or edit the detected validation commands.
2. Which tracker provider to install (default: `github`). This sets the config's `tracker` field and which descriptor lands in `.ai/trackers/`.
3. Which browser provider to install (default: `agent-browser`; `playwright` is
   the compatibility choice). Explain that the selected descriptor owns
   autonomous CLI/browser provisioning and that repository-native E2E suites
   remain authoritative.
4. Labels: install the full taxonomy above (recommended), keep a subset, or disable labels entirely.
5. QA gate on or off. Recommend on when the repo ships user-facing changes.
6. Where specs live (`paths.specs`, default `.ai/specs`) — confirm or point at an existing design-doc directory.
7. Optional repo-local review checklist path.
8. Project docs to generate (each only when missing): `SDLC.md` (recommended), `AGENTS.md` with the task-routing table (when no agent instruction file exists), `CODE_REVIEW.md`, and `BACKWARD_COMPATIBILITY.md`. On an existing-project setup (question 0), pair each opted-in doc with the reconciliation mode chosen there instead of generating it independently.
