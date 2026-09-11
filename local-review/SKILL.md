---
name: local-review
description: Review all uncommitted changes in the current repository (staged, unstaged, and untracked files) in one systematic pass and report numbered findings in chat, WITHOUT changing any code. Use when the user says "review", "review my changes", "review my code", "local review", or "look over the diff" and there is no GitHub PR review involved. Do NOT use for triaging GitHub PR review comments (use gh-review) or for applying fixes.
---

# Local review of uncommitted changes (comment only, never fix)

Review everything pending in the working tree and report numbered findings
in chat. **Do not edit any file.** The user decides which items get fixed and
will say so explicitly ("fix 2 and 4", "do it").

## Hard rules

1. **No code changes, commits, or PR comments** unless explicitly asked
   afterward. Assessment only.
2. **Scope is everything pending**: staged, unstaged, and untracked/new files.
   Never leave untracked files unreviewed.
3. **Verify before flagging.** Read the full surrounding function/file — do not
   flag something a later line already handles.
4. **If the diff is clean, say so.** Do not invent findings.

## Procedure

### 1. Gather the diff

```bash
git status --short
git diff HEAD
```

Read new untracked files directly (`??` entries in `git status --short`).

### 2. Review systematically by category

Work through the diff category by category — do not just scan for typos:

1. **Correctness** — logic errors, edge cases, off-by-one, unsigned
   underflow/overflow, uninitialized state
2. **Resource/state lifecycle** — leaks, stale caches, missing cleanup,
   signal connect/disconnect balance
3. **Threading and re-entrancy**
4. **Error handling and failure paths**
5. **API misuse and project-convention violations** — check repo
   instructions (AGENTS.md, CODING_STYLE.md, copilot-instructions) and apply
   their rules
6. **Tests** — missing coverage for new behavior, fragile assertions

Flag issues even if they are outside the lines directly changed but affected
by them.

### 3. Report

- **Number each finding** (1, 2, 3, …) so it can be referenced in follow-up
  discussion.
- **Every finding links to the exact source line(s)** as a markdown file link,
  e.g. `[file.cc](src/path/file.cc#L42)`. Look up real line numbers first —
  never quote code blocks instead of linking.
- **Order by severity**: bugs first, then robustness, then style/consistency.
- If a finding is speculative or low-risk, say so explicitly.
- Do not pad with praise or restate what the diff does.

## After the report

When the user asks to explain a finding: explain the issue, then ASK before
fixing. Never explain-and-fix in the same turn. When the user asks to fix
specific items: make the code change only — do not commit or push until told.
