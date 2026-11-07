# ADR 0001: Use Bits UI for headless primitives

- **Status:** Accepted
- **Date:** 2026-11-07
- **Related:** `rewrite-plan.md` §4.1, §4.6, §9.2.1; `AGENTS.md` §2, §6

## Context

The rewrite uses Svelte 5 + SvelteKit and a single token-based design system. It needs accessible, unstyled behavior for complex primitives: dialogs, menus, selects, tabs, tooltips, popovers, sliders, switches, calendars, and date pickers.

Two Svelte headless libraries were considered: **Melt UI** (builder API) and **Bits UI** (component API). The planning document initially listed both as alternatives.

## Decision

Use **Bits UI** for headless primitives.

- Wrap Bits UI components in `src/lib/components` and style them with design tokens.
- Feature screens import the wrappers, not Bits UI directly.
- Use Bits UI's Calendar / Date Picker / Date Field components instead of a separate date-picker dependency.
- Keep `@tanstack/svelte-table` for admin tables; Bits UI does not replace a data table.
- Keep Lucide (`lucide-svelte`) for icons.

## Consequences

- One headless primitive library and one accessibility pattern across the app.
- Dialog/menu/select/tabs/tooltip/popover/slider/switch/date behavior is not hand-built.
- Bits UI is built for Svelte 5, so no legacy store/builder compatibility layer is needed.
- The app owns all visual styling through tokens; no component library theme is imposed.
- Melt UI is rejected for this project; do not add it.

## Alternatives considered

- **Melt UI** — capable builder API, but lower-level and its stable API predates the Bits UI component model; two libraries would duplicate accessibility patterns.
- **Fully custom primitives** — maximum control but significantly more work and a11y risk for menus, dialogs, selects, and date pickers.
- **MUI / React ecosystem** — rejected with the framework change.
