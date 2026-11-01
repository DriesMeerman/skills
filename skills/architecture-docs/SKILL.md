---
name: architecture-docs
description: Build or update an interactive architecture docs site for the current repository with the atlas engine. The site has a TL;DR block map that zooms into the real diagrams, architecture diagrams with a component sidebar, flows that light up their path, and a features catalog. Asks which pages to make and whether to output a folder or a single page. Uses small, fast subagents for fact finding and a large model for the diagrams. Use for /architecture-docs or /architecture-docs update.
disable-model-invocation: true
---

# Architecture docs

Build an interactive docs site for this repository from the
[atlas](https://github.com/DriesMeerman/atlas) engine, or bring an existing one up to date.
Apply [unslop](../unslop/SKILL.md) to every piece of text the site shows and to your
report.

- `/architecture-docs` builds a new site. If one already exists, ask whether to update it
  or start over.
- `/architecture-docs update` updates an existing site to the current code (section 8).

**The engine is a starting point.** Copy its parts, then change any of them when this
project needs something different: a new view, a different layout, extra data fields,
another node shape, fonts, textures. Theme tokens are the minimum a part reads, not a limit
on the look. Record what you changed in the site's `SCHEMA.md`. Keep only these rules:
no build step, no runtime network requests, it opens over `file://`, content lives in
`data/`, and both themes stay readable.

## 1. Ask

Ask both questions in one go, with the host's question tool if it has one (Claude Code:
AskUserQuestion), otherwise in chat, and wait for the answers:

1. **Pages** (pick any): TL;DR, Architecture, Features. Default: all three. The TL;DR zooms
   into the architecture diagrams, so picking TL;DR builds the architecture data too; say
   so if Architecture is left out, and offer to show it.
2. **Output**: a folder (default) or a single HTML page. The folder keeps content in
   `data/*.js` and gets an `AGENTS.md` for later updates, which makes updating much easier.
   A single page is one file to share, rebuilt in full on every update.

Defaults: folder at `docs/architecture-site/`, single page at `docs/architecture.html`.
Use another path if the user names one or the project keeps docs elsewhere.

## 2. Read the project

Read the agent instructions (`AGENTS.md`, `CLAUDE.md`), the README, any architecture docs
(`docs/`, ADRs) and the top-level layout. Note:

- the source directories, the language and the main entry points;
- the remote and default branch (`git remote get-url origin`, `git rev-parse HEAD`), for
  source links. For GitHub, `refUrl` is
  `https://github.com/<owner>/<repo>/blob/{commit}/{path}#L{line}`; for GitLab,
  `.../-/blob/{commit}/{path}#L{line}`. Without a remote, leave `refUrl` empty and refs show
  as plain text;
- anything the agent instructions say about docs, generated files or paths not to touch.

Split the code into 3 to 8 **areas** of roughly equal size for the fact finders: one per
service, package or top-level module, with shared infrastructure as its own area.

## 3. Get the engine

```sh
atlas="$(mktemp -d)/atlas"
gh repo clone DriesMeerman/atlas "$atlas" -- --depth 1 \
  || git clone --depth 1 https://github.com/DriesMeerman/atlas "$atlas"
git -C "$atlas" rev-parse HEAD
```

Read `$atlas/README.md`, `$atlas/templates/site/SCHEMA.md` and `$atlas/DESIGN.md`. The demo
at `$atlas/examples/demo/` shows every view with made-up data; open it if you have a
browser tool.

## 4. Find the facts (small, fast subagents)

Start one fact finder per area, all at once, on the host's **smallest fast model** (see
"Models and subagents" below). Each is read-only and gets the area brief from
[briefs.md](briefs.md). Also start the style scout from the same file. Fact finders return
evidence, not design: components, what talks to what, state, failure handling, config,
user-facing features and the paths a request takes, each with `path:line`.

While they run, read the code paths that tie areas together yourself (entry points,
routers, queues, schema), so you can judge their answers.

When they return, check the sheets against each other: the same component named twice,
an edge one side reports and the other doesn't, a claim without a line. Send a follow-up
to the same fact finder for gaps rather than guessing.

## 5. Design (large model)

Hand the fact sheets to one designer on the host's **largest model** with the design brief
in [briefs.md](briefs.md), or do this step yourself if you are already running on it. The
designer decides:

