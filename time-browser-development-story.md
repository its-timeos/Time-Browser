# Building Time Browser: The Full Story of a One-Month Build

## Introduction

Over roughly a month — from July 29 to August 30 — a single, continuous Claude conversation turned into the build log of an entire desktop application. What started as a request to convert a bloated, 6,000-line Software Requirements Specification into a browsable HTML document ended with a working, packaged, installable macOS and Windows browser called **Time Browser**, complete with its own liquid-glass marketing website. In between sat four development phases, a handful of genuinely nasty runtime bugs, an honest audit that caught the project drifting from its own spec, and a lot of very deliberate "do not rewrite what already works" discipline. This is the story of how that happened, told from the beginning.

## Act 0: Taming the Spec

Before a single line of application code existed, there was the SRS — a Software Requirements Specification originally written for a product called "Nova," repurposed and renamed to Time Browser. It was enormous: over 6,000 lines of markdown, riddled with duplication from a messy Google Docs export, and describing a genuinely ambitious feature set spanning not just a browser but a Notes app, a Calculator with AST-based parsing, a Calendar, a Task manager, and more.

The first real task was mechanical but non-trivial: convert this markdown monster into a single polished, self-contained `index.html` documentation site. This meant writing a script to parse the markdown, de-duplicate repeated sections, and restructure the whole thing into semantic HTML with a proper cover page, a sticky sidebar table of contents generated from the heading hierarchy, a `Cmd/Ctrl+K` search overlay built on a client-side search index, syntax-highlighted code blocks with copy buttons, a scroll-progress bar, a right-edge "currently reading" section indicator, print stylesheets with running headers and page numbers, and a "Liquid Glass" visual language — dark canvas, translucent blurred panels, subtle borders, a blue accent — all done in vanilla JS with zero external dependencies (even avoiding a Google Fonts CDN link once external dependencies were flagged as undesirable).

This document ended up being more than just record-keeping. It became the spec the rest of the project would be measured against — and, as it turned out later, a spec the actual build would occasionally fall short of.

## Phase 1: The Scaffold

With the documentation in place, actual development began under a strict set of rules that would repeat, nearly verbatim, at the start of every subsequent phase: never rewrite the entire project, only modify files that need changes, never introduce a framework or bundler, everything handcrafted in vanilla HTML/CSS/ES6 modules, running on Electron.

Phase 1 built the skeleton:

- An **Electron main process** creating a `BrowserWindow` with a hidden inset title bar and macOS vibrancy, aiming for that native "liquid glass" look right from the window chrome itself.
- A **secure preload script** using `contextBridge` with `contextIsolation` enabled and `nodeIntegration` disabled — exposing only a minimal, deliberately narrow API surface (platform info, app version) rather than the raw Node runtime.
- A **CSS design token architecture** — `variables.css`, `reset.css`, `base.css`, `layout.css`, and more, all loaded in a strict cascade order, using a `--tb-` prefix mirroring the tokens already established in the SRS.
- The **application shell**: a sidebar, a header, and a dashboard area, wired together by a lightweight custom router.
- **Storage, settings, and theming** groundwork, plus foundational UI primitives — a modal system, a notification queue, and a generic `Registry` class that would go on to become the backbone of nearly every extensible system built afterward (themes, widgets, commands all register into instances of this same pattern).

Electron was pinned to version 31.3.0 at this stage — a detail that would matter enormously later.

## Phase 2: Making It Feel Alive

Phase 2 was where the shell actually became a product. This phase leaned hard on the Registry pattern established in Phase 1, using it to build:

- A **10-theme registry** — Default Dark, Midnight, Nord, Tokyo Night, Dracula, Ocean, Forest, Glass White, Cyber Blue, and AMOLED Black — implemented as CSS data attributes.
- A **16-property live customization pipeline** that writes CSS custom properties directly onto the document root, so changes to accent color, blur radius, or specular highlight intensity apply instantly with no reload.
- **Eight dashboard widgets** (Clock, Weather, Calendar, Todo, Notes, Recent Pages, Downloads, System Info), each implementing a common interface with an id, title, and render function, registered into a widget registry at startup.
- A **command palette** (`Cmd/Ctrl+K`) and a full keyboard shortcut system, both routed through a shared command registry rather than scattered event listeners.
- An **11-category settings panel**, schema-driven — the settings UI is generated from a data structure describing categories and fields, rather than hand-built for each option.
- The **omnibox input parser** — the logic that decides whether what you typed is a URL or a search query — plus a profile data foundation and layout presets.

