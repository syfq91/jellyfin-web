# Jellyfin Web UI — Lightweight Rewrite Plan

> Status: Draft v5 — modern-only, browser-only, Svelte 5 + SvelteKit, refreshed visual
> Scope: Full **behavioral** feature parity with the current `jellyfin-web` **modern layout**, implemented as a lightweight **browser-only** alternative client. Legacy layout, layout switching, TV webview targets, the native-shell/wrapper bridge, and the React/MUI stack are dropped. Stack is **Svelte 5 + SvelteKit**, with **Lucide** icons. Visual design is intentionally refreshed rather than pixel-copied.
> Analysis target: `/home/syafiq/code/jellyfin-web` (`jellyfin-web` 13.0.0)

---

## 0. TL;DR

The current web client is a very large React 18 + MUI 6 application that still carries a complete legacy jQuery/web-component client, a legacy view/controller system, two layout modes, a full admin dashboard, a first-run wizard, a full media player stack, Live TV/DVR, e-book/comic/PDF readers, SyncPlay, Chromecast, remote control, metadata/image/subtitle/lyrics editors, and 109 translation catalogs.

The rewrite should:

1. **Ship one modern responsive app** — modern layout only, browser-only. Delete the legacy layout/app, the `viewManager`/controller runtime, the layout switcher, all TV webview targets, and the `window.NativeShell` bridge. *(Resolved: legacy + TV + native shell dropped, modern forced.)*
2. **Keep the Jellyfin server API contract unchanged** — this app is a drop-in client, not a server change.
3. **Replace the React/MUI stack** with **Svelte 5 + SvelteKit** (SvelteKit SPA build; see §9.2.1). Drop jQuery, jstree, webcomponents.js, swiper, headroom, polyfill bloat, MUI/Emotion, and the `jellyfin-apiclient` legacy transport in favor of `@jellyfin/sdk` + a thin session manager. Use **Lucide** (`lucide-svelte`) for icons instead of the Material icon font.
4. **Reimplement every in-scope screen and feature listed in §3**, including admin and wizard. Legacy-layout screens are removed, but the modern app's embedded legacy pages (details, list, lyrics, queue, preferences, metadata editor, Live TV guide/tuner) are rebuilt in the new framework.
5. **Refresh the visual design** rather than pixel-copying the current app; parity is behavioral (see §9.2.5).
6. Target **< 500 KB gzip initial payload (JS + CSS combined)** with per-chunk budgets in §4.9, and route-level code splitting (Svelte makes this substantially easier than the current React/MUI build).
7. **Render server plugin configuration pages in a sandboxed iframe** *(resolved decision)*.

---

## 1. Goals and non-goals

### Goals

- **Behavioral feature parity** with all **in-scope** screens listed in §3: public/auth, user media browsing, item details, playback, user preferences, admin dashboard, setup wizard, Live TV/DVR, readers, editors, SyncPlay/remote/cast.
- **Modern layout only**: one responsive layout for desktop, tablet, and mobile. No `layoutManager`, no `desktop-legacy`/`mobile-legacy`/`tv` modes, no layout selector in preferences.
- **Drop-in deployment**: must work when served by the Jellyfin server under the existing web root (`/web/`), use hash routing (`/#/home`), preserve deep links and legacy `#!` bookmarks, and read `config.json` like today.
- **Lightweight framework**: Svelte 5 + SvelteKit + TypeScript; no React, MUI, Emotion, jQuery, or web components. Icons use Lucide (`lucide-svelte`), tree-shaken per icon. Aggressive code splitting; lazy-load heavy features (readers, PDF, transcoding UI, admin pages).
- **One data and styling model**: TanStack Query (Svelte adapter) for server state, Svelte runes/stores for client state, design tokens + component-scoped CSS for styling.
- **Refreshed visual design**: keep information architecture and behavior, update the visual language (see §9.2.5).
- **Accessible**: keyboard-first navigation, ARIA, focus management, reduced motion (no TV-remote layer).
- **Localized**: reuse the existing 109 locale catalogs (`src/strings/*.json`, 1,949 keys in `en-us.json`) and RTL support.

### Non-goals (for v1)

- Changing the Jellyfin server, its auth model, or its API.
- **Legacy layout/app**: `desktop-legacy`, `mobile-legacy`, TV layout, the legacy `viewManager`/controller runtime, and the legacy-only routes in `apps/legacy/routes` (`/home`, `/movies`, `/tv`, `/music`, `/livetv` legacy variants). Their *features* are ported to the new framework; the legacy runtime is not.
- **TV webview targets**: Tizen, webOS/Orsay, PS4/Xbox, and other TV browsers. No TV remote/focus layer and no TV-specific home-section behavior.
- **React, MUI, Emotion, and the current component stack** — replaced, not wrapped.
- **Pixel-for-pixel visual parity**: behavior parity is the acceptance criterion; the visual language is refreshed.
- Reimplementing server-side plugin UI in the client framework. Plugin configuration pages are server-served HTML and render in a sandboxed iframe route **(resolved)**.
- **Native-shell/wrapper support in any form** — no `window.NativeShell`, no Android/iOS bridge, no native download manager/client settings/device profile/plugins. The rewrite is browser-only (§9.2.3).

### Decision status

**Resolved**

- Legacy layout and TV webview: **dropped**.
- Layout switching: **removed** — modern layout is forced.
- UI framework: **Svelte 5 + SvelteKit** (accepted).
- Visual design: **refreshed**, not pixel-copied (§9.2.5).
- Downloads: **download only, no offline mode** (§9.2.4).
- Plugin configuration UI: **sandboxed iframe + typed `postMessage` bridge** (§9.2.6).
- Browser baseline: **evergreen** — last 2 Chrome/Edge/Firefox, Firefox ESR, Safari 16+, iOS 16+ (§9.2.2).
- Native-shell hooks: **removed — browser-only**; no `window.NativeShell` bridge (§9.2.3).
- PWA: **static app-shell service worker, post-parity** (§9.2.7).
- Server floor: **current server line matching the SDK** (§9.2.8).

No product or stack decisions remain open. Phase 0 starts directly with the SvelteKit scaffold and the design tokens.

---

## 2. How the current app works

### 2.1 Runtime composition

`src/RootApp.tsx` mounts:

- `PersistQueryClientProvider` → TanStack Query with IndexedDB persistence (`src/utils/query/queryClient.ts`, 24 h GC, 60 s stale, 401 eviction).
- `ApiProvider` → current `@jellyfin/sdk` `Api` + `serverId` + `user` (`src/hooks/useApi.tsx`), fed by `jellyfin-apiclient` `ServerConnections`.
- `UserSettingsProvider`, `WebConfigProvider`.
- `RootAppRouter` → `createHashRouter` (`src/RootAppRouter.tsx`).

Route composition (`src/RootAppRouter.tsx`):

```
RootAppLayout
├── modern OR legacy app routes   (chosen by layoutManager.modern)
├── dashboard app routes
├── wizard routes
└── !/*  → BangRedirect (legacy hashbang links)
```

`layoutManager` (`src/components/layoutManager.js`):

- Browser default layout = `modern` (`getDefaultLayout()` in `src/components/apphost.js`).
- Legacy modes = `desktop-legacy`, `mobile-legacy`, `tv` (`src/constants/layoutMode.ts`).
- Users can pick Auto / Desktop / Mobile / TV in Display preferences; native shells can override.

**Rewrite scope note:** the modern route tree is the only routing tree carried forward. The legacy modes above, `layoutManager`, and the layout selector are removed. The legacy *pages embedded by the modern app* (details, list, lyrics, queue, preferences, metadata/guide/tuner) remain in scope as features and are rebuilt in the new framework.

### 2.2 Routing model

Two route kinds:

- **Async routes** — lazy React page components (`components/router/AsyncRoute.tsx`), e.g. dashboard pages, modern library pages, search.
- **Legacy routes** — `viewManager` loads a controller JS + HTML template pair from `src/apps/{legacy,dashboard,wizard}/controllers/` and renders into the page (`components/router/LegacyRoute.tsx`, `viewManager/ViewManagerPage.tsx`). These are the old jQuery pages.

`ConnectionRequired` gates routes at `public`, `user`, `admin`, and `wizard` levels.

**Rewrite scope note:** the rewrite keeps neither route mechanism. Every route becomes a lazy Svelte route component; the legacy controller/view pairs are reference implementations only. The viewManager runtime, jQuery-based page lifecycle, and controller/HTML pair loading are deleted. The modern `/details`, `/list`, `/lyrics`, `/queue`, `/mypreferences*`, `/metadata`, and admin guide/tuner pages are rebuilt with the same features.

### 2.3 API/data layer

- `@jellyfin/sdk` (generated client + axios) for most modern code.
- `jellyfin-apiclient` (`src/lib/jellyfin-apiclient/ServerConnections.js`, `connectionManager.js`) for connection/session/WebSocket and legacy code paths.
- React Query hooks live next to features (`src/apps/*/features/**/api`, `hooks/api`).
- WebSocket events (`OutboundWebSocketMessageType`) invalidate queries (`components/QueryClientEventHandler.tsx`); server restart/shutdown clears the cache.

### 2.4 Styling and themes

- MUI 6 + Emotion for modern pages.
- SCSS + CSS variables for legacy pages (`src/styles`, `src/themes`).
- Themes: `appletv`, `blueradiance`, `catppuccin-*` (5 variants), `dark`, `light`, `purplehaze`, `wmc` (`src/themes`).
- Server branding CSS + user/machine custom CSS are injected (`components/CustomCss.tsx`).

### 2.5 Build

- Webpack 5 production/dev configs; Vite only for tests (`vite.config.ts`).
- Dynamic `import()` per route and per heavy component; workers for image/metadata tasks.
- PWA manifest (`src/manifest.json`), no service worker in the repo.

---

## 3. Complete screen and feature inventory

Legend: **P0** = needed for a usable client, **P1** = parity-critical, **P2** = advanced/long-tail.

### 3.1 Public / authentication / startup

| Route | Screen | Features | Phase |
|---|---|---|---|
| `/selectserver` | Server selection | Saved server list, connect, add server, delete, server status/version, auto-redirect if single server | P0 |
| `/addserver` | Connect to server | Host/URL input, validation, error states, remember server | P0 |
| `/login` | Login | Visual user picker (avatars, public users), manual username/password login, "remember me", Quick Connect, forgot password, change server, login disclaimer (server branding), sign in error handling | P0 |
| `/quickconnect` | Quick Connect | Initiate code, poll/authorize state, cancel, success/failure, deep-link from login | P1 |
| `/forgotpassword` | Forgot password | Launch reset flow / instructions (async page) | P1 |
| `/forgotpasswordpin` | Reset with PIN | PIN + new password form | P1 |
| startup | Connection gate | `ConnectionRequired` levels, server capability detection, forced logout, server unreachable/retry (`ConnectionErrorPage`), "server restarting/shutting down" handling, first-run redirect to wizard | P0 |

Notes:

- Login must support auth providers/password-reset providers returned by the server (`/Auth/Providers`, `/Auth/PasswordResetProviders`).
- Multi-server switching currently has a native-shell path (`window.NativeShell.selectServer`) and a web path (`/selectserver`, saved servers). The rewrite keeps the web path and drops the native path.

### 3.2 App shell (both layouts)

