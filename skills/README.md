# Framer Labs Skills

Claude Code skills that teach Claude how to build Framer components and plugins. Install these to get better AI assistance when working on Framer projects.

## Available Skills

| Skill | Description |
|-------|-------------|
| [framer-code-components-overrides](framer-code-components-overrides/) | Code Components, Code Overrides, property controls, WebGL shaders, and common patterns |
| [framer-plugins](framer-plugins/) | Framer Plugin SDK — ManagedCollection API, CMS sync, plugin modes, UI patterns, permissions, Marketplace submission |

## Installation

Install via [skills.sh](https://skills.sh):

```bash
# Code Components & Overrides
npx skills add fredm00n/framerlabs --skill framer-code-components-overrides

# Framer Plugins
npx skills add fredm00n/framerlabs --skill framer-plugins
```

This installs the skill into your project's `.claude/skills/` folder.

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
- Authoring — font handling (always spread the font object), `ControlType.Color` token unwrapping, full property-controls reference
- Rendering & SSR — hydration safety with SSR-init trap, `RenderTarget` canvas vs preview detection, `startTransition` for concurrent rendering, NPM imports
- CMS — variable-bound vs static text in overrides, CMS content timing, full CMS-in-code-components pattern (`useQueryData` + `findByFramerName`, plain-frame trap, canvas preview limitation)
- Overrides — variant control without prop access, triggering Framer-attached handlers via React fiber traversal (e.g. URL deep link to an overlay)
- DOM & Performance — scroll detection constraint, `useRef` vs `useState` for live-read targets (`useScroll` etc.), React Portals for z-index stacking context, writing state into the URL to drive Framer's native filters, styling native range inputs (cross-engine thumbs, dual-thumb technique), common patterns library
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
- Plugin modes (canvas, CMS, image, collection)
- Core `framer` API (UI management, canvas methods)
- ManagedCollection API (fields, items, upsert behavior)
- Field types and field data values
- Permissions system (`useIsAllowedTo`, `isAllowedTo`)
- Data storage decision tree (localStorage vs pluginData)
- CMS sync patterns from 32 official examples
- Common pitfalls and workarounds
- Marketplace submission workflow, listing asset specs, mandatory requirements, and pre-submission checklist

**File structure:**
```
framer-plugins/
├── SKILL.md
└── references/
    ├── api-reference.md
    ├── patterns.md
    ├── pitfalls.md
    └── marketplace.md
```

## License

MIT
