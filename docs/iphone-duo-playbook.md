# iPhone Duo App Playbook

Reference for designing and building apps that take full advantage of the iPhone Duo. Compiled November 4, 2026.

**Source note:** apple.com, developer.apple.com and most news sites could not be fetched from the research environment. The facts below come from search-result summaries of Apple's Newsroom and Support pages, plus GitHub sample code that was tested against the iOS 27.1 SDK. Confirm exact API signatures in the Xcode 27.1 SDK headers before shipping. Items marked **(unverified)** come from a single third-party source.

---

## 1. The hardware

| Item | Detail |
|---|---|
| Launch | Announced Sept 9, 2026; released Oct 23, 2026 |
| Form | Book-style fold ("folds like a passport") |
| Outer (cover) display | 5.4 in |
| Inner display | 7.6 in, about 50% larger than the iPhone 18 Pro Max |
| Folded size | 84.1 × 117.8 × 11.3 mm |
| Unfolded size | 164.6 × 117.8 × 5.2 mm (the thinnest iPhone) |
| Chip | A20 Pro, with a custom vapor chamber |
| Sensors | A sensor in each half; iOS uses them to detect posture and lying flat |
| Apple Pencil | Pairs with Apple Pencil (USB-C) |
| OS | iOS 27.1+ (the new APIs need the iOS 27.1 SDK and Xcode 27.1) |
| Storage | 256 GB / 512 GB / 1 TB / 2 TB |

## 2. The forms (postures) to design for

| Posture | How it is held | Hinge status | Good uses |
|---|---|---|---|
| **Closed** | One-handed, outer 5.4" display | `.closed` | Quick tasks, glances, notifications, one-handed use |
| **Fully open (flat)** | Two-handed tablet, 7.6" inner display | `.fullyOpen` (≈180°) | Split View, list + detail, canvas, video, reading, Pencil |
| **Book** | Partly open, held like a book, inner display in landscape | `.partiallyOpen` | Two-page reading, side-by-side compare, chat + content |
| **Tabletop / laptop** | Partly open, bottom half flat on a table | `.partiallyOpen` | Video and FaceTime on top, controls on the bottom; hands-free camera |
| **Tent** | Partly open, standing like a tent, outer display facing out | `.partiallyOpen` | StandBy, clocks, timers, showing something to a person across the table |
| **Flat on table** | Fully open, lying down | detected by the sensors | Viewing without a stand; shared viewing |
| **Rear-camera selfie** | Rear cameras face the subject, outer display faces the subject | n/a | High-quality selfies; the subject sees a live preview |

The crease reserved region (40 pt wide with 20 pt margins on each side) is **active only when the phone is partially folded.** When flat it is inactive, but you can still read its frame. Taps near the crease register less reliably, and iOS moves its own content away from it in partial folds.

## 3. What Apple ships (system patterns worth matching)

- **Split View on iPhone, a first.** Two apps, or two windows of the same app, side by side on the inner display. A half can show the Home Screen while you pick the second app. App pairs can be saved. The limit is two apps.
- **Vertical bar.** On both displays, iOS can move toolbars and tab bars to a vertical side bar. The trailing safe-area inset is 84 pt and the leading inset is 0, so safe areas are asymmetric.
- **Camera:**
  - **Duo Preview:** a live preview on the outer display so the subject can check framing.
  - **Smart Take:** set the phone on a table, start a session, and it captures automatically when it detects that you are posing or holding still.
  - **Kid Cue:** plays animations on the outer display to get a child's attention.
- **Duo FaceTime:** people standing near you can join the call through the outer display.
- **StandBy:** works in the partly folded tent posture.
- **Legacy apps:** apps built with the iOS 26 SDK or earlier show wide bars on both sides of the inner display. The inner display does not honor supported interface orientations, so a portrait-only lock will not protect a fixed layout.
- **"Made for iPhone Duo" badge:** appears on App Store product pages for optimized apps, which also get editorial collections.

Apps Apple highlighted at launch: Instagram Edits, VLLO and LTK (capture and edit), Zoom, Slack, Fantastical, Lyft, Detail, Netflix.

## 4. Developer API map (iOS 27.1 SDK)

