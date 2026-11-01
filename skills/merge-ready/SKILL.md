---
name: merge-ready
description: Merge the open PRs labelled ready-to-merge in a sensible order. Runs the full test suite once on the PRs merged together, on GitHub or locally, and without the owner's explicit go-ahead in this conversation stops there and reports. With it, merges, rechecks each PR before merging and repairs PRs that conflict afterwards. Use for /merge-ready.
model: haiku
---

# Merge ready PRs

Merge the PRs that [start-work](../start-work/SKILL.md) and
[start-group](../start-group/SKILL.md) labelled `ready-to-merge`. Apply
[unslop](../unslop/SKILL.md) to everything you write for the user and on PRs. Read the
project's rules first ([project-rules.md](../start-work/project-rules.md)): the full suite,
the CI workflow, any heavy-command wrapper and what is off limits. In T3 Code, also read
[t3-code.md](../start-work/t3-code.md) for waiting on the train; skip it elsewhere.

This skill runs on Haiku 5.5 (`claude-haiku-5-5`): it follows fixed steps and makes no
design calls. A conflict that needs a judgment call is reported, not resolved (step 4).

Steps 1 and 2 need no go-ahead. Merging (step 3 onward) runs only after the owner
says to merge in this conversation. Without a go-ahead, list and order the PRs, run
step 2, report the train result and stop before step 3. With one, run every step.

The go-ahead covers the PRs labelled `ready-to-merge` later in the same conversation
too. Ask the owner again only for a PR labelled after the go-ahead that is risky: a
schema migration, a change to authorization, permissions or sandboxing, payments, code the
project's parity rules cover, or anything touching what project-rules.md puts off limits.
Judge it from the PR's files and body. A PR that loses its label during the run drops out.

A green train from a run without go-ahead still counts when the owner then says to
merge, if `origin/main`, every PR head in the train and the set of labelled PRs are
unchanged. Otherwise build a new train. It counts toward the two-run limit in step 2.

`gh api` fills in `{owner}` and `{repo}` from the current checkout.

## 1. List and order

```sh
gh pr list --state open --label ready-to-merge --limit 50 --json number,title,headRefName,headRefOid,baseRefName,createdAt,files,body,closingIssuesReferences
gh pr list --state open --limit 100 --json number,headRefName,baseRefName,files
gh issue list --state open --label ci-red --json number,title
```

An open `ci-red` issue means a run on `main` is red. While one is open, the train
holds only the labelled PRs whose `closingIssuesReferences` include it, and the PRs they are
stacked on. Leave the rest out and report them as held by the `ci-red` issue. With no such PR,
merge nothing and report.

Order them by these rules, each one breaking ties in the one before:

1. Stacks go base first. A PR whose `baseRefName` is another PR's branch comes
   after that PR.
2. Named dependencies: "stacked on #m", "depends on #m" or an issue dependency in
   the body puts the PR after #m.
3. PRs that touch the same files go smaller first, by changed lines.
4. Oldest first.

Show the order as a numbered list with one line of reason per PR before the
train and before the first merge.

## 2. Test the train

Before the first merge, run the full suite once on all the PRs merged together. Every train
runs, even when the PRs change only docs, since tests may read those files too. `<out>` is
your scratchpad folder and `<date>` is today as `YYYY-MM-DD`.

1. Build the train in a temporary worktree from `origin/main`. Name the branch
   `train/<name>`, where `<name>` is `<date>` followed by the PR numbers in merge
   order, joined by `-`. For example, `train/2026-10-14-387-388-389-391-390`.
   - If the branch name would be longer than 60 characters, keep the first PR numbers
     that fit and end with `-plus<m>`, where `<m>` is the number of PRs left out:
     `train/2026-10-14-387-388-389-plus4`.
   - If that name was already used today, append `-2`, then `-3`, and so on. A name
     is used when `origin` has the branch: `git ls-remote --heads origin 'train/<name>'`.

   ```sh
   git fetch origin main
   git worktree add -b train/<name> <out>/train-<name> origin/main
   ```

   In that worktree, merge each PR in order, at the `headRefOid` from step 1:

   ```sh
   git fetch origin pull/<n>/head
   git merge --no-ff --no-edit -m "Train: #<n>" <headRefOid>
   ```

   On a conflict, run `git merge --abort` and drop the PR from the train, with every PR
   stacked on it, and note why. Dropped PRs are not merged in this run. A PR dropped for
   a conflict keeps its label; step 4 repairs it once it conflicts with `main`.
