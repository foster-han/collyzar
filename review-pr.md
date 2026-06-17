---
description: Review the PR for current feature branch
allowed-tools: Bash, Read, Glob, Grep, Write, Task, WebFetch
---


You are reviewing a Pull Request for the current feature branch. Follow these steps:
## Step 0: Identify the PR From Github ee
First, get the current branch and find the matching open PR:
```bash
# Get current branch name
BRANCH=$(git branch --show-current)
echo "Current branch: $BRANCH"

# Find the open PR for this branch
gh pr list --head "$BRANCH" --state open --json number,title,url,body

# Get detailed PR information
gh pr view --json number,title,url,body,commits,baseRefName
```
Extract:
- PR number and title
- The Linear ticket ID (pattern: JOLLI-XXX) from title or commits
- Base branch the PR targets
## Step 1: Understand What Changed
Gather commit and diff information:
```bash
# Get commits in this PR
gh pr view --json commits --jq '.commits[] | "\(.oid[0:7]) \(.messageHeadline)"'

# Get the full diff summary
gh pr diff --stat
```
Analyze and provide:
- A summary of what this feature/fix does based on commits
- List of all modified, added, and deleted files
- The Linear ticket number (JOLLI-XXX)
## Step 2: Check for UI Changes
Look for frontend changes by examining:
1. Files in `frontend/src/ui/` or similar UI directories
2. Component files (`.tsx`, `.jsx`)
3. CSS/styling changes (`.css`, `.module.css`, tailwind classes)
4. i18n/localization changes (`.content.ts` files)

For any UI changes found:
- Describe what UI elements were added/modified
- Provide step-by-step instructions on how to test the UI changes manually
- List any new user interactions or workflows
- Note any visual/styling changes

If no UI changes, state "No UI changes in this PR."
## Step 3: Find Main Call Stack / Flow
For the core feature being implemented:
1. Identify the entry point (API endpoint, UI component, CLI command)
2. Trace the call flow through:
  - Router/Controller layer
  - Service/Business logic layer
  - DAO/Database layer
3. Create a visual representation of the call stack

Use this format:
```
[Entry Point] Component/Route
    -> [Handler] Function/Method
        -> [Service] Business Logic
            -> [DAO] Database Operation
                -> [Model] Database Table
```
## Step 4: Code Quality Assessment
Review the code for:
- Adherence to project conventions (check CLAUDE.md)
- Test coverage for new/modified code
- Security considerations
- Performance implications
- Error handling
## Step 5: Generate Review Document
Save the review document to:
`~/Documents/CodeReviews/review-PR{number}-{sanitized-pr-title}-{date}.md`

For the filename:
- Sanitize the PR title: lowercase, replace spaces with hyphens, remove special characters
- Keep it reasonably short (truncate to ~50 chars if needed)
- Example: PR #230 "Closes JOLLI-331: Add a github workflow for tenant+org schema migrations"
-> `review-PR230-closes-jolli-331-add-github-workflow-2026-01-09.md`

The document MUST have this frontmatter format:
```markdown
---
title: "PR #{number}: {PR title}"
pr_number: {number}
pr_url: {GitHub PR URL}
branch: {branch name}
linear_ticket: {JOLLI-XXX or "N/A"}
reviewed_date: {YYYY-MM-DD}
commits_summary: |
  - {commit 1 short hash}: {commit 1 message}
  - {commit 2 short hash}: {commit 2 message}
  ...
---
```
After the frontmatter, include:
1. **Summary of Changes** - What this PR does
2. **UI Changes & Testing Instructions** - How to manually test UI
3. **Call Stack / Architecture Flow** - Code flow diagram
4. **Code Quality Notes** - Observations and concerns
5. **Recommendations** - Action items or suggestions

Format the date as YYYY-MM-DD