| Area | Features |
|---|---|
| App toolbar | Drawer/menu button, back button (browser history), server button, user-view navigation (Favorites + libraries + custom links + overflow menu), SyncPlay button/menu, Remote Play button/menu, Search button, user menu |
| Drawer | Header, Home, Favorites, custom server menu links (icon/name/url), library list with type icons, admin links when admin |
| User menu | Profile, Settings, Select server (web flow), Sign out |
| Now Playing bar | Artwork/title, play/pause, prev/next track, stop, seek progress, volume/mute, shuffle, repeat, Lyrics, AirPlay, overflow menu; opens full queue/remote control; theme song/video support |
| Header/backdrops | Per-page backdrop, transparency, logo/title, scroll behavior |
| Theme | Theme selection, custom CSS (user + server), dark/light, dashboard theme, faster animations, blurhash toggle |
| Screensavers | Disable, logo screensaver, backdrop screensaver + interval/timeout, slideshow interval |
| Keyboard/accessibility | Focus management, keyboard shortcuts, ARIA, reduced motion — **no TV remote/focus layer** (dropped) |
| Dialogs | Confirm, prompt/input, alert, action sheet, toast, loading/backdrop, error boundary pages, 404 fallback |

### 3.3 Library / browsing screens

Modern routes (`src/apps/modern/routes/asyncRoutes/user.ts`). Legacy app has parallel implementations listed at the end of this section.

| Route | Tab set (in order) | Notes |
|---|---|---|
| `/home` | Home, Favorites | Home sections are user-configurable (below) |
| `/movies` | Movies, Suggestions, Favorites, Collections, Genres, Studios, Playlists | Play All / Shuffle / Queue / New collection |
| `/tv` | Series, Suggestions, Upcoming, Genres, Studios/Networks, Episodes, Collections, Playlists | Play All / Shuffle, upcoming episodes |
| `/music` | Albums, Suggestions, Album Artists, Artists, Playlists, Songs, Genres, Collections | Play All / Shuffle, song list with artist/parent, add-to-list |
| `/books` | Folders, Books, Authors, Suggestions, Genres, Collections, Favorites | AudioBooks + Books |
| `/musicvideos` | Folders, Suggestions, Music Videos, Playlists | Play All / Shuffle |
| `/boxsets` | Collections, Favorites, Genres | New collection |
| `/playlists` | Playlists, Favorites | New playlist |
| `/homevideos` | Folders, Photos, Photo Albums, Videos | Photos/slideshow, play all |
| `/mixed` | Folders, Suggestions, Movies & Shows, Collections, Playlists | Mixed/unknown collection type |
| `/livetv` | Programs, Guide, Channels, Recordings, Schedule, Series (timers) | See Live TV §3.6 |
| `/list` | Generic list page | Legacy controller `list.js`; filters, sort, alphabet, paging |
| `/details` | Item detail page | See §3.4 |
| `/search` | Search | See §3.7 |
| `/video` | Video player | See §3.5 |
| `/queue` | Play queue / remote control | See §3.5 |
| `/lyrics` | Lyrics page | Synced/unsynced lyrics, auto-scroll |
| `/userprofile` | User profile | Avatar, name, password change, user info |

Shared library-page features:

