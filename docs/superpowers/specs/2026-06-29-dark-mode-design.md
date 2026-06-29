# Dark Mode — Design Spec

**Date:** 2026-06-29
**Branch:** `feature/dark-mode`

## Goal

Add a dark mode to Morse Walker using Bootstrap 5.3's native theming, with a
header toggle and persisted preference. First-time visitors follow their OS
color-scheme preference.

## Mechanism

Use Bootstrap 5.3's native `data-bs-theme` attribute on the root `<html>`
element. Setting it to `"dark"` or `"light"` recolors all Bootstrap components
via their built-in CSS variables. No custom palette remapping, no new
dependencies, no build changes.

## Behavior

### Theme resolution (first load, no flash)

An inline script in the `<head>` of `index.html` runs before the page renders
to avoid a flash of the wrong theme (FOUC):

1. If `localStorage.theme` is set (`"dark"` or `"light"`), apply it.
2. Otherwise, fall back to `window.matchMedia('(prefers-color-scheme: dark)')`
   — the "follow OS" default. This OS-derived value is **not** written to
   `localStorage`; it is only persisted once the user explicitly toggles.

The inline script also sets the initial `<meta name="theme-color">` value to
match the resolved theme.

### Toggle

- A small icon button placed in the existing header row (the `d-flex` at
  `src/index.html:80`), near the title.
- Uses Font Awesome (already loaded): a moon icon while in light mode, a sun
  icon while in dark mode.
- Clicking flips the theme, writes the new choice to `localStorage.theme`, and
  swaps the icon and `theme-color` meta tag.

### Toggle logic

Lives in `app.js` alongside the existing `localStorage` preference handling:

- `setTheme(theme)` — sets `data-bs-theme` on `<html>`, updates the toggle icon,
  updates the `theme-color` meta tag.
- A click handler on the toggle button that computes the next theme, calls
  `setTheme`, and persists to `localStorage`.

## Loose ends

- **`theme-color` meta tag** (currently `#fafafa` at `src/index.html:75-76`):
  set initially by the inline script, updated on toggle. Dark value to match the
  Bootstrap dark surface.
- **`404.html`** hardcoded grays (`#888`, `#555`): add a `prefers-color-scheme:
  dark` media query so the error page is legible in dark mode. (404.html is a
  standalone page outside the app bundle, so it relies on the OS preference
  rather than the toggle.)
- **`style.css`** `#aaa` placeholder color: acceptable on both themes — leave
  unchanged.

## Footprint

~40 lines across `index.html`, `app.js`, and `404.html`. No build changes, no
new dependencies.

## Testing

No DOM/UI test suite currently exists. Verify manually via the dev server
(`npm start`):

1. First load with no stored preference follows the OS color scheme.
2. Toggle flips the theme immediately and the icon updates.
3. Preference persists across a page reload.
4. No flash of the wrong theme on load.
5. 404 page is legible in dark mode (OS-driven).

## Out of scope

- Multiple/custom themes beyond light and dark.
- Per-component color customization.
- Animated theme transitions.