- **Layers**: 4 to 6, named for this system (not necessarily the demo's).
- **Nodes**: the components a reader needs, usually 15 to 50. Each gets a summary, a body
  that covers how it works, invariants and failure handling, and refs to real lines.
- **Edges** with a short label and a kind: `data`, `control` or `store`.
- **Diagrams**: an overview, then one per major path or subsystem. At most about 14 nodes
  each, laid out on a grid so the main path reads left to right and storage sits below.
  Every node should be on at least one diagram.
- **Flows**: the sequences a new engineer asks about first (a request start to finish, the
  main background job, a failure and its recovery, a deploy or restart). Group them with
  `group` paths when there are more than about 6, and give groups titles in `flowGroups`.
- **TL;DR**: 4 to 8 blocks that tell the system's story in reading order, each standing for
  real nodes and zooming into the diagram that shows them.
- **Features** (when picked): see section 6.
- **Theme**: from the style scout's findings. Match the project's look when it has one.
  When it has none, invent one that suits the project; don't reuse the engine default
  unless it fits.

The designer writes the `data/*.js` files and `theme.css` directly. A large system may need
two designers in sequence: architecture and flows first, then TL;DR and features, which
point at the nodes the first one made.

## 6. Features page

When Features is picked, the designer also writes `data/features.js` (format in
`SCHEMA.md`). Write for engineers on the team, mostly backend readers. Each feature says:

- what it does (`summary`, then `body` for how it works);
- how it is used or triggered (`usage`: endpoints, commands, schedules, events);
- the settings that control it (`config`, with where each is set and its default);
- what always holds (`invariants`) and what happens when things fail (`failures`, as
  when/then pairs), where the code shows it;
- the components it touches (`archNodes`, main one first), the flows it runs through
  (`flows`) and the code (`refs`).

Group features by what they are for, not by code layout. **Size decides the depth**: a
group with more than about 12 features gets `subgroups`, so the search can dig down with
`group/subgroup/`. Fewer than about 10 features in total needs no groups beyond one or
two. Add `aliases` for words readers will search for that the title doesn't use.

## Record more than the page shows

The models building the site read a lot of code. Keep what they learn: on a large project,
write the extra nodes, edges, flows or features the page doesn't need and mark them
`hidden: true`. They stay in `data/` for tools and later updates, and the page leaves them
out. Hide rather than delete when an item is accurate but adds noise.

## 7. Assemble and check

Folder output:

```sh
node "$atlas/scripts/add.mjs" --to <site> --title "<project>" --tagline "<one line>" \
  --commit "$(git rev-parse HEAD)" <parts>
```

where `<parts>` is `arch`, `tldr` and `features` as picked. This copies the engine into
`<site>/app/`, writes `index.html`, `theme.css`, `data/site.js`, `SCHEMA.md`, `SOURCE.md`,
`AGENTS.md`, `CLAUDE.md` and `atlas.json` where missing, and lists the loaded files in
`index.html`. Then put the designer's data and theme in place and fill in `data/site.js`
(views in reading order, `repoUrl`, `refUrl`, `commit`, and `generated` with today's
date). `SOURCE.md` gets the same commit and date.

Make any engine changes this project needs now, and record them in `SCHEMA.md`.

Check, and fix everything it reports:

```sh
node "$atlas/scripts/check.mjs" <site> --repo .
for f in <site>/app/*.js <site>/data/*.js; do node --check "$f"; done
```

Then verify the content, not just the shape. Give a fresh small subagent 15 to 25 refs
picked across the site, each with the sentence it supports, and ask it to say for each
whether the code at that line backs the sentence. Fix what it flags.

If a browser tool is available, open `<site>/index.html` (serve the folder with a static
server if the tool needs http) at 1600px and at a phone width, in both themes, and check:
every view loads with no console errors, a diagram's edges and labels don't overlap boxes,
picking a flow lights its path and opens the sidebar, the TL;DR zoom works, and the feature
search filters. Fix what looks wrong; layout problems are usually fixed in the grid
positions in `data/architecture.js`.

Single page output: build the folder in a temporary directory as above, run every check
there, then:

```sh
node "$atlas/scripts/inline.mjs" <tmp site> docs/architecture.html
```

Add the commit, the date and "regenerate with /architecture-docs" in an HTML comment at
the top.

Report: what was built and where, the page count of each kind (nodes, diagrams, flows,
blocks, features), anything left out and why, engine changes made for this project, and
how to open it. Do not commit unless the user asks.

## 8. Update

1. Read `<site>/SOURCE.md` for the commit the content describes, and `<site>/AGENTS.md`
   for anything the site does differently.
2. `git diff --stat <commit>..HEAD` and `git log --oneline <commit>..HEAD` over the source
   directories. With no changes, say so and stop.
3. Map changes to entries: search `data/*.js` for each changed path. Hand this to small
   subagents when many files changed, one per area, with the update brief from
   [briefs.md](briefs.md).
4. Edit the affected entries: line numbers, bodies, edges, flow steps. New components or a
   changed request path go to the large model, as in section 5, for the parts they touch.
5. Set the new commit and today's date in `SOURCE.md` and `data/site.js` (`commit`,
   `generated`).
6. Run the checks in section 7, then report what changed in the docs and why.

To pick up engine changes as well, run `node "$atlas/scripts/add.mjs" --to <site> --status`.
Update untouched files by re-running `add.mjs` with the part names. Merge upstream changes
into files the site edited by hand, or leave them; say which.

## Models and subagents

Both Claude Code and Codex can run this skill.

| Job | Claude Code | Codex |
|---|---|---|
| Fact finding, style scout, ref checks, update mapping | Agent tool, `model: "haiku"` | `spawn_agent` on the smallest fast model (Luna) |
| Diagram, flow, TL;DR, feature and theme design | Agent tool, `model: "opus"`, or yourself on Opus | `spawn_agent` on Sol, or yourself on Sol |

Start independent subagents together and wait for all of them. Give each one its brief and
nothing else it has to guess: the repository path, its area, what to return. If the host
cannot pick a model per subagent, use what it has and say so in the report. If it has no
subagents at all, do the fact finding yourself, area by area, and keep notes in the same
shape as the fact sheets.
