---
name: gh-review
description: Pull the latest Copilot (or other reviewer) review from a GitHub pull request and assess each comment for validity and a possible fix, WITHOUT changing any code. Use when the user says "new review", "check the review", "what did Copilot say", "review 141", or asks to look at PR review comments. Accepts an optional PR number; defaults to the PR for the current branch. Do NOT use to apply fixes — that only happens after the user explicitly confirms.
argument-hint: "[PR number] — optional; defaults to the current branch's PR"
---

# GitHub review triage (comment only, never fix)

Fetch the newest review on a PR, list every comment, and for each one state
whether it is valid and what the fix would be. **Do not edit any file.** The
user decides which items get fixed and will say so explicitly ("fix 2 and 4",
"do it", "a").

## Hard rules

1. **No code changes in this workflow.** Not even trivial ones. Not even if the
   comment is obviously right. Assessment only.
2. **Only the latest review.** Do not re-report comments from earlier reviews
   that were already handled unless they are re-raised.
3. **Never reply on GitHub.** Do not post PR comments, resolve threads, or
   react. Output goes to the user in chat only.
4. **Cover every comment.** Do not drop items; if a comment is a non-issue say
   so explicitly.

## Procedure

### 1. Identify the PR

If the user supplied a PR number (e.g. "review 141", "gh-review 141",
"check the review on #141"), use it directly. Otherwise use the PR for the
current branch:

```sh
# Explicit number
gh pr view PR --json number,url,headRefName,reviews --jq '{number,url,headRefName,reviews:[.reviews[]|{author:.author.login,state,submittedAt}]}'

# Current branch
gh pr view --json number,url,headRefName,reviews --jq '{number,url,headRefName,reviews:[.reviews[]|{author:.author.login,state,submittedAt}]}'
```

If the current branch has no PR and none was given, ask the user for the
number.

When the PR's `headRefName` differs from the checked-out branch, **checking out
the PR branch is the default**. Decide purely on tree state, never on which
branch happens to be active:

```sh
git status --porcelain          # empty => clean
```

- **Clean tree → `gh pr checkout PR`.** Do this before step 3. It is the only
  way to get language-server lookups, `grep` across the whole tree, and the
  ability to build/run tests when a verdict depends on runtime behaviour
  (e.g. signal connection types, ordering). Say in the report that you
  switched branches.
- **Dirty tree → do not switch.** Read the referenced code via the API using
  the PR **head SHA**, not the branch name (fork branches 404 on the upstream
  repo):

  ```sh
  SHA=$(gh api repos/{owner}/{repo}/pulls/PR --jq .head.sha)
  gh api "repos/{owner}/{repo}/contents/{path}?ref=$SHA" --jq .content | base64 -d
  ```

  Any cross-file reasoning (callers, signal connections, other tests) must
  also be checked against `$SHA`, never against the local checkout — the
  local branch may differ from the PR. Fetch every file the reasoning touches,
  not just the files the reviewer commented on. Note in the report that the
  local checkout was left alone and which assumptions could not be verified
  by build/test.

Being on a different branch is **not** by itself a reason to stay put; that
mistake was made on PR #15088 and degraded the analysis.

Pick the review with the newest `submittedAt` authored by
`copilot-pull-request-reviewer` (fall back to the newest review overall if the
user did not specify a reviewer).

### 2. Fetch the review body and its inline comments

```sh
# Summary body of the latest review
gh api repos/{owner}/{repo}/pulls/{pr}/reviews --jq 'sort_by(.submitted_at) | last | {id,user:.user.login,submitted_at,body}'

# Inline comments belonging to that review
gh api repos/{owner}/{repo}/pulls/{pr}/comments --paginate \
  --jq '[.[] | select(.pull_request_review_id == REVIEW_ID)]
         | .[] | {id, path, line: (.line // .original_line), body}'
```

Substitute `REVIEW_ID` from the first command. If the review has zero inline
comments, report the review body alone and stop.

### 3. Read the code each comment points at

For every comment, read the referenced file around the referenced line
(at least ±15 lines) so the assessment is grounded in the current code, not
the reviewer's paraphrase. Check whether the concern is already handled
elsewhere (a guard upstream, a test, a documented deviation).

### 4. List the review items

Before any assessment, print a numbered list of every inline comment so the
user can refer to items by number in discussion and in `gh-review-fix`:

```
## Review items (review <id>, <reviewer>, <submitted_at>)
1. path/file.ext:LINE — <reviewer's comment, quoted or tightly paraphrased>
2. path/file.ext:LINE — ...
```

Order by file path then line. Numbering is fixed for the rest of the
conversation; never renumber when discussing or fixing.

### 5. Assess each item

Under the list, one entry per item using the same numbers:

```
### N. path/file.ext:LINE
Verdict: VALID | PARTIALLY VALID | INVALID | STYLE/OPTIONAL
Why: <1–3 sentences grounded in the code you read>
Fix: <what the change would be, concretely; or "none needed">
Risk: <what the fix could break / tests that would need updating; or "low">
```

Then a short **Summary** line: how many valid / partial / invalid, and which
items you recommend fixing. If two comments would be fixed by the same change,
say so.

End with: *"No files changed. Tell me which items to fix or discuss."*

## Verdict guidance

- **VALID** — the reviewer is right and the code should change.
- **PARTIALLY VALID** — real concern, but the proposed fix is wrong or the
  scope is narrower than claimed. Give the correct fix.
- **INVALID** — the concern does not apply (already handled, misread code,
  contradicts a deliberate/documented decision). Cite the line or doc that
  shows this.
- **STYLE/OPTIONAL** — harmless either way; note if it conflicts with the
  file's existing conventions (e.g. the module already uses `sys.exit` in
  helpers throughout).

Be blunt. Copilot reviews frequently include false positives; do not soften an
INVALID verdict to be polite, and do not inflate a nit into VALID.

## After the user confirms

Only then, and only for the items named: make the edits, run the relevant
tests, and follow the repository's commit conventions (typically amend the
single PR commit and `git push --force-with-lease`). Wait for a separate
go-ahead before pushing if the user has not already given it.
