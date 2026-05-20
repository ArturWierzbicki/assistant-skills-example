---
name: autoremediate-recent-investigations
description: Review recent Assistant Investigations and submit resolution PRs for discovered issues.
---

# Autoremediate Recent Investigations

## Purpose

Review recent Assistant Investigations and submit resolution pull requests for findings remediable in one approved GitHub repository.

- Read, search, branch, write files, and create PRs only in the approved repo.
- Never merge PRs.

## Inputs

- `repo`: the only GitHub repo tool calls may target.
- `timeframe`: investigations to consider. Default: `most recent remediable within last 30 minutes`.
- `description`: what else to consider

Supported `timeframe`:

- `specific investigation `
- `most recent remediable within `
- `all remediable investigations from to , max `; if `n` is omitted, use `max 3`.

If `timeframe` is ambiguous, use the default.

## Explicitly Auto Approved Tool Calls // allowlist

Investigation reads:

- `investigation` with `command=list`.
- `investigation` with `command=summary` only for IDs from `command=list` or explicit prompt input.

GitHub reads:

- `get_me`.
- `search_repositories` with `query=repo:/` and `minimal_output=false`.
- `list_branches` with `owner=` and `repo=`.
- `search_code` with every query including `repo:/`.
- `get_file_contents` with `owner=`, `repo=`, and a repo path.
- `pull_request_read` only for PRs created by this run; allowed methods: `get`, `get_diff`, `get_status`, `get_files`, `get_check_runs`.

Pre-approved GitHub writes - you must be able to execute them without manual approvals for the matching owner, repo, branch:

- `create_branch` with `owner=`, `repo=`,
- `push_files` with `owner=`, `repo=`, 
- `create_pull_request` with `owner=`, `repo=`, and `base=`.

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

- Investigation: 
- Timeframe: 
- Affected service/component: 
- Symptom: 
- Root-cause hypothesis: 

## Validation

- 
- 
- 

## Rollback

- 

## Scope and safety

- Repository: 
- Files changed: 
- No production write was performed
- This PR was not merged automatically
```

Use source links from the investigation or repository. Do not cite local files.

## Run Output

Fix the issues in code, submit PRs, verify the PRs exist, then return a list of created PRs
For any potential issues, ALWAYS just open the PR, do not ask for confirmations.



## Hard Stops

- Cannot identify the approved repo.
- Tool call targets another repo.
- Write tool does not show repo, branch, and files.
