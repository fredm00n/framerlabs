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

### Option A: Project-based (recommended)

Copy the skill into your project's `.claude/skills/` folder:

```bash
# From your project root
mkdir -p .claude/skills
cp -r /path/to/framer-code-components-overrides .claude/skills/
```

The skill will only be available when Claude Code runs in that project.

### Option B: Global

Copy the skill into your personal Claude Code skills folder:

```bash
# Windows
cp -r /path/to/framer-code-components-overrides %USERPROFILE%\.claude\skills\

# macOS/Linux
cp -r /path/to/framer-code-components-overrides ~/.claude/skills/
```

The skill will be available across all your projects.

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

## License

MIT
