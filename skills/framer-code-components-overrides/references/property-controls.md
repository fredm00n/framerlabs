# Property Controls Reference

## Control Types

### String
```typescript
text: {
    type: ControlType.String,
    title: "Text",
    defaultValue: "Hello",
    placeholder: "Enter text...",
    displayTextArea: true, // For multiline
    preventLocalization: true, // Prevents automatic translation
}
```

Use `preventLocalization: true` for technical content that shouldn't be translated (API keys, code snippets, etc.).

### Number
```typescript
value: {
    type: ControlType.Number,
    title: "Value",
    defaultValue: 50,
    min: 0,
    max: 100,
    step: 1,
    unit: "px",
    displayStepper: true,
}
```

### Boolean
```typescript
enabled: {
    type: ControlType.Boolean,
    title: "Enabled",
    defaultValue: true,
    enabledTitle: "On",
    disabledTitle: "Off",
}
```

### Color
```typescript
color: {
    type: ControlType.Color,
    title: "Color",
    defaultValue: "#09f",
}
```

### Enum (Dropdown or Segmented)
```typescript
mode: {
    type: ControlType.Enum,
    title: "Mode",
    defaultValue: "auto",
    options: ["auto", "manual", "disabled"],
    optionTitles: ["Auto", "Manual", "Disabled"],
    displaySegmentedControl: true, // false for dropdown
}
```

### Image
```typescript
image: {
    type: ControlType.Image,
    title: "Image",
}
```

Returns URL string. For previews in the control panel, use `ControlType.ResponsiveImage`.

### ResponsiveImage / File — No defaultValue Support

**Critical**: `ControlType.ResponsiveImage` and `ControlType.File` do NOT support `defaultValue` in property controls. You must use component parameter destructuring instead:

```typescript
// ❌ WRONG - defaultValue is ignored
addPropertyControls(MyComponent, {
    image: {
        type: ControlType.ResponsiveImage,
        defaultValue: { src: "...", alt: "..." }, // This does nothing!
    },
})

// ✅ CORRECT - Use parameter destructuring
function MyComponent(props) {
    const {
        image = {
            src: "https://framerusercontent.com/images/GfGkADagM4KEibNcIiRUWlfrR0.jpg",
            alt: "Default image"
        }
    } = props

    return <img {...image} />
}

addPropertyControls(MyComponent, {
    image: { type: ControlType.ResponsiveImage },
})
```

Same pattern applies to `ControlType.File`:
```typescript
function MyComponent(props) {
    const {
        videoFile = "https://framerusercontent.com/assets/MLWPbW1dUQawJLhhun3dBwpgJak.mp4"
    } = props

    return <video src={videoFile} />
}
```

### File
```typescript
video: {
    type: ControlType.File,
    title: "Video",
    allowedFileTypes: ["mp4", "webm"],
}
```

### Font (Extended)
```typescript
font: {
    type: ControlType.Font,
    title: "Font",
    controls: "extended",
    defaultValue: {
        fontFamily: "Inter",
        fontWeight: 500,
        fontSize: 16,
        lineHeight: "1.5em",
    },
}
```

**Critical**: Always spread the entire font object in styles: `...props.font`

#### Variant-to-FontWeight Mapping

Framer uses `variant` names that map to CSS `font-weight` and `font-style`:

| Variant | font-weight | font-style |
|---------|-------------|------------|
| Thin | 100 | normal |
| Extra Light | 200 | normal |
| Light | 300 | normal |
| Regular | 400 | normal |
| Medium | 500 | normal |
| Semibold | 600 | normal |
| Bold | 700 | normal |
| Extra Bold | 800 | normal |
| Black | 900 | normal |
| Thin Italic | 100 | italic |
| Extra Light Italic | 200 | italic |
| Light Italic | 300 | italic |
| Italic / Regular Italic | 400 | italic |
| Medium Italic | 500 | italic |
| Semibold Italic | 600 | italic |
| Bold Italic | 700 | italic |
| Extra Bold Italic | 800 | italic |
| Black Italic | 900 | italic |

Use these variant names in `defaultValue.variant` when setting up Font controls.

### Transition
```typescript
transition: {
    type: ControlType.Transition,
    title: "Transition",
    defaultValue: { duration: 0.5 },
}
```

### ComponentInstance
```typescript
icon: {
    type: ControlType.ComponentInstance,
    title: "Icon",
}
```

Accepts a component from the canvas.

### Array
```typescript
items: {
    type: ControlType.Array,
    title: "Items",
    maxCount: 10,
    control: {
        type: ControlType.String,
    },
}
```

### Array with Image Preview
```typescript
images: {
    type: ControlType.Array,
    title: "Images",
    control: {
        type: ControlType.Object,
        controls: {
            image: {
                type: ControlType.ResponsiveImage,
            },
        },
    },
}
```

### Object (Grouped Controls)
```typescript
settings: {
    type: ControlType.Object,
    title: "Settings",
    icon: "effect", // UI icon: "object" | "effect" | "color" | "interaction" | "boolean"
    controls: {
        opacity: { type: ControlType.Number, defaultValue: 1 },
        color: { type: ControlType.Color, defaultValue: "#fff" },
    },
}
```

The `icon` property changes how the control appears in Framer's property panel. Options: `"object"`, `"effect"`, `"color"`, `"interaction"`, `"boolean"`.

## Conditional Visibility

Hide controls based on other prop values:

```typescript
advancedMode: {
    type: ControlType.Boolean,
    defaultValue: false,
},
advancedSetting: {
    type: ControlType.Number,
    hidden: (props) => !props.advancedMode,
},
```

## Markdown Descriptions

```typescript
setting: {
    type: ControlType.Number,
    description: "Controls the *intensity*.\n\n[Learn more](https://example.com)",
}
```

## Default Props Requirement

Property Controls only affect canvas. Always define `defaultProps` for:
- Preventing runtime errors
- Components instantiated from code
- Initial render before controls apply

```typescript
MyComponent.defaultProps = {
    text: "Default",
    color: "#000",
    font: {
        fontFamily: "Inter",
        fontWeight: 500,
        fontSize: 16,
    },
}
```

## Common Patterns

### Mode-Dependent Controls
```typescript
addPropertyControls(Component, {
    mode: {
        type: ControlType.Enum,
        options: ["image", "video"],
        defaultValue: "image",
    },
    image: {
        type: ControlType.Image,
        hidden: (props) => props.mode !== "image",
    },
    video: {
        type: ControlType.File,
        allowedFileTypes: ["mp4"],
        hidden: (props) => props.mode !== "video",
    },
})
```

### Nested Object with Conditional
```typescript
progressOptions: {
    type: ControlType.Object,
    title: "Progress",
    controls: {
        showProgress: {
            type: ControlType.Boolean,
            defaultValue: false,
        },
        progressFont: {
            type: ControlType.Font,
            controls: "extended",
            hidden: (props) => !props.showProgress,
        },
    },
}
```

Note: In nested objects, `hidden` receives the parent object, not root props.
