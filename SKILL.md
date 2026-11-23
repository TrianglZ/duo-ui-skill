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

## Step 3 — Clone the source screens first, then convert the clones

Before any Outer/Inner editing starts, **clone the full set of source screens for
this flow/sub-flow into the target section as plain, unmodified duplicates** — don't
build the Outer/Inner structure "fresh" by hand, and don't transform a reference to
the original source node in place. Two-phase process, always in this order:

1. **Clone phase**: duplicate every screen this pass covers into the target
   section (matching the source's own nesting — a flat flow stays flat, a flow
   with nested sub-sections gets nested sub-sections, don't flatten it into
   text-labeled rows as a substitute for real structure). Re-read the result
   (node count/names) to confirm the clones actually landed before moving on —
   don't trust a script's own reported success.
2. **Convert phase**: only once the clones exist do you start transforming them
   into Outer/Inner pairs per Steps 4–7 below — duplicate each clone once more
   into its Outer and Inner copy, then edit those two in place.

This matters most on multi-screen flows: cloning the whole batch first means a
mid-build structural correction (e.g., discovering the grouping should be
SECTION-in-SECTION, not flat rows with labels) costs you a delete-and-reclone, not
hand-reconstructed content too — caught on a real build where a flat first attempt
had to be thrown away and rebuilt nested once the real source structure was checked.

**Critical gotcha when the target is a nested SECTION (SECTION-in-SECTION): child
coordinates are parent-relative, not page-absolute, and this is easy to get wrong
exactly once you nest two levels deep.** A section that's a *direct* child of the
page has `node.x`/`node.y` equal to its true page position (no parent offset to
account for), which can quietly condition you into treating `section.x` as "the
page-absolute left edge" everywhere you use it. That assumption silently breaks the
moment you create a SECTION *inside* another SECTION: the nested section's own
`x`/`y` — and everything positioned relative to it — are relative to **its own
parent section**, not the page. Positioning a nested section at `parentSection.x +
120` (treating `parentSection.x` as page-absolute, which it is) and then positioning
*its* children at `nestedSection.x + 60` (now wrongly treating the nested section's
own `.x` as page-absolute too) double-counts the offset — the nested section ends up
positioned at `parentSection.x` points to the *right* of where it should be, and
every screen inside it compounds a second time on top of that. The visible symptom
is exactly what it sounds like from the outside: the nested section's own container
looks empty, and the real content renders as a disconnected island far to the right,
which reads as "a huge gap" when you screenshot the page — caught on a real build via
a user-reported visual gap that `node.x` reads alone didn't reveal (`.x` was
internally self-consistent and looked fine); comparing `node.x` against
`node.absoluteBoundingBox.x` on the nested section is what actually exposed it; a
mismatch between those two numbers for a nested section is diagnostic of this bug.
**Before positioning a nested section's children, always compute the local offset
you actually want (e.g. `120`, not `parentSection.x + 120`)** — only a section that
is a *direct child of the page* gets to treat its own `.x` as page-absolute.

## Step 4 — Build the outer-display version

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
- **Transient surfaces (sheets/alerts/menus) — confirmed by Apple, not just inferred.**
  Apple's own quote: "By default, outer-display sheets present vertically, and
  inner-display sheets are centered" (the real APIs are `presentationPlacement(_:)` in
  SwiftUI / `preferredPlacement` in UIKit for the inner-display placement, and
  `toolbarVerticalBehavior(_:)` / `preferredVerticalBarBehavior` to opt a sheet's own
  toolbar out of the vertical treatment — these are real, documented APIs, cross-check
  before trusting a *different* set of API names from a casual source). Concretely:
  a sheet/modal's **own internal** close/cancel control — separate from the screen's
  own nav bar — also moves to a floating vertical position on Outer (don't just
  convert the screen-level nav and leave a sheet's internal horizontal header
  untouched — a real build did exactly that and missed it on a QR check-in bottom
  sheet until caught). Detach the control from the sheet's header row, float it
  above/beside the sheet over the dimmed scrim at the same side-margin used by other
  floating icon buttons, and left-align whatever remains in the header (same
  remove-the-balance-spacer pattern as any other icon extraction). A single-button
  sheet may still opt to keep a horizontal bar instead, matching `toolbarVerticalBehavior`.
