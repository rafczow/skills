# Tracker provider: GitLab

This file is the GitLab implementation of the tracker operations contract (see `TEMPLATE.md` for the contract itself). Skills perform issue/MR state management through **named tracker operations** — `**get-issue**`, `**comment-pr**`, and so on — and this file defines what each operation means for GitLab, using the `glab` CLI.

At runtime: `om-setup-agent-pipeline` copies this file into the repository at `.ai/trackers/gitlab.md`, and the config's `tracker` field (`"gitlab"`) selects it. When a skill says "tracker operation **get-pr**", execute the command documented under that operation heading in the repo's copy. The repo's copy is authoritative: teams extend or override any operation by editing it, and every skill picks the change up on its next run.

Works against both `gitlab.com` and a self-managed instance — set the hostname once in Prerequisites and every command below inherits it.

> **Terminology.** The skills say "PR"; GitLab says "merge request". Every `**…-pr**` operation below acts on a **merge request**. The names are kept as-is because skills call operations by name.

## Prerequisites

- **`glab` CLI ≥ 1.112.0** and `jq` available (the label guards and every normalization filter parse JSON with it). Verify with the **auth-check** operation before a batch run; fail fast when unauthenticated.
- Install without root, where needed:
  ```bash
  VER=1.112.0
  curl -sSL -o /tmp/glab.tar.gz \
    "https://gitlab.com/api/v4/projects/gitlab-org%2Fcli/packages/generic/glab/${VER//./%2E}/glab_${VER//./%2E}_linux_amd64%2Etar%2Egz"
  tar xzf /tmp/glab.tar.gz -C /tmp bin/glab && install -m 0755 /tmp/bin/glab ~/.local/bin/glab
  ```
- **Authentication** is glab's own credential store, so no token ever lives in the repo:
  ```bash
  glab auth login --hostname <your-gitlab-host>     # e.g. gitlab.com, or a self-managed hostname
  ```
  The token needs the **`api`** scope. Label creation additionally needs the **Maintainer** role.
  In CI, set `GITLAB_HOST=<your-gitlab-host>` and `GITLAB_TOKEN=…` instead — those override the stored
  credentials. Never write a token into the working tree: this contract forbids committing resolved
  secrets, and a project's `.ai/` tree has no built-in ignore rule that would catch one.
- **Recommended identity: a project (or group) access token.** A token created with the **Maintainer**
  role directly on the project (Settings → Access tokens) rather than a personal PAT works cleanly as
  the automation identity: GitLab auto-generates a dedicated bot user for it (username pattern
  `project_<project-id>_bot_<random>`), and **current-user** and every claim signal (assignee,
  approvals, notes) work identically against it. This keeps the pipeline's identity project-scoped and
  independent of any one engineer's personal account — each project that adopts this descriptor creates
  and authenticates with its own token, never a shared or hardcoded one.
- All operations accept an optional `{repo}` via **`-R, --repo`** (`GROUP/PROJECT`, or a numeric
  project id). When omitted, `glab` infers it from the current checkout's git remote. Pass
  `-R {group}/{project}` explicitly whenever a skill operates on another project.

### Known limitations and footguns

Several of these are silent-wrong-result traps, not errors — they exit `0` while doing something other
than what was asked, which is exactly the failure mode that strands a pipeline run. Others are
permanent gaps in GitLab's data model versus `github.md`'s assumptions. All were verified live against
a real `glab` binary and a real project before being written down here — no command syntax below is
speculative.

**1. `glab mr merge` sets auto-merge by default.** The flag is `--auto-merge  Set auto-merge.
(true)`. A bare `glab mr merge 509` on a project with a running pipeline does **not** merge — it
schedules a merge-when-pipeline-succeeds and returns success, leaving the MR open. A skill that then
reports "merged" is wrong. **merge-pr** below therefore always passes `--auto-merge=false` for an
immediate merge, and only uses `--auto-merge` when the caller explicitly asked to queue the merge.

**2. `-F` means opposite things on sibling commands.** On `glab api` it is `--field` (a request
parameter). On `glab mr view`, `glab issue view`, `glab label list` and `glab ci get` it is
`--output` (the response format). Writing `glab api … -F json` sends a parameter literally named
`json`; writing `glab mr view 509 -F state=opened` is a parse error. The operations below spell out
`--output json` in full on read commands for exactly this reason — do not "shorten" them to `-F`.

**3. `glab api` has no `--jq` flag at all** (unlike `gh api --jq`). Every `glab api` call below pipes
its output to a separate `jq` invocation — `glab api "…" | jq '…'` — never `glab api "…" --jq '…'`,
which fails outright with `Unknown flag: --jq`. This is the single most consequential porting trap
when adapting `github.md`'s patterns: `--jq` **does** exist, and does work, on the convenience verbs
(`glab mr view`, `glab issue view`, `glab label list`, `glab ci get`, `glab ci list`) — just not on
the plain `glab api` passthrough this descriptor's mutations and most reads are built on. A port that
skips this and trusts the `gh api --jq` muscle memory will silently ship dozens of broken call sites.