This phase ended with a genuinely useful moment of engineering hygiene: a full static dependency analysis across all 33 project files, checking for missing imports and circular dependencies, which came back completely clean. In the course of that audit, a subtle design smell was caught and fixed — `settings.js` had its own default `searchEngine` value that duplicated (and could conflict with) a `searchEngine` field already living on the user's profile object. It was quietly removed in favor of the single source of truth.

## Phase 3: The Browser Engine, in Six Milestones

This was the largest phase by far, and it's where Time Browser actually became a *browser*. Rather than diving straight into code, this phase opened with an explicit planning step: inspect the existing architecture, and produce an ordered roadmap where foundational systems get built before the features that depend on them. The reasoning: real back/forward navigation, the Recent Pages widget, the Downloads feature, and the omnibox's search suggestions all depend on having actual browsing state to work with — so that state needed to exist first.

The six milestones, each validated (JS syntax, import resolution, circular dependency checks, CSS integrity) before moving to the next:

**Milestone 1 — Tab and Navigation Data Model.** The foundational state layer: a `WebContentsView`-based tab management system living in the Electron main process, plus a renderer-side mirror of that state (`browserState.js`) so the UI could reactively reflect what was happening in the actual browser engine without duplicating logic. Error handling was added for failed page loads via the `did-fail-load` event, deliberately filtering out error code `-3` (`ERR_ABORTED`), which fires on ordinary redirects and isn't a real failure worth surfacing to the user.

**Milestone 2 — Tab Strip UI.** The visible row of tabs, with loading indicators and error states, plus verification that mounting and unmounting the tab strip was fully re-entrant — meaning rapid navigation between routes (say, clicking Dashboard then Browser then Dashboard again quickly) wouldn't leave duplicate DOM nodes or orphaned event listeners behind. This phase also caught and removed a leftover unused `qs` import from `tabStrip.js` during a validation pass — a small thing, but exactly the kind of debt that was being swept up before it could compound.

**Milestone 3 — History.** Persisting visited pages (URL, title, favicon, timestamp) through the storage layer, feeding both a history view and the omnibox's suggestion dropdown.

**Milestone 4 — Downloads.** Built on Electron's session-level `will-download` event — attached once to `session.defaultSession` rather than per-tab, since it's a session-wide event. Each download got a unique ID and tracked state (filename, path, byte progress, status), broadcasting `browser:download-started`, `browser:download-updated`, and `browser:download-done` events to keep the renderer's UI synchronized. This logic was deliberately pulled out into its own `downloadManager.js` module rather than bolted onto the already-substantial `tabManager.js`. A dedicated Downloads route replaced what had been a placeholder toast notification, and the dashboard's Downloads widget was updated to show real data instead of stub content.

**Milestone 5 — Bookmarks (first pass).** A one-click bookmark toggle wired into the header, with the reactive state to back it, plus branded custom error pages and favicon-cache integration.

**Milestone 6 — Polish and completion checks**, closing out Phase 3 with updated README, CHANGELOG, and architecture documentation, and one more full validation pass.

## The First Real Bugs: What Static Validation Couldn't Catch

Everything up to this point had been validated *statically* — syntax checks, import graphs, circular dependency detection. None of that can catch what only shows up when an app actually runs. So the next stage of the project was a genuine runtime test on a real Mac, and it surfaced four distinct problems at once:

1. A hard crash: `TypeError: wc.navigationHistory?.canGoBack is not a function`
2. Overlapping UI elements
3. Scrolling that simply didn't work in places
4. Icons rendering as random, unrecognizable shapes
5. Some visible controls not responding to clicks at all

Each of these got its own dedicated investigation-and-fix pass, deliberately kept separate from one another so that fixing one didn't risk breaking something else.

### Bug #1: The Electron API Mismatch

The crash was the most urgent and the most interesting to diagnose. The error message was genuinely confusing on its face: `wc.navigationHistory?.canGoBack is not a function`. Optional chaining (`?.`) is supposed to short-circuit safely if `navigationHistory` is `undefined` — so for the error to fire at all, `navigationHistory` had to exist as a truthy object, just one that didn't have `canGoBack` as a callable method.