2. Pick where the train runs. It runs on GitHub when the repo has an active workflow that
   runs the full suite and can be started by hand (`workflow_dispatch` in its `on:`). Find
   it and read its state:

   ```sh
   grep -l workflow_dispatch .github/workflows/*.yml .github/workflows/*.yaml 2>/dev/null
   gh api repos/{owner}/{repo}/actions/workflows/<file> --jq .state
   ```

   `active` goes to item 3. No such workflow, or any other state such as
   `disabled_manually`, goes to item 4: run the train locally. Never enable or edit a
   workflow.
3. **On GitHub.** Push the train branch and start the workflow on it:

   ```sh
   git push origin train/<name>
   gh workflow run <file> --ref train/<name>
   gh run list --workflow <file> --branch train/<name> --event workflow_dispatch --json databaseId --jq '.[0].databaseId'
   ```

   The run can take a few seconds to appear in the list. Wait for it 15 minutes at most,
   with `timeout 900` on Linux and `perl -e 'alarm 900; exec @ARGV'` on macOS, then list
   its jobs:

   ```sh
   timeout 900 gh run watch <id> --exit-status; echo EXIT=$?
   gh run view <id> --json jobs --jq '.jobs[] | [.name, .conclusion] | @tsv'
   ```

   124 (`timeout`) or 142 (`alarm`) means the wait passed 15 minutes. Any other non-zero
   code is red. `EXIT=0` alone is not green: green means every test, build and check job
   shows `success`. When jobs show `skipped`, the workflow skipped its tests (a path filter, a
   budget gate, a condition). Read why with `gh run view <id>` and run the train locally
   (item 4) in the same worktree. That local run replaces the skipped one and does not count
   as a second run.

   A rerun of a red run tests the same commit, so it does not pick up a fix pushed to `main`
   since. Build a new train instead.
4. **Locally.** In the train worktree, install the dependencies and run the full suite,
   through the heavy-command wrapper when the project has one. Leave out tests the project
   marks as needing a browser or outside service when the machine lacks them, and say which.
   On macOS wrap the run in `caffeinate -i`, so the machine does not sleep, and keep it on
   power.

   ```sh
   <install command>; echo EXIT=$?
   caffeinate -i <full suite command> > <out>/train-<name>.log 2>&1; echo EXIT=$?
   ```

   Start the test run as a background Bash command and wait for its completion notice; a
   foreground call stops waiting after 10 minutes. `EXIT=0` is green. Anything else is red:
   read the end of the log for the failing test. A local train needs no push.
5. Green: without a go-ahead, clean up (item 8), report the train and where it ran, and stop.
   With a go-ahead, go to step 3 and merge the PRs in the train, at the same `headRefOid`s.
   If `origin/main` moved since the train was built, build a new train first. That
   counts as a run.
6. Red: read the failure (`gh run view <id> --log-failed` on GitHub, the log file locally)
   and find the PR whose diff holds the failing test or the code in the error. Take its
   `ready-to-merge` label off and comment on it with the failing test and the error line.
   Drop it and the PRs stacked on it, build a new train (with a new name from item 1) and
   run it once.
7. Stop and merge nothing when the second run is red, a wait passes 15 minutes, or the
   failure does not point at one PR. Never rerun a run without the owner's say. At most two
   train runs per `/merge-ready`. A run without go-ahead and the go-ahead that follows
   it count as one `/merge-ready`.
8. Afterwards, whatever the result, delete each train branch, on `origin` when it was
   pushed, and locally:

   ```sh
   git push origin --delete train/<name>
   git worktree remove --force <out>/train-<name>
   git branch -D train/<name>
   ```

