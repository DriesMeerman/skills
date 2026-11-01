---
name: full-send
description: Take a change all the way to a merged PR while the owner is not watching. Proves it, files it, reviews it, gets it ready and merges it. Use when the owner says "full send".
---

# Full send

Adapted from [tstack](https://github.com/t3dotgg/tstack) (MIT, see
[TSTACK-LICENSE](TSTACK-LICENSE)). Apply [unslop](../unslop/SKILL.md) to everything you
write. In T3 Code, also read [t3-code.md](../start-work/t3-code.md); skip it elsewhere.

The owner said "full send" because they are not watching this thread. Nobody reads your
updates until long after you stop. You own the work from where it is now to a merged PR.
Without a clear "full send" from the owner, do not take this autonomy.

The request lets you:

- merge this one PR once the merge gate passes, without a merge-ready train
- use the test methods that Proof needs without asking first

It does not lift any project rule ([project-rules.md](../start-work/project-rules.md)).
Never force-push: when the branch is behind, merge `origin/main` into it. Never push to
`main`, use `git stash`, or touch anything project-rules.md puts off limits. Run the app
only against temporary data and config.

## Testing permission

The request is the permission for the test methods Proof needs, such as browser automation,
screenshots or running the app locally, even where a file says to ask first. Rules about how
to test still hold: the project's test command and any heavy-command wrapper, and no real
messages, paid inference or GitHub mutations in tests. List each method you used in the
report.

## Steps

1. **Finish the change.** The smallest change that does what the owner asked. Fix the case
   that happened, not every case that could. Work on a branch from `origin/main`, in its own
   worktree when the checkout has other work. If the request names an issue, read its
   solution, Done when and dependencies first.
2. **Prove it.** See Proof.
3. **File the PR** with [file-pr](../file-pr/SKILL.md), as a draft.
4. **Review it** with [review.md](../start-work/review.md) and `sol-high`: one review of the
   draft head, posted, fixes as commits with a test each, "Review fixes" comment, threads
   replied to and resolved, `gh pr ready <n>`. Prove each fix before you push it.
5. **Get it ready.** Check [ready.md](../start-work/ready.md). If `main` moved and the PR is
   behind or conflicts, merge `origin/main`, rerun the tests, and push. A conflict fix that
   changes behaviour needs one review of the new head. Label it `ready-to-merge` when every
   rule holds.
6. **Merge** when the merge gate passes.
7. **Report**: the PR link, the merge commit, the proof, and each choice you made for the
   owner.

Stay active from step 1 to step 7. If you end your turn while a review or test run is
going, the work stops, because nobody will prompt you again. Start long commands in the
background and wait for their completion notice. Never write a polling or `sleep` loop,
except the bounded wait [review.md](../start-work/review.md) gives subagents.

## Proof

Proof is what you saw yourself in this session: command output, a reproduction, a
screenshot. Use the most direct one the change allows:

- Reproduce the failure, apply the fix, then watch the same reproduction pass.
- Add one focused test that fails without the fix. Check that with `git show HEAD:<path>`
  or a copy of the file, never `git stash`.
- Run the project's test command.
- For UI or runtime behaviour, run it against temporary data and look at it. Green mocked
  tests do not show that it works for the owner.

When you cannot prove a change, you are not sure of it. Ask (see below).

## Merge gate

All of these hold on the latest head:

- Proof covers every change in the diff.
- Every rule in [ready.md](../start-work/ready.md) holds, and the label is on.
- The tests passed on this head, after the last merge of `origin/main`.
- No open `ci-red` issue, unless this PR closes it.
- No human comment asks for changes.
- For a stacked PR, the base PR is merged and GitHub retargeted this one to `main`.

Merge with a merge commit, at the head you checked:

```sh
gh pr merge <n> --merge --match-head-commit <headRefOid>
```

Never squash, rebase, `--admin`, `--auto` or `--delete-branch`. When the repo only allows
squash merges, stop and ask. After the merge, run [merge-ready](../merge-ready/SKILL.md)
step 4 for PRs stacked on this one and step 5 for its issues.

## Asking the owner

A question in a plain message goes unseen, and the thread looks done. Ask with
`AskUserQuestion` in Claude Code or `request_user_input` in Codex. Put everything in the
question: the PR link, what blocks you, and the options with your pick first.

Ask only when you cannot go on alone:

- You cannot prove the change.
- The merge gate cannot pass and you cannot fix that yourself.
- The next step is outside the grant: deleting data, touching anything off limits or
  outside the repo, closing a PR.
- The goal is unclear in a way that changes the result.
- Another PR makes this one unneeded.

Decide everything else yourself and list those choices in the report. If no question tool
is there, or it returns no answer, leave the PR open and unmerged and start your final
message with the question.

Inside an [overnight](../overnight/SKILL.md) run there is no one to ask. Relabel the issue
`needs-human`, post one short comment saying what blocks it, and stop.
