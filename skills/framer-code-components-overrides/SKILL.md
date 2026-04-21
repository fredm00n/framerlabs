---
name: framer-code-components-overrides
description: "Create Framer Code Components and Code Overrides. Use when building custom React components for Framer, writing Code Overrides (HOCs) to modify canvas elements, implementing property controls, working with Framer Motion animations, handling WebGL/shaders in Framer, or debugging Framer-specific issues like hydration errors and font handling."
user-invocable: true
triggers:
  - "framer component"
  - "code component"
  - "code override"
  - "property controls"
  - "framer motion"
  - "webgl shader"
  - "hydration error"
  - "font handling framer"
license: MIT
metadata:
  author: fredm00n
  version: 1.0.0
---

# Framer Code Development

## Code Components vs Code Overrides

**Code Components**: Custom React components added to canvas. Support `addPropertyControls`.

**Code Overrides**: Higher-order components wrapping existing canvas elements. Do NOT support `addPropertyControls`.

## Development Workflow

1. **Create** component/override file with required `@framer` annotations
2. **Add property controls** via `addPropertyControls` (components only — overrides don't support them)
3. **Handle SSR** — wrap browser APIs in `isClient` state guard to prevent hydration mismatches
4. **Test both contexts** — check `RenderTarget.current()` to verify behavior on canvas vs preview
5. **Optimize for mobile** — add resize debouncing and touch handling for heavy animations

## Required Annotations

Always include at minimum:
```typescript
/**
 * @framerDisableUnlink
 * @framerIntrinsicWidth 100
 * @framerIntrinsicHeight 100
 */
```

Full set:
- `@framerDisableUnlink` — Prevents unlinking when modified
- `@framerIntrinsicWidth` / `@framerIntrinsicHeight` — Default dimensions
- `@framerSupportedLayoutWidth` / `@framerSupportedLayoutHeight` — `any`, `auto`, `fixed`, `any-prefer-fixed`

## Code Override Pattern

```typescript
import type { ComponentType } from "react"
import { useState, useEffect } from "react"

/**
 * @framerDisableUnlink
 */
export function withFeatureName(Component): ComponentType {
    return (props) => {
        // State and logic here
        return <Component {...props} />
    }
}
```

Naming: Always use `withFeatureName` prefix.

## Code Component Pattern

```typescript
import { motion } from "framer-motion"
import { addPropertyControls, ControlType } from "framer"

/**
 * @framerDisableUnlink
 * @framerIntrinsicWidth 300
 * @framerIntrinsicHeight 200
 */
export default function MyComponent(props) {
    const { style } = props
    return <motion.div style={{ ...style }}>{/* content */}</motion.div>
}

MyComponent.defaultProps = {
    // Always define defaults
}

addPropertyControls(MyComponent, {
    // Controls here
})
```

## Critical: Font Handling

**Never access font properties individually. Always spread the entire font object.**

```typescript
// ❌ BROKEN - Will not work
style={{
    fontFamily: props.font.fontFamily,
    fontSize: props.font.fontSize,
}}

// ✅ CORRECT - Spread entire object
style={{
    ...props.font,
}}
```

Font control definition:
```typescript
font: {
    type: ControlType.Font,
    controls: "extended",
    defaultValue: {
        fontFamily: "Inter",
        fontWeight: 500,
        fontSize: 16,
        lineHeight: "1.5em",
    },
}
```

## Critical: Wrap State Updates in startTransition

All React state updates in Framer must be wrapped in `startTransition()`:

```typescript
import { startTransition } from "react"

// ❌ WRONG - May cause issues in Framer's rendering pipeline
setCount(count + 1)

// ✅ CORRECT - Always wrap state updates
startTransition(() => {
    setCount(count + 1)
})
```

This is Framer-specific and prevents performance issues with concurrent rendering.

## Critical: Hydration Safety

Framer pre-renders on server. Browser APIs unavailable during SSR.

**Two-phase rendering pattern:**
```typescript
const [isClient, setIsClient] = useState(false)

useEffect(() => {
    setIsClient(true)
}, [])

if (!isClient) {
    return <Component {...props} /> // SSR-safe fallback
}

// Client-only logic here
```

**Never access directly at render time:**
- `window`, `document`, `navigator`
- `localStorage`, `sessionStorage`
- `window.innerWidth`, `window.innerHeight`

## Critical: Canvas vs Preview Detection

```typescript
import { RenderTarget } from "framer"

const isOnCanvas = RenderTarget.current() === RenderTarget.canvas

// Show debug only in editor
{isOnCanvas && <DebugOverlay />}
```

Use for:
- Debug overlays
- Disabling heavy effects in editor
- Preview toggles

## Property Controls Reference

See [references/property-controls.md](references/property-controls.md) for complete control types and patterns.

## Common Patterns

See [references/patterns.md](references/patterns.md) for implementations: shared state, keyboard detection, show-once logic, scroll effects, magnetic hover, animation triggers.

## Variant Control in Overrides

Cannot read variant names from props (may be hashed). Manage internally:

```typescript
export function withVariantControl(Component): ComponentType {
    return (props) => {
        const [currentVariant, setCurrentVariant] = useState("variant-1")

        // Logic to change variant
        setCurrentVariant("variant-2")

        return <Component {...props} variant={currentVariant} />
    }
}
```

## Scroll Detection Constraint

Framer's scroll detection uses viewport-based IntersectionObserver. Applying `overflow: scroll` to containers breaks this detection.

For scroll-triggered animations, use:
```typescript
const observer = new IntersectionObserver(
    (entries) => {
        entries.forEach((entry) => {
            if (entry.isIntersecting && !hasEntered) {
                setHasEntered(true)
            }
        })
    },
    { threshold: 0.1 }
)
```

## WebGL in Framer

See [references/webgl-shaders.md](references/webgl-shaders.md) for shader implementation patterns including transparency handling.

## NPM Package Imports

Standard import (preferred):
```typescript
import { Component } from "package-name"
```

Force specific version via CDN when Framer cache is stuck:
```typescript
import { Component } from "https://esm.sh/package-name@1.2.3?external=react,react-dom"
```

Always include `?external=react,react-dom` for React components.

## HLS Video Streaming

Chrome/Firefox don't natively support HLS. See [references/advanced-patterns.md](references/advanced-patterns.md#hls-video-streaming-m3u8) for the HLS.js dynamic import pattern with silent fallback and cleanup.

## Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| Font styles not applying | Accessing font props individually | Spread entire font object: `...props.font` |
| Hydration mismatch | Browser API in render | Use `isClient` state pattern |
| Override props undefined | Expecting property controls | Overrides don't support `addPropertyControls` |
| Scroll animation broken | `overflow: scroll` on container | Use IntersectionObserver on viewport |
| Shader attach error | Null shader from compilation failure | Check `createShader()` return before `attachShader()` |
| Component display name | Need custom name in Framer UI | `Component.displayName = "Name"` |
| TypeScript `Timeout` errors | Using `NodeJS.Timeout` type | Use `number` instead — browser environment |
| Overlay stuck under content | Stacking context from parent | Use React Portal to render at `document.body` level |
| Easing feels same for all curves | Not tracking initial distance | Track `initialDiff` when target changes for progress calculation |
| HLS video permanently pixelated | `.m3u8` in Chrome without HLS.js | Use HLS.js dynamic import pattern (see HLS section above) |

## Mobile Optimization

For particle systems and heavy animations:
- Implement resize debouncing (500ms default)
- Add size change threshold (15% minimum)
- Handle orientation changes with dedicated listener
- Use `touchAction: "none"` to prevent scroll interference

## CMS Content Timing

CMS content loads asynchronously after hydration. Processing sequence:
1. SSR: Placeholder content
2. Hydration: React attaches
3. CMS Load: Real content (~50-200ms)

Add delay before processing CMS data:
```typescript
useEffect(() => {
    if (isClient && props.children) {
        const timer = setTimeout(() => {
            processContent(props.children)
        }, 100)
        return () => clearTimeout(timer)
    }
}, [isClient, props.children])
```

## Text Manipulation in Overrides

Framer text uses deeply nested structure. Process recursively:

```typescript
const processChildren = (children) => {
    if (typeof children === "string") {
        return processText(children)
    }
    if (isValidElement(children)) {
        return cloneElement(children, {
            ...children.props,
            children: processChildren(children.props.children)
        })
    }
    if (Array.isArray(children)) {
        return children.map(child => processChildren(child))
    }
    return children
}
```

## Animation Best Practices

**Separate positioning from animation:**
```typescript
<motion.div
    style={{
        position: "absolute",
        left: `${offset}px`,  // Static positioning
        x: animatedValue,     // Animation transform
    }}
/>
```

**Split animation phases for natural motion:**
```typescript
// Up: snappy pop
transition={{ duration: 0.15, ease: [0, 0, 0.39, 2.99] }}

// Down: smooth settle
transition={{ duration: 0.15, ease: [0.25, 0.46, 0.45, 0.94] }}
```

## Safari SVG Fix

Force GPU acceleration for smooth SVG animations:
```typescript
style={{
    willChange: "transform",
    transform: "translateZ(0)",
    backfaceVisibility: "hidden",
}}
```

## Z-Index Stacking Context & React Portals

Parent stacking contexts trap `z-index`. Use React Portals to render overlays at `document.body` level. See [references/advanced-patterns.md](references/advanced-patterns.md#z-index-stacking-context--react-portals) for the full portal pattern.

## Loading States with Scroll Lock

Two-phase loading overlay with scroll prevention and fade-out transition. See [references/advanced-patterns.md](references/advanced-patterns.md#loading-states-with-scroll-lock) for the implementation including scroll-to-top on load.

## Easing Curves with Lerp Animations

Exponential lerp naturally produces ease-out. For ease-in or ease-in-out, track `initialDiff` when the target changes. See [references/advanced-patterns.md](references/advanced-patterns.md#easing-curves-with-lerp-animations) for the full easing implementation with property controls.