- **Tabs/menu**: per-collection tab set above; tab state in URL (`?tab=`).
- **Views**: grid vs. list, card layouts (auto/portrait/square/banner/backdrop), image type (Primary/Banner/Disc/Logo/Thumb), show title/year, overlay text/play/more.
- **Sort**: per-view sort fields (name, random, community/critic rating, date added, date played, parental rating, play count, release date, runtime, etc.) and asc/desc.
- **Filters**: genres, years, official ratings, tags, studios, video types, series status, episode status, features, audio language, subtitle language (implemented as individual filter components under `apps/modern/features/libraries/components/filter`).
- **Alphabet picker** (A–Z/#), reset filters, result count, pagination (user page-size preference).
- **Play All / Shuffle / Queue**, create collection (permission-gated), create playlist.
- **Home sections** (`src/constants/homeSectionType.ts`): small library tiles, library buttons, active recordings, resume video, resume audio, resume books, latest media, next up, Live TV; up to 10 sections, user order + visibility. **TV-only extra-section logic is dropped.**
- **Empty states**: no libraries (admin link to create), no items, no collections/episodes messages.
- **Backdrops** per item type, theme songs/videos, blurhash placeholders, lazy images.
- **Suggestions sections**: continue watching, latest, next up, frequently/recently played, recommendations ("because you watched/like/directed by/starring").

**Out of scope: legacy layout screens.** The legacy app routes (`apps/legacy/routes/legacyRoutes/user.ts`) duplicate the library experience and are dropped, not ported:

- Legacy `/home`, `/movies`, `/tv`, `/music`, `/livetv` controller variants.
- Legacy `desktop-legacy` / `mobile-legacy` / `tv` presentation.
- `/favorites` is not a route in either app; favorites is a tab of Home and of each library. The legacy `favorites.js` controller currently backs the Home tab and is rebuilt in the new framework.

**Migration rule:** the modern app embeds legacy controller/view pages for `/details`, `/list`, `/lyrics`, `/queue`, `/mypreferences*`, plus the admin metadata manager and Live TV guide/tuner pages, and the Home tab controller. All of these are *features in scope*, but they are rebuilt in the new framework — the `viewManager` runtime is not carried over.

### 3.4 Item detail page (`/details`, legacy `itemDetails/index`)

One route handles all item types. Sections/actions to reproduce:

**Hero / actions**

- Backdrop, logo, poster/thumb, title, misc info (year, rating, runtime, genres), tagline, overview with show-more, tags, external links.
- Buttons: Resume, Play/Replay, Download, Trailer, Instant Mix, Shuffle, Cancel series timer, Stop recording, Played toggle, Favorite/rating, Split versions, More.
- Track selectors: media source, video, audio, subtitle.
- Recording fields (record/series record) and series recording editor.
- Context menu: play, play all from here, queue, play next, add to queue, add to playlist, add to collection, favorite, mark played, download, edit metadata, edit images, edit subtitles, edit lyrics, identify, refresh metadata, remove from collection/playlist, move to top/bottom (queue), copy stream URL, share, record/cancel recording, delete item/season/series/episode/media, view album/artist, view lyrics, media info, more media info.

**Type-specific sections**

| Item type | Sections |
|---|---|
| Movie | cast & crew, guest cast, special features, scenes/chapters, collections, more like this, trailer, media info, similar |
| Series | seasons/episodes (collapsible), next up, upcoming on TV/schedule, cast & crew, special features, collections, more like this |
| Season | episodes, cast & crew, more from season |
| Episode | cast & crew, guest cast, more from season, next/previous episode, media info |
| BoxSet/Collection | collection items, cast & crew |
| Playlist | tracks/list view, play all, shuffle |
| MusicAlbum | tracks, more from artist, lyrics |
| MusicArtist / AlbumArtist | albums, appears on, more from artist, music videos |
| Song/Audio | lyrics, media info, album/artist links |
| Book / AudioBook | reader launch, chapters, more from author |
| MusicVideo | more from artist |
| Video / HomeVideo | media info, additional parts |
| Photo / PhotoAlbum | slideshow, image navigation, download |
| Folder | children list |
| Person | birthday, birthplace, death date, appears in, movies/shows |
| Studio | items |
| Genre / MusicGenre | items |
| Recording / Program / Channel / SeriesTimer | schedule, record controls, program guide, channel info |
| Live TV program | record, series record, tune |

**Item detail data/UX**

- Backdrop/logo/primary/thumb/banner images + image loader.
- Metadata list (premiere date, rating, runtime, official rating, community/critic rating, original language, country, studio, etc.).
- Played/favorite state sync across tabs/devices.
- Download (direct file download; no native download manager).
- Split/merge versions.
- Identify/refresh metadata shortcuts (admin).
- Person editor (admin metadata editor).

### 3.5 Playback and media viewers

**Video player** (`/video`, `apps/legacy/controllers/playback/video`)

- OSD auto-hide, title/subtitle, media status ("fetching data"), current/end time, ends-at clock.
- Seek slider with buffered ranges and **trickplay thumbnail previews** (per-resolution), chapter markers.
- Buttons: record, previous/next chapter, rewind/fast-forward, play/pause, previous/next track, rating, subtitles, audio, mute/volume slider, settings, AirPlay, Picture-in-Picture, fullscreen.
- Keyboard shortcuts (space, F, M, J/L, PageUp/PageDown, Shift+P/N, etc.).
- Subtitle track selection (primary + **secondary subtitles**), subtitle offset sync, subtitle styling/burn-in, forced subtitles.
- Audio track selection, downmix, audio normalization preferences.
- **Media segments**: intro/outro/recap skip prompt, configurable action per segment type.
- **Next Up overlay** and autoplay next episode; still-watching prompt.
- SyncPlay group indicator and control; remote play/cast.
- Playback error handling/retry, direct play vs. transcode status (`playmethodhelper`).
- Theme media (songs/videos) playback outside the main item.

**Audio playback / Now Playing**

- Now Playing bar (see §3.2) and full-screen queue/remote page (`/queue`).
- Queue editing: reorder, move top/bottom, remove, clear, save as playlist, play next.
- Shuffle/repeat modes, prev/next, seek, volume, mute.
- Lyrics view (`/lyrics`) with auto-scroll.
- Audio-only "remote control" mode for other sessions.

**Media viewers (plugin architecture in current app, `src/plugins`)**

- HTML video/audio players (native + HLS/FLV).
- Book reader (EPUB, `epubjs`, TOC, bookmarks/progress).
- Comic reader (CBZ/CBR via libarchive).
- PDF reader (`pdfjs-dist`).
- Photo viewer + slideshow.
- Chromecast player.
- Session/remote player (playback on another session).
- Screensavers (logo, backdrop).

**Playback negotiation**

- `PlaybackInfo` request with device profile/capabilities; direct play / direct stream / remux / transcode decision.
- Quality settings: home network, internet, music, Chromecast bitrate, max resolution, limit supported resolution.
- Preferred transcode codecs (H.264/HEVC/AV1, AAC/AC3/ALAC/DTS/FLAC/Opus), fMP4 HLS preference, segment length limit.
- Progress reporting (`/Sessions/Playing`, `/Progress`, `/Stopped`), session capabilities, remote command handling.
- Cast/Chromecast discovery (browser only). Native external-player intent is dropped with the native-shell bridge.
- Subtitle burn-in handling, fallback fonts (server-side).

### 3.6 Live TV / DVR

User (modern `/livetv`, legacy `/livetv`, plus `/details` for programs/recordings):

- **Programs**: active programs, upcoming by category (episodes, movies, sports, kids, news).
- **Guide**: channel grid with time navigation, program detail popup, record/series record, tune.
- **Channels**: channel list/cards with current program.
- **Recordings**: latest recordings, recording folders, play/delete.
- **Schedule**: active recordings, upcoming recordings, cancel.
- **Series**: series timers list, edit/cancel, schedule.
- Program/recording detail pages: record, series record, cancel, channel/program info.
- Recording creator dialog: record/series record, channel, start/end, padding, folder, tags, pre/post.
- Tuner picker dialog when multiple tuners.

Admin Live TV (`/dashboard/livetv`, `/dashboard/livetv/recordings`, `/dashboard/livetv/tuner`, `/dashboard/livetv/guide` — the latter two are legacy controllers today and are rebuilt in the new framework):

- Tuner devices (add/delete, HDHomeRun/M3U/XMLTV/etc. providers).
- Guide providers (add/delete/refresh guide data, mapping).
- DVR settings: recording path, movie path, series path, pre/post padding, post-processor + arguments, guide days, save NFO, save recording images, metadata saving.

### 3.7 Search

- Search field with debounce, query in URL (`?query=`), optional `parentId` / `collectionType` scope.
- Empty-state **suggestions** (recent/popular).
- Result sections ordered: Movies, Shows, Episodes, People, Studios, Playlists, Artists, Albums, Songs, Videos, Programs, Channels, Photo Albums, Photos, Audio Books, Books, Collections (`sectionSortOrder.ts`).
- Live TV program search, people/studios search, video search, search suggestions.
- Search entry points: toolbar button, item context menu, cast/person clicks.
- Empty/no-results states.

### 3.8 User preferences (`/mypreferencesmenu`)

| Page | Route | Settings |
|---|---|---|
| Settings menu | `/mypreferencesmenu` | Profile, Quick Connect (if enabled), Display, Home, Playback, Subtitles, Controls (gamepad, non-mobile), Admin links (Manage Server, Manage Metadata), Select server (web flow), Sign out |
| Display | `/mypreferencesdisplay` | Theme, dashboard theme (admin), custom CSS, disable custom CSS, screensaver + timeout + backdrop interval, slideshow interval, faster animations, blurhash; library page size, library backdrops, theme songs/videos, missing episodes; language + date/time locale; Next Up (max days, rewatching, episode images); item detail banner. **Layout-mode selector is removed** (modern forced). |
| Home | `/mypreferenceshome` | Order and visibility of 10 home sections |
| Playback | `/mypreferencesplayback` | Audio channels (auto/mono/stereo/5.1/7.1), audio language, play default audio track, disable VBR audio, video quality (home/internet/Chromecast), max resolution, limit supported resolution, music quality, prefer fMP4 HLS, cinema mode, autoplay next episode, still-watching prompt, remember audio/subtitle selections, next-video overlay, Chromecast version, skip forward/back lengths, media segment actions, DTS/TrueHD/Hi10p flags, segment length limit, preferred transcode video/audio codec, audio normalization, always remux FLAC/MP3. Native external-player intent is dropped with the bridge. |
| Subtitles | `/mypreferencessubtitles` | Subtitle mode (default/always/only forced/none), language preferences, burn-in behavior, subtitle appearance (font, size, color, outline, shadow, background, position), secondary subtitles |
| Controls | `/mypreferencescontrols` | Enable gamepad/keyboard navigation |
| Profile | `/userprofile` | User avatar, name, password change, user info |
| Quick Connect | `/quickconnect` | Authorize a device with a code |

Preferences are stored in server `DisplayPreferences` (synced) and localStorage (client-only).

### 3.9 Admin dashboard

Routes from `src/apps/dashboard/routes/_asyncRoutes.ts` (+ 2 legacy routes and 2 standalone routes).

| Route | Page | Feature summary |
|---|---|---|
| `/dashboard` | Overview | Server info, item counts, running tasks, active devices, activity log, alerts, server paths, scan libraries, restart, shutdown |
| `/dashboard/settings` | General | Server name, preferred display language, metadata path, cache path, parallel image encoding, library scan fan-out, Quick Connect enable, contribution link |
| `/dashboard/branding` | Branding | Custom CSS, login disclaimer, splashscreen enable/upload/size/delete |
| `/dashboard/users` | Users list | Edit, library access, parental control, delete, delete confirmation |
| `/dashboard/users/add` | Add user | Username, password, validation, cancel |
| `/dashboard/users/:userId/:tab` | Edit user | Tabs: **Profile** (name, auth provider, password reset provider, feature access: media playback, transcoding, remote control, shared devices, media deletion, downloads; remote bitrate; max active sessions; login attempts/lockout; hide from login; disable user; SyncPlay access), **Access** (library folders + channels), **Parental Control** (max parental rating per type, block unrated, allow/block tags, access schedules), **Password** (reset/current password, reset provider) |
| `/dashboard/libraries` | Libraries | List libraries, add library (wizard: type, folders, language, metadata/image providers, savers, monitoring, chapter extraction), scan all, rename, remove, edit images, manage library options, task progress |
| `/dashboard/libraries/display` | Library display | Date added behavior, display specials within seasons, group movies/shows into collections, folder view, external content in suggestions |
| `/dashboard/libraries/metadata` | Metadata | Preferred metadata language, country, dummy chapter duration, chapter image resolution, image settings |
| `/dashboard/libraries/nfo` | NFO | Enable NFO saving, save image paths, path substitution, per-user NFO settings |
| `/dashboard/playback/transcoding` | Transcoding | Hardware acceleration type, hardware decode codecs, hardware encoding, encoder preset, quality/CRF (H.264/H.265), tonemapping mode/algorithm/peak/desaturation/param/range, VPP/VideoToolbox tonemapping, Intel low-power encoders, deinterlace method + double-rate, FFmpeg throttling + delay, segment deletion/keep, temp path, FFmpeg path, thread count, fallback font path, HLS audio seek strategy, max muxing queue, downmix audio scale, stereo downmix algorithm, audio VBR, QSV/VAAPI devices |
| `/dashboard/playback/resume` | Resume | Min/max resume percentage, min resume duration, audiobook min/max resume |
| `/dashboard/playback/streaming` | Streaming | Remote client bitrate limit |
| `/dashboard/playback/trickplay` | Trickplay | Scan behavior (non-blocking/blocking), image interval, width resolutions, tile width/height, JPEG quality, qscale, process priority, threads, keyframe-only extraction, acceleration encoding |
| `/dashboard/devices` | Devices | Device cards/list, app name, user, last active, now-playing (direct play/stream/remux/transcode reason), edit name, delete, delete all, send message |
| `/dashboard/activity` | Activity | Activity log, level/type/user/time filters, overview, media details, log-level settings (slow response enabled/time) |
| `/dashboard/livetv` | Live TV | Tuner devices, guide providers, refresh guide |
| `/dashboard/livetv/recordings` | DVR | Default recording settings (paths, padding, post-processor, guide days, NFO/images) |
| `/dashboard/livetv/tuner` | Tuner setup (rebuild) | Add/edit tuner hosts/providers |
| `/dashboard/livetv/guide` | Guide data (rebuild) | Add/edit listings providers, refresh guide |
| `/dashboard/plugins` | Plugins | Installed/available/search/categories, install/uninstall/enable/disable, repository management, plugin details, changelog/revisions, specific-version install, restart warning |
| `/dashboard/plugins/:pluginId` | Plugin details | Version, developer, repository, status, enable/disable, uninstall, revision history, install version |
| `/dashboard/plugins/repositories` | Repositories | List, add, delete, confirmation |
| `/dashboard/tasks` | Scheduled tasks | Task list, status, last run, progress, start/stop |
| `/dashboard/tasks/:id` | Task details | Triggers (interval, daily, weekly, on application startup), time limits, add/remove trigger, edit task |
| `/dashboard/keys` | API keys | List (app name, date issued), create (app name), revoke, one-time key display |
| `/dashboard/backups` | Backups | List backups, create (metadata/database), restore, delete, progress dialogs, warnings |
| `/dashboard/logs` | Logs | Log list, view log file, search/copy, slow response settings |
| `/dashboard/logs/:file` | Log file | View, copy, retry on failure |
| `/dashboard/networking` | Networking | HTTP/HTTPS ports, enable HTTPS, certificate path/password, require HTTPS, base URL, bind IPv4/IPv6, LAN networks, known proxies, auto discovery, published server URI, external/internal URI, remote access allow/block list |
| `/metadata` | Metadata manager | Library tree + item metadata editor |
| `/configurationpage` | Plugin config | Server-provided plugin configuration pages |

Dashboard chrome: responsive drawer with sections **Server** (Dashboard, General, Branding, Users, Libraries [Libraries, Display, Metadata, NFO], Playback [Transcoding, Resume, Streaming, Trickplay]), **Devices** (Devices, Activity), **Live TV** (Live TV, DVR), **Plugins** (Plugins + plugin config menu entries), **Advanced** (Networking, API Keys, Backups, Logs, Scheduled Tasks); page tabs rebuilt in the new framework; help button; admin gate; metadata manager hides drawer.

### 3.10 Setup wizard (`/wizard/*`)

| Route | Screen | Features |
|---|---|---|
| `/wizard/start` | Start | Welcome, display language, server name, help link, Continue |
| `/wizard/user` | Admin user | Admin username/password/confirm, validation, "more users later" |
| `/wizard/library` | Libraries | Add/edit libraries with media library creator, help link, Back/Next |
| `/wizard/settings` | Metadata | Preferred metadata language, country |
| `/wizard/remoteaccess` | Remote access | Enable remote access, help, Back/Next |
| `/wizard/finish` | Finish | Completion message, previous, finish → app |

Wizard shell: progress/back/next, public access level (`ConnectionRequired level='wizard'`), no normal app chrome.

### 3.11 Cross-cutting editors and dialogs

These are used from many screens and must be rebuilt as shared components:

| Component | Features |
|---|---|
| Metadata editor | All item fields (name, sort name, original name, overview, tagline, genres, tags, studios, year, ratings, dates, runtime, index/season/episode numbers, album/artist fields, air time/days, status, 3D format, content type, production locations, people/cast editor, lock data, special episode fields), save + toast |
| Identify | Search metadata providers, pick match, apply |
| Refresh metadata | Replace all/images, missing metadata, recursive, confirmation |
| Image editor | Upload, delete, download, set primary/backdrop, reorder (left/right), search remote images, image options |
| Image downloader | Download all images for an item |
| Subtitle editor | List/download/delete/upload subtitles, search providers, language/forced/hearing-impaired flags, machine/AI-translated badges |
| Subtitle uploader/sync | Upload files, subtitle offset sync overlay |
| Lyrics editor/uploader | Edit/upload/search lyrics, synced lyrics |
| Collection editor | Add/remove items, reorder |
| Playlist editor | Create/add/remove/reorder, save queue as playlist |
| Media library creator/editor | Type, folders, language, metadata/image providers, savers, real-time monitoring, chapter extraction, library options |
| Recording creator | Record/series record, channel, start/end, padding, folder, tags |
| Tuner picker / TV provider dialogs | Choose tuner, configure provider |
| Share dialog | Copy/share item links |
| Access schedule editor | Per-user time/day access |
| Confirm/prompt/alert/action sheet/toast | Shared dialog primitives |
| Directory browser | Pick folders for paths |
| Sort/filter/alpha pickers | Shared library controls |
| Item context menu | See §3.4 actions |

### 3.12 Platform and advanced features

- **Themes**: built-in theme set + server/user custom CSS; theme storage per user.
- **i18n**: 109 locale JSON catalogs (1,949 keys in en-US), RTL, localized dates/numbers, language + date/time locale selection.
- **Accessibility/input**: keyboard shortcuts, gamepad, focus management, ARIA labels, reduced motion. **TV remote and TV webview support are dropped.**
- **Realtime**: WebSocket session events (play state, session updates, restart/shutdown), SyncPlay, remote control.
- **SyncPlay**: create/join/leave groups, group list, synchronized play/pause/seek, playback permission handling.
- **Remote play/cast**: choose target session, control it, Chromecast sender.
- **Downloads**: direct file download (`/Items/{id}/Download`) only. **No native manager, no offline media cache, no offline library.**
- **Multi-server**: switch server through the web flow (`/selectserver`, saved server list).
- **PWA**: manifest/icons/theme color; optional service worker in the rewrite.
- **Plugin surface**: server plugin config pages + plugin-provided menu entries.

---

## 4. Proposed rewrite architecture

### 4.1 Stack

| Concern | Current | Rewrite |
|---|---|---|
| Layout | modern + legacy + TV modes, layout switcher | **Modern responsive only; no switcher** |
| Platform bridge | optional `window.NativeShell` (Android/iOS/TV wrappers) | **None — browser-only** (§9.2.3) |
| Framework | React 18 + MUI 6 + Emotion | **Svelte 5 + SvelteKit 2 + TypeScript**; no React/MUI |
| Routing | react-router 6 hash router, 2 route systems | SvelteKit SPA shell (`ssr = false`, `adapter-static`) + a **hash router** (`svelte-spa-router` or a small typed hash router) so `/#/home` and legacy `#!` links keep working; one route system, lazy route chunks |
| Icons | `material-design-icons-iconfont` | **Lucide** via `lucide-svelte`, imported per icon and tree-shaken |
| UI primitives | MUI 6 components | **Bits UI** (headless, Svelte) + a small internal token-styled component library |
| Styling | SCSS + MUI + CSS vars | Design tokens + CSS custom properties + component-scoped Svelte styles; refreshed visual language |
| Tables | `material-react-table` | `@tanstack/svelte-table` or a small accessible table component |
| Date/time | `@mui/x-date-pickers` | Bits UI date components (Calendar / Date Picker / Date Field) + token styling |
| API | `@jellyfin/sdk` + `jellyfin-apiclient` + raw axios/jQuery | `@jellyfin/sdk` + thin `SessionManager` over axios; no jQuery |
| Server state | TanStack Query + IndexedDB persistence | `@tanstack/svelte-query` + IndexedDB persistence, WebSocket invalidation |
| Client state | Context + ad-hoc singletons + events | Svelte 5 runes + small typed stores for player/queue/session/UI |
| Player | `playbackmanager.js` + per-media plugins | `PlaybackController` facade + framework-agnostic adapters (HTML5, HLS, cast, remote, book/pdf/comic/photo) wrapped by Svelte components |
| i18n | globalize + JSON catalogs | `svelte-i18n` or a small custom adapter over the same JSON catalogs, lazy locale load |
| Build | Webpack 5 | SvelteKit + Vite + `@sveltejs/adapter-static` (SPA fallback, `ssr = false`) |
| Tests | Vitest + React Testing Library (partial) | Vitest + `@testing-library/svelte` + Playwright e2e against a real server |

### 4.2 Route map

Single router, code-split by area:

```
/                      → redirect /home
/selectserver /addserver /login /forgotpassword /forgotpasswordpin
/quickconnect
/home                  (Home / Favorites)
/movies /tv /music /books /musicvideos /boxsets /playlists /homevideos /mixed /livetv
/list /search /details /lyrics
/video /queue
/userprofile
/mypreferencesmenu /mypreferencesdisplay /mypreferenceshome
/mypreferencesplayback /mypreferencessubtitles /mypreferencescontrols
/dashboard/*           (admin, lazy chunk)
/metadata              (admin)
/wizard/*              (setup)
/configurationpage     (plugin iframe)
!/*                    (bang redirect compatibility)
*                      (404)
```

Guards:

- `RequireConnection(level: public | user | admin | wizard)` replaces `ConnectionRequired`.
- Admin routes are additionally lazy-loaded and blocked for non-admin users.
- **One layout only.** There is no `layoutManager`, no layout mode stored in preferences, and no `desktop-legacy`/`mobile-legacy`/`tv` route branch. The modern responsive layout is always used.
- **SvelteKit is the app/build framework, not the URL router.** The SPA is built with `@sveltejs/adapter-static` (`ssr = false`, `fallback: 'index.html'`) and mounted in a single root page. Top-level navigation uses a hash router so `/#/home` and legacy `#!` links work without server rewrites. SvelteKit's History-API router is not used for the top-level screen table.

### 4.3 Session and auth layer

```
SessionManager
├── server list (localStorage)            // selectserver/addserver
├── connect(url) → /System/Info/Public
├── authenticate(username, password)      // /Users/AuthenticateByName
├── quickConnect()                        // /QuickConnect/*
├── token storage + refresh/expiry
├── current Api (SDK) + serverId + user
├── logout / switch server
└── WebSocket (/socket) + event bus
```

Rules:

- Never store credentials; store access token + server/user id.
- One `ApiProvider` exposes `{ api, serverId, user, session }`.
- All requests go through a single axios instance with base URL, auth header, `X-Emby-*` device headers, 401 handling.
- Keep device profile/capabilities in one module (replaces `browserDeviceProfile.js`).

### 4.4 Data layer

- Query keys normalized by domain: `['user', userId, 'items', parentId, params]`, `['item', id]`, `['views', userId]`, `['sessions']`, etc.
- WebSocket events invalidate the relevant keys; server restart/shutdown clears cache.
- Persist only safe, cheap queries to IndexedDB (views, display prefs, last library state); keep playback/session data volatile.
- Mutations invalidate by prefix, not whole cache.

### 4.5 Playback architecture

```
PlaybackController
├── negotiate(item, options)        → /Items/{id}/PlaybackInfo
├── choose method (direct/stream/transcode)
├── attach adapter
│   ├── HtmlVideoAdapter (hls.js for HLS, native for direct)
│   ├── HtmlAudioAdapter
│   ├── CastAdapter (Chromecast)
│   ├── RemoteSessionAdapter (control another session)
│   ├── BookAdapter (epubjs)
│   ├── ComicAdapter (libarchive + canvas)
│   ├── PdfAdapter (pdfjs)
│   └── PhotoAdapter (slideshow)
├── tracks (audio/subtitle/secondary subtitle)
├── progress reporter (/Sessions/Playing|Progress|Stopped)
├── queue + repeat/shuffle + play-next
├── media segments (intro/outro skip)
├── trickplay (thumbnails)
└── SyncPlay participant
```

Keep player adapters lazy: the HLS/cast/book/pdf/comic chunks must not be in the initial bundle.

### 4.6 UI system

- One Svelte shell layout: toolbar + drawer + content + now-playing bar.
- Headless behavior from **Bits UI**; all visuals come from our own token-styled Svelte components.
- Design tokens: color, spacing, typography, radius, elevation, breakpoints, focus ring, motion.
- Icons: **Lucide** (https://lucide.dev/) via `lucide-svelte`; import icons individually (never a barrel import) so unused icons are tree-shaken. Wrap the common set in a small `Icon` component for consistent sizing/stroke and RTL flipping.
- Primitives: Button, IconButton, Menu, Select, TextField, Checkbox, Switch, Tabs, Dialog, Snackbar, Card, List, Chip, Progress, Skeleton, Tooltip.
- Media primitives: `PosterCard`, `BackdropCard`, `ListRow`, `PersonCard`, `ChapterCard`, `TrackRow`.
- Library primitives: `LibraryTabs`, `FilterBar`, `SortMenu`, `ViewMenu`, `AlphabetPicker`, `Pagination`, `EmptyState`.
- All strings through `t()`; no hard-coded English.
- Every interactive element keyboard-focusable; focus is restored predictably after dialogs/menus. No TV-specific focus layer.

### 4.7 Theming and branding

- Tokens are the contract: a new visual language is defined in Phase 0 (§9.2.5) and implemented as CSS custom properties.
- Map the existing theme IDs to the new theme presets so saved user preferences still resolve.
- Inject server custom CSS and per-user custom CSS in a controlled `<style>` (sanitize/limit).
- Respect `prefers-color-scheme`, `prefers-reduced-motion`.

### 4.8 Internationalization

- Reuse `src/strings/*.json` catalogs (copy them into the rewrite or fetch from the server web root).
- Lazy-load locale chunk; fall back to `en-us`.
- Support RTL (`dir="rtl"`), localized dates/times via `Intl`.
- Keep server-provided metadata language/country separate from UI language.

### 4.9 Performance budgets

| Metric | Target |
|---|---|
| Initial JS (gzip) | ≤ 180 KB (Svelte/SvelteKit runtime is small; verify after the Phase 0 scaffold) |
| Initial CSS (gzip) | ≤ 40 KB |
| Route chunk (gzip) | ≤ 120 KB |
| Player chunk (gzip) | ≤ 220 KB |
| Admin chunk (gzip) | ≤ 250 KB total, split per page |
| First contentful paint (mid mobile) | ≤ 1.5 s on LAN |
| Library grid | virtualize/paginate; no full-DOM render of 1,000+ cards |

### 4.10 Compatibility

- Serve at the same base path as the current web client; hash routing avoids server rewrite rules.
- SvelteKit is configured for static SPA output: `ssr = false`, `@sveltejs/adapter-static` with `fallback: 'index.html'`, and relative asset paths (`paths.relative`) so the same build works at `/web/` or a custom base. The top-level hash router keeps deep links free of server rewrite requirements.
- Preserve `#!/` and `#!` legacy links via a redirect route.
- Keep `config.json` support (server address, base path, etc.).
- **Modern browsers only.** TV webviews (Tizen/webOS/Orsay/PS4/Xbox) are explicitly out of scope. The exact evergreen baseline is decision §9.2 #2.
- **Browser-only runtime.** No `window.NativeShell` detection or Android/iOS wrapper bridge (decision §9.2 #3). Features that only existed in wrappers — native download manager, client settings, native device profile, native player plugins, native exit — are not part of the rewrite.

---

### 4.11 What gets deleted (vs. ported)

Delete from the rewrite once parity is verified; keep the original sources in the current repo as behavioral reference until then.

| Removed | Replacement |
|---|---|
| `src/apps/legacy/AppLayout.tsx`, legacy-only routes in `src/apps/legacy/routes/**` | One modern router + responsive shell |
| `src/components/viewManager/**`, `components/router/LegacyRoute.tsx`, legacy page lifecycle/events | Svelte routes and components |
| `src/apps/{legacy,dashboard,wizard}/controllers/**` | Rebuilt in the new framework (features ported; controllers are reference only) |
| `src/components/layoutManager.js`, `src/constants/layoutMode.ts`, layout-mode preference | No layout manager; modern forced |
| TV branches (`browser.tv`, TV home-section logic, TV focus/scroller behavior), TV webview checks | Removed; keyboard + responsive only |
| `src/elements/emby-*` custom elements, `webcomponents.js`, `classlist.js`, `element-closest-polyfill` | Svelte primitives + modern browser APIs |
| `jquery`, `jstree`, `material-design-icons-iconfont`, `headroom.js`, `screenfull`, `swiper` (where unused) | Browser APIs/Svelte implementations or small libraries |
| `jellyfin-apiclient` | `@jellyfin/sdk` + thin `SessionManager` |
| `window.NativeShell` integration: native branches in `components/apphost.js`, `scripts/shell.ts`, `global.d.ts` types, `ServerConnections` auth callbacks, `connectionManager.findServers` native path, `index.tsx` native plugin loading | Browser APIs: Fullscreen, `navigator.mediaSession`, `window.open`, anchor downloads, web server selection; no wrapper bridge |
| Native-only `AppFeature` flags (`DownloadManagement`, `ClientSettings`, `ExitMenu`, `ExternalPlayerIntent`, `NativeBluRayPlayback`, `NativeDvdPlayback`, `NativeIsoPlayback`, ...) | Removed; keep only flags that have a browser implementation |
| Dead `appHost.supportsPlayMethod` branch in `htmlVideoPlayer` | Removed; keep the real `player.supportsPlayMethod` capability |
| Legacy polyfills (`core-js`, `whatwg-fetch`, `abortcontroller-polyfill`, `resize-observer-polyfill`, etc.) | Browser APIs per §9.2.2 baseline |
| `src/apps/legacy/features/playback/**` ad-hoc player glue | `PlaybackController` + adapters |
| React, ReactDOM, MUI, Emotion, `material-react-table`, `@mui/x-date-pickers` | Svelte 5 + Bits UI + TanStack Table + token-styled components |

---

## 5. API surface to implement

This is the minimum server contract the rewrite consumes. All paths are relative to the Jellyfin base URL.

| Domain | Endpoints / SDK areas |
|---|---|
| System | `/System/Info`, `/System/Info/Public`, `/System/Configuration`, `/System/Configuration/{key}`, `/System/Restart`, `/System/Shutdown`, `/System/Logs`, `/System/Logs/Log`, `/System/ActivityLog/Entries`, `/System/Info/Storage` |
| Auth | `/Users/AuthenticateByName`, `/Users/Public`, `/Users`, `/Users/New`, `/Users/{id}`, `/Users/{id}/Policy`, `/Users/{id}/Password`, `/Auth/Providers`, `/Auth/PasswordResetProviders`, `/QuickConnect/Enabled`, `/QuickConnect/Initiate`, `/QuickConnect/Connect` |
| Startup | `/Startup/Configuration`, `/Startup/User`, `/Startup/RemoteAccess`, `/Startup/Complete` |
| Items | `/Items`, `/Items/{id}`, `/Items/{id}/Ancestors`, `/Items/{id}/Similar`, `/Items/{id}/SpecialFeatures`, `/Items/{id}/LocalTrailers`, `/Items/{id}/ThemeMedia`, `/Items/{id}/Download`, `/Items/{id}/RefreshMetadata`, `/Items/{id}/Metadata`, `/Items/{id}/MetadataImages`, `/Items/{id}/RemoteImages`, `/Items/{id}/Images/{type}`, `/Items/{id}/Images/{type}/{index}`, `/Items/{id}/Trickplay/...` |
| Filters | `/Items/Filters`, `/Items/Filters2`, `/Genres`, `/MusicGenres`, `/Studios`, `/Artists`, `/Persons` |
| Search | `/Search/Hints`, `/Items?searchTerm=...` |
| User data | `/Users/{id}/Items/Resume`, `/Users/{id}/Views`, `/Users/{id}/FavoriteItems`, `/UserFavoriteItems/{id}`, `/UserPlayedItems/{id}`, `/DisplayPreferences/{id}`, `/DisplayPreferences/{id}/...` |
| Playback | `POST /Items/{id}/PlaybackInfo`, `/Sessions/Playing`, `/Sessions/Playing/Progress`, `/Sessions/Playing/Stopped`, `/Sessions/Capabilities/Full`, `/Sessions`, `/Sessions/{id}/Message`, `/Sessions/{id}/Playing`, `/Sessions/{id}/Command`, `/MediaSegments/{itemId}` |
| Collections/playlists | `/Collections`, `/Playlists`, `/Playlists/{id}/Items`, `/Playlists/{id}/Items/{itemId}` |
| Library admin | `/Library/VirtualFolders`, `/Library/VirtualFolders/Name`, `/Library/VirtualFolders/LibraryOptions`, `/Library/VirtualFolders/Paths`, `/Library/Refresh`, `/Library/MediaFolders`, `/Libraries/AvailableOptions` |
| Live TV | `/LiveTv/Channels`, `/LiveTv/Programs`, `/LiveTv/Programs/Recommended`, `/LiveTv/GuideInfo`, `/LiveTv/Recordings`, `/LiveTv/Recordings/Folders`, `/LiveTv/Timers`, `/LiveTv/SeriesTimers`, `/LiveTv/TunerHosts`, `/LiveTv/ListingProviders`, `/LiveTv/Recordings/Series` |
| Plugins | `/Plugins`, `/Plugins/{id}`, `/Plugins/{id}/Enable`, `/Plugins/{id}/Disable`, `/Plugins/{id}/{version}/Delete`, `/Packages`, `/Packages/Installed/{name}`, `/Repositories`, plugin configuration pages |
| Tasks | `/ScheduledTasks`, `/ScheduledTasks/{id}`, `/ScheduledTasks/Running/{id}`, `/ScheduledTasks/{id}/Triggers` |
| Backups | `/Backup`, `/Backup/Create`, `/Backup/Restore`, `/Backup/{id}` |
| Devices | `/Devices`, `/Devices/Info`, `/Devices/Options`, `DELETE /Devices?deviceId=` |
| API keys | `/Auth/Keys`, `/Auth/Keys/{key}` |
| Branding | `/Branding/Configuration`, `/Branding/Splashscreen`, `/Branding/Css` |
| Localization | `/Localization/Options`, `/Localization/Cultures`, `/Localization/Countries` |
| Realtime | WebSocket `/socket` |

---

## 6. Parity matrix (acceptance criteria)

A screen is "done" when all its row checks pass.

### 6.1 Public/auth

- [ ] Server list persists and reconnects; add/remove server works.
- [ ] Manual login, visual user picker, remember-me, error states.
- [ ] Quick Connect initiates/polls/authorizes/cancels.
- [ ] Password reset (PIN) flow completes.
- [ ] Login disclaimer/custom branding renders.
- [ ] Unreachable/restarting server states recover without reload.

### 6.2 Library browsing

- [ ] Every library type route renders its exact tab set (tables in §3.3).
- [ ] Each tab's data query matches current behavior (filters, sort, paging, alphabet).
- [ ] Grid/list toggle, card shape, image type, title/year toggles persist per user/view.
- [ ] Play All / Shuffle / Queue act on the current filtered result set.
- [ ] Create collection/playlist works with permission rules.
- [ ] Home sections honor user order and visibility; empty-library state links admins to library creation.
- [ ] Suggestions sections produce the same section types (continue watching/latest/next up/recommendations).

### 6.3 Details

- [ ] All type-specific sections in §3.4 render or hide correctly per item type.
- [ ] Every context-menu action in §3.4 works with correct permission gating.
- [ ] Track/source selectors, played/favorite toggles, download, trailer, split versions.
- [ ] Admin-only edit/identify/refresh shortcuts appear only for admins.

### 6.4 Playback

- [ ] Direct play, direct stream, remux, and transcode paths all play.
- [ ] HLS playback with adaptive quality; seek with buffered ranges.
- [ ] Audio/subtitle/secondary-subtitle selection; subtitle offset sync; subtitle styling.
- [ ] Trickplay thumbnails, chapters, intro/outro skip, next-up overlay, autoplay next.
- [ ] Progress reporting and resume across devices.
- [ ] PiP, AirPlay (where supported), fullscreen, cast, remote control.
- [ ] SyncPlay create/join/leave and synchronized control.
- [ ] Queue reorder/remove/clear/save-as-playlist; shuffle/repeat.
- [ ] Book/comic/PDF/photo viewers open and preserve progress.
- [ ] Playback errors are recoverable and reported.

### 6.5 Preferences

- [ ] Every field in §3.8 persists to `DisplayPreferences`/localStorage and takes effect immediately where applicable.
- [ ] Home section ordering editor round-trips.
- [ ] Subtitle appearance preview matches player output.
- [ ] Settings are per-user and survive logout/login.

### 6.6 Admin

- [ ] Every route in §3.9 renders with its listed controls.
- [ ] Save/validation/error/toast behavior matches.
- [ ] Destructive actions require confirmation.
- [ ] Live task/progress updates via WebSocket.
- [ ] Plugin config pages render (iframe) and plugin menu entries appear.
- [ ] Non-admin users cannot reach admin routes.

### 6.7 Wizard

- [ ] Full first-run flow completes and creates server config/admin/library.
- [ ] Back/next state is preserved; validation blocks progress.
- [ ] Completing the wizard lands on `/home` logged in.

### 6.8 Cross-cutting

- [ ] All strings localized; RTL layout works.
- [ ] Keyboard-only navigation works for every screen (no TV remote layer required).
- [ ] Theme switch, custom CSS, screensavers work.
- [ ] Screens use the new design tokens (no pixel diff against the old app).
- [ ] No legacy layout routes, layout switcher, TV-specific code paths, or `window.NativeShell` reads exist.
- [ ] Native-only UI (download manager, client settings, exit app) is absent; downloads use the web path only.
- [ ] 404/error boundaries never show a blank page.
- [ ] Bundle budgets in §4.9 met.

---

## 7. Delivery plan

### Phase 0 — Foundation (2–3 weeks)

- Repo/tooling: Svelte 5 + SvelteKit + TS + lint/test/CI, static SPA build to `/web`.
- Configure SvelteKit for the drop-in target: `ssr = false`, `@sveltejs/adapter-static`, relative paths, and the top-level hash router. Add `lucide-svelte` and the shared `Icon` wrapper.
- Settle the §9.2.2 browser baseline.
- Produce the §9.2.5 visual direction and token set (type/space/color/elevation/focus), then build the first primitives.
- Session manager, SDK client, auth screens (select/add server, login, quick connect, reset).
- App shell: toolbar, drawer, routing, guards, i18n, theme tokens, error/404 pages.
- Delete the layout-manager concept: no layout mode in preferences, no legacy layout branch.
- Delete the native-shell bridge: no `window.NativeShell`, no `shell.ts`, no native-only menu entries or `AppFeature` flags (Appendix E checklist).
- Query client + WebSocket invalidation.
- Home screen (library tiles/buttons + resume/next-up/latest) using existing home-section semantics.

Exit: SvelteKit SPA + hash router verified at `/web/`, design tokens and Lucide icon wrapper in place, and the app can log in, browse server libraries, see home, navigate, and log out.

### Phase 1 — Browse and details (3–4 weeks)

- All library routes and tabs with filters/sort/paging/alphabet/grid-list.
- Search (suggestions + all result categories).
- Item detail page with all type-specific sections and context-menu actions.
- Images (primary/backdrop/logo/thumb/banner/disc), blurhash, lazy loading.
- `/list` generic list page and `/lyrics`.

Exit: full browsing parity for non-admin users; no playback yet except links.

### Phase 2 — Playback (4–5 weeks)

- Playback negotiation + HTML5 video/audio adapters + HLS.
- OSD, tracks, subtitles, trickplay, chapters, segments, next-up, progress reporting.
- Now Playing bar, queue page, shuffle/repeat.
- PiP/fullscreen/AirPlay; cast + remote control; SyncPlay.
- Quality/playback preferences.

Exit: parity checklist §6.4 (minus readers).

### Phase 3 — Preferences (1–2 weeks)

- Menu, display, home, playback, subtitles, controls, profile, quick connect.
- DisplayPreferences sync + custom CSS/themes/screensavers.

Exit: §6.5 complete.

### Phase 4 — Admin dashboard (4–5 weeks)

- Shell/drawer + overview widgets.
- Users, libraries (+ creator/editor), settings, branding.
- Playback/transcoding/resume/streaming/trickplay.
- Devices, activity, logs, tasks, keys, backups, networking.
- Plugins + repositories + iframe config pages.
- Metadata manager + metadata editor/image editor/identify/refresh.
- Live TV admin (tuners/providers/DVR).
- Live TV user screens + DVR.

Exit: §6.6 and Live TV parity.

### Phase 5 — Wizard + readers + polish (2–3 weeks)

- Wizard flow.
- Book/comic/PDF/photo viewers.
- Keyboard/accessibility audit, performance tuning, PWA.
- Visual/UX pass and migration of any missing long-tail dialog.
- Final sweep: remove every legacy layout/layout-manager/TV code path and confirm no route depends on `viewManager`.

Exit: §6.7–6.8 complete; release candidate.

### Estimated total

~15–21 engineer-weeks for one experienced frontend engineer, or ~8–10 calendar weeks with two engineers splitting user-app vs. admin/player. This includes 1–2 weeks for Svelte/SvelteKit ramp-up and the new design tokens. The modern-only scope removes roughly 2 weeks of duplicated legacy-layout parity work; the retained cost is rebuilding the legacy pages that the modern app currently embeds.

---

## 8. Testing and QA

- **Unit**: pure utilities (sort/filter mapping, media method selection, date/space formatting, query key builders) with Vitest.
- **Component**: `@testing-library/svelte` for forms, dialogs, cards, tabs.
- **Integration**: mock server (MSW) for API/WebSocket flows.
- **E2E**: Playwright against a real Jellyfin server (Docker) covering login, browse, play (HLS fixture), admin CRUD, wizard.
- **Visual**: component/page snapshots for the **new** design at mobile/tablet/desktop breakpoints; no diff against the old app (visual parity is not a goal).
- **Accessibility**: axe checks in CI; keyboard-only walkthroughs.
- **Performance**: bundle-size budget in CI, Lighthouse budget, route chunk report.
- **Compatibility**: matrix per the §9.2.2 browser floor; no TV webview testing.

---

## 9. Risks, assumptions, and decision briefs

### 9.1 Resolved decisions

| Decision | Resolution | Consequence |
|---|---|---|
| Legacy layout/app | **Dropped** | No `desktop-legacy`/`mobile-legacy` code, no legacy-only routes, no dual implementations to maintain. Legacy controller pages that the modern app still embeds are rebuilt as features in the new framework, not carried over. |
| TV webviews | **Dropped** | No Tizen/webOS/Orsay/PS4/Xbox support, no TV remote/focus layer, no TV-specific home-section rules. Browser baseline can be modern-only. |
| Layout switching | **Removed** | The modern responsive layout is always used. The Display preference no longer exposes a layout mode; `layoutManager` is deleted. |
| Plugin configuration UI | **Sandboxed iframe + typed `postMessage` bridge** | Plugin pages stay server-provided HTML. The Svelte app hosts them in a route and provides the surrounding admin chrome; the bridge covers theme, resize, toast, navigate, close, and dirty state with origin checks (§9.2.6). |
| UI framework | **Svelte 5 + SvelteKit** | React, ReactDOM, MUI, Emotion, and React-only libraries are removed. SvelteKit is a static SPA build; top-level navigation uses a hash router to preserve `/#/...` URLs; icons use Lucide (`lucide-svelte`) (§9.2.1). |
| UI primitives | **Bits UI** | Headless behavior for dialogs, menus, selects, tabs, tooltips, popovers, sliders, switches, calendars, and date pickers; wrapped in token-styled components. Melt UI is rejected; TanStack Table remains for admin tables. |
| Visual design | **Refreshed** | Behavior parity is required; pixel parity is not. New design language defined in Phase 0, implemented with tokens (§9.2.5). |
| Downloads | **Download only, no offline mode** | Per-item file download only; no native manager, no offline media cache, no service-worker media storage (§9.2.4). |
| Browser baseline | **Accepted: evergreen** | Last 2 Chrome/Edge/Firefox, Firefox ESR, Safari 16+, iOS 16+, Android Chrome last 2; no polyfill layer (§9.2.2). |
| Native-shell hooks | **Removed — browser-only** | No `window.NativeShell`, no Android/iOS wrapper bridge, no native device profile/download manager/client settings/plugins; browser APIs and the web server-selection flow replace them (§9.2.3). |
| PWA | **Accepted: static app-shell SW (post-parity)** | Cache static assets only; never API responses; versioned cache + update prompt (§9.2.7). |
| Server floor | **Accepted: current line matching SDK** | Document the minimum; show an unsupported-version page; gate features on capabilities (§9.2.8). |

### 9.2 Decision briefs (outcomes)

Each brief records the context, realistic options, trade-offs, and outcome. All briefs are now decision records; nothing is pending. Record each outcome as a short ADR in `docs/adr/` so later phases do not re-litigate it.

---

#### 9.2.1 UI framework: Svelte 5 + SvelteKit

**Status: accepted — Svelte 5 + SvelteKit, hash router, Lucide icons.**

**Context.** The current modern pages use React 18 + MUI 6 + Emotion, including `material-react-table` for admin tables and `@mui/x-date-pickers` for date input. Keeping that stack locks in a large baseline, a virtual-DOM runtime, and a theming model that fights the lightweight goal. The rewrite moves to Svelte 5 and uses **SvelteKit** for the project/build layer.

**Why SvelteKit.** SvelteKit provides the project structure, dev server, Vite build, asset pipeline, TypeScript integration, and the static SPA adapter (`@sveltejs/adapter-static`). It also keeps the door open to file-based routes and prerendering for non-app pages. Cost: a small client runtime and SvelteKit conventions.

**Hash-routing constraint.** SvelteKit's router is History-API based. The drop-in target requires `/#/home` and legacy `#!` links without server rewrites. Therefore:

- `ssr = false`, `@sveltejs/adapter-static` with `fallback: 'index.html'`, and `paths.relative` for base-path portability.
- A single root SvelteKit page mounts the app shell and a **hash router** (`svelte-spa-router` or a small typed hash router). Screen routes are defined in TypeScript and lazy-loaded per chunk.
- SvelteKit's file-based router is **not** used for the top-level screen table. If history routing ever becomes acceptable (i.e. the host can rewrite unknown paths to `index.html`), the hash router can be swapped for SvelteKit's file routes without changing screen components.

**Options considered.**

| Option | Pros | Cons | Status |
|---|---|---|---|
| **A. Svelte 5 + SvelteKit** | Small runtime; compiler-driven, no virtual DOM; SvelteKit build/dev/asset tooling; `adapter-static` SPA; headless UI (**Bits UI**), `@tanstack/svelte-query`, `@tanstack/svelte-table`; excellent transitions for OSD/sliders | Slightly more runtime/conventions than plain Vite+Svelte; Svelte ecosystem is smaller than React; some libraries need small custom implementations; hash router needed for drop-in URLs | **Chosen** |
| **B. Svelte 5 + Vite only** | Smallest possible build; simple | Loses SvelteKit tooling/structure; still needs a hash router; no advantage for this app | Rejected |
| **C. SolidJS** | Fine-grained reactivity; very small | Smaller ecosystem; fewer headless UI options; team ramp-up | Rejected |
| **D. Preact** | Closest to current React code; large ecosystem | Keeps React-style architecture; less bundle win; still removes MUI | Rejected |
| **E. Lit / web components** | Standards-based | Router/state/forms/tables all become bespoke; weakest fit | Rejected |

**Accepted.** **Option A**:

- **Svelte 5** with runes and TypeScript.
- **SvelteKit 2** + `@sveltejs/adapter-static`, `ssr = false`, `paths.relative`, SPA fallback `index.html`.
- **Hash router** for top-level screens (`/#/home`, `#!` compatibility); typed route table with lazy `import()` chunks.
- **Lucide** (https://lucide.dev/) via `lucide-svelte`, imported per icon and wrapped in a shared `Icon` component for sizing/stroke/RTL.
- Supporting stack: **Bits UI** (including its Calendar / Date Picker / Date Field components), `@tanstack/svelte-query`, `@tanstack/svelte-table`, `svelte-i18n`. Melt UI is rejected for this project.

**Affects.** Every screen, component library, build, testing tools, hiring/ramp-up, bundle budget, deployment layout.

**Decided.** Phase 0, before the shell and primitives are built.

---

#### 9.2.2 Browser baseline

**Status: accepted — Option A (evergreen).**

**Context.** The current `browserslist` includes Chrome 27–63, Edge 18, iOS > 10, and Firefox ESR. That breadth exists mainly for TV webviews and old embedded browsers. With TV dropped, the rewrite can require a modern engine. The exact floor determines transpilation, polyfills, CSS features, and the test matrix.

**Options.**

| Option | Baseline | Pros | Cons |
|---|---|---|---|
| **A. Evergreen** | Last 2 versions of Chrome/Edge/Firefox, Safari 16+, iOS 16+ | No polyfills; ES2022; modern CSS (nesting, `:has`, container queries, `dvh`); smallest/fastest build; simplest testing | Excludes old corporate/kiosk browsers; may exclude some smart displays |
| **B. Broad modern** | Chrome/Edge 90+, Firefox 90+, Safari 14+, iOS 14+ | Wider reach; still drops TV webviews | Light transpilation and a few polyfills; more CSS fallbacks; larger test matrix |
| **C. Legacy desktop** | Chrome 63+ | Maximum reach | Polyfills, restricted CSS, larger bundle, more bugs — contradicts the "lightweight" goal |

**Accepted.** **Option A.** The floor is: Chrome/Edge last 2 stable, Firefox last 2 + ESR, Safari 16+, iOS Safari 16+, Android Chrome last 2. Required platform features: ES2022, WebSocket, IndexedDB, `ResizeObserver`, `Intl`, MSE (for `hls.js`; Safari uses native HLS). Verify the floor against real usage data before release; bump iOS/Safari only if analytics justify it.

**Affects.** Build target, polyfills, CSS feature usage, Playwright/browser matrix, bundle size.

**Decided.** Phase 0 implementation detail.

---

#### 9.2.3 Native-shell hooks (Android/iOS)

**Status: accepted — Option A (web-only; remove the native-shell bridge).**

**Context.** `window.NativeShell` + `AppFeature` currently exposes features that only exist in native wrappers: client settings, download manager, exit app, multi-server switching, external players, physical volume, sharing, cast/screensaver hooks, native device profile, and native player plugins. TV shells are already dropped. The question was whether to keep an adapter for Android/iOS wrappers or remove the bridge entirely.

**Options.**

| Option | Pros | Cons |
|---|---|---|
| **A. Web-only** | Smallest surface; no dead code paths; no wrapper contract to maintain; simplest testing; no `window.NativeShell` anywhere | Android/iOS wrappers lose client settings, download manager, exit, multi-server native flow, external player intent, native device profile, native plugins |
| **B. Keep Android/iOS capability hooks** | Preserves wrapper features; `AppFeature` checks keep UI correct in both contexts | Extra code paths, adapter, mock/test mode, and a contract that must be maintained across wrappers |
| **C. Full native parity including TV** | — | Rejected: TV support is dropped |

**Accepted.** **Option A**: delete all `window.NativeShell` integration. Browser replacements:

| Former native hook | Browser replacement / result |
|---|---|
| `AppHost.init` device identity | Generated/stored device id + browser-derived device name |
| `AppHost.getDeviceProfile` | Browser device profile (`browserDeviceProfile`) |
| `AppHost.screen` | `window.screen` × `devicePixelRatio` |
| `AppHost.exit` / `ExitMenu` | No exit menu; browsers cannot reliably close themselves |
| `enableFullscreen` / `disableFullscreen` | Fullscreen API |
| `updateMediaSession` / `hideMediaSession` | `navigator.mediaSession` where available; otherwise no OS media controls |
| `updateVolumeLevel` | No-op; platform volume keys apply |
| `downloadFile(s)` / `openDownloadManager` | Direct `<a download>` / `/Items/{id}/Download`; no manager |
| `openClientSettings` | Removed (web settings live in `/mypreferences*`) |
| `openUrl` | `window.open` / anchor |
| `selectServer` / `findServers` | Web `/selectserver` flow + server discovery API |
| `onLocalUserSignedIn` / `onLocalUserSignedOut` | No wrapper callback |
| `getPlugins` | Built-in player plugins only |
| `getDefaultLayout` | Already obsolete (modern forced) |
| `appHost.supportsPlayMethod` | Dead code; delete the branch |

**Affects.** User menu (no download manager/client settings/exit entries), downloads (download-only), plugin registry (built-ins only), playback negotiation inside wrappers (browser profile only), `AppFeature` enum cleanup, test matrix (single browser mode).

**Decided.** Phase 0 cleanup. Appendix E documents the removal, the browser replacements, and the code to delete.

---

#### 9.2.4 Downloads and offline behavior

**Status: resolved — download only. Offline mode is out of scope.**

**Context.** The current web client can download an item file (`/Items/{id}/Download`), and native shells (now removed) used to expose a download manager. There is no offline library, and building one is a separate product: storage quotas, eviction, expiry, sync, and codec/bitrate choices all become client concerns.

**Options considered.**

| Option | Pros | Cons | Status |
|---|---|---|---|
| **A. Server-download only** | Simple; one request; works today; no storage/sync model | No offline playback; downloads are source-quality and may be large | **Chosen** |
| **B. Offline media cache** (Cache Storage/IndexedDB + service worker) | Real offline playback | Quota/eviction, no iOS background downloads, transcode selection, progress/resume, expiry, quota UI, conflict handling; large implementation | Rejected for v1 and not on the roadmap |
| **C. Native-only download manager** | Reuses wrapper capabilities | Only available in shells | Removed with the native-shell bridge (§9.2.3) |
| **D. A + C** | Web download plus native manager | Still no offline playback | Not applicable |

**Resolution.** Implement **A** only: the download action streams `/Items/{id}/Download` for individual/multiple items. Do not add Cache Storage/IndexedDB media storage, service-worker media interception, an offline library UI, or any "available offline" state. If offline is ever revisited, it is a new project with its own plan.

**Affects.** Item detail actions, multi-download, service worker scope (static assets only, §9.2.7), player source selection. No storage layer is added.

**Decided.** Resolved for v1.

---

#### 9.2.5 Visual design: refreshed visual language

**Status: resolved — refreshed visual design, behavior parity only.**

**Context.** The rewrite is the cheapest moment to fix hierarchy, density, contrast, and responsiveness issues carried by the current UI. The acceptance bar is therefore **behavioral parity**, not pixel parity: every screen and action must exist and work, but layout polish, typography, spacing, color, iconography, and component styling are new.

**Options considered.**

| Option | Pros | Cons | Status |
|---|---|---|---|
| **A. Recreate the current modern look** | Fastest; familiar; easy visual comparison | Carries over visual debt; no differentiation | Rejected |
| **B. Full visual refresh** | Better hierarchy/density/accessibility; modern feel | Design + review time; risk of scope creep; no pixel baseline | **Chosen direction** |
| **C. Hybrid: same IA, tokenized restyle** | Keeps navigation/flow parity while updating the look | Requires discipline to avoid layout drift | **Implementation approach** |

**Resolution.** Use **B + C**: keep the information architecture, navigation model, terminology, and screen flows; introduce a new visual language (spacing scale, type scale, color/contrast, elevation, focus states, density modes) implemented entirely through design tokens. Define the design brief and token set in Phase 0, build components against it, and do the final polish in Phase 5. Do not diff screenshots against the current app as an acceptance gate.

**Affects.** Design effort, token set, component library, screenshot tests (snapshot the new design only), user acceptance.

**Decided.** Phase 0 for direction and tokens; visual polish in Phase 5.

---

#### 9.2.6 Plugin iframe bridge depth

**Status: accepted — Option B (iframe + typed `postMessage` bridge).**

**Context.** Plugin configuration pages are server-rendered HTML served from the Jellyfin web root (`/web/configurationpage?name=...`). The iframe decision is settled; the remaining question is how much integration the host provides. Note that a same-origin iframe with `sandbox="allow-scripts allow-same-origin"` is **not** a security boundary (the frame can remove its own sandbox); treat admin-installed plugin pages as trusted server content. The sandbox's practical value here is preventing accidental parent-DOM access and isolating styles/scripts.

**Options.**

| Option | Pros | Cons |
|---|---|---|
| **A. Plain iframe, no bridge** | Minimum work; plugin pages behave as they do today | Theme mismatch; no host notifications; plugin cannot request navigation/close; inconsistent UX |
| **B. Iframe + typed `postMessage` bridge** | Theme tokens, resize, toast/notification, dirty-state, navigate/close, optional save events; consistent admin chrome | Requires a small protocol and plugin-side adoption; some legacy pages will ignore it; must handle origin checks |
| **C. Rewrite plugin HTML into the client framework** | Full visual consistency | Unbounded per-plugin work; breaks plugin contracts — rejected |

**Accepted.** **Option B**: same-origin iframe route with a minimal, versioned bridge (`ready`, `resize`, `theme`, `toast`, `navigate`, `close`, `dirty`). Enforce `event.origin` checks on both sides. Allow a per-plugin fallback to **A** if a page breaks. If true isolation is ever required, serve plugin pages from a separate origin; otherwise document that plugin pages are trusted.

**Affects.** Admin plugin routes, drawer menu entries, theming, notifications, navigation.

**Decided.** Phase 4 (plugins).

---

#### 9.2.7 PWA / service-worker scope

**Status: accepted — Option B (static app-shell cache, post-parity).**

**Context.** The app ships a manifest but no service worker. A service worker improves repeat loads and resilience, but adds cache/versioning complexity that can cause hard-to-debug stale UI.

**Options.**

| Option | Pros | Cons |
|---|---|---|
| **A. No service worker** | Simplest; hashed assets make deploys predictable | No offline shell; slower repeat loads on poor networks |
| **B. Static app-shell cache** | Fast repeat loads; offline login/error shell; installable | Cache invalidation/update UX; debugging; must never cache API responses |
| **C. Full offline data cache** | — | Overlaps decision 9.2.4; not recommended |

**Accepted.** **Option B**, post-parity (Phase 5), with a versioned cache, `skipWaiting` prompt, and a strict rule: cache only static assets, never `/Users`, `/Items`, PlaybackInfo, or other API responses.

**Affects.** Deployment, caching, update UX, offline error handling.

**Decided.** Phase 5.

---

#### 9.2.8 Minimum Jellyfin server version

**Status: accepted — Option A (current server line matching the SDK).**

**Context.** The current SDK is a pinned unstable build (`@jellyfin/sdk 0.0.0-unstable.202610291827`). Jellyfin's API is mostly additive, but Live TV, plugin, and playback fields vary across server releases. Supporting old servers multiplies the test matrix.

**Options.**

| Option | Pros | Cons |
|---|---|---|
| **A. Current server line** (match the SDK, e.g. 10.11+) | Full feature set; one compatibility target; simplest | Users on older servers must upgrade |
| **B. One previous line** (e.g. 10.10+) | Wider adoption | Capability checks and conditional UI; more tests |
| **C. Older/unspecified** | — | Unpredictable behavior; not supportable |

**Accepted.** **Option A**, documented in the README and enforced with a clear "unsupported server version" page. Read `/System/Info` at connect time and gate feature entry points on advertised capabilities where cheap.

**Affects.** API usage, Live TV/plugin features, test matrix, support burden.

**Decided.** Phase 0 (document), revisited at release.

---

### 9.3 Risks

| Risk | Mitigation |
|---|---|
| Feature surface is enormous (admin + Live TV + readers + players) | Strict parity matrix, phase gating, lazy admin/reader chunks |
| Playback edge cases (codecs, HLS, subtitles, cast) | Reuse proven libraries (hls.js, libass-wasm, Cast SDK) and add adapter tests; keep fallback to server transcode |
| Server/plugin config pages are server HTML | iframe route + typed bridge; treat plugin pages as trusted admin content; per-plugin fallback |
| SvelteKit SPA + hash-router integration friction | Prototype `adapter-static` + `ssr = false` + hash router in Phase 0; keep the router behind one module so SvelteKit file routes can replace it later |
| Svelte ecosystem gaps (headless UI, table, date picker) | Prototype the 5 hardest primitives early; budget one small custom component per missing library |
| Rewrite-in-new-framework slows early velocity | Phase 0 vertical slice; pair on the first 3 screens; keep the parity matrix as the scope gate |
| Browser-baseline regret (too narrow/too broad) | Decide §9.2.2 in Phase 0; validate against real usage; keep the build target a one-line change |
| i18n drift | Reuse existing catalogs; automated key-diff check |
| Plugin-provided menu entries and custom CSS can break layout | Defensive rendering + sanitization |
| SyncPlay/remote session semantics | Port existing SyncPlay core logic and test with two clients |
| Data loss in metadata editors | Explicit save/cancel, dirty-state guards, confirmations |
| Offline scope creep | Keep §9.2.4 explicit; no offline claims until implemented |

### 9.4 Assumptions

- The rewrite is served by the Jellyfin server at the existing web root and is API-compatible.
- The framework is Svelte 5 + SvelteKit (static SPA + hash router); icons are Lucide. React/MUI/Emotion are not used.
- The visual language is new; behavior parity is the acceptance bar (§9.2.5).
- The minimum server version follows §9.2.8 (recommended: current line matching the SDK).
- Browser baseline follows §9.2.2 (evergreen; no TV webviews).
- Only the modern layout is supported; desktop, tablet, and mobile web are the target surfaces.
- There is no native-shell bridge; the rewrite is browser-only (§9.2.3).
- Downloads are download-only; there is no offline mode (§9.2.4).
- Plugin configuration pages remain server-provided HTML and are trusted admin content.
- The existing locale catalogs, theme IDs, and `config.json` contract can be reused.
- No server change is required for any in-scope feature.

---

## 10. Appendix

### A. Source-of-truth files used for this inventory

Routing and shell:

- `src/RootApp.tsx`, `src/RootAppRouter.tsx`
- `src/apps/{modern,legacy,dashboard,wizard}/routes/routes.tsx`
- `src/apps/modern/routes/asyncRoutes/{user,public}.ts`
- `src/apps/modern/routes/legacyRoutes/{user,public}.ts`
- `src/apps/legacy/routes/asyncRoutes/{user,public}.ts`
- `src/apps/legacy/routes/legacyRoutes/{user,public}.ts`
- `src/apps/dashboard/routes/_asyncRoutes.ts`, `_legacyRoutes.ts`
- `src/apps/wizard/routes/routes.tsx`
- `src/components/router/*`, `src/components/viewManager/ViewManagerPage.tsx`
- `src/components/layoutManager.js`, `src/constants/layoutMode.ts`
- `src/components/apphost.js`, `src/constants/appFeature.ts`

> Scope note: legacy route files and `viewManager` sources are listed for completeness and as reference behavior. They are **not** carried into the rewrite, except as evidence for the features that the modern app embeds (details, list, lyrics, queue, preferences, metadata, Live TV guide/tuner).

User app:

- `src/apps/modern/AppLayout.tsx`, `src/apps/legacy/AppLayout.tsx`
- `src/apps/modern/features/libraries/**`
- `src/apps/modern/features/preferences/**`
- `src/apps/modern/components/**`
- `src/apps/legacy/controllers/**`, `src/apps/legacy/features/**`
- `src/apps/legacy/routes/search.tsx`, `src/apps/legacy/routes/user/settings/index.tsx`
- `src/components/homesections/**`, `src/constants/homeSectionType.ts`
- `src/components/itemDetails/**`, `src/components/itemContextMenu.js`
- `src/components/playback/**`, `src/components/nowPlayingBar/**`, `src/components/playbackSettings/**`
- `src/plugins/**` (players, screensavers, SyncPlay)
- `src/scripts/libraryMenu.js`

Admin:

- `src/apps/dashboard/AppLayout.tsx`
- `src/apps/dashboard/components/**`
- `src/apps/dashboard/features/**`
- `src/apps/dashboard/routes/**`
- `src/apps/dashboard/controllers/**` (legacy guide/tuner)

Shared:

- `src/components/metadataEditor/**`, `imageeditor/**`, `subtitleeditor/**`, `mediaLibraryCreator/**`, `mediaLibraryEditor/**`
- `src/components/{filtermenu,sortmenu,viewSettings,playlisteditor,collectionEditor,recordingcreator,tvproviders,directorybrowser,accessSchedule}/**`
- `src/utils/query/queryClient.ts`, `src/hooks/**`
- `src/themes/**`, `src/strings/*.json`
- `src/manifest.json`, `src/index.html`

### B. REA analysis evidence

The repository was analyzed with REA `analyze_javascript_application` over `src/` (directory format):

- **Evidence IDs**
  - Parent analysis: `ev_e5693461d556c1e92feb19518e2213dd13465b33bc732199921d4d19faf5dac2`
  - Summary view: `ev_5b9f0ac0a95c669e1354c1a18f89f8a5e221a0040ebc2e918b6729539951fe32`
  - Route summary view: `ev_8adb8502836ae65939e9ed4953e3684e44ecd7a6d7f015dc0777d7731a6b93f6`
- **Observed scale**: 1,027 relevant files, 848 parsed JS/TS files, 14,384,700 text bytes, 458,286 AST nodes, 11,429 findings, 1 parse failure.
- **Static route inventory (partial)**: 12 route patterns / 18 occurrences (5 `page`, 1 `system`, 6 `unknown`), 74 interpretation gaps. Patterns found included `/`, `/*`, `/*/*`, `/*/video`, `!/*`, `/login`, `/forgotpasswordpin`, `/wizard`, `/wizard/start`, `/dashboard/users`, `/dashboard/users/add`, `/dashboard/livetv/guide`, `/dashboard/livetv/tuner`.
- **Limitation**: the extractor cannot enumerate routes assembled from imported arrays, JSX `path` props, and dynamic route builders; the gaps are explicit in the retained Evidence. The screen inventory in §3 was therefore completed from the route config files listed in Appendix A (authoritative for this source repository), not from the static route view alone.
- **Feature tracing**: a `playbackManager` trace exceeded the 10 MiB MCP response budget (`resource_constraint`); its complete Evidence was retained (`ev_918614e64afe1ff8e6395bd6359a8b98c51b631295dbdd2b1695cb5c965e8dbd`) and exported as a 306,733,382-byte bundle to `/tmp/opencode/jellyfin-web-rea-evidence.json` (11 records). No conclusion in this document depends on the truncated trace.
- **Skill guidance respected**: for a complete source repository, source files are the primary authority; REA graph/route evidence is used as corroboration and to document coverage limits.

### C. Screen count summary

| Area | Count |
|---|---|
| Public/auth/startup | 6 |
| User app routes (modern; includes rebuilt details/list/lyrics/queue/preferences) | 25 |
| Admin dashboard routes (incl. guide/tuner/metadata/plugin config, rebuilt in the new framework) | 31 |
| Wizard steps | 6 |
| **Top-level screens** | **~68** |
| Shared editors/dialogs | ~25 |
| Legacy-only routes removed | 5 (legacy home/movies/tv/music/livetv variants) |
| Locales | 109 |

### D. Current bundle/dependency hot spots to eliminate

`react`, `react-dom`, `@mui/*`, `@emotion/*`, `material-react-table`, `jquery`, `jstree`, `webcomponents.js`, `classlist.js`, `element-closest-polyfill`, `intersection-observer`, `resize-observer-polyfill`, `proxy-polyfill`, `abortcontroller-polyfill`, `whatwg-fetch`, `core-js`, `material-design-icons-iconfont`, `headroom.js`, `screenfull`, `swiper`, `flv.js` (only if needed), `window.NativeShell` integration and `scripts/shell.ts`, native-only `AppFeature` flags, legacy `viewManager` controllers, and the dual layout implementations.

### E. Native-shell removal (web-only)

This appendix records what the native-wrapper bridge did, why it is being removed, and what replaces each hook. **Decision: remove all `window.NativeShell` support** (§9.2.3); the rewrite is browser-only. TV shells were already out of scope.

#### E.1 What it was

- A native wrapper (Android/iOS) injected a global object named `window.NativeShell` before the web bundle ran. The type contract lived in `src/global.d.ts`.
- `src/components/apphost.js` detected it and either delegated to `NativeShell.AppHost` or fell back to browser-derived behavior.
- Native features were advertised through string capability checks (`appHost.supports(AppFeature.X)`); when `NativeShell` was present, its `AppHost.supports()` was the sole authority.
- The wrapper also provided lifecycle/auth/media/download callbacks with no browser equivalent.

The rewrite does not detect or call this global. The table below is retained as a migration reference.

#### E.2 Hooks used by the current web app (historical)

| Hook | Purpose | Current consumers (examples) | Browser fallback |
|---|---|---|---|
| `AppHost.init()` | Return app/device identity (`deviceId`, `deviceName`, optional `appName`/`appVersion`) at boot | `src/index.tsx` (`await appHost.init()`), `ServerConnections` | Generated/stored device id + browser device name |
| `AppHost.supports(feature)` | Capability gate for UI and playback | ~60 call sites: user menu, preferences, player, downloads, subtitles, casting | `supportedFeatures()` list in `apphost.js` |
| `AppHost.getDeviceProfile(builder, version)` | Build the playback device profile (codecs, HDR, containers, bitrate limits) | `appHost.getDeviceProfile()` in `htmlVideoPlayer`/`htmlAudioPlayer` → PlaybackInfo | `src/scripts/browserDeviceProfile.js` |
| `AppHost.screen()` | Physical screen dimensions → max transcode resolution | `apphost.js` device-profile width limit | `window.screen` × `devicePixelRatio` |
| `AppHost.deviceId/deviceName/appName/appVersion()` | Device registration and API client identity | `ServerConnections` | Generated id / stored name / `Jellyfin Web` + package version |
| `AppHost.exit()` | Exit the native app | `appHost.exit()` from user menu/back handling | Removed — no exit menu |
| `AppHost.getDefaultLayout()` | Default layout mode (legacy/TV/mobile/desktop) | `layoutManager` | **Dropped** — modern is forced (§9.2.1) |
| `enableFullscreen()` / `disableFullscreen()` | Native fullscreen control | `scripts/shell.ts`, book player OSD, video OSD | Fullscreen API |
| `updateMediaSession(info)` / `hideMediaSession()` | OS lock-screen/media notification metadata and controls | `mediaSessionSubscriber.ts` | `navigator.mediaSession` where available |
| `updateVolumeLevel(volume)` | Notify wrapper of hardware/remote volume changes | Player volume paths | No-op |
| `downloadFile(item)` / `downloadFiles(items)` | Hand downloads to the native manager | `scripts/fileDownloader.js`, `shell.downloadFiles` | Anchor download (`/Items/{id}/Download`) |
| `openDownloadManager()` | Open the native download manager UI | User settings/menu | Removed — no download manager |
| `openClientSettings()` | Open native client settings | User settings/menu | Removed — web preferences only |
| `openUrl(url, target)` | Open external URLs in the wrapper | Link buttons, forgot-password PIN file | `window.open(url, target)` |
| `selectServer()` | Switch servers via the native flow | Logout/multi-server menu | Navigate to `/selectserver` |
| `findServers(timeout)` | Local server discovery | `connectionManager.js` | Server discovery API / empty list |
| `onLocalUserSignedIn(user, token)` / `onLocalUserSignedOut(info)` | Sync auth state into the wrapper | `ServerConnections` | No-op |
| `getPlugins()` | Return native player plugin IDs to load | `index.tsx` plugin loading | Built-in player plugins only |

Capability flags currently used by the UI (grouped):

- **Shell/navigation:** `DownloadManagement`, `ClientSettings`, `Exit`, `ExitMenu`, `MultiServer`, `ExternalLinks`, `TargetBlank`, `Sharing`, `FileDownload`, `FileInput`, `Screensaver`, `DisplayLanguage`.
- **Playback:** `RemoteControl`, `RemoteAudio`, `RemoteVideo`, `Fullscreen`, `PhysicalVolumeControl`, `HtmlAudioAutoplay`, `HtmlVideoAutoplay`, `ExternalPlayerIntent`, `NativeBluRayPlayback`, `NativeDvdPlayback`, `NativeIsoPlayback`, `Chromecast`, `CastMenuHashChange`.
- **Subtitles:** `SubtitleAppearance`, `SubtitleBurnIn`.
- **Obsolete for the rewrite:** `DisplayMode` (layout selector removed).

#### E.3 Removal checklist and browser replacements

Delete every `window.NativeShell` read. Files/areas to change:

- `src/components/apphost.js` — remove native branches in `init`, `supports`, `getDeviceProfile`, `screen`, `deviceName/deviceId`, `appName/appVersion`, `exit`; `supports()` always uses the browser feature list. `getDefaultLayout` is already obsolete.
- `src/global.d.ts` — remove the `NativeShell*` interfaces and `Window.NativeShell`.
- `src/scripts/shell.ts` — delete; replace call sites with browser APIs (`window.open`, Fullscreen API, `navigator.mediaSession`, anchor download).
- `src/lib/jellyfin-apiclient/ServerConnections.js` — remove `onLocalUserSignedIn` / `onLocalUserSignedOut` wrapper calls.
- `src/lib/jellyfin-apiclient/connectionManager.js` — remove the `NativeShell.findServers` branch; use server discovery only.
- `src/index.tsx` — remove `window.NativeShell.getPlugins()` merging and native feature filters; keep built-in plugin loading.
- `src/apps/legacy/features/playback/utils/mediaSessionSubscriber.ts` — remove the `hasNativeShell` branch; use `navigator.mediaSession` only.
- `src/plugins/bookPlayer/BookOsd/BookOsd.tsx` — use the Fullscreen API instead of native fullscreen calls.
- `src/plugins/htmlVideoPlayer/plugin.js` — remove `window.NativeShell`/webOS fullscreen branches and the dead `appHost.supportsPlayMethod` branch.
- `src/scripts/browserDeviceProfile.js` — remove the Tizen/NativeShell condition (TV dropped).
- `src/apps/modern/components/AppToolbar/index.tsx` — back button uses browser history, not `window.NativeShell`.
- `src/components/toolbar/AppUserMenu.tsx`, `src/scripts/libraryMenu.js`, `src/apps/legacy/routes/user/settings/index.tsx` — remove Download Manager, Client Settings, and Exit app entries.
- `src/utils/dashboard.js` — `selectServer()` always navigates to `/selectserver`; logout follows the web multi-server preference.
- `src/scripts/fileDownloader.js` — always use the web download path.
- `src/components/displaySettings/*`, `src/apps/modern/features/preferences/*` — remove native-only menu entries and flags (`DisplayMode` is already removed).

Capability enum cleanup:

- **Remove native-only flags:** `DownloadManagement`, `ClientSettings`, `ExitMenu`, `ExternalPlayerIntent`, `NativeBluRayPlayback`, `NativeDvdPlayback`, `NativeIsoPlayback`, `CastMenuHashChange` (if only native), `Exit` (TV-only), `DisplayMode` (layout removed).
- **Keep browser-relevant flags:** `FileDownload`, `Sharing`, `ExternalLinks`, `TargetBlank`, `FileInput`, `Screensaver`, `DisplayLanguage`, `Fullscreen`, `RemoteControl`, `RemoteAudio`/`RemoteVideo`, `PhysicalVolumeControl`, `HtmlAudioAutoplay`/`HtmlVideoAutoplay`, `Chromecast`, `SubtitleAppearance`, `SubtitleBurnIn`, `MultiServer` (web setting).
- Recompute the browser feature list without TV branches (`browser.tv`, Tizen, webOS, Orsay, PS4, XboxOne) and without `NativeShell` delegation.

Behavior after removal:

| Former native hook | Browser result |
|---|---|
| Device identity | Generated/stored device id + browser-derived device name |
| Device profile | `browserDeviceProfile` only |
| Screen size | `window.screen` × `devicePixelRatio` |
| Fullscreen | Fullscreen API |
| Media session | `navigator.mediaSession` where available; otherwise no OS media controls |
| Volume updates | No-op; platform volume keys apply |
| Downloads | Direct `<a download>` anchor download of `/Items/{id}/Download` |
| URLs | `window.open` / anchors |
| Server selection | Web `/selectserver` flow + saved servers |
| Server discovery | Server discovery API |
| Native plugins | Not available; built-in players only |
| Exit app | No menu entry; browsers cannot reliably close themselves |
| Auth callbacks | No wrapper sync |

Consequences:

- Android/iOS wrappers cannot host this client without re-adding their own bridge. If that requirement returns, add a single adapter module — do not scatter `window.NativeShell` checks again.
- Wrappers would be treated as generic browsers, so native device-profile accuracy (HDR/codecs) is lost. Acceptable because wrappers are out of scope.
- Bundle and test matrix get smaller: one runtime mode, no native mock fixtures, no wrapper-specific feature branches.

Gotchas found in the current code:

- `Exit` and `ExitMenu` are distinct: `Exit` is browser/TV back-exit support, `ExitMenu` is the menu entry and is effectively native-only. Both are removed in the web-only rewrite.
- `DownloadManagement` and `ClientSettings` are never produced by the browser feature list — they exist only for native wrappers, so they disappear with the bridge.
- `getPlugins()` was the extension point for native player implementations; with the bridge removed, the plugin registry contains built-in players only.
- `appHost.supportsPlayMethod(...)` is referenced by the HTML video player but never defined by the browser `appHost`; it is dead code and is dropped.
