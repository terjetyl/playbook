# Skill: Responsive UI/UX

## Principles for this stack

Apps are React + Vite with plain CSS (no Tailwind). The form builder has a complex 3-column desktop layout that must degrade to a single-column stack on mobile. Public-facing pages (form player, landing, respondent portal) are the most mobile-visited surfaces — prioritise those.

## Mobile-first CSS

Write base styles for mobile, use `min-width` breakpoints to enhance for desktop:

```css
/* Mobile: single column */
.builder-layout {
  display: flex;
  flex-direction: column;
  gap: 1rem;
  padding: 1rem;
}

/* Tablet and up */
@media (min-width: 768px) {
  .builder-layout {
    flex-direction: row;
  }
}

/* Desktop */
@media (min-width: 1200px) {
  .builder-layout {
    display: grid;
    grid-template-columns: 280px 1fr 320px;
  }
}
```

## Standard breakpoints (use these consistently)

| Name    | Width        | Use for                             |
| ------- | ------------ | ----------------------------------- |
| mobile  | < 768px      | Single column, full-width inputs    |
| tablet  | 768px–1199px | 2-column layouts, sidebar collapses |
| desktop | ≥ 1200px     | Full multi-column layouts           |

Define as CSS custom properties in a single file:

```css
/* The breakpoints as variables aren't usable in @media — keep them documented */
/* --bp-tablet: 768px; */
/* --bp-desktop: 1200px; */
```

## Touch targets

Interactive elements must be at least **44×44px** on mobile (Apple HIG / WCAG 2.5.5):

```css
/* Any button, link, or interactive element */
.btn,
button,
[role="button"] {
  min-height: 44px;
  min-width: 44px;
  padding: 0.6rem 1rem;
}

/* Form inputs */
input,
textarea,
select {
  min-height: 44px;
  font-size: 16px; /* prevents iOS zoom on focus */
}
```

`font-size: 16px` on inputs **prevents the iOS Safari auto-zoom** — never go below this on mobile.

## Form player (most mobile-critical surface)

The public form player (`FormPlayer.tsx`) is embedded in customer websites and visited on phones. Rules:

```css
/* FormPlayer.css */
.form-player {
  max-width: 640px;
  margin: 0 auto;
  padding: 1rem;
}

/* Full-width inputs on mobile */
.form-field {
  width: 100%;
}

/* Rating stars — big enough to tap */
.rating-option {
  min-width: 44px;
  min-height: 44px;
  font-size: 1.5rem;
}

/* Multi-select options — stack vertically on mobile */
.multiselect-options {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

@media (min-width: 480px) {
  .multiselect-options {
    flex-direction: row;
    flex-wrap: wrap;
  }
}
```

## Navigation on mobile

The main `Layout.tsx` sidebar must collapse on mobile. Pattern:

```tsx
const [sidebarOpen, setSidebarOpen] = useState(false);

return (
  <>
    {/* Hamburger only on mobile */}
    <button
      className="mobile-menu-toggle"
      onClick={() => setSidebarOpen(true)}
      aria-label="Open menu"
    >
      ☰
    </button>

    {/* Sidebar */}
    <nav className={`sidebar ${sidebarOpen ? "sidebar--open" : ""}`}>
      <button
        className="sidebar__close"
        onClick={() => setSidebarOpen(false)}
        aria-label="Close menu"
      >
        ×
      </button>
      {/* nav items */}
    </nav>

    {/* Overlay to close sidebar on tap-outside */}
    {sidebarOpen && (
      <div className="sidebar-overlay" onClick={() => setSidebarOpen(false)} />
    )}
  </>
);
```

```css
.sidebar {
  position: fixed;
  top: 0;
  left: 0;
  height: 100dvh;
  width: 280px;
  transform: translateX(-100%);
  transition: transform 0.25s ease;
  z-index: 100;
}
.sidebar--open {
  transform: translateX(0);
}
.mobile-menu-toggle {
  display: block;
}

.sidebar-overlay {
  position: fixed;
  inset: 0;
  background: rgba(0, 0, 0, 0.4);
  z-index: 99;
}

@media (min-width: 768px) {
  .sidebar {
    position: sticky;
    transform: none;
    height: 100vh;
  }
  .mobile-menu-toggle {
    display: none;
  }
  .sidebar-overlay {
    display: none;
  }
}
```

## Tables on mobile

Tables break layout on small screens. Use a card-list pattern for mobile:

```css
/* Hide table on mobile, show cards */
.data-table {
  display: none;
}
.data-cards {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

@media (min-width: 768px) {
  .data-table {
    display: table;
    width: 100%;
  }
  .data-cards {
    display: none;
  }
}
```

Or use `overflow-x: auto` as a minimal fix for unimportant tables:

```css
.table-wrapper {
  overflow-x: auto;
  -webkit-overflow-scrolling: touch;
}
```

## Modals on mobile

Modals should become bottom sheets on small screens:

```css
.modal {
  position: fixed;
  inset: 0;
  display: flex;
  align-items: flex-end; /* bottom sheet on mobile */
  justify-content: center;
  padding: 0;
}

.modal__content {
  width: 100%;
  max-height: 90dvh;
  overflow-y: auto;
  border-radius: 1rem 1rem 0 0;
  padding: 1.5rem;
}

@media (min-width: 768px) {
  .modal {
    align-items: center;
    padding: 1rem;
  }
  .modal__content {
    width: auto;
    min-width: 480px;
    max-width: 640px;
    max-height: 85dvh;
    border-radius: 0.75rem;
  }
}
```

## Viewport height — use `dvh` not `vh`

`100vh` breaks on mobile browsers due to the address bar. Use `100dvh` (dynamic viewport height) instead:

```css
.full-height {
  height: 100dvh;
} /* correct */
.full-height {
  height: 100vh;
} /* breaks on iOS Safari */
```

## Testing responsive layouts

```bash
# Playwright: test at multiple viewport sizes
test('form player works on mobile', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 }); // iPhone 14
  await page.goto(`/form/${FORM_ID}`);
  // ...
});
```

In the browser: Chrome DevTools → Device Toolbar → test at 375px (iPhone SE) and 768px (iPad).

## useIsMobile hook

The project already has `hooks/useIsMobile.ts`. Use it for JS-driven layout decisions (not CSS-only ones):

```tsx
const isMobile = useIsMobile(); // checks window.innerWidth < 768

// Use for: hiding components, changing interaction patterns
// Do NOT use for: layout that CSS can handle alone
```
