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
}
```

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
    controls: {
        opacity: { type: ControlType.Number, defaultValue: 1 },
        color: { type: ControlType.Color, defaultValue: "#fff" },
    },
}
```

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
