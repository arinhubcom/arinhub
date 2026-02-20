---
name: arinhub-code-reviewer
description: "Orchestrate a comprehensive code review by launching parallel review subagents, deduplicating findings, and submitting the review. Use when asked to 'review PR 123', 'review my changes','code review PR #123', 'review local changes'. Input: PR number, URL, or nothing for local changes."
argument-hint: "PR number or URL (e.g., 100, #456, https://github.com/owner/repo/pull/789), or omit for local changes"
---

# Code Reviewer

Orchestrate a comprehensive code review by running multiple review strategies in parallel, merging and deduplicating findings into a review file. Supports both remote PRs and local uncommitted changes.

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
REVIEW_FILE=~/.agents/arinhub/code-reviews/pr-code-review-${REPO_NAME}-${PR_NUMBER}.md
```

**If `MODE=local`:**

Determine the repository name from git remote. Use the current branch name for identification, sanitizing slashes to dashes so file paths remain valid.

```
REPO_NAME=<repository name>
BRANCH_NAME=$(git branch --show-current | tr '/' '-')
REVIEW_FILE=~/.agents/arinhub/code-reviews/local-code-review-${REPO_NAME}-${BRANCH_NAME}.md
```

Create `~/.agents/arinhub/code-reviews/` and `~/.agents/arinhub/diffs/` directories if they do not exist.

### 3. Initialize Review File

**If `MODE=remote`:**

Create the review file with a header:

```markdown
# PR Review: ${REPO_NAME} #${PR_NUMBER}

**Date:** <current date>
**PR:** <PR URL>

## Issues

<!-- Issues from parallel review agents merged below. No duplicates. -->
```

**If `MODE=local`:**

Create the review file with a header:

```markdown
# Local Review: ${REPO_NAME} (${BRANCH_NAME})

**Date:** <current date>
**Branch:** ${BRANCH_NAME}

## Issues

<!-- Issues from parallel review agents merged below. No duplicates. -->
```

### 4. Prepare Diff and Working Tree

Save the diff to a shared file so subagents can read it. In remote mode, also check out the PR branch so tools that require a working tree (e.g., `react-doctor`) operate on the correct code.

**If `MODE=remote`:**

```bash
DIFF_FILE=~/.agents/arinhub/diffs/pr-diff-${REPO_NAME}-${PR_NUMBER}.md
gh pr diff ${PR_NUMBER} > ${DIFF_FILE}

# Check out the PR branch to ensure the working tree reflects the PR code for subagents that require it (e.g., react-doctor).
gh pr checkout ${PR_NUMBER}
```

**If `MODE=local`:**

```bash
DIFF_FILE=~/.agents/arinhub/diffs/local-diff-${REPO_NAME}-${BRANCH_NAME}.md
git diff HEAD > "${DIFF_FILE}"
```

No checkout is needed in local mode — the working tree already contains the changes.

### 5. Detect React Code

Spawn a subagent to analyze `${DIFF_FILE}` and determine whether the changes contain React code. The subagent must read the diff file and return `HAS_REACT=true` or `HAS_REACT=false`.

Set `HAS_REACT=true` if **any** of these conditions are found in the diff:

- **File extensions**: Changed files include `.tsx`, `.jsx`, or paths under common React directories (e.g., `components/`, `hooks/`, `pages/`)
- **React core imports**: `import ... from 'react'`, `import ... from "react"`, `require('react')`, `require("react")`
- **React DOM**: `import ... from 'react-dom'`, `import ... from 'react-dom/client'`
- **JSX syntax**: Diff hunks contain JSX elements (`<Component`, `<div`, `/>`, `React.createElement`)
- **React hooks**: Usage of `useState`, `useEffect`, `useRef`, `useMemo`, `useCallback`, `useContext`, `useReducer`, `useLayoutEffect`, or custom `use*` hooks
- **React ecosystem packages**: Imports from `react-router`, `react-hook-form`, `@tanstack/react-query`, `@tanstack/react-table`, `react-redux`, `zustand`, `jotai`, `recoil`, `next`, `@next/`, `styled-components`, `@emotion/`, or similar React-centric libraries

Otherwise set `HAS_REACT=false`.

### 6. Launch Parallel Review Subagents

Spawn subagents **in parallel** (do not wait for one to finish before starting the next). Each subagent must return ONLY a structured list of issues using the format from the Issue Format Reference section below. No subagent may submit a review — they only return findings.

Pass the diff file path (`${DIFF_FILE}`) to each subagent so they can read the diff directly. Inform each subagent that the working tree is already on the correct branch (PR branch in remote mode, current branch in local mode). **No subagent should run `gh pr checkout` or switch branches.**

- If `HAS_REACT=true`: spawn **four** subagents (A, B, C, D).
- If `HAS_REACT=false`: spawn **three** subagents (A, B, C) — skip Subagent D.

#### Subagent A: code-reviewer

**If `MODE=remote`:** Spawn a subagent to review PR `${PR_NUMBER}` using the `code-reviewer` skill. Pass `${DIFF_FILE}` for diff context.
**If `MODE=local`:** Spawn a subagent to review local changes using the `code-reviewer` skill. Pass `${DIFF_FILE}` for diff context.

Instruct it to return only the list of issues found — no review submission.

#### Subagent B: octocode-roast

Spawn a subagent to invoke the `octocode-roast` skill. Pass `${DIFF_FILE}` so it can read the diff directly.

**If `MODE=remote`:** Instruct it to review the diff in `${DIFF_FILE}` for PR `${PR_NUMBER}`.
**If `MODE=local`:** Instruct it to review the diff in `${DIFF_FILE}` for local changes.

Instruct it to return only the list of issues found — no review submission.

#### Subagent C: pr-review-toolkit

**If `MODE=remote`:** Spawn a subagent to review PR `${PR_NUMBER}` using the `pr-review-toolkit:review-pr` command with `all parallel` mode. Pass `${DIFF_FILE}` for diff context.
**If `MODE=local`:** Spawn a subagent to review local changes using the `pr-review-toolkit:review-pr` command with `all parallel` mode. Pass `${DIFF_FILE}` for diff context.

Instruct it to return only the list of issues found — no review submission.

#### Subagent D: react-doctor (only if `HAS_REACT=true`)

Spawn a subagent to run `react-doctor` on the working tree. The tool runs via `npx -y react-doctor@latest . --verbose --diff` and requires the working tree to be on the correct branch (already ensured by Step 4).

**If `MODE=remote`:** Inform the subagent that the PR branch is already checked out. Instruct it to review the React code in the current working tree with diff context from `${DIFF_FILE}`.
**If `MODE=local`:** Inform the subagent that the working tree already contains the local changes. Instruct it to review the React code in the current working tree with diff context from `${DIFF_FILE}`.

Instruct it to diagnose React-specific issues (performance, hooks misuse, component anti-patterns, security) and return only the list of issues found — no review submission.

### 7. Merge and Deduplicate Issues

Collect issues from all subagents (three or four, depending on `HAS_REACT`) and deduplicate:

1. Parse each subagent's response into individual issues.
2. For each issue, create a fingerprint from: `file path` + `line number range` + `concern category`.
3. Two issues are duplicates if they share the same file, overlapping line ranges (within ±5 lines), and address the same concern (use semantic comparison, not exact string matching).
4. When duplicates are found, keep the most detailed/actionable version.
5. Tag each kept issue with its source(s): `[code-reviewer]`, `[octocode-roast]`, `[pr-review-toolkit]`, `[react-doctor]`, or combination if multiple agents found it.

### 8. Write Issues to Review File

Append deduplicated issues to the review file, grouped by severity:

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

### 9. React Health Report (only if `HAS_REACT=true`)

If `HAS_REACT=true`, append the full output from the `react-doctor` subagent (Subagent D) to the review file under a dedicated section:

```markdown
## React Health

