# Accessibility (a11y) Guide

## Core Principles

- **Semantic HTML:** Use correct HTML elements (`<button>`, `<nav>`, `<main>`, `<article>`)
- **Keyboard Navigation:** All interactive elements must be keyboard accessible
- **Screen Reader Support:** Provide descriptive labels and ARIA attributes
- **Color Contrast:** Maintain WCAG AA minimum (4.5:1 for normal text, 3:1 for large text)

---

## ARIA Attributes

```typescript
// Good: Semantic HTML with appropriate ARIA
<button aria-label="Close dialog" onClick={handleClose}>
  <XIcon aria-hidden="true" />
</button>

<nav aria-label="Main navigation">
  <ul>
    <li><a href="/">Home</a></li>
  </ul>
</nav>

// Bad: Generic div without proper role/labels
<div onClick={handleClose}>
  <XIcon />
</div>
```

---

## Keyboard Navigation

- **Focus Management:** Trap focus in modals, restore focus when closing
- **Skip Links:** Provide "Skip to main content" for keyboard users
- **Tab Order:** Ensure logical tab order matches visual order

```typescript
// Example: Modal focus trap
import { useEffect, useRef } from 'react';

function Modal({ isOpen, onClose, children }: ModalProps) {
  const modalRef = useRef<HTMLDivElement>(null);
  const previousFocusRef = useRef<HTMLElement | null>(null);
  
  useEffect(() => {
    if (isOpen) {
      previousFocusRef.current = document.activeElement as HTMLElement;
      modalRef.current?.focus();
    } else {
      previousFocusRef.current?.focus();
    }
  }, [isOpen]);
  
  return (
    <div
      ref={modalRef}
      role="dialog"
      aria-modal="true"
      aria-labelledby="modal-title"
      tabIndex={-1}
    >
      {children}
    </div>
  );
}
```

---

## Alternative Text

- **Images:** Provide descriptive `alt` text (empty string for decorative images)
- **Icons:** Use `aria-label` or `aria-labelledby` for icon-only buttons
- **SVG:** Add `<title>` and `role="img"` for meaningful graphics

```typescript
// Meaningful image
<img src="/product.jpg" alt="Blue cotton t-shirt with crew neck" />

// Decorative image
<img src="/divider.svg" alt="" aria-hidden="true" />

// Icon button
<button aria-label="Search">
  <SearchIcon aria-hidden="true" />
</button>
```

---

## 3D/Spatial Accessibility

- **Provide 2D Alternatives:** Offer non-3D interface for users who can't access WebGL/XR
- **Motion Sensitivity:** Respect `prefers-reduced-motion` for animations
- **Spatial Audio Captions:** Provide visual indicators for spatial audio cues

```typescript
// Respect motion preferences
const prefersReducedMotion = window.matchMedia('(prefers-reduced-motion: reduce)').matches;

// Three.js: Disable auto-rotation if user prefers reduced motion
if (!prefersReducedMotion) {
  controls.autoRotate = true;
}
```

---

## Testing Tools

- **Lighthouse:** Run accessibility audits in Chrome DevTools
- **axe DevTools:** Browser extension for detailed a11y testing
- **Keyboard Only:** Test entire app using only keyboard (no mouse)
- **Screen Reader:** Test with NVDA (Windows), VoiceOver (Mac), or JAWS

---

## Common Patterns

### Skip to Main Content
```typescript
<a href="#main-content" className="sr-only focus:not-sr-only">
  Skip to main content
</a>

<main id="main-content">
  {/* Page content */}
</main>
```

### Form Labels
```typescript
// Explicit label association
<label htmlFor="email">Email</label>
<input id="email" type="email" name="email" />

// Implicit label (less preferred)
<label>
  Email
  <input type="email" name="email" />
</label>

// aria-label for inputs without visible labels
<input type="search" aria-label="Search products" />
```

### Live Regions
```typescript
// Announce dynamic content changes to screen readers
<div role="status" aria-live="polite">
  {message}
</div>

<div role="alert" aria-live="assertive">
  {errorMessage}
</div>
```
