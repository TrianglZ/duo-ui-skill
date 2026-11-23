# iPhone Duo — Research Notes (source-grounded)

Raw reference material collected from Apple's own Tech Talks/HIG and design-community
analysis (Nov 2026). `SKILL.md` is the actionable workflow; this file is the lookup
reference for exact numbers, API names, and rules when you need to double-check a
detail mid-conversion. iPhone Duo shipped **October 23, on iOS 27.1** — it is Apple's
first foldable iPhone, not Duo Mobile (2FA) or Duolingo.

## Device facts

| Property | Value |
|---|---|
| Outer display (closed) | 5.4" — roughly 90% of the screen area of a standard flagship iPhone when closed |
| Inner display (open) | 7.6" |
| Closed-shape | Passport-style, compact |
| Minimum SDK to build against | iOS 27 (iOS 27.1 for full-screen/extended-edge behavior) |
| App Store screenshot sizes | 1398×2034px (outer), 2007×2853px (inner) |
| Screenshot requirement deadline | Mandatory from **April 2027** |
| Poses | Flat (fully open, tablet-like), partially folded / book mode, tent/edge-standing, closed |

## Size classes (the only two you design for)

Do **not** design per-pose. Apple's own guidance: focus on two size classes and let
Auto Layout / SwiftUI handle the continuum in between.

| Display | Portrait | Landscape |
|---|---|---|
| Outer | Regular height / **Compact width** | Compact / Compact |
| Inner | **Regular / Regular** (behaves like a small iPad; supports sidebars) | Regular / Regular |

Inner display does **not** honor "supported interface orientations" the way a phone
screen traditionally does — build adaptive, not orientation-locked, layouts.

```swift
// SwiftUI
@Environment(\.horizontalSizeClass) private var horizontalSizeClass
@Environment(\.verticalSizeClass) private var verticalSizeClass

// UIKit
traitCollection.horizontalSizeClass
traitCollection.verticalSizeClass
```

Never branch on device identity (`if iPhone Duo`) or `UIScreen.main` — both are
explicitly called out by Apple as anti-patterns. Use `window?.windowScene?.screen` or
`traitCollection.displayScale` instead.

## Component placement rules

**Outer display, vertical side bar (SDK opt-in):**
- Lives on the **right edge**. Fixed width, flexible height — built for symbol-only
  icons (SF Symbols), not long text labels.
- Order top → bottom: primary nav (Back/Close) → prominent action (Done) → everything
  else in its original grouping, with vertical spacing between groups.