**4. `glab mr note --message` is deprecated; its replacement has no file-input flag.** Use
`glab mr note create {prNumber} -m "<text>"`, not the bare `glab mr note {prNumber} --message`
(still works, but prints a deprecation warning to stderr on every call). Neither form accepts a
`--message-file`/`-F <path>`-style flag for the body — the way every multi-line comment in this
descriptor supplies one is shell substitution, `-m "$(cat <path>)"`, not a native file flag.

**5. `glab label create` takes `--name`, not a positional argument.** Differs from
`gh label create <name>`. Also: GitLab wants a **`#`-prefixed** hex color; `gh` does not require the
`#`.

**6. Review verdicts are genuinely absent from GitLab's data model.** No amount of clever API use
produces a native "request changes" object the way GitHub's review API has one. `reviewDecision` is
*derived* (see below) from approval state plus a label — which means the verdict is only as durable as
that label. This is a permanent, unavoidable semantic gap versus `github.md`, not something a future
revision of this file is expected to solve more cleverly.

**7. Positive: GitLab's own gaps versus GitHub.** Three concrete spots where GitLab is easier to work
with than the GitHub equivalent in `github.md`:
- **Image evidence** (`attach-image-evidence`): GitLab's uploads endpoint (`POST /projects/:id/uploads`)
  returns ready-made markdown directly — no evidence branch, no base64 juggling, no shell-arg-limit
  workaround — and it renders inline even on a **private** project, which `github.md` explicitly says
  it cannot do for GitHub.
- **Thread resolution**: GitLab's discussions API reports `resolved` per thread; GitHub REST does not
  expose this (`github.md` notes the gap explicitly). A GitLab-based `list-review-comments` can skip
  already-resolved threads instead of re-litigating them.
- **Atomic label transitions**: `add_labels` + `remove_labels` land in one `PUT`, so the
  mutually-exclusive pipeline-label swap is a single call instead of GitHub's remove-loop-then-add.

## Conventions

- **Issues are `#<iid>`; merge requests are `!<iid>`.** Both are *project-scoped* and numbered
  **independently**, unlike GitHub's single shared `#123` namespace — `#509` and `!509` are different
  objects. Any skill logic that matches `#\d+` and assumes it found a PR/MR reference is wrong on
  GitLab — MR references use `!`. When writing a cross-link in text, use the correct sigil.
- A merge request declares what it resolves with **`Closes #<iid>`** in its **description**; GitLab
  closes the issue on merge. `Fixes` and `Resolves` work identically. To reference without
  auto-closing, write a plain URL.
- MRs open as **drafts** when a skill says so. GitLab models draft state as a **`Draft:` title
  prefix** — `glab mr create --draft` adds it, `glab mr update --ready` removes it. The API's
  `draft` boolean is derived from the title and is read-only.
- Claim/lock signals on an issue or MR: **assignee** set to the automation user, the **`in-progress`
  label**, and a **`🤖`-prefixed note** with a timestamp. All three are set on claim and all three
  are readable back by **get-issue** / **get-pr**. The `ci-monitoring` label is **not** a claim
  signal — it marks work that is finished and reported while its CI-result follow-up is still owed,
  and never makes another skill back off.
- Long, multi-line comment bodies are posted with `--message "$(cat file)"` so formatting survives.
- CI status truth comes from **get-pr-checks**; the *required* set comes from
  **get-required-checks**. When neither is readable, treat every reported job as required.
