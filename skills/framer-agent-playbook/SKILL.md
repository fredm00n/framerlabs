---
name: framer-agent-playbook
description: Load before the first Framer agent call. Use as soon as a session is about to connect to, read, edit, build or publish a Framer project through Framer Agent, Framer's agent bridge - any `npx @framer/agent` command (formerly framer-dalton), `framer.*` calls in an exec script, a `framer-api` script or Val Town val, a Framer API key (fr_...), or Framer's own `framer` skill. Covers pages, content and CMS edits, text and colour styles, components and variants, breakpoints, forms, images, publishing and clean-up, and working safely in a live production site. Holds the deletes and overwrites that cannot be undone, a build checklist, and the DSL and API traps found on real builds that Framer's own skill leaves out, filed by topic.
user-invocable: true
license: MIT
metadata:
  author: fredm00n
  version: 1.0.0
---

# Framer Agent playbook

Framer's official `framer` skill teaches the DSL and the API. This skill holds what it leaves out:
what broke on real builds, what lies to you, and what can never be taken back. Every finding keeps
the date it was found, so a stale one can be re-checked.

Read it **before** the first call, not after the first error. Your user's instructions always win
over the defaults here.

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
   one component with two variants (Collapsed, Expanded), repeated by a Collection List. For `.tsx`,
   load `framer-code-components-overrides`.
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

## Related

- `framer-code-components-overrides`: writing `.tsx` for Framer.
- `framer-plugins`: the Plugin SDK.
