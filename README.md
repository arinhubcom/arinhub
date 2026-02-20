# ArinHub

Collection of AI agents, hooks, and skills.

## Agent Skills

| Skill                                                                                          | Description                                                                                                                        | Use when                                                                                                                                                              |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`arinhub-code-reviewer`](skills/arinhub-code-reviewer/SKILL.md)                               | Orchestrate a comprehensive code review by launching parallel review subagents, deduplicating findings, and submitting the review. | `"review PR 123"`, `"review my changes"`, `"code review PR #123"`, `"review local changes"`, or omit input for local changes                                          |
| [`arinhub-submit-code-review`](skills/arinhub-submit-code-review/SKILL.md)                     | Submit code review from chat session or review file to a GitHub PR.                                                                | `"submit PR #123 review"`, `"submit review to PR 123"`, `"submit code review"`                                                                                        |
| [`arinhub-verify-requirements-coverage`](skills/arinhub-verify-requirements-coverage/SKILL.md) | Verify that a PR or local changes fully implement the requirements described in a linked GitHub issue.                             | `"verify PR 123"`, `"verify my changes"`, `"check PR #123 coverage"`, `"verify PR #123 against issue #42"`, `"check local changes"`, `"do my changes cover issue 42"` |

### How to Use `arinhub-code-reviewer`

#### Local Changes

```sh
/arinhub-code-reviewer
# or
review my changes
```

#### GitHub Pull Request

```sh
# navigate to the PR repository first
/arinhub-code-reviewer PR 123
# or
review PR 123
```

### `arinhub-code-reviewer` — Required Commands & Skills

The orchestrator launches parallel subagents that depend on external commands and skills:

| Subagent | Skill / Command                                                                                                                      | Required From                              |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ |
| A        | [`code-reviewer`](https://github.com/google-gemini/gemini-cli/blob/main/.gemini/skills/code-reviewer/SKILL.md)                       | skill                                      |
| B        | [`octocode-roast`](https://github.com/bgauryy/octocode-mcp/blob/main/skills/octocode-roast/SKILL.md)                                 | skill                                      |
| C        | [`pr-review-toolkit:review-pr`](https://github.com/anthropics/claude-code/blob/main/plugins/pr-review-toolkit/commands/review-pr.md) | command                                    |
| D        | [`react-doctor`](https://github.com/millionco/react-doctor/blob/main/skills/react-doctor/SKILL.md)                                   | skill (only when diff contains React code) |

Additionally, after the review phase:

| Step                | Skill                                                                                          | When                |
| ------------------- | ---------------------------------------------------------------------------------------------- | ------------------- |
| Verify requirements | [`arinhub-verify-requirements-coverage`](skills/arinhub-verify-requirements-coverage/SKILL.md) | remote PR and local |
| Submit review       | [`arinhub-submit-code-review`](skills/arinhub-submit-code-review/SKILL.md)                     | remote PR only      |

Install all required commands and skills:

```sh
claude plugin install pr-review-toolkit
npx skills add arinhubcom/arinhub -y -g -s arinhub-code-reviewer -s arinhub-submit-code-review -s arinhub-verify-requirements-coverage
npx skills add google-gemini/gemini-cli -y -g -s code-reviewer
npx skills add bgauryy/octocode-mcp -y -g -s octocode-roast
npx skills add millionco/react-doctor -y -g -s react-doctor
```

To update all installed skills to their latest versions:

```sh
npx skills update
```

> **Note:** `pr-review-toolkit` is an official Claude Code plugin. Official plugins have automatic updates enabled by default.
