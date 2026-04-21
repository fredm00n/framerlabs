# Advanced Framer Patterns

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
