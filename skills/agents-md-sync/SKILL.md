---
name: agents-md-sync
description: Sync project AGENTS.md files with a central rules repository without requiring Python or another project runtime. Use when Codex initializes a project, starts documentation work, bug fixes, feature development, updates AGENTS.md, compares local AI rules with the central AGENTS.md, honors .agents-sync.json opt-out settings, or pushes a project AGENTS.md back to git@github.com:tiankongzhise/agents_repo.git using the current GitHub repository name as the branch.
---

# AGENTS.md Sync

## Core Rule

Synchronize `AGENTS.md` by using universally available project tools. Prefer GitHub CLI (`gh`) when it is installed and authenticated; otherwise use plain `git`. Do not require Python, Node, Go, Rust, or any project runtime in the target project.

Use this skill in three moments:

- Project initialization, before creating or finalizing local `AGENTS.md`.
- Before documentation edits, bug fixes, feature work, refactors, tests, migrations, or deployment changes.
- After local `AGENTS.md` changes, before finishing the task, to publish the project version back to the central repository.

For exact PowerShell and Bash command templates, read `references/commands.md` when executing the workflow.

## Defaults

- Central repository: `git@github.com:tiankongzhise/agents_repo.git`
- Central branch: `main`
- Central file: `AGENTS.md`
- Local file candidates: configured path, then `AGENTS.md`, `agents.md`, `AGENTS.MD`
- Local config: `.agents-sync.json`
- Change reports: `docs/agents-sync/`
- Publish branch: sanitized current GitHub repository name, not owner name

Honor `.agents-sync.json` if present. Treat missing config as:

```json
{
  "enabled": true,
  "auto_update": true,
  "auto_publish_on_agents_change": true,
  "central_repo": "git@github.com:tiankongzhise/agents_repo.git",
  "central_branch": "main",
  "central_agents_path": "AGENTS.md",
  "local_agents_path": "AGENTS.md",
  "change_log_dir": "docs/agents-sync"
}
```

If `enabled` is `false`, do not fetch, merge, or publish automatically. If `auto_update` is `false`, fetch may be used for inspection, but do not write local `AGENTS.md` unless the user explicitly asks.

Read `.agents-sync.json` directly as text when no JSON parser is available. This skill only needs a few simple fields for v1: `enabled`, `auto_update`, `auto_publish_on_agents_change`, path overrides, and central repository overrides. Do not introduce a runtime dependency just to parse optional config.

## Tool Selection

1. Check `git --version`. If Git is unavailable, generate manual instructions and stop before mutating files.
2. Check `gh --version` and `gh auth status`.
3. Use `gh` for reading GitHub repository metadata and raw central `AGENTS.md` only when it works.
4. Fall back to `git ls-remote`, `git fetch`, `git show`, and `git remote get-url origin` for all required operations.
5. If SSH access to `git@github.com:tiankongzhise/agents_repo.git` is blocked, retry the same Git flow with `https://github.com/tiankongzhise/agents_repo.git` before giving up.
6. Use temporary directories under the local workspace or system temp. Do not leave cloned central repositories behind unless needed for troubleshooting.

## Initialize Or Preflight

1. Read the local config and stop if synchronization is disabled.
2. Fetch the central `AGENTS.md` from `main`.
3. Locate the local `AGENTS.md`. If none exists during project initialization, create `AGENTS.md` from the central file and write a report.
4. If a local file exists, compare it with the central file and classify central rules:
   - Add: general, applicable, and non-conflicting constraints.
   - Skip: project-specific, technology-mismatched, duplicate, or already represented constraints.
   - Review: constraints that might conflict with local policy or change project workflow.
5. Write automatic additions only inside a managed block:

```markdown
<!-- agents-sync:start source=tiankongzhise/agents_repo branch=main -->
...
<!-- agents-sync:end -->
```

Do not rewrite local hand-authored sections outside the managed block.

## Change Report

After every automatic local update, create `docs/agents-sync/<YYYYMMDD-HHMMSS>-agents-sync.md` with:

- Central repository, branch, and commit if available.
- Local `AGENTS.md` path.
- Constraints added automatically.
- Constraints skipped and why.
- Constraints requiring user review.
- Commands or tool path used: `gh` or `git`.

Tell the user about the report and summarize the automatic changes in the final response.

## Publish Local AGENTS.md Back

When local `AGENTS.md` changes and publishing is enabled:

1. Resolve the current project GitHub repository name.
   - Prefer `gh repo view --json name --jq .name`.
   - Fall back to parsing `git remote get-url origin`.
2. Sanitize the name to lowercase letters, digits, `.`, `_`, and `-`; replace other characters with `-`.
3. Create a temporary clone or worktree of the central repository.
4. Start from the latest central `main`.
5. Create or reset the publish branch named after the GitHub repository.
6. Replace central `AGENTS.md` with the current project's local `AGENTS.md`.
7. Commit only if there is a file difference. Use a Chinese commit message when working in a Chinese project.
8. Push the branch to the central repository.
9. If network or authentication blocks the push, report the exact failed command and leave local project changes untouched.

## Merge Judgment

Automatically merge only constraints that are clearly reusable across projects, such as UTF-8 handling, Git history reading, sensitive information rules, evidence preservation, documentation consistency, and sandbox issue recording.

Do not automatically merge central constraints that name a product, service URL, database schema, release phase, queue, cloud provider, UI framework, or domain-specific path unless the target project already uses that item.

If a local rule is stricter than the central rule, keep the local rule and record the central rule as skipped because already covered.

If a central rule conflicts with a local rule, do not change the local file automatically; put it in the review list.
