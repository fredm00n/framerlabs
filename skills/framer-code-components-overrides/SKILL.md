---
name: framer-code-components-overrides
description: Create Framer Code Components and Code Overrides. Use when building custom React components for Framer, writing Code Overrides (HOCs) to modify canvas elements, implementing property controls, working with Framer Motion animations, handling WebGL/shaders in Framer, or debugging Framer-specific issues like hydration errors and font handling.
---

# Framer Code Development

Section tags: `[C]` applies to code components, `[O]` to code overrides, `[C/O]` to both.

## Pitfalls (quick lookup)

| Issue | Cause | Fix |
|-------|-------|-----|
| Variable text not found in override | Reading only `props.children` | Check `props.text` first — variable-bound text bypasses children |
| Font styles not applying | Accessing font props individually | Spread entire font object: `...props.font` |
| Hydration mismatch | Browser API in render | Use `isClient` state pattern |
| Dimensions stuck at 0 / SSR'd size persists | Initial state read from `window` already equals real value, `setState` no-ops | Init state to 0, flip in effect (see [Hydration Safety](#hydration-safety)) |
| Color value crashes when user binds a token | `ControlType.Color` returns `{value: "#xxx"}` for tokens, string for static | Unwrap with `tok(v)` before use (see [Color Tokens](#color-tokens-controltypecolor-c)) |
| Override props undefined | Expecting property controls | Overrides don't support `addPropertyControls` |
| Scroll animation broken | `overflow: scroll` on container | Use IntersectionObserver on viewport (see [Scroll Detection](#scroll-detection-constraint-co)) |
| Scroll/animation silently stops working when target ID is set | `useScroll` target stored in `useState` captures null on first render | Use `useRef` for live-read targets (see [Live-Read Refs](#live-read-refs-useref-not-usestate-co)) |
| Named CMS layer not found by `findByFramerName` | Layer is a dynamic component instance — name not on `data-framer-name` | Wrap dynamic component in a plain frame carrying the expected name |
| HLS video permanently pixelated | `.m3u8` in Chrome without HLS.js | Use HLS.js dynamic import pattern (see [HLS Video Streaming](#hls-video-streaming-m3u8-c)) |
| Overlay stuck "half-pressed" / needs two clicks to close | Triggering Framer interactions with synthetic events (`dispatchEvent`) | Call the React handler directly via fiber traversal (see [references/fiber-handlers.md](references/fiber-handlers.md)) |
| Overlay stuck under content | Stacking context from parent | Use React Portal to render at `document.body` level |
| Shader attach error | Null shader from compilation failure | Check `createShader()` return before `attachShader()` |
| TypeScript `Timeout` errors | Using `NodeJS.Timeout` type | Use `number` instead — browser environment |
| Component display name | Need custom name in Framer UI | `Component.displayName = "Name"` |
| Easing feels same for all curves | Not tracking initial distance | Track `initialDiff` when target changes (see [references/patterns.md](references/patterns.md)) |

## Contents

- [Foundations](#foundations) — components vs overrides, annotations, starter templates
- [Authoring](#authoring) — property controls, fonts, color tokens
- [Rendering & SSR](#rendering--ssr) — hydration, canvas detection, concurrent rendering, npm imports
- [CMS](#cms) — text timing in overrides, code-component CMS pattern
- [Overrides — specific patterns](#overrides--specific-patterns) — variant control, fiber handlers
- [DOM & Performance](#dom--performance) — scroll detection, live-read refs, portals, common patterns
- [Media](#media) — HLS video, WebGL
- [Debug](#debug) — gated logging

---

## Foundations

### Code Components vs Overrides

**Code Components `[C]`**: Custom React components added to canvas. Support `addPropertyControls`.

**Code Overrides `[O]`**: Higher-order components wrapping existing canvas elements. Do NOT support `addPropertyControls`.

### Required Annotations `[C/O]`

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

### Code Override Pattern `[O]`

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

### Code Component Pattern `[C]`

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

---

## Authoring

### Property Controls Reference `[C]`

See [references/property-controls.md](references/property-controls.md) for complete control types and patterns.

### Font Handling `[C/O]`

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

### Color Tokens (`ControlType.Color`) `[C]`

A `ControlType.Color` value arrives as a **plain string** when the user picks a static color, but as a **`{ value: "#xxx" }` object** when bound to a Framer color token. Components that read the value directly break the moment the user binds a token.

Always unwrap:

```typescript
const tok = (v: any) =>
    v && typeof v === "object" && "value" in v ? v.value : v

const bg = tok(props.background) // string in both cases
```

Use `tok()` wherever a color prop is consumed for parsing, CSS strings, or canvas styles. Same wrapper shape may appear on other token-bindable controls (sizes, shadows) — check before assuming.

---

## Rendering & SSR

### Hydration Safety `[C/O]`

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

**Initial state must match SSR, then flip in an effect:**

```typescript
// ❌ BROKEN — looks "smart" but isn't
const [vw, setVw] = useState(
    typeof window !== "undefined" ? window.innerWidth : 0
)

useEffect(() => {
    setVw(window.innerWidth) // no-op: state already equals window.innerWidth
}, [])
```

The initial state already equals the real value on the client, so `setState` becomes a no-op and the SSR'd dimensions (always 0) persist forever in the rendered DOM.

```typescript
// ✅ CORRECT — start at SSR-safe value, force a re-render via effect
const [vw, setVw] = useState(0)
const [vh, setVh] = useState(0)

useEffect(() => {
    const onResize = () => {
        setVw(window.innerWidth)
        setVh(window.innerHeight)
    }
    onResize()
    window.addEventListener("resize", onResize)
    return () => window.removeEventListener("resize", onResize)
}, [])

// Hide the first paint (SSR'd at 0) until the effect has populated real values
<div style={{ opacity: vw > 0 ? 1 : 0 }} />
```

Pairing the `> 0` opacity gate with the flip-in-effect hides the first paint, otherwise you see a flash from 0/default → real size on refresh.

### Canvas vs Preview Detection `[C/O]`

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

### Concurrent Rendering: Wrap State Updates in `startTransition` `[C/O]`

Framer runs on React's concurrent renderer. Multi-setter updates in event handlers (steppers, async chains, form fields) can stutter under load. Wrap non-urgent updates:

```typescript
import { startTransition } from "react"

const handleClick = () => {
    startTransition(() => {
        setQty((q) => ({ ...q, [id]: q[id] + 1 }))
        setError(null)
    })
}

const onSubmit = async () => {
    startTransition(() => {
        setLoading(true)
        setError(null)
    })
    try {
        const res = await fetch(...)
        // ...
    } catch (e) {
        startTransition(() => {
            setError(e.message)
            setLoading(false)
        })
    }
}
```

Don't wrap the user-input setter itself (`onChange` → `setValue`) — that one needs to feel immediate.

### NPM Package Imports `[C/O]`

Standard import (preferred):
```typescript
import { Component } from "package-name"
```

Force specific version via CDN when Framer cache is stuck:
```typescript
import { Component } from "https://esm.sh/package-name@1.2.3?external=react,react-dom"
```

Always include `?external=react,react-dom` for React components.

---

## CMS

### Content Timing in Overrides `[O]`

CMS text arrives in `props.text` asynchronously (~50–200ms after hydration). For variable-bound text from component props, it's synchronous on first render — no delay needed.

The reliable pattern for both: use `resolvePlainText(props)` (see [Text in Overrides](#text-in-overrides-o) below) and gate on the value being non-empty:

```typescript
const plainText = resolvePlainText(props)
// plainText is "" until content arrives → gate your animation on plainText.length > 0
```

Avoid 100ms arbitrary delays — they cause race conditions when the element is already in the viewport on load.

### Text in Overrides `[O]`

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

### CMS in Code Components `[C]`

Code components consume a Framer CMS Collection List via a `ControlType.ComponentInstance` slot, then walk the resulting React element tree to extract per-item content. Core helpers:

- `useQueryData` + `getCollectionData` to materialise items
- `findByFramerName` to extract named layers from each item's template
- Plain frames must wrap dynamic components if their name needs to be discoverable
- `getPropertyControls(WrappedComponent)` to inherit controls when one CMS component wraps another

Full pattern, helper code, and traps: see [references/cms.md](references/cms.md).

---

## Overrides — Specific Patterns

### Variant Control `[O]`

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

### Triggering Framer-Attached Handlers `[O]`

Synthetic DOM events (`dispatchEvent`) don't reliably trigger Framer Motion handlers — they leave the element in a half-pressed state. Instead, walk the React fiber tree from the DOM node up to the handler-bearing fiber and call it directly:

```typescript
const onTap = findFiberHandler(wrapper, "onTap")
onTap?.({} as any, {} as any)
```

Full helper, debugging snippets, deep-link use case, and maintenance risks: see [references/fiber-handlers.md](references/fiber-handlers.md).

---

## DOM & Performance

### Scroll Detection Constraint `[C/O]`

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

### Live-Read Refs: `useRef`, Not `useState` `[C/O]`

Hooks that read `.current` live on every event (Framer Motion's `useScroll`, IntersectionObserver targets, RAF loops) **must** receive a `useRef`. Storing the target in `useState` captures `null` on the first hook call and never re-subscribes once state flips.

The trap is that this often *appears* to work — `useScroll` silently falls back to window scroll, so the page seems to animate at first glance until you pin the target with `id="..."` and everything freezes.

```typescript
// ❌ BROKEN — useScroll captures target: null on first render
const [scrollEl, setScrollEl] = useState<HTMLElement | null>(null)
useEffect(() => { setScrollEl(document.getElementById("section")) }, [])
const { scrollYProgress } = useScroll({ target: scrollEl })

// ✅ CORRECT — useScroll reads ref.current live on each event
const scrollRef = useRef<HTMLElement | null>(null)
useEffect(() => { scrollRef.current = document.getElementById("section") }, [])
const { scrollYProgress } = useScroll({ target: scrollRef })
```

Applies to any API that reads through a ref handle per event — not just `useScroll`.

### Z-Index Stacking Context & React Portals `[C/O]`

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

**Canvas vs Published:** Portals work in both canvas editor and published site. No RenderTarget check needed.

### Common Patterns

See [references/patterns.md](references/patterns.md) for shared state, keyboard detection, show-once logic, scroll effects, magnetic hover, animation triggers, mobile optimization, Safari SVG fix, loading-state scroll lock, easing curves with lerp animations.

---

## Media

### HLS Video Streaming (`.m3u8`) `[C]`

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

### WebGL in Framer `[C]`

See [references/webgl-shaders.md](references/webgl-shaders.md) for shader implementation patterns including transparency handling.

---

## Debug

### Debug Logging `[C/O]`

Gate every `console.log` in a component behind a module-level boolean so production builds don't leak data or noise. Never sprinkle `console.log` directly — toggling them off later means hunting them down.

```typescript
const debugMode = false // flip to true when debugging this component

if (debugMode) console.log("Honeypot active, fields:", values)
```

Especially important for components that handle user input (form values), auth state, or third-party tokens — these will end up in production console logs otherwise.
