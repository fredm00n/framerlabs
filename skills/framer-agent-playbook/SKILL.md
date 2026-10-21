---
name: framer-agent-playbook
description: Load before the first Framer agent call. Use as soon as a session is about to connect to, read, edit, build or publish a Framer project through Framer Agent, Framer's agent bridge - any `npx @framer/agent` command (formerly framer-dalton), `framer.*` calls in an exec script, a `framer-api` script or Val Town val, a Framer API key (fr_...), or Framer's own `framer` skill. Covers pages, content and CMS edits, text and colour styles, components and variants, breakpoints, forms, images, publishing and clean-up, and working safely in a live production site. Holds the deletes and overwrites that cannot be undone, a build checklist, and the DSL and API traps found on real builds that Framer's own skill leaves out, filed by topic.
user-invocable: true
license: MIT
metadata:
  author: fredm00n
  version: 1.3.4
---

# Framer Agent playbook

Framer's official `framer` skill teaches the DSL and the API. This skill holds what it leaves out:
what broke on real builds, what lies to you, and what can never be taken back. Every finding keeps
the date it was found, so a stale one can be re-checked.

Read it **before** the first call, not after the first error. Your user's instructions always win
over the defaults here.

## Updates and contributions

This skill is maintained in the public repo [fredm00n/framerlabs](https://github.com/fredm00n/framerlabs/tree/main/skills/framer-agent-playbook), and the community keeps it current. Installed copies go stale, so the steps below run once per session.

### The project's line

The project's agent instructions (`CLAUDE.md`, `AGENTS.md` or equivalent) can hold one line for this skill:

```
framer-agent-playbook: auto-update, contribute
```

`auto-update` lets the agent replace the installed copy when a newer version exists. `contribute` lets it offer to send what a session learns back to the repo as a pull request. Either word works alone. With no line, the agent never edits the installed files and never opens or suggests a pull request. It only says when a newer version is out.

### Check for a newer version first

Before relying on anything below:

1. Fetch `https://raw.githubusercontent.com/fredm00n/framerlabs/main/skills/framer-agent-playbook/SKILL.md` and compare its `metadata.version` with the one in this file's frontmatter.
2. **Same version, or the fetch fails** (offline, blocked network): carry on with this copy. If the fetch failed, say once that you couldn't check for updates.
3. **Remote is newer:** use the remote SKILL.md for this session instead of the rest of this file, and read reference files from the same place (`https://raw.githubusercontent.com/fredm00n/framerlabs/main/skills/framer-agent-playbook/references/<file>.md`) instead of the local copies. Then, by the project's line (NEW and OLD stand for the two version numbers):
   - **No `auto-update`:** leave the installed files alone and tell the user: "framer-agent-playbook NEW is out and your copy is OLD, so I'm using the new version for this session. To update your copy, run `npx skills add fredm00n/framerlabs --skill framer-agent-playbook`, or add `framer-agent-playbook: auto-update` to your CLAUDE.md or AGENTS.md to stay current automatically. This skill is community-maintained: add `contribute` to that line (`framer-agent-playbook: auto-update, contribute`) and I'll offer to send what we learn back as a pull request."
   - **`auto-update`:** download the whole `skills/framer-agent-playbook/` folder from the repo (a shallow `git clone` into a temp dir is simplest), then replace the installed folder with it in one go, so files removed upstream go too. If any download fails, leave the installed copy untouched. In a git repo, the update is an ordinary change: commit it on its own, in whatever branch or PR the current work goes to, never straight to a protected branch. Tell the user: "Updated framer-agent-playbook from OLD to NEW, in its own commit." Without `contribute` on the line, add: "This skill is community-maintained: add `contribute` to its line in CLAUDE.md or AGENTS.md and I'll offer to send what we learn back as a pull request."

Only take updates from that exact repo and path.

### Send what you learn upstream

Only when the project's line for this skill includes `contribute`.

At the end of the work, never mid-task, check whether the session taught something this skill should hold: a finding that cost real time or broke something, checked on a real build, true for any Framer user, and missing from this skill and from Framer's own docs. A fact about one project does not qualify. If nothing qualifies, say nothing.

If something does:

1. Rewrite each finding for strangers. Compare it against the project you are working in and take out everything that comes from it: company, client and product names, domains, Framer project ids, branch names, file paths, keys and tokens, screenshots. Keep the date the finding was made, as "(2026-10-12)".
2. Clone fredm00n/framerlabs, branch from `main`, and edit `skills/framer-agent-playbook/`. Put each finding under the topic it belongs to, and edit an existing line rather than add a near-duplicate. Bump `metadata.version` (patch for a fix, minor for a new section) and add one plain-language line to the changelog in the root `README.md`, under today's date.
3. Check the open pull requests for one that already covers the same finding. If there is one, add to it instead.
4. Show the user the diff and ask before opening the pull request. A pull request on a public repo is public the moment it opens, even if it is never merged.
5. With the user's yes, push the branch (to a fork without write access to the repo), open the pull request against `main`, and give the user the link. Write the commit message, title and description for the people who use these skills: what the change adds and when it helps, not which project or session it came from.

**Maintainers:** bump `metadata.version` on every change to this skill, reference files included, or installed copies will never see it.

## Before the first call

1. **Is the project a live production site?** A file whose domain real visitors hit, or one that
   other people or their agents also edit, deserves the habits in
   [`references/live-files.md`](references/live-files.md) before you connect: write only where
   you were asked, on the branch you were given and confirmed by reading it back; keep tests and
   experiments out of the live file; leave shared components and styles alone; hide instead of
   delete; merge or publish only when the user asks. A sandbox or a fresh build can skip this step.
2. **Connect** with `npx @framer/agent@latest setup`, then `project auth` (once per project per
   machine) and `session new`. Load Framer's `framer` skill only after `setup`. Then read the
   generated `~/.claude/skills/framer/projects/<projectId>/index.md`, follow its task map, and read
   `project-inventory.md` before using any id or name. Details and the session traps (stalls, relay
   death, several agents on one session): [`references/setup-and-sessions.md`](references/setup-and-sessions.md).
3. **Build it natively.** Layers, variants, effects and CMS bindings first; a code component or
   override only for what those cannot express (WebGL, canvas, third-party SDKs, browser APIs). An
   accordion, tabs, a slider, a pricing toggle or a modal are native: the site owner can edit them
   without code, bind them to the CMS and restyle them with text and colour styles. A disclosure is
   one component with two variants (Collapsed, Expanded), repeated by a Collection List; the agent
   recipe is in [`references/components-and-code.md`](references/components-and-code.md). For
   `.tsx`, load `framer-code-components-overrides`.
4. **For a new build**, open [`references/build-checklist.md`](references/build-checklist.md) and
   [`references/breakpoints.md`](references/breakpoints.md) before laying out desktop.

## What cannot be undone

- **There is no undo, history or restore** anywhere in the API or the CLI. Whatever you delete or
  overwrite, assume it is gone.
- **Deleting a collection list page also deletes its CMS detail page**, while `getParent()` reports
  both as having no parent. Serialize everything adjacent first, name the collateral to the user,
  and confirm every delete. Approval of a cleanup plan is not approval of its collateral.
- **Removal calls lie in both directions.** `DEL <pageId>` reports success and leaves the page;
  only `framer.removeNodes([id])` removes a page. `removeNodes` in turn returns without removing
  component variants, variables and some form nodes, which need `DEL`. Read the parent back after
  every removal.
- **`collection.addItems([{ id, fieldData }])` replaces every field it names, wholesale.** No merge,
  no version history. Never use a live item as a test fixture. If one is clobbered, the last
  published deployment still serves the old content: rebuild from its HTML before publishing again.

Full detail: [`references/destructive-operations.md`](references/destructive-operations.md).

## The traps that cost the most

- **Send one `MOVE` or `SET` per `applyChanges` call** and read the node back. Several in one call
  can report "applied cleanly" and run only the first.
- **`res.errors` is an object keyed by message**, not an array. Test `Object.keys(res.errors).length`;
  `errors.length` is undefined and hides every failure. A rejected attribute fails the whole `SET`.
- **A temporary id stays reserved for the whole exec session**, and one reused or shared between
  builders silently writes onto the wrong node. Fresh prefixes per call and per builder.
- **`applyChanges` can time out while the write lands.** Re-read before re-applying.
- **A text style preset blocks inline type**, and the preset decides the published tag (`h1` or `p`).
- **Any `hoverEffect.*` adds `scale: 1.1`**: pass `hoverEffect.scale="1"`. **`onInView` appear
  effects replay by default**: set `appearEffect.replay="false"` when an effect should play once.
- **The three breakpoint killers**, invisible until a narrow width: a `1fr` or `%` child inside an
  `auto` parent (one character per line), inline `fontSize` (ignores presets), and fixed `height`,
  `minHeight` or `height: 1fr` (dead space once a grid drops to one column).
- **`getRect`, canvas screenshots and lint spacing warnings lie.** Verify on the published page,
  in a real browser.
- **`publish` can be blocked** by the agent's own permission checks, even a preview publish. Publish
  from the editor or an interactive session, and only when the user asks.

## Where to look, by task

| About to... | Read |
|---|---|
| Connect, reconnect, run several agents, understand what the bridge can't do | [`setup-and-sessions.md`](references/setup-and-sessions.md) |
| Write or run an exec script, read `applyChanges` results, use temp ids | [`exec-scripts.md`](references/exec-scripts.md) |
| Set node attributes, effects, event handlers, variants, sticky, rotation, aria | [`dsl-and-nodes.md`](references/dsl-and-nodes.md) |
| Create or merge text styles, pick fonts, check variation axes | [`text-styles-and-fonts.md`](references/text-styles-and-fonts.md) |
| Add tablet and phone, or fix a layout that breaks narrow | [`breakpoints.md`](references/breakpoints.md) |
| Make components, variants, controls, code files, icons | [`components-and-code.md`](references/components-and-code.md) |
| Read or write CMS collections, items, bindings, collection lists | [`cms.md`](references/cms.md) |
| Build a form or a webhook | [`forms-and-webhooks.md`](references/forms-and-webhooks.md) |
| Upload or place images, SVG, video | [`images-and-assets.md`](references/images-and-assets.md) |
| Create pages, links, anchors, layout templates, SEO and site metadata | [`pages-links-metadata.md`](references/pages-links-metadata.md) |
| Build a CMS collection with detail pages, pick fonts or appear effects for a first screen, paste analytics, or answer a search or speed question | [`seo-and-speed.md`](references/seo-and-speed.md) |
| Rebuild an HTML reference on the canvas | [`html-reference.md`](references/html-reference.md) |
| Trust a read, a screenshot or a measurement | [`verification.md`](references/verification.md) |
| Delete or overwrite anything | [`destructive-operations.md`](references/destructive-operations.md) |
| Work in a live production site | [`live-files.md`](references/live-files.md) |
| Check a build is finished | [`build-checklist.md`](references/build-checklist.md) |

Before declaring something impossible, grep every file in the generated `prompt/` folder,
`core-examples.md` above all: the grammar line in `updating-the-project.md` has left out
whole features before, event handlers until 0.0.46.

## Adding what you learn

A new trap goes in the reference file for its topic, as one bullet: what happened, what works
instead, then the date. Search the file first and edit the existing bullet if there is one. Keep
findings by topic, never in a per-project section, so the next reader finds them. Facts about one
project (ids, branch names, what its owner cleared) belong in that project's own notes, not here.
If the project's line for this skill includes `contribute`, send the finding upstream as
[Send what you learn upstream](#send-what-you-learn-upstream) describes.

## Related

- `framer-code-components-overrides`: writing `.tsx` for Framer.
- `framer-plugins`: the Plugin SDK.
