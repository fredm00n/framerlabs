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

Build custom React components and Code Overrides in Framer.

**Topics covered:**
- Code Component vs Code Override patterns
- Required `@framer` annotations
- Font handling (critical: always spread the font object)
- Hydration safety and SSR
- Canvas vs Preview detection with `RenderTarget`
- All property control types with examples
- Conditional control visibility
- Shared state between overrides
- Scroll effects, magnetic hover, animation triggers
- WebGL transparency and shader compilation
- Mobile optimization, CMS content timing, Safari SVG fixes
- React Portals for z-index stacking context
- Loading states with scroll lock
- Easing curves for lerp animations
- HLS video streaming (.m3u8) with dynamic HLS.js import
- Variable-bound vs static text in overrides (`props.text` first, children fallback)
- Triggering Framer-attached handlers from code via React fiber traversal (e.g. URL deep link to an overlay)

**File structure:**
```
framer-code-components-overrides/
├── SKILL.md
└── references/
    ├── property-controls.md
    ├── patterns.md
    └── webgl-shaders.md
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
