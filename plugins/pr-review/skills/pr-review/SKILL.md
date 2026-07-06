---
name: pr-review
description: Comprehensive PR code review with inline GitHub comments
user_invocable: true
arguments:
  - name: pr
    description: "PR number or URL to review"
    required: true
---

# PR Review Skill

Perform a thorough code review on a pull request and post feedback as **inline GitHub comments** on specific lines, plus a top-level summary comment.

## Instructions

### Step 1 — Gather PR context

```bash
# Get PR metadata (title, body, base branch, author)
gh pr view {{pr}} --json title,body,baseRefName,headRefName,author,labels,number

# Get the full diff
gh pr diff {{pr}}

# List changed files
gh pr diff {{pr}} --name-only
```

Read every changed file in full (not just the diff) to understand surrounding context. Also read any closely related files (imports, callers, tests) when needed to judge correctness.

Before reviewing, check for project-specific conventions in the repository:
- `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or similar at the repo root
- `README.md` and `CONTRIBUTING.md`
- `.claude/` or `docs/` directories
- Linter configs (`.eslintrc`, `pyproject.toml`, etc.)

Apply those conventions during the review — do not impose generic preferences when the project has documented its own.

### Step 2 — Review the changes

Analyze every changed file against **all** of the following focus areas. Only report **actual issues that need to be resolved** — do not praise, do not nitpick style that linters already enforce.

#### Code Quality
- Architecture/layering violations (e.g., domain depending on infrastructure, UI depending on DB)
- Missing or incorrect error handling; unhandled edge cases
- Dead code, unused imports, duplicated logic
- Naming that misleads or obscures intent
- Functions/classes/modules doing too much (low cohesion)
- Hardcoded values that should be config or constants
- Inconsistent patterns within the same codebase

#### Security
- Injection vulnerabilities (SQL, command, XSS, path traversal, SSRF, etc.)
- Missing input validation / sanitization at system boundaries
- Authentication or authorization gaps (missing auth checks, role checks, IDOR)
- Secrets, credentials, or API keys committed in code or logs
- Unsafe rendering of user input (e.g., `dangerouslySetInnerHTML`, unescaped templates)
- Missing rate limiting on public endpoints
- Insecure cryptography (weak algorithms, hardcoded keys, missing salts)
- CSRF / CORS misconfigurations

#### Performance
- N+1 queries, missing indexes, unbounded result sets (no LIMIT/pagination)
- Unnecessary re-renders or recomputations (missing memoization, unstable dependencies)
- Blocking operations on hot paths (sync I/O, heavy computation)
- Memory leaks (uncleaned subscriptions, listeners, intervals, large caches without eviction)
- Over-fetching or transferring large payloads when a subset is needed
- Inefficient algorithms where simpler/faster alternatives exist

#### Architecture & Conventions
- Violations of project-specific conventions documented in `CLAUDE.md` / `AGENTS.md` / `README.md`
- Imports crossing boundaries that the project explicitly isolates
- Missing registration in plugin/registry systems when the project uses one
- Inconsistent error/response patterns vs. existing handlers
- Direct logging primitives where the project uses a logger abstraction
- Missing i18n/translations for user-facing strings (when project is internationalized)
- Direct usage of low-level APIs where the project provides wrappers

#### Testing
- Missing tests for new business logic, edge cases, or critical paths
- Tests that mock the system under test (false confidence)
- Missing edge case coverage (empty inputs, unauthorized access, concurrent modifications, error paths)
- Flaky patterns (time-based assertions without freezing, order-dependent tests)
- Tests that don't actually assert meaningful behavior

#### Documentation & Configuration
- New env vars not added to `.env.example` (or equivalent)
- Schema/database changes without migration files
- Breaking API changes without changelog or version bump
- New features or config options missing documentation
- Outdated comments or docs that no longer match the code

### Step 3 — Post inline comments

For each issue found, post an **inline review comment** on the exact file and line using the GitHub API:

```bash
# Post a single inline comment on a specific line of the diff
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments \
  --method POST \
  -f body="<comment>" \
  -f commit_id="<latest_commit_sha>" \
  -f path="<file_path>" \
  -F line=<line_number> \
  -f side="RIGHT"
```

To get the latest commit SHA:
```bash
gh pr view {{pr}} --json commits --jq '.commits[-1].oid'
```

**Comment format** for each inline comment:

```
**[<SEVERITY>]** <category>

<Clear description of the issue>

<Suggested fix or direction — be specific, show code when helpful>
```

Severity levels:
- `CRITICAL` — Must fix before merge (security holes, data loss, broken logic)
- `ISSUE` — Should fix before merge (bugs, missing validation, perf problems)
- `SUGGESTION` — Consider improving (readability, minor optimization, better pattern)

### Step 4 — Post summary comment

After all inline comments are posted, post a **single top-level comment** summarizing the review:

```bash
gh pr comment {{pr}} --body "$(cat <<'EOF'
## Code Review Summary

**Files reviewed:** <count>
**Issues found:** <critical_count> critical, <issue_count> issues, <suggestion_count> suggestions

### Critical issues
- <list or "None">

### Key issues
- <list or "None">

### Suggestions
- <list or "None">

---
*Automated review by Claude Code*
EOF
)"
```

### Rules

- **Do NOT approve or request changes** — only leave comments. The human reviewer decides.
- **Be specific** — every comment must reference the exact problem and ideally show a fix.
- **No fluff** — no "great job", no "looks good overall", no filler.
- **Group related issues** — if the same mistake repeats across files, comment on the first occurrence and mention "same issue in X other files".
- **Respect project conventions** — review against the rules documented in the repo (`CLAUDE.md`, `AGENTS.md`, `README.md`, `CONTRIBUTING.md`, `.claude/`, etc.), not generic preferences.
- **Match the stack** — apply checks relevant to the languages/frameworks present in the diff; skip categories that don't apply.
- If the PR is clean and no issues are found, post only a brief summary comment stating that no issues were identified.
