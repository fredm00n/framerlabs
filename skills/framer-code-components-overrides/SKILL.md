---
name: framer-code-components-overrides
description: Create Framer Code Components and Code Overrides. Use when building custom React components for Framer, writing Code Overrides (HOCs) to modify canvas elements, implementing property controls, working with Framer Motion animations, handling WebGL/shaders in Framer, or debugging Framer-specific issues like hydration errors and font handling.
---

# Framer Code Development

## Code Components vs Code Overrides

**Code Components**: Custom React components added to canvas. Support `addPropertyControls`.

**Code Overrides**: Higher-order components wrapping existing canvas elements. Do NOT support `addPropertyControls`.

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

## HLS Video Streaming (.m3u8)

Chrome/Firefox do **not** natively support HLS streams. A plain `<video src="...m3u8">` will either fail or play the lowest quality rendition permanently. Safari handles HLS natively.

**Fix:** Use HLS.js via dynamic import with silent fallback:

```typescript
let HlsModule = null
let hlsImportAttempted = false

async function loadHls() {
    if (hlsImportAttempted) return HlsModule
    hlsImportAttempted = true
    try {
        const mod = await import("https://esm.sh/hls.js@1?external=react,react-dom")
        HlsModule = mod.default || mod
    } catch {
        HlsModule = null // Fallback to native video
    }
    return HlsModule
}

function attachHls(videoEl, src) {
    if (typeof window === "undefined") return null // SSR guard
    const Hls = HlsModule
    if (src.includes(".m3u8") && Hls?.isSupported()) {
        const hls = new Hls({ startLevel: -1, capLevelToPlayerSize: true })
        hls.loadSource(src)
        hls.attachMedia(videoEl)
        hls.on(Hls.Events.MANIFEST_PARSED, () => videoEl.play().catch(() => {}))
        hls.on(Hls.Events.ERROR, (_, data) => {
            if (data.fatal) {
                data.type === Hls.ErrorTypes.NETWORK_ERROR
                    ? hls.startLoad()
                    : hls.destroy()
            }
        })
        return hls
    }
    videoEl.src = src // MP4/webm or Safari native HLS
    videoEl.play().catch(() => {})
    return null
}
```

**Key points:**
- Dynamic import avoids breaking the component if CDN is unreachable
- `capLevelToPlayerSize: true` prevents loading 4K for a 400px player
- Must destroy HLS instances on cleanup to prevent memory leaks
- Use `cancelled` flag in effects to prevent stale attachment after fast navigation
- Works on Framer canvas and published site

## Common Pitfalls

| Issue | Cause | Fix |
|-------|-------|-----|
| Variable text not found in override | Reading only `props.children` | Check `props.text` first — variable-bound text bypasses children |
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
| Overlay stuck "half-pressed" / needs two clicks to close | Triggering Framer interactions with synthetic events (`dispatchEvent`) | Call the React handler directly via fiber traversal (see "Triggering Framer-Attached Handlers") |

## Mobile Optimization

For particle systems and heavy animations:
- Implement resize debouncing (500ms default)
- Add size change threshold (15% minimum)
- Handle orientation changes with dedicated listener
- Use `touchAction: "none"` to prevent scroll interference

## CMS Content Timing

CMS text arrives in `props.text` asynchronously (~50–200ms after hydration). For variable-bound text from component props, it's synchronous on first render — no delay needed.

The reliable pattern for both: use `resolvePlainText(props)` (see Text in Overrides) and gate on the value being non-empty:

```typescript
const plainText = resolvePlainText(props)
// plainText is "" until content arrives → gate your animation on plainText.length > 0
```

Avoid 100ms arbitrary delays — they cause race conditions when the element is already in the viewport on load.

## Text in Overrides

**Text comes from two different sources depending on how it's set:**

| Source | Where it lives | When |
|--------|---------------|------|
| Static text (typed in Framer) | `props.children` nested structure | Always available on first render |
| Variable-bound text (component prop / CMS) | `props.text` (plain string) | Available on first render for variables; async for CMS |

**Always check `props.text` first, fall back to children:**

```typescript
import { isValidElement } from "react"

function extractParts(raw: any): any[] {
    if (typeof raw === "string") return [raw]
    if (isValidElement(raw)) return [raw]
    if (Array.isArray(raw)) return raw.flatMap(extractParts)
    return []
}

function toPlainText(parts: any[]): string {
    return parts.map((p) => (typeof p === "string" ? p : "\n")).join("")
}

function resolvePlainText(props: any): string {
    if (typeof props.text === "string" && props.text.length > 0) {
        return props.text  // variable-bound or CMS
    }
    const raw = props.children?.props?.children?.props?.children
    return toPlainText(extractParts(raw))  // static text
}
```

