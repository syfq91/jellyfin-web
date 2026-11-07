# AGENTS.md

Instructions for AI coding agents working on the Jellyfin Web lightweight rewrite.

## 0. Read this first

**Authority order:** `rewrite-plan.md` → this file → existing source code.

- `rewrite-plan.md` is the product/architecture source of truth. It contains the full screen/feature inventory (§3), architecture (§4), parity matrix (§6), delivery phases (§7), and all decision records (§9).
- This file tells agents *how* to work in the repo. It does not replace the plan.
- The current `src/` tree is **upstream `jellyfin-web` 13.0.0** (React + MUI + legacy jQuery). Treat it as the **behavioral reference**, not the target implementation.
- If a request conflicts with the plan or this file, stop and ask. Do not silently expand scope or re-litigate resolved decisions.

Before starting a task, read at minimum:

1. `rewrite-plan.md` §0 (TL;DR), §1 (goals), §9.1 (resolved decisions).
2. The relevant screen/feature section in §3.
3. The parity checklist in §6 and the phase plan in §7.

**Status:** planning complete; implementation starts with the Phase 0 SvelteKit scaffold. Do not assume a Svelte app exists yet — check the working tree first.

---

## 1. Project summary

A lightweight, **browser-only** alternative UI for Jellyfin.

- **Behavioral parity** with the current modern web UI; **no pixel parity**.
- **Modern layout only**; no legacy layout, no TV layout, no layout switcher.
- **No native-shell bridge** (`window.NativeShell` is removed).
- **Download only**; no offline mode.
- **Plugin configuration pages** are server HTML rendered in a sandboxed iframe with a typed `postMessage` bridge.
- Deployed as a static SPA served from the Jellyfin web root (`/web/`), hash-routed (`/#/home`), with legacy `#!` link compatibility.
- **Modern browsers only** — last 2 Chrome/Edge/Firefox, Firefox ESR, Safari 16+, iOS 16+, Android Chrome last 2.

### Stack

