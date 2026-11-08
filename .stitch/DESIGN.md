---
name: Jellyfin Web
colors:
  surface: '#202020'
  surface-dim: '#101010'
  surface-bright: '#303030'
  surface-container-lowest: '#101010'
  surface-container-low: '#1a1a1a'
  surface-container: '#202020'
  surface-container-high: '#282828'
  surface-container-highest: '#303030'
  on-surface: '#ffffff'
  on-surface-variant: 'rgba(255, 255, 255, 0.7)'
  inverse-surface: '#f2f2f2'
  inverse-on-surface: '#101010'
  outline: 'rgba(255, 255, 255, 0.12)'
  outline-variant: 'rgba(255, 255, 255, 0.05)'
  surface-tint: '#00a4dc'
  primary: '#00a4dc'
  on-primary: 'rgba(0, 0, 0, 0.87)'
  primary-container: '#00729a'
  on-primary-container: '#ffffff'
  inverse-primary: '#33b6e3'
  secondary: '#00a4dc'
  on-secondary: 'rgba(0, 0, 0, 0.87)'
  secondary-container: '#00455c'
  on-secondary-container: '#ffffff'
  tertiary: '#f2b01e'
  on-tertiary: '#000000'
  tertiary-container: '#4a3600'
  on-tertiary-container: '#ffd270'
  error: '#c62828'
  on-error: '#ffffff'
  error-container: '#8a1c1c'
  on-error-container: '#ffdad6'
  primary-fixed: '#33b6e3'
  primary-fixed-dim: '#00a4dc'
  on-primary-fixed: '#000000'
  on-primary-fixed-variant: '#003549'
  secondary-fixed: '#33b6e3'
  secondary-fixed-dim: '#00a4dc'
  on-secondary-fixed: '#000000'
  on-secondary-fixed-variant: '#003549'
  tertiary-fixed: '#ffd270'
  tertiary-fixed-dim: '#f2b01e'
  on-tertiary-fixed: '#000000'
  on-tertiary-fixed-variant: '#4a3600'
  background: '#101010'
  on-background: '#ffffff'
  surface-variant: '#242424'
typography:
  display-lg:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 28px
    fontWeight: '700'
    lineHeight: 38px
    letterSpacing: '-0.01em'
  headline-md:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 24px
    fontWeight: '600'
    lineHeight: 32px
    letterSpacing: '0'
  title-md:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 18px
    fontWeight: '600'
    lineHeight: 25px
    letterSpacing: '0'
  body-base:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 15px
    fontWeight: '400'
    lineHeight: 21px
    letterSpacing: '0'
  body-bold:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 15px
    fontWeight: '600'
    lineHeight: 21px
    letterSpacing: '0'
  label-caps:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 13px
    fontWeight: '600'
    lineHeight: 18px
    letterSpacing: '0.04em'
  caption:
    fontFamily: '"Noto Sans", sans-serif'
    fontSize: 13px
    fontWeight: '400'
    lineHeight: 18px
    letterSpacing: '0'
rounded:
  sm: 0.2em
  DEFAULT: 0.2em
  md: 0.25em
  lg: 0.5em
  xl: 0.7em
  full: 9999px
spacing:
  unit: 4px
  xs: 4px
  sm: 8px
  md: 16px
  lg: 24px
  xl: 32px
  card-gap: 0.6em
  bottom-pad: 5em
---

# Design System: Jellyfin Web

## 1. Visual Theme & Atmosphere

Jellyfin Web delivers a premier, immersive personal home-theater experience engineered to celebrate visual media. Anchored in deep carbon black (`#101010`) and warm dark gray surfaces (`#202020`), the interface deliberately fades into the background so vibrant movie posters, television series art, and high-definition backdrop photography command center stage. An energetic electric cyan accent (`#00a4dc`) provides immediate focal clarity, guiding user interaction across desktop monitors, mobile touchscreens, and expansive 10-foot television displays without introducing visual noise.

