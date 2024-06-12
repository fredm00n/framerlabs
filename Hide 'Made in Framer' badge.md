# Untitled

Created time: June 12, 2024 9:58 AM
Status: Not started

This code hides the Made in Framer badge.

Create a new Code Override and apply to any Canvas element.

```tsx
import type { ComponentType } from "react"

export function withoutFramerBadge(Component): ComponentType {
    return (props) => {
        return (
            <>
                <style>
                    {`
                        #__framer-badge-container { display: none !important; }
                    `}
                </style>

                <Component {...props} />
            </>
        )
    }
}
```
