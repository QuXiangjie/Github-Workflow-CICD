# GitHub Project Maintenance Workflow

A complete, step-by-step reference for maintaining a GitHub project — covering both bug fixes and new features using **GitHub Flow**.

---

## Prerequisites: One-Time Repo Setup

Before any work begins, configure these once per repository.

### Branch Protection via Rulesets (Settings → Rules → Rulesets)

> **Note:** GitHub has migrated the old "Branch protection rules" (Settings → Branches) to the newer **Rulesets** system. You will NOT find a "Branches" menu item in the sidebar. Instead, look for **Rules** under "Code and automation."

#### Step 1 — Navigate to Rulesets

1. Go to your repo → **Settings**
2. In the left sidebar, under **Code and automation**, click **Rules** to expand it
3. Click **Rulesets**
4. Click **New ruleset** → **New branch ruleset**

#### Step 2 — Basic Configuration

| Field | Value |
|-------|-------|
| Ruleset name | `main-protection` |
| Enforcement status | **Active** |
| Bypass list | Leave empty (so even admins cannot bypass) |

#### Step 3 — Set Target Branches

1. Under **Target branches**, click **Add target**
2. Select **Include default branch** — this targets `main`
3. You can also add patterns like `release/*` if needed later

#### Step 4 — Enable Rules

Check the following rules:

**Require a pull request before merging**

- Set **Required approvals** = 1 (or more for larger teams)
- This ensures no one can push directly to `main` — all changes must go through a PR with at least 1 reviewer approval

**Require status checks to pass**

- Click **+ Add checks** and type your CI job name (e.g., `test`) — this name must match the `jobs:` key in your `.github/workflows/ci.yml`
- If you have multiple jobs (e.g., `lint`, `build`), add them all
- Expand **Additional settings** and enable:
  - ✅ **Require branches to be up to date before merging** — forces the PR branch to include the latest `main` commits before CI runs, so you're testing the actual code that will land after merge (recommended for solo/small teams; can cause "queue effects" on large teams)
  - Leave "Do not require status checks on creation" unchecked unless you need to create branches before CI is configured

**Block force pushes**

- Prevents `git push --force` to `main`, which could rewrite history and break things

**Require linear history** (optional)

- Forces squash or rebase merges only — no merge commits
- Results in a cleaner `git log` on `main`

#### Step 5 — Save

Click **Create** at the bottom to activate the ruleset.

#### Verification

After saving, try pushing directly to `main`:

```bash
git checkout main
echo "test" >> README.md
git add . && git commit -m "test direct push"
git push origin main
# Expected: rejected by branch protection
```

GitHub should reject the push with an error like:
`remote: error: GH006: Protected branch update failed`

This confirms your ruleset is working — all changes must now go through a PR.

### GitHub Actions CI Pipeline

Create `.github/workflows/ci.yml`:

```yaml
name: CI
on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20
      - run: npm ci
      - run: npm run lint
      - run: npm test
      - run: npm run build
```

### Issue and PR Templates

Create `.github/ISSUE_TEMPLATE/bug_report.md`:

```markdown
---
name: Bug Report
about: Report a bug to help us improve
labels: bug
---

**Describe the bug**
A clear description of what the bug is.

**Steps to reproduce**
1. Go to '...'
2. Click on '...'
3. See error

**Expected behavior**
What you expected to happen.

**Actual behavior**
What actually happened.

**Environment**
- OS: [e.g. macOS 15]
- Browser: [e.g. Chrome 131]
- Version: [e.g. v1.3.0]
```

Create `.github/ISSUE_TEMPLATE/feature_request.md`:

```markdown
---
name: Feature Request
about: Suggest a new feature
labels: enhancement
---

**Problem statement**
What problem does this solve?

**Proposed solution**
Describe the feature you'd like.

**Alternatives considered**
Other approaches you've thought about.
```

Create `.github/pull_request_template.md`:

```markdown
## Summary
Brief description of changes.

## Related Issue
Closes #<issue_number>

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Checklist
- [ ] Tests added/updated
- [ ] Linting passes
- [ ] Documentation updated (if applicable)
```

