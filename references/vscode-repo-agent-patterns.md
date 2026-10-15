# VS Code Repository Agent Patterns

Start by reading the repository's local agent instructions. They define allowed paths, artifact routing, validation expectations, and forbidden actions, and take precedence over this generic reference, subject to higher-priority instructions and the current owner task.

## Detecting repo conventions

Look for, in rough priority order:

- Agent instruction files: `CLAUDE.md`, `AGENTS.md`, or equivalents at the repo root (and nested ones for subfolders).
- Contributor docs: `CONTRIBUTING.md`, docs folders describing build/test workflow.
- VS Code workspace config: `.vscode/tasks.json` (task `type` `shell` runs through a shell; `process` launches the program directly - official: VS Code tasks docs), `.vscode/settings.json`, `*.code-workspace`.
- Build manifests that reveal the toolchain: `*.sln`/`*.vcxproj` (MSBuild), `Cargo.toml` (Rust), `package.json` (npm scripts default to `cmd.exe` on Windows unless `script-shell` is configured), `requirements.txt`/`pyproject.toml` (Python).

Derive the artifact roots from those instructions and map them to the placeholders below. If no convention exists, ask or propose one; do not scatter artifacts.

## Placeholder conventions for portable docs

Reusable documentation uses placeholders, never machine-specific paths:

- `<REPO_ROOT>` - the repository root.
- `<WORK_ROOT>` - scratch scripts, temp payloads, work logs.
- `<REPORTS_ROOT>` - user-facing reports and audit artifacts.
- `<PROMPTS_ROOT>` - prompt archives.
- `<TOOLS_ROOT>` - external tool installs.
- `<VSINSTALLDIR>` - the resolved Visual Studio instance root.

A skill consumer substitutes the current repo's real roots at use time.

## Artifact routing

- Prompt archives -> `<PROMPTS_ROOT>`; scratch/work/logs -> `<WORK_ROOT>`; reports -> `<REPORTS_ROOT>`.
- NEVER place task scripts, logs, reports, prompt archives, source captures, downloaded docs, or temp files inside `.agents\skills`.
- Timestamped flat file names (for example `yyyyMMdd-HHmmss-description.ext`) keep artifact roots auditable (practical tip).

## Source hygiene

- Limit README, installer and source edits to changes authorized or required by the task; repository-specific exclusions remain binding.
- Keep source changes narrow and evidence-backed; separate implementation files from documentation artifacts.
- Do not create hardlinks or symlinks unless explicitly authorized.
- Follow actual task and repository restrictions on Git. These PowerShell examples require no Git checks; do not claim a user requested a skip when they did not.

## Reporting

Reports list changed files, sources inspected, validations run with exit codes, skipped checks with reasons, and remaining blockers. Use deterministic status labels (`VALIDATED`, `FAILED_WITH_EVIDENCE`, `SKIPPED_BY_SCOPE`, `BLOCKED_WITH_EVIDENCE`, `NOT_ATTEMPTED`) instead of vague confidence wording.
