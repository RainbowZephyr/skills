---
name: merge-summary
description: Generate a merge request description from branch commits using the bundled `pull_request_template.md`. The user supplies the MR title and optionally a base branch (default `dev`). Returns raw markdown to copy.
argument-hint: <mr-title> [--base <branch>]
disable-model-invocation: true
allowed-tools: Read, Bash(git log *), Bash(git show *), Bash(git diff *), Bash(git branch *), Bash(git rev-parse *)
---

# Merge Summary

Generate a merge request description for the current branch. This skill is read-only.

## Inputs

Parse the skill argument (`$ARGUMENTS`). If it is empty or unsubstituted, use what the user wrote in their message instead.

- **Base branch** — the value after `--base`, if present. Otherwise `dev`.
- **MR title** — everything else, used verbatim. Do not generate or reword it.

**The MR title is required.** If there is none, stop and ask the user for it before doing anything else.

**IMPORTANT**: Run each `git` command on its own, with no `&&`, `||`, `;`, or pipes, so tool permissions match.

## Step 1 — Identify the current branch, ticket, and base

Run `git branch --show-current` to get the current branch name.

If the branch name contains a ticket ID (e.g. `task-123`), reference it in the Description. If there is none, omit the ticket reference.

Resolve the base branch to a ref that exists:

1. Run `git rev-parse --verify --quiet <base>`. If it succeeds, use `<base>`.
2. Otherwise run `git rev-parse --verify --quiet origin/<base>`. If it succeeds, use `origin/<base>`.
3. If neither exists, stop and tell the user the base branch was not found.

## Step 2 — Read the merge request template

Read `pull_request_template.md` from this skill's directory (the folder containing this SKILL.md). Replace `<MR Title>` with the MR title.

**CRITICAL**: The template structure MUST be respected exactly:

- Keep ALL sections, even if a section has no relevant content — leave its placeholder or checklist.
- Do not add new sections.
- Do not reorder sections.
- Keep the template's `<!-- ... -->` guidance comments in place.

## Step 3 — Analyze commits on this branch

Run `git log --oneline --no-merges <base>..HEAD` to list the branch's own commits (merges of other branches into this one are excluded as noise).

For each commit, run `git show --stat <sha>` to understand the scope.

For commits with non-obvious messages, run `git show <sha>` to read the actual changes.

Run `git diff --name-only <base>..HEAD` to get the full list of changed files. Steps 4 and 5 use it.

Synthesize the commits into a coherent narrative. Focus on:

- **What** changed (features, fixes, refactors)
- **Why** it changed (motivation, context, linked issues)
- **Important details** reviewers should know (breaking changes, migration steps, trade-offs)

Do not just list commit messages verbatim — group related commits and describe the overall intent.

## Step 4 — Derive the Change Type

Tick exactly one box (`- [ ]` → `- [x]`) based on the conventional-commit types of the branch's commits. A type matches with or without a scope, e.g. `feat:` and `feat(api-v2):` are both `feat`.

| Commit type                                   | Tick             |
| --------------------------------------------- | ---------------- |
| `feat`                                        | ✨ New feature   |
| `fix`                                         | 🐛 Bug fix       |
| `refactor`                                    | ♻️ Refactor      |
| `perf`                                        | ⚡ Performance   |
| `docs`                                        | 📝 Documentation |
| `chore`, `ci`, `build`, `style`, `test`       | 🔧 Chore         |

For mixed types, pick the one that represents the **primary** change — use the changed files to judge where the weight of the work is (e.g. a `feat` with supporting `docs` and `test` commits is a New feature). If it is genuinely unclear, leave all boxes unchecked and mention the ambiguity in Step 6.

## Step 5 — Fill in the template

Fill the template sections in their template order:

1. **`🏷️ Change Type`** — the single tick from Step 4.

2. **`📝 Description`** — a bold one-line summary of the intent (with the ticket, if any), then a short paragraph synthesized from the commit narrative. Example:

   ```text
   **Adds a v2 setup endpoint that accepts an optional gid** (per task-123).

   `POST /api/v2/setup` takes an optional `gid` and carries it through office setup to the created office, instead of always generating a new one. Non-string gids are rejected with a 422.
   ```

3. **`👀 What to Look For`** — point the reviewer at the areas that need the most attention, based on the changed files and the diffs you read: tricky logic, schema or migration changes, auth/permission changes, edge cases, anything risky to deploy. A short bullet list is fine. If nothing stands out, leave the placeholder.

## Step 6 — Present to the user

Show the filled template as raw markdown in a single fenced code block for the user to copy. After it, note anything that needs their attention (e.g. an ambiguous Change Type).

**NEVER:**

- Use `--force` or any destructive flags.
- Run `git push`, `git commit`, or any other git write command — this skill is read-only.
