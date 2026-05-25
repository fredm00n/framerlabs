# Triggering Framer-Attached Handlers from Code

When you need to programmatically fire a Framer/Framer Motion interaction (open an overlay, trigger a tap, etc.), **synthetic DOM events do not work reliably**. Framer Motion attaches handlers like `onTap` as React handlers, not native DOM listeners — synthetic events take a different code path and leave Framer Motion's internal state desynchronised. Symptoms include stuck press/focus state, two-click-to-close bugs, and other "half-pressed" weirdness that persists for the rest of the session on that element.

## Reach into the React fiber tree and call the handler directly

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

## Use case: URL deep link to an overlay

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

The 500ms timeout here is waiting for Framer's overlay wrapper to mount, not for CMS content — different concern from the CMS Content Timing pattern.

## Debugging React internals

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

## Maintenance risks

- `__reactFiber$...` / `__reactProps$...` are React internals. The `$<suffix>` changes between React builds; the prefixes have been stable for years but are not officially supported API.
- Framer Motion handler names (`onTap` etc.) could change with future Framer updates.
- Fiber depth to reach the handler is project-dependent — 15 is a safe ceiling but may need to grow if Framer restructures wrappers.