The visual atmosphere balances modern streaming elegance with the utilitarian speed of a responsive media browser. Structural elements prioritize crisp boundaries and disciplined information density: cards maintain subtle, intentional corner rounding (`0.2em`), floating navigation surfaces utilize translucent black overlays and backdrop gradients, and interactive elements rely on physical scale cues (`transform: scale(1.07)` to `scale(1.3)`) rather than garish shadows to deliver responsive feedback. Whether browsing an expansive movie archive or controlling playback with the persistent audio bar, the experience feels cohesive, fluid, and cinematic.

---

## 2. Color Palette & Roles

The color architecture is grouped functionally into foundation, interactive, typographic, and status roles, centered around the flagship Dark theme while honoring Jellyfin's multi-theme flexibility.

### Primary Foundation
- **Deep Carbon Black (`#101010`)** — Root background (`$background-default`); creates infinite depth on OLED displays and recedes behind media backdrops.
- **Charcoal Surface (`#202020`)** — Primary surface layer (`$background-paper`, `$appBar-defaultBg`); used for app bars, navigation drawers, card backgrounds, and flyout menus.
- **Elevated Charcoal (`#242424` / `#282828`)** — Card padder containers, visual card boxes, and elevated modals.
- **Raised Snackbar (`#303030`)** — Toast notifications, snackbar toasts, and high-elevation dialog overlays.
- **Translucent Obsidian (`rgba(0, 0, 0, 0.4)`)** — Semi-transparent app bar and detail ribbon backdrop overlays (`$appBar-transparentBg`).
- **Dimming Mask (`rgba(0, 0, 0, 0.3)` to `rgba(0, 0, 0, 0.7)`)** — Touch navigation masks, dialog backdrops, and inner card metadata footer overlays.

### Accent & Interactive
- **Jellyfin Electric Cyan (`#00a4dc`)** — Brand primary accent (`$primary-main`, `$secondary-main`); used for primary action buttons (`.button-submit`), focus rings, progress bars, active tab states, and selected navigation options.
- **Deep Marine Cyan (`#00729a`)** — Primary dark variant (`$primary-dark`); applied during hover and active states on primary action triggers.
- **Vibrant Sky Cyan (`#33b6e3`)** — Primary light variant (`$primary-light`); used for highlighting, active scrubber heads, and high-visibility badges.
- **Cyan Sheer Hover (`rgba(0, 164, 220, 0.2)`)** — Soft hover tint for icon buttons, menu options, and interactive card headers.
- **Neutral Contained Surface (`#424242`)** — Secondary neutral buttons (`.raised`, `.fab`); shifts to `#616161` on hover.
- **Streaming Media Blue (`rgb(51, 136, 204)`)** — Disc, media source, and audio badge pill backgrounds.

### Typography & Text Hierarchy
- **Pure White (`#ffffff`)** — Primary text (`$text-primary`); high-contrast headers, active media titles, button labels, and selected tabs.
- **Soft Muted Slate (`rgba(255, 255, 255, 0.7)`)** — Secondary text (`$text-secondary`); episode details, artist metadata, timestamps, unselected tabs, and form field descriptions.
- **Subtle Hairline (`rgba(255, 255, 255, 0.12)`)** — Structural dividers (`$divider`), collapsible separators, and unfocused border lines.
- **Dark Neutral Text (`rgba(0, 0, 0, 0.87)`)** — Contrast text on top of electric cyan buttons and light theme text.

### Functional States
- **Crimson Error / Watched Fill (`#c62828`)** — Deletion confirmation actions (`.button-delete`), error alert badges, and recording warnings.
- **Coral Rose Indicator (`#d15353`)** — Active recording progress bars and heart rating icons (`.ratingbutton-icon-withrating`).
- **Amber Gold (`#f2b01e`)** — Favorite star icons and rating highlights.
- **Info Ocean (`#0288d1` / `#039be5`)** — Informational notification banners and kids programming guide cells.
- **Live Guide Category Accents:**
  - Sports: **Royal Indigo (`#3949ab`)**
  - Movies: **Deep Amethyst (`#5e35b1`)**
  - News: **Forest Green (`#43a047`)**
  - Active Channel: **Graphite (`#1e1e1e`)**

---

## 3. Typography Rules

