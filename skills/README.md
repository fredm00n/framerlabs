# Framer Labs Skills

Agent skills that teach AI coding agents how to build Framer sites, components and plugins. Install these to get better AI assistance when working on Framer projects.

## Available Skills

| Skill | Description |
|-------|-------------|
| [framer-code-components-overrides](framer-code-components-overrides/) | Code Components, Code Overrides, property controls, WebGL shaders, and common patterns |
| [framer-agent-playbook](framer-agent-playbook/) | Building and editing Framer sites through Framer Agent — what can't be undone, DSL and API traps, breakpoints, CMS, forms, and working safely in live client sites |
| [framer-plugins](framer-plugins/) | Framer Plugin SDK — plugin modes, canvas insertion + code files, CMS/ManagedCollection sync, permissions, licensing, Marketplace submission |

## Installation

Install via [skills.sh](https://skills.sh):

```bash
# Code Components & Overrides
npx skills add fredm00n/framerlabs --skill framer-code-components-overrides

# Framer Plugins
npx skills add fredm00n/framerlabs --skill framer-plugins

# Framer Agent playbook
npx skills add fredm00n/framerlabs --skill framer-agent-playbook
```

This installs the skill into your project's `.claude/skills/` folder.

### Updates and contributions

Every skill checks this repo for a newer version once per session and uses the latest copy if yours is stale. One line per skill in your project's `CLAUDE.md` or `AGENTS.md` decides what else it may do:

```
framer-agent-playbook: auto-update, contribute
```

- `auto-update`: replace the installed folder with the newer version, in its own commit.
- `contribute`: at the end of a session, offer to open a pull request here with what the session learned, rewritten without anything from your project, and only after you've seen the diff and said yes. Without write access, the agent works from a fork.

Either word works alone. With no line, the skill tells you when an update exists and does nothing else.

<details>
<summary>Manual installation</summary>

Each skill consists of multiple files. Download the entire skill folder, not just individual files.

**Project-based** (in `.claude/skills/`):
```bash
mkdir -p .claude/skills
cp -r /path/to/<skill-name> .claude/skills/
```

**Global** (available across all projects):
```bash
# Windows
cp -r /path/to/<skill-name> %USERPROFILE%\.claude\skills\

# macOS/Linux
cp -r /path/to/<skill-name> ~/.claude/skills/
```

</details>

## framer-code-components-overrides

Build custom React components and Code Overrides in Framer. Sections are tagged `[C]` (code component), `[O]` (code override), or `[C/O]` (both).

**Topics covered:**
- Foundations — Code Component vs Code Override, required `@framer` annotations, starter templates
- Authoring — font handling (always spread the font object), `ControlType.Color` token unwrapping, full property-controls reference (including separator lines)
- Rendering & SSR — hydration safety with SSR-init trap, why `useId()` must never link elements (overrides that go dead after a publish), inline style tags that React escapes, `RenderTarget` canvas vs preview detection, `startTransition` for concurrent rendering, NPM imports
- CMS — variable-bound vs static text in overrides, CMS content timing, full CMS-in-code-components pattern (`useQueryData` + `findByFramerName`, plain-frame trap, canvas preview limitation), and why every CMS field is public in published data files
- Overrides — variant control without prop access, why an override produced by a factory won't appear in Framer's override picker (must be a literal export), triggering Framer-attached handlers via React fiber traversal (e.g. URL deep link to an overlay), driving component instances (`onClick` over `onTap`, `variant` by name, scoped CSS), how native Framer forms submit and how to add a computed hidden field
- DOM & Performance — scroll detection constraint, `useRef` vs `useState` for live-read targets (`useScroll` etc.), React Portals for z-index stacking context, writing state into the URL to drive Framer's native filters, styling native range inputs (cross-engine thumbs, dual-thumb technique), measuring children through Framer's `display: contents` wrappers, adding DOM next to React's nodes without breaking it, showing new CMS items without a reload, common patterns library
- Media — HLS video streaming (`.m3u8`) with dynamic HLS.js import, WebGL transparency and shader compilation
- Debug — gated `console.log` via module-level `debugMode` flag
- Common patterns library — shared state, scroll effects, magnetic hover, animation triggers, mobile optimization, Safari SVG fix, loading states with scroll lock, easing curves for lerp animations

**File structure:**
```
framer-code-components-overrides/
├── SKILL.md
└── references/
    ├── property-controls.md
    ├── patterns.md
    ├── webgl-shaders.md
    ├── cms.md              # CMS-in-code-components pattern
    ├── fiber-handlers.md   # Triggering Framer handlers via fiber traversal
    └── hls-video.md        # HLS.js dynamic-import video streaming
```

## framer-plugins

Build, debug, and modify Framer plugins using the Plugin SDK.

**Topics covered:**
- Starting a new plugin (scaffolder, picking modes, the duplicate-`id` cloning trap)
- Plugin modes (canvas, CMS, image, collection)
- Core `framer` API (UI management, canvas insertion methods)
- Canvas code files & hosted modules — `createCodeFile`, matching instances by stable identifier, cross-project module URLs
- ManagedCollection API, field types, the silent-sync algorithm, slugs (in its own CMS reference)
- Permissions system (`useIsAllowedTo`, `isAllowedTo`) and the permission-gating contract the Marketplace's automated code review expects
- Data storage decision tree (localStorage vs pluginData), licensed-plugin project-remix handling, and server-side licensing keyed on the Framer user id
- CMS sync patterns from official examples
- Common pitfalls and workarounds
- Marketplace submission: workflow, listing asset specs, how the automated post-publication review works (shipping sources, greppable permission guards), submission identity gotchas (manifest ids, name collisions), and a pre-submission checklist

**File structure:**
```
framer-plugins/
├── SKILL.md
└── references/
    ├── api-reference.md
    ├── cms-managed-collections.md   # CMS plugins: ManagedCollection, sync, field types, slugs
    ├── patterns.md
    ├── pitfalls.md
    └── marketplace.md
```

## framer-agent-playbook

Edit and build Framer sites through Framer Agent (`npx @framer/agent`). It sits next to Framer's own `framer` skill, which teaches the DSL and the API, and holds what that skill leaves out: what broke on real builds, what reports success and does nothing, and what can never be taken back. Every finding carries the date it was found.

**Topics covered:**
- Before the first call: connecting, sessions, several agents on one project, and what the bridge cannot do
- What cannot be undone: deletes and overwrites with no history, a collection page that takes its CMS detail page with it, removal calls that report success and remove nothing, `addItems` replacing whole CMS items
- Exec scripts and the DSL: one change per `applyChanges`, reading errors correctly, temporary ids, effects, event handlers, variants, sticky, rotation, accessibility
- Text styles and fonts, components and code files, CMS collections and bindings, forms and webhooks, images, SVG and video, pages, links and SEO
- Breakpoints: building tablet and phone, and the three layout traps that only show at narrow widths
- Rebuilding an HTML reference on the canvas, and which reads, measurements and screenshots to trust
- Working in a live production site: branches, shared components, hiding instead of deleting, merging and publishing only when asked
- A build checklist for knowing a page is finished

**File structure:**
```
framer-agent-playbook/
├── SKILL.md
└── references/
    ├── setup-and-sessions.md
    ├── exec-scripts.md
    ├── dsl-and-nodes.md
    ├── text-styles-and-fonts.md
    ├── breakpoints.md
    ├── components-and-code.md
    ├── cms.md
    ├── forms-and-webhooks.md
    ├── images-and-assets.md
    ├── pages-links-metadata.md
    ├── html-reference.md
    ├── verification.md
    ├── destructive-operations.md
    ├── live-files.md
    └── build-checklist.md
```

## License

MIT
