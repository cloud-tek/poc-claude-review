# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is a reusable GitHub Actions workflow repository that provides automated code review capabilities using Claude Code. It's designed to be called from other repositories to integrate AI-powered code review into their CI/CD pipelines.

## Workflow Architecture

### Main Workflow: [.github/workflows/claude-code-review.yml](.github/workflows/claude-code-review.yml)

The workflow executes in the following sequence:

1. **Checkout**: Full git history (`fetch-depth: 0`)
2. **GitHub App Token Generation**: Creates a token from App credentials
3. **Setup PR Data Directory**: Creates `./tmp` directory for storing review data
4. **Restore PR Comment Cache**: Restores cached comment data from previous runs using GitHub Actions cache
5. **Fetch Current Comments**: Retrieves bot review comments from GitHub API and merges with cached data
6. **Claude Code Review**: Runs with specific MCP tools and max 25 turns
7. **Save PR Comment Cache**: Persists comment data to cache for subsequent runs
8. **Quality Gate**: Checks `./tmp/review-results.json` for critical/high priority issues and fails build if found
9. **Summary Comment**: Posts/updates a sticky PR comment with review breakdown
10. **Artifacts**: Uploads review results and comment data with 30-day retention

### Key Files and Locations

- **Workflow Definition**: [.github/workflows/claude-code-review.yml](.github/workflows/claude-code-review.yml)
- **PR Comments Data**: `./tmp/pr-comments.json` (persisted via GitHub Actions cache, contains bot review comments with IDs for updates)
- **Review Results**: `./tmp/review-results.json` (runtime, contains structured review findings)
- **Review Summary**: `./tmp/review-summary.md` (runtime, generated markdown summary)
- **GitHub Comments Temp**: `./tmp/github-comments.json` (runtime, raw GitHub API response)
- **Merged Comments Temp**: `./tmp/merged-comments.json` (runtime, temporary merge result)

### Review Results JSON Structure

The `./tmp/review-results.json` file follows this structure:

```json
{
  "summary": "string - overall review summary",
  "has_critical_issues": boolean,
  "has_high_priority_issues": boolean,
  "critical_count": number,
  "high_priority_count": number,
  "recommendation_count": number,
  "minor_count": number
}
```

### Severity Levels

The workflow uses four severity levels that determine build status:

- **Critical**: Blocks merge, fails build (exit 1)
- **High Priority**: Blocks merge, fails build (exit 1)
- **Recommendations**: Non-blocking, build passes
- **Minor/Nitpicks**: Non-blocking, build passes

## Customization Points

When modifying the workflow behavior, these are the key configuration points:

### Line 144: Review Skill
Currently set to `/dotnet-code-review`. Change this to target different tech stacks.

### Line 147: Max Turns
Set to `--max-turns 25`. Increase for more thorough reviews, decrease for faster execution.

### Line 148: Allowed Tools
Configured MCP tools and Claude Code tools:
- `mcp__github*`: GitHub API operations
- `mcp__github_inline_comment__create_inline_comment`: Create inline PR comments
- `mcp__github_comment__update_claude_comment`: Update existing comments
- Standard tools: `Bash,Read,Grep,Glob,Edit,Write`

### Line 149: Claude Model
Currently `claude-sonnet-4-5-20250929`. Update for different model versions.

### Lines 52-54: Cache Configuration
Cache key pattern and restore-keys can be adjusted:
- `key`: Currently uses `pr-{number}-comments-{sha}` for commit-specific caching
- `restore-keys`: Fallback pattern `pr-{number}-comments-` finds most recent PR cache

## Duplicate Comment Prevention via Cache

The workflow implements duplicate comment prevention using GitHub Actions cache:

### Cache Strategy

**Cache Key Pattern**: `pr-{number}-comments-{sha}`
- Unique per PR and commit SHA
- Restore-keys allow fallback to previous commits: `pr-{number}-comments-`
- Cache persists across workflow runs for the same PR

### Comment Tracking Process

1. **Restore Cache** (step "Restore PR Comment Cache"):
   - Attempts to restore `./tmp/pr-comments.json` from cache
   - Uses SHA-based key for exact match, falls back to PR-based key pattern