Jellyfin Web employs **Noto Sans** as its authoritative primary typographic voice, delivering broad multilingual coverage, geometric clarity, and comfortable legibility across viewing distances.

### Hierarchy & Weights
- **Display / H1 (`1.8rem` / `~28px` — Bold `700` or Regular `400`)** — Page titles, hero collection headers, album/movie titles in detail ribbons. Line-height `1.35`.
- **Headline / H2 (`1.5rem` / `~24px` — SemiBold `600` or Regular `400`)** — Section shelf headers ("Next Up", "Latest Movies", "Recently Added TV").
- **Subheading / H3 (`1.17rem` / `~18px` — SemiBold `600`)** — Modal titles, card section subheadings, settings section titles.
- **Body Regular (`93%` baseline / `~15px` — Regular `400`)** — Media descriptions, plot overviews, settings labels, list item descriptions. Line-height `1.35`.
- **Button / Interactive (`15px` — SemiBold `600`)** — Action buttons, tab labels, quick filters. Text transform is strictly set to `none` (no uppercase forced).
- **Secondary / Card Subtitle (`86%` baseline / `~13px` — Regular `400`)** — Release years, parental ratings, runtime durations, episode subtitle titles. Single-line ellipsis truncation.
- **Monospace (`ui-monospace`, `Menlo`, `Monaco`, `Cascadia Mono`)** — Subtitle offsets, log viewer, JSON configuration editors.

### Device-Specific Scale Adaptations
- **10-Foot Television Layout (`.layout-tv`)** — All typography is magnified by `125%` baseline (minimum `20px`) to comply with lean-back television ergonomics (Tizen/webOS compliance).
- **Mobile Handheld Layout (`.layout-mobile`)** — Baseline is scaled to `90%` to preserve horizontal information density on compact touch screens.

### Spacing Principles
- Letter-spacing maintains neutral tracking (`0` to `-0.01em`) on large display headers to keep multi-word media titles compact and readable.
- Small indicators, media flags, and badges apply subtle positive tracking (`0.04em`) for clear optical distinction against card imagery.
- Line heights remain comfortably compact (`1.25` - `1.35`) so text containers never compromise vertical thumbnail budgets.

---

## 4. Component Stylings

### Buttons
- **Shape & Geometry:** Subtle corner radius (`0.2em` / `~3-4px`), padding `0.9em 1em`, inline-flex alignment.
- **Primary CTA (`.button-submit`):** Jellyfin Electric Cyan (`#00a4dc`) background with high-contrast text (`rgba(0, 0, 0, 0.87)`). Hover transitions smoothly to `#00729a`.
- **Secondary Contained (`.raised` / `a[data-role="button"]`):** Neutral gray background (`#424242`) with secondary text color; hovers to `#616161`.
- **Icon Buttons (`.paper-icon-button-light`):** Circular bounding box, transparent background, margin `0 0.29em`. Hover triggers a soft cyan glow (`rgba(0, 164, 220, 0.2)`).
- **Floating Action Buttons (`.fab`):** Circular `50%` radius, padding `0.6em`, elevated with ambient drop-shadow.
- **Focus Dynamics:** TV/keyboard focus triggers an authoritative physical zoom (`transform: scale(1.2)` for standard buttons, `scale(1.3)` for tab buttons).

### Cards & Media Containers
- **Corner Radius:** Standardized at `0.2em` (`--jf-card-borderRadius: 0.2em`) across posters, backdrop thumbnails, square covers, and blurhash placeholders.
- **Aspect Ratios:**
  - **Portrait / Poster:** `padding-bottom: 150%` (2:3 aspect ratio for movies and series).
  - **Backdrop / Landscape:** `padding-bottom: 56.25%` (16:9 aspect ratio for episodes and video clips).
  - **Square:** `padding-bottom: 100%` (1:1 aspect ratio for music albums, artists, and live folders).
  - **Banner:** `padding-bottom: 18.5%` (wide horizontal banners).
