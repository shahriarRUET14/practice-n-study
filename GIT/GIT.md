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

---

## 📘 Code Review Guidelines

### 🎯 Purpose of Code Review

The goal of code review is to ensure correctness, maintain quality, reduce bugs, and help each other grow.

Reviews must be constructive, respectful, and focused on the code, not the person.

### ⭐ Top 5 Critical Review Areas (Must-Check)

**1. Correctness of Logic**
- Validate business logic against requirements
- Confirm all edge cases and failure scenarios are handled
- Ensure no leftover test or experimental code

**2. Security**
- No hardcoded credentials, tokens, or secrets
- Validate and sanitize all external inputs
- No sensitive data in logs or responses
- Follow secure patterns for auth, DB access, and external calls

**3. Performance**
- No N+1 queries — review SQL carefully
- Avoid expensive operations inside loops
- Ensure caching or memoization is used where appropriate
- Check API and DB boundaries for potential bottlenecks

**4. Readability & Maintainability**
- Code should be understandable without explanation
- Clear naming (methods, variables, classes)
- Functions should be small and single-purpose
- Remove dead code, unused imports, unnecessary comments

**5. Error Handling**
- Use meaningful error messages
- Proper exception flow — no silent failures
- API controllers must return consistent error formats
- Avoid swallowing errors just to "make it work"

### 🧩 Architecture & Structure

- Code follows team patterns and conventions (folder structure, naming)
- Separation of concerns: controllers → services → repositories
- Avoid tight coupling; prefer interfaces/abstractions where needed
- Reuse existing helpers, services, or utilities

### 🔐 API & Integration Rules

- Proper and consistent HTTP status codes
- Clear request/response schema
- Pagination and filters where required
- No breaking changes without versioning or communication

### 🧪 Testing Guidelines

- Critical logic must have unit tests
- API/integration tests for business workflows
- Tests should be meaningful, not just for coverage
- Avoid flaky tests; isolate external dependencies

### 📊 Logging & Observability

- Log important business events
- Use the correct log level (info/warn/error)
- **Do NOT log:**
  - passwords
  - tokens
  - financial sensitive data

### 📚 Documentation & Comments

- Write documentation or explanations for complex logic
- Update README, API docs, or migration notes if needed
- Avoid obvious or redundant comments

### 🤝 Review Etiquette

- Be respectful; suggest improvements instead of criticizing
- Focus on the code quality, not the coder
- Provide clear reasons for requested changes
- Approve when the code is good — not perfect

### 🟢 When to Approve

Approve if:
- All critical areas are addressed
- No security or performance risks
- Code is readable and maintainable
- Tests pass and cover the important logic