# Common Framer Patterns

## Shared State Between Overrides

Use Framer's store for state shared across multiple overrides. **One file can hold every override** (the setters *and* the reader) — they share state because they all call the same module-level `useStore`. A separate store file is only needed when overrides in *different* code files must share the same store (then `export` the hook from its own file and import it in both).

Worked example — three CTAs each set a component's active variant; a fourth override applies it:

```typescript
import type { ComponentType } from "react"
import { createStore } from "https://framer.com/m/framer/store.js@^1.0.0"

const useStore = createStore({ variant: "Variant 1" })

// Setters — apply one to each CTA. Keep them as LITERAL functions; a factory
// would hide them from Framer's override picker (see warning below).
export function withSetVariant1(Component): ComponentType {
    return (props) => {
        const [, setStore] = useStore()
        return <Component {...props} onClick={(e) => { props.onClick?.(e); setStore({ variant: "Variant 1" }) }} />
    }
}
export function withSetVariant2(Component): ComponentType {
    return (props) => {
        const [, setStore] = useStore()
        return <Component {...props} onClick={(e) => { props.onClick?.(e); setStore({ variant: "Variant 2" }) }} />
    }
}
export function withSetVariant3(Component): ComponentType {
    return (props) => {
        const [, setStore] = useStore()
        return <Component {...props} onClick={(e) => { props.onClick?.(e); setStore({ variant: "Variant 3" }) }} />
    }
}

// Reader — apply to the component that owns the variants.
export function withVariant(Component): ComponentType {
    return (props) => {
        const [store] = useStore()
        return <Component {...props} variant={store.variant} />
    }
}
```

**Do not DRY the setters with a factory.** A `const withSetVariant1 = makeSetter("Variant 1")` will NOT surface in the override picker — Framer's scanner only recognizes literal override exports. Repeat the literal function per CTA. See SKILL.md → *Overrides Must Be Literal Exports (picker detection)*.

Notes:
- **Variant prop** is the variant's *name* as a string (`"Variant 1"`) and must match the panel exactly (case + spacing).
- Compose `props.onClick?.(e)` so you don't clobber any click behaviour already on the CTA.
- Reading variant names back *from* props is unreliable (may be hashed) — drive them from the store/state instead.

## Keyboard Sequence Detection

```typescript
const targetSequence = ["d", "r", "u", "t", "o"]

export function withCodeword(Component): ComponentType {
    return (props) => {
        const [keySequence, setKeySequence] = useState([])
        const [triggered, setTriggered] = useState(false)

        useEffect(() => {
            const handler = (event) => {
                const newSequence = [...keySequence, event.key]
                if (newSequence.length > targetSequence.length) {
                    newSequence.shift()
                }

                const isMatch = targetSequence.every(
                    (key, i) => key === newSequence[i]
                )

                if (isMatch) {
                    setTriggered(true)
                    setKeySequence([])
                } else {
                    setKeySequence(newSequence)
                }
            }

            window.addEventListener("keydown", handler)
            return () => window.removeEventListener("keydown", handler)
        }, [keySequence])

        return <Component {...props} variant={triggered ? "secret" : "default"} />
    }
}
```

## Show-Once Logic (localStorage)

```typescript
export function withShowOnce(Component): ComponentType {
    return (props) => {
        const [shouldShow, setShouldShow] = useState(() => {
            try {
                return !localStorage.getItem("hasShown")
            } catch {
                return true
            }
        })

        useEffect(() => {
            if (shouldShow) {
                localStorage.setItem("hasShown", "true")
            }
        }, [shouldShow])

        if (!shouldShow) return null

        return <Component {...props} />
    }
}
```

## Random Variant on Mount

```typescript
export function withRandomVariant(Component): ComponentType {
    return (props) => {
        const [variant, setVariant] = useState("variant-1")

        useEffect(() => {
            const num = Math.floor(Math.random() * 4) + 1
            setVariant(`variant-${num}`)
        }, [])

        return <Component {...props} variant={variant} />
    }
}
```

## Toggle on Keystroke

```typescript
export function withKeyToggle(Component): ComponentType {
    return (props) => {
        const [isActive, setIsActive] = useState(false)

        useEffect(() => {
            const handler = (event) => {
                if (event.key === "f" || event.key === "F") {
                    setIsActive(prev => !prev)
                }
            }

            window.addEventListener("keydown", handler)
            return () => window.removeEventListener("keydown", handler)
        }, [])

        return (
            <Component
                {...props}
                style={{ ...props.style, opacity: isActive ? 1 : 0 }}
            />
        )
    }
}
```

## Screenshot Exclude Class

```typescript
export function withScreenshotExclude(Component): ComponentType {
    return (props) => {
        const className = (props.className || "") + " screenshot-exclude"
        return <Component {...props} className={className} />
    }
}
```

## Scroll Velocity Detection