| Need | SwiftUI | UIKit |
|---|---|---|
| Hinge status and angle | `onHingeChange(isEnabled:_:)` → `DeviceHinge` (`status`, `angle` in degrees, 180° = flat) | `UIHingeInteraction`, `UIHinge` (`status`, `angle` in **radians**; `Status` also has `.unknown`) |
| Crease and camera cut-outs | `GeometryProxy.reservedRegions(kind: .division / .occlusion, options: [.includeInactive])` → `frame`, `margins`, `isActive` | `UIView.reservedRegions(kind:)` |
| Two panes that follow the fold | `ArrangementView { primary } secondary: { … }` with `.arrangementViewStyle(.split / .overlay / .automatic)` or a custom `ArrangementViewStyle` | `UIArrangementViewController` with `UISplitArrangement` / `UIOverlayArrangement` |
| Pane sizing | `splitArrangementLayoutRatio(minHorizontal:idealHorizontal:)`, `splitArrangementLayoutSize`, `splitArrangementFixedLayoutSize`, `overlayArrangementEdge(_:)` | — |
| Vertical toolbars and tab bars | `toolbarVerticalBehavior(_:)`, `toolbarVerticalCompressionBehavior(.prefersTabBar)`, `@Environment(\.toolbarVerticalEdge)`, `.axisBehavior(.verticalPreferred)`, `.visibilityPriority(_:)`, `ToolbarItem(placement: .topBarPinnedTrailing)`, `ToolbarOverflowMenu` | `UITraitCollection.verticalBarEdge` |
| Content on the outer display while the camera runs inside | `.sceneAccessory { CameraCaptureAccessory(isEnabled:) { SubjectView() }.onAvailabilityChange { … } }`; `ExternalNonInteractiveAccessory` for non-interactive content | `UISceneAccessory.cameraCapture(sceneConfiguration:userInfo:)` → `UISceneAccessoryRegistration` |
| Camera direction | Direction coordinator; the change handler receives an `AVCaptureDeviceDirectionMap` of forward- and backward-facing devices **(unverified)** | same |
| Margins | `contentMargins(for: .container)`, `GeometryProxy.contentMargins(for:)` | — |
| Testing | Xcode 27.1 **Device Hub**: Closed / Partially Open / Open; hold Option and drag the slider to set any angle | same |

Behavior found by testing the samples:
- `DeviceHinge` is `nil` outside a hierarchy that provides updates, and `DeviceHinge.Status` is a struct, not an enum.
- Reserved regions appear only after the first layout pass. Read them in `body` and do not cache them.
- The outer display has **no** reserved regions, not even inactive ones.
- A split arrangement folds into a stack on the outer display, and shows only its primary view when the phone is closed and held in landscape.
- Avoid `maxHorizontal` in a split ratio, because it stops the divider from following the fold. A fixed pane size also moves the divider off the fold.
- On iPhone, `openWindow` opens full screen, not in Split View.
- Hinge update timing and precision are set by the system. If you only need the posture, use `status`.

Cross-platform: Expo Hinge (React Native); `foldable`, `duo_kit` and `nitro_fold_duo` (Flutter).

## 5. Hard rules

1. Build against the iOS 27.1 SDK, or the app is letterboxed.
2. Choose layout from **size class** (outer display = compact width, inner display = regular width) and available space (`onGeometryChange`), never from device model, orientation, idiom or `UIScreen.main`.
3. Use **posture** (`status`) and size class for layout, and the **raw hinge angle** only for effects and interactions.
4. Keep tappable controls, text and faces out of the active crease region. A region's frame already includes its margins.
5. Treat safe areas as asymmetric. Do not write `width - inset.left * 2`.
6. Keep state continuous. Folding or unfolding must not lose scroll position, drafts, playback or selection.
7. Support two windows of your own app, and share their state through one `@Observable` model.
8. Test every posture, plus Split View halves, in Device Hub.

## 6. Creative patterns (idea bank)

**Layouts that use both halves**
- **List | detail** that collapses to one navigation stack on the outer display.
- **Companion pane:** the main content plus a live helper (notes, AI chat, chart, transcript).
- **Extended canvas:** one continuous surface, such as a map, whiteboard or timeline, with a gutter over an even-column grid at the inactive fold.
- **Overlay panel:** search results floating over a map when flat, side by side when in book posture.
- **Two-page spread:** books, PDFs, plan sets and comics, one page per side when in book posture.

**Posture as an input**
- **Tabletop media:** the picture on the top half, the scrubber and controls on the bottom.
- **Laptop mode:** content on top, a custom keyboard, trackpad, mixer or game controller on the bottom.
- **Hinge as a control:** the angle drives a dial, zoom, a timeline scrub, a 3D page turn, a book flip or a reveal animation. Example: DuoAngle is a hinge protractor with haptic alerts at target angles.
- **Page-turn flick:** a quick partial close and reopen acts as a gesture (HingeKit detects this).
- **Open-to-expand:** start a task on the outer display, then unfold to continue with more detail, such as a notification becoming a full thread or a map pin becoming the full route planner.
- **Close-to-glance:** folding switches to a compact live view, such as a timer, a ride ETA or now playing.

