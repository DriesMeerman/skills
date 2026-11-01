# Review and follow-up

Shared by [start-work](SKILL.md) and [start-group](../start-group/SKILL.md). The agent that
wrote the PR runs these steps from the PR's own worktree, after the PR is open as a draft
(`gh pr create --draft`) and pushed. When CI does not run on pull requests, the author's
local tests and the merge-ready train are the checks.

PR comments hold review content only: findings, fixes, decisions. How the review ran, tool
errors, sandbox limits, timeouts and retries go in the final report to the orchestrator or
the user, never on the PR.

Below, `<n>` is the PR number and `<out>` is the agent's scratchpad subfolder. Set the
reviewer to `sol-medium`, `sol-high` or `opus`. Only Sol and Opus are allowed. Never use
Fable or Astra. `gh api` fills in `{owner}` and `{repo}` from the current checkout.

## 1. Run the review

`<skills>` is the folder this skill was loaded from, such as `.agents/skills` or
`~/.claude/skills`.

```sh
review=$(<skills>/start-work/review.sh <reviewer> <n> <out>); echo "exit=$?"
```

It can take up to 30 minutes, and a foreground Bash call stops waiting after 10. Start
it as a background Bash command, wait for its completion notice, then read the exit status
and the review file. Do nothing else on the PR until it has finished. In T3 Code, wait as
[t3-code.md](t3-code.md) says instead.

A subagent gets no completion notice once its turn ends, and its background commands die
with the turn. A subagent stays in its turn until the review is done. It writes the exit
status to a file and checks for it with foreground calls of under 10 minutes each, repeated
until the status is there:

```sh
# background
<skills>/start-work/review.sh <reviewer> <n> <out> > <out>/review-run.txt 2>&1; echo "exit=$?" >> <out>/review-run.txt
# foreground, repeat until it prints the exit line
perl -e 'alarm 540; exec @ARGV' sh -c 'until grep -q "^exit=" <out>/review-run.txt; do sleep 20; done'; tail -2 <out>/review-run.txt
```

The last two lines are the review file's path and `exit=<status>`.

The script:

- refuses unless `HEAD` is the PR head and the worktree is clean (exit 2)
- refuses a head it already reviewed (exit 3). Review once per pushed head. Run again only
  when the user asks after new pushes, with `REVIEW_AGAIN=1` for the same head.
- takes `--since <sha>` as a last argument to review only `git diff <sha>...HEAD`, with the
  same model, settings and guide. Use it for a re-review after a train fix or after commits
  that came later than the posted review. Without it the whole PR is reviewed.
- fetches the PR's base branch, so a stacked PR is diffed against its own base
- runs the reviewer without API keys in its environment, so Codex uses its ChatGPT login
  and Claude its claude.ai login. Nothing reads or copies a token.
- tells the reviewer to read the project's review guide first when one exists (set
  `REVIEW_GUIDE=<path>` to name it, otherwise it looks in the places
  [project-rules.md](project-rules.md) lists), to read the diff and code only, and not to
  run tests, builds or anything that needs the network or a port. The author runs the tests.
- stops the reviewer after 30 minutes
- writes the review to `<out>/review-<reviewer>-<sha>.md` and prints that path. The log
  goes next to it as `.log`.

Sol runs `codex exec review --base origin/<base> -m gpt-6.1-sol` with
`model_reasoning_effort`, `sandbox_mode="read-only"`, `approval_policy="never"` and the
guide text in `developer_instructions`. `codex review` rejects a prompt together with
`--base`, and `developer_instructions` is the way around that.

Opus runs `claude -p --model claude-opus-5-5 --effort high` with only `Read`, `Grep`,
`Glob` and `git diff`, `git log` and `git show` allowed, in `dontAsk` mode, so it cannot
edit or wait for a prompt. A fresh headless process is used instead of the `/code-review`
skill because the skill runs on the calling agent's model, which may be Sonnet, and with
the author's context.

Exit 4 means the reviewer's CLI is missing. For `sol-*`, run it again with `opus` and say so
in the report. Exit 1 means the review failed. Read the log, put the cause in the final
report, post nothing and go on.

