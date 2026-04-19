---
name: audit-repo
description: >
  Performs a comprehensive audit of the current JS/TS/Node repository and produces a
  prioritized, actionable list of bugs, optimizations, and possible evolutions.
  Trigger this skill whenever the user asks to audit, analyze, or review a repository,
  mentions finding bugs, checking code quality, reviewing documentation, or wants a
  global health check of a project — even if they don't use the word "audit".
  Examples: "audit the repo", "find bugs in this project", "what could be improved here",
  "review the codebase", "analyse le dépôt", "fais un audit", "qu'est-ce qui ne va pas dans ce projet".
---

# audit-repo

Perform a full audit of the current JS/TS/Node repository, then guide the user through
selecting items to address and creating a GitHub issue for each one.

---

## Phase 0 — Context check

Before anything else, verify you are in a JS/TS/Node project:

- Run `git rev-parse --show-toplevel` to confirm a git repository is present. If not,
  ask the user which repository they want to audit.
- Check that `package.json` exists at the root. If it doesn't, explain that this skill
  targets JS/TS/Node projects and ask the user to confirm they want to proceed anyway.

---

## Phase 1 — Audit

Read the project in parallel across these dimensions. Aim for breadth and depth: don't
just skim file names, actually read the source code, test files, and config.

### 1a — Project overview
- `package.json`: name, version, scripts, dependencies, devDependencies, exports, main/module fields
- `README.md` (and any other `.md` at root): declared API, usage examples, install instructions
- Public API exports (`src/lib/index.*`, `src/index.*`, `index.*`, or whatever the package entry point is)

### 1b — Source code
Read all source files systematically. Look for:

**Bugs**
- Logic errors, off-by-one, incorrect conditions (especially around edge cases and falsy values)
- Race conditions, async/await misuse, missing `await`
- Unhandled promise rejections or missing error boundaries
- Memory leaks (event listeners not removed, timers not cleared)
- Incorrect type coercion or implicit conversions
- Dead code that could mask issues
- API surface inconsistencies (props behaving differently than documented)

**Accessibility** (if the project produces UI — components, actions, DOM manipulation)
- Missing ARIA roles, labels, or attributes
- Focus management issues
- Keyboard navigation gaps
- Color contrast or motion sensitivity issues not handled

### 1c — Tests
- Check coverage breadth: which code paths have no tests?
- Look for tests that pass trivially without actually verifying behavior
- Missing regression tests for known edge cases
- Test setup/teardown issues that could cause flaky tests

### 1d — Documentation
- Inconsistencies between README and actual code behavior
- Outdated examples (options/props that no longer exist, or new ones not documented)
- Missing documentation for public API members
- Misleading or ambiguous descriptions

### 1e — Tooling
- **Build**: `vite.config.*`, `rollup.config.*`, `webpack.config.*`, `tsconfig.json` — misconfigurations, suboptimal settings
- **Tests**: `vitest.config.*`, `jest.config.*` — coverage thresholds, missing reporters
- **Linting/formatting**: `.eslintrc*`, `eslint.config.*`, `prettier.config.*`, `.prettierrc*` — rules missing, conflicting, or not enforced
- **CI/CD**: `.github/workflows/`, `.gitlab-ci.yml` — missing steps, no caching, outdated action versions
- **Package**: `package.json` scripts completeness, `engines` field, `files`/`exports` fields correctness, `publint` issues
- **Dependencies**: outdated major versions, security advisories, unnecessary dependencies

### 1f — Evolutions
Think about what features or improvements would genuinely add value to the project:
- Features commonly expected by users of this type of library/tool
- Gaps in the current API compared to what the documentation promises
- Patterns the code is almost doing, but not quite

For each proposed evolution, analyze the public API exports and documented behavior to
determine if it would introduce a **breaking change** (removal or modification of existing
public API, behavior change for existing valid inputs).

---

## Phase 2 — Report

Present findings as a numbered checklist. Only include a category if it has items.
Within each category, sort by severity/impact (most critical first).

Use this exact format:

```
## 🐛 Bugs

- [ ] **1. [CRITICAL]** Short title — one-sentence description of the problem
- [ ] **2. [HIGH]** Short title — one-sentence description
- [ ] **3. [MEDIUM]** Short title — one-sentence description
- [ ] **4. [LOW]** Short title — one-sentence description

## ⚡ Optimizations

- [ ] **5. [PERF]** Short title — one-sentence description
- [ ] **6. [DOC]** Short title — one-sentence description
- [ ] **7. [TOOLING]** Short title — one-sentence description
- [ ] **8. [A11Y]** Short title — one-sentence description

## 🚀 Evolutions

- [ ] **9.** Short title — one-sentence description
- [ ] **10. [BC]** Short title — one-sentence description *(breaking change)*
```

**Severity levels for bugs**: `CRITICAL` (data loss, security, crashes in normal use) /
`HIGH` (incorrect behavior affecting most users) / `MEDIUM` (edge case, workaround exists) /
`LOW` (cosmetic, rarely triggered)

**Optimization tags**: `PERF` / `DOC` / `TOOLING` / `A11Y` / `TEST` / `SECURITY` / `STYLE`

Then ask:

> Reply with the numbers of the items you want to address (e.g. `1, 3, 5`) or `all` to
> select everything. I'll create a GitHub issue for each selected item.

---

## Phase 3 — Issue creation

Once the user confirms their selection, create one GitHub issue per selected item.

Run issue creations in parallel where possible to save time.

**For bugs**, use this body template:

```markdown
## Summary
<1-2 sentences describing the bug>

## Root cause
<What in the code causes this — file, function, line if known>

## Impact
<Who is affected and how severely>

## Expected behavior
<What should happen instead>
```

**For optimizations**, use this body template:

```markdown
## Summary
<1-2 sentences describing the current problem>

## Current state
<What the code/docs/tooling looks like now, and why it's suboptimal>

## Proposed improvement
<Concrete suggestion for what to change>
```

**For evolutions**, use this body template:

```markdown
## Summary
<1-2 sentences describing the proposed feature>

## Motivation
<Why this would be valuable — user need, gap in the API, common pattern>

## Proposed approach
<High-level technical direction>

## Breaking changes
<"None" or specific description of what existing behavior would change and migration path>
```

**Labels to apply automatically**:
- Bugs → `bug`
- `[DOC]` optimizations → `documentation`
- `[PERF]` optimizations → `performance` (create if it doesn't exist)
- `[A11Y]` optimizations → `accessibility` (create if it doesn't exist)
- `[TOOLING]` / `[TEST]` / `[STYLE]` optimizations → `chore`
- `[SECURITY]` optimizations → `security` (create if it doesn't exist)
- Evolutions → `enhancement`
- Evolutions with `[BC]` → add `breaking change` label (create if it doesn't exist)

After all issues are created, print a summary table:

| # | Title | Issue |
|---|-------|-------|
| 1 | Bug title | #42 |
| 5 | Optim title | #43 |