## 3. Merge one at a time

Only PRs in a green train merge. For each one, in order:

1. Check the rules in [ready.md](../start-work/ready.md) again, live, and that
   the label is still on. If anything changed since the go-ahead (a head other
   than the one in the train, conflict, new unresolved thread or comment, label gone,
   a new open `ci-red` issue it does not close), skip the PR and note why. A PR stacked on a
   skipped PR is skipped too.
2. Merge with a merge commit:

   ```sh
   gh pr merge <n> --merge --match-head-commit <headRefOid>
   ```

   `--match-head-commit` makes GitHub refuse if the head moved after the check.
   Never squash, rebase, `--admin` or `--auto`. When the repo allows only squash merges,
   stop before the first merge and ask the owner. Never pass `--delete-branch`: a PR
   stacked on the branch would close instead of moving to `main`. Never push to `main`.
3. Record the merge commit: `gh pr view <n> --json mergeCommit --jq .mergeCommit.oid`.

## 4. After each merge

- PRs stacked on the merged one: confirm with `gh pr view <m> --json baseRefName`
  that GitHub retargeted them to `main`. If not, retarget with
  `gh pr edit <m> --base main`.
- For every open PR, ready or not, check `mergeable`. For each one that now
  conflicts:
  1. Take its `ready-to-merge` label off if it has one.
  2. In a worktree on its branch (reuse the live one if there is one), run
     `git fetch origin && git merge origin/main`, resolve the conflicts, then run the
     project's test command. This is not a train: the repaired PR merges only in a later
     green train.
  3. If the check passes, commit and push normally. If it fails, or a conflict
     needs a judgment call, push nothing and report it.
  4. The new head needs a review ([review.md](../start-work/review.md), with
     `review.sh` on the new head) before it can be labelled again. It drops out
     of this run.
- Continue with the next PR in the order.

Stop at the first failure the rules above do not cover, such as a merge that
GitHub refuses for another reason, and report.

## 5. Update related issues

Run this after the last merge of the run, for the PRs merged in it. Skip any PR
that was dropped, skipped or repaired: its issue stays as it is until the PR merges.

1. A closing reference (`Closes #n`) closes its issue on merge. Check each one is
   `CLOSED`. If one is still open, read why before doing anything. Never close an
   issue by hand.
2. A `Part of #n` reference gets one comment on `#n`: the PR number, what it
   merged, and what the PR body lists as still open. Use only what the PR body
   says. Do not add labels.
3. If `#n` is a child in an epic body (a `- [ ] #n` line), tick that box with
   `gh issue edit`, but only when `#n` is `CLOSED`. An open child may still have
   work left, so leave its box unchanged. Change no other text in the epic.
4. Keep each comment to a few lines, in unslop style, with no run details. Start it with
   the [leaving-pr-comments](../leaving-pr-comments/SKILL.md) callout.

```sh
gh pr view <n> --json closingIssuesReferences,body --jq '.closingIssuesReferences[].number, (.body | scan("Part of #[0-9]+"))'
gh issue view <i> --json state --jq .state
gh issue comment <i> --body-file <out>/comment-<i>.md
```

Report each issue touched, with its state and what changed.

## Limits

Never force-push, push to `main`, use `git stash`, delete a branch other than
the skill's own `train/` branches, or override branch protection. Reviews use
Sol or Opus only, never Fable or Astra. Leave everything about the run itself
out of the PRs.

## Report

The train runs (branch `train/<name>`, where it ran, run link or local exit status, result,
PRs dropped and why, PRs held by a `ci-red` issue). A
run without go-ahead ends here, with the PRs that would merge and a note that
merging needs the owner's go-ahead. Otherwise continue with a table of
merged PRs (PR, title, merge commit), then skipped PRs with the reason, then
labels removed and why, then PRs repaired after a conflict (new head, local test
result, review still needed).

Then the issues from step 5 (issue, state, what changed).