- See **Step 5's "Sheets stay centered on Inner"** note below for how the same
  surface should look once widened.
- **Typography**: keep the same type scale, but check Large/AX Dynamic Type sizes —
  the outer display is narrow enough that long labels must reflow (wrap, truncate
  gracefully, or drop to a secondary line), never clip or push off-screen.

## Step 5 — Build the inner-display version

The inner display is regular-width/regular-height — this is where the real
"Duo-ness" of the conversion happens. Pick exactly one of these three patterns for
this screen (don't mix patterns within one screen):

| Pattern | When to use | What changes |
|---|---|---|
| **Split View** | Bucket 4 (primary) has a natural bucket-5 companion (list↔detail, inbox↔composer, dashboard↔drill-down) | Show both simultaneously side-by-side via an **Arrangement (split style)** or `NavigationSplitView`/`UISplitViewController`. Preserve navigation history across fold/unfold automatically. |
| **Reflow** | No natural split exists, but the content is a single vertical stack that has room to breathe | The same vertical stack becomes a **two-column layout** (e.g. a form's fields, a settings list, a grid that was 2-wide becomes 3–4-wide) — same content, same order, just re-flowed into the extra width. |
| **Tab-as-Sidebar** | This screen owns the app's tab bar and the screen is information-dense | Convert the tab bar to a **sidebar** (`defaultTabBarPlacement(.sidebar)` / `tabBarController.sidebar`) instead of leaving it bottom-aligned. |

**Tab-pair merge — when the source screen owns its own in-screen tab/segmented
control** (e.g. a "My shift / Event info" toggle that swaps the content below it —
not the app's bottom tab bar, that's the separate Tab-as-Sidebar case), that control
is itself the trigger for a specific Split View variant, and skipping this
recognition is a confirmed real-build failure mode (a from-scratch rebuild defaulted
to a plain single-source Split View and came out visibly worse than the flow that
originated this pattern — caught by the user comparing the two side by side). When
you see this control:

1. **Each tab state gets its own real Outer.** Don't synthesize a tab's content by
   grabbing just its content block — find or build that tab's own full source
   screen (Hero/Content/Sheet, the works) and convert it to a normal Outer, same as
   any other screen. If a later tab's content lives only as a shared panel reused
   elsewhere in the file (not duplicated per state), that's fine — reuse it, but
   still produce a full Outer conversion of it, not a bare content fragment.
2. **Row layout is `[Outer tab 1][Outer tab 2]…[Inner merged]`, left to right in tab
   order — never the default `[Outer][Inner]`.** The per-tab Outers function as
   reference columns: a reader can trace "this Outer's content → this half of the
   merged Inner" directly, left to right, without scrolling. Position them at the
   same standard offsets used elsewhere in the file for multi-column rows (e.g. 393pt
   Outer, ~40pt gap, next 393pt Outer, ~140pt gap, then the 760pt Inner) — don't
   collapse straight to a 2-column row just because the easier path only needed one
   source screen.
3. **Shared chrome renders once in the merged Inner — never once per column.** Any
   element identical across every tab (hero image, title block, sticky CTA bar, home
   indicator) stays outside the split row and renders a single time. Only each tab's
   *unique* content goes inside its own column.
4. **The tab control itself is deleted from the merged Inner, not kept per column.**
   It's redundant once both panels are visible at once — replace it with a plain
   text header per column (reusing each tab's own label, e.g. "My shift" / "Event
   info" as a static heading, not a segmented control). Two columns each still
   showing their own live segmented-control instance (one toggled left, one toggled
   right) is the single most common tell that this step was skipped — check for it
   specifically before calling an Inner done.

**Before splitting two things side-by-side, check whether either one is an
inherently wide, horizontally-scrolling component** (a chip/pill row, a card
carousel) that was sized to fill its *original* full-width container. Squeezing it
into a half-width split column doesn't make it responsive — the chips just overflow
and clip past the narrower edge, invisible unless someone scrolls, which reads as
broken on a static design surface. A row of N chips/pills needs roughly their
summed width (plus gaps) to display without clipping; if that exceeds the split
column's width, don't force the split — stack that section full-width instead
(this is just Reflow, not a failure — "Choose your role" as a full-width chip row
above "Choose a shift" as its own full-width section is a legitimate, deliberate
layout, not a fallback). Caught on a real build: a Role chip row forced into a
~348pt split column clipped its last chip; the fix was reverting that one pairing
to stacked full-width sections while keeping genuine Split View everywhere else it
still fit.

**A bottom footer with two or more stacked buttons (mobile-width forced them
vertical) should usually go side-by-side on Inner**, not stay stacked-but-centered
and not stretch full-width either — each button fills an equal share of the footer
row (primary filled, secondary ghost/outline, ordered the same as the mobile stack
top-to-bottom becomes leading-to-trailing). This earns the width without the
"stretched into an oversized pill" problem a single stretched button can avoid but
multiple stacked ones can't. Confirmed on a real build: a "View my request" +
"Back to home" footer, originally stacked because mobile had no room, became two
buttons side-by-side spanning the full inner width with standard margins — reads
as a deliberately-designed footer, not a padded leftover.

**MANDATORY, not optional: appending a button to an existing multi-button footer
must be verified with a genuinely separate follow-up read, every time — this exact
bug has slipped through on three different builds despite "fixing" it each time.**
Appending a new FILL button to an *already-existing* auto-layout bar (e.g. adding
"Leave event" next to a screen's own primary action) reliably balloons the bar to
the sum of the buttons' natural widths (a tell-tale wrong number like `1127` instead
of `760`, with buttons at a wrong share like `535.5` instead of `352`) — and setting
`resize()` + `primaryAxisSizingMode = 'FIXED'` + per-button `layoutSizingHorizontal =
'FILL'` *in the same script* that did the appending does not reliably prevent this,
even though reading the bar's width back at the end of that same script often
reports the correct number. The in-script report cannot be trusted here — it has
reported success while the real, settled value was still wrong. **The only
verification that has actually caught this: a brand-new, independent `use_figma`/
`figma_execute` call, after the build script has fully returned, that re-reads
`bar.width` and each button's width from scratch.** Do this for every screen where
a button was appended to a pre-existing bar (fresh bars you build from scratch, like
a lone "Leave event" bar with no pre-existing sibling, are not subject to this and
don't need the extra pass) — not just the first screen of a batch, every one,
because the bug has appeared on some screens in a batch and not others with no
predictable pattern. If the independent read shows the wrong width, fix it with yet
another separate script (re-apply `resize()` + `primaryAxisSizingMode = 'FIXED'`
immediately after the FILL-assignment loop, inside that new script, then it holds)
— don't just re-run the original build script, which reintroduces the same
in-script-report-lies-to-you problem. Treat "I called the fix function" and "I
confirmed the fix independently" as two separate, both-required steps — this is the
concrete instance of Rule 5 (figma-use skill) that's actually bitten this project
repeatedly, so don't let "the return value looked right" substitute for it.

**Don't double-margin a content block that already paddings itself.** A `Body`/
`Sheet`-style container is often already built with its own internal left/right
padding (e.g. 20pt each side — check `paddingLeft`/`paddingRight` before assuming).
If you then *also* shrink that block to a narrower width and center it in the wider
frame (to "not stretch it full width"), you've added a second margin on top of the
one it already has — the content ends up with double the intended margin and
visibly doesn't reach the screen edge the way every other full-bleed chrome row
does. The fix: resize that block to the **full** inner width and let its own
built-in padding be the only margin — don't add external centering on top of a
component that already insets itself. Caught on a real build (twice, independently,
across two different flows) after the user pointed at the gap and said the content
should fill the screen.

**Anti-pattern — don't default to "widen the frame, center the content."** Just adding side margins and centering the existing mobile layout is not one of the three patterns above, and it reads as mobile-design-with-padding, not a screen that earns the wider canvas — confirmed the hard way on a real build where exactly this got rejected on sight. Before falling back to a merely-wider centered column, actively look for *something* to pair it with: two sequential steps/pickers that could show together (Split View), a recap/summary that exists elsewhere in the same flow and could sit alongside this content for context (also Split View — reusing real existing content, not inventing new copy), or a list/grid that can reflow. Only use a wider centered column as the last resort when no real pairing or reflow exists at all, and even then prefer widening to a *comfortable reading measure*, not an arbitrary "same content, more margin" center job.

**Sheets stay centered on Inner — this is the one deliberate, Apple-confirmed exception to
the anti-pattern above, not a violation of it.** A sheet/modal (bottom sheet, form sheet,
action sheet) is not page content and doesn't need a Split View or Reflow pairing — Apple's
own default for the inner display is a centered sheet at its natural/default size over a
dimmed background, keeping its normal horizontal toolbar (Cancel/Close leading, Done
trailing), same as the closest documented analogue (iPadOS: "prefer the page or form sheet
presentation styles... centering its content on top of a dimmed background view"). Don't
stretch a sheet's content to fill the full inner width and don't force it into a split
column — resize only the scrim/background behind it to the full new width/height, keep the
sheet itself at its own natural width, and recenter it. (Apple does let a sheet dock to one
side instead — `presentationPlacement(.leading/.trailing)` — but centered is the default
and the safe choice absent a specific reason to dock it.)

**The sheet's own shape changes too, not just its position — it becomes a floating card,
not a bottom sheet.** A source screen's bottom sheet (flat bottom edge flush with the screen,
rounded top corners only, a drag grabber, anchored to the bottom) is a *compact-width*
shape — on Inner it should convert into a form-sheet-style floating card confirmed against
a real Apple reference screenshot (an "Animal Encounters" sample app sheet,
`presentationPlacement`-driven): **round all four corners** (not just the top two), **give it
margin on every side** — vertically centered or at least clearly off both the top and bottom
edges, never still flush to the bottom — and **remove the grabber** (a floating card isn't
drag-resizable the way a bottom sheet is; if the grabber stays, the shape reads as "a bottom
sheet that got recentered," not a real floating dialog). The close button floats
**overlapping the card's own top-right corner** (half on/half off the card, over the dimmed
scrim behind it) rather than sitting inline inside the card's header row — this is a
different detail from the *outer*-display vertical-sheet-control rule above (there the close
button floats separately beside the whole sheet; here it floats on the card's own corner).
Width stays at the sheet's own natural/compact width — don't stretch it just because there's
more canvas.

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

## Step 6 — Reconcile the two versions

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

## Frame height — hug content, but floor it at the real device height

A screen's frame should **hug its true content height** (a long scrolling screen
shows its real height, not get clipped to a fixed viewport) but never go *shorter*
than the real device's screen height — a sparse screen (an empty state, a short
confirmation screen) still needs to read as a full phone screen, not shrink-wrap
into something stubby that obviously isn't a real device.

Known real logical-point screen heights (derived from Apple's own App Store
screenshot pixel requirements — 1398×2034 outer / 2007×2853 inner, both @3x, see
the reference file): **Outer floor 678pt, Inner floor 951pt.** Use these as the
minimum regardless of what width convention the project itself uses (most existing
iOS screen libraries have their own established screen width, e.g. 393pt — don't
force the width to match 466/669, only floor the *height*).

- **If the screen's root frame is genuine Figma auto-layout** (`primaryAxisSizingMode:
  'AUTO'` and hugging): set `frame.minHeight = 678` (or `951`) directly — Figma's
  native hug-with-floor behavior does the rest.
- **If the screen's root frame is `layoutMode: 'NONE'`** (absolute-positioned
  children — common when the "screen" frame just contains a scrollable content
  child plus floating chrome like a tab bar or home indicator): `minHeight` does
  nothing on a NONE-layout frame, since it never auto-sizes in the first place.
  Compute the floor yourself: read the real settled height of the scrollable
  content child in a **separate** tool call (reading height in the same script
  that just changed the layout can return a stale pre-reflow value — a known Figma
  plugin-API gotcha), then `frame.resize(frame.width, Math.max(trueContentHeight,
  deviceMinHeight))`. Afterward, re-anchor any floating bottom chrome (tab bar,
  home indicator) using the screen's *own* original bottom-margin convention
  (measure it from an untouched source screen — don't assume a number), not
  whatever absolute position it happened to be left at.

## Step 7 — Produce the output

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
  Tab-as-Sidebar) you chose and why, and call out anything from Step 6's checklist
  that couldn't be fully resolved, rather than silently shipping a partial
  conversion.

## Step 8 — Mandatory verification checklist (run per screen, every time — not a one-off sanity check at the end)

This step exists because every item below is a *specific, already-happened* failure
on this project, each one previously treated as "fixed" after a single correction,
then repeated on a later screen or a later flow. None of these are optional, and
none of them are satisfied by a build script's own returned values — every one
requires a **separate, independent `use_figma`/`figma_execute` call issued after the
build script has fully returned**, not a check written into the same script that did
the building. If you skip the independent call and only trust the build script's own
report, you have not actually run this checklist, no matter how confident the report
looked.

1. **Sticky-bar / multi-button footer width.** Any screen where a button was
   *appended* to a pre-existing bar (not a bar built fresh from scratch): independently
   re-read `bar.width` and every button's width. Expect the bar at the Inner's full
   content width (e.g. 760) and buttons at an equal share — a bar width equal to the
   sum of the buttons' natural widths (plus padding/gap) means it reverted. Confirmed
   recurring on three separate builds despite an in-script fix each time.
2. **Orphan/duplicate nodes.** After any script that crashed, was retried, or ran a
   placeholder/test call against real IDs: list the target section's children by
   name and look for near-duplicate names or a frame positioned at an obviously
   wrong coordinate (e.g. `y: 0` when every real row starts past `y: 200`) — a
   crash-then-retry reuses the same staging input and silently leaves a half-built
   orphan behind.
3. **Nested-section coordinates.** Any time a SECTION is created *inside another
   SECTION* (not a direct child of the page): compare `nestedSection.x` against
   `nestedSection.absoluteBoundingBox.x`. If `absoluteBoundingBox.x` is larger by
   roughly the parent section's own `.x`, every child inside is double-offset — fix
   by computing the small intended local offset, not `parentSection.x + offset`.
4. **Shared-chrome duplication inside a merged Inner.** For every tab-pair-merge
   screen: confirm there is exactly **one** `Title Block` in the Inner's `Sheet` (not
   one per column) and **zero** live `Tabs` segmented-control instances (replaced by
   plain text headers). A screenshot showing two visually-identical header/tab
   rows side by side is the same miss every time.
5. **Label position validity.** After changing any label's font size (which changes
   its height): confirm `label.y` was recomputed from the new height against the row
   it sits above (`rowY - gap - newHeight`), not left at a stale offset computed for
   the old, shorter height — and confirm `label.x` actually matches the row's `x`,
   not a few points off from an earlier coordinate fix's rounding.

Do all five for every screen/row a build or fix pass touches — not just the first
one in a batch, and not just once at the end of a whole flow. A batch of otherwise-
identical screens has, in practice, had some pass and some fail the same check with
no predictable pattern, so partial-batch verification (spot-checking the first row
and assuming the rest match) is not sufficient.

## Quick reference — what moves vs. what stays

| Element | Outer display | Inner display |
|---|---|---|
| Nav/toolbar icon buttons | Vertical side strip (right edge) | Standard horizontal bar |
| Text buttons / segmented controls | Stay horizontal | Stay horizontal |
| Tab bar | Bottom-aligned | Bottom-aligned, *or* sidebar if info-dense |
| Primary content | Inset from side strip (or full-width if immersive/static) | Split/Reflow/Sidebar pattern — see Step 5 |
| Secondary/detail content | Separate screen (push/present) | Shown alongside primary (Split View) |
| Sheets/alerts | Controls move to side strip (or opt out for single-button) | Standard horizontal bar |
| Grid/list density | Same as a normal iPhone screen | Reflows to more columns/wider cards |
