# ArinHub

Collection of AI agents, hooks, and skills.

## Agent Skills

| Skill | Description | Use when |
| ----- | ----------- | -------- |
| [`arinhub-code-reviewer`](skills/arinhub-code-reviewer/SKILL.md) | Orchestrate a comprehensive code review by launching parallel review subagents, deduplicating findings, and submitting the review. | `"review PR 123"`, `"code review PR #123"`, `"code review"`, `"review code changes"` |
| [`arinhub-submit-code-review`](skills/arinhub-submit-code-review/SKILL.md) | Submit code review from chat session or log file to a PR or display it locally. | `"submit PR review"`, `"submit review to PR #123"`, `"submit code review"`, `"show code review"` |
| [`arinhub-verify-requirements-coverage`](skills/arinhub-verify-requirements-coverage/SKILL.md) | Verify that a PR or local changes fully implement the requirements described in a linked GitHub issue. | `"verify PR coverage"`, `"check PR coverage"`, `"does PR implement the issue"`, `"is PR complete"`, `"verify my changes"` |