- **Surface & Shadow:** Non-image placeholders and visual cards utilize dark elevation (`#242424`) with subtle drop shadow: `0 0.0725em 0.29em 0 rgba(0, 0, 0, 0.37)`.
- **TV & Keyboard Focus Treatment:** Hover/focus zooms cards smoothly (`transform: scale(1.07, 1.07)` with `200ms ease-out` transition), accompanied by an electric cyan focus ring (`0.5em` border width, `border-radius: calc(0.2em + 0.5em)`).
- **Person Cards:** Circular option (`50%` border-radius and `circle(50% at 50% 50%)` clip-path) for actors, directors, and creators.

### Navigation
- **Top Skin Header (`.skinHeader`):** Fixed to top with safe area insets (`env(safe-area-inset-top)`). Features a smooth transition from fully transparent to semi-transparent (`rgba(0, 0, 0, 0.4)`) or solid `#202020` on scroll.
- **Slide-Out Nav Drawer (`.mainDrawer` / `.touch-menu-la`):** Solid charcoal background (`#101010` / `#202020`), soft right drop shadow (`2px 0 12px rgba(0, 0, 0, 0.4)`), smooth transform animations (`40ms` - `260ms`). Active items display cyan backgrounds (`#00a4dc`) with dark contrast text.
- **Section Tabs (`.emby-tab-button`):** Horizontal scrollable strip; active tab displays pure white text and an active indicator; hover/focus switches to cyan.

### Inputs & Forms
- **Fields (`.emby-input`, `.emby-textarea`, `.emby-select`):** Dark translucent fill (`rgba(255, 255, 255, 0.09)`), subtle border (`0.16em solid transparent`), corner radius `0.2em`. Minimum `16px` font size on mobile to prevent iOS viewport auto-zoom.
- **Focus State:** Border transitions to solid electric cyan (`#00a4dc`); associated label highlights to cyan.
- **Checkboxes & Radios:** Hairline circular/square outlines that fill with electric cyan upon selection.

### Domain-Specific Components
- **Now Playing Bar (`.nowPlayingBar`):** Persistent bottom media deck (`height: 4.2em`), positioned above content and fixed at bottom with `padding-bottom: calc(env(safe-area-inset-bottom))`. Contains album/poster thumbnail, song/movie title, artist/show metadata, center playback controls, and volume/queue actions. Top edge features a continuous thin cyan progress bar (`.nowPlayingBarPositionContainer`).
- **Video On-Screen Display (OSD):** Gradient scrims fading top-to-bottom (`linear-gradient(180deg, rgba(16, 16, 16, 0.75) 0%, transparent 100%)`) and bottom-to-top (`linear-gradient(0deg, rgba(16, 16, 16, 0.75) 0%, transparent 100%)`), housing chapter scrubbers, subtitles, audio track pickers, and playback controls.
- **Media Badges & Indicators (`.playedIndicator`, `.countIndicator`):** Circular `2em x 2em` badges pinned to the top-right corner of cards with cyan background (`#00a4dc`), pure white checkmark/count, and ambient drop shadow.
- **EPG Live TV Guide:** Dense timeline grid with sticky channel headers on the left and horizontal time slot headers on top. Cells colored by genre category for instant visual scanning.

---

## 5. Layout Principles

### Grid & Structure
- **Fluid Multi-Row Media Shelf:** Horizontal carousels and vertical flex grids (`.itemsContainer`, `.vertical-wrap`).
- **Card Spacing:** Standard gap margin of `0.6em` between card tiles (`0.4em` in dense/focused viewports), and `1.8em` margin below rows (`1.2em` on mobile screens).
- **Responsive Breakpoint Scale:**
  - `31.25em` (~500px): Small phone landscape / compact tablet portrait.
  - `43.75em` (~700px): Phablets and small tablets.
  - `50.00em` (~800px): Tablets and small laptops (desktop navigation switch).
  - `75.00em` (~1200px): Desktop monitors (3-4 backdrop cards or 6-7 portrait cards per row).
  - `87.50em` (~1400px): Widescreen displays.
  - `100.00em` (~1600px): Ultra-wide monitors.
  - `120.00em` (~1920px): Full HD 1080p TV layout.
  - `131.25em` (~2100px) & `156.25em` (~2500px): 2K and 4K UHD viewing environments.

