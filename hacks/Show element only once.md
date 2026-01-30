# Show Element Only Once

Show a popup, banner, or tooltip only on the first visit. Uses localStorage to remember if it's been shown.

```tsx
import type { ComponentType } from "react"
import { useState, useEffect } from "react"

export function withShowOnce(Component): ComponentType {
    return (props) => {
        const [isClient, setIsClient] = useState(false)
        const [shouldShow, setShouldShow] = useState(false)

        useEffect(() => {
            setIsClient(true)
            try {
                const hasShown = localStorage.getItem("hasShownBanner")
                if (!hasShown) {
                    setShouldShow(true)
                    localStorage.setItem("hasShownBanner", "true")
                }
            } catch {
                // localStorage not available, show anyway
                setShouldShow(true)
            }
        }, [])

        // Don't render anything during SSR
        if (!isClient) return null

        // Hide if already shown before
        if (!shouldShow) return null

        return <Component {...props} />
    }
}
```

To reset during testing, run in browser console:
```js
localStorage.removeItem("hasShownBanner")
```

Change `"hasShownBanner"` to a unique key if you have multiple show-once elements.
