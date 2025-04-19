# Git Branching Strategy and Workflow

You're hitting a common issue that stems from not clearly separating your **deployment branches** from your **development flow**. Let’s break it down and get you set up with proper Git branch conventions to prevent conflicts in the future.

---

### 🧠 Why You're Getting Merge Conflicts

If `prod` was created from `development` and now you're merging features into both or directly into `prod`, you're essentially creating **two diverging histories**. Once the branches start to differ (e.g., a hotfix in prod not present in dev, or different merging orders), Git will treat them as separate and start throwing conflicts.

---

## ✅ Recommended Git Branching Strategy

### 🔱 Branch Convention

| Branch             | Purpose                                                   |
|--------------------|-----------------------------------------------------------|
| `main` / `prod`    | **Production** ready code only. Stable. Protected.        |
| `development` / `dev` | Latest **integration branch**, where all features get merged first. |
| `feature/XYZ`      | Individual feature branches from `development`.           |
| `hotfix/XYZ`       | Urgent fixes from `prod` or `main`.                       |
| `release/XYZ` (optional) | Staging or pre-production testing branch, created from `development`. |

---

## ✅ Setup Guide

### 1. **Fix the Immediate Problem**
### Case 1.  **If brances your current main/prod branch is child of other branch like development**
Since your `prod` is diverging, do this:

#### Option A: Realign `prod` with `development` (recommended if prod isn’t too different)

```bash
# Checkout prod and merge development into it
git checkout prod
git merge development
# Resolve conflicts
git commit
# Push
git push origin prod
```

## ✅ Best Practices for Committing

1. **Commit Often, But Not Too Often**  
   - Avoid committing every minor change (e.g., fixing a typo). Instead, group related changes logically into a single commit.

2. **Write Clear Commit Messages**  
   - Use the format described in this guide: `<type>(optional scope): <subject-description>`.  
   - For example: `feat(api): Add support to create coupons`.

3. **Test Before Committing**  
   - Ensure your changes work as expected before committing to avoid introducing bugs into the codebase.
## ✅ GIT commit message writing

## Commit style

| Commit Type | Description                                 |
| ----------- | ------------------------------------------- |
| feat        | Introduce a new feature to the codebase     |
| fix         | Fix a bug in the codebase                   |
| docs        | Create/update documentation                 |
| style       | Feature and updates related to styling      |
| refactor    | Refactor a specific section of the codebase |
| test        | Add or update code related to testing       |
| chore       | Regular code maintenance                    |
### Example
- Use the following format for commit messages: `<type>(optional scope): <subject-description>`.
- For example: `feat(api): Add support to create coupons`.
- This format helps to provide a clear and concise description of the changes made in each commit.