---
name: address-pr-comments
description: This skill should be used when the user asks to "address PR comments", "respond to review comments", "handle reviewer feedback", "fix PR review feedback", "go through review comments", or any similar request to process open GitHub pull request review threads on the current branch.
version: 1.0.0
---

# Address PR Review Comments

Fetch open review threads, let the user decide how to handle each, implement accepted changes, commit per comment, reply on GitHub, and resolve accepted threads.

## Steps

- [ ] **1. Preflight: verify `gh` is available and authenticated**
  ```bash
  gh auth status
  ```
  - If `gh` is not installed → stop and tell the user:
    > `gh` CLI is required. Install it with `brew install gh` (macOS) or visit https://cli.github.com, then run `gh auth login`.
  - If `gh` is installed but not authenticated → stop and tell the user:
    > Not logged in to GitHub CLI. Run `gh auth login` and try again.
  Only continue once both checks pass.

- [ ] **2. Verify an open PR exists for the current branch**
  ```bash
  gh pr view --json number,url 2>/dev/null
  ```
  - If this fails or returns nothing → stop and tell the user:
    > No open PR found for the current branch. Create one first with the `pr` skill.
  - Capture `PR_NUMBER` and `PR_URL` for use in later steps.

- [ ] **3. Get the repo owner and name**
  ```bash
  gh repo view --json nameWithOwner --jq '.nameWithOwner'
  ```
  Capture as `OWNER_REPO` (e.g. `acme/my-repo`). Split on `/` to get `OWNER` and `REPO` separately.

- [ ] **4. Fetch all unresolved review threads via GraphQL**
  Run the following query (substituting actual values for `OWNER`, `REPO`, and `PR_NUMBER`):
  ```bash
  gh api graphql -f query='
  query {
    repository(owner: "OWNER", name: "REPO") {
      pullRequest(number: PR_NUMBER) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 20) {
              nodes {
                databaseId
                author { login }
                path
                line
                originalLine
                diffHunk
                body
              }
            }
          }
        }
      }
    }
  }
  '
  ```
  Filter to only threads where `isResolved: false`. For each unresolved thread build a mapping entry:
  - `threadNodeId` — the thread `id` field (e.g. `PRRT_kwDOMIX3f...`); used for the resolve mutation
  - `firstCommentDatabaseId` — the integer `databaseId` of `comments.nodes[0]`; used for `in_reply_to` on reply POSTs
  - `reviewer` — `comments.nodes[0].author.login`
  - `path` — `comments.nodes[0].path`
  - `line` — `comments.nodes[0].line` (may be `null` if the diff line is outdated — display as "line unknown")
  - `diffHunk` — `comments.nodes[0].diffHunk`
  - `comments` — full `comments.nodes` array (to show multi-comment reply chains)

  **Fallback:** If the GraphQL query fails (auth scope, rate limit, etc.) fall back to the REST endpoint to at least get comment bodies:
  ```bash
  gh api "repos/OWNER/REPO/pulls/PR_NUMBER/comments"
  ```
  In fallback mode, thread resolve will not be available — skip Step 8f and warn the user.

- [ ] **5. Handle the "no open comments" case**
  If there are zero unresolved threads after filtering → tell the user:
  > No unresolved review threads found on this PR. All comments have already been resolved.
  Then stop.

- [ ] **6. Display a numbered list of unresolved threads**
  Format all unresolved threads like this example:
  ```
  Found 3 unresolved review threads on PR #42:

  [1] @reviewer — src/app/foo.py:18
      > "Consider renaming this variable for clarity."

  [2] @reviewer — migrations/0010-index.sql (line unknown)
      > "The lookup query also filters role='user', should be reflected here."
      > @you (reply): "Good catch, will fix."

  [3] @reviewer — src/app/bar.py:55
      > "Dead code — is anything using this?"
  ```
  If a thread has multiple comments (a reply chain) show all of them indented under the thread entry.

