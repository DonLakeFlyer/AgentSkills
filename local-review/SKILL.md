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
6. **Duplication within the diff** — repeated blocks, near-identical
   functions/branches, or copy-pasted QML/C++ that could collapse into one
   helper, loop, or shared control. Name the concrete collapse (which
   occurrences, into what).
7. **Duplication against the existing codebase** — search the repo
   (`grep_search`/`semantic_search`) for existing helpers, controls, or
   utilities that already do what new code reimplements, and for existing
   code elsewhere that does what a new piece of code does. Report both
   directions: new code that should reuse something existing, and new code
   that is general enough to become a reusable component other parts of the
   code should adopt (name those call sites).
8. **Tests** — missing coverage for new behavior, fragile assertions

Flag issues even if they are outside the lines directly changed but affected
by them.

### 3. Report

- **Number each finding** (1, 2, 3, …) so it can be referenced in follow-up
  discussion. Numbering is fixed for the rest of the conversation; never
  renumber when discussing or fixing.
- **Every finding that references code gets a clickable code pointer** to the
  exact source line, as a markdown link with the **absolute file path and a
  1-based line number**, e.g. `[resolveResult](/abs/path/to/file.cc:42)` or
  `[file.cc](/abs/path/to/file.cc:42)`. Do not use relative paths or `#L42`
  anchors; those are not clickable in the editor. Use `/` path separators,
  link text that is the file name or symbol, and no line ranges. Look up real
  line numbers first, and use a link every time you mention a file or symbol,
  not just the first. Never quote code blocks instead of linking.
- **Order by severity**: bugs first, then robustness, then style/consistency.
- Do not pad with praise or restate what the diff does.

One entry per finding:

```
### N. [file.ext:LINE](/abs/path/to/file.ext:LINE) — <short title>
Severity: BUG  ("fix 2 and 4", or "do it" meaning every item marked FIX):
make the code change only, run the relevant tests — do not commit or push

Why: <1–3 sentences grounded in the code you read; say so if speculative>
Fix: <what the change would be, concretely>
Risk: <what the fix could break / tests that would need updating; or "low">
Recommendation: FIX | OPTIONAL | SKIP — <one-line reason>
```

Recommendation guidance:

- **FIX** — a real defect, a likely false positive/negative in behaviour the
  user relies on, or a cheap change that removes clear confusion.
- **OPTIONAL** — harmless either way; worth doing only if already touching
  the code.
- **SKIP** — the cost (refactor, new test harness, churn) outweighs the
  benefit, or the concern is speculative. Say what would change the call.

Then a short **Summary**: counts by severity and the list of items you
recommend fixing (e.g. "Recommend fixing 1, 2, 4; skip 3"). If two findings
would be fixed by the same change, say so.

End with: *"No files changed. Tell me which items to fix or discuss."*

## After the report

When the user asks to explain a finding: explain the issue, then ASK before
fixing. Never explain-and-fix in the same turn. When the user asks to fix
specific items: make the code change only — do not commit or push until told.