- **Autofix conflict resolution (`om-auto-review-pr`'s autofix step) uses a merge commit, never a
  rebase.** `git merge origin/{baseRefName}` in the isolated worktree, resolve, commit, then a plain
  `git push origin {localBranch}:{headRefName}` — fast-forward-compatible on top of the existing
  remote branch, so no force-push is ever needed. A rebase would rewrite the branch's existing commit
  and require `--force`, which conflicts with "never force-push unasked". This is provider-agnostic
  guidance (see `om-auto-fix-pr/references/base-merge.md`); it is called out here because GitLab's
  merge-request UI nudges toward rebase in a way that makes it easy to reach for by habit.

### State mapping

Skills expect GitHub's vocabulary. Normalize at this boundary, never in the skill:

| GitLab `state` | Serialize as |
| --- | --- |
| `opened` | `OPEN` |
| `merged` | `MERGED` |
| `closed` | `CLOSED` |
| `locked` | `OPEN` |

### Field mapping

GitLab REST answers snake_case names that differ from the ones skills were written against. **get-pr**
and **get-issue** below apply this rename with `jq` so callers see the same shape `github.md` returns.

| Skill field | GitLab source |
| --- | --- |
| `number` | `iid` |
| `body` | `description` |
| `url` | `web_url` |
| `isDraft` | `draft` |
| `baseRefName` / `headRefName` | `target_branch` / `source_branch` |
| `headRefOid` | `sha` |
| `author.login` / `assignees[].login` | `author.username` / `assignees[].username` |
| `labels[].name` | `labels[]` (plain strings — wrap into objects) |
| `mergeable` / `mergeStateStatus` | `detailed_merge_status` |
| `createdAt` / `mergedAt` / `closedAt` | `created_at` / `merged_at` / `closed_at` |
| `closingIssuesReferences` | `GET /merge_requests/:iid/closes_issues` (separate call) |

### Review verdicts are derived, not stored

This is the **single biggest semantic gap versus `github.md`**. GitHub stores a review verdict per
review; GitLab has no equivalent object. Derive `reviewDecision`:

| Verdict | How to determine it |
| --- | --- |
| `APPROVED` | `GET /merge_requests/:iid/approvals` → `approved_by[]` is non-empty |
| `CHANGES_REQUESTED` | the `changes-requested` pipeline label is present |
| `COMMENTED` | notes exist, but neither of the above |

Because `CHANGES_REQUESTED` is carried by a **label**, `labels.enabled: false` in the config
destroys the ability to express it. A run that disables labels must treat every request-changes
verdict as advisory and say so in its report.

## Label guards

Every label mutation goes through an existence guard, so a missing label degrades to a logged skip
instead of a failure, and `labels.enabled: false` skips label operations entirely.

Mutations go through **`glab api`**, not `glab mr update --label`. The convenience verb resolves
milestones, assignees and reviewers alongside the label write, so an unrelated permission gap on one
of those aborts the whole call — and the caller sees an error about a field it never asked to change
while the label was never applied. `glab api` writes only the fields named. Keep it that way.

Two differences from `github.md` worth knowing, both in GitLab's favour:

- GitLab accepts `add_labels` **and** `remove_labels` in one `PUT`, so the mutually exclusive
  pipeline label is a single atomic call rather than a remove-loop followed by an add.
- GitLab has **separate** issue and MR endpoints. On GitHub both route through `/issues/`, so
  `apply_issue_label` is a one-line delegation there; here it is a genuinely distinct call.

```bash
# $LABELS_ENABLED comes from the config: jq -r '.labels.enabled' .ai/agentic.config.json
# $PIPELINE_LABELS likewise:            jq -r '.labels.pipeline|join(" ")' .ai/agentic.config.json
# $REPO_FLAG is "-R {group}/{project}" when a skill targets another project, else empty.

label_exists() {
  glab api --paginate "projects/:fullpath/labels" $REPO_FLAG \
    | jq -r '.[].name' | grep -Fxq "$1"
}

# $1 = label, $2 = MR iid
apply_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  if label_exists "$1"; then
    glab api -X PUT "projects/:fullpath/merge_requests/$2" $REPO_FLAG \
      -f "add_labels=$1" --silent
  else
    echo "Skipping label '$1' (not defined in this project). Create it with: glab label create --name '$1'"
  fi
}

# $1 = label, $2 = issue iid. A distinct endpoint on GitLab — not a delegation.
apply_issue_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  if label_exists "$1"; then
    glab api -X PUT "projects/:fullpath/issues/$2" $REPO_FLAG -f "add_labels=$1" --silent
  else
    echo "Skipping label '$1' (not defined in this project). Create it with: glab label create --name '$1'"
  fi
}

# Removal needs no existence check: removing an absent label is a no-op, not an error.
remove_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  glab api -X PUT "projects/:fullpath/merge_requests/$2" $REPO_FLAG \
    -f "remove_labels=$1" --silent >/dev/null 2>&1 || true
}
remove_issue_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  glab api -X PUT "projects/:fullpath/issues/$2" $REPO_FLAG \
    -f "remove_labels=$1" --silent >/dev/null 2>&1 || true
}

# Pipeline labels are mutually exclusive. $1 = MR iid, $2 = label.
# One call: add the wanted label, drop every sibling.
set_pipeline_label() {
  if [ "$LABELS_ENABLED" != "true" ]; then return 0; fi
  if ! label_exists "$2"; then
    echo "Skipping pipeline label '$2' (not defined in this project)."
    return 0
  fi
  others=$(for l in $PIPELINE_LABELS; do [ "$l" = "$2" ] || printf '%s,' "$l"; done)
  glab api -X PUT "projects/:fullpath/merge_requests/$1" $REPO_FLAG \
    -f "add_labels=$2" -f "remove_labels=${others%,}" --silent
}
```

`:fullpath` is a `glab api` placeholder that expands to the current project's full path; with
`-R group/project` it expands to that project instead, so the existence check and the mutation always
address the same place.

Read the labels back (**get-pr** → `.labels`) whenever label state gates a later decision — a
mutation the guard logged as skipped is a normal outcome, and a run that assumes it landed branches
wrongly. This matters more here than on GitHub, because `CHANGES_REQUESTED` *is* a label.

## Operations

### Identity and repository

#### auth-check
Verify the CLI is present, new enough, and authenticated **against the configured instance**. →
non-zero when unauthenticated.

Pin the hostname. A bare `glab auth status` also reports on `gitlab.com`, which glab configures by
default and which may not be the target on a self-managed instance; its failure is noise, and in a
mixed setup a healthy `gitlab.com` entry can make an unauthenticated target instance look fine at a
glance.
```bash
glab auth status --hostname "$GITLAB_HOST" || exit 1

GLAB_VERSION=$(glab --version | sed -n '1s/.*glab \([0-9][0-9.]*\).*/\1/p')
MIN_GLAB_VERSION=1.112.0
if [ "$(printf '%s\n%s\n' "$MIN_GLAB_VERSION" "$GLAB_VERSION" | sort -V | head -n1)" != "$MIN_GLAB_VERSION" ]; then
  echo "WARNING: glab $GLAB_VERSION predates $MIN_GLAB_VERSION — flag names below may differ. Upgrade per Prerequisites."
fi
```

#### current-user
→ the automation user's username.
```bash
CURRENT_USER=$(glab api user | jq -r '.username')
```

#### repo-info
→ the project's full path, numeric id, and default branch.
```bash
glab api "projects/:fullpath" | jq '{fullPath:.path_with_namespace, id:.id, defaultBranch:.default_branch}'
REPO=$(glab api "projects/:fullpath" | jq -r '.path_with_namespace')
```

#### default-branch
→ the project's default branch (used when the config's `baseBranch` is `"auto"`).
```bash
BASE_BRANCH=$(glab api "projects/:fullpath" 2>/dev/null | jq -r '.default_branch // empty')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH=$(git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's@^refs/remotes/origin/@@')
[ -z "$BASE_BRANCH" ] && BASE_BRANCH="main"
```

