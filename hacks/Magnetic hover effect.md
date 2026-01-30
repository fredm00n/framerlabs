# Magnetic Hover Effect

Makes elements subtly follow the cursor when hovered, creating a "magnetic" feel.

Create a Code Override and apply to buttons, cards, or any element you want to feel interactive.

```tsx
import type { ComponentType } from "react"
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

Adjust the `0.3` multiplier to control intensity (higher = more movement).
