# GitHub plumbing

Read PR state from GitHub every time. Local notes hold reasoning only, never
thread state. Every write is visible to other people, so show the user the
exact payload before sending it.

## Commands

- **One plain command per Bash call.** Each call is a fresh shell, so write
  every resolved value (owner, repo, number, SHA, path) literally. No shell
  variables, `$(...)`, pipes or `&&`.
- **Temp files go in one directory,** made with `mktemp -d` at the start of
  the run. Reuse its literal path, shown below as `<tmp>`. Never write into
  the repo.
- **Bodies go through a file,** written with the Write tool. Never put free
  text in double quotes: backticks in it run under zsh.

## Identify the PR

```bash
gh pr view <PR> --json number,url,headRefName,headRefOid,baseRefName,author,isDraft,state
gh api user --jq .login
```

`<PR>` is the number or URL the user gave. With none, `gh` picks the current
branch's PR. The second command names the viewer, to tell own threads from
others'.

Take `<owner>` and `<repo>` from `url`
(`https://github.com/<owner>/<repo>/pull/<N>`). From then on:

- pass `-R <owner>/<repo>` to every `gh pr` and `gh issue` command;
- write `<owner>` and `<repo>` literally into `gh api` paths and fields.
  Never use the `{owner}` and `{repo}` placeholders: they resolve to the
  current directory's repo, which may not be the PR's;
- key PR notes as `<owner>/<repo>#<N>`.

## Review threads

The thread ID (`PRRT_…`) is the stable key for notes. Use `fullDatabaseId`
for REST comment IDs: `databaseId` is deprecated and can overflow.

```bash
gh api graphql --paginate -f owner='<owner>' -f repo='<repo>' -F pr=<N> -f query='
query($owner: String!, $repo: String!, $pr: Int!, $endCursor: String) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100, after: $endCursor) {
        pageInfo { hasNextPage endCursor }
        nodes {
          id isResolved isOutdated path line originalLine diffSide subjectType
          viewerCanReply viewerCanResolve resolvedBy { login }
          comments(first: 50) {
            totalCount
            nodes { fullDatabaseId author { login } body path line url createdAt }
          }
        }
      }
    }
  }
}' --jq '.data.repository.pullRequest.reviewThreads.nodes[]'
```

`--paginate` walks `reviewThreads` only. When a thread's
`comments.totalCount` exceeds the nodes returned, page its comments by
thread ID:

```bash
gh api graphql --paginate -f id='<PRRT_id>' -f query='
query($id: ID!, $endCursor: String) {
  node(id: $id) { ... on PullRequestReviewThread {
    comments(first: 100, after: $endCursor) {
      pageInfo { hasNextPage endCursor }
      nodes { fullDatabaseId author { login } body url createdAt }
    } } } }' --jq '.data.node.comments.nodes[]'
```

- The first comment opens the thread; its author is the commenter.
- **A thread is done when `isResolved` is true,** whoever resolved it.
  `resolvedBy` is a weak signal: PR authors often resolve, and bots never do.
- An unresolved thread whose last comment is the viewer's is waiting on the
  commenter.
- `line` is null on an outdated thread. Fall back to `originalLine`; `path`
  still holds.

## Reply to a thread

Write the body to a file under `<tmp>`, then reply to the thread's first
comment. The reply posts at once.

```bash
gh api --method POST repos/<owner>/<repo>/pulls/<N>/comments/<first comment id>/replies -F body=@<tmp>/reply-1.md
```

The comment ID is the first comment's `fullDatabaseId`. The response's
`html_url` is the reply's URL.