### Issues

#### get-issue
`{issueId}` → issue data, normalized to the field names skills expect.
```bash
glab api "projects/:fullpath/issues/{issueId}" | jq '{
  number: .iid, title: .title, url: .web_url, body: .description,
  state: (if .state == "opened" then "OPEN" else "CLOSED" end),
  author: {login: .author.username},
  labels: [.labels[] | {name: .}],
  assignees: [(.assignees // [])[] | {login: .username}],
  createdAt: .created_at, closedAt: .closed_at
}'
```
Comments come from **list-issue-comments** (GitLab does not inline them on the issue object).

#### search-issues
Text query + state → matching issues. `in=title,description` scopes the search.
```bash
glab api --paginate "projects/:fullpath/issues?state=opened&search=<query>&in=title,description" \
  | jq '.[] | {number: .iid, title: .title, url: .web_url}'
```

#### create-issue
Title, body, assignee, labels → created issue URL.
```bash
glab issue create --title "<title>" --description "<body>" \
  --assignee <username> --label "<comma,separated>" --yes
```
Scripted form, when the convenience verb's interactive prompts get in the way:
```bash
glab api -X POST "projects/:fullpath/issues" \
  -f "title=<title>" -f "description=<body>" -f "labels=<comma,separated>" | jq -r '.web_url'
```

#### close-issue
`{issueId}`, closing comment. Comment first so the cross-link is visible on the closed issue.
```bash
glab issue note {issueId} --message "<comment>"
glab api -X PUT "projects/:fullpath/issues/{issueId}" -f "state_event=close" --silent
```
GitLab has no close *reason* ("completed" vs "not planned"); state that in the closing comment instead.

#### comment-issue
`{issueId}`, body.
```bash
glab issue note {issueId} --message "$(cat <path>)"
```

#### update-issue
`{issueId}`, new title and/or body. Edits the issue's own fields only — labels and assignees have
their own operations. Pass only what changed.
```bash
glab api -X PUT "projects/:fullpath/issues/{issueId}" -f "title=<title>" --silent
glab api -X PUT "projects/:fullpath/issues/{issueId}" -f "description=$(cat <path>)" --silent
```

#### assign-issue / unassign-issue
GitLab assigns by numeric **user id**, not username, so resolve first. `assignee_ids=0` clears.
```bash
UID_=$(glab api "users?username=<username>" | jq -r '.[0].id')
glab api -X PUT "projects/:fullpath/issues/{issueId}" -f "assignee_ids=$UID_" --silent
glab api -X PUT "projects/:fullpath/issues/{issueId}" -f "assignee_ids=0"     --silent
```

#### label-issue / unlabel-issue
Always through the guards: `apply_issue_label "<label>" {issueId}` / `remove_issue_label "<label>" {issueId}`.

#### get-issue-comment
Note id → body, author, URL. GitLab notes have no own web URL; build the anchor.
```bash
glab api "projects/:fullpath/issues/{issueId}/notes/{noteId}" \
  | jq '{body, user: .author.username, id}'
```