**Two audiences, two screens**
- **Presenter / audience:** you see notes and controls inside while the other person sees the content on the outer display, using the scene accessory pattern.
- **Face-to-face tools:** live translation or captions, a customer-facing checkout or signature, a quiz host and a player, a board game with two sides.
- **Camera for the subject:** a framing preview, pose guides, a countdown or kid-attention animations on the outer display; Smart Take-style automatic capture in tabletop posture.
- **Tent displays:** a meeting timer, a name card, a scoreboard, a clock, a menu or a status sign facing the room.

**Productivity**
- **Pencil on the inner display:** markup, signing and sketching; two-up compare with annotations on one side.
- **Drag and drop between halves** of your own app, or across Split View.
- **Field and inspection tools:** a photo on one half and a checklist or form on the other. Plans or a map on one side and notes on the other.
- **Dashboards:** dense when unfolded, reduced to key numbers on the outer display.

**Games**
- Playfield on top, controls on the bottom in tabletop posture. Keep touch controls out of the crease.
- Two-player head-to-head across the halves when flat.
- The hinge as a game mechanic, such as a slingshot, a drawbridge or a pop-up book.

## 7. Web apps (ondaxa.com and PWAs)

There is no web hinge API in these sources. On the web, use what is available:
- Responsive CSS breakpoints based on **container or viewport width**, not device detection. The inner display gets the wide layout.
- `env(safe-area-inset-*)` with `viewport-fit=cover`. Expect the inset on one side only.
- Keep content continuous on resize (fold and unfold fire a resize event), and keep form state.
- Check Safari on iPhone Duo for CSS Viewport Segments (`horizontal-viewport-segments` / `env(viewport-segment-*)`) support before relying on it **(not confirmed for iPhone Duo)**.

## 8. Pre-release checklist

- [ ] Built with Xcode 27.1 / iOS 27.1 SDK; no letterboxing
- [ ] Closed, open, book, tabletop, tent and landscape-closed tested in Device Hub
- [ ] Split View: left half, right half, and two windows of the same app
- [ ] No controls or key content in the active crease
- [ ] Vertical bar placement and toolbar overflow priorities are correct
- [ ] Asymmetric safe areas are handled
- [ ] Fold and unfold keep all state
- [ ] Outer display used where it adds value (camera subject, presenter, tent)
- [ ] Hinge angle used only for effects, not layout
- [ ] Apple Pencil works on the inner display (if relevant)

## Sources

- Apple Newsroom: [Apple unveils iPhone Duo](https://www.apple.com/newsroom/2026/09/apple-unveils-iphone-duo/), [Open up to new possibilities](https://www.apple.com/newsroom/2026/10/open-up-to-new-possibilities-with-iphone-duo/), [Developers bring new app experiences](https://www.apple.com/newsroom/2026/10/developers-bring-incredible-new-app-experiences-to-iphone-duo/)
- Apple: [iPhone Duo](https://www.apple.com/iphone-duo/), [Tech Specs](https://www.apple.com/iphone-duo/specs/), [Support: Pose iPhone Duo](https://support.apple.com/guide/iphone/pose-iphone-duo-iphv3p93k1bm/ios), [Support: What's new with iPhone Duo](https://support.apple.com/en-by/guide/iphone/whats-new-with-iphone-duo-iphge85m9f19/27/ios/27)
- [MacRumors: How iOS 27 adapts to iPhone Duo](https://www.macrumors.com/2026/09/10/apple-details-how-ios-27-adapts-to-iphone-duo/), [MacRumors roundup](https://www.macrumors.com/roundup/iphone-duo/), [9to5Mac: Split screen](https://9to5mac.com/2026/09/12/apple-nailed-iphone-duo-split-screen-even-as-ipad-multitasking-feels-convoluted/), [heise: Multitasking](https://www.heise.de/en/news/iPhone-Duo-How-to-use-multitasking-mode-11503361.html), [CNN review](https://www.cnn.com/2026/10/21/tech/apple-iphone-duo-review), [T-Mobile listing](https://www.t-mobile.com/cell-phone/apple-iphone-duo)
- GitHub: [awesome-iphone-duo](https://github.com/MIDJUG3Nine/awesome-iphone-duo), [iPhone-Duo-by-Examples](https://github.com/artemnovichkov/iPhone-Duo-by-Examples), [iphone-duo-skills](https://github.com/mirzaaghazadeh/iphone-duo-skills)
