# WebGL Shaders in Framer

## Basic Setup

```typescript
import { useEffect, useRef } from "react"

export default function ShaderComponent(props) {
    const canvasRef = useRef(null)

    useEffect(() => {
        const canvas = canvasRef.current
        const gl = canvas.getContext("webgl", { alpha: true })

        if (!gl) {
            console.error("WebGL not supported")
            return
        }

        // Setup and render loop here

        return () => {
            // Cleanup
        }
    }, [])

    return (
        <canvas
            ref={canvasRef}
            style={{
                width: "100%",
                height: "100%",
                display: "block",
            }}
        />
    )
}
```

## Transparency Support

Four requirements for transparent shaders:

### 1. Alpha-enabled Context
```typescript
const gl = canvas.getContext("webgl", { alpha: true })
```

### 2. Blending Configuration
```typescript
gl.enable(gl.BLEND)
gl.blendFunc(gl.SRC_ALPHA, gl.ONE_MINUS_SRC_ALPHA)
```

### 3. Transparent Clear Color
```typescript
gl.clearColor(0.0, 0.0, 0.0, 0.0)
gl.clear(gl.COLOR_BUFFER_BIT)
```

### 4. Fragment Shader Alpha Handling
```glsl
// Accumulate alpha in fragment shader
cl.a += smoothstep(2.5, 0.0, rz) * 0.1;

// Limit maximum opacity
color.a = min(color.a, 0.7);
```

## Shader Compilation Safety

**Always check shader compilation before attaching:**

```typescript
function createShader(gl, type, source) {
    const shader = gl.createShader(type)
    gl.shaderSource(shader, source)
    gl.compileShader(shader)

    if (!gl.getShaderParameter(shader, gl.COMPILE_STATUS)) {
        console.error("Shader compilation failed:", gl.getShaderInfoLog(shader))
        gl.deleteShader(shader)
        return null
    }
    return shader
}

function createProgram(gl, vertexShader, fragmentShader) {
    // Guard against null shaders
    if (!vertexShader || !fragmentShader) return null

    const program = gl.createProgram()
    gl.attachShader(program, vertexShader)
    gl.attachShader(program, fragmentShader)
    gl.linkProgram(program)

    if (!gl.getProgramParameter(program, gl.LINK_STATUS)) {
        console.error("Program linking failed:", gl.getProgramInfoLog(program))
        gl.deleteProgram(program)
        return null
    }
    return program
}

// Usage with null checks
const vertexShader = createShader(gl, gl.VERTEX_SHADER, vertexSource)
if (!vertexShader) return

const fragmentShader = createShader(gl, gl.FRAGMENT_SHADER, fragmentSource)
if (!fragmentShader) return

const program = createProgram(gl, vertexShader, fragmentShader)
if (!program) return
```

## Canvas Resize Handling

```typescript
function resizeCanvasToDisplaySize(canvas) {
    const displayWidth = canvas.clientWidth
    const displayHeight = canvas.clientHeight

    if (canvas.width !== displayWidth || canvas.height !== displayHeight) {
        canvas.width = displayWidth
        canvas.height = displayHeight
        return true
    }
    return false
}

// In render loop
function render() {
    resizeCanvasToDisplaySize(gl.canvas)
    gl.viewport(0, 0, gl.canvas.width, gl.canvas.height)

    // Draw calls...

    requestAnimationFrame(render)
}
```

## Uniform Updates

```typescript
const resolutionLocation = gl.getUniformLocation(program, "resolution")
const timeLocation = gl.getUniformLocation(program, "time")

let startTime = performance.now()

function render() {
    gl.useProgram(program)
    gl.uniform2f(resolutionLocation, gl.canvas.width, gl.canvas.height)
    gl.uniform1f(timeLocation, (performance.now() - startTime) * 0.001)

    // Draw...
    requestAnimationFrame(render)
}
```

## Cleanup Pattern

```typescript
useEffect(() => {
    let animationId

    // Setup...

    function render() {
        // Draw...
        animationId = requestAnimationFrame(render)
    }
    render()

    return () => {
        if (animationId) {
            cancelAnimationFrame(animationId)
        }
        // Additional cleanup: delete shaders, programs, buffers
    }
}, [])
```

## Canvas Positioning for Effects

For effects that need to extend beyond component bounds:

```typescript
const safetyMargin = 100

// Canvas larger than container
const canvasWidth = containerWidth + safetyMargin * 2
const canvasHeight = containerHeight + safetyMargin * 2

// Position with negative offset
<canvas
    style={{
        position: "absolute",
        top: -safetyMargin,
        left: -safetyMargin,
        width: canvasWidth,
        height: canvasHeight,
        pointerEvents: "none",
    }}
/>
```

Container needs `overflow: visible` for this pattern.
