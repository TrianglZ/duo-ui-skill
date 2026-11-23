---
name: duo-ui
description: Convert any normal/standard iPhone screen — a Figma frame, a screenshot/mockup, or a SwiftUI/UIKit layout — into an iPhone Duo (Apple's foldable, 5.4" outer / 7.6" inner display) adapted design, producing correct outer-display and inner-display versions with vertical side toolbars, Arrangement-based split/overlay layouts, reserved-region and fold-safe spacing, and Dynamic-Type-safe reflow, per Apple's own iPhone Duo HIG and Tech Talks. Use this skill whenever the user asks to "make this a Duo screen," "convert/adapt/port this to iPhone Duo," "duo-ify this," mentions iPhone Duo, foldable iPhone, dual-display, inner/outer display, or hands you an existing iOS screen and asks how it should look or be rebuilt for the foldable form factor — even if they never say "iPhone Duo" by name and just describe a fold/dual-screen/outer-inner iPhone scenario.
---

# Duo UI — Convert Normal iPhone Screens to iPhone Duo

iPhone Duo is Apple's foldable iPhone (shipped Oct 23, iOS 27.1): a 5.4" outer
display that opens to a 7.6" inner display. This skill turns an existing, normal
single-screen iPhone design into a Duo-correct pair — **outer** and **inner**
versions of the same screen, built from the same hierarchy, never two unrelated
designs.

For exact numbers, API names, and the full source-grounded research behind every
rule below, see `references/apple-duo-hig-notes.md` — read it whenever you need to
double-check a detail rather than relying on memory, since Apple's own guidance is
the authority here, not this summary.

## The one rule everything else follows

**One adaptive app, two display sizes — never two apps.** Every control, every piece
of content, and the full navigation hierarchy must exist on both outer and inner.
The inner display isn't "the outer display, but bigger" — it's a genuinely different
size class (regular/regular vs. compact/regular) that earns the right to show more
at once, not just stretch what's already there.

## Step 1 — Identify what you're converting

- **Figma frame**: the common case in a Figma-based project. Load `figma-use` (and
  `figma-generate-design` if assembling from design-system components) before any
  `use_figma` call — this skill does not replace those, it tells you *what* Duo
  shape to build, they tell you *how* to execute in the Plugin API.
- **Screenshot / static mockup**: read the image, enumerate every UI element (nav
  bar, content, tab bar, FAB, etc.) before converting — you can't adapt what you
  haven't inventoried.
- **SwiftUI/UIKit code**: read the view hierarchy; conversion means replacing fixed
  layout with size-class-driven layout, not visually redrawing anything.

## Step 2 — Inventory the screen's elements by role

Before touching layout, sort every element on the source screen into these buckets —
the rest of this skill tells you what happens to each bucket:

1. **Navigation controls** — back/close, title, primary action (Done/Save), overflow
   menu.
2. **Tab bar** (if this screen owns one).
3. **Toolbar items** — icon-only actions, text-label actions, segmented controls.
4. **Primary content** — the thing the screen is actually about (list, detail, form,
   media, card grid).
5. **Secondary/contextual content** — anything that would be a natural "detail pane"
   companion to #4 (a list's selected item, a dashboard's drill-down, a composer next
   to a conversation).
6. **Transient surfaces** — sheets, popovers, alerts, menus.
7. **Full-bleed/background art** — hero images, gradient headers, non-interactive
   backgrounds.

## Step 3 — Build the outer-display version

The outer display is compact-width — treat this as "the normal iPhone screen,
corrected," not a from-scratch design:

- **Navigation controls → vertical side strip on the right edge**, symbol-only
  (SF Symbols, no text labels), fixed width/flexible height. Order top-to-bottom:
  back/close → prominent action (Done) → everything else in original grouping order
  with vertical spacing between groups.
- **Exception — keep these horizontal, do not move them to the side strip**:
  text-only buttons, segmented controls, anything that mixes symbol+text, complex
  custom toolbar views, keyboard accessory bars.
- **Tab bar stays bottom-aligned** — it does not move to the side on the outer
  display.
- **Primary content**: default to the *offset* approach — apply a horizontal
  safe-area inset so content clears the side strip automatically. Only go full-width
  if the content is immersive and non-scrolling (a static hero/background) with no
  interactive element near the side strip. For a photo/hero header, the common
  pattern is full-width background art + an inset, scrollable foreground — keep every
  *interactive* element inside the inset part.
- **Secondary/contextual content** from Step 2 bucket 5: does **not** appear here —
  push to it via normal navigation (push/present), same as a standard iPhone screen.
  The outer display shows one hierarchy level at a time.
- **Transient surfaces**: sheets/alerts/menus default to moving their controls into
  the side strip too; a single-button sheet may opt to keep a horizontal bar instead.
