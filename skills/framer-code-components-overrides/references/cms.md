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

**The slot arrives as an array.** Framer now passes a `ComponentInstance` control as an array of
one element (the agent DSL lists it as `{"type":"slot","maxCount":1}`). Given the array,
`getCollectionData` finds no `query` and the component draws its empty state on the published page.
Unwrap it first (2026-10-16):

```typescript
const list = Array.isArray(props.collectionList) ? props.collectionList[0] : props.collectionList
const { query, propertyControlsByName, childrenFunction } = getCollectionData(list)
```

From an agent, fill the slot with the list's id, a ground node of the page:
`SET <instanceId> $control__<name>.0="<listId>"`. `<name>` comes from the control's `title`, not
its key: a control keyed `collectionList` with title "Collection" is `$control__collection`.

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

## Read fields the template does not show

`addFieldsToSelect` (same module) adds fields to the query, so the component can read a value the
item template never binds, such as a date to place items on a calendar. Rows come back keyed by
field id, in the list's sort order, and line up with the rendered items:

```typescript
import { getCollectionData, addFieldsToSelect } from "https://framer.com/m/CMSLibrary-09eo.js"

const select = addFieldsToSelect(["Date"], query, propertyControlsByName)
const rows = useQueryData({ ...query, select })
const dateId = propertyControlsByName.Date.id
const items = getCollectionListItems(list) // rows[i] belongs to items[i]
```

## Dates are wall-clock times written as UTC

A CMS date with time holds the time the editor typed, written as UTC: 7 pm is
`"…T19:00:00.000Z"`. Framer's own date text shows it unchanged in every time zone (7:00 PM in New
York and in Tokyo, checked on a published page with the browser's time zone switched). So take an
item's day from `getUTCFullYear()`, `getUTCMonth()` and `getUTCDate()`; `toLocaleDateString()`
would move a late event to the next or previous day for some visitors. Today's date is a separate
choice, usually the venue's time zone through `Intl.DateTimeFormat({ timeZone })` (2026-10-16).

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
