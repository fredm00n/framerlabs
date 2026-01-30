# Shared State Between Overrides

Make multiple elements react to each other using Framer's built-in store.

Useful for: tabs, accordions, modals, any "click here to change something over there" pattern.

```tsx
import type { ComponentType } from "react"
import { createStore } from "https://framer.com/m/framer/store.js@^1.0.0"

// Create a shared store (define once, use in multiple overrides)
const useStore = createStore({
    activeTab: "tab-1",
})

// Apply to clickable elements (tabs, buttons, etc.)
export function withTabTrigger(Component): ComponentType {
    return (props) => {
        const [store, setStore] = useStore()

        const handleClick = () => {
            setStore({ activeTab: "tab-2" })
        }

        return <Component {...props} onClick={handleClick} />
    }
}

// Apply to elements that should react to the state
export function withTabContent(Component): ComponentType {
    return (props) => {
        const [store] = useStore()

        // Show/hide based on active tab
        if (store.activeTab !== "tab-1") {
            return null
        }

        return <Component {...props} />
    }
}
```

You can also use it to control variants:

```tsx
export function withVariantReactor(Component): ComponentType {
    return (props) => {
        const [store] = useStore()
        return <Component {...props} variant={store.activeTab} />
    }
}
```
