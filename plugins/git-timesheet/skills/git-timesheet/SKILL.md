---
name: git-timesheet
description: Generate ERP-ready timesheet entries from git commit history using the git-report CLI. Use whenever the user asks for a work report, timesheet, activity log, or monthly summary from git commits — regardless of current directory or whether any CSV exists yet. Handles date ranges, author filters, and branch filters.
---

# Git Timesheet

Generate a timesheet from git commit history using `git-report`, then synthesize the result into ERP-ready entries.

## When to use

- The user asks something like: "give me the report for March", "build a timesheet for last month", "summarize my commits from January to March", "what did John work on this week", or "prepare my activity log for the ERP".
- No pre-existing CSV is required — this skill generates one via `git-report`.

## Step 1 — Extract parameters from the user's request

Parse the user's message to extract:

| Parameter | Flag | Default | Examples |
|-----------|------|---------|---------|
| Repo path | `-r` | `.` (current directory) | `~/projects/myapp`, `/path/to/repo` |
| Since date | `-s` | (none — all history) | `2026-01-01`, `1 month ago`, `2026-06` → `2026-06-01` |
| Until date | `-u` | (none — today) | `2026-01-31`, `2026-06` → `2026-06-30` |
| Author(s) | `-a` | (none — all authors) | name or email; repeat flag for multiple |
| Branch(es) | `-b` | (none — all branches) | repeat flag for multiple |

**Date inference rules:**
- "June" or "June 2026" → `-s 2026-06-01 -u 2026-06-30`
- "last month" → `-s <first day of previous month> -u <last day of previous month>`
- "this month" → `-s <first day of current month> -u <today>`
- "Q1 2026" → `-s 2026-01-01 -u 2026-03-31`
- "last week" → `-s <last Monday> -u <last Sunday>`
- A plain year "2025" → `-s 2025-01-01 -u 2025-12-31`

**Author inference:** if the user says "my commits" or "my work" and no author is specified, use the git user from the repo (`git config user.name` or `git config user.email` in the target repo).

**Branch inference:** if the user does not mention a branch, do **not** pass `-b` at all — let `git-report` include all branches. Only add `-b` when the user explicitly names one or more branches.

## Step 2 — Ensure git-report is installed

Before running anything, check whether `git-report` is available:

```bash
command -v git-report
```

If found, skip to Step 3. If not found, install it automatically:

### 2a — Detect platform

```bash
OS=$(uname -s | tr '[:upper:]' '[:lower:]')   # darwin | linux
ARCH=$(uname -m)                               # x86_64 | arm64 | aarch64
```

Map to asset name:
- `OS=darwin, ARCH=arm64`  → `git-report-<version>-darwin-arm64`
- `OS=darwin, ARCH=x86_64` → `git-report-<version>-darwin-amd64`
- `OS=linux,  ARCH=aarch64 or arm64` → `git-report-<version>-linux-arm64`
- `OS=linux,  ARCH=x86_64` → `git-report-<version>-linux-amd64`
- Windows (PowerShell, `$env:OS` contains `Windows`) → `git-report-<version>-windows-amd64.exe`

If the combination is not in this list, tell the user their platform is unsupported and stop.

### 2b — Fetch the latest version tag

```bash
VERSION=$(gh release view --repo djoufson/git-report --json tagName --jq '.tagName' | sed 's/^v//')
```

If `gh` is not available, hard-code `VERSION=1.0.0` and note this may not be the latest.

### 2c — Download and install

```bash
ASSET="git-report-${VERSION}-${OS}-${ARCH_MAPPED}"   # ARCH_MAPPED from the table above
URL="https://github.com/djoufson/git-report/releases/download/v${VERSION}/${ASSET}"
TMP=$(mktemp)
curl -fsSL "$URL" -o "$TMP"
chmod +x "$TMP"
```

Install to the first writable location in this order:
1. `/usr/local/bin/git-report` — try `mv "$TMP" /usr/local/bin/git-report` directly (no sudo); works on most developer machines.
2. If that fails (permission denied), install to `~/.local/bin/git-report` instead:
   ```bash
   mkdir -p ~/.local/bin
   mv "$TMP" ~/.local/bin/git-report
   ```
   Then warn the user: "`~/.local/bin` was used — make sure it is in your `$PATH` before running this skill again."

On Windows, download the `.exe` to `$env:LOCALAPPDATA\Programs\git-report\git-report.exe` and tell the user to add that directory to their `PATH` if it isn't already.

