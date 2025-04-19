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
