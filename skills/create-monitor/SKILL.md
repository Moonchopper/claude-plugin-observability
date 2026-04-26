---
name: create-monitor
description: Use when a user wants to create or provision a Datadog monitor — alerting on metrics, logs, or events. Trigger phrases include "create a monitor", "set up a Datadog alert", "alert on", "page when".
allowed-tools: Bash, Read, Glob, Grep, Edit, Write, Skill
---

# Create Datadog Monitor

## Goal
Help the user open a compliant Terraform PR to `Moonchopper/datadog-operations`
that provisions a Datadog monitor. End state: a PR-handoff payload has been
written via the `pr-handoff` skill with the drafted file and PR body.

## Execution-model boundary
You author a PR. You do NOT call the Datadog API directly. You do NOT run
`terraform apply`. Human review and the existing CI/CD pipeline handle
deployment. If the user asks you to bypass review or apply directly, decline
and explain this boundary.

## Step 1 — Locate the functional repo

Run the detection ladder, in order. Stop at the first hit.

**1a. CWD match.** Run:
```bash
git rev-parse --show-toplevel 2>/dev/null
git config --get remote.origin.url 2>/dev/null
```
If the toplevel exists AND the origin URL matches `*datadog-operations*`,
set `REPO_ROOT` to the toplevel and skip to Step 2.

**1b. Conventional paths.** For each of these paths, check if it is a valid
clone of the target (i.e. `.git/` exists and origin matches):
- `~/src/Moonchopper/datadog-operations`
- `~/code/Moonchopper/datadog-operations`
- `~/git/Moonchopper/datadog-operations`
- `~/work/Moonchopper/datadog-operations`

First match wins.

**1c. Ask the user.** If nothing was found, ask: "I couldn't find a clone
of `Moonchopper/datadog-operations` locally. Where is it, or shall I clone
it to `~/src/Moonchopper/datadog-operations`?" Wait for the answer before
proceeding.

**Auth preflight.** Before attempting any `gh api` fallback, run:
```bash
gh auth status
```
If it fails, halt and direct the user: "Please run `gh auth login` first."

## Step 2 — Freshness check

On a local hit, run:
```bash
cd "$REPO_ROOT"
git fetch --quiet origin
git rev-list --left-right --count HEAD...origin/main 2>/dev/null
```
Interpret the output: `<ahead>\t<behind>`.

- If behind > 0: show the user "Your clone is <behind> commits behind
  origin/main. Pull, proceed with local, or abort?" Wait for answer.
- If diverged (ahead > 0 AND behind > 0): warn and let the user decide.

## Step 3 — Load golden-path content

Read, in this order:
- `$REPO_ROOT/agent/golden-paths/create-monitor/README.md`
- `$REPO_ROOT/agent/golden-paths/create-monitor/steps.md`
- `$REPO_ROOT/agent/golden-paths/create-monitor/best-practices.md`
- `$REPO_ROOT/agent/golden-paths/create-monitor/example.tf`

Parse `best-practices.md` for the list of referenced practice files. For
each reference, Read the corresponding file under
`$REPO_ROOT/agent/best-practices/`. The expected references are:
- `monitor-notification-target.md`
- `monitor-evaluation-window-floor.md`

## Step 4 — Walk the user through the steps

Follow the numbered steps in `steps.md` verbatim. At Step 1 of `steps.md`
(confirm inputs), collect from the user:

- `name` — monitor name (human-readable)
- `team` — team slug (e.g. `foobar`)
- `env` — environment (`prod`, `stage`, `dev`)
- `query` — the metric/log/service expression to evaluate
- `window` — evaluation window (e.g. `5m`, `15m`)
- `notify_target` — e.g. `@slack-platform-team`, `@pagerduty-platform-oncall`

If the user's initial prompt already provides these, confirm them rather
than re-asking. Suggest defaults where reasonable but let the user override.

## Step 5 — Pre-draft best-practice check

For each referenced practice file, apply its `## How to check a draft`
section to the user's confirmed inputs (not yet drafted files). Check the
"skip if" preconditions first; skip practices that don't apply.

For each violation:
1. Summarize: practice name, what is wrong, suggested change.
2. Offer two options: **Accept** (apply the change) or **Reject** (require
   non-empty free-form rationale).
3. The agent records. The agent does not argue. If the rationale is empty,
   re-ask once, then accept any non-blank text.

Collect all accepted rationales into `overrides[]`.

## Step 6 — Draft the change

Create `$REPO_ROOT/terraform/monitors/$team.tf` using `example.tf` as
the template. Substitute `<name>`, `<team>`, `<env>`, `<query>`, `<window>`,
`<notify_target>` with the confirmed values.

Run:
```bash
cd "$REPO_ROOT"
terraform fmt terraform/monitors/$team.tf
```

Run (from `$REPO_ROOT/terraform`):
```bash
terraform validate
```

Halt on any validation failure and surface the error to the user.

## Step 7 — Post-draft best-practice check

Re-run each referenced practice's "How to check" against the drafted file
contents (not the inputs). Handle violations exactly as in Step 5.

## Step 8 — Show diff and await approval

Run:
```bash
cd "$REPO_ROOT"
git diff --no-color terraform/monitors/$team.tf
```

Show the diff to the user. Ask: "Approve and prepare the PR payload, or
iterate further?" Wait for answer.

## Step 9 — Hand off to `pr-handoff`

Assemble the payload:
- `drafted_files`: `[{path: "terraform/monitors/$team.tf", contents: <contents>}]`
- `pr_body`: fill the PR body template from `steps.md` with the confirmed values
- `override_rationale`: `overrides[]` collected above

Invoke the `pr-handoff` skill by calling the `Skill` tool with:
- `skill: "observability:pr-handoff"`
- `args: <a JSON object string containing the three payload keys, NOT a JSON-encoded string of a JSON string>`

For example, `args` should be the literal text:
`{"drafted_files": [{"path": "...", "contents": "..."}], "pr_body": "...", "override_rationale": [...]}`
— not the same content wrapped in another layer of quoting/escaping.

Do NOT substitute any alternative. Specifically:
- Do NOT run `git commit` or `gh pr create` from this skill. Those are
  `pr-handoff`'s job — not yours.
- Do NOT write the handoff JSON directly with the `Write` tool. The
  `pr-handoff` skill owns the artifact path and schema.
- Do NOT ask the user "should I commit this instead?" The answer is no.

If the `Skill` tool returns ANY error — "not available", "failed", a
non-success status from inside `pr-handoff`, or anything else — halt
and surface the error verbatim. Do not fall back, do not retry with a
different tool, and do not invent a workaround.

## Error handling

If any step halts (auth failure, validation failure, missing file, etc.),
DO NOT silently continue. Surface the error to the user and stop.
