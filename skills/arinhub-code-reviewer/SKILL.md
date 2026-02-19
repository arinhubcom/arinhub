---
name: arinhub-code-reviewer
description: "Orchestrate a comprehensive code review by launching parallel review subagents, deduplicating findings, and submitting the review. Use when asked to 'review PR 123', 'code review PR #123', 'code review', 'review code changes'. Input: PR number, URL, or nothing for local changes."
argument-hint: "PR number or URL (e.g., 100, #456, https://github.com/owner/repo/pull/789), or omit for local changes"
---

# Code Reviewer

Orchestrate a comprehensive code review by running multiple review strategies in parallel, merging and deduplicating findings into a log file. Supports both remote PRs and local uncommitted changes.

## Input

- **PR number or URL** (optional): Accepts `123`, `#123`, or full URL. If omitted, reviews local changes.

## Procedure

### 1. Determine Review Target

- **Remote PR**: If the user provides a PR number or URL (e.g., "Review PR #123"), target that remote PR. Set `MODE=remote`.
- **Local Changes**: If no specific PR is mentioned, or if the user asks to "review my changes", target the current local file system changes (staged and unstaged). Set `MODE=local`.

### 2. Resolve Identifier and Repository

**If `MODE=remote`:**

Extract the PR number. Determine the repository name from git remote or the provided URL.

```
PR_NUMBER=<extracted number>
REPO_NAME=<repository name, e.g. "my-app">
LOG_FILE=~/.agents/logs/arinhub-pr-review-${REPO_NAME}-${PR_NUMBER}.md
```

**If `MODE=local`:**

Determine the repository name from git remote. Use the current branch name for identification.

```
REPO_NAME=<repository name>
BRANCH_NAME=$(git branch --show-current)
LOG_FILE=~/.agents/logs/arinhub-local-review-${REPO_NAME}-${BRANCH_NAME}.md
```

Create `~/.agents/logs/` directory if it does not exist.

### 3. Initialize Log File

**If `MODE=remote`:**

Create the log file with a header:

```markdown
# PR Review: ${REPO_NAME} #${PR_NUMBER}

**Date:** <current date>
**PR:** <PR URL>

## Issues

<!-- Issues from parallel review agents merged below. No duplicates. -->
```

**If `MODE=local`:**

Create the log file with a header:

```markdown
# Local Review: ${REPO_NAME} (${BRANCH_NAME})

**Date:** <current date>
**Branch:** ${BRANCH_NAME}

## Issues

<!-- Issues from parallel review agents merged below. No duplicates. -->
```

### 4. Detect React Code

Before launching subagents, check whether the changes contain React code.

**If `MODE=remote`:**

```
gh pr diff ${PR_NUMBER} --name-only
```

**If `MODE=local`:**

```
git diff --name-only HEAD && git diff --cached --name-only
```

Set `HAS_REACT=true` if **any** of these conditions are met:

- Changed files include `.tsx` or `.jsx` extensions
- Changed files contain `import ... from 'react'` or `import ... from "react"` patterns

Otherwise set `HAS_REACT=false`.

### 5. Launch Parallel Review Subagents

Spawn subagents **in parallel** (do not wait for one to finish before starting the next). Each subagent must return ONLY a structured list of issues using the format from the Issue Format Reference section below. No subagent may submit a review — they only return findings.

- If `HAS_REACT=true`: spawn **four** subagents (A, B, C, D).
- If `HAS_REACT=false`: spawn **three** subagents (A, B, C) — skip Subagent D.

#### Subagent A: code-reviewer

**If `MODE=remote`:** Spawn a subagent to review PR `${PR_NUMBER}` using the `code-reviewer` skill.
**If `MODE=local`:** Spawn a subagent to review local changes using the `code-reviewer` skill. Instruct it to review the current working tree changes (staged and unstaged).

Instruct it to return only the list of issues found — no review submission.

#### Subagent B: octocode-roast

**If `MODE=remote`:**

1. Run `gh pr checkout ${PR_NUMBER}` to switch to the PR branch.
2. Invoke the `octocode-roast` skill with the prompt `review PR`.

**If `MODE=local`:**

1. Invoke the `octocode-roast` skill with the prompt `review local changes`.

Instruct it to return only the list of issues found — no review submission.

#### Subagent C: pr-review-toolkit

**If `MODE=remote`:** Spawn a subagent to review PR `${PR_NUMBER}` using the `pr-review-toolkit:review-pr` command with `all parallel` mode.
**If `MODE=local`:** Spawn a subagent to review local changes using the `pr-review-toolkit:review-pr` command with `all parallel` mode.

Instruct it to return only the list of issues found — no review submission.

#### Subagent D: react-doctor (only if `HAS_REACT=true`)

**If `MODE=remote`:** Spawn a subagent to review PR `${PR_NUMBER}` using the `react-doctor` skill.
**If `MODE=local`:** Spawn a subagent to review local changes using the `react-doctor` skill. Instruct it to focus on the changed files.