The root cause traced back to Electron's own API history: `webContents.navigationHistory`, with methods like `canGoBack()` and `canGoForward()`, was introduced in **Electron 33**. This project was pinned to **Electron 31.3.0** (confirmed as `31.7.7` when actually running), which doesn't have that API at all — it uses the older, flatter methods directly on `webContents`: `canGoBack()`, `canGoForward()`, `goBack()`, `goForward()`. Code written for a newer Electron API had apparently ended up in `tabManager.js`'s `serializeTab()` function, incompatible with the pinned version actually installed. The fix was narrow and surgical: every `navigationHistory`-based call was replaced with the Electron-31-compatible direct methods, the rest of the codebase was scanned for any other version-sensitive API usage (none found), and — critically — Electron itself was *not* upgraded, since that risked a much larger cascade of compatibility issues elsewhere. The fixed file was re-packaged and handed back for the person to test without touching anything else in the project.

### Bug #2: Icons That Weren't Icons

Investigating the "random symbols" complaint revealed the actual cause was almost charming in its simplicity: there were no real icons anywhere in the app. Every "icon" was an empty `<span>` styled with CSS alone — background colors, `border-radius` tricks, and `clip-path` polygons standing in for glyphs. This is a common placeholder technique, but it doesn't scale to a real, recognizable icon set, and it was exactly what the person was seeing as "incorrect/random-looking symbols."

The fix was a systematic pass across the entire UI: cataloging every icon-bearing element — eight sidebar navigation icons plus a collapse toggle, header controls (back, forward, refresh, home, bookmark star, theme switcher, notification bell, settings, search), the command palette's search icon, omnibox suggestion markers (star for bookmarks, magnifying glass for search rows) — and replacing every CSS shape-hack with a real, consistent inline SVG icon system. Elements that were legitimately abstract UI (loading spinners, status dots, decorative bullets) were correctly left alone rather than being "fixed" into something they were never meant to be. Accessibility was preserved throughout — icon-only buttons kept their ARIA labels. The obsolete CSS shape rules were stripped out afterward across every affected stylesheet, and a full validation pass (including checking both static HTML icons and any JS-generated ones) confirmed nothing was left behind.

### Bug #3: The Classic Flexbox Overflow Bug

The overlap and broken-scrolling complaints turned out to share one root cause, and it's a bug nearly every web developer has hit at least once: **missing `min-height: 0` on flex and grid children.**

By default, a flex item won't shrink below the size of its content — so when a scrollable area (like the dashboard) is nested inside a flex container with `overflow: hidden` further up the chain, the content doesn't scroll; it just gets silently clipped, because the flex item was trying to grow to fit everything and the ancestor was cutting it off instead. The same problem, in grid form, was also present: rows sized with `auto` let their content overflow rather than respecting the container's actual allocated space.

The fix threaded through the entire layout chain — `.app-shell` → `.main-area` → `.dashboard` → `.browser-route` → `.sidebar` and `.sidebar__nav` — adding `min-height: 0` at every level of that chain, and using `grid-template-rows: minmax(0, 1fr)` where grid sizing was in play, so rows would fill available space without growing past it. This mattered for more than just visual polish: the app's native `WebContentsView` (the actual embedded browser viewport) gets its bounds calculated from `getBoundingClientRect()` on its container element in the DOM. If that container's size was wrong because of the flexbox bug, the native browser viewport itself would be positioned or sized incorrectly — which explains the "unresponsive controls" complaint too: clicks landing on the wrong coordinates because the visual layout and the actual native view bounds had drifted apart.

## Theme Synchronization: Making External Sites Match the App

With the crash and layout bugs fixed, the next request was more of a feature than a bug: when Time Browser is in a dark theme, embedded pages (particularly search engines like Google, Bing, and DuckDuckGo) should *also* render in dark mode where they support it — and vice versa for light themes — without ever injecting arbitrary CSS into someone else's website.

The chosen mechanism was elegant: rather than injecting CSS or JavaScript into every page (fragile, and exactly the kind of thing the constraints explicitly ruled out), the app uses the **Chrome DevTools Protocol**, attaching a debugger session to each tab's `WebContentsView` and issuing the `Emulation.setEmulatedMedia` command with `prefers-color-scheme` set appropriately. This is the same mechanism Chrome's own DevTools uses to preview dark/light mode — it changes what the `prefers-color-scheme` media query *reports* to the page, which triggers the site's own CSS to respond naturally if the site supports it, and does nothing harmful if it doesn't.

