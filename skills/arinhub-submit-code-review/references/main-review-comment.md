# Main Review Comment Template

The top-level review comment submitted with the `event` field (`APPROVE` or `COMMENT`).

## Format

```
<sentence-1: what was reviewed>. <sentence-2: key observations or outcome>.
```

## Rules

- Write exactly 2 sentences summarizing what was reviewed and key observations
- If no High Priority issues are found, start with `LGTM` and use `APPROVE` as the event type
- Do not use emojis in the main review comment

## Examples

### Approve (no High Priority issues)

```
LGTM -- reviewed the new caching layer across 4 files including cache invalidation logic. All paths handle TTL expiry correctly and edge cases are well covered.
```

### Comment (High Priority issues found)

```
Reviewed authentication middleware and token refresh logic across 3 files. Found a race condition in token refresh that can lead to dropped requests under concurrent access.
```
