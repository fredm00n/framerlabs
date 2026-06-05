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
| Override missing from Framer's picker dropdown | Export value is produced by *calling* a factory/HOC-generator — Framer's static scanner only recognizes literal override exports | Write each override as a literal `export function withX(Component)` or `export const withX = (Component) => …` (see [Overrides Must Be Literal Exports](#overrides-must-be-literal-exports-picker-detection-o)) |
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
| URL-bound filters don't react to a programmatic URL write | `replaceState`/`pushState` don't fire `popstate` | Dispatch it manually after writing (see [Writing State into the URL](#writing-state-into-the-url-c)) |
| Slider drag floods browser history / traps Back | `pushState` on every `onChange` | Use `replaceState` for high-frequency writes |
| Range input shows blue native fill / unstyleable thumb | Native `<input type="range">` chrome | `appearance:none` + neutralise track + custom thumb per engine (see [Styling Native Range Inputs](#styling-native-range-inputs-c)) |

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

### Overrides Must Be Literal Exports (picker detection) `[O]`

Framer's override picker **statically scans** the file and only lists exports it can syntactically recognize as an override — a literal `function` declaration or an arrow assigned directly to the export. An export whose value is the **return of a function call** is invisible to the scanner: it won't appear in the Override dropdown (even though it would work if referenced by name).

This bites hardest when you try to DRY up several near-identical overrides with a factory:

```typescript
// ❌ BROKEN — factory hides the overrides from the picker.
// Only literal functions in the file get listed; these three never appear.
const setVariant = (name) => (Component) => (props) => {
    const [, setStore] = useStore()
    return <Component {...props} onClick={() => setStore({ variant: name })} />
}
export const withSetVariant1 = setVariant("Variant 1") // not listed
export const withSetVariant2 = setVariant("Variant 2") // not listed
export const withSetVariant3 = setVariant("Variant 3") // not listed

// ✅ CORRECT — each override is a literal function; all are listed.
export function withSetVariant1(Component): ComponentType {
    return (props) => {
        const [, setStore] = useStore()
        return <Component {...props} onClick={() => setStore({ variant: "Variant 1" })} />
    }
}
```

Both literal forms are recognized: `export function withX(Component) {…}` **or** `export const withX = (Component) => (props) => …`. The rule: **never produce an override by calling a helper** — repeat the literal per override, even if it's more verbose. (Full worked example — 3 setters + 1 reader sharing a store — in [references/patterns.md](references/patterns.md) → *Shared State Between Overrides*.)

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

### Writing State into the URL `[C]`

A component can drive **Framer's native URL-bound filtering** by writing a query param. Framer's filters do the reading/filtering — your component only writes the param and keeps it in sync. Two non-obvious rules for the write side:

- **Use `replaceState`, not `pushState`, for high-frequency writes** (slider drags, live filters). `pushState` adds a history entry per event and traps the Back button.
- **`history.replaceState` / `pushState` do NOT fire `popstate`.** Only browser-driven navigation (Back/Forward, typed URL) fires it. So after writing, dispatch it yourself or the rest of the page never sees the change:

```typescript
const url = new URL(window.location.href)
url.searchParams.set(paramName, String(value))
window.history.replaceState({}, "", url.toString())
// replaceState is silent — fire popstate so URL-bound consumers re-sync
window.dispatchEvent(new PopStateEvent("popstate"))
```

The component also listens on `popstate` to re-read the param on Back/Forward and hydrate its own UI (clamp/validate — it's user-editable). Init state from props, not the URL, so SSR matches; do the first URL read in `useEffect`. This is **intra-page only** — it does not navigate, so it sidesteps the "cross-page nav is unreliable in published Framer" trap. It's the same idea as a custom-event state bus, but over the browser's native `popstate` channel, so Framer's own URL filters pick it up for free.

### Styling Native Range Inputs `[C]`

Framer offers no styling for native `<input type="range">`, and browsers paint their own blue track/thumb. To fully restyle:

- `appearance: none` (+ `-webkit-appearance: none`) and `background: transparent` on the input.
- Neutralise the track in **both** engines: `::-webkit-slider-runnable-track` **and** `::-moz-range-track`.
- Re-draw the thumb in **both** `::-webkit-slider-thumb` and `::-moz-range-thumb` — `appearance:none` removes the native thumb too, so recreate it per engine.
- Draw the visible track/fill as absolutely-positioned `<div>`s underneath; the transparent input sits on top.
- Scope selectors with a className (`.rs-thumb`) inside the `<style>` tag — bare `input[type=range]` leaks to every range on the page.

**Dual-thumb (two stacked inputs over one track):** set `pointer-events: none` on each *input* (so overlapping inputs don't block each other or the track) and `pointer-events: auto` on the *thumb pseudo-elements* only — just the thumbs stay grabbable. Enforce non-crossing in the `onChange` handlers (clamp each thumb against the other ± one step), not in the DOM.

### Common Patterns

See [references/patterns.md](references/patterns.md) for shared state, keyboard detection, show-once logic, scroll effects, magnetic hover, animation triggers, mobile optimization, Safari SVG fix, loading-state scroll lock, easing curves with lerp animations.

---

## Media

### HLS Video Streaming (`.m3u8`) `[C]`

Chrome/Firefox do **not** natively support HLS — a plain `<video src="...m3u8">` fails or plays the lowest rendition forever (Safari handles it natively). Fix: load HLS.js via dynamic import with silent fallback to native video.

See [references/hls-video.md](references/hls-video.md) for the full `loadHls` / `attachHls` implementation and cleanup notes.

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
