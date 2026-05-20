---
name: Autoremediate Recent Investigations
description: Review recent Assistant Investigations, identify findings remediable in one approved GitHub repo, and submit resolution PRs with evidence, validation, and rollback.
---

# Autoremediate Recent Investigations

## Purpose

Review recent Assistant Investigations and submit resolution pull requests for findings safely remediable in one approved GitHub repository.

- Read, search, branch, write files, and create PRs only in the approved repo.
- Never merge PRs.

## Inputs

- `repo`: the only GitHub repo tool calls may target. 
- `timeframe`: investigations to consider. Default: `most recent remediable within last 30 minutes`.
- `description`: what else to consider

Supported `timeframe`:

- `specific investigation <url-or-id>`
- `most recent remediable within <duration>`
- `all remediable investigations from <start> to <end>, max <n>`; if `n` is omitted, use `max 3`.

If `timeframe` is ambiguous, use the default.

## Explicitly Auto Approved Tool Calls // allowlist

Investigation reads:

- `investigation` with `command=list`.
- `investigation` with `command=summary` only for IDs from `command=list` or explicit prompt input.

GitHub reads:

- `get_me`.
- `search_repositories` with `query=repo:<owner>/<repo>` and `minimal_output=false`.
- `list_branches` with `owner=<owner>` and `repo=<repo>`.
- `search_code` with every query including `repo:<owner>/<repo>`.
- `get_file_contents` with `owner=<owner>`, `repo=<repo>`, and a repo path.
- `pull_request_read` only for PRs created by this run; allowed methods: `get`, `get_diff`, `get_status`, `get_files`, `get_check_runs`.

GitHub writes:

- `create_branch` with `owner=<owner>`, `repo=<repo>`, `branch=assistant/autoremediate-*`.
- `push_files` with `owner=<owner>`, `repo=<repo>`, `branch=assistant/autoremediate-*`.
- `create_pull_request` with `owner=<owner>`, `repo=<repo>`, `head=assistant/autoremediate-*`, and `base=<default-branch>`.

Auto-approve GitHub writes only when the GitHub server credential or admin policy is restricted to the resolved repo.

## Repo Scope Gate

1. Normalize `repo` to `owner/name`
2. State the resolved repo before tool use.
3. Use only tool calls whose arguments show that repo.
4. Ignore search results from any other repo.
5. If a write call cannot show repo, branch, and files before execution, stop.
6. If an investigation points to another repo, skip it as outside scope.

## Investigation Discovery Gate

1. Prefer explicit investigation URLs or IDs.
2. Otherwise call `investigation` with `command=list`.
3. Keep only investigations inside `timeframe`.
4. For candidates, call `investigation` with `command=summary`.
5. If list/summary is unavailable, stop and report `no investigation access`.

Do not invent investigations. Do not use old chat memory as proof.

## Remediability Gate

An investigation is remediable only if:

1. It describes a concrete code or config defect.
2. The affected component maps to files in the approved repo.

## Evidence Workflow

1. Read investigation summary and evidence.
2. Extract service, alert, timeframe, impact, suspected root cause, and links.
3. Search the approved repo for service names, labels, paths, alerts, manifests, or identifiers.
4. Use `search_code` and `get_file_contents`.
5. Read target files before proposing a change.
6. Prefer one small fix.

## Branch and PR Rules

1. Use the repository default branch as base.
2. Create `assistant/autoremediate-<short-investigation-id>-<yyyy-mm-dd>`.
3. Modify only required files.
4. Do not touch generated, vendored, lock, or unrelated formatting files unless required.
5. Create a PR.
6. Never merge.
7. Combine investigations only when they share the same root cause and files.

## PR Title

`Resolve <service-or-component> issue from Assistant investigation <short-id>`

If there is no short ID:

`Resolve <service-or-component> issue from recent Assistant investigation`

## PR Body

```text
## Summary

- What changed
- Why this is the smallest safe fix

## Investigation evidence

- Investigation: <url-or-id>
- Timeframe: <timeframe>
- Affected service/component: <service-or-component>
- Symptom: <symptom>
- Root-cause hypothesis: <hypothesis>

## Validation

- <validation step 1>
- <validation step 2>
- <validation step 3>

## Rollback

- <how to revert safely>

## Scope and safety

- Repository: <owner/name>
- Files changed: <files>
- No production write was performed
- This PR was not merged automatically
```

Use source links from the investigation or repository. Do not cite local files.

## Run Output

Write back on slack via MCP or if slack is not available - return a list of created PRs

## Hard Stops

- Cannot access requested investigations.
- Cannot identify the approved repo.
- Tool call targets another repo.
- Write tool does not show repo, branch, and files.
- GitHub MCP tools are auto-approved without repo restriction.
- Fix requires credentials, customer data, production mutation, or privileged live-system access.
- Only available action is merge, deploy, live Kubernetes change, Jira/ServiceNow update, Slack notification, cloud write, incident/alerting write, or database write.
