---
name: arinhub-review-pr
description: "Orchestrate a comprehensive pull request code review by launching parallel review subagents, deduplicating findings, and submitting the review. Use when asked to 'review PR', 'code review PR #123', 'arinhub review', or 'full PR review'. Input: PR number or URL."
argument-hint: "PR number or URL (e.g., 100, #456, https://github.com/owner/repo/pull/789)"
---

# Review PR

Orchestrate a comprehensive code review of a pull request by running multiple review strategies in parallel, merging and deduplicating findings into a log file, verifying implementation coverage against the linked issue, and submitting the review to the PR.

## Input

- **PR number or URL** (required): Accepts `123`, `#123`, or full URL.

## Procedure

### 1. Resolve PR Identifier and Repository

Extract the PR number. Determine the repository name from git remote or the provided URL.

```
PR_NUMBER=<extracted number>
REPO_NAME=<repository name, e.g. "my-app">
LOG_FILE=~/.agents/logs/arinhub-pr-review-${REPO_NAME}-${PR_NUMBER}.md
```

Create `~/.agents/logs/` directory if it does not exist.

### 2. Initialize Log File

Create the log file with a header:

```markdown
# PR Review: ${REPO_NAME} #${PR_NUMBER}

**Date:** <current date>
**PR:** <PR URL>

## Issues

<!-- Issues from parallel review agents merged below. No duplicates. -->
```

### 3. Launch Parallel Review Subagents

Spawn **four subagents in parallel**. Each agent performs an independent code review and returns a structured list of issues.

Spawn all four subagents **in parallel** (do not wait for one to finish before starting the next). Each subagent must return ONLY a structured list of issues using the format from the Issue Format Reference section below. No subagent may submit a review — they only return findings.

#### Subagent A: code-reviewer

Spawn a subagent to review PR `${PR_NUMBER}` using the `code-reviewer` skill. Instruct it to return only the list of issues found — no review submission.

#### Subagent B: octocode-research

Spawn a subagent to review PR `${PR_NUMBER}` using the `octocode-research` skill with the `reviewPR` prompt. Instruct it to return only the list of issues found — no review submission.

#### Subagent C: pr-review-toolkit

Spawn a subagent to review PR `${PR_NUMBER}` using the `pr-review-toolkit:review-pr` command with `all parallel` mode. Instruct it to return only the list of issues found — no review submission.

#### Subagent D: react-doctor

Spawn a subagent to review PR `${PR_NUMBER}` using the `react-doctor` skill. Instruct it to diagnose React-specific issues (performance, hooks misuse, component anti-patterns, security) in the changed files and return only the list of issues found — no review submission.

### 4. Merge and Deduplicate Issues

Collect issues from all four subagents and deduplicate:

1. Parse each subagent's response into individual issues.
2. For each issue, create a fingerprint from: `file path` + `line number range` + `concern category`.
3. Two issues are duplicates if they share the same file, overlapping line ranges (within ±5 lines), and address the same concern (use semantic comparison, not exact string matching).
4. When duplicates are found, keep the most detailed/actionable version.
5. Tag each kept issue with its source(s): `[code-reviewer]`, `[octocode]`, `[pr-review-toolkit]`, `[react-doctor]`, or combination if multiple agents found it.

### 5. Write Issues to Log File

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
  items.forEach(item => {
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
**Sources:** code-reviewer, octocode, pr-review-toolkit, react-doctor
````

### 6. React Health Report

Append the full output from the `react-doctor` subagent (Subagent D) to the log file under a dedicated section:

```markdown
## React Health

<full react-doctor report>
```

This section captures React-specific diagnostics (performance, hooks, component patterns, security) separately from the general deduplicated issues above.

### 7. Verify PR Implementation Coverage

After writing issues, spawn a subagent to verify PR `${PR_NUMBER}` using the `arinhub-verify-pr-implementation` skill. The subagent must return the full PR Coverage report in markdown format.

Append the returned coverage report to the end of the log file under a new section:

```markdown
## PR Coverage

<coverage report content from arinhub-verify-pr-implementation>
```

### 8. Submit PR Review

Spawn a subagent to submit the review for PR `${PR_NUMBER}` using the `arinhub-submit-pr-review` skill. Pass the log file path (`${LOG_FILE}`) so the subagent reads issues from it. The subagent must follow the `arinhub-submit-pr-review` procedure for deduplication against existing PR comments before submission.

### 9. Report to User

Present a summary:

- Path to the log file
- Total issues found (by severity)
- PR coverage percentage
- Whether the review was submitted successfully
- The PR URL for reference

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

- All four review subagents run in parallel to minimize total review time.
- The log file is the single source of truth — all findings are merged there before submission.
- Deduplication uses semantic comparison: if two agents flag the same concern on the same code, only one entry is kept.
- The log file persists at `~/.agents/logs/` for future reference and audit.
- If a subagent fails or times out, proceed with results from the remaining agents and note the failure in the log file.