Instruct it to diagnose React-specific issues (performance, hooks misuse, component anti-patterns, security) and return only the list of issues found — no review submission.

### 6. Merge and Deduplicate Issues

Collect issues from all subagents (three or four, depending on `HAS_REACT`) and deduplicate:

1. Parse each subagent's response into individual issues.
2. For each issue, create a fingerprint from: `file path` + `line number range` + `concern category`.
3. Two issues are duplicates if they share the same file, overlapping line ranges (within ±5 lines), and address the same concern (use semantic comparison, not exact string matching).
4. When duplicates are found, keep the most detailed/actionable version.
5. Tag each kept issue with its source(s): `[code-reviewer]`, `[octocode-roast]`, `[pr-review-toolkit]`, `[react-doctor]`, or combination if multiple agents found it.

### 7. Write Issues to Log File

Append deduplicated issues to the log file, grouped by severity:

````markdown
### Critical

- **[source]** `path/to/file.ts:42` — Description of the issue.

  ```ts
  // the problematic code from the PR diff
  const result = unsafeOperation(input);
  ```

  ```diff
  - const result = unsafeOperation(input);
  + const result = safeOperation(sanitize(input));
  ```

### Improvements

- **[source]** `path/to/file.ts:88-95` — Description of the issue.

  ```ts
  // the problematic code from the PR diff
  items.forEach((item) => {
    process(item);
  });
  ```

  ```diff
  - items.forEach(item => {
  -   process(item);
  - });
  + await Promise.all(items.map(item => process(item)));
  ```

### Nitpicks

- **[source]** `path/to/file.ts:12` — Description of the issue.

  ```ts
  // the relevant code snippet
  let x = getValue();
  ```

---

**Total issues:** N (X critical, Y improvements, Z nitpicks)
**Sources:** code-reviewer, octocode-roast, pr-review-toolkit[, react-doctor] (include react-doctor only if HAS_REACT=true)
````

### 8. React Health Report (only if `HAS_REACT=true`)

If `HAS_REACT=true`, append the full output from the `react-doctor` subagent (Subagent D) to the log file under a dedicated section:

```markdown
## React Health

<full react-doctor report>
```

This section captures React-specific diagnostics (performance, hooks, component patterns, security) separately from the general deduplicated issues above.

If `HAS_REACT=false`, skip this section entirely.

### 9. Verify PR Requirements Coverage (only if `MODE=remote`)

**Skip this step if `MODE=local`.**

After writing issues, spawn a subagent to verify PR `${PR_NUMBER}` using the `arinhub-verify-requirements-coverage` skill. The subagent must return the full PR Requirements Coverage report in markdown format.

Append the returned coverage report to the end of the log file under a new section:

```markdown
## PR Requirements Coverage

<coverage report content from arinhub-verify-requirements-coverage>
```

### 10. Submit PR Review (only if `MODE=remote`)

**Skip this step if `MODE=local`.**

Spawn a subagent to submit the review for PR `${PR_NUMBER}` using the `arinhub-submit-code-review` skill. Pass the log file path (`${LOG_FILE}`) so the subagent reads issues from it. The subagent must follow the `arinhub-submit-code-review` procedure for deduplication against existing PR comments before submission.

### 11. Report to User

**If `MODE=remote`:**

Present a summary:

- Path to the log file
- Total issues found (by severity)
- PR coverage percentage
- Whether the review was submitted successfully
- The PR URL for reference

**If `MODE=local`:**

Present a summary:

- Path to the log file
- Total issues found (by severity)
- Branch name and list of changed files reviewed

## Issue Format Reference

Each issue in a subagent response must follow this structure:

````markdown
- **Severity:** critical | improvement | nitpick
  **File:** path/to/file.ts
  **Line(s):** 42 (or 42-50)
  **Description:** Clear explanation of the problem.
  **Code:**
  ```ts
  // the problematic code from the PR diff
  const result = unsafeOperation(input);
  ```
  **Suggestion:**
  ```diff
  - const result = unsafeOperation(input);
  + const result = safeOperation(sanitize(input));
  ```
````

## Important Notes

- Review subagents run in parallel to minimize total review time (three or four, depending on whether the changes contain React code).
- The `react-doctor` subagent is only launched when the diff contains `.tsx`/`.jsx` files or React imports. This avoids unnecessary React diagnostics on non-React changes.
- The log file is the single source of truth — all findings are merged there before submission.
- Deduplication uses semantic comparison: if two agents flag the same concern on the same code, only one entry is kept.
- The log file persists at `~/.agents/logs/` for future reference and audit.
- If a subagent fails or times out, proceed with results from the remaining agents and note the failure in the log file.
- In `MODE=local`, steps 9 (Verify PR Requirements Coverage) and 10 (Submit PR Review) are skipped entirely — the review is output only to the log file and presented to the user.
