# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kiaalap is a Bootstrap 5 education management dashboard with 65+ HTML pages using Handlebars templating and Vite build system. Originally a Bootstrap 4/jQuery template, it has been partially modernized — jQuery is still imported in `main.js` and exposed globally, despite the intent to be jQuery-free.

## Build Commands

```bash
npm run dev          # Dev server on port 3000 (auto-opens browser)
npm run build        # Production build to dist/
npm run preview      # Preview production build
npm run lint:html    # HTML validation (html-validate)
npm run lint:css     # SCSS linting (Stylelint)
npm run lint:js      # ESLint for JavaScript
npm run format       # Prettier formatting
npm run clean        # Remove dist/
```

No test suite exists (`npm test` exits with error).

## Architecture

### Tech Stack

- **Bootstrap 5.3.8** - CSS framework
- **Vite 7.3.1** - Build tool with `vite-plugin-handlebars`
- **Chart.js 4.5.1** - All charting (replaced Morris.js/C3/D3)
- **Simple-DataTables 10.0** - Vanilla JS data tables
- **Bootstrap Icons 1.13.1 + Font Awesome 7** - Both loaded from node_modules

### Entry Points and Load Order

HTML pages load resources through two Handlebars partials:

1. **`src/partials/head.hbs`** — Opens `<html>`, loads CSS:
   - `/src/scss/main.scss` (Bootstrap + Bootstrap Icons + Font Awesome)
   - `/src/css/dashboard.css` (dashboard-specific styles)
   - Inline styles for sidebar collapse behavior

2. **`src/partials/scripts.hbs`** — Loads JS, closes `</body></html>`:
   - `/src/js/main.js` as ES module (the true JS entry point)

**`src/js/main.js`** imports and initializes everything:

- Imports `main.scss` (Vite CSS-in-JS)
- Imports Bootstrap, AOS, CountUp, Chart.js, dayjs, Swiper, SimpleBar, TomSelect
- Exposes all libraries on `window.*` for use in page-level inline scripts
- Imports `./layout.js`, `./dashboard.js`, `./charts.js`

### Known Code Duplication

Three JS files have **overlapping sidebar toggle and Bootstrap init logic**:

- `src/js/dashboard.js` — Sidebar toggle, search, submenu, active highlighting, tooltips/popovers
- `src/js/layout.js` — Sidebar toggle, active highlighting, resize handling, tooltips/popovers
- `src/js/main.js` — Sidebar toggle, tooltips/popovers, CountUp, SimpleBar, TomSelect

Similarly, chart initialization is duplicated:

- `src/js/charts.js` — Charts + counters + progress bar animations (loaded via `main.js`)
- `src/js/charts-responsive.js` — Same charts + counters (loaded per-page via `additionalJS` in vite config)

### Two SCSS Entry Points

- **`src/scss/main.scss`** — Loaded by `head.hbs`. Imports Bootstrap, Bootstrap Icons, Font Awesome. Minimal custom styles.
- **`src/scss/app.scss`** — More comprehensive. Imports Bootstrap, Bootstrap Icons, variables, component partials (sidebar, header, buttons, dropdowns), utilities. Not currently loaded by any partial.

### Handlebars Template System

Every HTML page follows this structure:

```html
{{> head}}
{{> sidebar}}
<div class="main-wrapper" id="mainWrapper">
    {{> header}}
    <main class="dashboard-content" id="main-content">
        <div class="container-fluid">
            <!-- Page content -->
        </div>
    </main>
    {{> footer}}
</div>
{{> scripts}}
```

**Context is set per-page in `vite.config.js`** via `getPageContext(filename)`:

- Navigation states (which menu is active/expanded)
- Page title, breadcrumbs, description
- `additionalCSS` / `additionalJS` arrays for page-specific assets

### Navigation State: Dual Approach

Active menu states are set **twice**:

1. **Server-side** via Handlebars context in `vite.config.js` — sets `navigation.*` booleans that `sidebar.hbs` uses for `active` classes and `show` on collapse
2. **Client-side** in `dashboard.js` — JS checks `window.location.pathname` and adds `active` classes

### Adding a New Page

1. Create `newpage.html` in project root with standard Handlebars partials
2. Add page config in `vite.config.js` → `pageConfigs` object (title, breadcrumb, additionalCSS/JS)
3. Update navigation state logic in `getPageContext()` if the page belongs to an existing category
4. Add to sidebar in `src/partials/sidebar.hbs` if needed
5. Build automatically includes all `*.html` files (excludes `*template*` and `*-new*` patterns)

### Chart Pages

Pages with charts need this in their `vite.config.js` page config:

```javascript
additionalCSS: ['src/css/charts-layout.css'],
additionalJS: ['node_modules/chart.js/dist/chart.umd.js', 'src/js/charts-responsive.js']
```

Note: Chart.js is also imported via `main.js` as `chart.js/auto`, so it's loaded twice on chart pages.

### Handlebars Helpers

Available in `src/helpers/handlebars-helpers.js`:

- `eq` — Equality check: `{{#if (eq page 'index')}}`
- `exists`, `isActive`, `truncate`, `formatDate`, `conditionalClass`, `json`
- Math: `add`, `subtract`, `multiply`
- String: `lowercase`, `uppercase`, `startsWith`, `endsWith`, `contains`
- Array: `length`, `first`, `last`

### CSS Architecture

Active entry point: `src/scss/main.scss`
Dashboard styles: `src/css/dashboard.css` (plain CSS, not SCSS)

SCSS component partials (in `app.scss` but not currently active):

- `components/sidebar.scss` — Sidebar styling
- `components/dashboard.scss` — Dashboard-specific styles
- `bootstrap-overrides/` — Bootstrap customizations (buttons, cards, forms, navbar)
- `base/_variables.scss` — Custom variables

### Sidebar Behavior

Managed in `src/js/dashboard.js` (and duplicated in `layout.js`):

- **Desktop (>768px)**: Toggles `.collapsed` on sidebar + `.full-width` on main wrapper. State persisted to `localStorage` key `sidebarCollapsed`
- **Mobile (<=768px)**: Opens as overlay with `.active` class + backdrop. No persistence

### Vite Aliases

Defined in `vite.config.js`:

- `~bootstrap` → `node_modules/bootstrap`
- `~@fortawesome` → `node_modules/@fortawesome`
- `@` → `./src`

## Deployment

Full source deployment (not static build). Server must allow access to `node_modules/`. See `DEPLOYMENT_GUIDE.md` for Apache/Nginx configs.

## Key Libraries

| Library          | Where Used             | Init Pattern                                          |
| ---------------- | ---------------------- | ----------------------------------------------------- |
| Chart.js         | Dashboard + chart pages | Guard with `if (typeof Chart === 'undefined') return` |
| Simple-DataTables | `data-table.html`     | `new DataTable('#tableId', options)`                  |
| Quill            | `tinymc.html`          | Rich text editor                                      |
| FullCalendar     | `events.html`          | Calendar events                                       |
| Leaflet          | Map pages              | Interactive maps                                      |
| CropperJS        | `images-cropper.html`  | Image cropping                                        |
| Tom Select       | Advanced forms         | Enhanced select dropdowns (`.tom-select` class)       |
| SimpleBar        | Scrollable areas       | Custom scrollbars (`[data-simplebar]` attribute)      |
| CountUp.js       | Dashboard counters     | `.countup` class with `data-count` attribute          |
| AOS              | Scroll animations      | Initialized outside DOMContentLoaded in `main.js`     |
