---
name: optimize
description: >
  Reads a source file, intelligently finds code blocks whose time or space
  complexity can be reduced, rewrites them (preserving identical output), and
  optionally commits the changes to an optimize branch. Does NOT create a PR.
  Use when the user says "optimize", "improve performance", "reduce complexity",
  "make this faster", or "optimize <file>".
---

# Code Optimizer

Analyzes any source file for algorithmic inefficiencies, proposes provably correct rewrites that reduce time or space complexity, verifies the changes against the original, and optionally commits them to a dedicated branch.

---

## Step 1 — Resolve the target

`$ARGUMENTS` can be any of the following. Detect which case applies and handle accordingly:

**Case A — No arguments**
Ask: *"What would you like to optimize? (provide a file path, folder path, branch name, or GitHub repo URL)"*

**Case B — Single code file path** (e.g., `sorting.cpp`, `./src/main.py`)
Read the file directly with the Read tool. Proceed to Step 2.

**Case C — Folder path** (e.g., `./src/`, `.`)
List all code files in the folder recursively (ignore non-code files such as `.md`, `.txt`, `.json`, `.yaml`, images, binaries, etc.). Present the list to the user via `AskUserQuestion` and ask which file(s) to optimize. If the user picks one file, proceed to Step 2. If multiple, process them one at a time in sequence and repeat Steps 2–5 for each.

