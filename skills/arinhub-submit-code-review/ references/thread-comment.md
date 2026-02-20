# Thread Comment Template

Each inline thread comment posted on a specific line or line range in the PR diff.

## Format

### With suggestion

````md
<emoji> [<Severity>] <title>

<explanation-why>

```suggestion
<replacement-code>
```
````

### Without suggestion (pure observation)

```md
<emoji> [<Severity>] <title>

<explanation-why>
```

## Severity Prefixes

| Severity        | Emoji | Usage                                           |
| --------------- | ----- | ----------------------------------------------- |
| High Priority   | 🚨    | Bugs, security issues, data loss risks          |
| Medium Priority | ✨    | Non High Priority enhancements, better patterns |
| Low Priority    | 📝    | Minor style, formatting, non-functional         |

## Rules

- Keep each thread comment concise and actionable
- Explain the "why" not just the "what"
- Prefer suggested changes over plain comments whenever a concrete fix can be proposed
- The explanation text goes **before** the suggestion block in the `body`
- Each comment `line` must fall within a diff hunk for the given `path`
- `side` must be `"RIGHT"` (the new version of the file) for comments with suggestions
- Do not use emojis anywhere except the title prefix

## Examples

### High Priority -- with suggestion

````md
🚨 [High Priority] Unvalidated user input passed to SQL query

The `userId` parameter is interpolated directly into the query string, which is vulnerable to SQL injection. Use a parameterized query instead.

```suggestion
const result = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
```
````

### Medium Priority -- with suggestion

````md
✨ [Medium Priority] Simplify conditional with optional chaining

The nested null checks can be replaced with optional chaining for better readability.

```suggestion
const name = user?.profile?.displayName ?? 'Anonymous';
```
````

### Low Priority -- without suggestion

```md
📝 [Low Priority] Consider extracting magic number to a named constant

The timeout value `3000` appears in multiple places. A named constant like `DEFAULT_TIMEOUT_MS` would make the intent clearer and centralize future changes.
```
