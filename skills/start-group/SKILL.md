---
name: start-group
description: Pick a group of the most important open issues and PRs in this repo and run them in parallel with subagents. Each subagent opens its PR, reviews its own PR and processes the comments. Use for /start-group.
---

# Start group

Run several tasks from the repo's open work at once. Apply
[unslop](../unslop/SKILL.md) to everything you write for the user, PR bodies
and comments. Read the project's rules first
([project-rules.md](../start-work/project-rules.md)). Survey and rank as in
[start-work](../start-work/SKILL.md) steps 1 and 2, then pick a group instead of one item.
In T3 Code, also read [t3-code.md](../start-work/t3-code.md), which says how to start and
wait for the agents there; skip it elsewhere.
An open `ci-red` issue (a red run on `main`) goes first, in the first wave, even without
`agent-ready`: merge-ready merges nothing else until it is fixed. The same filter applies:
only issues labelled `agent-ready` go in the wave table, plus PRs that need work. With none,
say so and offer to run [refine-issues](../refine-issues/SKILL.md) on the top three. Do
not fill the table with unlabelled issues.

## 1. Ask about scope

Ask how much work to take: 2, 3, 5 or a number the user gives. Offer "until the
important p1 items are covered" as an option. Work past one wave's limit runs in
later waves (section 2).

Do not ask about review. Each subagent reviews its own PR with Codex
`gpt-6.1-sol`, reasoning high (`sol-high`), or `opus` when Codex is not installed. The plan
table (section 2) has a reviewer column. For a task whose scope is very small, such as a
one-file fix, a docs or copy change, or a test-only change, write `sol-high (sol-medium
would do)` there, and still run Sol high. Never suggest Sol medium for a task touching
authorization, permissions, sandboxing, migrations, payments or code the project's parity
rules cover. Use another reviewer only when the user names one: `sol-medium`, `opus`
(`claude-opus-5-5` in a separate headless run) or no review. Only Sol and Opus are allowed
models. Never use Fable or Astra.

The test step always runs and is not an option.

## 2. Plan the group in waves

Before launching anything, write the group as a table: task, issue, branch,
owned paths, depends on, wave, model, reviewer. Keep it in the session. Do not write it
to a file in the repo or open a PR for it.

```
| Task | Issue | Branch | Owned paths | Depends on | Wave | Model | Reviewer |
| --- | --- | --- | --- | --- | --- | --- | --- |
| Session store | #57 | session-store | src/sessions/, src/db/migrations/ | none | 1 | opus | sol-high |
| Login page | #58 | login-page | web/login/ | #57 | 2 (stacks on #57) | sonnet | sol-high (sol-medium would do) |
```

- A wave is at most 4 subagents on a laptop and at most 6 on a bigger machine or in a
  cloud session. The next wave starts when the current wave's PRs are open and reviewed,
  not merged.
- Owned paths are the files and folders a task may change. Two tasks in one
  wave never own the same file. Hot files (project-rules.md) get one writer per wave.
- A task never shares a wave with what it depends on. When its wave starts
  and that PR is still open, it stacks on that PR's branch. If both change the
  same hot file, it waits until that PR is merged instead.
- Take items in rank order. Check live worktrees and open PRs, and treat the
  files they change as owned, so the group does not collide with another
  session.
- Show the table. Go ahead unless the user redirects.

## 3. Run the subagents

Run one wave at a time. One subagent per task, each in its own git worktree
and branch. Use the model the issue's Implementer line names. With no line, pick one
by the implementer rule in [refine-issue](../refine-issue/SKILL.md). Launch a wave's
subagents in a single message.

Give each subagent:

- the instruction that on a blocker it cannot resolve it relabels its issue (`gh issue
  edit <n> --add-label needs-human --remove-label agent-ready`), posts one short comment
  saying what blocks it, and stops, then reports to you
- the issue number and the instruction to read its solution, acceptance tests
  and dependencies first
- its owned paths, and that it reports back to you instead of editing a file
  outside them
- the project rules you found: the test command, any heavy-command wrapper, parity rules
  and what is off limits. Tests never send real messages, call paid inference or mutate
  GitHub.
- the limits: open a draft PR, never merge, force-push, push to `main`, or touch anything
  off limits
- never `git stash`, since all worktrees share one stash list
- its own scratchpad subfolder for PR bodies and temp files
- run the tests and read the exit status before pushing, through the heavy-command
  wrapper when the project has one, so only one runs at a time across all subagents
- after its PR is open, the steps in section 4, with its reviewer from the
  table
- to stay in its turn until its review has finished, waiting as review.md says for
  subagents. Its background commands die when it ends its turn, and the review is lost.

Stacked items branch from the base PR's branch. When a base branch moves,
merge `origin/main` into the branch in its worktree and never force-push.

## 4. Each subagent reviews its own PR

There is no cross-PR pass. When a subagent has pushed its branch and opened its
PR as a draft, it follows [review.md](../start-work/review.md) with its
reviewer from the table: one review of the draft head, posted on the PR, then the
fixes as separate commits in one push, the tests, `gh pr ready <n>`, one "Review fixes"
comment, a reply in each thread, and the `ready-to-merge` label when the PR meets the rules
in [ready.md](../start-work/ready.md). A stacked PR gets the label only when its base PR is
merged or labelled. Each subagent waits for its own review to finish before it posts or
fixes anything.

Subagents report how the review ran (tool errors, sandbox limits, timeouts) to
you, not on the PR. When a wave's PRs are open and reviewed, start the next
wave. When all subagents finish, report a table: PR, title, local test
result, review result, ready label, open items. Below it, list any review-run
problems.

Subagents never merge. When the last wave is done and reported, run
[merge-ready](../merge-ready/SKILL.md) from the main session. It runs on Haiku 5.5 and
merges only after the owner's go-ahead in this conversation. Without one, it lists the
labelled PRs and the order and stops.
