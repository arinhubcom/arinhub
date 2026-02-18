---
name: arinhub-submit-pr-review
description: 'Submit pull request (PR) review from chat session to PR conversations. Use when asked to "submit PR review", "submit review to PR #123", or "review pull request". Identifies issues in the current chat session, creates line-specific review comments, avoids duplication with existing comments, and submits the review via GitHub CLI.'
argument-hint: "PR number or URL (e.g., 123, #456, https://github.com/owner/repo/pull/789)"
---

# Submit PR Review

Submit a structured code review with line-specific comments to a GitHub pull request. Identifies issues in the current chat session, checks for duplicate comments, and submits the review only if new issues are found.

## Input

- **PR number or URL** (required): The pull request identifier. Accepts:
  - Number: `123`
  - Hash-prefixed: `#123`
  - Full URL: `https://github.com/owner/repo/pull/123`

## Procedure

### 1. Resolve PR Identifier

Extract the PR number from the user input. Strip any `#` prefix or parse the number from a URL.

```
PR_NUMBER=<extracted number>
```

### 2. Fetch PR Metadata

Gather PR details:

```bash
gh pr view $PR_NUMBER --json number,title,body,baseRefName,headRefName,files,url
```

### 3. Fetch Existing Review Comments

Retrieve all existing review comments to prevent duplication:

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/comments --paginate --jq '.[] | {id, path, line, body, user: .user.login}'
```

Also fetch top-level review bodies:

```bash
gh api repos/{owner}/{repo}/pulls/$PR_NUMBER/reviews --paginate --jq '.[] | {id, body, state, user: .user.login}'
```

### 4. Get Issue List

Get a list of issues identified during the code review in the current chat session.

For each issue found, record:

- `path`: The relative file path
- `line`: The specific line number in the new version of the file (must be within the diff hunk). For multi-line issues, this is the **last** line of the range.
- `start_line` (optional): The first line of a multi-line range. Only set when the issue spans more than one line.
- `body`: A concise, actionable comment explaining the issue
- `suggestion` (optional): The replacement code that should replace the line(s) from `start_line` (or `line`) through `line`. Include this whenever you can propose a concrete fix. The suggestion content is the **exact code** that will replace the selected lines -- do not include ` ```suggestion ` fences here, they are added automatically in Step 7.

### 5. Deduplicate Comments

For each issue identified in Step 4, compare against existing comments from Step 3:

- **Skip** if an existing comment on the same `path` and `line` (or nearby range +/- 3 lines) already addresses the same concern
- **Skip** if the issue is already mentioned in any top-level review body
- Use semantic comparison, not exact string matching -- if the existing comment covers the same problem, even with different wording, skip the new comment

### 6. Decision Gate

- If **no new issues** remain after deduplication: **Do not submit a review.** Inform the user that no new issues were found.
- If **new issues exist**: Proceed to Step 7.

### 7. Submit the Review

Use the GitHub API to submit a review with inline comments:

```bash
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
```

For **multi-line** suggestions, add `start_line` and `start_side`:

```json
{
  "path": "<file-path>",
  "start_line": <first-line>,
  "line": <last-line>,
  "start_side": "RIGHT",
  "side": "RIGHT",
  "body": "<comment-text>\n\n```suggestion\n<replacement-code>\n```"
}
```

If a comment has **no suggestion** (pure observation), omit the ` ```suggestion ``` ` block from the body and the `side` field.

Rules for the review body:

- Write exactly 2 sentences summarizing what was reviewed and key observations
- If no critical issues are found, start with `LGTM` and use `APPROVE` as the event type
- If critical issues are found, start with a brief statement of the main concern and use `COMMENT` as the event type
- Do not use emojis anywhere in the body or comments

Rules for individual comments:

- Keep each comment concise and actionable
- Explain the "why" not just the "what"
- **Prefer suggested changes** over plain comments whenever a concrete fix can be proposed -- use the ` ```suggestion ``` ` block format
- The explanation text goes **before** the suggestion block in the `body`
- Do not use emojis in any comment text
- Each comment `line` must fall within a diff hunk for the given `path`
- `side` must be `"RIGHT"` (the new version of the file) for comments with suggestions

### 8. Report Result

After submission, confirm to the user:

- Number of review comments submitted
- The PR URL for reference
- Brief list of issues flagged

If no review was submitted (Step 6), explain that no new issues were found beyond existing review comments.

## Important Notes

- Use `APPROVE` when no critical issues are found, otherwise use `COMMENT`. Never use `REQUEST_CHANGES` unless the user explicitly asks
- The `line` field in review comments must reference a line that appears in the diff -- comments on unchanged lines will be rejected by the API
- For multi-line suggestions, use `start_line` and `line` together to define the range being replaced; both must be within the diff hunk
- An empty suggestion block (` ```suggestion\n``` `) means "delete these lines"
- The content inside ` ```suggestion ``` ` replaces the selected line(s) verbatim -- ensure correct indentation and formatting
- Never fabricate issues -- only flag genuine concerns backed by evidence in the code