2. **Fetch from GitHub API** (step "Fetch Current Review Comments"):
   - Retrieves all bot comments via `/repos/{repo}/pulls/{pr}/comments`
   - Filters for `user.type == "Bot"`
   - Saves to `./tmp/github-comments.json`

3. **Merge Data**:
   - If cache exists: Merges cached data with GitHub API data by comment ID
   - GitHub API data takes precedence (source of truth)
   - Handles deleted comments, manual edits, and external changes

4. **Claude Code Review**:
   - Reads `./tmp/pr-comments.json` before posting any comments
   - For each file:line, checks if comment exists
   - EXISTS → Updates using `mcp__github_comment__update_claude_comment`
   - NOT EXISTS → Creates using `mcp__github_inline_comment__create_inline_comment`
   - Adds new comments to JSON file for subsequent operations

5. **Save Cache** (step "Save PR Comment Cache"):
   - Persists updated `./tmp/pr-comments.json` to cache
   - Uses `if: always()` to ensure cache is saved even on failure

### Comment Data Structure

```json
[
  {
    "id": "comment_id",
    "path": "file/path.cs",
    "line": 42,
    "original_line": 42,
    "position": 42,
    "body": "comment text",
    "user": "bot-username",
    "created_at": "2024-01-15T10:30:00Z",
    "updated_at": "2024-01-15T14:20:00Z"
  }
]
```

**Field Descriptions:**
- `line`: Current line number in the PR diff (null if comment is outdated after code changes)
- `original_line`: Original line number where comment was posted (preserved even when outdated)
- `position`: Position in the diff (used by GitHub for rendering)

### Handling Outdated Comments

When code changes are pushed to a PR, GitHub marks inline comments as "outdated" if the lines they reference have changed:

**Before code change:**
```json
{
  "id": 12345,
  "path": "src/Example.cs",
  "line": 42,
  "original_line": 42,
  "body": "Fix this issue"
}
```

**After code change (comment becomes outdated):**
```json
{
  "id": 12345,
  "path": "src/Example.cs",
  "line": null,           // ❌ Set to null
  "original_line": 42,    // ✅ Preserved
  "body": "Fix this issue"
}
```

**Comment Matching Strategy:**

The workflow matches comments using this logic:
1. **For current comments** (`line` is not null): Match on `path` + `line`
2. **For outdated comments** (`line` is null): Match on `path` + `original_line`
3. This prevents creating duplicate comments for issues that were already flagged

**Why this matters:**
- Multiple outdated comments on the same file all have `line: null`
- Without `original_line`, they're indistinguishable
- Using `original_line` allows proper deduplication even after code changes

### Cache Lifecycle Example

```
Run 1 (SHA: abc123):
  Restore: ❌ No cache → Fetch: 0 comments → Review: Create 5 → Save: Cache with 5

Run 2 (SHA: def456):
  Restore: ✅ Previous cache (5) → Fetch: 5 from GitHub → Merge: 5 unique
  Review: Update 2, create 3 → Save: Cache with 8

Run 3 (SHA: def456, re-run):
  Restore: ✅ Exact match (8) → Fetch: 8 from GitHub → Merge: 8 unique
  Review: Update 8 → Save: Cache with 8
```

## Required Secrets

Calling repositories must provide four secrets:

- `TOKEN`: GitHub token with repo access
- `CLAUDE_CODE_OAUTH_TOKEN`: OAuth token from Anthropic
- `CLAUDE_CODE_ACTION_APP_ID`: GitHub App ID
- `CLAUDE_CODE_ACTION_PRIVATE_KEY`: GitHub App private key (PEM format)

## GitHub App Permissions

The GitHub App requires:

- **Contents**: Read
- **Pull requests**: Read & Write
- **Issues**: Read & Write

## Testing Workflow Changes

Since this is a reusable workflow repository, testing requires:

1. Push changes to a test branch
2. Update the calling workflow reference to use the test branch: `@test-branch-name`
3. Create/update a PR in the calling repository to trigger the workflow
4. Review workflow logs in the calling repository's Actions tab
5. Check for artifacts uploaded as `code-review-results-pr-{number}`

The workflow uses `continue-on-error: true` for the Claude Code Review step, so partial failures won't prevent summary generation.
