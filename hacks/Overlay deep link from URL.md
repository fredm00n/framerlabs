# Overlay Deep Link from URL

Open a Framer overlay automatically when the page loads with a matching URL parameter (e.g. `/team?member=fredo`). A free alternative to Framer's paid Trigger/Convert deep-link feature.

Apply this override to a Text layer inside a CMS item, bound to a per-item slug field. On mount, the override matches `?member=` against the layer's text, walks up to Framer's tappable overlay wrapper, finds the `onTap` handler that Framer Motion attached, and invokes it directly. Then cleans the URL via `history.replaceState`.

```tsx
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
```

Why it works:
- Framer Motion attaches `onTap` as a React handler, not a native DOM listener. Dispatching synthetic `pointerdown` / `click` events opens the overlay but leaves Framer Motion in a half-pressed state — the overlay then needs two clicks to close for the rest of the session.
- Walking the React fiber tree (`fiber.return`) reaches the Framer Motion component a few levels above the DOM node and calls the exact handler a real tap would call. Same code path = no stuck state.
- `el.closest("[tabindex]")` reliably finds the overlay's tappable wrapper from a child override; Framer renders overlay triggers with `tabindex="0"`.

Requirements:
- The text the override is applied to must be bound to a per-item slug field (so `props.text` holds the slug).
- The overlay must be configured natively on the parent wrapper in Framer.
- URL format: `?member=<slug-value>` exact-match against the text.
- Rename `"member"` to whatever URL param your project needs — the rest is project-agnostic.

Maintenance notes:
- Relies on React internals (`__reactFiber$...`). The `$<suffix>` changes between React builds; the prefix has been stable for years but is not officially supported.
- If Framer renames `onTap` or restructures its overlay wrappers, the override breaks until the handler name string (and fiber depth ceiling) is updated.
