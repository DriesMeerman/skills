# Running in T3 Code

Read this file only when the session runs in T3 Code, which you can tell from its tools
(`delegate_task`, `t3_background_wait`, `link_pull_request` and other `t3_*` tools from the
`t3-code` server) or its orchestration instructions. Outside T3 Code, skip it.

T3 Code runs agents as threads it can wake, link to PRs and stop. Inside it, the rules below
replace the skills' own ways of starting agents, waiting and watching PRs. Every other rule in
the skills still holds. When T3 Code's own instructions disagree with this file, theirs win.

## Agents

- Call `orchestrator_capabilities` once per thread, before the first `delegate_task`. Call it
  again only after an error. Take models from its list and put reasoning settings in
  `target.options`. When the model the plan names is not listed, say so in the plan, and do
  not swap it for another unasked.
- A task that gets its own branch and PR (a [start-group](../start-group/SKILL.md) or
  [overnight](../overnight/SKILL.md) task) runs as one `delegate_task` with
  `isolation: "worktree"`, `role: "implementation"` and `mode: "async"`, titled with its
  branch. T3 prepares the worktree from the parent's commit, so fetch first and start from
  current `origin/main`. Do not create child worktrees with `git worktree add`, and do not
  put checkout setup in the task text. The task text names the branch to create in that
  worktree, or the base PR branch for a stacked task.
- The child gets no history from the parent. Its task text is the full start-group brief,
  with this file's path so it reads this file too.
- Launch a wave's delegates in one message and keep each `taskId`. Do not poll them: no
  `task_status` on a running task, no `sleep`, no watchers. Follow the `next` that
  `delegate_task` returns, and read each result with `task_status` when its completion wakes
  you. A wave is done when every child's result is in.
- A stuck child (overnight step 3): read its `task_status` once. If it has not moved, stop it
  with `task_cancel` and note it. Never use `t3_thread_interrupt` on a child thread. Stop
  every delegate you replace or no longer need with `task_cancel`.
- A follow-up review or re-check is a new `delegate_task` with the brief, the findings and
  any objections, never a message into the old thread. Fix small findings yourself.
- Short read-only helpers that finish within the turn, such as the grounding subagents in
  [refine-issues](../refine-issues/SKILL.md), stay on the harness's own tool: `Agent` with
  `run_in_background: false` in Claude Code, `spawn_agent` and then wait in Codex.

## Waiting on long commands

A review, a test run or a train can outlast a turn. In T3 Code, a delegated child is its
own thread, and its background commands keep running when its turn ends.

- **Claude:** start the command (`review.sh`, a test run, `gh run watch` for a train) as a
  background Bash command, then call `t3_background_wait` with its task ID before ending the
  turn. The thread stays in Working and wakes when the command exits. Read the exit status
  and output then, and only then post or fix anything.
- **Codex:** `t3_background_wait` is not available. Run the command in the foreground and
  keep the turn open until it exits. Never end the turn while it runs.
- The bounded foreground wait in [review.md](review.md) for subagents is for harnesses
  without these tools. Do not use it here.

## PRs

- Right after you open a PR or push to one, call `link_pull_request` with its full URL. Link
  each PR once, every layer of a stack included. If linking fails, say so in the report.
- A delegated child never calls `watch_pull_request`. Its report ends with this line:
  `PR <full URL>, head <SHA>, checks and reviews <state>`.
- In the main thread, waiting on a PR's checks or reviews (review.md step 4, full-send's
  merge gate) uses `watch_pull_request` instead of `gh pr checks --watch`, `sleep` or a loop.
  Push first, then watch and end the turn. T3 wakes you on failed checks, passed required
  checks, new comments or conflicts. A wake is not a merge decision: check
  [ready.md](ready.md) and the merge gate yourself. Do not call watch again on a wake.
- Before the final report, call `unwatch_pull_request` on every PR this thread watches.

## Other

- Secrets: call `request_secret` and pass its `secretRef` on. Never ask for one in chat.
- Name a T3 thread in a report by its `link`, as given, never by a bare id or title.
- Questions to the owner still go through full-send's asking rules (`AskUserQuestion` in
  Claude, `request_user_input` in Codex).
