# Auto-Sized Text Fix

When using auto-sized components (`@framerSupportedLayoutWidth auto`) with text, the content can unexpectedly collapse or wrap incorrectly.

Apply `minWidth: max-content` to text elements to prevent this:

```tsx
/**
 * @framerSupportedLayoutWidth auto
 * @framerSupportedLayoutHeight auto
 */
export default function TextComponent(props) {
    const { label, font } = props

    return (
        <span
            style={{
                minWidth: "max-content", // Prevents collapse
                ...font,
            }}
        >
            {label}
        </span>
    )
}
```

For components with multiple text elements, apply to each:

```tsx
<div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
    <h1 style={{ minWidth: "max-content", ...props.titleFont }}>
        {props.title}
    </h1>
    <p style={{ minWidth: "max-content", ...props.bodyFont }}>
        {props.body}
    </p>
</div>
```

This ensures the component properly calculates its intrinsic width based on the actual text content rather than collapsing to zero or wrapping prematurely.
