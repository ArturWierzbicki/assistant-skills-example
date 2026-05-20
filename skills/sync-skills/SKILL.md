---
name: sync-skills-from-github
description: Sync Assistant Skills from a required path in one GitHub repository into Grafana Assistant.
---

# Sync Skills From GitHub

## Inputs

- `repo`: GitHub repo to read. 
- `path`: required path inside `repo`. Reject if missing, blank, `.`, `/`, absolute, URL-like, contains `..`, contains backslashes, or contains wildcards.

Normalize `repo` to `<owner>/<repo>`. Normalize `path` by trimming whitespace, removing leading `./`, collapsing repeated `/`, and removing a trailing `/`. Use one source ref for the whole run: the repo default branch from `search_repositories`, unless the repo URL already pins a branch or commit and `path` agrees with it. Use that ref on every GitHub read and report it.

## Explicitly Approved Tool Calls

GitHub MCP reads, always scoped to the resolved repo and required path:

- `search_repositories` with `query=repo:<owner>/<repo>` and `minimal_output=false`.
- `get_file_contents` only with `owner=<owner>`, `repo=<repo>`, `ref=<source ref>`, and `path` equal to the required path, a child of it, or a discovered `SKILL.md` below it.
- `search_code` only as an accelerator, never as source of truth, with a query containing `repo:<owner>/<repo>`, `path:<path>`, and `filename:SKILL.md`.

Assistant Skill reads/writes, scoped to the current Grafana Assistant tenant and execution identity:

- `search_skills` with queries based on discovered source skill names.
- `manage_skills` with `operation=read` only for IDs returned by `search_skills`.
- `manage_skills` with `operation=create`, `name=<source name>`, `body=<source body plus GitHub source marker>`, `scope=user`, and `includeInKnowledgebase=true`; call only after manual confirmation unless controlled-sync policy is approved.
- `manage_skills` with `operation=update`, `id=<matched skill id>`, `name=<source name>`, and `body=<source body plus GitHub source marker>`; call only after manual confirmation unless controlled-sync policy is approved.

Never call GitHub write tools. Never call `manage_skills` with `operation=delete`. Never use Slack, Jira, ServiceNow, Kubernetes, cloud, database, incident, alerting, deployment, or repository-write tools.

## Workflow

1. Validate inputs. If `path` is absent or invalid, stop with: `path is required and must be a safe path inside the repo`.
2. Resolve `repo`. If omitted, use the default. Reject non-GitHub URLs or values that cannot become `<owner>/<repo>`.
3. Verify repo access with `search_repositories`; resolve and freeze `source_ref` for this run.
4. Discover source skills by recursive `get_file_contents` directory traversal. Treat the requested path as: direct `SKILL.md`, a skill folder containing `SKILL.md`, or a parent directory containing skill folders. Use `search_code` only to suggest candidate paths, then verify by directory/file reads. Stop before writes if discovery exceeds 50 skills, depth 5, or any page/listing cannot be read completely.
5. For each `SKILL.md`, parse YAML front matter. Require non-empty `name`, non-empty `description`, and non-empty markdown body. Prefer lowercase hyphen-case names but do not reject display-style names. Ignore `SKILLS.md` because it is not the official skill filename.
6. Reject duplicate source names after trim and casefold.
7. If a skill directory contains files other than `SKILL.md`, skip it as `supporting_files` unless the user explicitly requests `SKILL.md`-only sync. Report the sibling file names.
8. Build the desired Assistant body as: source body without YAML front matter, then a stable import marker: `<!-- grafana-assistant-import-source: {"owner":"<owner>","repo":"<repo>","branch":"<source_ref>","path":"<skill path>"} -->`.
9. For each valid source skill, use `search_skills` only to find candidates. Read every candidate with `manage_skills operation=read`, paging with `offset` until `hasMore=false`, then parse the GitHub source marker from the full body.
10. Update only an existing Skill whose source marker matches the same owner, repo, and path. If a same-name Skill has no marker or a different marker, skip as `name_collision`; do not overwrite it.
11. If no matching marked Skill exists, create only after manual confirmation. In background runs without controlled-sync policy, report `planned_create` instead of calling `manage_skills create`.
12. If one matching marked Skill exists, update only when the full desired body differs after trimming outer whitespace. In background runs without controlled-sync policy, report `planned_update` instead of calling `manage_skills update`.
13. Do not delete Assistant Skills missing from GitHub.
14. Continue after per-skill read/create/update failures; put those in `failed`. There is no rollback. Stop the whole run only for invalid inputs, repo access failure, unsafe path, incomplete discovery, or exceeded blast-radius limits.
15. Report counts and names for created, updated, unchanged, planned_create, planned_update, skipped, rejected, and failed skills. Include repo, path, source_ref, and every skipped/rejected/failed reason.

## Output

Return a compact sync report:

```text
repo: <owner>/<repo>
path: <path>
source_ref: <branch-or-sha>
created: <n> [...]
updated: <n> [...]
unchanged: <n> [...]
planned_create: <n> [...]
planned_update: <n> [...]
skipped: <n> [...]
rejected: <n> [...]
failed: <n> [...]
```

If `manage_skills` is unavailable, do a dry run only and return the exact create/update plan.