### Dependabot (Automated Dependency Updates)

Create `.github/dependabot.yml`:

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 5
```

---

## Workflow A: Fixing a Bug

### Step 1 — Issue Triage

A user (or you) files an issue using the bug report template.

**You triage by:**

- Confirming the bug is reproducible
- Adding labels: `bug`, `priority: high/medium/low`
- Assigning the issue to yourself (or a team member)
- Adding to a milestone (e.g., `v1.3.1`)

```
Example issue: #42 — "Search crashes on special characters like & and %"
Labels: bug, priority: high
Milestone: v1.3.1
Assignee: you
```

### Step 2 — Create a Branch

```bash
# Always start from an up-to-date main
git checkout main
git pull origin main

# Create a branch — naming convention: fix/<issue#>-<short-description>
git checkout -b fix/42-search-special-chars
```

**Branch naming conventions:**

| Type | Pattern | Example |
|------|---------|---------|
| Bug fix | `fix/<issue#>-<desc>` | `fix/42-search-special-chars` |
| Feature | `feat/<issue#>-<desc>` | `feat/55-dark-mode` |
| Hotfix | `hotfix/<issue#>-<desc>` | `hotfix/99-auth-bypass` |
| Docs | `docs/<desc>` | `docs/api-usage-guide` |
| Chore | `chore/<desc>` | `chore/upgrade-deps` |

> **Note:** The branch name is a human convention only — it does NOT automatically link to the issue. The link happens via commit messages and PR descriptions (see Steps 3 and 4).

### Step 3 — Write the Fix and Commit

```bash
# Make your code changes...
# Write or update tests to cover the fix...

# Stage the changes
git add src/search.js tests/search.test.js

# Commit with a conventional commit message + closing keyword
git commit -m "fix: encode special chars in search query

Fixes #42 — wraps user input in encodeURIComponent()
before passing to the search API endpoint.

Added unit tests for &, %, +, and # characters."
```

**Closing keywords that auto-close issues on merge:**

`close`, `closes`, `closed`, `fix`, `fixes`, `fixed`, `resolve`, `resolves`, `resolved`

All are case-insensitive. They work in both commit messages and PR descriptions. The auto-close only triggers when the PR merges into the **default branch** (usually `main`).

**Conventional Commits format:**

```
<type>: <short summary>

<optional body with details>

<optional footer with issue references>
```

Common types: `fix`, `feat`, `docs`, `chore`, `refactor`, `test`, `style`, `perf`, `ci`

### Step 4 — Push and Open a Pull Request

```bash
git push -u origin fix/42-search-special-chars
```

Then open a PR on GitHub (or use `gh pr create` with the GitHub CLI):

```bash
# Using GitHub CLI
gh pr create \
  --title "fix: encode special chars in search query" \
  --body "## Summary
Wraps user input in encodeURIComponent() before the API call.

## Related Issue
Closes #42

## Type of Change
- [x] Bug fix

## Checklist
- [x] Tests added/updated
- [x] Linting passes" \
  --base main
```

**What this does:**

- Creates the PR: `fix/42-search-special-chars → main`
- `Closes #42` in the PR body links issue #42 — it will auto-close on merge
- Triggers CI (GitHub Actions) automatically
- Requests review per branch protection rules

### Step 5 — CI Runs Automatically

GitHub Actions runs your CI workflow. The PR page shows check status:

```
✅ Lint (ESLint)        — passed in 12s
✅ Unit tests (Jest)    — passed in 38s
✅ Build (vite build)   — passed in 21s
```

If any check fails, push a fix to the same branch — CI re-runs automatically:

```bash
# Fix the failing test, then:
git add .
git commit -m "fix: handle edge case in test"
git push
```

### Step 6 — Code Review

A teammate reviews the PR on GitHub:

- **Approve** — code looks good, tests pass, no concerns
- **Request changes** — something needs to be fixed before merge
- **Comment** — leave feedback without formal approval/rejection