## 2. Post the review

Read the review file. Turn each finding into one line, most severe first:

```
- **P1 — `src/actions.ts:79`**: Dispatch checks the nonce but not the attempt token. A save can wait in the queue and write after the user cancels the request. Recheck the attempt token when recording dispatch.
```

Use paths relative to the repo root (Codex prints absolute paths and `:79-79` ranges, so
cut those down). Keep the reviewer's priority. Drop anything that is about tooling, the
sandbox or what the reviewer could not run.

The review body, written to `<out>/review-body.md`:

```
> [!NOTE]
> 🤖 **Claude Opus 5.5 responding on behalf of <owner>**

**Codex review (Sol 6.1, high)**

- **P1 — `src/actions.ts:79`**: ...
- **P2 — `src/store.ts:360`**: ...

_Not verified by Claude. <One or two sentences: which findings look most worth checking first and why.>_
```

Use `Codex review (Sol 6.1, medium)` or `Opus review (Opus 5.5, high)` as the header to
match the reviewer. With no findings the body is the callout, the header and the line
`No findings.`, and nothing else. The callout names the model posting the review, as
[leaving-pr-comments](../leaving-pr-comments/SKILL.md) says. Every comment and reply below
starts with it too.

Post it as one review with an inline comment per finding. The owner's account posts it, and
GitHub does not let an author approve or request changes on their own PR, so the event is
always `COMMENT`. Write the inline comments to `<out>/comments.json`, one per finding:

```json
[
  { "path": "src/actions.ts", "line": 79, "side": "RIGHT",
    "body": "**P1** Dispatch checks the nonce but not the attempt token. <scenario>. <fix>." }
]
```

```sh
jq -n --arg sha "$(git rev-parse HEAD)" --rawfile body <out>/review-body.md --slurpfile c <out>/comments.json \
  '{commit_id: $sha, event: "COMMENT", body: $body, comments: $c[0]}' > <out>/review.json
gh api --method POST repos/{owner}/{repo}/pulls/<n>/reviews --input <out>/review.json > <out>/review-posted.json
```

With no findings, post the body alone: `gh pr review <n> --comment --body-file <out>/review-body.md`.

An inline comment must point at a line in the diff. If GitHub answers 422, drop the
`comments` entries whose line is not in the diff (check with
`git diff -U0 origin/<base>...HEAD -- <path>`) and post again. Those findings stay in the
body only. Then add links from the body to the inline comments:

```sh
rid=$(jq .id <out>/review-posted.json)
gh api repos/{owner}/{repo}/pulls/<n>/reviews/$rid/comments --jq '.[] | [.id, .path, .line, .html_url] | @tsv'
```

Append ` ([thread](<html_url>))` to the matching line in `review-body.md`, then:

```sh
gh api --method PUT repos/{owner}/{repo}/pulls/<n>/reviews/$rid -F body=@<out>/review-body.md
```

Posting is approved, so do not ask first. If posting fails, keep the files in `<out>` and
report the error.

## 3. Follow up

Check every finding against the code before acting. Codex comments carry no proof. For each
one decide: fixed, declined (with the reason), moved to an issue, or left for the owner.

