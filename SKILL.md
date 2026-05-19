---
name: bugfix
description: >
  Reads a source file, identifies bugs, fixes them, and creates a pull request
  with a title and description explaining the bugs found and the changes made.
  Use when the user says "bugfix", "fix bugs in", "find bugs", "debug my code",
  "auto fix", or "review and fix <file>".
---

# Bug-Fix and PR Creator

Analyzes any source file for bugs, applies precise fixes, and opens a GitHub pull request documenting what was found and changed.

---

## Step 1 — Resolve the target file

- If `$ARGUMENTS` is provided, treat it as the file path.
- Otherwise, ask the user: *"Which file should I analyze for bugs?"*
- Use the Read tool to load the file contents.
- Identify the language from the file extension.

---

## Step 2 — Analyze for bugs

Carefully examine the code for all of the following:

- **Off-by-one errors** — loop bounds, array indices, fence-post issues
- **Out-of-bounds access** — reading or writing past the end of an array/buffer
- **Null / uninitialized access** — dereferencing null pointers, using uninitialized variables
- **Infinite loops** — missing or incorrect termination conditions
- **Logic errors** — wrong operators, flipped conditions, incorrect boolean expressions
- **Resource leaks** — unclosed file handles, sockets, or allocated memory never freed
- **Integer overflow / underflow**
- **Unreachable / dead code** — branches that can never execute
- **Type mismatches** — signed/unsigned comparison, implicit narrowing
- **Concurrency issues** — race conditions, missing locks (if applicable)

Present findings as a numbered list. For each bug include:
1. **Bug type** (e.g., "Off-by-one error")
2. **Location** — line number and the exact problematic line of code
3. **Explanation** — why this is a bug and what it causes at runtime
4. **Proposed fix** — the corrected line(s) of code

If no bugs are found, tell the user clearly, then ask via `AskUserQuestion`:

> "No bugs were found in this file. Would you like me to create a pull request anyway (e.g., to document the review or as a no-op change)?"

Options:
- **"Yes, create a PR"** → proceed to Step 4 with a commit message such as `chore(<filename>): no bugs found — routine review`
- **"No, that's fine"** → stop here

---

## Step 3 — Confirm with the user

Ask via `AskUserQuestion`:

> "I found [N] bug(s) above. Would you like me to fix them and create a pull request?"

Options:
- **"Yes, fix and create PR"** → proceed to Step 4
- **"No, just show me the bugs"** → stop here

---

## Step 4 — Create a fix branch

Determine the base filename without extension (e.g., `sorting` from `sorting.cpp`).

Run:
```bash
git checkout main
git pull origin main
git checkout -b bugfix/<base-filename>
```

If `main` does not exist, use `master` or the repo's default branch.

---

## Step 5 — Apply the fixes

Use the Edit tool to apply each fix precisely to the file.

Rules:
- Change only the buggy lines — do not reformat, rename, or refactor unrelated code.
- Apply all fixes in a single pass.

---

## Step 6 — Commit

```bash
git add <file-path>
git commit -m "fix(<filename>): <one-line summary of all bugs fixed>"
```

---

## Step 7 — Push and open the pull request

```bash
git push -u origin bugfix/<base-filename>
```

Then create the pull request using **whatever method is available** — probe silently and use the first option that works:

1. **MCP GitHub tool** — If a connected MCP server exposes a `create_pull_request` (or equivalent) tool, use it directly. No CLI required.
2. **`gh` CLI** — If `gh` is installed and authenticated (`gh auth status` exits 0), run `gh pr create`.
3. **REST API via `curl`** — If neither of the above is available but `curl` and a `GITHUB_TOKEN` environment variable are present, call the GitHub REST API (`POST /repos/{owner}/{repo}/pulls`) directly.
4. **Manual instructions** — If none of the above succeed, output the exact PR title and body as plain text so the user can paste them into the GitHub web UI at `https://github.com/<owner>/<repo>/compare/bugfix/<base-filename>`.

Do not ask the user which method to use. Try each option in order, move on silently if one fails, and only fall back to manual instructions as a last resort.

Whichever method is used, populate the PR with the following content (use actual bug details from Step 2 — no placeholder text):

**Title:** `fix(<filename>): <concise bug summary>`

**Body:**
```
## Bugs Found

<For each bug: type, location, what it causes>

## Root Cause

<Why the bug(s) exist — e.g., incorrect loop bound, missing null check>

## Changes Made

<For each fix: show the old line(s) and new line(s) as a before/after diff>

## How to Verify

<Steps to build/run and confirm the bug is gone — e.g., compile and run, run tests, check output>
```
