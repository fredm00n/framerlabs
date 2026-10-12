---
name: framer-code-components-overrides
description: Create Framer Code Components and Code Overrides. Use before writing or editing any `.tsx` for Framer, including deciding whether a feature needs code at all. Covers building custom React components for Framer, writing Code Overrides (HOCs) to modify canvas elements, implementing property controls, working with Framer Motion animations, handling WebGL/shaders in Framer, and debugging Framer-specific issues like hydration errors and font handling.
user-invocable: true
license: MIT
metadata:
  author: fredm00n
  version: 1.1.0
---

# Framer Code Development

Section tags: `[C]` applies to code components, `[O]` to code overrides, `[C/O]` to both.

## Updates and contributions

This skill is maintained in the public repo [fredm00n/framerlabs](https://github.com/fredm00n/framerlabs/tree/main/skills/framer-code-components-overrides), and the community keeps it current. Installed copies go stale, so the steps below run once per session.

### The project's line

The project's agent instructions (`CLAUDE.md`, `AGENTS.md` or equivalent) can hold one line for this skill:

```
framer-code-components-overrides: auto-update, contribute
```

`auto-update` lets the agent replace the installed copy when a newer version exists. `contribute` lets it offer to send what a session learns back to the repo as a pull request. Either word works alone. With no line, the agent never edits the installed files and never opens or suggests a pull request. It only says when a newer version is out.

### Check for a newer version first

Before relying on anything below:

1. Fetch `https://raw.githubusercontent.com/fredm00n/framerlabs/main/skills/framer-code-components-overrides/SKILL.md` and compare its `metadata.version` with the one in this file's frontmatter.
2. **Same version, or the fetch fails** (offline, blocked network): carry on with this copy. If the fetch failed, say once that you couldn't check for updates.
3. **Remote is newer:** use the remote SKILL.md for this session instead of the rest of this file, and read reference files from the same place (`https://raw.githubusercontent.com/fredm00n/framerlabs/main/skills/framer-code-components-overrides/references/<file>.md`) instead of the local copies. Then, by the project's line (NEW and OLD stand for the two version numbers):
   - **No `auto-update`:** leave the installed files alone and tell the user: "framer-code-components-overrides NEW is out and your copy is OLD, so I'm using the new version for this session. To update your copy, run `npx skills add fredm00n/framerlabs --skill framer-code-components-overrides`, or add `framer-code-components-overrides: auto-update` to your CLAUDE.md or AGENTS.md to stay current automatically. This skill is community-maintained: add `contribute` to that line (`framer-code-components-overrides: auto-update, contribute`) and I'll offer to send what we learn back as a pull request."
   - **`auto-update`:** download the whole `skills/framer-code-components-overrides/` folder from the repo (a shallow `git clone` into a temp dir is simplest), then replace the installed folder with it in one go, so files removed upstream go too. If any download fails, leave the installed copy untouched. In a git repo, the update is an ordinary change: commit it on its own, in whatever branch or PR the current work goes to, never straight to a protected branch. Tell the user: "Updated framer-code-components-overrides from OLD to NEW, in its own commit." Without `contribute` on the line, add: "This skill is community-maintained: add `contribute` to its line in CLAUDE.md or AGENTS.md and I'll offer to send what we learn back as a pull request."

Only take updates from that exact repo and path.

### Send what you learn upstream

Only when the project's line for this skill includes `contribute`.

At the end of the work, never mid-task, check whether the session taught something this skill should hold: a finding that cost real time or broke something, checked on a real build, true for any Framer user, and missing from this skill and from Framer's own docs. A fact about one project does not qualify. If nothing qualifies, say nothing.

If something does:

1. Rewrite each finding for strangers. Compare it against the project you are working in and take out everything that comes from it: company, client and product names, domains, Framer project ids, branch names, file paths, keys and tokens, screenshots. Keep the date the finding was made, as "(2026-10-12)".
2. Clone fredm00n/framerlabs, branch from `main`, and edit `skills/framer-code-components-overrides/`. Put each finding under the topic it belongs to, and edit an existing line rather than add a near-duplicate. Bump `metadata.version` (patch for a fix, minor for a new section) and add one plain-language line to the changelog in the root `README.md`, under today's date.
3. Check the open pull requests for one that already covers the same finding. If there is one, add to it instead.
4. Show the user the diff and ask before opening the pull request. A pull request on a public repo is public the moment it opens, even if it is never merged.
5. With the user's yes, push the branch (to a fork without write access to the repo), open the pull request against `main`, and give the user the link. Write the commit message, title and description for the people who use these skills: what the change adds and when it helps, not which project or session it came from.

**Maintainers:** bump `metadata.version` on every change to this skill, reference files included, or installed copies will never see it.

## Native First: Is Code Needed at All? `[C/O]`

Unless the user explicitly asks for a code component or a code override, build the feature natively
in Framer, with layers, variants, effects and CMS bindings. Write code only for what those cannot
express: WebGL, canvas, third-party SDKs, scroll maths, DOM and browser APIs.

This holds hardest for proven UI patterns. An FAQ accordion, tabs, a testimonial slider, a pricing
toggle, a nav dropdown and a modal are all native in Framer, and the native build is the better one:
the site owner edits it without code, binds it to the CMS, restyles it with text and colour styles,
and it adapts to breakpoints and survives Framer upgrades. A code component for one of these is
over-engineering even when it works and looks right.

A disclosure or accordion is one component with two variants, Collapsed and Expanded, and nothing
else. No wrapper component around the set: repeat the item with a Collection List so the content
comes from the CMS. No props, arrays of objects, or font, weight and colour controls that re-create
what native layers already do.

When you cannot tell whether native can do it, ask the user before writing code. Asking costs one
message; a wrong guess costs a whole build.

## Pitfalls (quick lookup)

| Issue | Cause | Fix |
|-------|-------|-----|
| Variable text not found in override | Reading only `props.children` | Check `props.text` first — variable-bound text bypasses children |
| Font styles not applying | Accessing font props individually | Spread entire font object: `...props.font` |
| Hydration mismatch | Browser API in render | Use `isClient` state pattern |
| Hydration error #425 at `style`, and SSR CSS with `>` selectors or quoted attribute values never applies | React 18's server renderer escapes `<style>{css}</style>` text, so `>` ships as `&gt;` and `"` as `&quot;` | Render `<style dangerouslySetInnerHTML={{ __html: css }} />` |
| Dimensions stuck at 0 / SSR'd size persists | Initial state read from `window` already equals real value, `setState` no-ops | Init state to 0, flip in effect (see [Hydration Safety](#hydration-safety-co)) |
| Override finds no element after a later publish, with no error and no code change | It queried its own element by a React `useId()`; when hydration differs from the server render, the DOM keeps the server's id | Make the id in the browser after mount, or use a ref (see [Ids That Link Elements](#ids-that-link-elements-co)) |
| Child has no width, `offsetLeft` or `offsetParent`; `scroll-snap-align` on it is ignored | Framer wrapped each child of the frame in a `display: contents` element | Step to the first descendant with a box, measure with `getBoundingClientRect` (see [Measuring a Frame's Children](#measuring-a-frames-children-o)) |
| Color value crashes when user binds a token | `ControlType.Color` returns `{value: "#xxx"}` for tokens, string for static | Unwrap with `tok(v)` before use (see [Color Tokens](#color-tokens-controltypecolor-c)) |
| Override props undefined | Expecting property controls | Overrides don't support `addPropertyControls` |
| Override missing from Framer's picker dropdown | Export value is produced by *calling* a factory/HOC-generator — Framer's static scanner only recognizes literal override exports | Write each override as a literal `export function withX(Component)` or `export const withX = (Component) => …` (see [Overrides Must Be Literal Exports](#overrides-must-be-literal-exports-picker-detection-o)) |
| Scroll animation broken | `overflow: scroll` on container | Use IntersectionObserver on viewport (see [Scroll Detection](#scroll-detection-constraint-co)) |
| Scroll/animation silently stops working when target ID is set | `useScroll` target stored in `useState` captures null on first render | Use `useRef` for live-read targets (see [Live-Read Refs](#live-read-refs-useref-not-usestate-co)) |
| Named CMS layer not found by `findByFramerName` | Layer is a dynamic component instance — name not on `data-framer-name` | Wrap dynamic component in a plain frame carrying the expected name |
| Every named layer of a CMS item missing at once | The whole repeated Collection List item was made a component | Keep the item a plain frame (see [The Repeated Item Is Never a Component](#the-repeated-item-is-never-a-component-c)) |
| Override that rewrites `props.style` height does nothing on the published page | Framer emits sizes as generated CSS classes, and the page root carries an inline `min-height: 100vh` | Set the value inline, with `!important` against the root (see [Published Sizing](#published-sizing-an-override-cannot-see-o)) |
| URL changes but the page does not load after `router.navigate()` or a scripted link click | Framer softens programmatic cross-page navigation into a desynced route change | Gate in place instead (see [Cross-Page Navigation](#no-programmatic-cross-page-navigation-co)) |
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
| Override-wired button ignores Enter and Space | `onTap` is a pointer gesture; keyboard and `.click()` never fire it | Pass `onClick` to the instance (see [Component Instances](#component-instances-onclick-variant-scoped-css-o)) |
| DOM added by an override vanishes, or React throws `insertBefore` / `removeChild` errors | The override replaced or edited children React owns | Insert siblings next to React's nodes, never replace them (see [Inserting DOM Next to React's Nodes](#inserting-dom-next-to-reacts-nodes-o)) |
| Hidden form value arrives empty | Value written after Framer read the form, or the input was injected by code | Native hidden input, filled in a capture-phase `submit` listener (see [Framer Forms](#framer-forms-o)) |
| New CMS item only shows after a reload | CMS content is baked into each published deployment | Re-fetch the page and insert the new cards (see [Refreshing Published CMS Content](#refreshing-published-cms-content-in-place-o)) |
| "Private" CMS field readable by anyone | Framer publishes every field of every item in `.framercms` files | Keep private data out of the CMS (see [CMS Data Is Public](#cms-data-is-public-co)) |

## Contents

- [Native First](#native-first-is-code-needed-at-all-co) — whether the feature needs code at all
- [Foundations](#foundations) — components vs overrides, annotations, starter templates, multi-file components
- [Authoring](#authoring) — property controls, fonts, color tokens
- [Rendering & SSR](#rendering--ssr) — hydration, canvas detection, concurrent rendering, npm imports
- [CMS](#cms) — text timing in overrides, the repeated item, code-component CMS pattern, what the CMS makes public
- [Overrides — specific patterns](#overrides--specific-patterns) — variant control, fiber handlers, component instances, Framer forms
- [DOM & Performance](#dom--performance) — scroll detection, live-read refs, portals, published sizing, cross-page navigation, inserting next to React, live CMS refresh, common patterns
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

Most Framer code files can start with `// @ts-nocheck`. Framer's hook and prop types are loose, and some
patterns (conditional hooks, fiber walking) fail strict TypeScript.

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

Overrides take no property controls and cannot query the CMS; they read CMS content only once it is
rendered (see [CMS](#cms)). Their settings live in a constants block at the top of the file.

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

### Multi-File Components `[C]`

A component split across several source files needs one Framer code file per source file, and every
import between them carries its extension: `./constants.tsx`, not `./constants`.

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

Do that work in `useEffect`, never `useLayoutEffect`, or expect hydration mismatches and
first-paint flashes.

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

### Ids That Link Elements `[C/O]`

Never link elements through a React-generated id (`useId()`), for example an override that finds its own element with `document.querySelector('[data-x="' + useId() + '"]')`. When a published page hydrates with a tree that differs from the server render, the DOM keeps the server's id while the client code looks for its own. The lookup returns null, nothing throws, and controls that worked on one publish are dead on the next.

Use a ref when the target forwards refs. Otherwise make the id in the browser after mount and render the attribute only once it exists:

```typescript
function useClientId() {
    const [id, setId] = useState<string | null>(null)
    useEffect(() => {
        setId("x-" + Math.random().toString(36).slice(2, 10))
    }, [])
    return id
}

const id = useClientId()
<Component {...props} {...(id ? { "data-x": id } : {})} />
// every lookup returns early while id is null
```

After any publish or code change, retest every interactive element on the page, not only the one you edited: this kind of break shows up in code nobody touched.

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

### The Repeated Item Is Never a Component `[C]`

Never turn the whole repeated item of a Collection List into a component. A `findByFramerName`
walker reads the React element tree through `props.children`, and a component instance's inner tree
is not there until it renders. Wrapping the item in a component therefore hides every named layer at
once (on one build the background, the video and the moving titles all vanished, and the item
collapsed into one background layer). Components are fine inside each named frame; the named frames
stay plain frames at the item's top level.

### CMS in Code Components `[C]`

Code components consume a Framer CMS Collection List via a `ControlType.ComponentInstance` slot, then walk the resulting React element tree to extract per-item content. Core helpers:

- `useQueryData` + `getCollectionData` to materialise items
- `findByFramerName` to extract named layers from each item's template
- Plain frames must wrap dynamic components if their name needs to be discoverable
- `getPropertyControls(WrappedComponent)` to inherit controls when one CMS component wraps another

Full pattern, helper code, and traps: see [references/cms.md](references/cms.md).

### CMS Data Is Public `[C/O]`

A published page's HTML carries only the fields a page binds, but Framer also publishes every
collection as `.framercms` data files on `framerusercontent.com/cms/...`, holding **every field of
every item**. Load-more pagination and other runtime reads fetch them, and anyone can download
them. A field no page shows (an email, an internal note) is still public, and files from older
deployments stay online after the field is deleted.

- Never store private data in Framer CMS, whether it arrives from a form, a plugin or the server API.
- An override that gates CMS content hides it on screen only; the data is still in the files.
- Private or per-user data belongs in an external store read at runtime (Supabase with row-level
  security, for example), not in the CMS.

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

### Component Instances: `onClick`, `variant`, Scoped CSS `[O]`

An override on a canvas component instance can drive it from code:

```typescript
export function withRefreshButton(Component): ComponentType {
    return (props) => {
        const [loading, setLoading] = useState(false)
        return (
            <Component
                {...props}
                variant={loading ? "Loading" : "Default"} // variant *name* from the canvas
                onClick={() => run(setLoading)}         // onClick, not onTap
                aria-label="Refresh"
            />
        )
    }
}
```

- **Use `onClick`, not `onTap`.** Both reach the instance's root, but `onTap` is a pointer gesture:
  Enter and Space on the focused element, and a scripted `.click()`, never fire it.
- **`variant` takes the variant's name**, so a code-driven state (loading, active) reuses variants
  designed on the canvas.
- **A loading state tied to a looping animation should end on a whole loop.** If an icon spins at
  one turn per second, keep the Loading variant for `max(1, ceil(elapsed / 1000))` seconds; a
  request that finishes in 300 ms otherwise snaps the icon back mid-turn.
- **Built-in components without style controls** (the pagination "Load More" button, for one) can
  be restyled from any override by injecting a `<style>` scoped to
  `[data-framer-name="<instance name>"]`. Name the instance on the canvas first; layer names survive
  publishing, generated `.framer-*` classes do not.

### Framer Forms `[O]`

How a native Framer form submits, from a form that posts to a webhook:

- The browser posts to `api.framer.com/forms/v1/forms/<id>/submit` with the global `fetch`, adding a
  proof-of-work header. The form also renders its own hidden honeypot inputs.
- **The Success variant shows as soon as Framer stores the submission**, before any webhook or
  email destination runs. The page never learns whether the webhook accepted it; if visitors need
  that, the webhook has to expose its verdict (a small status endpoint the override polls).
- **Detect a successful submission** by wrapping `window.fetch` once and matching a `POST` to
  `api.framer.com/forms/` with `res.ok`. The request body is the submitted data, so the override
  can read a field from it.
- **Add a computed field** (a token, a visitor IP) with a real hidden input on the canvas
  (`formInputHidden` plus a `formInputName`), not one injected by code, and fill it from a
  capture-phase `submit` listener on the form, which runs before Framer's handler reads the form.

```typescript
useEffect(() => {
    const form = document.querySelector("form")
    const input = form?.querySelector('input[name="ip"]') as HTMLInputElement | null
    if (!form || !input) return
    const fill = () => {
        const set = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, "value")?.set
        set?.call(input, valueRef.current) // native setter so React sees the change
    }
    form.addEventListener("submit", fill, true) // capture phase
    return () => form.removeEventListener("submit", fill, true)
}, [])
```

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

### Measuring a Frame's Children `[O]`

Framer can wrap each child of a frame (component instances in a stack, for one) in a `display: contents` element. That wrapper has no box: `children[i]` reports no width, no `offsetLeft` and no `offsetParent`, and `scroll-snap-align` set on it is ignored. Step down to the first descendant that has a box, and measure with `getBoundingClientRect()` relative to the scroller rather than `offsetLeft`.

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

### Published Sizing an Override Cannot See `[O]`

Two kinds of size on a published Framer page never reach an override's props:

- A frame's height is emitted as a generated CSS class (`.framer-xxxxxxx { height: 100vh }`), not as
  an inline style, so there is no height in `props.style` to read or rewrite. Pass an explicit inline
  `height` instead; an inline style outranks the generated class.
- Every page root carries an inline `min-height: 100vh` on `<div data-framer-root>`. Beating it takes
  `!important`.

Both matter for full-height pages on phones: `100vh` is the large viewport, measured with the browser's
URL bar hidden, so a one-screen page scrolls a few pixels until both sources say `100svh`. The
simplest fix is site-wide custom CSS, `[data-framer-root] { min-height: 100svh !important }`, plus
the frame's own height; an override that sets both inline is the fallback (2026-08-19). Target published elements by `[data-framer-name="…"]`, which survives publishing, never by
the generated `.framer-*` classes, which change.

### No Programmatic Cross-Page Navigation `[C/O]`

A published Framer site is a single-page app, and it softens every scripted move to another page,
`router.navigate()`, a `.click()` on an anchor and `window.location.replace()` alike, into a desynced
route change: the URL updates and the page does not load. Gate or coordinate in place instead: write
a `localStorage` token, fire a custom window event, and let an override on the current page react. The one
exception is a redirect that runs on a full page load, in a head script or before the app has booted:
there `location.replace()` is a real navigation and works.

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

The component also listens on `popstate` to re-read the param on Back/Forward and hydrate its own UI (clamp/validate — it's user-editable). Init state from props, not the URL, so SSR matches; do the first URL read in `useEffect`. This is **intra-page only** — it does not navigate, so it sidesteps the [cross-page navigation](#no-programmatic-cross-page-navigation-co) trap. It's the same idea as a custom-event state bus, but over the browser's native `popstate` channel, so Framer's own URL filters pick it up for free.

### Styling Native Range Inputs `[C]`

Framer offers no styling for native `<input type="range">`, and browsers paint their own blue track/thumb. To fully restyle:

- `appearance: none` (+ `-webkit-appearance: none`) and `background: transparent` on the input.
- Neutralise the track in **both** engines: `::-webkit-slider-runnable-track` **and** `::-moz-range-track`.
- Re-draw the thumb in **both** `::-webkit-slider-thumb` and `::-moz-range-thumb` — `appearance:none` removes the native thumb too, so recreate it per engine.
- Draw the visible track/fill as absolutely-positioned `<div>`s underneath; the transparent input sits on top.
- Scope selectors with a className (`.rs-thumb`) inside the `<style>` tag — bare `input[type=range]` leaks to every range on the page.

**Dual-thumb (two stacked inputs over one track):** set `pointer-events: none` on each *input* (so overlapping inputs don't block each other or the track) and `pointer-events: auto` on the *thumb pseudo-elements* only — just the thumbs stay grabbable. Enforce non-crossing in the `onChange` handlers (clamp each thumb against the other ± one step), not in the DOM.

### Inserting DOM Next to React's Nodes `[O]`

When an override must add elements inside a container Framer renders (a Collection List, say),
**add siblings and leave React's own nodes alone**. React keeps references to the children it
rendered and moves, updates and removes them later (pagination appends pages, a tab switch
unmounts a list). Replacing or re-parenting those children breaks the next reconciliation;
foreign nodes placed *between* them survive, because React only touches its own.

- Insert with `reactNode.before(mine)` / `after(mine)` and mark yours (`data-mine="<key>"`), so you
  can find, skip and re-place them.
- Watch the container with a `MutationObserver` to re-place your nodes when React remounts or
  appends. Make placement **idempotent**: if a container already shows exactly your nodes, do
  nothing. Otherwise every unrelated re-render removes and re-inserts them, which restarts their
  animations.
- Call `observer.takeRecords()` right after your own edits, so they do not trigger another pass.
- Never hide React's node with an inline style it also manages; React may rewrite it. Use an
  attribute plus an injected stylesheet rule if you must hide something.

### Refreshing Published CMS Content In Place `[O]`

A Framer site is a static deployment: a new CMS item exists only in the next published version of
the page, and the running page never learns about it. To show it without a reload:

1. Fetch the current page again (`fetch(location.pathname + "?x=" + Date.now(), { cache: "no-store" })`).
   Framer serves pages with `max-age=0`, so this returns the newest deployment.
2. Parse it with `DOMParser` and read the list's rendered items by `[data-framer-name="..."]`. The
   server-rendered HTML contains the first page of a Collection List (with the sort it was given on
   the canvas, whatever the URL's query says), and each date field carries `<time datetime>`, which
   is enough to key and re-sort items.
3. Compare against what the page loaded with, and insert only the new items (see above).

After a form submission, publishing can take 5 to 15 seconds: poll every couple of seconds with a
time limit, and give the visitor a visible state while waiting.

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
