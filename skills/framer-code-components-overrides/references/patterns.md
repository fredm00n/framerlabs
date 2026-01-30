# Common Framer Patterns

## Shared State Between Overrides

Use Framer's store for state shared across multiple overrides:

```typescript
import { createStore } from "https://framer.com/m/framer/store.js@^1.0.0"

const useStore = createStore({
    variant: "default",
    count: 0,
})

export function withTrigger(Component): ComponentType {
    return (props) => {
        const [store, setStore] = useStore()

        const handleClick = () => {
            setStore({ variant: "active" })
        }

        return <Component {...props} onClick={handleClick} />
    }
}

export function withReactor(Component): ComponentType {
    return (props) => {
        const [store] = useStore()
        return <Component {...props} variant={store.variant} />
    }
}
```

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
