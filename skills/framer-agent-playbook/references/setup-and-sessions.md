# Setup and sessions

The `framer` skill is the entry point for editing a connected Framer project. It is **not** part
of this skill; Framer's official agent bridge installs it at machine level into
`~/.claude/skills/framer` and `~/.agents/skills/framer`.

Mandatory sequence, in order:

```bash
npx @framer/agent@latest setup
```

```bash
npx @framer/agent@latest project auth "<project-url-or-id>" "<api-key>"
npx @framer/agent@latest session new "<project-url-or-id>"
```

**The API key is only needed once per project on this machine.** `project auth` saves the
credentials. `session new` takes only the project URL or ID, not a positional API key. If you pass
one anyway it is ignored, a browser authorization flow opens, and the call times out after about
90s. For saved projects, `npx @framer/agent@latest project list` gives you the IDs and
`session new "<projectId>"` reuses the saved authorization. When the user says they will provide a
new target, wait for it instead of connecting to an older sandbox. Saved authorization is
machine-specific.

`setup` also installs a sibling `framer-code-components` authoring-reference skill at machine level,
distinct from this repo's `framer-code-components-overrides` skill.

- **The package was renamed.** `framer-dalton` became `@framer/agent`. Older notes may still say
  `framer-dalton`; treat those as the same tool, and update anything hardcoding the old name.
- **Sessions are per project.** `session list` shows the open ones with their `projectId`; a second
  project needs its own `session new`, and every `exec` then needs the matching `-s <id>`.
- **Opening a new session kills the previous session's connection**, so finish a batch before
  reconnecting.
- **A session left idle for about an hour expires** ("Session 1 is invalid or has expired"). Run
  `session new` again (2026-10-11).
- **`session new` fails about half the time with `Vekter stalled after "showUI"`. Run it again.**
  Vekter is Framer's editor runtime on their side; the headless instance loaded the UI and then
  filed no perf mark for 60s. It is not a credential problem (a bad key gives `API key does not have
  access to this project`). Measured 2026-08-22 on a client build: stall, success, success. Each
  attempt burns about 85s.
- **Session ids are reused across projects.** After the relay restarts, `session new` hands out
  `1` again, so a script that ran with `-s 1` on one project now runs on whichever project was
  connected last. Run `session list` (it shows each id's `projectId`) before any write, above all
  when a live project is connected on the same machine (2026-10-10).
- **Two session-level module failures are fixed only by `session new`, never by retrying:**
  `ensureComponentsInLoader: Some modules are missing` (pages with a ShaderNode) and
  `Assertion Error: The importMap has to exist on the module` (reading a `ComponentNode`) (2026-08-21).
- **Relay death.** `Failed to get connection for session: Timed out` on every call means the local
  relay server died. `session new` restarts it and reissues session id `1` (existing `exec -s 1`
  invocations keep working). `relay restart` is more destructive: it wipes the session's shared
  exec `state` and can destroy sessions outright. The write path (`applyChanges`) can die while
  reads still work, so test with a trivial `SET` before re-dispatching builders. After a relay or
  editor crash, `session new` may throw `FramerAPIError: Timed out` for about 10 minutes while stale
  headless editors clear server-side. Retry every 90s instead of restarting anything.
- **Several agents on one session** (validated across about 10 builder agents on session 1): agents
  must never write exec `state`, never run `session new` or `relay restart`, and must treat a
  timed-out write as possibly landed. Re-read before re-applying, or you get duplicate `… 2` tokens
  and doubled sections. Temp ids are shared too, see [Running exec scripts](exec-scripts.md).

## The generated project context

`session new` **no longer writes a project-scoped skill** to `.agents/skills/` (changed around
0.0.4x). It generates a context folder inside the installed `framer` skill.

**The grammar line is not the whole API.** The giant `SET` line in `updating-the-project.md` has left
out whole features (event handlers, until 0.0.46). Before declaring something impossible and handing work
back to the user, grep every file in the generated `prompt/` directory, `core-examples.md`
especially.

**Authored** skills (`framer-code-components-overrides`, `framer-plugins`) can live in a repo's
`.claude/skills/`. The generated context cannot move into a repo: it lives in a machine-level skill
directory and is regenerated on every `session new`.

## How the bridge talks to Framer

**`@framer/agent` is not an MCP server.** The CLI bundles `framer-api` (the Server API) and opens
its WebSocket to `wss://api.framer.com/channel/headless-plugin`, which drives a headless Framer
editor (Vekter) on Framer's side. The `relay` is a local tRPC server on `127.0.0.1` that holds that
connection between `exec` calls. The package has no MCP SDK or JSON-RPC code, and its README says no
Framer MCP server is needed. Any Framer MCP server is a third-party alternative. Checked in
the 0.0.46 and `framer-api` 5.1.0 packages, 2026-10-04.

**Auto-branching.** With auto-branching on, the first write of a fresh `session new` opens a new
branch with an `adjective-noun` name and prints `[FRAMER_BRANCH_CHANGE]`; every later write in that
session lands on it. `framer.agent.getActiveBranch()` names it, `framer.agent.renameBranch(id, title)`
renames it, `publish` on it updates only a branch preview URL
(`<site>--<branch-slug>-<id>.framer.app`) and leaves production alone, and
`framer.agent.mergeBranch("main")` merges it and switches the session to main, after which a
`publish` goes to production. CMS items and code file edits ride the branch too. Merge and publish
only when the user asks (on a live site, see [`live-files.md`](live-files.md)). Seen on a project
its owner cleared for it, 2026-10-10. Deletes stay irreversible on a branch too until it is confirmed otherwise (see
[Destructive operations](destructive-operations.md)).

## What the bridge cannot do

- **`framer.agent.startConversation()` and `continueConversation()` are employee-only** ("This
  method is only available to Framer employees"), so the bridge cannot delegate to Framer's own
  agent. Verified 2026-09-07 on 0.0.44.
- **No shader authoring.** `readShaderControls` is the only shader call and `createCodeFile` only
  knows components and overrides. The "Agent Generated Shaders" beta (announced 2026-09-07) is
  reachable only from the Agent panel inside the editor. On the account tested that day the panel was
  not on the beta: the example prompt produced a catalog `liquid-gradient` ShaderNode with
  `blendingMode="color"` and no code file. Re-test once the beta is enabled for the workspace.
- **`framer.publish()` from `framer-api` in a Val Town val works** and is not behind that gate: a
  webhook call published and went live in 5.6 s (2026-09-25).
- **`publish` is gated by the auto-mode permission classifier** ("Production Deploy"), even
  `{action:"preview"}`. Publish from an interactive session or the editor. If your agent's harness
  gates production deploys, a refused publish went through once the user had asked for it in plain
  words in the conversation (2026-10-11).
  `framer.publish()` can also fail with a bare "Publish failed" on a valid project; it is
  intermittent (three successes running on 2026-08-18), so retry, then fall back to the editor.
