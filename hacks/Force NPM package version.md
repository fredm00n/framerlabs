# Force NPM Package Version

Framer caches npm packages aggressively. When you need a specific version (or the cache is stuck on an old one), use esm.sh to bypass it.

## Standard import (uses Framer's cache)

```tsx
import { Component } from "package-name"
```

## Force specific version via CDN

```tsx
import { Component } from "https://esm.sh/package-name@1.2.3"
```

## For React components (important!)

Always add `?external=react,react-dom` to avoid duplicate React instances:

```tsx
import { Component } from "https://esm.sh/package-name@1.2.3?external=react,react-dom"
```

## Example with a real package

```tsx
// Force lucide-react version 0.263.1
import { Check, X } from "https://esm.sh/lucide-react@0.263.1?external=react,react-dom"
```

This is useful when:
- A package update broke something and you need to pin an older version
- Framer's cache is serving stale code
- You need a beta/canary release