### Whitespace Strategy
- **Persistent Bottom Inset:** All scrollable page views mandate a `5em` bottom padding (`calc(env(safe-area-inset-bottom) + 5em)`) to ensure content is never obscured by the persistent music player bar.
- **Reading & Form Constraints:** Settings forms and text-heavy detail blocks are strictly capped at `max-width: 54em` (~864px) to preserve ergonomic line length.
- **Safe Area Insets:** Systematic integration of `env(safe-area-inset-*)` across top headers, left/right drawer edges, and bottom playback bars for notch and edge-gesture safety.

### Alignment & Visual Balance
- Left-aligned text hierarchy with full bidirectional RTL (`[dir="rtl"]`) support, mirroring padding, margins, drawer alignments, and icons.
- Content shelves prioritize full bleed horizontal scrolling (`.emby-scroller`) paired with left/right scroll buttons on desktop and swipe velocity on touch devices.

---

## 6. Design System Notes for Stitch Generation

### Language to Use
- **Atmospheric Keywords:** Cinematic, dark-mode-first, media-centric, immersive backdrop, electric cyan accents, crisp micro-radii, 10-foot television ergonomics, lean-back media center, clean metadata hierarchy.
- **Avoid:** Generic neon gradients, oversized bubbly radii (`> 8px` on cards), heavy skeuomorphic drop shadows, pastel backgrounds, cluttered multi-colored CTA groups.

### Color References
- **Root Background:** `#101010` (Deep Carbon Black)
- **Primary Surface / App Bar:** `#202020` (Charcoal Surface)
- **Elevated Container / Card Box:** `#242424` (Elevated Charcoal)
- **Toast / Floating Overlay:** `#303030` (Raised Charcoal)
- **Primary CTA / Focus Accent:** `#00a4dc` (Jellyfin Electric Cyan)
- **Primary CTA Hover:** `#00729a` (Deep Marine Cyan)
- **High-Visibility Highlight:** `#33b6e3` (Vibrant Sky Cyan)
- **Favorite / Star:** `#f2b01e` (Amber Gold)
- **Error / Deletion / Played Check:** `#c62828` (Crimson Error)
- **Watched / Active Recording:** `#d15353` (Coral Rose)
- **Primary Text:** `#ffffff` (Pure White)
- **Secondary Text:** `rgba(255, 255, 255, 0.7)` (Soft Muted Slate)
- **Border / Divider:** `rgba(255, 255, 255, 0.12)` (Subtle Hairline)

### Component Prompts
- **Media Card Shelf:**
  > "Dark cinematic media shelf on #101010 background. Section title 'Next Up' in 24px Noto Sans white. Horizontal row of 16:9 backdrop cards with 0.2em subtle rounded corners, dark #242424 placeholders, subtle drop shadow, poster artwork filling container, and a 2em circular #00a4dc played indicator badge with a white checkmark in the upper-right corner. Two lines of metadata underneath: bold white title and muted 70% opacity episode subtitle."
- **Persistent Now Playing Deck:**
  > "Fixed bottom audio playback bar with #202020 charcoal background and a hairline 1px divider. A 2px #00a4dc cyan progress bar runs across the very top edge. Left side displays a 48px square album thumbnail with track title in white and artist in 70% muted gray. Center displays circular media buttons (shuffle, previous, play/pause fab in electric cyan #00a4dc, next, repeat). Right side displays volume slider and playlist queue button."
- **Video Player OSD:**
  > "Video player controls overlay with deep charcoal gradient scrims at top and bottom over paused movie frame. Top bar displays back arrow, movie title 'Interstellar (2014)', and stream quality badge. Bottom bar features a full-width cyan timeline scrubber with chapter markers, elapsed time '1:14:22 / 2:49:00', large center play button, forward/rewind 10s buttons, audio track selector chip, and subtitle picker."

### Incremental Iteration
- When designing new screens in Stitch for Jellyfin, start with dark `#101010` canvas and build elevation upward using `#202020` and `#242424`.
- Maintain the strict `0.2em` border radius on cards and input controls to ensure consistency with existing Jellyfin Web UI.
- Always allocate `5em` (`80px`) of clear bottom margin to avoid overlapping with the persistent playback bar.