- [ ] **7. Ask: bulk or one-by-one?**
  Use `AskUserQuestion`:
  > How would you like to address these N comment(s)?
  > - **one-by-one** — I'll walk you through each and ask what you want to do (accept, push back, or ask a question)
  > - **bulk** — I'll treat all as accepted and implement changes automatically

  Capture the answer as `MODE`.

- [ ] **8. Process each unresolved thread**
  Iterate through the numbered list from Step 6. For each thread:

  **8a. Display full context**
  Show the thread number, reviewer, `path:line` (or "line unknown"), the `diffHunk` (if non-empty, as a fenced code block), and all comment bodies in the chain.

  **8b. Determine the action**
  - If `MODE = one-by-one`: use `AskUserQuestion`:
    > Comment [N] from @reviewer on `path:line`:
    > "`comment body`"
    >
    > What would you like to do?
    > - **accept** — implement the suggested change
    > - **push back** — disagree; I'll ask for your reasoning
    > - **question** — ask the reviewer something; I'll ask what you want to say

    - If **push back**: use `AskUserQuestion` again: "What is your reasoning? This will be posted as a reply." Capture as `reply_text`.
    - If **question**: use `AskUserQuestion` again: "What question would you like to ask?" Capture as `reply_text`.
    - If **accept**: `ACTION = accept`.

  - If `MODE = bulk`: set `ACTION = accept` for all threads without prompting.

  **8c. If `ACTION = accept`: implement the code change**
  Read the file at `path`. Use the `diffHunk` and comment body to understand what change is needed. Apply the edit with your editing tools. Only change lines within the scope of the comment — do not refactor surrounding code.

  If the file at `path` no longer exists in the working tree, use `AskUserQuestion` to ask the user how to proceed before editing.

  **8d. If `ACTION = accept`: commit the change**
  ```bash
  git add <affected files>
  git commit -m "PR Feedback: adjusted <what> based on @<reviewer>'s comment"
  ```
  - `<what>` is a concise 2–5 word phrase describing what was changed (e.g. "variable rename", "dead code removal", "SQL role filter")
  - One new commit per accepted comment — never amend a previous commit

  **8e. Post a reply to the thread**
  Always post a reply, regardless of action:
  ```bash
  gh api "repos/OWNER/REPO/pulls/PR_NUMBER/comments" \
    -X POST \
    -f body="REPLY_BODY" \
    -F in_reply_to=FIRST_COMMENT_DATABASE_ID
  ```
  - `in_reply_to` must be the integer `databaseId` (not the GraphQL node ID)
  - For **accept**: write a concise reply such as "Addressed — renamed the variable to `checkin_uuid` for clarity."
  - For **push back**: use the `reply_text` from Step 8b
  - For **question**: use the `reply_text` from Step 8b

  If the POST fails (e.g. comment was deleted), warn the user and continue with the next thread.

  **8f. If `ACTION = accept`: resolve the thread**
  ```bash
  gh api graphql -f query='
  mutation {
    resolveReviewThread(input: {threadId: "THREAD_NODE_ID"}) {
      thread { isResolved }
    }
  }
  '
  ```
  - `THREAD_NODE_ID` is the `threadNodeId` captured in Step 4
  - Do **not** resolve threads where the action was push back or a question — leave them open for the reviewer
  - If the mutation fails (e.g. insufficient permissions), warn the user: "Could not auto-resolve thread [N] — you may need to resolve it manually on GitHub." Then continue.

- [ ] **9. Final summary**
  After processing all threads, output a table:
  ```
  Done. PR review summary:

  [1] @reviewer / src/app/foo.py:18        → accepted + committed + resolved
  [2] @reviewer / migrations/0010.sql      → accepted + committed + resolved
  [3] @reviewer / src/app/bar.py:55        → pushed back — left open

  PR: https://github.com/OWNER/REPO/pull/PR_NUMBER
  ```
  Then output the PR URL:
  ```bash
  gh pr view --json url --jq '.url'
  ```