- **Stays horizontal, never moves to the side bar:** text-only buttons, segmented
  controls, items that switch between symbol and text, complex custom toolbar views,
  keyboard accessory bars, and — specifically — **inner-display portrait bars** ("the
  only pose where we've kept horizontal bars").
- Tab bars stay **bottom-aligned** on the outer display (they do not move to the side).

**Inner display:**
- Standard horizontal bars in both landscape and portrait.
- Tab bar can instead be presented as a **sidebar** — best for information-dense apps
  (Apple's own example: Health).

```swift
// SwiftUI
TabView { … }.defaultTabBarPlacement(.sidebar)

// UIKit
tabBarController.sidebar.preferredPlacement = .sidebar
```

**Split-view multitasking (50/50, two apps side by side or two windows of one app):**
- Controls move to the **outer edge** of whichever half the app occupies (app on the
  left → its controls sit on the left), staying in thumb reach and away from the
  shared center divider.
- Tapping the divider between two apps pairs/unpairs them.

**Sheets:**
- Outer display: default behavior moves sheet controls to the side bar; can opt out
  to keep controls in a horizontal bar (best for a single-button sheet). Sheets stop
  just short of the front camera; the status bar repositions around it.
- Inner display: sheets always use standard horizontal bars.
- When the device is partially folded, sheets **slide to avoid resting in the fold**.

## New layout containers: Arrangements

A new container type for laying out exactly two views across the fold.

- **Split style** — lays out horizontally when the available space is wider than
  tall, vertically otherwise. Use this where you'd otherwise reach for `HStack`/
  `VStack` to express a **main–detail** relationship.
- **Overlay style** — stacks vertically in the normal case, goes side-by-side when
  folded. Use this where you'd reach for `ZStack` to express a
  **foreground/background** relationship.
- **Never** nest an Arrangement inside a `NavigationSplitView`, a `List`, or a
  `ScrollView` — those containers already manage their own adaptive behavior and the
  two systems fight each other.

## Three honest patterns for the inner display

Pick per-screen, not per-app:

1. **Split View** — surface multiple hierarchy levels at once (Mail's classic
   list + message pattern). Use `NavigationSplitView` (SwiftUI) /
   `UISplitViewController` (UIKit) — both preserve navigation history automatically
   across fold/unfold.
2. **Reflow** — a vertically-stacked layout becomes a two-column layout once there's
   more horizontal space (Apple's own example: Music).
3. **Tab-as-Sidebar** — the tab bar converts into a sidebar for information-dense
   apps.

## Safe areas, reserved regions, and the fold

**Reserved regions** (new in iOS 27.1 — `ReservedRegion` in SwiftUI,
`UIView.ReservedRegion` in UIKit): outer camera, inner camera (active during video),
and the fold (active only while partially closed). System components (sheets,
alerts, menus, popovers) avoid these automatically; **custom** UI must query the
region APIs and displace itself manually.

**Displacement rule:** *interactive* elements move away from the fold — toward the
non-folded side in book pose, to the bottom half on a table (so touch targets stay
reachable and stable). *Scrolling content* does **not** need to avoid the fold.
Apple's framing: "moving an element far away from its source weakens their visual
relationship" — so displace the minimum distance needed, not maximally.

**Safe-area math** — the outer display's insets are asymmetric (a control strip on
one side only), so never assume symmetric insets:

```swift
// ❌ wrong — assumes insets are symmetric
let width = view.bounds.width - view.safeAreaInsets.left * 2

// ✅ correct
let width = view.bounds.inset(by: view.safeAreaInsets).width
```

**Foreground vs. background content:**
- Foreground/interactive content → stays **inside** the safe area.
- Full-bleed background art/imagery → can `.ignoresSafeArea()` and extend under the
  camera/fold regions, since it's non-interactive.
- A common mixed pattern: full-width background header image, with a scrollable
  foreground content column that stays inset — keep **all interactive elements**
  inside the inset, scrollable part.

**Corner-matching (Concentricity APIs, iOS 26+):** keeps edge-to-edge shapes' corner
radii matched to the device's actual physical screen corners.

```swift
// SwiftUI
ConcentricRectangle().fill(Color.green).padding(8.0).ignoresSafeArea()

// UIKit
UICornerConfiguration
```

## Content alignment on the outer display — three approaches

1. **Offset (default/recommended):** use horizontal safe-area insets so content
   automatically avoids the side control strip.
2. **Full-width:** reserved for immersive, non-scrolling layouts (e.g. a static hero/
   background) where no interactive element conflicts with the Dynamic Island, status
   bar, or side controls.
3. **Mixed/layered:** full-width background + inset scrollable foreground — the most
   common real-world pattern for content-forward screens (event/profile headers,
   editorial cards).

## Typography & visual language

- Standard Dynamic Type scale still applies: SF Pro, Body = 17pt, Large Title = 34pt
  at the default size.
- The outer display is narrow enough that **large Dynamic Type sizes can push labels
  off-screen** — forms and multi-step flows must **reflow**, never truncate, at AX
  sizes.
- Spacing convention: **8pt grid with 4pt subdivisions** (unchanged from standard
  iOS — no new grid system for Duo).
- Visual language is **Liquid Glass** (iOS 26+): translucency, reflection, refraction,
  fluid transitions. Controls are visually separated from content via depth/contrast/
  material rather than opaque chrome. In practice, many Liquid Glass buttons get
  pulled fully out of the content area into the side control strip, leaving the
  content area quiet (typography and controls recede so the actual product/media is
  the visual event).

## Picture-in-picture

Pinned video resizes the host app vertically in real time as the fold angle changes;
when partially folded, video can extend to fill one half of the screen.

## Testing matrix (minimum coverage before calling a screen "Duo-ready")

| Dimension | Options to cover |
|---|---|
| Display | Outer, Inner |
| Orientation | Portrait, Landscape |
| Pose | Flat, ~45° fold (book), tent, closed |
| Dynamic Type | Default, XL, AX (largest) |
| Multitasking | Full screen, 50/50 split view |
| Other states | Keyboard visible, empty state, loading, error, long/localized text |

Also check: Auto Layout / constraint conflicts, clipped labels, overlapping controls
near tab/toolbar edges, hand-occlusion near the vertical side rail, fold-transition
animation smoothness (ideally on physical hardware, not just simulator).

## Hard anti-patterns (called out explicitly by Apple / practitioners)

- Fixed widths, breakpoints, or layout metrics tied to a specific screen size.
- Centering content on the outer display instead of using safe-area insets.
- Custom bespoke layouts per pose (reserve pose-specific layouts *only* for
  deliberate hands-free table/tent optimization — and even then, preserve every
  control and the full information hierarchy).
- Interactive elements sitting inside the fold region.
- Device-identity checks (`if iPhone Duo`) instead of size-class / trait-collection
  logic.
- Simply stretching an existing single-screen iPhone layout onto the inner display
  without rethinking information density — the inner display is a genuine second
  size class, not "the same screen but bigger."
- Treating the outer display as a "glance-only" secondary screen — it's ~90% the
  area of a normal iPhone and must support full primary-flow usage on its own.

## Tooling / dev support

- **Xcode 27.1 + Device Hub**: iPhone Duo simulator with on-screen pose controls
  (open/close/rotate/fold); preview the inner display and drag an app into Split
  View to test vertical-UI-on-either-side behavior.
- **App Resizability Skill** (Xcode 27.1): automated modernization/audit tool that
  checks SwiftUI layouts against adaptive-layout best practices for iPhone Duo.
- Figma/Sketch design kits are published through Apple Design Resources
  (developer.apple.com/design/resources).

## Sources

- [Design for iPhone Duo — Apple Developer Tech Talks](https://developer.apple.com/videos/play/tech-talks/111466/)
- [Prepare your app for iPhone Duo — Apple Developer Tech Talks](https://developer.apple.com/videos/play/tech-talks/111461/)
- [Designing for iPhone Duo: What Moves, Splits, and Stays](https://blakecrosley.com/blog/designing-for-iphone-duo)
- [iPhone Duo UX Design: How Apple Redesigned Mobile Interfaces](https://designsuite.ai/iphone-duo-ux-design-the-new-mobile-experience-apple-created/)
- [Responsive iOS App Design for iPhone Duo: UX, Layouts & Testing](https://softdoes.com/insights/responsive-ios-app-design-for-iphone-duo)
- [iPhone Duo App Redesign Checklist: 7 UX Checks](https://www.orizon.co/blog/iphone-duo-app-redesign-checklist)
- [iPhone Duo UI and UX: A Masterclass in Small Details](https://www.gadgetpilipinas.net/2026/10/iphone-duo-ui-ux-masterclass/)

These facts are current as of Nov 2026 research and describe a product that shipped
weeks before this file was written — re-verify against current Apple HIG
(`developer.apple.com/design/human-interface-guidelines/designing-for-iphone-duo`) if
this file is being read long after that date, since Apple may revise guidance.
