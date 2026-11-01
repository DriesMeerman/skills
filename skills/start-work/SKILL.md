---
name: start-work
description: Pick the most important open issue or PR in this repo and start working on it in the current session, then run a Codex Sol high review and a follow-up pass on its comments. Use for /start-work.
---

# Start work

Take one task from the repo's open work and do it here. Apply
[unslop](../unslop/SKILL.md) to everything you write for the user, PR bodies
and comments. Read the project's rules first, as [project-rules.md](project-rules.md) says.
In T3 Code, also read [t3-code.md](t3-code.md); skip it elsewhere.

The review flow in [review.md](review.md) is adapted from
[tstack](https://github.com/t3dotgg/tstack) (MIT, see [TSTACK-LICENSE](TSTACK-LICENSE)).

## 1. Survey open work

Run these and read the results. Do not start from memory.

```sh
gh issue list --state open --limit 100 --json number,title,labels,createdAt,body
gh pr list --state open --limit 50 --json number,title,headRefName,baseRefName,isDraft,reviewDecision,statusCheckRollup,mergeable,author
git worktree list
git status --short --branch
```

Read the epics and plan documents when an issue points at them. Note which
issues have an open PR already, which PRs stack on others (check
`baseRefName`), and which worktrees are live.

## 2. Rank

Candidates are open issues labelled `agent-ready` (see
[refine-issue](../refine-issue/SKILL.md)) and open PRs. Leave out issues labelled
`needs-human` and issues with no `agent-ready` label. PRs that wait on the owner rank as
below.

If no `agent-ready` issue is left and no PR needs work, say so, and offer to run
[refine-issues](../refine-issues/SKILL.md) on the top three, or
[issue-triage](../issue-triage/SKILL.md) to go through them on one page. Do not pick an
unlabelled issue on your own.

An open issue labelled `ci-red` (a red scheduled or push run on `main`) is the pick, even
without `agent-ready`, unless another session is already on it. Order the rest this way, and
say why for the top pick:

1. Labels: `p0`, then `p1`, then the rest.
2. Blockers: an item that unblocks other issues or PRs beats one that does not.
   PRs waiting on the owner (a failed train, review comments, conflicts) rank above
   fresh issues at the same label.
3. Age: oldest first.

Skip items another session is already working on: an open PR from a live
worktree, or an issue with a linked open PR. Skip issues whose dependencies are
still open unless the dependency is the pick. Show the pick, two runners-up and
the reason in a few lines. Go with the pick unless the user redirects.

## 3. Pick the reviewer

Do not ask. The reviewer is Codex `gpt-6.1-sol` with reasoning high (`sol-high`). When the
`codex` CLI is not installed or not signed in, use `opus` and say so in the report.

When the pick is very small, such as a one-file fix, a docs or copy change, or a
test-only change, add one line to the pick saying Sol medium (`sol-medium`) would
do, and keep going with Sol high. Never suggest Sol medium for anything touching
authorization, permissions, sandboxing, migrations, payments or code the project's
parity rules cover.

Use another reviewer only when the user names one: `sol-medium`, `opus`
(`claude-opus-5-5` in a separate headless run) or no review. Only Sol and Opus are
allowed models. Never use Fable or Astra.

## 4. Do the work

- If you hit a blocker you cannot resolve (an owner decision, missing access, an unclear
  requirement), run `gh issue edit <n> --add-label needs-human --remove-label agent-ready`,
  post one short comment saying what blocks it, and stop. Keep the draft PR if there is one.
- Read the issue's solution, acceptance tests and dependencies first.
- Work on a branch from current `origin/main`, or from the base PR's branch for a
  stacked item. Use a separate worktree if the current checkout has other work.
- Follow the project's rules, its parity rules included. Tests never send real messages,
  call paid inference or mutate GitHub.
- Run the project's test command (project-rules.md) and read the exit status before
  pushing.
- Open a draft PR for the issue. Never merge, force-push, push to `main`, or touch anything
  project-rules.md puts off limits. Never use `git stash`.

## 5. Review and follow-up

Push the branch and open the PR as a draft with `gh pr create --draft`. Then follow
[review.md](review.md) with the reviewer from step 3: run one review of the draft head, post
it, fix what is real as separate commits and push them once, post one "Review fixes"
comment, reply in each thread, mark the PR ready with `gh pr ready <n>`, and label the PR
`ready-to-merge` when it meets the rules in [ready.md](ready.md).

This skill never merges itself. As its last step, after the report below, it runs
[merge-ready](../merge-ready/SKILL.md), which runs on Haiku 5.5. Merge-ready merges only
after the owner's go-ahead in this conversation. Without one, it lists the labelled PRs
and the order and stops.

Finish with the PR link, the local test exit statuses, the review result, whether the PR is
labelled ready and anything left open. Problems with the review run itself (tool errors,
sandbox limits, timeouts) go here, never on the PR.