<full react-doctor report>
```

This section captures React-specific diagnostics (performance, hooks, component patterns, security) separately from the general deduplicated issues above.

If `HAS_REACT=false`, skip this section entirely.

### 10. Verify Requirements Coverage

Spawn a subagent to verify requirements coverage using the `arinhub-verify-requirements-coverage` skill. Pass the diff file path (`${DIFF_FILE}`) so the subagent can read the diff directly without fetching it again. The subagent must return the full requirements coverage report in markdown format.

**If `MODE=remote`:** Pass PR `${PR_NUMBER}` and `${DIFF_FILE}` to the subagent. It will use the diff file for analysis and resolve the linked issue automatically.

**If `MODE=local`:** Pass `${DIFF_FILE}` to the subagent. The subagent will analyze the diff against the linked issue.

Append the returned coverage report to the end of the review file under a new section:

```markdown
## Requirements Coverage

<coverage report content from arinhub-verify-requirements-coverage>
```

### 11. Submit PR Review (only if `MODE=remote`)

**Skip this step if `MODE=local`.**

Spawn a subagent to submit the review for PR `${PR_NUMBER}` using the `arinhub-submit-code-review` skill. Pass the review file path (`${REVIEW_FILE}`) so the subagent reads issues from it. The subagent must follow the `arinhub-submit-code-review` procedure for deduplication against existing PR comments before submission.

### 12. Report to User

**If `MODE=remote`:**

Present a summary:

- Path to the review file
- Total issues found (by severity)
- PR coverage percentage
- Whether the review was submitted successfully
- The PR URL for reference

**If `MODE=local`:**

Present the review file (`${REVIEW_FILE}`) content to the user and a summary:

- Path to the review file
- Total issues found (by severity)
- Requirements coverage percentage (if available)
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
- The review file is the single source of truth — all findings are merged there before submission.
- Deduplication uses semantic comparison: if two agents flag the same concern on the same code, only one entry is kept.
- The review file persists at `~/.agents/arinhub/code-reviews/` for future reference and audit.
- If a subagent fails or times out, proceed with results from the remaining agents and note the failure in the review file.
- The diff file persists at `~/.agents/arinhub/diffs/` and is shared read-only across all subagents. The PR branch checkout happens once in Step 4 before subagents launch — no subagent should run `gh pr checkout` or switch branches on its own.
- In `MODE=local`, step 11 (Submit PR Review) is skipped — the review is output only to the review file and presented to the user. Step 10 (Verify Requirements Coverage) runs if a linked issue can be determined from the branch name or user input.