After installing, verify with `git-report --version`. If verification fails, show the error and stop.

## Step 3 — Run git-report

Determine the output path for the CSV (prefer `/tmp/git-report-<project>.csv` to avoid cluttering the user's repo):

```bash
git-report \
  -r <repo-path> \
  [-s <since>] \
  [-u <until>] \
  [-a <author>] \
  [-b <branch>] \
  -o /tmp/git-report-<project-name>.csv
```

- If the user mentions multiple repos, run `git-report` once per repo and collect all CSVs.
- Derive the project name from the repo directory name (e.g., `/home/user/myapp` → `myapp`).
- If `git-report` exits with an error, show the error and stop.

## Step 4 — Read and parse the CSV

Each CSV has a header row and one row per commit. The columns are at least:

```
Branch,Commit Hash,Short Hash,Author,Email,Date,Message,Files Changed,Lines Added,Lines Deleted
```

- The `Date` column is an ISO timestamp; only the date portion (YYYY-MM-DD) matters for grouping.
- The `Message` field may contain commas — it is always quoted when so; respect CSV quoting.
- If the file is empty (header only), report no activity for that project and skip synthesis.

## Step 5 — Synthesize timesheet entries

### Grouping rules

- Default granularity: **one entry per `(project, date, branch)` triple.** If two days of work happen on the same branch, that's two entries (one per day).
- A single commit appearing on multiple branches: attribute it to the more specific branch (feature branch over `main`/`develop`). Use commit timestamp as tiebreaker.
- Skip pure merge commits (0 lines added, 0 lines deleted, message starting with `Merge branch`) unless they are the *only* activity on that branch that day — then keep a single "merge develop" line.
- If a commit message starts with a Conventional Commit prefix (`feat(...)`, `fix(...)`, `chore(...)`, `refactor(...)`, `polish(...)`, `docs(...)`, `style(...)`, `ci(...)`), you may strip the prefix in Markdown bullets when it would just repeat what's already implied. Keep it verbatim in the CSV summary.

### Ordering

- Multiple projects: alphabetical unless the user specified an order.
- Within a project: chronological (oldest day first); within a day: branches alphabetical.

## Step 6 — Write output

Always produce **two files**. Write them to the directory the user is working from (or a path the user specified):

1. **`timesheet.csv`** — bulk-importable into the ERP.
   - Columns: `Date,Project,Branch/Task,Summary`
   - One row per `(Date, Project, Branch)` triple.
   - `Summary` is a single quoted cell containing all commit messages for that group, joined with `; `. Quote any cell that contains a comma.

2. **`timesheet.md`** — human-readable review copy.
   - Organized as: `## <Project>` → `### <Date> — \`<branch>\`` → one-liner summary → bullet list of commit messages.
   - The one-liner summary is a single italic sentence (e.g., `*Implemented user authentication flow and fixed token expiry edge case.*`) that captures the theme of that group's commits in plain language. Write it in past tense, as a task description. Keep it under 120 characters.
   - Capitalize/clean up commit messages lightly.
   - At the top, print the `git-report` parameters used (period, author, repo) so the user knows exactly what was included.
   - Note any repos/periods with no commits.

## Step 7 — End summary

End with:
- The `git-report` command(s) that were run.
- Count of day-entries per project.
- Total commit count processed.
- List of any repos/periods with zero activity.

## Quality bar

- Every commit in every non-empty CSV must be represented somewhere in the output (unless it's a skipped merge commit). Spot-check: total bullet count in Markdown ≈ total non-merge commit rows across inputs.
- Commit messages should be readable as task descriptions — prefer light cleanup over heavy rewriting.
- Don't invent work. If a commit message is terse ("fix bug", "wip"), keep it terse.
- Don't estimate hours unless the user explicitly asks.

## Example invocations

> "Give me the timesheet for June 2026 on this project."

→ Infer `-s 2026-06-01 -u 2026-06-30`, repo = current directory, author = git config user. Run `git-report`, synthesize output.

> "I want a report of John's work on the auth branch from January to March."

→ `-s 2026-01-01 -u 2026-03-31 -a "John" -b auth`, repo = current directory.

> "Build me an activity log for last month across ~/projects/frontend and ~/projects/backend."

→ Run `git-report` twice (one per repo), merge results, produce unified `timesheet.csv` and `timesheet.md` with two project sections.
