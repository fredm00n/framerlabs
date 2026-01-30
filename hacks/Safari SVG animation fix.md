# Safari SVG Animation Fix

SVG animations can be choppy or laggy on Safari. Force GPU acceleration to fix it.

Create a Code Override and apply to SVG elements or their containers.

```tsx
import type { ComponentType } from "react"

export function withSafariSVGFix(Component): ComponentType {
    return (props) => {
        return (
            <Component
                {...props}
                style={{
                    ...props.style,
                    willChange: "transform",
                    transform: "translateZ(0)",
                    backfaceVisibility: "hidden",
                }}
            />
        )
    }
}
```

These properties force the browser to use hardware acceleration, which fixes Safari's rendering issues with SVG transforms and animations.