```typescript
export function withScrollBlur(Component): ComponentType {
    return (props) => {
        const [blur, setBlur] = useState(0)
        const lastScrollY = useRef(0)
        const lastTime = useRef(Date.now())

        useEffect(() => {
            let animationId

            const handleScroll = () => {
                const now = Date.now()
                const deltaTime = now - lastTime.current
                const deltaScroll = Math.abs(window.scrollY - lastScrollY.current)

                if (deltaTime > 0) {
                    const velocity = deltaScroll / deltaTime
                    setBlur(Math.min(velocity * 10, 20))
                }

                lastScrollY.current = window.scrollY
                lastTime.current = now
            }

            const decay = () => {
                setBlur(prev => {
                    const next = prev * 0.9
                    return next < 0.1 ? 0 : next
                })
                animationId = requestAnimationFrame(decay)
            }

            window.addEventListener("scroll", handleScroll, { passive: true })
            animationId = requestAnimationFrame(decay)

            return () => {
                window.removeEventListener("scroll", handleScroll)
                cancelAnimationFrame(animationId)
            }
        }, [])

        return (
            <Component
                {...props}
                style={{
                    ...props.style,
                    backdropFilter: `blur(${blur}px)`,
                    WebkitBackdropFilter: `blur(${blur}px)`,
                }}
            />
        )
    }
}
```

## Magnetic Hover Effect

```typescript
import { motion, useSpring } from "framer-motion"

const SPRING_CONFIG = { damping: 100, stiffness: 1000 }

export function withMagnet(Component): ComponentType {
    return (props) => {
        const springX = useSpring(0, SPRING_CONFIG)
        const springY = useSpring(0, SPRING_CONFIG)

        const handleMove = (e) => {
            const rect = e.currentTarget.getBoundingClientRect()
            const centerX = rect.left + rect.width / 2
            const centerY = rect.top + rect.height / 2

            springX.set((e.clientX - centerX) * 0.3)
            springY.set((e.clientY - centerY) * 0.3)
        }

        const handleLeave = () => {
            springX.set(0)
            springY.set(0)
        }

        return (
            <motion.div
                onPointerMove={handleMove}
                onPointerLeave={handleLeave}
                style={{
                    x: springX,
                    y: springY,
                    willChange: "transform",
                    transform: "translateZ(0)",
                    backfaceVisibility: "hidden",
                }}
            >
                <Component {...props} />
            </motion.div>
        )
    }
}
```

## Multi-Column Text Layout

```typescript
export function withColumns(Component): ComponentType {
    return (props) => {
        return (
            <div style={{
                columns: "300px auto",
                columnGap: "50px",
                maxWidth: "100%",
            }}>
                <Component {...props} style={{ ...props.style, margin: 0 }} />
            </div>
        )
    }
}
```

## Scroll Frame Sequence

```typescript
export default function ScrollSequence(props) {
    const { images, scrollSensitivity = 50 } = props
    const [frame, setFrame] = useState(0)
    const accumulator = useRef(0)
    const lastY = useRef(0)

    useEffect(() => {
        const handler = () => {
            const delta = window.scrollY - lastY.current
            lastY.current = window.scrollY

            accumulator.current += Math.abs(delta)

            if (accumulator.current >= scrollSensitivity) {
                const change = Math.floor(accumulator.current / scrollSensitivity)
                accumulator.current %= scrollSensitivity

                setFrame(prev => {
                    const next = delta > 0 ? prev + change : prev - change
                    return ((next % images.length) + images.length) % images.length
                })
            }
        }

        window.addEventListener("scroll", handler, { passive: true })
        return () => window.removeEventListener("scroll", handler)
    }, [images.length, scrollSensitivity])

    return <img src={images[frame]} style={{ width: "100%", height: "100%" }} />
}
```

## Animation Trigger Modes

```typescript
// On Load - triggers immediately
useEffect(() => {
    if (props.triggerOnLoad && !hasTriggered) {
        setHasTriggered(true)
        setTriggerTime(Date.now())
    }
}, [])

// On Scroll - triggers when visible
useEffect(() => {
    if (props.triggerOnLoad) return

    const observer = new IntersectionObserver(
        ([entry]) => {
            if (entry.isIntersecting && !hasTriggered) {
                setHasTriggered(true)
                setTriggerTime(Date.now())
            }
        },
        { threshold: 0.1 }
    )

    if (ref.current) observer.observe(ref.current)
    return () => observer.disconnect()
}, [props.triggerOnLoad, hasTriggered])
```

## Auto-Sized Text Fix

When using auto-sized components (`@framerSupportedLayoutWidth auto`) with text, apply `minWidth: max-content` to prevent unexpected collapse or wrapping:

```typescript
<span style={{
    minWidth: "max-content",
    ...props.font,
}}>
    {props.label}
</span>
```

See [Auto-sized text fix](../../../hacks/Auto-sized%20text%20fix.md) for detailed explanation.

## Default Media Placeholder URLs

Working placeholder URLs for components with media controls:

```typescript
const placeholders = {
    image: "https://framerusercontent.com/images/GfGkADagM4KEibNcIiRUWlfrR0.jpg",
    video: "https://framerusercontent.com/assets/MLWPbW1dUQawJLhhun3dBwpgJak.mp4",
    audio: "https://framerusercontent.com/assets/8w3IUatLX9a5JVJ6XPCVuHi94.mp3",
}
```

Use these when setting default values via parameter destructuring (since `ControlType.ResponsiveImage` and `ControlType.File` don't support `defaultValue`).

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

## Mobile Optimization

For particle systems and heavy animations:
- Implement resize debouncing (500ms default)
- Add size change threshold (15% minimum)
- Handle orientation changes with dedicated listener
- Use `touchAction: "none"` to prevent scroll interference

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
