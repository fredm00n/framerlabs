# CMS in Code Components

Code components consume a Framer CMS collection through a `ControlType.ComponentInstance` Collection List, then walk the resulting React element tree to extract per-item content. The mechanism is undocumented but stable.

## The property control accepts a Collection List instance

```typescript
addPropertyControls(MyCMSComponent, {
    collectionList: {
        type: ControlType.ComponentInstance,
        title: "Collection",
    },
})
```

The user drops a Framer-native Collection List on the canvas and wires it to this slot.

## Extract items via `useQueryData` + `getCollectionData`

```typescript
import { useQueryData, RenderTarget } from "framer"
import { Children } from "react"
import { getCollectionData } from "https://framer.com/m/CMSLibrary-09eo.js"

function getCollectionListItems(collectionList) {
    const { query, childrenFunction } = getCollectionData(collectionList)
    if (!query || !childrenFunction) return []

    const data = useQueryData(query)
    const clChildren = childrenFunction(data)

    let children = []
    if (Array.isArray(clChildren)) {
        children = clChildren
    } else if (Array.isArray(clChildren?.props?.children?.[0])) {
        children = clChildren.props.children[0]
    } else if (Array.isArray(clChildren?.props?.children)) {
        children = clChildren.props.children
    }
    return Children.toArray(children)
}
```

Item content lives at `item.props.children.props.children` — read past two wrapper levels to reach the user's template.

## Extract named layers from each item with `findByFramerName`

Framer surfaces canvas layer names via the `data-framer-name` attribute. Walk the React element tree:

```typescript
import { isValidElement } from "react"

function findByFramerName(node, name) {
    if (node == null) return null
    if (Array.isArray(node)) {
        for (const child of node) {
            const found = findByFramerName(child, name)
            if (found) return found
        }
        return null
    }
    if (!isValidElement(node)) return null
    if (node.props?.["data-framer-name"] === name) return node
    if (node.props?.name === name) return node // fallback wrapper case
    return findByFramerName(node.props?.children, name)
}
```

## The plain-frame trap

`findByFramerName` only finds layers whose name ends up on `data-framer-name`. **Plain frames get wrapped by Framer in a component carrying `data-framer-name`**, but **dynamic component instances do NOT propagate their canvas name** — the instance name you set when renaming on the canvas is editor-only metadata and never lands on a prop the walker can see.

If a named slot needs to be a dynamic component (hover variants, internal animations), wrap it in a plain frame carrying the expected name:

```
Item template
├── Background     ← plain frame, named "Background"
└── MovingTitles   ← plain frame, named "MovingTitles"
    └── <dynamic component>   ← any name; hover/variants live here
```

## Canvas preview limitation

On the canvas, `useQueryData` returns **only one representative item**, not the full list. Components that depend on iterating the list cannot preview meaningfully in the editor. Two strategies:

```typescript
if (isCanvas) {
    return <>{props.collectionList}</> // Framer's default vertical list preview
}

// Or a placeholder if even that doesn't make sense:
function CanvasPlaceholder({ title, subtitle }) {
    return (
        <div style={{ /* dashed border, centered text */ }}>
            <p>{title}</p>
            <p>{subtitle}</p>
        </div>
    )
}
```

## Inherit controls from a wrapped component

If your CMS component wraps another code component (e.g. a CMS slideshow wrapping a generic slideshow), spread its property controls so the user sees all the same knobs:

```typescript
import { getPropertyControls } from "framer"
import GenericSlideshow from "./GenericSlideshow.tsx"

function getComponentProps(component) {
    const { slots, ...rest } = getPropertyControls(component)
    return rest // drop the slots control — the CMS version provides its own
}

addPropertyControls(CMSSlideshow, {
    collectionList: { type: ControlType.ComponentInstance, title: "Collection" },
    ...getComponentProps(GenericSlideshow),
})
```
