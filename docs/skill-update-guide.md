# Skill Update Guide: Handle Outdated Comments

This guide shows how to update the `/dotnet-code-review` skill to properly handle outdated comments using `original_line`.

## Problem

When code changes are pushed, GitHub sets `line: null` on outdated comments. Multiple outdated comments on the same file all have `line: null`, making them indistinguishable. This causes duplicate comment creation.

## Solution

Update the skill's comment matching logic to use `original_line` when `line` is null.

---

## Required Changes to `/dotnet-code-review` Skill

### 1. Update "Comment Deduplication" Section

**Find this text:**
```
The file `./tmp/pr-comments.json` contains inline review comments from previous review iterations. This file has the following structure:

```json
[
  {
    "id": 123456,
    "path": "src/Example.cs",
    "line": 42,
    "body": "Previous comment text",
    "user": "bot-username[bot]"
  }
]
```
```

**Replace with:**
```
The file `./tmp/pr-comments.json` contains inline review comments from previous review iterations. This file has the following structure:

```json
[
  {
    "id": 123456,
    "path": "src/Example.cs",
    "line": 42,
    "original_line": 42,
    "position": 42,
    "body": "Previous comment text",
    "user": "bot-username[bot]"
  }
]
```

**Note:** When code changes are pushed, GitHub marks comments as "outdated" by setting `line` to `null`, but `original_line` is preserved. Use `original_line` for matching when `line` is null.
```

### 2. Update "Comment Matching Logic" Section

**Find this text:**
```
1. **Check for existing comment**: Search `./tmp/pr-comments.json` for a comment with:
   - Same `path` (file path)
   - Same `line` (line number)
   - `user` is a bot (ends with `[bot]`)
```

**Replace with:**
```
1. **Check for existing comment**: Search `./tmp/pr-comments.json` for a comment with:
   - Same `path` (file path)
   - Same `line` (line number) OR same `original_line` if `line` is null (outdated comment)
   - `user` is a bot (ends with `[bot]`)
```

### 3. Update "Example Workflow" Section

**Find this code block:**
```bash
# 1. Load existing comments
EXISTING=$(cat ./tmp/pr-comments.json)

# 2. For each issue, check if comment exists
COMMENT_ID=$(echo "$EXISTING" | jq -r --arg path "src/Example.cs" --arg line "42" \
  '.[] | select(.path == $path and .line == ($line | tonumber)) | .id')

# 3. Update or create
if [ -n "$COMMENT_ID" ] && [ "$COMMENT_ID" != "null" ]; then
  # UPDATE existing comment
  gh api -X PATCH "/repos/$REPO/pulls/comments/$COMMENT_ID" \
    -f body="Updated issue description"
else
  # CREATE new comment
  # Use MCP tool: mcp__github_inline_comment__create_inline_comment
fi
```

**Replace with:**
```bash
# 1. Load existing comments
EXISTING=$(cat ./tmp/pr-comments.json)

# 2. For each issue, check if comment exists (handle outdated comments)
COMMENT_ID=$(echo "$EXISTING" | jq -r \
  --arg path "src/Example.cs" \
  --arg line "42" \
  '.[] | select(
    .path == $path and
    (
      (.line != null and (.line | tostring) == $line) or
      (.line == null and (.original_line | tostring) == $line)
    )
  ) | .id' | head -1)

# 3. Update or create
if [ -n "$COMMENT_ID" ] && [ "$COMMENT_ID" != "null" ]; then
  # UPDATE existing comment
  gh api -X PATCH "/repos/$REPO/pulls/comments/$COMMENT_ID" \
    -f body="Updated issue description"
else
  # CREATE new comment
  # Use MCP tool: mcp__github_inline_comment__create_inline_comment
fi
```

### 4. Add Explanation After Example

**Add this text after the example workflow:**

```
**How the matching works:**

- **Current comments** (`line` is not null): Match on `path` + `line`
- **Outdated comments** (`line` is null): Match on `path` + `original_line`
- This prevents duplicate comments even after code changes

**Example:**
```json
// Original comment (before code change)
{"id": 123, "path": "foo.cs", "line": 42, "original_line": 42}

// After code change (comment becomes outdated)
{"id": 123, "path": "foo.cs", "line": null, "original_line": 42}

// Matching logic will find it using original_line when checking line 42
```
```

---

## Testing the Changes

After updating the skill:

1. Create a test PR
2. Run the review workflow (creates comments)
3. Push new code changes (comments become outdated)
4. Run the review workflow again
5. Verify: No duplicate comments are created

## Summary of Changes

| Section | Change |
|---------|--------|
| Comment Data Structure | Added `original_line` and `position` fields |
| Comment Matching Logic | Use `original_line` when `line` is null |
| Example Workflow | Updated jq query to handle both cases |
| Documentation | Added explanation of outdated comment handling |

## Expected Behavior

**Before fix:**
```
Run 1: Create comment at line 42
Code change pushed
Run 2: Comment has line=null, can't match, creates duplicate
```

**After fix:**
```
Run 1: Create comment at line 42
Code change pushed
Run 2: Comment has line=null but original_line=42, matches, updates existing comment ✅
```