#### list-issue-comments
`{issueId}` → conversation notes. `system` notes are GitLab's activity log (label changes, etc.) —
filter them out unless the caller wants the audit trail.
```bash
glab api --paginate "projects/:fullpath/issues/{issueId}/notes" \
  | jq '.[] | select(.system | not) | {id, user: .author.username, body, createdAt: .created_at}'
```

#### update-comment
`{noteId}`, new body → rewrite a note in place. This is how marker-idempotent comments (label
rationale, verification, claim take-overs) avoid duplicating on re-runs: find the `🤖 …` marker via
**list-issue-comments** / **list-review-comments**, then update that note.
```bash
glab api -X PUT "projects/:fullpath/issues/{issueId}/notes/{noteId}"          -f "body=$(cat <path>)" --silent
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}/notes/{noteId}" -f "body=$(cat <path>)" --silent
```
A note can only be edited by its author; a take-over by a different automation user must post a
replacement that says it supersedes the earlier one.

### Pull requests (merge requests)

#### get-pr
`{prNumber}` → MR data, normalized to the field names skills expect (see Field mapping).
```bash
glab api "projects/:fullpath/merge_requests/{prNumber}" | jq '{
  number: .iid, title: .title, url: .web_url, body: .description,
  state: (if .state == "opened" then "OPEN" elif .state == "merged" then "MERGED" elif .state == "closed" then "CLOSED" else "OPEN" end),
  author: {login: .author.username},
  isDraft: .draft,
  baseRefName: .target_branch, headRefName: .source_branch, headRefOid: .sha,
  mergeable: (.detailed_merge_status == "mergeable"),
  mergeStateStatus: .detailed_merge_status,
  labels: [.labels[] | {name: .}],
  assignees: [(.assignees // [])[] | {login: .username}],
  createdAt: .created_at, mergedAt: .merged_at, closedAt: .closed_at,
  changedFiles: .changes_count
}'
```
Three fields need their own calls, by design — request them only when the caller names them:
```bash
# reviewDecision (derived — see "Review verdicts are derived, not stored")
glab api "projects/:fullpath/merge_requests/{prNumber}/approvals" | jq '.approved_by | length'
# closingIssuesReferences
glab api "projects/:fullpath/merge_requests/{prNumber}/closes_issues" | jq '[.[] | {number: .iid, url: .web_url}]'
# additions/deletions
glab api "projects/:fullpath/merge_requests/{prNumber}/changes" | jq '.changes | length'
```

#### list-prs
State filters + limit → MRs. GitLab states are `opened` / `merged` / `closed` / `locked`.
```bash
glab api --paginate "projects/:fullpath/merge_requests?state=opened&order_by=updated_at" \
  | jq '.[] | {number: .iid, title: .title, url: .web_url, author: .author.username,
               labels: .labels, draft: .draft, headRefName: .source_branch,
               baseRefName: .target_branch, updatedAt: .updated_at}'

# merged since a date (om-auto-update-changelog, om-close-fixed-issues)
glab api --paginate "projects/:fullpath/merge_requests?state=merged&updated_after=${SINCE_DATE}" \
  | jq '.[] | {number: .iid, title: .title, url: .web_url, body: .description,
               author: .author.username, mergedAt: .merged_at, mergeCommit: .merge_commit_sha,
               baseRefName: .target_branch, headRefName: .source_branch, labels: .labels}'

# closed without merging
glab api --paginate "projects/:fullpath/merge_requests?state=closed&updated_after=${SINCE_DATE}" \
  | jq '.[] | select(.merged_at == null) | {number: .iid, title: .title, url: .web_url, closedAt: .closed_at}'
```

#### search-prs
Free-text query (for example an issue reference) + state → matching MRs.
```bash
glab api --paginate "projects/:fullpath/merge_requests?state=opened&search=<query>&in=title,description" \
  | jq '.[] | {number: .iid, title: .title, url: .web_url, state: .state}'
```
To find the MR that closes a given issue, prefer the exact reverse lookup:
```bash
glab api "projects/:fullpath/issues/{issueId}/related_merge_requests" | jq '.[] | {number: .iid, url: .web_url}'
```

#### create-pr
Base branch, draft flag, title, body → MR URL + number.
```bash
glab mr create --target-branch "$BASE_BRANCH" --draft \
  --title "<title>" --description "$(cat <path>)" --yes
PR_URL=$(glab mr view --output json --jq '.web_url')
PR_NUMBER=$(glab mr view --output json --jq '.iid')
```
`glab mr create` reads the current branch as the source and pushes it if needed. Add
`--remove-source-branch` only when the caller asked for it.

#### update-pr
`{prNumber}`, new title and/or body → rewritten in place (not a comment). Pass only what changed.
```bash
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}" -f "title=<title>" --silent
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}" -f "description=$(cat <path>)" --silent
```

