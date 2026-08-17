# Skill install locations: canonical path, per-agent shims, local overrides

Where an installed skill's files physically live, and how a repo-local change to
one skill relates to that install, is a different question from what this
skill's config or the Project docs section covers — this reference is the
one place that answers it, for pointing people at when the layout is unclear.

## The canonical location

A project-level install of this collection puts every skill's directory —
`SKILL.md` plus its `references/` — under `.agents/skills/` at the repository
root. This is the **canonical, cross-agent location**: it is committed (or
otherwise provisioned) once per repository and is what every other location
below ultimately points back to. Codex and Cursor read `.agents/skills/`
natively; no extra step is needed for those two.

A global (per-user, cross-repository) install instead uses the coding agent's
own home-directory skills path (for Claude Code, `~/.claude/skills`; for Codex,
`~/.codex/skills`) — see `references/skill-coverage.md` for how `SKILLS_ROOT`
resolves in that case. The rest of this document is about the project-level
`.agents/skills/` layout, which is the one that needs the shim/override
mechanics explained.

## The `.claude/skills/` shim

Claude Code does not read `.agents/skills/` directly — it only looks under
`.claude/skills/` (project-level) or `~/.claude/skills/` (global). A project
that wants Claude Code to see a project-level install therefore maintains a
**shim**: one symlink per skill directory under `.claude/skills/`, each
pointing back into the matching `.agents/skills/<skill-name>/` directory.
`.agents/skills/` stays the single source of truth — editing
`.agents/skills/<skill-name>/SKILL.md` is what takes effect everywhere,
including through the symlink; `.claude/skills/<skill-name>` is never edited
directly, since it is not a real directory.

The project's own install script (not part of this collection — each
consuming repository provides its own wrapper, commonly invoked as something
like `install-skills.sh` or a matching `yarn`/`npm` script) is what creates and
maintains this shim. Re-run it:

- after adding or removing a skill from the collection (so the symlink set
  matches), or
- with an offline/no-network flag (commonly `--no-external` or equivalent) to
  just resync the `.claude/skills/` shim layer against whatever is already
  present under `.agents/skills/`, without touching the network.

If a skill "isn't found" inside Claude Code on a project that otherwise has it
under `.agents/skills/`, the shim is the first thing to check — a missing or
stale symlink there is indistinguishable from a missing skill until you look.

## The `.codex/skills/` legacy mirror

Codex reads `.agents/skills/` natively in current builds. An older Codex build
that predates that support needs the same kind of mirror as Claude Code: a
`.codex/skills/` directory of symlinks back into `.agents/skills/`. Project
install scripts that support this typically gate it behind an explicit flag
(commonly `--legacy-links`) rather than creating it by default, since current
Codex builds don't need it and an unused mirror is one more thing to keep in
sync.

## Repo-local overrides: `.ai/skills/<name>/SKILL.md`

Shadowing one installed skill's behavior for a single repository is a
**different mechanism** from anything above, and lives in a different
location on purpose: `.ai/skills/<skill-name>/SKILL.md`, not anywhere under
`.agents/skills/` or its shims. Full contract — what a local override file can
and cannot do, how a skill discovers and applies it, the safety clause that
stops it from relaxing base-skill rules — is in `references/agentic-setup.md`
→ Per-skill local overrides; the short version, to tell the two mechanisms
apart:

- **Editing `.agents/skills/<name>/SKILL.md` directly** changes the installed
  copy of the collection skill itself. It works, but the next time the
  collection is upgraded (re-running the project's install tool to pull newer
  skill versions) that edit is overwritten wholesale — there is no merge.
- **Creating `.ai/skills/<name>/SKILL.md` instead** adds a separate,
  repository-owned file that the base skill's own step 0 explicitly checks
  for and layers on top as an **extension** (repo-specific commands, paths,
  labels, templates, extra gate steps — repo specifics win on overlap, but the
  local file can never relax the installed skill's safety rules, expand tool
  or network access, or redirect its outputs). This survives a collection
  upgrade untouched, because the upgrade only ever touches
  `.agents/skills/` (and its shims), never `.ai/skills/`.

Use the second form for anything meant to persist. Reach for the first only
when intentionally forking a skill away from the collection's future updates.

A repo-local override at `.ai/skills/<name>/SKILL.md` also counts as
"available" for the cross-skill coverage check (`references/skill-coverage.md`)
— a skill that name-references another skill the repo has overridden locally,
rather than installed from the collection, is not reported as missing.
