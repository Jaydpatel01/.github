---
name: code-quality-and-review
description: Maintain code correctness, readability, and safety across a codebase. Use when reviewing code, performing refactoring, enforcing code quality standards, or conducting technical debt analysis.
---

# Skill: Code Quality and Review

## Purpose
Maintain a codebase that is correct, readable, and safe to change. Code quality is not about aesthetics — it is about reducing the cost and risk of every future change. Code that is hard to understand is code that is hard to fix without introducing new bugs.

## When to Use
Apply this skill continuously: before opening a pull request, during code review, when working in legacy code, and when evaluating the health of a codebase.

## Core Principles
- **Code is read far more than it is written.** Optimize for the reader, not the writer.
- **Clarity over cleverness.** A simple, obvious implementation is better than a clever one that requires a comment to explain.
- **Names are the most important documentation.** A well-named variable, function, and class eliminates the need for many comments.
- **Small, focused units.** Functions and classes that do one thing are easier to test, understand, and reuse.
- **Explicit over implicit.** Make dependencies, data flow, and side effects visible.
- **Leave the campsite cleaner than you found it.** Every change is an opportunity to improve the code around it.

## Code Quality Standards

### Naming Conventions
- Variables: descriptive nouns — `userAccountBalance`, not `bal`, not `x`.
- Functions: verb + noun — `calculateOrderTotal()`, `sendWelcomeEmail()`, `validateUserInput()`.
- Booleans: `is`, `has`, `should`, `can` prefix — `isAuthenticated`, `hasPermission`, `shouldRetry`.
- Constants: `SCREAMING_SNAKE_CASE` for true constants; camelCase for configuration objects.
- Avoid abbreviations unless they are universally understood in the domain (e.g., `id`, `url`, `http`).
- Avoid generic names: `data`, `info`, `temp`, `obj`, `result` — use the actual domain concept.

### Function Design
- **Single Responsibility**: a function should do exactly one thing and do it completely.
- **Size**: a function that does not fit on one screen (≈ 50 lines) is almost always doing too many things.
- **Arguments**: limit to 3 parameters. More than 3 → use an options object or a parameter object.
- **No side effects in query functions**: a function that returns data should not modify state.
- **Command/Query Separation (CQS)**: functions either perform an action (command) or return data (query), not both.
- **Avoid boolean flags**: a function `process(user, true)` is unclear. Create `processForAdmin(user)` and `processForUser(user)`.
- **Early returns**: use guard clauses to handle invalid/edge cases at the top of the function, reducing nesting depth.

### Class and Module Design
- **SOLID Principles**:
  - **S** — Single Responsibility: one reason to change
  - **O** — Open/Closed: open for extension, closed for modification
  - **L** — Liskov Substitution: subtypes must be substitutable for their base type
  - **I** — Interface Segregation: many small interfaces over one large one
  - **D** — Dependency Inversion: depend on abstractions, not concretions
- Keep modules cohesive: everything in a module should belong together conceptually.
- Minimize coupling: modules should know as little as possible about each other's internals.

### Error Handling
- Use typed error classes that convey meaning: `NotFoundError`, `ValidationError`, `ConflictError`, `AuthorizationError`.
- Never swallow errors silently: `catch(e) {}` is almost always wrong. Log and re-throw, or handle explicitly.
- Fail loud in development, fail gracefully in production.
- Never expose internal error details (stack traces, database error messages) to clients.
- Handle every possible rejection from Promises/async functions.

### Comments
Use comments only for:
- Explaining *why* a decision was made that the code cannot express (e.g., "// Using a workaround for a known bug in library X v2.1")
- Warning about non-obvious gotchas (e.g., "// This must run before Y because of Z")
- Providing context for complex algorithms (link to the paper or explanation)

Do not comment:
- What the code does (the code should be readable enough to show this)
- Code that has been commented out — delete it; version control remembers everything