#### comment-pr
`{prNumber}`, body. `glab mr note --message` is deprecated in favor of `glab mr note create`.
```bash
glab mr note create {prNumber} -m "$(cat <path>)"
```

#### attach-image-evidence
`{prNumber}`, a comment body, a `{slug}`, and local image paths → one comment with the images
embedded **inline**, returning the comment URL.

GitLab has a first-class uploads endpoint that returns ready-made markdown, so this is
**substantially simpler than the GitHub path** — no evidence branch, no base64, no shell-arg-limit
workaround — and it renders inline on a **private** project, which `github.md` explicitly cannot do.
Nothing is pushed to any branch.

```bash
BODY=$(cat <body-file>)
for img in <image-paths>; do
  MD=$(glab api -X POST "projects/:fullpath/uploads" --form "file=@${img}" | jq -r '.markdown')
  BODY="${BODY}"$'\n'"${MD}"
done
glab mr note create {prNumber} -m "$BODY"
glab api "projects/:fullpath/merge_requests/{prNumber}" | jq -r '.web_url'
```
Uploads are scoped to the project and inherit its visibility, so evidence on a private project stays
private. If the upload is rejected (attachment size limit, or a token without `api` scope), post the
comment with the local artifact paths and say inline rendering was unavailable — do not fail the caller.

#### assign-pr / unassign-pr
By numeric user id, as with issues.
```bash
UID_=$(glab api "users?username=<username>" | jq -r '.[0].id')
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}" -f "assignee_ids=$UID_" --silent
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}" -f "assignee_ids=0"     --silent
```

#### label-pr / unlabel-pr
Always through the guards: `apply_label "<label>" {prNumber}`, or
`set_pipeline_label {prNumber} "<label>"` for the mutually exclusive pipeline group; direct removal
`remove_label "<label>" {prNumber}`.

#### get-pr-diff
`{prNumber}` → full diff, or the changed-file list.
```bash
glab mr diff {prNumber}
glab api "projects/:fullpath/merge_requests/{prNumber}/changes" | jq -r '.changes[].new_path'
```

#### get-pr-files
`{prNumber}` → changed files with per-file status, mapped to GitHub's `added`/`removed`/`modified`.
```bash
glab api "projects/:fullpath/merge_requests/{prNumber}/changes" | jq '
  .changes[] | {
    path: .new_path,
    status: (if .new_file then "added" elif .deleted_file then "removed"
             elif .renamed_file then "renamed" else "modified" end)
  }'
```

#### checkout-pr
`{prNumber}` → the MR's head available locally (works for fork MRs, where the source branch is not
on `origin`).
```bash
glab mr checkout {prNumber}
```

#### review-pr
`{prNumber}`, verdict, body. **GitLab has no native "request changes" verdict** — see "Review
verdicts are derived, not stored". The approve/unapprove endpoints are on GitLab's free tier, so both
paths below work on any plan; only approval *rules* are a paid-tier feature.

```bash
# approve
glab mr note create {prNumber} -m "$(cat <path>)"
glab mr approve {prNumber}
set_pipeline_label {prNumber} "review"        # or merge-queue, per the caller

# request changes: revoke any standing approval, state the findings, carry the verdict on the label
glab mr revoke {prNumber} 2>/dev/null || true   # no-op when this user has not approved
glab mr note create {prNumber} -m "$(cat <path>)"
set_pipeline_label {prNumber} "changes-requested"
```
GitLab **permits self-approval** by default (unlike GitHub, which rejects it) — a project may forbid
it via a paid-tier approval rule, in which case `glab mr approve` fails with a permissions error.
Surface that rather than working around it. Because the request-changes verdict rides on a label, a
skill running with `labels.enabled: false` cannot express it and must say so in its report.

#### merge-pr
`{prNumber}`; squash is the default strategy. **`--auto-merge=false` is mandatory for an immediate
merge** — see Known limitations #1. When `get-required-checks` reported `resolveDiscussions: true`, run
**resolve-discussions** first — a merge attempt otherwise fails on any resolvable thread still open,
including ones this descriptor's own comments opened (see that operation).
```bash
glab mr merge {prNumber} --squash --auto-merge=false --yes
glab mr merge {prNumber} --squash --auto-merge=false --yes --remove-source-branch  # only when asked
glab mr merge {prNumber} --squash --auto-merge --yes                               # queue behind CI, deliberately
```
Read the result back — a queued merge leaves `state: opened`:
```bash
glab api "projects/:fullpath/merge_requests/{prNumber}" | jq -r '.state'
```

#### mark-pr-ready
Promote a draft MR (strips the `Draft:` title prefix).
```bash
glab mr update {prNumber} --ready
```

