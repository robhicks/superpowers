# Publishing Specs & Plans

Shared by `brainstorming` (the spec) and `writing-plans` (the plan). It decides **where a finished artifact goes** so specs and plans stop landing as committed Markdown files in the target repo (often on `main`/`master`).

The tracker — a JIRA ticket or a GitHub issue — is the **human-facing record of truth**. When a tracker is used, a throwaway **local working copy** is also written into the git scratch dir so reviewer subagents and the plan executor still have a real file path to read. The legacy "commit a file into the repo" behavior is the fallback only.

## Routing — evaluate in order

Pick the canonical home for the artifact:

1. **JIRA** — when a ticket key is discoverable AND `$JIRA_TOKEN` is set.
   - **Key discovery:** `git rev-parse --abbrev-ref HEAD`, then match `[A-Z][A-Z0-9]+-[0-9]+` (matches `feature/ABC-123-...` and `ABC-123-...`). If the branch has no match, use a key the user named in conversation.
   - **Key found but `$JIRA_TOKEN` unset → STOP.** Do not silently fall back — the user clearly intended JIRA. Say:
     > "Found JIRA key `<KEY>` but `$JIRA_TOKEN` isn't set. Create a personal access token in JIRA (Profile → Personal Access Tokens), `export JIRA_TOKEN=…`, and I'll continue."
2. **GitHub issue** — else when `git remote get-url origin` resolves to a `github.com` host and `gh auth status` succeeds.
3. **Local file (legacy fallback)** — otherwise. Write to `docs/superpowers/specs/YYYY-MM-DD-<topic>-design.md` or `docs/superpowers/plans/YYYY-MM-DD-<feature>.md` and commit it, exactly as before. Skip the tracker and scratch steps below — the committed file *is* the working copy.

### Local working copy (tracker homes only)

When the canonical home is JIRA or GitHub, also write the artifact to the git scratch dir before publishing:

```bash
SCRATCH="$(git rev-parse --git-path superpowers)"   # e.g. .git/superpowers — never tracked, never committed
mkdir -p "$SCRATCH/specs" "$SCRATCH/plans"
# write spec → "$SCRATCH/specs/YYYY-MM-DD-<topic>-design.md"
# write plan → "$SCRATCH/plans/YYYY-MM-DD-<feature>.md"
```

Hand **this path** to reviewer subagents (`[SPEC_FILE_PATH]` / `[PLAN_FILE_PATH]`) and to `executing-plans` / `subagent-driven-development`. It lives inside `.git/`, so it never appears in the working tree, never gets committed, and never lands on `main`/`master`.

## Publishing commands

### JIRA (Server/Data Center, REST API v2)

Base URL: `${JIRA_BASE_URL:-https://jira.familysearch.org}`. Post the artifact as a **comment** on the ticket — the spec is one comment, the plan is a second comment. This never touches the ticket's description.

```bash
BASE="${JIRA_BASE_URL:-https://jira.familysearch.org}"
jq -Rs '{body: .}' < "$SCRATCH/specs/<file>.md" \
  | curl -fsS -X POST "$BASE/rest/api/2/issue/$KEY/comment" \
      -H "Authorization: Bearer $JIRA_TOKEN" \
      -H "Content-Type: application/json" --data @-
```

- `jq -Rs '{body: .}'` JSON-encodes the entire Markdown file as the comment body — never hand-build the JSON. If `jq` is unavailable, fall back to the GitHub or local route rather than risk a malformed request.
- `curl -f` makes HTTP errors (401/403/404) fail the command instead of silently posting nothing — check the exit code and surface failures to the user.
- Surface the ticket URL `$BASE/browse/$KEY` to the user.
- **Known limitation:** this JIRA renders *wiki markup*, not Markdown, so `##` headers and ```` ``` ```` fences appear as raw text. The content stays fully readable; richer rendering (Markdown→wiki conversion) is a future enhancement, not a blocker.

### GitHub issue (`gh`, already authenticated)

The **spec creates the issue**; the **plan is a comment** on that same issue.

```bash
# Spec → new issue. Capture and remember the URL.
ISSUE_URL=$(gh issue create \
  --title "<Feature> — spec & plan" \
  --body-file "$SCRATCH/specs/<file>.md")

# Plan → comment on the same issue.
gh issue comment "$ISSUE_URL" --body-file "$SCRATCH/plans/<file>.md"
```

- After creating the issue in `brainstorming`, **state `$ISSUE_URL` explicitly in the handoff** so the same agent reuses it when `writing-plans` posts the plan comment.
- **Fast Path (plan only, no spec):** there is no issue yet, so the plan *creates* the issue — `gh issue create --title "<Feature> — plan" --body-file "$SCRATCH/plans/<file>.md"`.
- GitHub issue bodies/comments render Markdown natively, so no conversion is needed.

## What to tell the user

After publishing, report both the human record and the working copy:

- **JIRA:** "Spec/plan posted to `<BASE>/browse/<KEY>`. Local working copy: `<scratch path>`."
- **GitHub:** "Spec/plan posted to `<ISSUE_URL>`. Local working copy: `<scratch path>`."
- **Local fallback:** "Spec/plan written and committed to `<repo path>`."

## Configuration

- `JIRA_TOKEN` — Bearer personal access token for the JIRA Server/DC instance (required for the JIRA route).
- `JIRA_BASE_URL` — JIRA instance URL (default `https://jira.familysearch.org`).
