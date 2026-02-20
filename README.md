# ArinHub

Collection of AI agents, hooks, and skills.

## Agent Skills

| Skill                                                                                          | Description                                                                                                                        | Use when                                                                                                                  |
| ---------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------- |
| [`arinhub-code-reviewer`](skills/arinhub-code-reviewer/SKILL.md)                               | Orchestrate a comprehensive code review by launching parallel review subagents, deduplicating findings, and submitting the review. | `"review PR 123"`, `"code review PR #123"`, `"code review"`, `"review code changes"`                                      |
| [`arinhub-submit-code-review`](skills/arinhub-submit-code-review/SKILL.md)                     | Submit code review from chat session or log file to a PR or display it locally.                                                    | `"submit PR review"`, `"submit review to PR #123"`, `"submit code review"`, `"show code review"`                          |
| [`arinhub-verify-requirements-coverage`](skills/arinhub-verify-requirements-coverage/SKILL.md) | Verify that a PR or local changes fully implement the requirements described in a linked GitHub issue.                             | `"verify PR coverage"`, `"check PR coverage"`, `"does PR implement the issue"`, `"is PR complete"`, `"verify my changes"` |

### `arinhub-code-reviewer` — Required Skills & Commands

The orchestrator launches parallel subagents that depend on external commands and skills:

| Subagent | Skill / Command                                                                                                                      | Required From                              |
| -------- | ------------------------------------------------------------------------------------------------------------------------------------ | ------------------------------------------ |
| A        | [`code-reviewer`](https://github.com/google-gemini/gemini-cli/blob/main/.gemini/skills/code-reviewer/SKILL.md)                       | skill                                      |
| B        | [`octocode-roast`](https://github.com/bgauryy/octocode-mcp/blob/main/skills/octocode-roast/SKILL.md)                                 | skill                                      |
| C        | [`pr-review-toolkit:review-pr`](https://github.com/anthropics/claude-code/blob/main/plugins/pr-review-toolkit/commands/review-pr.md) | command                                    |
| D        | [`react-doctor`](https://github.com/millionco/react-doctor/blob/main/skills/react-doctor/SKILL.md)                                   | skill (only when diff contains React code) |

Additionally, after the review phase:

| Step                | Skill                                                                                          | When           |
| ------------------- | ---------------------------------------------------------------------------------------------- | -------------- |
| Verify requirements | [`arinhub-verify-requirements-coverage`](skills/arinhub-verify-requirements-coverage/SKILL.md) | remote PR only |
| Submit review       | [`arinhub-submit-code-review`](skills/arinhub-submit-code-review/SKILL.md)                     | remote PR only |

Install all required commands and skills:

```sh
claude plugin install pr-review-toolkit
npx skills add arinhubcom/arinhub -y -g -s arinhub-code-reviewer -s arinhub-submit-code-review -s arinhub-verify-requirements-coverage
npx skills add google-gemini/gemini-cli -y -g -s code-reviewer
npx skills add bgauryy/octocode-mcp -y -g -s octocode-roast
npx skills add millionco/react-doctor -y -g -s react-doctor
```