#### get-pr-checks
`{prNumber}` → the head pipeline's jobs with name, status, and link.
```bash
glab ci get --merge-request {prNumber} --with-job-details --output json \
  --jq '.jobs[]? | {name: .name, state: .status, link: .web_url}'
```
Raw form, and the way to reach older pipelines for the same MR:
```bash
glab api "projects/:fullpath/merge_requests/{prNumber}/pipelines" | jq '.[] | {id, status, sha, web_url}'
glab api "projects/:fullpath/pipelines/{pipelineId}/jobs" | jq '.[] | {name, status, web_url}'
```
Do not assume that only heavyweight/release jobs run on ordinary MRs and skip checking for others —
verify what actually runs per-MR in the target project rather than inheriting an assumption from
elsewhere. An MR with **no** pipeline returns nothing, which is a valid state (a brand-new MR before
CI has started, or one from a fork without runner access) — treat "no pipeline" as "nothing to wait
for yet", never as a failure.

#### get-required-checks
GitLab has **no per-check required flag** like GitHub branch protection. The nearest equivalents are
a project-level merge setting and the protected-branch rule. When neither is readable, treat every
job reported by **get-pr-checks** as required.
```bash
glab api "projects/:fullpath" | jq '{pipelineMustSucceed: .only_allow_merge_if_pipeline_succeeds,
                                     resolveDiscussions: .only_allow_merge_if_all_discussions_are_resolved}'
glab api "projects/:fullpath/protected_branches/{baseRefName}" 2>/dev/null \
  | jq '{name, mergeLevels: .merge_access_levels, pushLevels: .push_access_levels}'
```
`pipelineMustSucceed: true` means *the whole pipeline* must be green — the required set is then every
job in it. `false` means no job is strictly required, and merge readiness rests on review state alone.

#### resolve-discussions (GitLab-specific — no `TEMPLATE.md` equivalent)
When `resolveDiscussions` (above) is `true`, `merge-pr` fails until every **resolvable** discussion
thread on the MR reads `resolved: true` — and this bites automation specifically, not just human
review threads: GitLab marks some of *this descriptor's own* notes as resolvable by default (observed
live: a claim comment, a label-rationale comment, and a completion comment all came back
`resolvable: true, resolved: false`), so a bot that only ever posts and never resolves can permanently
block its own merge. Check first with **list-review-comments**' discussions call (`.[] | {id,
resolvable: .notes[0].resolvable, resolved: .notes[0].resolved}`), then resolve every
`resolvable: true, resolved: false` thread the run's own comments opened — never a human reviewer's
thread, which stays theirs to resolve:
```bash
glab api -X PUT "projects/:fullpath/merge_requests/{prNumber}/discussions/{discussionId}" \
  -f "resolved=true" | jq '{id, resolved: .notes[0].resolved}'
```
Run this as part of **merge-pr**'s preflight (or immediately before it in the autofix loop), not as
a one-off — any resolvable note this descriptor posts (claim, label-rationale, completion, CI-result)
can independently block the merge gate the same way. See TEMPLATE.md's "resolvable discussions" callout
for the general question every tracker author should ask about this pattern — this operation is
GitLab's concrete answer to it.

#### get-pr-comment / get-review-comment
GitLab distinguishes a plain note from one anchored to the diff by whether it carries a `position`.
```bash
glab api "projects/:fullpath/merge_requests/{prNumber}/notes/{noteId}" \
  | jq '{body, user: .author.username, inline: (.position != null), createdAt: .created_at}'