Implementing this required classifying all ten themes as dark or light (only Glass White was light; everything else was dark), wiring an IPC channel so the renderer could notify the main process whenever the active theme changed, and applying the emulation both to brand-new tabs before their first paint and retroactively to already-open tabs — so switching themes updated live pages without forcing a reload.

## The Audit: Catching the Drift Between Spec and Reality

Before starting the packaging phase, there was a deliberate pause for an honest audit — comparing what the SRS actually promised against what had genuinely been built, file by file, rather than assuming completion just because a route or a README line existed.

The audit's findings were candid: the SRS described a considerably more ambitious system than what four phases had actually produced. A `time://calculator` with AST/Shunting-Yard parsing and arbitrary-precision arithmetic — never implemented. IndexedDB as the storage layer — the actual app used `localStorage` instead. A formal `EventBus` — not present in the form the SRS described. On the Phase 3 side, bookmarks lacked their hierarchical folder structure, Netscape HTML import/export, and drag-and-drop ordering; downloads lacked pause/resume/cancel controls, search/filtering, and storage analytics; there was no dedicated `time://` internal protocol handler at all.

Rather than treating this as a failure, it became a scoped follow-up task list, worked through in explicit, checkpointed parts: the `time://` protocol infrastructure first (registered as a privileged scheme, served via Electron's `protocol.handle` API), then download pause/resume/cancel controls wired through new IPC channels, then download search/filter/categorization/sorting and storage analytics, and finally — the largest single piece of remaining work — bookmarks.

## Rebuilding Bookmarks Properly

The existing bookmark system was a flat array — a real, working feature, but not the hierarchical, folder-based system the SRS called for. Rather than throwing it away, the new data model was designed for **safe migration**: a single flat array of nodes, where each node is either a folder or a bookmark, carrying a `parentId` (null for root-level items) and an `order` field for sequencing — a structure that could represent the old flat data as a degenerate special case and migrate it forward without data loss. Deleting a folder was deliberately made non-destructive: rather than recursively deleting its contents, children get reparented up to the folder's own parent, so nothing a person had bookmarked could vanish by accident. The old four-function API (add, remove, check, toggle) was kept backward-compatible so nothing calling into it elsewhere in the app needed to change.

Building the actual Bookmarks view surfaced two small but real bugs during self-review: leftover dead code in the breadcrumb renderer calling a malformed, fabricated ID (`folderId + '::self'`), and — more substantively — the breadcrumb trail never displayed the *current* folder's own name, only its ancestors, so navigating into "Work → Projects" would show "All Bookmarks / Work" and silently drop "Projects" from the trail. Both were caught, and the second was fixed as a small, surgical change rather than a rewrite: using the existing hierarchy APIs to correctly resolve and append the current folder to the breadcrumb, with an explicit instruction not to invent fake IDs to work around it.

## The Shippability Pivot

At a certain point, priorities shifted deliberately: rather than continuing to chase every feature the original SRS described, the goal became a **stable, genuinely shippable browser** — Browser, Tabs, Navigation, Bookmarks, Downloads, Dashboard, Settings, Themes, and the sidebar, all working cleanly. Features that were visibly exposed in the UI but not actually functional — Notes, Todo, Calendar, Calculator — had their navigation entries hidden rather than removed, preserving the underlying code and data models for future work without leaving broken-looking dead links in the shipped product. As one instruction in the conversation put it plainly: "Make the browser shippable, not make the roadmap infinite."

## Phase 4: From Source Code to a Real Installer

With the bookmarks integration checkpoint complete — wired into the router, added to the sidebar, verified against the `time://` protocol, and checked for migration data loss — the project moved to packaging.

The tool of choice was **electron-builder**, chosen over the alternative (Electron Forge) because it worked as a drop-in addition to the existing project structure rather than requiring any scaffolding changes, and it handles both macOS DMG and Windows NSIS installer output from a single configuration.

There was an immediate environmental constraint to work around: the development sandbox was a Linux VM with no display and no macOS tooling — `hdiutil`, needed for DMG creation, simply isn't available outside macOS, and even installing `electron-builder` itself was blocked because the sandbox had no npm registry access. So the scope of what could actually be done remotely was narrowed to what was safely possible: adding `electron-builder` as a devDependency, adding the `build`, `build:mac`, and `build:win` npm scripts, writing the electron-builder configuration block (app ID, product name, file inclusion rules, per-platform targets), and generating a `build/icon.png` — a single 1024×1024 source image that electron-builder can auto-derive both a macOS `.icns` and Windows `.ico` from, sidestepping the fact that `iconutil` (macOS-only) wasn't available either.

There was a minor but instructive hiccup here: after this configuration work was done, the person reported that their *local* copy of the project — downloaded earlier — didn't actually contain any of it, and `npm run` only listed the original `start` and `dev` scripts. The explanation was simple: the changes were real and present in the sandbox, but the ZIP the person had downloaded predated them. A fresh export was generated, explicitly re-validated (valid JSON, version still untouched at `0.0.7`, all 49 JS files unchanged, icon present) before being handed over again — a small reminder that "I made the change" and "you have the change" are two different claims that both need verifying.

## The Build, for Real

From here, the actual build had to happen on the person's own Mac, since that's the only place the necessary tooling existed. The sequence was simple:

```bash
cd TimeBrowser
npm install
npm run build:mac
```

The build output, pasted back into the conversation, showed a clean run: electron-builder 24.13.3, targeting `darwin x64`, downloading the matching Electron 31.7.7 binary, explicitly skipping code signing (`reason=identity explicitly is set to null` — exactly as intended, since real Apple code-signing requires a $99/year Developer ID and was deliberately deferred), and producing `dist/Time Browser-0.0.7.dmg` alongside its blockmap file and the unpacked `.app` bundle.

Because it was unsigned, launching it for the first time required the standard Gatekeeper workaround — right-click the app, choose **Open**, and confirm past the "unidentified developer" warning — rather than a plain double-click, which macOS would refuse outright. Once launched, real-world testing confirmed the whole stack actually worked end to end: the window opened, the sidebar and dashboard rendered, and — the part most worth stress-testing, since `WebContentsView` is native code that can behave differently once bundled into an `.asar` archive versus running unpacked via `npm start` — tabs could genuinely navigate to real websites.

The Windows build followed the identical pattern and didn't require a Windows machine at all — electron-builder cross-builds NSIS installers from macOS — just `npm run build:win`, producing `dist/Time Browser Setup 0.0.7.exe`.

## Epilogue: The Marketing Site

With a working, packaged application in hand, the final arc of the project shifted from engineering to presentation: a standalone, self-contained marketing and download website — the kind of page you land on after clicking "Download" on any real desktop app's homepage. It reused the same liquid-glass design tokens as the app itself for visual continuity, built a CSS-only floating browser-window mockup (no real screenshots needed) with a live-updating clock in its status bar, six feature cards describing only what had actually shipped (deliberately excluding Notes, Calculator, and the other hidden-but-unfinished apps), OS-aware download cards for macOS and Windows, a step-by-step macOS install guide that explained the Gatekeeper warning honestly rather than glossing over it, a documentation section linking out to the earlier SRS-turned-docs page, and an FAQ addressing the things people actually ask before installing an early-access app: does it need an account, does it collect telemetry, what platforms are supported.

The download buttons started life as clearly-marked placeholders — `href="#"` with a tooltip noting they needed real release URLs — specifically so nothing implied a working download existed before one actually did. Once the real `.dmg` (101MB) and `.exe` (84MB) files existed and were placed alongside the HTML, those placeholders were swapped for real relative links pointing straight at the installer filenames, turning the page from a mockup into an actually-functioning download destination.

## Closing Thought

What makes this project notable isn't any single feature — it's the discipline that ran underneath the whole thing. Nearly every message from the person carried the same refrain: *don't restart, don't rewrite what already works, resume from exactly where you stopped, validate before moving on.* That constraint shaped everything — the Registry pattern reused across themes, widgets, and commands instead of three bespoke systems; the audit that honestly flagged where the build had drifted from its own spec instead of pretending it hadn't; the bug fixes that stayed surgical (swap `navigationHistory.canGoBack()` for `webContents.canGoBack()`, add `min-height: 0` at each level of a specific CSS chain) instead of ballooning into rewrites. A month of incremental, checkpointed work turned a 6,000-line spec document into an actual, installable, working browser — and then into a working website to hand it out from.
