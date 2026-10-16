---
name: orca-work-placement
description: >-
  Where isolated work goes in Orca, and how the agent that does it is named, modelled, and
  briefed. Load it whenever a request implies the change should not land on the branch the
  current checkout sits on — "브랜치 따서 작업해줘", "워크트리 만들어서 …", "main에 바로
  말고 따로 떼서", "PR로 올릴 거야", "새 워크트리에서 작업", "worktree 생성", "따로 떼서
  진행하자" — or whenever an agent is launched into a worktree. Isolation means an
  Orca-created worktree, never `git worktree add` and never a branch in the working
  checkout. Creating a worktree is not by itself orchestration; supervision is a separate,
  explicitly requested mode.
---

# Orca Work Placement

Two concerns: **where** isolated work is placed (Rule 1) and **how** the agent that does it
is launched (Rules 2–4). These constrain the decision, not the API — take every subcommand
and flag from the version-matched guide (`orca skills get orca-cli`), never from memory or
from the snippets below.

## Rule 1 — Isolation means an Orca-created worktree

**When the change should not land on the branch the current checkout is sitting on, the
isolated checkout is created by Orca.** That covers the ordinary case where the session is
on a branch that must stay clean and something worth committing comes up, and the explicit
case where the user says the work belongs on its own branch, in its own worktree, or behind
a PR.

```bash
orca worktree create --name <slug> --repo <owning repo> --agent claude --prompt "<task>" --setup run --json
```

- **`--repo` names the repo that owns the files**, which is not always the repo this session
  sits in. Orca infers the repo from the current worktree, so when registered repos are
  nested inside one another — submodules, monorepo subprojects — the umbrella repo silently
  wins if `--repo` is omitted, and the worktree is created against the wrong project.
  Resolve it with `orca repo list` whenever the target files are not in this repo's own
  tree.
- **Reuse before creating.** Run `orca worktree list --json` first: if a worktree already
  exists for that ticket, branch, or slug, work there via its full `<repo-id>::<path>`
  selector. Reuse costs nothing extra and avoids two checkouts competing for one branch.
- **Lineage and Git base are separate decisions.** `--no-parent` for independent work; child
  lineage only when the work genuinely stacks under the active worktree. Omit
  `--base-branch` to get the repo default, and never base on the current feature branch
  unless the user asked for stacked work.
- **Always `--setup run`** so the repo's setup hook runs. Read the receipt rather than
  assuming the result: if the repo has nested checkouts or generated files the task depends
  on, confirm they are populated before sending work in.
- `--name` is the slug form of the label in Rule 2.

**Forbidden, and both fail silently:**

- **`git worktree add`.** Orca creates worktrees under its own workspaces root and tracks
  them as app state. A checkout made anywhere else is external, and a repo whose
  `externalWorktreeVisibility` is `hide` will not show it at all. The work still exists on
  disk — it is just invisible in the app the user manages their work from, which is worse
  than either alternative.
- **A branch in the working checkout** for work meant to be isolated. It takes the checkout
  the user is sitting in hostage, which is the thing a worktree exists to avoid.

**Who does the work in it:**

- **Default — a fresh agent tab in that worktree** (`--agent claude --prompt "<task>"`). The
  worktree card and its agent live together, which is how the app is navigated. Read
  `result.agentTerminalHandle` from the receipt (falling back to
  `result.startupTerminal.handle`) and rename it per Rule 2.
- **This session continues instead** — create without `--agent` and work in the returned
  path. Keeps the context already built here, at the cost of the tab staying under the
  original worktree.
- **A supervised worker** — only when the user asks to supervise, monitor, wait for results,
  or coordinate several agents. See the `orca-orchestration-rules` skill.

**Creating a worktree is not orchestration.** Do not attach a Task, a dispatch preamble, or
a completion wait to a plain worktree launch. The cost is not theoretical: an unsupervised
Run leaves the task unsettled and the terminal unreleased, and the injected preamble
forbids the agent from asking the human directly, so a blocked agent has no one to ask.

## Rule 2 — The tab name says what the agent owns

Orca tabs otherwise show opaque handles. Rename **immediately after the launch receipt
returns, before any wait**, so the user can find the tab while it is still working.

