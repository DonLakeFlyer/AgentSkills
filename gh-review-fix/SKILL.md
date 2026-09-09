---
name: gh-review-fix
description: Apply fixes for numbered items from a preceding gh-review triage — edit, run the relevant tests/build, amend into the PR's existing commit, force-push-with-lease, hide the addressed Copilot review as resolved, and request a fresh Copilot review. Use when the user says "fix N and M", "fix all valid", "apply the review fixes", or "do it" after a gh-review report. Requires a gh-review report in the current conversation; if there is none, run gh-review first.
---

# GitHub review fix cycle

Second half of the review loop. `gh-review` assessed the comments; this skill
acts on the ones the user picks and then closes the loop on GitHub.

## Preconditions

- A `gh-review` report with numbered items exists in this conversation. If it
  does not, stop and run `gh-review` first — never fix from memory of an older
  review.
- Working tree is clean apart from changes you are about to make
  (`git status --short`). If it is dirty, ask before proceeding.

## Procedure

### 1. Ask which items to fix

If the user's message does not already name the items (e.g. "fix 1, 3, 4",
"fix all valid", "a"), ask with a single question listing the numbered items
and their verdicts from the report. Do not start editing until the selection
is unambiguous. "All" means every VALID and PARTIALLY VALID item; INVALID
items are never fixed unless named explicitly.

### 2. Fix the selected items

- Re-read each target file before editing; the report may be minutes old.
- Apply exactly the fix described in the report for that item (or the user's
  amended version). Do not fold in unrelated cleanups or touch unselected
  items.
- If two items share a fix, do it once and note both numbers.
- Keep repository conventions (style, comment density, naming).

### 3. Verify

Run what the change touches, from the repo root:

| Changed area | Command |
| --- | --- |
| Python (`analyzer/`, `detector/`, `simulator/`) | `.venv/bin/python -m pytest <affected test dirs>` |
| C++ controller | `cmake --build build --target MavlinkTagController2` then `ctest --test-dir build` |
| Decimator / shared wire format | `cmake --build build` then `ctest --test-dir build` |
| Docs / markdown only | none, but re-read the rendered section |

If a fix changes behaviour a test asserts, update the test in the same edit.
Do not proceed with failing tests; fix or report back.

### 4. Amend into the PR commit

The PR is always **one commit**. Never create a second one.

```sh
git add <explicit paths only>          # never -A or .
git commit --amend --no-edit
git push --force-with-lease
git --no-pager log --oneline -1 && git --no-pager status --short
```

If the PR unexpectedly has more than one commit, stop and ask which commit to
amend into rather than guessing.

### 5. Reply to and resolve every review thread

Unresolved threads keep showing up in the Files-changed view even after the
review is hidden, so close out **every** inline comment from the triaged
review — fixed or not:

- **Fixed items:** reply with one line, e.g. `Fixed in <short-sha>.`
- **Unfixed items** (INVALID / STYLE / user declined): reply with the one-line
  reason from the gh-review verdict, e.g. `Not changing — data_file is
  informational only; psd_spectrum.py never reads it.`

Then resolve the thread. Reply via REST, resolve via GraphQL (the only API
that exposes thread resolution):

```sh
# Reply (COMMENT_ID = inline comment id from gh-review step 2)
gh api -X POST repos/OWNER/REPO/pulls/PR/comments/COMMENT_ID/replies -f body='Fixed in abc1234.'

# Map comment ids -> thread ids
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!){
    repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
      reviewThreads(last:50){ nodes{ id isResolved
        comments(first:1){ nodes{ databaseId } } } } } } }' \
  -f owner=OWNER -f repo=REPO -F pr=PR \
  --jq '.data.repository.pullRequest.reviewThreads.nodes[] | select(.isResolved==false) | {thread:.id, comment:.comments.nodes[0].databaseId}'

# Resolve each thread
gh api graphql -f query='
  mutation($id:ID!){ resolveReviewThread(input:{threadId:$id}){
    thread{ isResolved } } }' -f id=THREAD_NODE_ID
```

Only touch threads whose first comment belongs to the triaged review; leave
threads from other reviewers or earlier reviews alone.

### 6. Hide the old review as resolved

Once every thread from step 5 is resolved, minimize the triaged review itself
so its summary disappears from the PR timeline (the "Hide → Resolved" action
in the UI). Skip this if any VALID item from the review was deliberately left
unfixed.

```sh
# Node id of the review triaged by gh-review (match on databaseId)
gh api graphql -f query='
  query($owner:String!,$repo:String!,$pr:Int!){
    repository(owner:$owner,name:$repo){ pullRequest(number:$pr){
      reviews(last:5){ nodes{ id databaseId isMinimized } } } } }' \
  -f owner=OWNER -f repo=REPO -F pr=PR

gh api graphql -f query='
  mutation($id:ID!){ minimizeComment(input:{subjectId:$id,classifier:RESOLVED}){
    minimizedComment{ isMinimized minimizedReason } } }' -f id=REVIEW_NODE_ID
```

### 7. Request a fresh Copilot review

The repository's default Copilot review effort is **Balanced**, so re-request
via API without asking:

```sh
gh api -X POST repos/OWNER/REPO/pulls/PR/requested_reviewers \
  -f 'reviewers[]=copilot-pull-request-reviewer[bot]'
```

Copilot clears its pending-reviewer slot as soon as it starts, so
`gh pr view --json reviewRequests` may read empty. Confirm via the issue event
log instead:

```sh
gh api repos/OWNER/REPO/issues/PR/events \
  --jq '.[] | select(.event=="review_requested") | {created_at, requested_reviewer:.requested_reviewer.login}' | tail -1
```

Do not use `gh pr edit --add-reviewer` (GraphQL; fails on Projects-classic
deprecation).

### 8. Report

One short block: items fixed (numbers), tests run and result, new commit
hash, threads replied/resolved, review hidden (or why not), and that a new
Copilot review was requested. Then stop — the next `gh-review` run picks up
the new review.

## Hard rules

- Never fix items the user did not select.
- Never create a new commit; always amend.
- Never `git add -A`, `git add .`, or push without `--force-with-lease`.
- Resolve every thread from the triaged review, each with a one-line reply
  (fixed / why not). Never resolve a thread silently.
- Never hide a review while a VALID item from it is still open.
- Replies are limited to the one-liners in step 5; never post anything else on
  the PR.
