---
name: github-resolve-pr
description: "Use when the user asks to review, analyze, address, or resolve comments pushed on a GitHub pull request - fetches all unresolved review threads, judges each comment on its merits, applies a fix when the comment is valid, always replies to every thread (fix summary or rationale for not fixing), and resolves every thread after the reply is posted — whether the comment was fixed or declined with rationale"
---

# GitHub Resolve PR Comments

## Overview

When a reviewer (human or bot such as Copilot, CodeRabbit, Sourcery) leaves comments on a PR, the author must close the loop on every comment — not just the ones that are trivially actionable. This skill defines a consistent workflow: fetch unresolved threads, judge each one on technical merit, apply a fix when warranted, and always leave an explicit reply (either a fix summary or a rationale for declining). Resolve every thread after the reply is posted — fixed ones once the commit is pushed, declined ones once the rationale is on the record. The reply is what preserves the conversational trail; resolution just hides the thread from the default view so remaining work is visible.

Silence is not acceptable. A thread with no reply leaves the reviewer guessing whether the feedback was seen, rejected, or forgotten. Resolving without replying is the same defect in disguise.

## When to Use

- The user asks you to "review", "analyze", "address", "look at", "resolve", or "handle" the comments on a PR
- A reviewer (human or bot) has posted comments and the user wants them handled
- After pushing changes, the user wants remaining unresolved threads triaged
- The user mentions a specific reviewer ("look at what Copilot said", "CodeRabbit flagged things")

## Do NOT Use When

- Writing a brand-new PR (use `github-create-update-pr`)
- Cleaning up after a PR is merged/closed (use `github-complete-pr`)
- The user explicitly asks to *ignore* review comments or to merge without addressing them

## Rules

### 1. Fetch all unresolved review threads

Use the GraphQL API to get thread IDs alongside comment bodies — REST endpoints alone don't expose the `isResolved` state or the thread node ID needed to resolve later.

```bash
gh api graphql -f query='
query {
  repository(owner: "<OWNER>", name: "<REPO>") {
    pullRequest(number: <PR>) {
      reviewThreads(first: 50) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id
          isResolved
          comments(first: 20) {
            pageInfo { hasNextPage endCursor }
            nodes {
              databaseId
              author { login }
              path
              originalLine
              body
            }
          }
        }
      }
    }
  }
}'
```

Capture for each unresolved thread: `thread.id` (for the resolve mutation), the latest comment's `databaseId` (for replies via REST), the `path` and `originalLine` (for locating the code), and the `body` (the feedback itself).

**Pagination matters on large PRs.** The query above caps at 50 threads and 20 comments per thread. If `reviewThreads.pageInfo.hasNextPage` is `true`, you are missing threads — re-run with `after: "<endCursor>"` (via a `$threadsCursor` variable) and merge pages until `hasNextPage` is `false`. Likewise, if any thread's `comments.pageInfo.hasNextPage` is `true`, paginate that thread's comments before replying so you are seeing the latest turn of the conversation, not a stale mid-thread snapshot. See the paginated snippet in the Full Recipe below.

### 2. Skip protected paths — do not auto-resolve

Some file paths are considered stable after definition and must not be auto-resolved. Comments on these paths should remain open for manual handling.

**Protected paths:**

- `docs/superpowers/**`

When an unresolved thread targets a file matching a protected path:

1. **Do not apply any fix** to the file.
2. **Do not resolve** the thread.
3. **Post an explicit reply on the PR thread** explaining that the path is protected, no automatic code change was made, and the thread is being left open for manual handling:
   ```
   This path is protected (`docs/superpowers/**`), so I did not apply an automatic fix here. Leaving this thread open for manual review.
   ```
4. **Optionally log / output a note** for local visibility:
   ```
   ⏭️ Skipping thread <thread-id> — targets protected path docs/superpowers/<file>. Replied on the thread and left it open for manual review.
   ```
5. Move on to the next thread.

### 3. Judge each comment on technical merit

For every unresolved thread (that was not skipped in §2), make a deliberate decision:

- **Valid** — the comment identifies a real bug, ambiguity, inconsistency, security issue, or documentation defect. Fix it.
- **Invalid** — the comment is wrong, based on a misreading, proposes something that conflicts with an explicit requirement, or suggests a change with worse tradeoffs than the current code. Do not fix.
- **Partially valid** — the concern is real but the suggested fix is wrong, or only part of the comment is actionable. Fix what is correct; explain what is not.

Do not auto-accept bot suggestions just because they come from a bot. Do not auto-reject human feedback just because you disagree at first glance. Read the code around `path:originalLine` before deciding.

### 4. Always reply to every thread

Every unresolved thread must receive a reply before being resolved — regardless of the decision. The reply must:

- **If fixed**: include the commit SHA and a one-or-two-sentence summary of what was changed. Do not just say "fixed" — say *what* was fixed and *how*, so the reviewer can verify without re-reading the diff.
- **If not fixed**: state the rationale. Reference the specific constraint, requirement, or tradeoff that overrides the suggestion. "Disagree" or "won't fix" alone is not sufficient.
- **If partial**: describe the fix applied and explain why the rest was declined.

```bash
gh api -X POST \
  repos/<OWNER>/<REPO>/pulls/<PR>/comments/<COMMENT_DATABASE_ID>/replies \
  -f body='<reply text>'
```

### 5. Apply fixes on the PR branch, not on main

Work on the PR's head branch. Confirm with `git branch --show-current` before editing. If you are on the default branch, check out the PR branch (`gh pr checkout <PR>`).

### 6. Commit fixes with a descriptive conventional commit

Use the `git-commit` skill. Prefer one commit per logically-distinct fix over a single "address review comments" mega-commit — but don't split trivial wording fixes across multiple commits either. A commit that covers two related typos is fine; a commit that covers a shell-logic fix and a documentation restructure is not.

Commit message examples:

```
fix(skills): use explicit if/else for remote branch deletion
fix(api): surface push errors instead of swallowing them as "already deleted"
docs(readme): clarify installation prerequisites per review feedback
```

Reference the reviewer concern in the commit body, not the title.

### 7. Push before replying

Replies carry the fix commit SHA. Push first, then reply — the SHA must resolve on the remote when the reviewer clicks it.

```bash
git push
```

### 8. Resolve every thread after reply + push

Use the `resolveReviewThread` GraphQL mutation with the `thread.id` captured in step 1:

```bash
gh api graphql -f query='
mutation($id: ID!) {
  resolveReviewThread(input: { threadId: $id }) {
    thread { id isResolved }
  }
}' -f id=<THREAD_NODE_ID>
```

Order matters:

- **Fixed threads**: `fix → push → reply → resolve`. Replying before pushing creates a broken SHA reference.
- **Declined threads**: `reply (with rationale) → resolve`. The rationale stays in the thread history; resolution just hides it from the default "unresolved" view so the remaining work is visible.

Resolve every thread this way — fix or decline — **as long as a substantive reply was posted first**. Resolution without a reply is forbidden (see Rule §9); it looks like feedback is being silenced.

### 9. Never resolve without replying first

Resolving a thread that has no author reply is silencing feedback, regardless of whether you intended to fix it or decline it. The reply is what makes the decision transparent to the reviewer; resolution just tidies the UI.

If you cannot articulate a rationale for declining, that is a signal the comment probably warrants a fix — do not resolve to avoid the conversation.

**Reviewer-flagged disagreements.** If the reviewer pushes back on your rationale and re-opens the thread (or posts a counter-argument), treat that as a fresh unresolved thread: read it, reply again with an updated decision, and only then resolve.

## Full Recipe

```bash
OWNER=<owner>
REPO=<repo>
PR=<number>

# 1. Fetch unresolved threads, paginating through all review-thread pages.
#    If any thread reports comments.pageInfo.hasNextPage=true, paginate that
#    thread's comments before replying so you see the latest turn.
THREADS_CURSOR=null
while :; do
  page=$(gh api graphql \
    -F owner="$OWNER" \
    -F repo="$REPO" \
    -F pr="$PR" \
    -F threadsCursor="$THREADS_CURSOR" \
    -f query='
      query($owner: String!, $repo: String!, $pr: Int!, $threadsCursor: String) {
        repository(owner: $owner, name: $repo) {
          pullRequest(number: $pr) {
            reviewThreads(first: 50, after: $threadsCursor) {
              pageInfo { hasNextPage endCursor }
              nodes {
                id
                isResolved
                comments(first: 20) {
                  pageInfo { hasNextPage endCursor }
                  nodes { databaseId author { login } path originalLine body }
                }
              }
            }
          }
        }
      }')

  echo "$page"

  has_next=$(echo "$page" | jq -r '.data.repository.pullRequest.reviewThreads.pageInfo.hasNextPage')
  [ "$has_next" != "true" ] && break
  THREADS_CURSOR=$(echo "$page" | jq -r '.data.repository.pullRequest.reviewThreads.pageInfo.endCursor')
done

# 2. For each unresolved thread: read the body, inspect the code, decide valid/invalid.

# 3. Apply fixes on the PR branch.
gh pr checkout "$PR"
# ... edit files ...

# 4. Commit and push.
git add <files>
git commit -m "<conventional commit message referencing the review>"
git push

# 5. Reply to each thread (use the latest comment's databaseId).
gh api -X POST \
  "repos/$OWNER/$REPO/pulls/$PR/comments/<COMMENT_DATABASE_ID>/replies" \
  -f body="Fixed in <short-sha>. <one-sentence summary of the change>"

# 6. Resolve every thread for which you posted a reply (fix or decline).
gh api graphql -f query='
mutation($id: ID!) {
  resolveReviewThread(input: { threadId: $id }) {
    thread { id isResolved }
  }
}' -f id=<THREAD_NODE_ID>
```

## Red Flags — STOP

- About to resolve a thread without replying → reply first; silent resolution is the one thing this skill forbids
- About to reply before pushing a fix → push first so the SHA is valid
- About to accept every bot suggestion without reading the surrounding code → read first, decide second
- About to decline a comment and resolve with "won't fix" and nothing else → write a rationale the reviewer can evaluate, then resolve
- About to batch fixes across unrelated threads into a single "address review" commit → split by concern
- Comment references a file/line that no longer exists because the diff moved → re-read the current file at `path` before assuming the feedback is stale
- About to resolve or fix a comment on a protected path (`docs/superpowers/**`) → skip it and leave the thread open for manual handling
