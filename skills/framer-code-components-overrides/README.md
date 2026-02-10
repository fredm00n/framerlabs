# Framer Code Components & Overrides

A Claude Code skill for building custom React components and Code Overrides in Framer.

## What's Included

- **Code Components**: Custom React components with property controls
- **Code Overrides**: HOCs for modifying canvas elements
- **Property Controls**: Complete reference for all control types
- **WebGL/Shaders**: Shader implementation patterns with transparency support
- **Common Patterns**: Shared state, keyboard detection, scroll effects, animations
- **Pitfall Prevention**: Hydration errors, font handling, Safari fixes

## Installation

```bash
npx skills add fredm00n/framerlabs --skill framer-code-components-overrides
```

This installs the skill into your project's `.claude/skills/` folder.

<details>
<summary>Manual installation</summary>

This skill consists of multiple files. Download the entire `framer-code-components-overrides` folder, not just individual files.

**Project-based** (in `.claude/skills/`):
```bash
mkdir -p .claude/skills
cp -r /path/to/framer-code-components-overrides .claude/skills/
```

**Global** (available across all projects):
```bash
# Windows
cp -r /path/to/framer-code-components-overrides %USERPROFILE%\.claude\skills\

# macOS/Linux
cp -r /path/to/framer-code-components-overrides ~/.claude/skills/
```

</details>

## Usage

Once installed, the skill is automatically available. Claude Code will use it when you're working on Framer components, or you can invoke it directly:

```
/framer-code-components-overrides
```

## Skill Contents

```
framer-code-components-overrides/
├── SKILL.md                         # Main instructions
└── references/
    ├── property-controls.md         # All control types
    ├── patterns.md                  # Common implementations
    └── webgl-shaders.md             # Shader patterns
```

## Topics Covered

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
- Mobile optimization
- CMS content timing
- Safari SVG fixes
- React Portals for z-index stacking context
- Loading states with scroll lock
- Easing curves for lerp animations with initial distance tracking

## License

MIT