- **Typography**: keep the same type scale, but check Large/AX Dynamic Type sizes —
  the outer display is narrow enough that long labels must reflow (wrap, truncate
  gracefully, or drop to a secondary line), never clip or push off-screen.

## Step 4 — Build the inner-display version

The inner display is regular-width/regular-height — this is where the real
"Duo-ness" of the conversion happens. Pick exactly one of these three patterns for
this screen (don't mix patterns within one screen):

| Pattern | When to use | What changes |
|---|---|---|
| **Split View** | Bucket 4 (primary) has a natural bucket-5 companion (list↔detail, inbox↔composer, dashboard↔drill-down) | Show both simultaneously side-by-side via an **Arrangement (split style)** or `NavigationSplitView`/`UISplitViewController`. Preserve navigation history across fold/unfold automatically. |
| **Reflow** | No natural split exists, but the content is a single vertical stack that has room to breathe | The same vertical stack becomes a **two-column layout** (e.g. a form's fields, a settings list, a grid that was 2-wide becomes 3–4-wide) — same content, same order, just re-flowed into the extra width. |
| **Tab-as-Sidebar** | This screen owns the app's tab bar and the screen is information-dense | Convert the tab bar to a **sidebar** (`defaultTabBarPlacement(.sidebar)` / `tabBarController.sidebar`) instead of leaving it bottom-aligned. |

Inner-display specifics regardless of pattern:
- **All toolbar/nav items go back to standard horizontal bars** — the vertical side
  strip is an outer-display-only device. Portrait and landscape both use horizontal
  bars on the inner display.
- **Background/foreground layering (Overlay-style Arrangement)**: if bucket 7
  (full-bleed art) needs to coexist with bucket 4/5 content in a foreground/
  background relationship rather than a main/detail one, use an **Arrangement
  (overlay style)** instead of split — it stacks normally and goes side-by-side only
  when the device is physically folded.
- **Never nest an Arrangement inside a NavigationSplitView, List, or ScrollView** —
  pick one adaptive container per region, they don't compose.
- **Fold-safety**: any interactive element that could land near the physical hinge
  must displace toward the non-folded side (or to the bottom half, if the device is
  tabled). Scrolling content does not need this treatment. Displace the minimum
  distance necessary — don't throw an element across the whole screen just to clear
  the fold.
- **Reserved regions**: query them (outer camera / inner camera / fold) rather than
  hardcoding a margin — the active set changes with pose (e.g. inner camera only
  matters mid-video-call).

## Step 5 — Reconcile the two versions

Before calling the conversion done, check:

- Does every navigation destination reachable on the outer display also exist on the
  inner display, and vice versa? (No outer-only or inner-only features.)
- Is the *information hierarchy* identical even though the *layout* differs — i.e.
  would a user who only ever uses the outer display be equally capable as one who
  only ever uses the inner display?
- Did any element end up centered on the outer display instead of inset via safe
  area? (Common mistake — fix it.)
- Did any interactive control end up inside the fold region, or inside a camera
  reserved region?
- Does long/localized text or the largest Dynamic Type size still fit without
  clipping, on the outer display specifically (it's the tight constraint)?

## Step 6 — Produce the output

- **In Figma**: create (or duplicate into) two frames per screen, clearly named
  `<Screen name> – Outer` and `<Screen name> – Inner`, sized to the outer/inner
  display dimensions (see reference file for App Store screenshot dimensions as a
  proxy for aspect ratio if exact device frame isn't available as a preset). Reuse
  the project's real design-system components for every control — this skill decides
  *where* things go, the project's visual-system skill(s) decide *how they look*. If
  this is a Yoram project screen, also load the relevant `yoram-mobile-*` skill(s) so
  the converted frames stay on-system.
- **In code**: express the conversion as size-class-driven view composition (size
  class checks, `ReservedRegion`/`UIView.ReservedRegion` queries, Arrangement/
  NavigationSplitView containers) — never as a device-identity branch.
- Always state which of the three inner-display patterns (Split View / Reflow /
  Tab-as-Sidebar) you chose and why, and call out anything from Step 5's checklist
  that couldn't be fully resolved, rather than silently shipping a partial
  conversion.

## Quick reference — what moves vs. what stays

| Element | Outer display | Inner display |
|---|---|---|
| Nav/toolbar icon buttons | Vertical side strip (right edge) | Standard horizontal bar |
| Text buttons / segmented controls | Stay horizontal | Stay horizontal |
| Tab bar | Bottom-aligned | Bottom-aligned, *or* sidebar if info-dense |
| Primary content | Inset from side strip (or full-width if immersive/static) | Split/Reflow/Sidebar pattern — see Step 4 |
| Secondary/detail content | Separate screen (push/present) | Shown alongside primary (Split View) |
| Sheets/alerts | Controls move to side strip (or opt out for single-button) | Standard horizontal bar |
| Grid/list density | Same as a normal iPhone screen | Reflows to more columns/wider cards |
