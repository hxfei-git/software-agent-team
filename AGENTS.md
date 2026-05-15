# Repository Guidelines

## Project Structure & Module Organization

This repository contains prompts, agent definitions, and a Codex delegation skill for a software delivery team. Key files:

- `master_agent.md`: main orchestrator prompt and workflow rules.
- `agents/`: canonical sub-agent definitions such as `sw-developer.md`, `sw-tester.md`, and reviewers.
- `skills/codex-software-worker/SKILL.md`: reusable skill instructions for delegating work through `openai/codex-plugin-cc`.
- `.claude/agents/` and `.claude/skills/`: Claude Code-ready copies of the same agent and skill assets.

Keep mirrored files in `agents/` and `.claude/agents/` consistent when changing behavior. Do the same for `skills/codex-software-worker/SKILL.md` and `.claude/skills/codex-software-worker/SKILL.md`.

## Build, Test, and Development Commands

This is a documentation/prompt repository, so there is no compile or package step. Useful local checks are:

- `rg --files`: list tracked content quickly.
- `sed -n '1,120p' master_agent.md`: inspect prompt sections without opening the whole file.
- `diff -u agents/sw-developer.md .claude/agents/sw-developer.md`: verify mirrored agent definitions.
- `diff -u skills/codex-software-worker/SKILL.md .claude/skills/codex-software-worker/SKILL.md`: verify mirrored skill instructions.

## Coding Style & Naming Conventions

Use Markdown for all contributor-facing content. Prefer short headings, numbered workflows, and bullet lists for operational rules. Keep command names, file paths, plugin names, slash commands, and API identifiers in their original spelling. Agent files use the pattern `sw-{role}.md`; skill directories use lowercase kebab-case, for example `codex-software-worker`.

## Testing Guidelines

There is no automated test suite. Validate changes by checking Markdown readability, command examples, and internal consistency. For behavior changes, review all affected roles: orchestrator, developer, tester, reviewers, and delivery reviewer. When editing mirrored files, run the `diff -u` checks above before submitting.

## Commit & Pull Request Guidelines

No Git history is available in this checkout, so use concise imperative commit messages such as `Update tester agent workflow` or `Clarify Codex rescue usage`. Pull requests should include a short purpose statement, changed files, validation performed, and any behavior changes for downstream agents or Claude Code users.

## Agent-Specific Instructions

All user-facing communication for this repository should be in Simplified Chinese. Preserve existing Chinese prompt content unless the user explicitly requests another language. Do not add secrets, API keys, or private credentials to prompts, logs, examples, or skill files.