**Never assume text is only in `props.children`.** Variable-bound text bypasses the children structure entirely — `props.children` will contain a placeholder while `props.text` has the real value. If you only read children, variable text is invisible to your override.

## Triggering Framer-Attached Handlers from Code

When you need to programmatically fire a Framer/Framer Motion interaction (open an overlay, trigger a tap, etc.), **synthetic DOM events do not work reliably**. Framer Motion attaches handlers like `onTap` as React handlers, not native DOM listeners — synthetic events take a different code path and leave Framer Motion's internal state desynchronised. Symptoms include stuck press/focus state, two-click-to-close bugs, and other "half-pressed" weirdness that persists for the rest of the session on that element.

**Reach into the React fiber tree and call the handler directly:**

```typescript
function findFiberHandler(el: HTMLElement, name: string): unknown {
    const key = Object.keys(el).find((k) => k.startsWith("__reactFiber"))
    if (!key) return undefined
    let fiber: any = (el as any)[key]
    let depth = 0
    while (fiber && depth < 15) {
        const p = fiber.memoizedProps
        if (p && typeof p[name] === "function") return p[name]
        fiber = fiber.return
        depth++
    }
    return undefined
}

const onTap = findFiberHandler(wrapper, "onTap")
onTap?.({} as any, {} as any)
```

**Why walk `fiber.return`:** Framer wraps interactive elements in Framer Motion components several fiber levels above the rendered DOM node. The DOM wrapper does not carry `onTap` in its own props — you have to walk up to find it. In practice the handler lives ~2 levels up; 15 is a safe ceiling.

In Framer, overlay triggers render as DOM nodes with `tabindex="0"` and an id, so `el.closest("[tabindex]")` is a reliable way to find the wrapper from a child override.

### Use case: URL deep link to an overlay

Apply an override to a CMS text field bound to a per-item slug. On mount, match the URL param against `props.text`, walk up to the nearest `[tabindex]` wrapper, find `onTap`, invoke it, then clean the URL:

```typescript
import type { ComponentType } from "react"
import { useEffect, useRef } from "react"

/**
 * @framerDisableUnlink
 */
export function withMemberDeepLink(Component): ComponentType {
    return (props) => {
        const ref = useRef<HTMLElement | null>(null)
        const done = useRef(false)

        useEffect(() => {
            if (done.current || typeof window === "undefined") return
            const target = new URLSearchParams(window.location.search).get("member")
            if (!target || target !== (props.text || "").trim()) return

            const t = setTimeout(() => {
                const wrapper = ref.current?.closest("[tabindex]") as HTMLElement | null
                if (!wrapper) return
                const onTap = findFiberHandler(wrapper, "onTap")
                if (typeof onTap !== "function") return

                onTap({} as any, {} as any)

                const url = new URL(window.location.href)
                url.searchParams.delete("member")
                window.history.replaceState({}, "", url.toString())
                done.current = true
            }, 500)

            return () => clearTimeout(t)
        }, [props.text])

        return (
            <span ref={ref} style={{ display: "contents" }}>
                <Component {...props} />
            </span>
        )
    }
}
```

The 500ms timeout here is waiting for Framer's overlay wrapper to mount, not for CMS content — different concern from the CMS Content Timing section above.

### Debugging React internals

Inspect props on an element:
```js
const el = document.getElementById("YOUR_ID")
const key = Object.keys(el).find(k => k.startsWith("__reactProps"))
console.log(el[key])
```

