# Live production sites

A live Framer project is the file a real domain serves, often one the site owner's team and their
own agents also edit. A mistake there reaches the public site or wipes someone else's work, and the
`@framer/agent` API has no undo. These are the habits that kept client builds safe (2026-09 to
2026-10). They are defaults: the user's own instructions win, and where the project's own notes are
stricter, follow those.

A file counts as live when a real domain serves it, even a holding page, or when other people edit
it. A personal sandbox or a fresh build is not live.

## Before the first write

- **Get the task, the page and the branch from the user in words.** Write only where you were
  asked; every other page stays as it is. Approval of a plan is not approval of its collateral.
- **The API key stays in the session.** Never in a repo, memory, docs or a commit.
- **Read the project before writing anything:** the pages you will touch, the components, text
  styles, colour tokens and CMS collections they use, and where else each one appears.
- **No tests or experiments in a live file** (2026-09-29). A probe, a trial write, or a check of
  how Framer behaves (does a branch follow main, where does a merge land) goes in a separate test
  project, a duplicate of the file if the user can make one. The live file gets only the real,
  approved build steps.

## Branches

- **Work on the branch the user named, not on main.** A new `session new` attaches to main. In
  every session and every script, switch to the task branch and confirm it with
  `framer.agent.getActiveBranch()` before the first write, then read back a node that exists only on
  that branch. If the branch cannot be confirmed, do not write.
- **Check the output of every write for `[FRAMER_BRANCH_CHANGE]`.** On a client build (2026-10-07)
  an agent write made while confirmed on the task branch opened a new sub-branch instead. Tell the
  user which branch the edit landed on; a parent cannot merge to main while a sub-branch is open.
- **Merge or publish only when the user asks**, even though the API allows both. Publishing is
  whole-site: it replaces everything the domain serves, including drafts other people have open,
  such as an unfinished blog post.
- **Leave project-wide settings alone:** branching mode, locales, domains, the plan, plugins. A
  file where the owner's own agent also works may need to stay on `Manual` branching (2026-09-21).

Branch mechanics and their traps are in [`setup-and-sessions.md`](setup-and-sessions.md) § How the
bridge talks to Framer.

## While building

- **Do not edit a shared component, text style, colour token or layout template** unless the user
  has cleared it and you know every page it reaches. One edit spreads to every page that uses it,
  and nav and footer sit on nearly every page. When a page needs a variant, duplicate the item into
  your own folder in the file.
- **Reuse before creating.** Search the file for a component, style, token or asset that already
  does the job. Anything new goes in one folder named for the work and is listed in the build notes,
  so the user can account for it to the site owner.
- **Hide, never delete.** A section being replaced goes into a hidden backup frame and stays there
  until the owner signs off. Deleting is irreversible and its reach is invisible from the API:
  removing a collection page also removes its CMS detail page. See
  [`destructive-operations.md`](destructive-operations.md).
- **CMS items are not test fixtures.** Collections synced by a plugin (from an applicant tracker,
  for one) are read-only.
- **Check side effects** on every page that uses a changed element, at all three breakpoints.
- **New copy reaches every locale.** A string added on a localised page lands untranslated in the
  other locales.

## After building

- Keep a copy of every code file written into the live file, and pull it back after each change;
  Framer stays the source of truth.
- Record what changed, on which branch, the node ids, and everything new that was created. Say
  what is waiting on the user: a merge, a publish, or a sub-branch to fold in.

The quality checklist for the build itself is [`build-checklist.md`](build-checklist.md),
including its section § In a client's existing file.
