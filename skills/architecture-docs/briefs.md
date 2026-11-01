# Subagent briefs

Fill in the `<...>` parts. Each brief stands alone: the subagent has no other context.

## Fact finder (small, fast model, one per area)

```
Read-only. Do not edit files. Repository: <repo path>. Your area: <area name>, which is
<directories or packages>. Other areas, for context only: <list>.

Collect facts for an architecture docs site. Every fact needs evidence as path:line from
this checkout. Leave out anything you could not find in the code; say "not found" instead
of guessing.

Return markdown with these sections:

1. Components: each distinct part of this area (a service, module, worker, store, external
   system it calls). For each: a name, one sentence on what it does, the entry file and
   line, the 2 to 5 key functions or types with lines, and the state it owns.
2. Interactions: every call, message, queue, HTTP request, database read or write, file or
   cache access between a component here and anything else, as
   "A -> B: what, how (call | queue | http | db | file | event), path:line".
3. Paths: the sequences a request, job or event takes through this area, step by step,
   each step with the component and path:line. Include at least one failure path (retry,
   timeout, crash recovery) if the code has one.
4. Failure handling and invariants: what the code guarantees and how it recovers, with
   lines (transactions, idempotency keys, locks, retries, timeouts, validation).
5. Configuration: settings, env vars and flags read here, with where they are read and the
   default.
6. User-facing features: what a user, operator or calling service can do that this area
   provides, how it is triggered (endpoint, command, UI, schedule), and the lines.
7. Open questions: anything that looked important but you could not settle.

Keep each item short. Line numbers must point at the code itself, not at an import.
```

## Style scout (small, fast model)

```
Read-only. Repository: <repo path>.

Find the project's visual style, for theming a docs site. Look for: brand colours (CSS
custom properties, Tailwind or other theme config, design tokens, SCSS variables), fonts,
an existing docs site or app theme (dark and light), a logo or favicon colour, and the
general feel (dense or airy, sharp or rounded, playful or plain). Look in the README and
docs for any stated design rules.

Return: the files you read; the colours found, as hex with where each is used (background,
text, accent, error...); fonts; the feel in two sentences; and "no project style found" if
there is nothing to go on. Do not invent colours.
```

## Designer (largest model)

```
Repository: <repo path>. Engine: <atlas path>; read templates/site/SCHEMA.md, the header
comment of each file in parts/, and examples/demo/data/ for the shape of good data. Site
folder: <site path>. Pages: <picked pages>.

Attached: the fact sheets from <n> areas, and the style scout's report.

Design and write the site's content for engineers new to this codebase, mostly backend
readers. Write data/architecture.js<, data/tldr.js><, data/features.js> and theme.css.

- Use only facts from the sheets or code you read yourself. Every ref must point at a real
  line that backs the sentence it supports.
- Layers: 4 to 6, named for this system.
- Diagrams: an overview plus one per major path or subsystem, at most about 14 nodes each.
  Place nodes on the grid so the main path reads left to right, callers sit left of what
  they call, storage below and external systems at the edges. Avoid long edges that cross
  the whole diagram; add a focused diagram instead.
- Flows: the 4 to 12 sequences a new engineer asks about first, including at least one
  failure or recovery path. Group them with "group" paths when there are more than about 6.
  Each step names one node and says what happens there, with a ref where it helps.
- TL;DR: 4 to 8 blocks in reading order that tell the whole story; each block's zoom
  focus must be on its zoom diagram.
- Features: for engineers on the team. Each feature: summary, body (how it works), usage
  (how it is triggered), config (key, where, default), invariants, failures (when/then),
  archNodes (main one first), flows, refs. Group by purpose; give any group with more than
  about 12 features subgroups. Add aliases for likely search words. <omit if not picked>
- Record more than the page needs: extra nodes, edges, flows or features that are accurate
  but would crowd the page go in the data with hidden: true.
- Theme: match the project style from the scout's report. If there is none, invent a look
  that fits the project and say what you chose. Define --layer-<id> for every layer, for
  both dark and light, with text on panels at 4.5:1 or better. You may add tokens, fonts
  (local or system only) and overrides for any part.
- You may change engine files in <site path>/app/ when this project needs it. List each
  change and why.

Write plain sentences: no filler, no marketing words. Then run
node <atlas path>/scripts/check.mjs <site path> --repo <repo path> and fix every error.
Report what you made, what you left out, and any engine changes.
```

## Ref checker (small, fast model)

```
Read-only. Repository: <repo path>. For each item below, open the file at the line and
say whether the code there backs the sentence: "yes", "partly: <why>" or "no: <what the
code does>". If the line is off but the code is nearby, give the right line.

<numbered list: sentence | path:line>
```

## Update mapper (small, fast model, one per area)

```
Read-only. Repository: <repo path>. The docs site in <site path> describes commit
<old sha>; the code is now at <new sha>. Changed files in your area:
<git diff --stat lines>.

For each changed file: read the diff (git diff <old>..<new> -- <file>), then find every
entry in <site path>/data/*.js that refers to it (search for the path and for changed
function names). For each entry say what is now wrong: a moved line (give the new one),
a changed behaviour (quote the new code with path:line), a removed component, or "still
right". List changes that no entry covers but a reader would need (a new component, a
new step in a flow).
```