**Case D — Git branch name** (e.g., `bugfix/sorting`, `feature/auth`)
Run `git diff main...<branch> --name-only` (or against the repo's default branch) to list files changed on that branch. Filter to code files only. If only one code file changed, load it automatically and proceed to Step 2. If multiple, present the list via `AskUserQuestion` and ask which to optimize.

**Case E — GitHub repo URL** (e.g., `https://github.com/owner/repo`)
Parse `owner` and `repo` from the URL. Ask via `AskUserQuestion`:

> "Would you like me to clone the repo locally so I can apply changes, or just read files remotely (read-only analysis)?"

- **"Clone locally"** → intelligently detect the best available method: try `gh repo clone` first; if `gh` is not installed, try `git clone`. Treat the cloned folder as Case C.
- **"Read remotely (analysis only)"** → intelligently detect the best available method in this order:
  1. **GitHub MCP tools** (if available in this session) — use `get_file_contents` to list and read files.
  2. **`gh` CLI** — `gh api repos/{owner}/{repo}/git/trees/HEAD?recursive=1` to list; read with `gh api repos/{owner}/{repo}/contents/{path} --jq '.content' | base64 --decode`.
  3. **`curl` against GitHub REST API** — same endpoints, unauthenticated for public repos.

  Filter to code files. If multiple, ask which to optimize. Read and proceed to Step 2.
  Note: remote analysis is read-only — skip Steps 4–5 (apply/commit) unless the repo was cloned.

After resolving, read each target file with the Read tool and detect the language from the file extension.

---

## Step 2 — Intelligent optimization analysis

For **each function and logical block** in the file, follow this two-phase process before proposing anything:

### Phase A — Deep understanding

Read the block carefully and answer these questions internally before suggesting any change:
- What is this block's **abstract goal**? (what problem does it solve, not just what lines it runs)
- What are all **valid input cases**, including edge cases? (empty collections, single elements, duplicates, negatives, zeros, null/nil, max/min boundary values, already-optimal input, worst-case input)
- What is the **exact expected output or side effect** for each of those cases?
- What **invariants** must the optimized code preserve? (e.g., stable ordering, in-place mutation, specific error behavior)
- Are there any **subtle assumptions** baked into the current implementation? (e.g., relying on iteration order, short-circuit evaluation, mutable state)

Only after fully understanding the block, proceed to Phase B.

### Phase B — Optimization reasoning

Reason from first principles. Do not limit to a fixed list — consider any dimension relevant to the code:

- **Algorithmic complexity** — is there a fundamentally better algorithm for this problem class? (sorting, searching, graph traversal, string matching, combinatorics, etc.)
- **Data structure choice** — would a different container eliminate repeated scans, lookups, or insertions?
- **Redundant computation** — is the same value computed more than once within a loop or across calls when it could be precomputed or cached?
- **Loop structure** — can multiple sequential passes be merged into one? Can inner-loop invariants be hoisted out?
- **Recursion vs iteration** — does a recursive formulation recompute overlapping subproblems that dynamic programming or memoization handles in linear time?
- **Space vs time trade-offs** — is there an auxiliary structure that trades space for time, or can the code operate in-place to reduce space?
- **Language/library idioms** — does the language or its standard library offer a built-in that is asymptotically better than the manual implementation?

### Correctness gate (required — do not skip)

For every proposed optimization, explicitly verify each edge case before including it in the output:

- Empty input → same result?
- Single element → same result?
- All duplicate values → same result?
- Already-optimal input (e.g., already sorted) → same result?
- Reverse worst-case input → same result?
- Boundary values (INT_MAX, INT_MIN, 0, negative, overflow edge) → same result?
- Any special behavior the original code has that must be preserved?

If **any** edge case cannot be verified to produce identical output, **do not propose that optimization**.

### Output format for each verified opportunity

```
[N] <Function/block name>, lines <X>–<Y>
  Current complexity : time O(...), space O(...)
  After optimization : time O(...), space O(...)
  What it does       : <plain-English abstract goal of this block>
  Why it is slow     : <the specific inefficiency>
  Better approach    : <the more efficient algorithm or structure, and why it works>
  Edge cases checked : <list every edge case examined with the verdict for each>
  Proposed new code  :
    <the replacement code>
```

If **no** optimization opportunities exist, tell the user clearly and stop.

---

## Step 3 — Confirm optimizations to apply

Ask via `AskUserQuestion`:

> "I found [N] optimization opportunity(ies) above. Would you like me to apply all of them, or select specific ones?"

Options:
- **"Apply all"** → apply every proposed change
- **"Let me choose"** → list the opportunities by number and ask the user to pick
- **"No, just the analysis"** → stop here

---

## Step 4 — Apply the rewrites

Use the Edit tool to apply each confirmed rewrite precisely.

Rules:
- Change **only** the identified blocks — do not reformat, rename, or touch unrelated code.
- Add a one-line comment above each changed block noting the complexity improvement (e.g., `// O(n log n) via std::sort — was O(n²) bubble sort`).
- Apply all changes in a single pass per file.

---

## Step 5 — Verify correctness

For every applied change, verify that the new code produces identical output to the original:

1. **Compiled languages** (C/C++, Java, Go, Rust, etc.) — save the original file to a temp copy, compile and run both versions with the same representative inputs (including all edge cases from Step 2), then diff the outputs.
2. **Scripted languages** (Python, JavaScript, Ruby, etc.) — run both versions with representative inputs and compare stdout/return values.
3. **Cannot execute** (remote read-only, missing compiler/runtime) — reason symbolically and clearly state which assumptions were made instead of running the code.

Report:
- Whether outputs matched for all tested inputs
- Exactly which inputs were used (so the user can reproduce)
- Which edge cases were tested

If verification **fails** for any change:
- Revert that specific change using the Edit tool (restore the original lines)
- Report the discrepancy clearly
- Do not include that change in the commit

---

## Step 6 — Optionally commit to a branch

Ask via `AskUserQuestion`:

> "Verification passed. Would you like me to commit these optimized changes to an `optimize/<filename>` branch?"

Options:
- **"Yes, create branch and commit"** → run:
  ```bash
  git checkout main           # use master or the repo's default branch if main doesn't exist
  git pull origin main
  git checkout -b optimize/<base-filename>
  git add <file-path>
  git commit -m "perf(<filename>): <one-line summary of optimizations applied>"
  ```
  After the commit, output the branch names clearly so the user's other skills can use them:
  ```
  Source branch : main
  New branch    : optimize/<base-filename>
  ```
- **"No, keep changes in working tree"** → stop here; optimizations are already applied locally
