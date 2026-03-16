---
name: git-workflow
description: Git commit conventions and branch strategy rules. Use when committing
  code or managing branches.
---

# Git Workflow Rules

## 1. Commit Message Convention

### Format

```text
<type>(<scope>): <subject>

[optional body]

[optional footer]
```

### Types

| Type       | Description                         |
| ---------- | ----------------------------------- |
| `feat`     | New feature                         |
| `fix`      | Bug fix                             |
| `docs`     | Documentation changes               |
| `style`    | Code formatting (no logic change)   |
| `refactor` | Code refactoring                    |
| `test`     | Adding/updating tests               |
| `chore`    | Build/config changes                |
| `perf`     | Performance improvement             |
| `ci`       | CI/CD configuration changes         |
| `revert`   | Revert previous commit              |

### Writing Rules

- Subject line under 50 characters
- Use imperative mood (Add, Fix, Update, etc.)
- No period at end of subject
- Wrap body at 72 characters
- Explain "what" and "why" in body, not "how"

### Example

```text
feat(auth): Add JWT token refresh logic

Previously, users had to re-login when token expired.
Added refresh token mechanism for automatic renewal.

Closes #123
```

---

## 2. Branch Strategy

### Branch Structure

| Branch        | Purpose                | Protected |
| ------------- | ---------------------- | --------- |
| `main`        | Production deployment  | Yes       |
| `develop`     | Development integration| Yes       |
| `feature/*`   | Feature development    | No        |
| `release/*`   | Release preparation    | No        |
| `hotfix/*`    | Production hotfix      | No        |
| `bugfix/*`    | Bug fix (from develop) | No        |

### Branch Naming

```text
feature/<issue-number>-<short-description>   # e.g., feature/123-add-login
bugfix/<issue-number>-<short-description>    # e.g., bugfix/456-fix-auth-error
hotfix/<issue-number>-<short-description>    # e.g., hotfix/789-patch-crash
release/<version>                            # e.g., release/1.2.0
```

### Merge Strategy

- `feature` → `develop`: Squash and merge
- `release` → `main`: Merge commit
- `hotfix` → `main` + `develop`: Merge commit

---

## 3. Pull Request Guidelines

### PR Title

Same format as commit message: `<type>(<scope>): <subject>`

### PR Description Template

```markdown
## Summary

- Brief description of changes

## Changes

- Specific changes made

## Test Plan

- [ ] Test item 1
- [ ] Test item 2

## Screenshots (if UI changes)

## Related Issues

Closes #<issue-number>
```

### Pre-Review Checklist

- [ ] Commit message convention followed
- [ ] Unnecessary files/comments removed
- [ ] Conflicts resolved
- [ ] CI passed

---

## 4. Commit Practices

### Atomic Commits

- One commit = one logical change
- Do not mix "feature + bugfix + refactor" in single commit

### Commit Split Example

```text
# Bad
git commit -m "Add login feature and fix header bug and update README"

# Good
git commit -m "feat(auth): Add login feature"
git commit -m "fix(ui): Fix header alignment"
git commit -m "docs: Update README with login instructions"
```

### FAQ

#### How to amend the last commit message

```bash
git commit --amend
```

#### How to squash multiple commits

```bash
git rebase -i HEAD~3
```

---

## 5. Git Safety Rules

### Never Do

- `git push --force` on main/master/develop
- Commit secrets to public repositories
- Modify pushed commits after rebase/force push

### Recovery Guide

| Situation                     | Solution                           |
| ----------------------------- | ---------------------------------- |
| Committed to wrong branch     | `git cherry-pick` or `git reset`   |
| Committed sensitive info      | `git filter-repo` or BFG Cleaner   |
| Want to undo merge            | `git reset --hard ORIG_HEAD`       |
| Restore specific file         | `git checkout HEAD~1 -- <file>`    |