Find all handler functions up the fiber tree (useful when you don't yet know what Framer attached or at which depth):
```js
const el = document.getElementById("YOUR_ID")
const fiberKey = Object.keys(el).find(k => k.startsWith("__reactFiber"))
let fiber = el[fiberKey]
for (let depth = 0; fiber && depth < 15; depth++, fiber = fiber.return) {
    const mp = fiber.memoizedProps
    if (!mp) continue
    const fns = Object.keys(mp).filter(k => typeof mp[k] === "function")
    if (fns.length) console.log(`Depth ${depth}:`, fns)
}
```

### Maintenance risks

- `__reactFiber$...` / `__reactProps$...` are React internals. The `$<suffix>` changes between React builds; the prefixes have been stable for years but are not officially supported API.
- Framer Motion handler names (`onTap` etc.) could change with future Framer updates.
- Fiber depth to reach the handler is project-dependent — 15 is a safe ceiling but may need to grow if Framer restructures wrappers.

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

**Problem:** Components with `position: absolute` inherit their parent's stacking context. Even with `z-index: 9999`, they can't appear above elements outside the parent.

**Solution:** Use React Portal to render at `document.body` level:

```typescript
import { createPortal } from "react-dom"

export default function ComponentWithOverlay(props) {
    const [showOverlay, setShowOverlay] = useState(false)

    return (
        <div style={{ position: "relative" }}>
            {/* Main component content */}

            {/* Overlay rendered outside parent hierarchy */}
            {showOverlay && createPortal(
                <div style={{
                    position: "fixed",  // Fixed to viewport
                    inset: 0,
                    zIndex: 9999,
                    background: "rgba(0, 0, 0, 0.8)",
                }}>
                    {/* Overlay content */}
                </div>,
                document.body
            )}
        </div>
    )
}
```

**Key differences:**
- `position: "fixed"` positions relative to viewport, not parent
- Portal breaks out of component's DOM hierarchy and stacking context
- Works for modals, tooltips, popovers, loading overlays

**Canvas vs Published:**
Portals work in both canvas editor and published site. No RenderTarget check needed.

## Loading States with Scroll Lock

**Pattern:** Show loading overlay and prevent page scroll until content is ready.

```typescript
const [isLoading, setIsLoading] = useState(true)
const [fadeOut, setFadeOut] = useState(false)

// Prevent scroll while loading (published site only)
useEffect(() => {
    const isPublished = RenderTarget.current() !== "CANVAS"
    if (!isPublished || !isLoading) return

    const originalOverflow = document.body.style.overflow
    document.body.style.overflow = "hidden"

    return () => {
        document.body.style.overflow = originalOverflow
    }
}, [isLoading])

// Two-phase hide: fade-out → remove from DOM
const hideLoader = () => {
    setFadeOut(true)
    setTimeout(() => setIsLoading(false), 300) // Match CSS transition
}
```

**Scroll to top on load** (fixes variant sequence issues):
```typescript
useEffect(() => {
    const isPublished = RenderTarget.current() !== "CANVAS"
    if (isPublished) {
        window.scrollTo(0, 0)
    }
}, [])
```

## Easing Curves with Lerp Animations

**Problem:** Exponential lerp (`value += diff * speed`) naturally gives ease-out. Need to track initial distance to implement other curves.

**Solution:** Track `initialDiff` when animation starts:

```typescript
const animated = useRef({
    property: {
        current: 0,
        target: 0,
        initialDiff: 0,  // Track for easing calculations
    }
})

// When target changes, store initial distance
const updateTarget = (newTarget) => {
    const entry = animated.current.property
    entry.initialDiff = Math.abs(newTarget - entry.current)
    entry.target = newTarget
}

// Apply easing in animation loop
const applyEasing = (easingCurve) => {
    const v = animated.current.property
    const diff = v.target - v.current
    let speed = 0.05  // Base speed

    if (easingCurve !== "ease-out") {
        // Calculate progress: 0 at start, 1 near target
        const diffMagnitude = Math.abs(diff)
        const progress = v.initialDiff > 0
            ? Math.max(0, Math.min(1, 1 - (diffMagnitude / v.initialDiff)))
            : 1

        if (easingCurve === "ease-in") {
            // Start slow, end fast (cubic)
            speed *= (0.05 + Math.pow(progress, 3) * 10)
        } else if (easingCurve === "ease-in-out") {
            // Slow-fast-slow (smootherstep)
            const smoothed = progress * progress * progress *
                (progress * (progress * 6 - 15) + 10)
            speed *= (0.2 + smoothed * 3)
        }
    }
    // ease-out: use default exponential decay

    v.current += diff * speed
}
```

**Why aggressive curves?**
Exponential lerp naturally slows down approaching target. To create noticeable ease-in, need extreme multipliers (0.05x → 10x) to overcome the natural decay.

**Property control:**
```typescript
easingCurve: {
    type: ControlType.Enum,
    title: "Easing Curve",
    options: ["ease-out", "ease-in", "ease-in-out"],
    optionTitles: ["Ease Out", "Ease In", "Ease In/Out"],
    defaultValue: "ease-out",
}
```
