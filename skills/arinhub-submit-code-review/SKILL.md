---
name: arinhub-submit-code-review
description: 'Submit code review from chat session or log file to a PR or display it locally. Use when asked to "submit PR review", "submit review to PR #123", "submit code review", or "show code review". For remote PRs, creates line-specific review comments. For local changes, outputs findings to the terminal.'
argument-hint: "PR number or URL (e.g., 123, #456, https://github.com/owner/repo/pull/789), or omit for local changes"
---

# Submit Code Review

Submit a structured code review with line-specific comments to a GitHub pull request, or display findings locally when no PR is targeted. Identifies issues in the current chat session or log file, checks for duplicate comments (remote only), and submits the review or outputs it.

## Input

- **PR number or URL** (optional): The pull request identifier. Accepts:
  - Number: `123`
  - Hash-prefixed: `#123`
  - Full URL: `https://github.com/owner/repo/pull/123`
  - If omitted, operates in local mode and outputs findings to the terminal.

## Procedure

### 1. Determine Review Target

- **Remote PR**: If the user provides a PR number or URL (e.g., "Submit review to PR #123"), target that remote PR. Set `MODE=remote`.
- **Local Changes**: If no specific PR is mentioned, or if the user asks to "submit review" or "show review" without a PR reference, target the current local changes. Set `MODE=local`.

### 2. Resolve PR Identifier (only if `MODE=remote`)

Extract the PR number from the user input. Strip any `#` prefix or parse the number from a URL.

```
PR_NUMBER=<extracted number>
```

### 3. Fetch PR Metadata (only if `MODE=remote`)

Gather PR details:

```bash
gh pr view $PR_NUMBER --json number,title,body,baseRefName,headRefName,files,url
```

### 4. Fetch Existing Review Comments (only if `MODE=remote`)

Retrieve all existing review comments to prevent duplication:

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments --paginate --jq '.[] | {id, path, line, body, user: .user.login}'
```

Also fetch top-level review bodies:

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/reviews --paginate --jq '.[] | {id, body, state, user: .user.login}'
```

### 5. Get Issue List

Get a list of issues from one of these sources (in priority order):

1. **Log file**: If a log file path is provided (e.g., from `arinhub-code-reviewer` orchestrator), read the file and extract all issues from the `## Issues` section.
2. **Current chat session**: If no log file is specified, collect issues identified during the code review in the current chat session.

For each issue found, record:

- `path`: The relative file path
- `line`: The specific line number in the new version of the file (must be within the diff hunk). For multi-line issues, this is the **last** line of the range.
- `start_line` (optional): The first line of a multi-line range. Only set when the issue spans more than one line.
- `body`: A concise, actionable comment explaining the issue
- `suggestion` (optional): The replacement code that should replace the line(s) from `start_line` (or `line`) through `line`. Include this whenever you can propose a concrete fix. The suggestion content is the **exact code** that will replace the selected lines -- do not include ` ```suggestion ` fences here, they are added automatically in Step 8.

### 6. Deduplicate Comments

**If `MODE=remote`:** For each issue identified in Step 5, compare against existing comments from Step 4:

- **Skip** if an existing comment on the same `path` and `line` (or nearby range +/- 3 lines) already addresses the same concern
- **Skip** if the issue is already mentioned in any top-level review body
- Use semantic comparison, not exact string matching -- if the existing comment covers the same problem, even with different wording, skip the new comment

**If `MODE=local`:** Skip deduplication — there are no existing PR comments to compare against. All issues from Step 5 proceed.

### 7. Decision Gate

- If **no new issues** remain after deduplication (or no issues at all in local mode): **Do not submit a review.** Inform the user that no issues were found.
- If **new issues exist**: Proceed to Step 8 (remote) or Step 8a (local).

### 8. Submit the Review (only if `MODE=remote`)

Use the GitHub API to submit a review with inline comments:

Preflight validation checklist (run before submission):

- Validate JSON payload before sending (e.g., `jq . review.json >/dev/null`)
- Ensure each comment has valid `path`, `line`, and `side: "RIGHT"`
- For multi-line comments, include `start_line` and `start_side: "RIGHT"` together with `line`
- Confirm `line` (and `start_line` for ranges) is inside the PR diff hunk for that file
- If a suggestion is included, append it in `body` using ` ```suggestion ` fences (do not pre-wrap suggestion content in fences earlier)
- Ensure suggestion replacement code preserves indentation and exact intended final content
- Use an empty suggestion block (` ```suggestion\n``` `) only when the intent is to delete selected lines

````bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/reviews \
  --method POST \
  --input - <<'EOF'
{
  "event": "APPROVE or COMMENT",
  "body": "<two-sentence-summary>",
  "comments": [
    {
      "path": "<file-path>",
      "line": <line-number>,
      "side": "RIGHT",
      "body": "<comment-text>\n\n```suggestion\n<replacement-code>\n```"
    }
  ]
}
EOF
````

For **multi-line** suggestions, add `start_line` and `start_side`:

````json
{
  "path": "<file-path>",
  "start_line": <first-line>,
  "line": <last-line>,
  "start_side": "RIGHT",
  "side": "RIGHT",
  "body": "<comment-text>\n\n```suggestion\n<replacement-code>\n```"
}
````

If a comment has **no suggestion** (pure observation), omit the ` ```suggestion ``` ` block from the body. Still include `"side": "RIGHT"` so that all comments are anchored to the new version of the file.

Rules for the review body:

- Write exactly 2 sentences summarizing what was reviewed and key observations
- If no critical issues are found, start with `LGTM` and use `APPROVE` as the event type
- Do not use emojis anywhere in the body or comments

Rules for individual comments:

- Keep each comment concise and actionable
- Explain the "why" not just the "what"
- **Prefer suggested changes** over plain comments whenever a concrete fix can be proposed -- use the ` ```suggestion ``` ` block format
- The explanation text goes **before** the suggestion block in the `body`
- Each comment `line` must fall within a diff hunk for the given `path`
- `side` must be `"RIGHT"` (the new version of the file) for comments with suggestions
- If an issue is critical, use emoji prefix 🚨 in title of the comment body
- If an issue is improvement (not critical), use emoji prefix ✨ in title of the comment body
- If an issue is a nitpick (minor style or non-functional), use emoji prefix 📝 in title of the comment body
- Do not use emojis in the main review body -- only in individual comment titles

### 8a. Display Review Locally (only if `MODE=local`)

Instead of submitting to GitHub, present the findings directly to the user. For each issue, display:

- Severity emoji prefix (🚨 critical, ✨ improvement, 📝 nitpick)
- File path and line number
- Description of the issue
- Suggested fix (if available), formatted as a diff

Group issues by severity (critical first, then improvements, then nitpicks).

After displaying all issues, provide a summary:

- Total number of issues by severity
- Branch name and list of changed files reviewed

Skip to Step 11 (Report Result) after displaying.

### 9. Report Result

**If `MODE=remote`:**

After submission, confirm to the user:

- Number of review comments submitted
- The PR URL for reference
- Brief list of issues flagged

If no review was submitted (Step 7), explain that no new issues were found beyond existing review comments.

**If `MODE=local`:**

Confirm the review was displayed. Include:

- Total issues found (by severity)
- Branch name reviewed

### 10. Extract PR Coverage (only if `MODE=remote`)

**Skip this step if `MODE=local`.**

Look for a PR Coverage section in the same source used in Step 5:

1. **Log file**: If a log file was used, look for a `## PR Coverage` or `## Coverage` section and extract its full content.
2. **Current chat session**: If no log file was used, look for any PR Coverage report or coverage summary produced during the current chat session.

If no PR Coverage is found, skip to the end -- this step is optional.

### 11. Post PR Coverage Comment (only if `MODE=remote`)

**Skip this step if `MODE=local`.**

**This step runs only if PR Coverage was found in Step 10. It must be the very last action -- execute it after all other steps (including the review submission and result report) are complete.**

Post the coverage report as a standalone PR comment:

```bash
gh pr comment $PR_NUMBER --body "$(cat <<'EOF'
<coverage-content>
EOF
)"
```

- Use the PR Coverage content exactly as found -- do not modify, summarize, or reformat it
- This comment is independent of the review; post it even if no review was submitted in Step 7
- This must be the very last API call in the entire procedure to ensure the coverage comment appears at the bottom of the PR conversation

## Important Notes

- Use `APPROVE` when no critical issues are found, otherwise use `COMMENT`. Never use `REQUEST_CHANGES` unless the user explicitly asks
- The `line` field in review comments must reference a line that appears in the diff -- comments on unchanged lines will be rejected by the API
- For multi-line suggestions, use `start_line` and `line` together to define the range being replaced; both must be within the diff hunk
- An empty suggestion block (` ```suggestion\n``` `) means "delete these lines"
- The content inside ` ```suggestion ``` ` replaces the selected line(s) verbatim -- ensure correct indentation and formatting
- Never fabricate issues -- only flag genuine concerns backed by evidence in the code
- In `MODE=local`, steps 4, 6 (dedup), 8, 10, and 11 are skipped or simplified — the review is displayed directly in the terminal without any GitHub API interaction