```bash
orca terminal rename --terminal <agent_terminal_handle> --title "<label>" --json
```

- **The title is the work, not the job number.** `로그인 500 원인 추적`, not `worker 2` or
  `task_a1b2`. A title that reads as a category (`백엔드 조사`) fails the test as soon as
  three tabs could wear it.
- Keep it short enough to survive tab truncation (~30 chars), in the language the user is
  using. The worktree `--name` is the slug form of the same label.
- **One label, everywhere** — tab title, the line printed to the user when the launch is
  announced, and the way they will refer to it later.
- Renaming after the fact is the mechanism that works on every path; do not switch to a
  create-time display name on some paths and rename on others.
- A renamed terminal reads as user-modified, so Orca will not auto-dispose of it. Closing a
  tab is the user's call.

## Rule 3 — Opus by default, the named model when one is named

- **Default: Opus.** `--agent claude` inherits the user's Claude Code default model, so the
  ordinary worktree launch needs nothing extra — confirm that default is `opus` rather than
  assuming it, and pass the model explicitly on any path that accepts one.
- **When the user names a model or effort** ("sonnet으로 돌려", "codex gpt-5.5", "effort
  high"), that wins for exactly the agents they named, and the default resumes on the next
  launch. Say which model each tab got when announcing it.
- If a model is named but no agent is, infer the agent from the model family (Claude models
  → `--agent claude`, GPT/Codex models → `--agent codex`) and say which one you picked.
- A model id is **opaque and forwarded to that agent's CLI**: pass a value that CLI accepts,
  never an Orca-side invention.
- **`orca worktree create` has no model flag.** To pin a non-default model, create the
  worktree without `--agent`, then
  `orca terminal create --worktree id:<full-id> --command 'claude --model <id>'`. Use this
  two-step only when the model actually differs from the default — it can leave an unused
  fallback shell beside the agent, so target only the agent handle, and close the other
  terminal only after `terminal list` confirms it is an unused shell.

## Rule 4 — Confirm the goal before launching complex or subtly-conditioned work

A launched agent gets one shot at the prompt it was given. Follow-up messages land in a
session that may not read them until it is already done, so the goal is settled **before the
launch**, with the user.

**Ask when any of these hold:**

- "Done" could be read two ways, or the user's phrasing defers it ("알아서", "적당히").
- The task mutates anything — files, branches, commits, migrations, production data — and
  the exact target or blast radius is not pinned down.
- Conditions are subtle: edge cases, partial application, ordering constraints, "except
  when…" carve-outs, or behavior that differs per environment, user, or locale.
- The work spans repos, or the placement decision (which repo owns the files) changes what
  the answer even means.
- Two plausible readings would produce materially different work, so guessing wrong wastes
  the whole launch.

**Do not ask** for read-only investigation with an obvious success criterion, or when the
answer is already in this session — restate it in the prompt instead.

**How to ask:** one batched round of concrete, optioned questions, before launching. Cover
only what changes the work: definition of done, scope boundary (what must not be touched),
deliverable shape (findings in the tab, a commit, a PR, a file), and the subtle condition
itself, stated the way it should appear in the prompt. Then **copy the answers into the
prompt verbatim** as the success criteria — do not paraphrase them into something looser.

**If the user declines to specify** ("그냥 알아서 해"), proceed — but write the assumptions
into the prompt as explicit, named assumptions and print them when announcing the launch, so
a wrong one surfaces immediately instead of at completion.

## Launch checklist

1. Does this belong on the current branch? If not → Rule 1: reuse or create an Orca worktree
   in the repo that owns the files.
2. Goal is pinned (Rule 4) → prompt written self-contained, with success criteria, a fence
   naming exactly what may be written, and the repo's `AGENTS.md`/`CLAUDE.md` constraints
   that bear on it (never-push rules, commit-message conventions, skill routing).
3. Launch — agent tab by default, this session only when context continuity is worth the
   misplaced tab, supervised worker only on request. Model per Rule 3; read the receipt.
4. `terminal rename` to the label (Rule 2).
5. Announce one line: `label · worktree · model · what it was told to do`.
