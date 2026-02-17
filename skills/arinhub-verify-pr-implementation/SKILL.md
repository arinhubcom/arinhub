---
name: arinhub-verify-pr-implementation
description: 'Verify that a pull request fully implements the requirements described in its linked GitHub issue. Use when asked to "verify PR implementation", "check PR coverage", "does PR implement the issue", "verify PR against issue", "is PR complete", or "PR completeness check". Extracts the linked issue from the PR body or GitHub linked issues, analyzes the diff against issue requirements, and reports either missing items or confirms 100% coverage.'
argument-hint: "PR number or URL (e.g., 123, #456, https://github.com/owner/repo/pull/789)"
---

# Verify PR Implementation

Verify that a pull request fully implements the requirements described in its linked GitHub issue. Extracts the issue reference, analyzes the PR diff against the issue description, and produces a coverage report.

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

Gather PR details including body and linked issues:

```bash
gh pr view $PR_NUMBER --json number,title,body,baseRefName,headRefName,files,url
```

### 3. Extract Linked Issue Number

Determine the related issue using these methods in priority order:

**Method A -- Closing keywords in PR body:**

Search the PR body for GitHub closing keywords followed by an issue reference:

- `closes #N`, `fixes #N`, `resolves #N` (and their variants: `close`, `closed`, `fix`, `fixed`, `resolve`, `resolved`)
- Full URL references: `closes https://github.com/owner/repo/issues/N`

**Method B -- GitHub linked issues API:**

```bash
gh api graphql \
  -F owner='{owner}' \
  -F repo='{repo}' \
  -F pr_number=$PR_NUMBER \
  -f query='
    query($owner: String!, $repo: String!, $pr_number: Int!) {
      repository(owner: $owner, name: $repo) {
        pullRequest(number: $pr_number) {
          closingIssuesReferences(first: 10) {
            nodes { number title }
          }
        }
      }
    }
  ' --jq '.data.repository.pullRequest.closingIssuesReferences.nodes'
```

**Method C -- Issue reference in PR body:**

If Methods A and B yield no results, scan the PR body for any `#N` pattern or issue URL that is not a closing keyword reference.

**No issue found:** If no linked issue can be determined, inform the user and stop. Do not guess or fabricate an issue number.

```
ISSUE_NUMBER=<extracted number>
```

If multiple issues are found, process each one and produce a separate coverage report per issue.

### 4. Fetch Issue Details

Retrieve the full issue description:

```bash
gh issue view $ISSUE_NUMBER --json number,title,body,labels
```

### 5. Extract Requirements from Issue

Parse the issue body and title to identify discrete, testable requirements. Look for:

- Checklist items (`- [ ]` or `- [x]`)
- Numbered steps or acceptance criteria
- Explicit behavioral descriptions ("should", "must", "when X then Y")
- UI changes, API changes, or data model changes mentioned
- Edge cases or error handling requirements
- Non-functional requirements (performance, security, accessibility)

Produce a numbered list of requirements:

```
R1: <requirement description>
R2: <requirement description>
...
```

If the issue body is vague or contains no clear requirements, use the issue title and any available context to infer the expected behavior. Flag inferred requirements clearly.

### 6. Fetch PR Diff

Get the full diff for the pull request:

```bash
gh pr diff $PR_NUMBER
```

Also review the list of changed files from Step 2 to understand the scope of changes.

### 7. Analyze Coverage

For each requirement from Step 5, determine whether the PR diff addresses it:

- **Covered**: The diff contains code changes that directly implement the requirement
- **Partially covered**: The diff addresses some aspects but misses edge cases or details
- **Not covered**: No code changes in the diff relate to this requirement

Use evidence from the diff to justify each assessment. Do not speculate -- base judgments on actual code changes.

### 8. Produce Report

Generate one of two outputs based on the analysis:

**If all requirements are covered (100% coverage):**

```
## PR Coverage: 100%

All requirements from issue #<ISSUE_NUMBER> are implemented in PR #<PR_NUMBER>.

### Requirements
| # | Requirement | Status |
|---|-------------|--------|
| R1 | <description> | Covered |
| R2 | <description> | Covered |

### Summary
<2-3 sentences confirming that the PR fully addresses the issue, highlighting key implementation decisions>
```

**If any requirements are missing or partially covered:**

```
## PR Coverage: <percentage>%

PR #<PR_NUMBER> does not fully implement issue #<ISSUE_NUMBER>.

### Requirements
| # | Requirement | Status | Notes |
|---|-------------|--------|-------|
| R1 | <description> | Covered | |
| R2 | <description> | Not covered | <what is missing> |
| R3 | <description> | Partially covered | <what is incomplete> |

### Missing Implementation
<For each uncovered or partially covered requirement, describe specifically what code changes are needed to complete the implementation>

### Summary
<2-3 sentences describing the overall gap between the issue requirements and the PR implementation>
```

### 9. Report to User

Present the coverage report from Step 8. Include:

- The issue number and title for context
- The PR URL for reference
- The coverage percentage
- Clear next steps if coverage is incomplete

## Important Notes

- Never fabricate requirements that are not present or implied in the issue
- If the issue lacks acceptance criteria, clearly state which requirements were inferred from context
- Do not evaluate code quality -- this skill only checks implementation completeness against the issue description
- For issues with sub-tasks or linked child issues, only evaluate the requirements in the specific linked issue
- When multiple issues are linked, report coverage for each issue separately