```

#### list-review-comments
`{prNumber}` → the notes anchored to a file and line (conversation notes come from
**list-issue-comments**' MR equivalent, below).
```bash
# inline diff comments, with thread resolution state
glab api --paginate "projects/:fullpath/merge_requests/{prNumber}/discussions" | jq '
  .[] | select(.notes[0].position != null)
      | {discussionId: .id,
         resolved: (.notes[0].resolved // false),
         comments: [.notes[] | {id, user: .author.username, path: .position.new_path,
                                line: (.position.new_line // .position.old_line), body}]}'

# MR conversation notes (the list-issue-comments equivalent for an MR)
glab api --paginate "projects/:fullpath/merge_requests/{prNumber}/notes" \
  | jq '.[] | select(.system | not) | select(.position == null) | {id, user: .author.username, body}'
```
Unlike GitHub REST — which `github.md` notes cannot expose thread resolution — GitLab **does** report
`resolved`, so a skill can skip already-resolved threads instead of re-litigating them. A discussion's
`notes` array is the thread in order, so no `reply_to` reconstruction is needed.

### CI runs

CI status for an *MR* comes from **get-pr-checks** / **get-required-checks**. These address pipelines
directly — for a bare branch, or when a diagnosis needs the actual logs.

#### list-runs
Branch (or SHA) → recent pipelines.
```bash
glab ci list --ref {branch} --per-page 20 --output json \
  --jq '.[] | {id, status, sha, ref, url: .web_url, createdAt: .created_at}'
```

#### get-run
Pipeline id → status plus per-job breakdown.
```bash
glab ci get --pipeline-id {runId} --with-job-details --output json
```

#### get-run-failed-logs
Pipeline id → logs of the failed jobs only. The primary CI-failure diagnosis input. `glab ci trace`
takes one job, so enumerate the failed jobs first.
```bash
for JOB in $(glab api "projects/:fullpath/pipelines/{runId}/jobs?scope[]=failed" | jq -r '.[].id'); do
  echo "===== job $JOB ====="
  glab api "projects/:fullpath/jobs/${JOB}/trace" | tail -200
done
```

#### rerun-failed
Pipeline id → retry only the failed jobs. `POST /pipelines/:id/retry` is exactly this — it does not
re-run jobs that already succeeded. Use it to disambiguate a flake before changing code.
```bash
glab api -X POST "projects/:fullpath/pipelines/{runId}/retry" | jq '{id, status}'
```
Note `glab ci retry <job>` retries a **single job**, not the pipeline — not what this operation means.

#### watch-run
Block until the pipeline for a branch finishes.
```bash
glab ci status --branch {branch} --wait
```
Where the branch is not checked out, poll **get-run** until `status` leaves
`created|waiting_for_resource|preparing|pending|running`, honouring `ci.maxWaitMinutes` from the config.

### Labels

#### list-labels
→ every label name in the project. Paginated: the API default is 20 per page, so a fixed
`glab label list` truncates a taxonomy this size.
```bash
glab api --paginate "projects/:fullpath/labels" | jq -r '.[].name'
```

#### create-label
Name, color, description. GitLab wants a **`#`-prefixed** hex color, and `glab label create` takes
the name as a **`--name` flag, not a positional argument** (unlike `gh label create <name>`). Never
delete, rename, or recolor an existing label.
```bash
glab label create --name <name> --color "#<hex>" --description "<description>"
```

#### ensure-label-taxonomy
Create every label in the config's taxonomy that does not exist yet; skip the ones **list-labels**
already reports. Needs the **Maintainer** role.

```bash
create_if_missing() {   # $1 name, $2 hex (no #), $3 description
  if glab api --paginate "projects/:fullpath/labels" | jq -r '.[].name' | grep -Fxq "$1"; then
    echo "exists: $1"
  else
    glab label create --name "$1" --color "#$2" --description "$3"
  fi
}

# pipeline (mutually exclusive)
create_if_missing review            0366d6 "Ready for code review"
create_if_missing changes-requested b60205 "Reviewer requested changes"
create_if_missing qa                fbca04 "Manual QA in progress"
create_if_missing qa-failed         b60205 "Manual QA failed"
create_if_missing merge-queue       0e8a16 "Approved, ready to merge"
create_if_missing blocked           b60205 "Blocked by a dependency"
create_if_missing do-not-merge      b60205 "Hard merge block"
# category
create_if_missing bug               d73a4a "Bug fix"
create_if_missing feature           a2eeef "New capability"
create_if_missing refactor          cfd3d7 "No behavior change"
create_if_missing security          b60205 "Security-relevant change"
create_if_missing dependencies      0366d6 "Dependency update"
create_if_missing documentation     0075ca "Docs only"
# meta
create_if_missing needs-qa          fbca04 "Requires manual QA before merge"
create_if_missing skip-qa           0e8a16 "Low risk, QA not required"
create_if_missing qa-approved       0e8a16 "Manual QA passed"
create_if_missing qa-self-verified  c5def5 "Self-QA exception used"
create_if_missing in-progress       c5def5 "An automated skill is working on this"
create_if_missing ci-monitoring     d4c5f9 "Work complete and reported; agent is watching CI results"
create_if_missing do-not-close      c5def5 "Humans only: never auto-close this issue"
# priority
create_if_missing priority-low      e4e669 "Cosmetic or follow-up work"
create_if_missing priority-medium   fbca04 "Ordinary bug or feature"
create_if_missing priority-high     d93f0b "Release-blocking"
create_if_missing priority-extreme  b60205 "Outage or security incident"
# risk
create_if_missing risk-low          0e8a16 "Isolated, low blast radius"
create_if_missing risk-medium       fbca04 "Ordinary change with tests"
create_if_missing risk-high         b60205 "Wide blast radius, review deeply"
```
That is the 26 labels of the config's taxonomy plus `do-not-close`, which is not in the `meta` group
in a default `.ai/agentic.config.json` but is required by **om-close-fixed-issues**.

> **A shared, live project.** Labels are visible to the whole team and this operation is
> additive-only. Run it deliberately, once, rather than as a side effect of another skill.

## Verification discipline

Every command in this file was checked against the real `glab` binary (`--help` output) and a live
GitLab project before being written down, including reversing wrong assumptions mid-session (that
`--jq` worked on `glab api`; that ordinary MRs might have no pipeline). Treat that as a requirement
for any future edit to this file, not just how it happened to be written the first time: no command
syntax here should ship un-run against a real `glab` install.