| Concern | Choice |
|---|---|
| Framework | Svelte 5 (runes) + SvelteKit 2 + TypeScript |
| Build | SvelteKit + Vite + `@sveltejs/adapter-static` (`ssr = false`, SPA fallback) |
| Routing | Hash router for top-level screens; SvelteKit's History router is not used for the screen table |
| Icons | Lucide (https://lucide.dev/) via `lucide-svelte`, per-icon imports |
| UI primitives | Bits UI or Melt UI + internal token-styled components |
| Styling | Design tokens + CSS custom properties + component-scoped Svelte styles |
| Server state | `@tanstack/svelte-query` + IndexedDB persistence + WebSocket invalidation |
| Client state | Svelte 5 runes + small typed stores |
| API | `@jellyfin/sdk` + a thin `SessionManager`; no `jellyfin-apiclient` |
| i18n | `svelte-i18n` (or a small adapter) over the existing JSON catalogs |
| Tests | Vitest + `@testing-library/svelte` + Playwright (real Jellyfin server) |

---

## 2. Hard constraints (do not violate)

- **No React, ReactDOM, MUI, Emotion, jQuery, jstree, web components, or Material icon fonts.** Do not add them "temporarily."
- **No legacy layout**, no `layoutManager`, no `desktop-legacy`/`mobile-legacy`/`tv` branches, no layout preference.
- **No `window.NativeShell`** and no native-only UI (download manager, client settings, exit app, native device profile, native plugins).
- **No offline media cache**, no service-worker media interception, no "available offline" state.
- **No top-level history routing.** Keep `/#/...` and `#!` links working; hash routing is a product requirement.
- **No pixel-for-pixel visual parity requirement.** Behavior parity is the acceptance bar; use the new design tokens.
- **Lucide only** for icons. No icon font, no mixed icon sets.
- **All user-facing strings go through the i18n layer.** No hard-coded English.
- **One data layer.** No ad-hoc `fetch`/`axios` calls outside the API/session modules.
- **One styling system.** No hard-coded colors/spacing outside tokens; no second CSS framework.
- **Do not rewrite plugin configuration pages in Svelte.** Keep the server HTML iframe.
- **Do not re-implement server behavior.** The Jellyfin API is the contract.

If a dependency or pattern is missing, prefer a small local implementation or ask. Do not introduce a heavy framework/library without an explicit decision.

---

## 3. Repository map (current reference)

| Path | Meaning |
|---|---|
| `rewrite-plan.md` | The plan and decision log — read before implementing |
| `src/RootAppRouter.tsx` | Current route composition (modern/legacy/dashboard/wizard selection) |
| `src/apps/modern/` | Modern user app: library pages, preferences, toolbar/drawer |
| `src/apps/dashboard/` | Admin dashboard (the closest thing to a modern React implementation) |
| `src/apps/legacy/` | Legacy app + controller/view pages — **reference only** |
| `src/apps/wizard/` | First-run wizard |
| `src/components/` | Shared legacy + modern components (players, editors, dialogs) |
| `src/strings/*.json` | Translation catalogs (109 locales, ~1,949 en-US keys) |
| `src/themes/` | Existing theme presets and theme storage |
| `src/plugins/` | Player, screensaver, SyncPlay plugins |
| `package.json` | Current build/test/lint commands |

Key reference files for behavior:

- Routing: `src/apps/*/routes/routes.tsx`, `src/apps/{modern,legacy}/routes/{asyncRoutes,legacyRoutes}/*`
- Playback: `src/components/playback/playbackmanager.js`, `src/apps/legacy/controllers/playback/video/*`
- Home sections: `src/components/homesections/homesections.js`, `src/constants/homeSectionType.ts`
- Item details: `src/apps/legacy/controllers/itemDetails/*`
- Now playing / queue: `src/components/nowPlayingBar/*`, `src/apps/legacy/controllers/playback/queue/*`
- Preferences: `src/apps/modern/features/preferences/*`, `src/apps/legacy/routes/user/settings/*`
- Admin: `src/apps/dashboard/**`
- Native-shell removal checklist: `rewrite-plan.md` Appendix E

> Legacy code is the best record of edge-case behavior. Read it, port the behavior, do not copy its architecture (jQuery, custom elements, view/controller lifecycle).

---

## 4. Target layout (when scaffolding starts)

Follow SvelteKit conventions, but keep feature code grouped by domain:

```
src/
  routes/
    +layout.ts            # export const ssr = false
    +page.svelte          # app shell + hash router mount
  lib/
    api/                  # SDK client, SessionManager, query keys
    components/           # primitives, icons, cards, lists, dialogs
    features/
      auth/ home/ libraries/ details/ playback/ search/
      preferences/ admin/ wizard/ livetv/
    player/               # PlaybackController + adapters
    platform/             # browser capability detection (no NativeShell)
    styles/               # tokens, global styles
    i18n/                 # catalogs + helpers
    router/               # typed hash route table + guards
  strings/                # copied/adapted translation catalogs
tests/
  unit/
  e2e/
```

- Route screens are lazy `import()` chunks.
- Heavy features (admin, readers, PDF, HLS/cast) must not be in the initial chunk.
- A feature folder owns its components, hooks/runes, queries, and tests. Avoid a global `components/` dumping ground.

This layout is a guideline; record structural changes in `docs/adr/`.

---

## 5. Commands

Use the commands in `package.json`. Do not invent command names.

**Current reference app (until the rewrite is scaffolded):**

```bash
npm ci
npm run serve                  # webpack dev server
npm run build:development
npm run build:production
npm run build:check            # tsc --noEmit
npm run lint
npm run stylelint
npm run test                   # Vitest, single run
```

**Target SvelteKit app (add these when scaffolding):**

```bash
npm run dev                    # SvelteKit dev server
npm run build                  # adapter-static SPA build
npm run preview
npm run check                  # svelte-check + tsc
npm run lint
npm run test                   # Vitest
npm run test:e2e               # Playwright against a real Jellyfin server
```

Always run the checks relevant to the files you touched. For UI work that means at least: type check, lint, unit/component tests, and a manual/hash-route smoke test.

---

## 6. Architecture rules

### Boot and session

- One `SessionManager` owns server list, connection, auth token, current `Api`, current user, and WebSocket.
- One `ApiProvider`-equivalent exposes `{ api, serverId, user, session }` to the app.
- All requests go through one configured SDK client/axios instance with auth headers and 401 handling.
- Do not store credentials. Persist only server URLs, device id, token, and user id.
- Handle server restart/shutdown events by invalidating/clearing cached data.

### Routing

- SvelteKit is the app/build framework; the top-level screen table uses the **hash router**.
- A single root page mounts the shell and router. Route entries are typed and lazy-loaded.
- `RequireConnection(level: public | user | admin | wizard)` guards mirror the current behavior.
- Admin routes are lazy and blocked for non-admins.
- Preserve `/#/...`, `#!`, and `#!/` link forms.
- Never introduce a layout mode or a second route system.

### Data

- Server state lives in TanStack Query. Define query keys per domain and invalidate by prefix.
- WebSocket events invalidate the relevant keys; do not poll unless a feature requires it.
- Mutations invalidate affected queries and show a toast on success/failure.
- Persist only cheap, safe queries to IndexedDB. Never persist credentials or playback/session state.
- Use the generated SDK types. No hand-written API response shapes unless the SDK lacks them.

### Player

- All playback goes through `PlaybackController`; components do not call media elements or APIs directly.
- Adapters: HTML5 video/audio, HLS (lazy), cast, remote session, book, comic, PDF, photo.
- Playback negotiation uses `PlaybackInfo`; device profile is browser-only.
- Progress reporting, queue, shuffle/repeat, tracks, subtitles, trickplay, segments, and next-up live in the controller/adapters, not in screen components.
- Heavy player chunks must stay out of the initial bundle.

### UI, styling, icons

- Use the design tokens for color, spacing, typography, radius, elevation, motion, focus.
- Use `lucide-svelte` imports per icon; wrap common icons in a shared `Icon` component.
- Use Bits UI/Melt UI for accessible behavior; style them with tokens.
- No ad-hoc inline styles for themeable values.
- All strings via `t()`; no hard-coded English.
- Keyboard-first: every interactive element focusable, visible focus, predictable focus return after dialogs/menus.
- Respect `prefers-reduced-motion` and `prefers-color-scheme`.
- Sanitize server-provided Markdown/HTML before rendering (metadata, plugin content, custom CSS is server-admin trusted but still isolated).

### Plugins

- Plugin configuration pages load in a same-origin iframe route.
- Use the typed `postMessage` bridge (`ready`, `resize`, `theme`, `toast`, `navigate`, `close`, `dirty`) with origin checks.
- Treat plugin pages as trusted admin content; do not claim the iframe is a security boundary.
- Never rewrite plugin HTML into Svelte.

### Platform

- Browser-only. No `window.NativeShell`, no native menu entries, no native device profile, no native player plugins.
- Use `navigator.mediaSession`, Fullscreen API, `window.open`, and anchor downloads as replacements.
- No TV remote/focus layer.

---

## 7. Code conventions

- **TypeScript strict.** Avoid `any`; if unavoidable, isolate it and explain why.
- **Svelte 5 idioms.** Prefer runes (`$state`, `$derived`, `$effect`) and snippets. Avoid legacy `export let`/`$:` patterns in new code.
- **Naming.** `PascalCase.svelte` for components, `camelCase.ts` for modules, one public component per file.
- **Imports.** Use the `$lib` alias; keep feature-internal imports relative. No barrel imports for icons or heavy modules.
- **State.** Keep state local. Promote to a store only when shared across siblings/routes. Services (session, player, query client) are explicit modules, not globals.
- **Errors.** User-visible failures get a toast/inline error with a translated message; log technical context with `console.error` and the feature name.
- **CSS.** Component-scoped styles + tokens. No global CSS except resets/tokens. No `!important` unless documented.
- **Comments.** Explain *why* (constraints, server quirks), not *what*.
- **Dead code.** Delete it; do not leave commented-out implementations.
- **Docs.** If you change a decision, update `rewrite-plan.md` and add a short ADR in `docs/adr/`.

---

## 8. Testing and definition of done

Every change should leave the app in a state where:

- [ ] Type check passes.
- [ ] Lint passes.
- [ ] Relevant unit/component tests pass.
- [ ] New behavior has a test (or a documented reason it cannot be unit-tested).
- [ ] Keyboard navigation and focus behavior still work.
- [ ] Strings are translated or added to the catalog.
- [ ] No new heavy dependency or large initial chunk.
- [ ] The relevant `rewrite-plan.md` parity checklist item is updated if the screen/feature is now complete.
- [ ] No hard constraint from §2 was violated.

Testing layers:

- **Unit:** pure utilities (sort/filter mapping, playback method decisions, date/space formatting, query keys).
- **Component:** `@testing-library/svelte` for forms, dialogs, cards, tabs.
- **Integration:** MSW for API/WebSocket flows.
- **E2E:** Playwright against a real Jellyfin server (login, browse, play, admin CRUD, wizard).
- **Visual:** snapshots of the new design at mobile/tablet/desktop; never diff against the old app.
- **Accessibility:** axe in CI and keyboard-only walkthroughs.

---

## 9. Agent workflow

1. **Locate the plan section.** Find the screen/feature in §3 and its parity rows in §6.
2. **Read the reference implementation.** Use the current source to learn behavior, not architecture.
3. **Check decisions.** §9.1 lists resolved decisions; do not reopen them. If a new decision is needed, record it in §9.2 and `docs/adr/`.
4. **Make the smallest coherent change.** Do not refactor unrelated code in the same task.
5. **Verify.** Run type check, lint, and relevant tests; smoke-test affected routes.
6. **Update docs.** Parity checklist, plan, ADR, or AGENTS.md if conventions changed.
7. **Report.** List changed files, commands run, results, and any remaining gaps.

Additional rules:

- Do not delete the upstream `src/` reference tree until the corresponding parity is verified.
- Do not modify generated files (`package-lock.json`, build output) by hand.
- Do not add dependencies without checking the plan's stack and bundle impact.
- Do not run destructive git commands (`reset --hard`, `clean -fd`, force push) unless explicitly instructed.
- When a task is ambiguous, ask; do not guess product behavior.

---

## 10. Common pitfalls

- **SvelteKit + hash routing.** SvelteKit's router is History-API based; do not try to make it produce `/#/...` URLs. Keep the hash router in the root page.
- **SSR.** Set `ssr = false` for the SPA. Still guard direct `window`/`document` access in modules that might run during build/prerender.
- **Base path.** Use `paths.relative` and test the build under `/web/` and a non-root base; do not hard-code `/web/`.
- **Deep links.** Always verify `/#/details?id=...`, `#!`, and `#!/` forms.
- **API 401.** Cancel/evict unauthorized query data instead of showing stale content.
- **Query cache.** Do not persist API responses that contain secrets; keep playback/session state volatile.
- **Large lists.** Paginate/virtualize library grids; do not render thousands of cards.
- **HLS/subtitles.** Keep player adapters lazy; test direct play, direct stream, remux, and transcode.
- **Plugin iframe.** A same-origin iframe with `allow-same-origin` is not a security boundary; treat plugin pages as trusted and validate `event.origin`.
- **Translation keys.** Reuse the existing catalogs; run a key-diff if new keys are added.
- **Icons.** Never `import * from 'lucide-svelte'`; import named icons individually.
- **Server version.** Document and enforce the minimum server version; gate features on capabilities.

---

## 11. References

- Plan: `rewrite-plan.md`
  - §3 screen/feature inventory
  - §4 architecture
  - §6 parity checklist
  - §7 delivery phases
  - §9 decision briefs and resolved decisions
  - Appendix E native-shell removal
- Current app entry points: `src/RootApp.tsx`, `src/RootAppRouter.tsx`, `src/apps/*/routes/*`
- Current playback: `src/components/playback/playbackmanager.js`, `src/plugins/*`
- Current admin: `src/apps/dashboard/**`
- Translations: `src/strings/*.json`
- Themes: `src/themes/**`
- Decision records: `docs/adr/` (create when the first decision needs recording)

When in doubt: behavior comes from the current app, architecture comes from the plan, and the user decides scope.
