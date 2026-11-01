# 3stack skills

A collection of skills I share with my agents.

Each skill lives in `skills/<name>/SKILL.md` and uses the [Agent Skills format](https://agentskills.io/specification). Two kinds live here:

- **Imported** skills stay exactly as written upstream. Added source metadata records the original URL and the exact upstream commit.
- **Authored** skills are written or adapted in this repo. They work in any repository: they read each project's own rules (agent instructions, test commands, CI) instead of assuming them.

## Available skills

### Writing

| Skill | Purpose | Origin |
| --- | --- | --- |
| [unslop](skills/unslop/SKILL.md) | Remove AI writing patterns. | Imported from [cursor/plugins](https://github.com/cursor/plugins/blob/main/pstack/skills/unslop/SKILL.md) |

### Docs

| Skill | Purpose | Origin |
| --- | --- | --- |
| [architecture-docs](skills/architecture-docs/SKILL.md) | Build or update an interactive architecture docs site for a repo: a TL;DR map that zooms into the real diagrams, flows that light up their path, and a searchable features catalog. Small subagents gather facts and a large model designs the diagrams. | Authored, uses the [atlas](https://github.com/DriesMeerman/atlas) engine |

architecture-docs runs only when invoked (`disable-model-invocation: true`). It needs Node
and git, and copies the atlas engine into the project's `docs/architecture-site/`. It works
in Claude Code and Codex.

### Work loop

Skills for longer, more autonomous work on a GitHub repo: pick work, ship it as reviewed PRs, and merge in safe batches. They share the labels `agent-ready`, `needs-human` and `ready-to-merge`, and create them in a repo when they are missing.

| Skill | Purpose | Origin |
| --- | --- | --- |
| [issue-triage](skills/issue-triage/SKILL.md) | Recommend what to do with each open issue on one local HTML page, where you accept changes, answer questions and add notes, then apply the reply. | Authored |
| [refine-issue](skills/refine-issue/SKILL.md) | Check whether an issue is ready for a coding agent and label it `agent-ready` or `needs-human`. | Authored |
| [refine-issues](skills/refine-issues/SKILL.md) | Pick the most important unready issues, fill in what an agent needs from the code, and label them. | Authored |
| [start-work](skills/start-work/SKILL.md) | Pick the top `agent-ready` issue or PR, do it, open a draft PR, and run a Codex or Opus review and its follow-up. | Authored, review flow adapted from [tstack](https://github.com/t3dotgg/tstack) |
| [start-group](skills/start-group/SKILL.md) | Run several tasks in parallel subagent waves, each with its own worktree, PR and review. | Authored |
| [full-send](skills/full-send/SKILL.md) | Take one change all the way to a merged PR while you are away. | Adapted from [tstack](https://github.com/t3dotgg/tstack) |
| [half-send](skills/half-send/SKILL.md) | Same as full-send, but stop with the PR labelled `ready-to-merge`. | Adapted from [tstack](https://github.com/t3dotgg/tstack) |
| [file-pr](skills/file-pr/SKILL.md) | Open a short draft PR that leads with the problem. | Adapted from [tstack](https://github.com/t3dotgg/tstack) |
| [leaving-pr-comments](skills/leaving-pr-comments/SKILL.md) | Mark and format the comments agents post on your account. | Adapted from [tstack](https://github.com/t3dotgg/tstack) |
| [merge-ready](skills/merge-ready/SKILL.md) | Test the `ready-to-merge` PRs together as one train, then merge them in order after your go-ahead. | Authored |
| [overnight](skills/overnight/SKILL.md) | Run start-group waves unattended, then one merge train at the end. | Authored |
| [midnight-oil](skills/midnight-oil/SKILL.md) | Overnight on the clock instead of a wave count. | Authored |

The work-loop skills link to each other by relative path, so install them together. They also link to `unslop`. Shared rules live in [skills/start-work/project-rules.md](skills/start-work/project-rules.md) (how to find a project's test command, hot files and limits), [review.md](skills/start-work/review.md), [ready.md](skills/start-work/ready.md) and [review.sh](skills/start-work/review.sh). [t3-code.md](skills/start-work/t3-code.md) says how agents start, wait and watch PRs when they run in [T3 Code](https://github.com/pingdotgg/t3code); the skills read it only there.

They need `gh` signed in, and for reviews the `codex` CLI (Sol) or `claude` CLI (Opus); `review.sh` reports a missing CLI so the skills fall back to Opus. issue-triage needs Node to build its page and keeps its files in `~/.cache/issue-triage/`, outside the repo.

Put project specifics in the project's `AGENTS.md` or `CLAUDE.md`, not in these skills: the test command, the full suite, hot files, paths agents must not touch, parity rules and a review guide.

## Use with agents

Clone this repository:

```sh
git clone https://github.com/DriesMeerman/3stack-skills.git
```

Copy or link each skill directory you want into your agent's configured skills directory, for example `ln -s "$PWD/3stack-skills/skills/start-work" ~/.claude/skills/start-work`. Keep bundled licenses and supporting files with each skill. Repeat for each agent, or configure them to read the same shared directory.

For agents that accept skill files directly, ask them to load `skills/<name>/SKILL.md`. Make each agent load the skill it needs. Loading a skill into one agent does not establish that another agent has loaded it.

Unslop retains its upstream `disable-model-invocation: true` setting. Invoke it explicitly in clients that honor that setting. merge-ready sets `model: haiku` for clients that honor it.

## Add a skill

To import one, create `skills/<name>/` and copy the upstream `SKILL.md` and any supporting files it needs. Preserve existing content, filenames, relative references, and license notices. Add attribution under the frontmatter's `metadata` mapping:

```yaml
metadata:
  source: https://github.com/owner/repo/blob/main/path/to/SKILL.md
  source-commit: "<full upstream commit SHA>"
```

If the skill already has `metadata`, add these keys to that mapping. Update the table above after importing a skill.

## Licenses

Imported skills retain their upstream licenses. Unslop is distributed under the MIT license, with the original notice in [skills/unslop/LICENSE](skills/unslop/LICENSE).

Authored skills are distributed under the MIT license, with the notice in each skill's own `LICENSE` file, so a skill copied on its own keeps it.

full-send, half-send, file-pr, leaving-pr-comments and start-work's review flow are adapted from tstack, distributed under the MIT license. Each of those skill directories also carries the original notice as `TSTACK-LICENSE`.