Sort each finding yourself, whatever priority the reviewer gave it (from tstack's sol-loop):

- **blocker**: the case the PR is for is wrong. **should**: a real bug callers will hit
  within the PR's goal. **nit**: other paths, rare cases, small costs, extra compatibility,
  style.
- Fix blockers and shoulds with the smallest change. Prefer deleting or narrowing code over a
  new lock, cache, helper or option.
- Decline a nit in one line, or move it to an issue when it will matter later.
- When you disagree, say why with the line, test or output that shows it. Do not agree just
  to close a thread.
- After the fixes, compare `git diff --shortstat origin/<base>...HEAD` with the reviewed head.
  If the review grew the PR past its goal, cut first.

- Fix the real ones. Each fix gets a test that fails without it. Check that by copying the
  file aside or using `git show HEAD:<path>`, never `git stash`. One commit per fix where
  that is practical.
- Run the project's test command ([project-rules.md](project-rules.md)) and read the numbers.
- Make all fixes as separate commits, then push once. Never push one fix at a time, and
  never force-push.
- File what you do not fix now as an issue (file and line, the failure, a proposed fix,
  acceptance tests), or add it to an existing issue.

Write one PR comment to `<out>/review-fixes.md` and post it in step 4, after the PR is out of
draft, with `gh pr comment <n> --body-file <out>/review-fixes.md`. It names the local test
result on the new head:

```
**Review fixes**

All five Codex points were real. Each fix has a test that fails without it. The tests pass on the new head.

Fixed:
- **Dispatch skipped the attempt check** (`src/actions.ts`). <What went wrong and what changed.> b465f4c
- **<Title>** (`<file>`). <What changed.> <sha>

Follow-up:
- <What was moved and why.> <issue or comment link>

Declined:
- **<Title>**. <Why, with the line that shows it.>

Left for the owner:
- <Decision the owner has to make.>

Ready to merge: <the checks that passed, see ready.md>.
```

The last line appears only when the PR meets the rules in [ready.md](ready.md) (step 5).

Write "none" under a heading that has nothing, or fold them into one line as
`Declined: none. Left for the owner: nothing new.` Short commit SHAs become links on GitHub.
When you point the owner at questions or decisions in a file or section, name the file
and link it at the PR head, e.g.
`[section 12 of design.md](https://github.com/<owner>/<repo>/blob/<head sha>/docs/design.md#12-open-questions)`.

Then reply in each inline thread with one short line, and resolve the thread:

```sh
gh api --method POST repos/{owner}/{repo}/pulls/<n>/comments/<comment id>/replies -f body='Fixed in b465f4c. Dispatch now rechecks the attempt token.'
gh api graphql -F owner='{owner}' -F name='{repo}' -F n=<n> -f query='query($owner: String!, $name: String!, $n: Int!) { repository(owner: $owner, name: $name) { pullRequest(number: $n) { reviewThreads(first: 100) { nodes { id isResolved comments(first: 1) { nodes { databaseId } } } } } } }'
gh api graphql -f query='mutation($id: ID!) { resolveReviewThread(input: { threadId: $id }) { thread { isResolved } } }' -f id=<thread id>
```

Match threads by the first comment's `databaseId`. Resolve fixed, declined and moved
threads once they have a reply ("Fixed in …", "Declined: …", "Moved to #…"); the reply and
the "Review fixes" comment keep the record. Leave a thread open only when it needs an owner
decision, and list it under "Left for the owner". Do not review the new head again unless the
user asks or a fix for a failed train made it necessary.

## 4. Mark the PR ready

Mark the PR ready with `gh pr ready <n>`. When CI runs on pull requests, wait for its checks
on the head through `gh pr checks <n> --watch` in a background command and read the result.
Otherwise there is nothing to wait for: the tests from step 3 are the PR's check. Post the
"Review fixes" comment, if the review had findings, and go on to step 5.

When a merge-ready train later fails on the PR, merge-ready takes the label off and comments
with the failing test. Fix it as a commit, run the tests and push. That fix is not a review
fix, so [ready.md](ready.md) rule 1 stops holding: run one review of the new head, post it
and follow step 3 for its findings.

## 5. Mark it ready

Check the rules in [ready.md](ready.md). When all hold, add the `ready-to-merge` label and end
the "Review fixes" comment with the `Ready to merge` line, or post that line alone when the
review had no findings. When one does not hold, leave the label off and say which rule in the
final report. Take the label off again whenever a rule stops holding. Never merge. Merging
happens only through [merge-ready](../merge-ready/SKILL.md) after the owner's go-ahead.

## 6. Report

The final report to the orchestrator or user has: the PR link, the local test exit statuses,
the counts of findings fixed, declined, moved and left, whether the PR is labelled
`ready-to-merge` (and if not, which rule failed), and anything about the run itself
(review exit status, a fallback to Opus, timeouts, tool errors, sandbox limits, a 422 that
moved findings into the body). That last part never goes on the PR.