You address feedback by pushing additional commits to the same branch. Each push updates the PR and re-triggers CI.

### Step 7 — Merge

Once CI passes and you have the required approvals:

```
Option 1: "Squash and merge" — combines all commits into one clean commit (recommended for bug fixes)
Option 2: "Merge commit" — preserves all individual commits
Option 3: "Rebase and merge" — replays commits on top of main
```

After merge:

- Issue #42 is auto-closed (because of `Closes #42`)
- The feature branch is deleted on GitHub (enable "Automatically delete head branches" in repo settings)

Clean up locally:

```bash
git checkout main
git pull origin main
git branch -d fix/42-search-special-chars
```

---

## Workflow B: Adding a New Feature

### Step 1 — Issue or Discussion

A feature request is filed (or you create one). For larger features, consider opening a **GitHub Discussion** first to gather feedback before committing to an issue.

```
Example issue: #55 — "Add dark mode support"
Labels: enhancement, priority: medium
Milestone: v1.4.0
Assignee: you
```

For significant features, add a task checklist to the issue:

```markdown
## Tasks
- [ ] Design dark mode color palette
- [ ] Add theme toggle component
- [ ] Implement CSS custom properties for theming
- [ ] Persist user preference in localStorage
- [ ] Update documentation
```

### Step 2 — Create a Branch

```bash
git checkout main
git pull origin main
git checkout -b feat/55-dark-mode
```

### Step 3 — Develop Iteratively

For features that take multiple days, commit frequently in logical chunks:

```bash
# First commit: the foundation
git add src/theme/colors.js src/theme/provider.jsx
git commit -m "feat: add theme color definitions and context provider

Part of #55 — defines light/dark color tokens and
React context for theme state management."

# Second commit: the UI component
git add src/components/ThemeToggle.jsx src/components/ThemeToggle.test.js
git commit -m "feat: add theme toggle component

Part of #55 — toggle switch in the header that flips
between light and dark themes."

# Third commit: integration
git add src/styles/global.css src/App.jsx
git commit -m "feat: wire up CSS custom properties for dark mode

Closes #55 — all components now respond to theme context.
User preference persisted in localStorage."
```

**Keeping your branch up to date** (if main has moved ahead):

```bash
# Option A: Rebase (cleaner history, preferred for feature branches)
git fetch origin
git rebase origin/main

# If there are conflicts:
# 1. Fix the conflicting files
# 2. git add <fixed-files>
# 3. git rebase --continue

# Option B: Merge main into your branch (simpler but creates merge commits)
git fetch origin
git merge origin/main
```

### Step 4 — Push and Open a Draft PR Early

For longer features, open a **Draft PR** early so teammates can see progress:

```bash
git push -u origin feat/55-dark-mode

# Open as draft using GitHub CLI
gh pr create \
  --title "feat: add dark mode support" \
  --body "## Summary
Adds system-wide dark mode with user toggle.

## Related Issue
Closes #55

## Status
🚧 Work in progress — feedback welcome on the approach.

## Type of Change
- [x] New feature

## Checklist
- [x] Theme color definitions
- [x] Toggle component
- [ ] CSS custom properties integration
- [ ] Tests
- [ ] Documentation" \
  --draft \
  --base main
```

When the feature is ready for review, convert the Draft PR to "Ready for review" on GitHub (or via CLI: `gh pr ready`).

### Step 5–7 — CI, Review, and Merge

Same as the bug fix workflow. For features:

- Use **"Squash and merge"** to create one clean commit if the individual commits aren't meaningful on their own
- Use **"Merge commit"** or **"Rebase and merge"** if each commit tells a useful story

---

## Workflow C: Release

After accumulating several fixes and features, cut a release.

### Step 1 — Tag the Release

```bash
git checkout main
git pull origin main

# Create an annotated tag
git tag -a v1.4.0 -m "Release v1.4.0: dark mode, search fix, dependency updates"
git push origin v1.4.0
```

### Step 2 — Create a GitHub Release

Using the GitHub CLI:

```bash
gh release create v1.4.0 \
  --title "v1.4.0" \
  --notes "## What's New
- feat: dark mode support (#55)

## Bug Fixes
- fix: encode special chars in search (#42)

## Maintenance
- chore: bump axios to 1.7.2 (Dependabot)
- docs: update API usage guide"
```

Or do it in the GitHub UI: Releases → Draft a new release → Choose the tag → Write release notes → Publish.

### Step 3 — (Optional) Auto-Deploy on Release

Add a deploy workflow in `.github/workflows/deploy.yml`:

```yaml
name: Deploy
on:
  release:
    types: [published]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run build
      - name: Deploy to production
        run: |
          # Your deployment commands here
          echo "Deploying ${{ github.ref_name }}..."
```

### Semantic Versioning Quick Reference

| Version Bump | When | Example |
|-------------|------|---------|
| **PATCH** (x.x.1) | Bug fixes, no API changes | `v1.3.0 → v1.3.1` |
| **MINOR** (x.1.0) | New features, backward compatible | `v1.3.1 → v1.4.0` |
| **MAJOR** (1.0.0) | Breaking changes | `v1.4.0 → v2.0.0` |

---

## Workflow D: Handling External Contributions

When someone outside your team opens a PR:

### Step 1 — Review the PR

```bash
# Check out their branch locally for testing
gh pr checkout 78

# Or manually:
git fetch origin pull/78/head:pr-78
git checkout pr-78

# Run tests locally
npm test
```

### Step 2 — Provide Feedback

- Be kind and specific in review comments
- If changes are needed, select "Request changes" (not just "Comment")
- For first-time contributors, check that they've signed a CLA (if applicable)

### Step 3 — Merge or Close

- If the PR is good: approve and merge
- If the PR needs work: request changes and wait for updates
- If the PR is not aligned with project direction: explain why and close it politely

---

## Ongoing Maintenance Checklist

### Weekly

- Review and merge Dependabot PRs (dependency updates)
- Triage new issues (label, assign, prioritize)
- Respond to open PRs from contributors

### Monthly

- Close stale issues (no activity for 30+ days, not actionable)
- Review and update project milestones
- Check CI pipeline for flaky tests or slow jobs

### Per Release

- Update `CHANGELOG.md`
- Update `README.md` if features/APIs changed
- Tag and publish a GitHub Release with notes
- Deploy (if not auto-deployed)

---

## Quick Command Reference

```bash
# === Daily workflow ===
git checkout main && git pull origin main     # Sync
git checkout -b fix/42-description            # Branch
git add . && git commit -m "fix: message"     # Commit
git push -u origin fix/42-description         # Push
gh pr create --base main                      # PR

# === Keep branch current ===
git fetch origin && git rebase origin/main    # Rebase

# === After merge ===
git checkout main && git pull origin main     # Sync
git branch -d fix/42-description              # Clean up

# === Release ===
git tag -a v1.4.0 -m "Release v1.4.0"        # Tag
git push origin v1.4.0                        # Push tag
gh release create v1.4.0 --notes "..."        # Release

# === Useful GitHub CLI commands ===
gh issue list                                 # List issues
gh issue create                               # Create issue
gh pr list                                    # List PRs
gh pr status                                  # Your PR status
gh pr checks 43                               # CI status for PR
gh pr merge 43 --squash                       # Squash merge
```

---

## Summary: The Complete Cycle

```
Issue filed
  ↓
Triage (label, assign, milestone)
  ↓
Branch from main (fix/42-desc or feat/55-desc)
  ↓
Code + test + commit (with "Fixes #42" or "Closes #55")
  ↓
Push → Open PR (or Draft PR for features)
  ↓
CI runs automatically (lint, test, build)
  ↓
Code review (approve / request changes)
  ↓
Merge into main (squash / merge / rebase)
  ↓
Issue auto-closes ← linked via closing keyword
  ↓
Branch auto-deleted
  ↓
Repeat for next issue...
  ↓
Tag + Release when ready (v1.x.x)
  ↓
Deploy (manual or auto on release)
```