### Complexity Management
- **Cyclomatic complexity**: keep functions below 10 (ideally below 7).
- **Cognitive complexity**: if you need to trace more than 3 levels of nesting to understand the flow, refactor.
- **Magic numbers and strings**: replace with named constants — `const MAX_LOGIN_ATTEMPTS = 5` not `if (attempts > 5)`.
- **DRY (Don't Repeat Yourself)**: three or more duplications of similar logic should be extracted and named.

## Static Analysis and Tooling

### Linting (catch bugs and style issues)
- **JavaScript/TypeScript**: ESLint with `@typescript-eslint/recommended`, `eslint-plugin-unicorn` (modern JS patterns), `eslint-plugin-sonarjs` (code smell detection).
- **Python**: Ruff (fast, comprehensive) or Flake8 + Black + isort.
- **Go**: `golangci-lint` with standard linters enabled.
- Configure linters to run in CI and fail the build on errors.

### Type Safety
- Enable TypeScript strict mode: `"strict": true` in `tsconfig.json`.
- This enables: `strictNullChecks`, `noImplicitAny`, `strictFunctionTypes`, etc.
- Never use `any` type; use `unknown` when the type is genuinely unknown and then narrow it.
- Validate runtime data against type schemas at trust boundaries (Zod, Yup, io-ts).

### Formatting (eliminate style debates)
- Use **Prettier** (JS/TS/CSS) or **Black** (Python) — automated, non-negotiable formatting.
- Configure as a pre-commit hook so formatted code is the only code that can be committed.
- Do not include formatting discussions in code reviews — let the tool handle it.

### Dependency Management
- Keep dependencies up to date — outdated dependencies are a security risk.
- Run `npm audit` / `pip-audit` / `go mod tidy` weekly and in CI.
- Use exact versions in package lock files to ensure reproducible builds.
- Audit every new dependency: is it actively maintained? What is its license? What is its download/community size?
- Prefer standard library solutions over third-party dependencies for simple tasks.
- Remove unused dependencies promptly.

### Code Complexity Analysis
- Use complexity analysis tools: `complexity-report` (Node.js), Radon (Python), `gocyclo` (Go).
- Flag functions exceeding cyclomatic complexity of 10 in CI.
- Refactor complex functions into smaller, testable units before adding new logic to them.

## Pull Request and Code Review Standards

### Opening a PR
A PR description must include:
- **What** was changed and why (not just "fixed bug" — explain the root cause and the fix).
- **How to test**: steps for the reviewer to manually verify the change.
- **Screenshots / recordings** for UI changes.
- **Risk assessment**: what could go wrong? What has been tested?
- **Checklist**: tests added/updated, documentation updated, CHANGELOG entry, no hardcoded secrets.

PR size guidelines:
- Small PRs (< 400 lines changed) are reviewed faster and more thoroughly.
- Break large features into sequential, independently mergeable PRs.
- A PR that adds a feature and refactors unrelated code is two PRs.

### Reviewing Code
Reviewers look for:

**Correctness**
- Does the code do what the description says?
- Are edge cases handled? (empty inputs, nulls, race conditions, large inputs)
- Are error paths handled correctly?
- Are there off-by-one errors or incorrect boundary conditions?

**Security**
- Is user input validated before use?
- Could this code be exploited? (injection, IDOR, privilege escalation)
- Are there hardcoded secrets, tokens, or credentials?
- Are new dependencies trusted and free of known CVEs?

**Maintainability**
- Is the code easy to understand without additional context?
- Are functions and variables named clearly?
- Is there duplicated logic that should be extracted?
- Is the abstraction level consistent?

**Performance**
- Are there N+1 query problems (calling the database inside a loop)?
- Are large datasets loaded into memory when they should be streamed?
- Are expensive operations memoized or cached appropriately?

**Tests**
- Are there tests for the new/changed behavior?
- Do the tests test behavior or implementation?
- Are edge cases covered?

### Review Etiquette
- Distinguish between: required changes, suggested improvements, and optional nits.
- Use prefixes: `[required]`, `[suggestion]`, `[nit]`, `[question]`.
- Request changes only for correctness, security, or significant maintainability issues.
- Style issues should be automated away — do not waste review time on formatting.
- Be specific: instead of "this is confusing", say "I had to re-read this 3 times to understand the control flow. Could we extract the condition into a named function `isEligibleForDiscount()`?"
- Approve when requirements are met, not when the code is perfect.

### Code Review SLAs
- First review: within 1 business day of PR being opened.
- Re-review after changes: within 4 hours.
- A PR that sits unreviewed for > 2 business days should trigger an escalation.

## Technical Debt Management
- Track technical debt explicitly in a `TECH_DEBT.md` or as labeled GitHub Issues.
- Each debt item should include: what the problem is, what the ideal solution is, estimated effort, and impact.
- Reserve 20% of sprint capacity for addressing technical debt.
- Do not accumulate debt silently — when taking a shortcut, create the debt item immediately.
- Prioritize debt that causes bugs, slows down development, or creates security risk.

## Deliverables
- ESLint/Ruff/golangci-lint configuration committed to the repository
- Prettier / Black configuration committed
- Pre-commit hooks configured (husky + lint-staged for JS/TS, pre-commit framework for Python)
- TypeScript strict mode enabled
- `npm audit` / `pip-audit` running in CI with failure threshold
- PR template with required checklist
- Code review guidelines documented in `CONTRIBUTING.md`
- Complexity thresholds configured in CI
