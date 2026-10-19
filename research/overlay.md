# Notchlight research — native overlay design and idle cost on macOS 27

Ticket: [What native overlay design can stay idle cheaply?](https://github.com/mujak27/notchlight/issues/4) (research; evidence and method only, no budget — a budget is a human decision).
Host under study: personal Mac Apple M5 Pro 16", macOS 27.0.1 (build 26A434), Mac17,8, arm64, 24 GiB. One host; portability not required.
Scope of this report: the **overlay surface** (the thing that draws beside/over the notch and stays alive at idle). It does **not** cover where notification or media data comes from — that is tickets 01/02/04/05.
Constraints assumed: no SIP disabling, no system modification, no new permission prompts for this research, no code executed, no app prototype. **No measured numbers are stated anywhere below; every "cheap"/"expensive" statement is a design argument or an Apple-documented expectation, not a benchmark.**
Method: read-only primary-source research against Apple's own documentation, support pages, and the archived *Energy Efficiency Guide for Mac Apps*. Each claim is cited to the URL that owns it. Where Apple does not document a behaviour, it is flagged as an unknown requiring an on-host test rather than asserted.
Written: 2026-10-19. Research branch: `research/overlay` (based on `origin/main`, `bacc646`).

---

## 0. Headline

**Candidate, not conclusion:** a notch overlay *surface* appears buildable from public AppKit alone — an event-driven **borderless, non-activating `NSPanel`** with no private API and, in principle, no repeating idle timer. This is a **presentation-layer candidate only.** It assumes nothing about the notification/media *ingestion* paths (independently researched in tickets 01/02/04/05), and framework overhead (AppKit or SwiftUI) is not established here. **Whether it stays visible over a native full-screen app is U1, unverified** (see §3). **Actual idle CPU/energy is unmeasured**; measurements are explicitly out of scope for this ticket, and the §7 workload is a *method* to obtain them.

Apple's "zero energy impact" is a **guidance target, not a physical-zero guarantee** — it says an app that isn't being interacted with should report no energy impact [S38], not that an always-visible window consumes nothing. Avoid designs that wake the app on a timer while idle; do not treat "no timer" as proven, and do not treat the panel as free.

Three things the documentation does **not** settle, and which therefore remain on-host tests, not assumptions:

| Question | Why it is unresolved | Where it goes |
|---|---|---|
| Does a `.canJoinAllSpaces` non-activating panel actually stay visible over a **native full-screen** app on macOS 27? | Apple documents `canJoinAllSpaces` and `fullScreenAuxiliary` separately and never states their combined effect on this OS version. | §3, unknown U1 |
| Which window **level** is correct (`.statusBar` vs `.floating` vs above menu bar)? | Apple lists the levels but gives no overlay guidance; a level that is too high risks drawing over the login window / lock screen. | §2, decision D2 |
| Does SwiftUI-on-`NSHostingView` cost measurable idle wakeups compared with a pure-AppKit `NSView`? | No Apple document quantifies this, and benchmarking is explicitly out of scope here. | §6, unknown U5 |

The rest of this report gives the evidence for each part of that **candidate**, the ways it can still fail, and a measurement workload.

---

## 1. Notch geometry and safe area

### 1.1 What Apple exposes

- `NSScreen.safeAreaInsets` (macOS 12.0+): "The distances from the screen's edges at which content isn't obscured… **On some Macs, the insets reflect the portion of the screen covered by the camera housing.** If your app offers a custom full-screen experience, apply the specified insets to the screen's frame rectangle to obtain the area within which it is safe to display your content." [S1]
- `NSScreen.auxiliaryTopLeftArea` / `auxiliaryTopRightArea` (macOS 12.0+): "The unobscured portion of the top-left [/right] corner of the screen." Discussion: "**If the top inset of the screen's `safeAreaInsets` property contains a non-zero value, the rectangle in this property is the visible top-left portion of the screen.** The rectangle is specified in **global screen coordinates** and lies **outside the safe area**… **If the top portion of the screen isn't obscured, the value of this property is `nil`** in Swift; in Objective-C, the value is `NSZeroRect`." [S2][S3]
- `NSScreen.frame` and `NSScreen.visibleFrame` (the latter excludes the menu bar / Dock), plus `NSScreen.screens` and `NSScreen.main`. [S4]

### 1.2 Consequences for the island (evidence-backed)

1. **Notch detection is public and explicit.** `safeAreaInsets.top != 0` plus a non-nil `auxiliaryTopLeftArea` is the documented signal that a camera housing exists and where the two "wings" beside it are. No private API and no hard-coded notch size is needed. [S1][S2][S3]
2. **The two auxiliary rectangles are the natural anchoring geometry.** They are already in global screen coordinates and are guaranteed unobscured, so the *compact* island can be laid out inside `auxiliaryTopLeftArea`/`auxiliaryTopRightArea` and the *expanded* island below the notch, using `safeAreaInsets` to avoid underlapping the housing. [S1][S2][S3]
3. **The island surface itself can sit over the housing**, because it is an app-drawn window, not app content inside a full-screen space: the safe-area rules in [S1] govern *where the system places your content*, not where a borderless overlay window may be positioned. (Inference from [S1] + [S5]; low risk, but the exact pixel alignment should be confirmed visually on the host — see U2.)
4. **Menu-bar height should be read, not assumed.** The screen containing the menu bar is always index `0` of `NSScreen.screens`, and `NSScreen.main` is *not necessarily* that screen — it is the screen with keyboard focus. [S4] So, on a multi-display setup, the "menu-bar screen" must be picked by index/`frame`, not by `NSScreen.main`.
5. `NSWindow.screen` reports the screen the window is mostly on and is `nil` while the window is offscreen [S18] — relevant when re-anchoring after a display change (§3).

**Straightforward:** geometry is fully supported publicly. **Unknown U2:** exact sub-pixel placement over the housing (does the OS clip anything at the very top edge?) is not documented; verify by screenshot on the host.

---

## 2. Panel, window, and app lifecycle

### 2.1 Panel versus window

Apple: `NSPanel` is "A special kind of window that typically performs a function that is auxiliary to the main window" [S5]; `NSWindow.StyleMask.nonactivatingPanel` = "The window is a panel or a subclass of `NSPanel` that does not activate the owning app." [S8] This is exactly the property an island needs: clicking it must not steal focus from the app underneath.

`NSPanel.isFloatingPanel` guidance is directly relevant and, read literally, argues *against* leaning on standard panel behaviour: a panel should float above other windows only if it is small, is "oriented more to the mouse than to the keyboard", "doesn't become the key window or becomes so only when needed", needs to stay visible while the user works in other windows, and — the default — "**it hides when the app is deactivated (the default behavior for panels)**." [S6] A notch island wants to float and to *not* hide on deactivate, so it will deliberately depart from two of these defaults.

Relevant defaults Apple documents:

| Property | Default | Island likely wants | Citation |
|---|---|---|---|
| `NSWindow.hidesOnDeactivate` | `false` for `NSWindow`, **`true` for `NSPanel`** | `false` (island must persist when another app is frontmost) | [S9] |
| `NSPanel.becomesKeyOnlyIfNeeded` | `false` | `true` (island is mouse-oriented, not keyboard-first) | [S7] |
| `NSPanel.isFloatingPanel` | `false` | `true` (must sit above normal windows) | [S6] |
| `NSView.needsPanelToBecomeKey` | `false` | leave `false` unless a text field is added — "If the panel is a non-activating panel, then it becomes key only if the hit view returns `true` from `needsPanelToBecomeKey`." | [S7][S40] |
| `NSWindow.ignoresMouseEvents` | `false` | keep `false` by default so hover-to-expand works; `true` only if the compact pill is deliberately made click-through (see the note below) | [S41] |
| `NSWindow.canBecomeVisibleWithoutLogin` | `false` | leave `false` — documented only for the **login window**; it is *not* a documented guarantee about the modern Lock Screen (see U3) | [S15] |

> **Hover vs click-through is a real conflict, not a free choice.** `ignoresMouseEvents = true` makes the window "transparent to mouse events" [S41], which is exactly what would stop pointer enter/exit reaching the pill — so the compact state cannot both be click-through *and* expand on hover. Two coherent options, left to the human:
> - **(a)** keep `ignoresMouseEvents = false` and expand on hover (`NSTrackingArea` pointer enter/exit [S19]); the cost is that the compact pill **occludes clicks** in the (small) region it covers.
> - **(b)** make the compact state click-through and use **no hover** (or a hover mechanism that does not depend on the window receiving events — this has *not* been validated here and needs independent investigation).
>
> Do not prescribe `ignoresMouseEvents = true` as a ready design: as written it contradicts hover-to-expand. This is **U8** in §8.

### 2.2 Stacking level

`NSWindow.Level` lists, from lowest to highest: `normal`, `floating`, `modalPanel`, `mainMenu`, `statusBar`, `popUpMenu`, `submenu`, `tornOffMenu`, `screenSaver`. [S10] An island that must sit beside/over the menu bar area implies a level around `mainMenu`/`statusBar`. Apple gives no overlay-specific guidance; the concern with a very high level (`.screenSaver`) is that it may draw over the login/lock surface, but Apple documents the *login window* specifically and nothing here establishes the modern Lock Screen — treat that as **U3**, not a proven exclusion. [S10][S15] **This is decision D2, not a finding.**

### 2.3 App lifecycle: accessory / agent

- `NSApplication.ActivationPolicy.accessory`: "The application doesn't appear in the Dock and doesn't have a menu bar, but it **may be activated programmatically or by clicking on one of its windows**… corresponds to `LSUIElement` being `1`." [S12]
- `LSUIElement`: "A Boolean value indicating whether the app is an **agent app that runs in the background** and doesn't appear in the Dock." [S13]

So an agent/accessory app is the documented shape for a resident overlay. Two consequences to plan for: with no menu bar there must be another entry point (the island itself, or a status item — `NSStatusBar` explicitly warns status items are "not guaranteed to be available at all times" and should be sparing [S42]); and `NSApplication.presentationOptions` (hide Dock / auto-hide menu bar) is documented as being for fullscreen apps such as games or kiosks and has strict mutual-exclusion rules [S14] — it is **not** the right lever here; do not hide the system menu bar.

**Launch behaviour:** the map has not decided launch-at-login; note only that whatever mechanism is chosen must not require the island to be in the Dock (accessory policy already removes it). Not researched further here.

---

## 3. Fullscreen, Spaces, and external displays

The relevant control is `NSWindow.collectionBehavior`, "Window collection behaviors related to Mission Control, Spaces, and Stage Manager." Apple states the mutual-exclusion rule explicitly: "**primary, auxiliary, and canJoinAllApplications only apply to full screen and Stage Manager. They're also mutually exclusive. Specify at most one per window.**" [S11]

Behaviours documented (each with its own page):

| Behaviour | Apple's wording | Use for the island |
|---|---|---|
| `canJoinAllSpaces` (10.5+) | "The window can appear in all spaces." Discussion: "**The menu bar behaves this way.**" | candidate — "menu-bar-like" is the intent, not a verified outcome (U1) [S11] |
| `stationary` (10.6+) | "Mission Control doesn't affect the window, so it stays visible and stationary, like the desktop window." | candidate — it should not fly around in Mission Control (U1) [S11] |
| `fullScreenAuxiliary` (10.7+) | "The window displays on the same space as the full screen window." | candidate (intent only) — "displays on the same space" is Apple's wording, not a verified over-full-screen overlay (U1) [S11] |
| `fullScreenNone` (10.7+) | "The window doesn't support full-screen mode." | not relevant (island never goes full-screen) [S11] |
| `auxiliary` (13.0+) | "marking this window as auxiliary… becomes auxiliary for both Stage Manager and full screen… Auxiliary windows prefer being shown alongside primary windows… Use this collection behavior for About or Settings windows as well as utility panes." Mutually exclusive with `primary`/`canJoinAllApplications`. | candidate for Stage Manager coexistence [S11] |
| `transient` / `managed` | Spaces + Mission Control variants | not needed [S11] |
| `ignoresCycle` | excludes the window from Cmd-\` cycling | yes — avoid the island appearing in the app/window cycle [S11] |

**The concrete unknown (U1):** Apple documents `canJoinAllSpaces` and `fullScreenAuxiliary` as independent behaviours and never states their **combined** effect on macOS 27 — e.g. whether a joined window is drawn over a native full-screen app on the built-in display. The combination `[.canJoinAllSpaces, .stationary, .fullScreenAuxiliary]` is the documented-intent answer, but it must be **observed**, and the outcome may differ between the built-in notch display and an external display (see below).

**Screen and Space changes are events, not polls** (see §4):

- `NSApplication.didChangeScreenParametersNotification` — "Posted when the configuration of the displays attached to the computer is changed… The system posts this notification on the main actor." [S16]
- `NSWorkspace.activeSpaceDidChangeNotification` — "A notification that the workspace posts when a Spaces change occurs." Apple adds an important wiring note: "**To receive this notification, use `notificationCenter` to register for it. If you use a different notification center to register, you won't receive the notification.**" [S17]

**External displays.** `NSScreen.screens` returns all screens and screen index `0` holds the menu bar; `NSScreen.main` follows keyboard focus, not the menu bar. [S4] The map's only host is the built-in display of a Mac17,8, so the practical rule is: *find the screen whose `safeAreaInsets.top` is non-zero and whose `auxiliaryTopLeftArea` is non-nil; if none exists (external/clamshell), the island has no notch to anchor to and its behaviour must be decided* — that behaviour is **not specified by any Apple document and is a product decision (D3)**, not a technical finding. Also note the mirror case: a window parked at fixed global coordinates on a notch display can end up off-screen when that display is disconnected, which is what `NSWindow.screen == nil` reflects [S18] — so the anchor must be recomputed on `didChangeScreenParametersNotification` [S16].

---

## 4. Event-driven vs polling, and the platform's energy guidance

### 4.1 Apple states the preference plainly

- *Reducing your app's battery use*: "Do less work… Work more efficiently… Avoid misusing APIs", and "Consider whether your app can update its state less frequently. Look for opportunities to remove activity altogether." [S26]
- *Scheduling CPU work efficiently*: "**Prefer an event-driven design, in which your app's code receives notifications when it needs to perform work, instead of polling for available tasks.**" It also advises caching results, reusing structures, and using the **lowest QoS that makes sense** because "the system uses more energy-efficient scheduling for tasks with lower QoS". [S20]
- The archived guide's stated idle expectation, which is the closest Apple comes to a target: for the Xcode **Energy impact gauge**, "**When users aren't interacting with your app, it should have zero energy impact.**" and, for the CPU gauge, "Spikes when your app is supposed to have low CPU activity or when it is idle may indicate problem areas." [S38]

### 4.2 Where a timer *is* justified, and how to make it cheap

Apple does not say "never use timers". It says make them deferrable and tolerant:

- `Timer.tolerance` (macOS 10.9+): "Setting a tolerance for a timer allows it to fire later than the scheduled fire date. **Allowing the system flexibility in when a timer fires increases the ability of the system to optimize for increased power savings** and responsiveness… The system reserves the right to apply a small amount of tolerance to certain timers regardless of the value of this property." Default is `0` (no tolerance). [S21]
- `NSBackgroundActivityScheduler` (macOS 10.10+): "similar to a `Timer`… however, `NSBackgroundActivityScheduler` gives the system flexibility to determine the most efficient time to execute based on energy usage, thermal conditions, and CPU use." Apple's list of intended uses includes "**activities occurring in intervals of 10 minutes or more**" — that is **usage guidance for the right tool, not an enforced API minimum**; it also exposes `shouldDefer` for circumstances like "the user has unplugged the Mac". [S22]
- `CADisplayLink` (macOS 14.0+): the correct driver for *animations that are actually running* — "a timer object that allows your app to synchronize its drawing to the refresh rate of the display". Frame rates are constrained: "actual frame rates are always a factor of the maximum refresh rate" and can be forced down by "system policy and user preference — including Low Power Mode, critical thermal state, and accessibility settings"; `isPaused` stops notifications and `invalidate()` tears the link down. [S35]

### 4.3 The island's effects, classified

| Effect | Event path (preferred) | Timer needed? |
|---|---|---|
| Island appears/relocates when a display is attached, removed, or rearranged | `NSApplication.didChangeScreenParametersNotification` [S16] | no |
| Island stays put when the user switches Spaces / enters Mission Control | `NSWindow.collectionBehavior` [S11] + `activeSpaceDidChangeNotification` if a redraw is needed [S17] | no |
| Compact ↔ expanded on hover | `NSTrackingArea` (AppKit) [S19] / SwiftUI `onHover` — pointer enter/exit events | no |
| Notification arrives, is dismissed, changes | the capture layer's own events (tickets 01/04) — AXObserver, if that is the route | no |
| Now-playing changes (track, play/pause, artwork) | the media layer's own events (tickets 02/05) | no |
| Light/dark and Low Power Mode adaptation | `NSProcessInfoPowerStateDidChange` / `thermalStateDidChangeNotification` [S23][S24][S25] | no |
| **Optional** progress position of a playing track | advance it locally from the last known position + timestamp (interpolation) rather than re-querying the source; a repeating display update is still needed to redraw it | **yes for redraw; this is a display-refresh cost, not necessarily data-source polling** |

That last row is a real design fork, not an implementation detail, and it must not be read as "the only place anything polls":

- **Progress is a display-refresh cost first, a data question second.** Drawing a moving bar needs a periodic *redraw* (e.g. `TimelineView` [S34] / `CADisplayLink` [S35]); it does **not** require repeatedly polling the media source — the position can be interpolated from the last event's timestamp. A source that only emits coarse or infrequent events may need an occasional resync, but that is a property of the *chosen adapter*, not of the overlay.
- **The overlay cannot promise "no polling anywhere."** Whether the media or notification adapters poll is decided in tickets 01/02/04/05; if an adapter is event-driven this row stays a display-only cost, and if it is not, the overlay inherits its cadence. This report only removes the *overlay's own* need for a repeating timer in a no-progress compact state.
- **The non-progress compact indicator** ("track + transport buttons, no live progress") needs no periodic redraw at all; adding live progress is what forces one. **Decision D4.**

### 4.4 Animation lifecycle

- *Improving your app's rendering efficiency*: "**Limit the frame rate and duration of animations.** Use animations to communicate changes and updates in your app, **and pause or stop them when the update completes.**" Higher frame rates cost more power. It also notes that Liquid Glass effects cost extra system work, and advises updating "the smallest region that needs updating" via `setNeedsDisplay(_:)`. [S27]
- *Understanding and improving SwiftUI performance*: the system "requires views to update before the system renders the next display frame"; long bodies or frequent updates "reduce overall system efficiency". It names the Instruments **SwiftUI** template and its **Update Groups** / **Hitches** lanes for finding these. [S28]
- `NSAnimationContext` exists for AppKit-driven animations; `CADisplayLink` for custom ones [S35][S36]. The island's expand/collapse should be a finite animation that stops, never an animation that runs while idle, and the tiered content (compact → expanded) is the natural place to add nothing else. [S27]

---

## 5. `LSUIElement` and Accessibility permissions

- **Agent app:** `LSUIElement = 1` ⇔ `ActivationPolicy.accessory`; no Dock icon, no menu bar. [S12][S13]
- **Accessibility (AX) permission** — needed only if the notification/media layers use the Accessibility API (which ticket 01's report found is the demonstrated public route for notification mirroring):
  - `AXIsProcessTrusted()` — "Returns whether the current process is a trusted accessibility client." [S37]
  - `AXIsProcessTrustedWithOptions(_:)` — same, plus `kAXTrustedCheckOptionPrompt`, "`ACFBooleanRef` indicating whether the user will be informed if the current process is untrusted… **Prompting occurs asynchronously and does not affect the return value.**" [S37]
  - Permission is a per-app user grant in Privacy & Security; the island cannot self-grant, and the ticket's constraint set forbids triggering prompts during this research. Plan for the *revoked-mid-session* case (ticket 01 §7 covers the capture side).
- **Overlay permissions:** the surface itself (a borderless panel that only draws and takes pointer events) does **not** need Screen Recording, Accessibility, or Full Disk Access. Keep that separation — the overlay must remain functional-as-a-surface even if AX is denied, otherwise a permission failure blanks the island. (Design argument from [S12][S26]; no Apple doc states it.)

---

## 6. SwiftUI trade-offs (observation, not a benchmark)

No benchmark was run and none is implied. What Apple actually documents:

- **`NSHostingView` is the sanctioned bridge** for putting SwiftUI into an AppKit hierarchy: "You use `NSHostingView` objects to integrate SwiftUI views into your AppKit view hierarchies… an `NSView` object that manages a single SwiftUI view". It exposes `safeAreaRegions` and `sizingOptions` for layout, and forwards AppKit events. [S31] So `NSPanel` + `NSHostingView` is a documented combination, and the AppKit panel carries all the windowing behaviour in §2–§4 regardless of how the content is drawn.
- **SwiftUI can be made cheap but is not free by construction.** Apple's own performance article says "views that update too frequently… consume resources", recommends caching expensive work out of `body`, and provides the Instruments SwiftUI template to find long or frequent updates. [S28] That is a *measurement obligation*, not a verdict.
- **Two SwiftUI affordances map onto the island's needs:** `TimelineView` for any scheduled redraw (with the caveat in §4.3) [S34], and `Canvas` for "immediate mode drawing" which "**might provide better performance for a complex drawing that involves dynamic data**" — but Canvas "doesn't offer interactivity or accessibility for individual elements". [S32] `drawingGroup()` composites into an offscreen image first. [S33]
- **Reduce-motion must be honoured**: animations should respect the accessibility reduce-motion setting; the environment value exists (`accessibilityReduceMotion`) and the rendering-efficiency guidance tells you to minimise animations generally. [S27][S43]

**Honest position:** for a surface this small (a compact pill plus an expanded panel), the choice between a pure-AppKit `NSView` and `NSHostingView`-hosted SwiftUI is a **maintainability decision, not a settled performance one**. Neither option has been shown to be quiet at idle here: whether SwiftUI invalidates views only on real state changes, and whether either emits idle wakeups, is **unknown U5** until measured (§7). Do not assume a static window is free — measure it.

---

## 7. Proposed validation workload and tools (method only)

The goal is to let the human *later* measure idle CPU, battery/energy, wakeups, and memory — with no pre-agreed numbers and no numbers invented here. Every tool below is first-party.

### 7.1 Tools and what each is authoritative for

| Tool | What it reports | Citation |
|---|---|---|
| Xcode **debug gauges → Energy Impact** | app-level energy impact while running under Xcode; "when users aren't interacting… it should have zero energy impact" | [S38][S29] |
| Xcode debug gauges → **CPU**, **Disk**, **Memory** | CPU spikes at idle, recurring small I/O, footprint | [S38] |
| Instruments **SwiftUI** template | view-update frequency/length, Update Groups, Hitches (if SwiftUI content is used) | [S28] |
| Instruments **Time Profiler** | where CPU time goes during any wake | [S38][S28] |
| Instruments **Allocations / Leaks** | memory growth over the workload | [S29] |
| Activity Monitor **Energy** pane | `Energy Impact`, `12 hr Power`, `App Nap`, `Preventing Sleep`, `Graphics Card` | [S39][S38] |
| Activity Monitor **CPU** pane | "% CPU… When your app isn't active, the number should be 0.0" | [S38] |
| `powermetrics` | CPU time split by QoS ("helpful for ensuring that quality of service classes are properly implemented") | [S38] |
| `dtrace`, `fs_usage` | kernel/low-level and file-system activity during idle | [S38] |
| `pmset -g` | whether "any power assertions are in place" (is the app preventing sleep?) | [S38] |
| `XCTCPUMetric`, `XCTMemoryMetric` in an XCTest performance test | regression guard: CPU time/cycles/instructions, and memory delta before/after a measured block | [S44][S45] |
| MetricKit | power/battery metrics for a shipped build | [S29] |
| `NSProcessInfo.thermalState` / `isLowPowerModeEnabled` | correlating a run with thermal/Low-Power conditions | [S24][S25] |

Caveats to record with any future results: Apple says the Instruments **Power Profiler** instrument is documented for **iPhone/iPad (iOS 26+)** and that "power impact metric values are different for different device models, so you can't compare values… recorded on different models" [S30] — on macOS the energy signal to trust is the Xcode Energy Impact gauge [S38][S29] plus Activity Monitor's Energy pane [S39], and the machine-specific units must never be compared across machines.

### 7.2 Deterministic workload (states the human should measure separately)

Each state is measured over a **fixed idle window chosen by the human** (the duration is a method parameter, not a budget), on AC power and again on battery:

1. **Cold launch / idle, nothing playing.** Island at its compact resting state, no media, no notifications. This is the baseline the ticket cares about most.
2. **Idle with media playing.** Compact indicator live, transport idle, **no live progress** (then repeat with a progress bar if D4 chooses one — this isolates the periodic-redraw cost).
3. **Hover expand / collapse.** Repeated N times; measure the animation's cost and confirm it stops.
4. **Notification arrival then dismissal.** Requires the capture layer; measure the island's incremental wake, not the capture layer's.
5. **Full-screen app frontmost, built-in display.** The U1 test: is the island still drawn, still above, still not stealing focus?
6. **Space switches and Mission Control.** Island must not vanish or fly around ([S11]); watch for a redraw storm on `activeSpaceDidChangeNotification` [S17].
7. **External display connect/disconnect and clamshell.** Confirm re-anchoring and that nothing is left at stale global coordinates (`NSWindow.screen == nil` case) [S16][S18].
8. **Sleep → wake; lock → unlock.** Confirm the island returns, and separately confirm whether it appears on the **login window** vs the modern **Lock Screen** (only the former is documented via `canBecomeVisibleWithoutLogin` — U3) [S15].
9. **Low Power Mode on/off and a thermally stressed run.** The island should reduce animation/updates rather than keep its cost [S23][S24][S25].
10. **Permission revoked mid-session** (AX denied while the island is up, from ticket 01 §7) — the overlay should stay alive and quiet.

### 7.3 Observations to capture per state

- CPU % at idle (the archived guide's own standard: it should be ~0 when the app isn't active [S38]).
- Energy Impact / `12 hr Power` trend [S39].
- Wakeups: compare `powermetrics`/`dtrace` output between an "island running" run and a "island not running" run of the same idle window [S38].
- Power assertions in `pmset -g` [S38].
- Memory: Activity Monitor footprint + Instruments Allocations growth over a long idle [S29][S39].
- Whether App Nap engages while the island is not frontmost, and whether that is acceptable for an always-visible overlay [S39].

### 7.4 Regression harness (for later, if the human approves one)

An XCTest performance test that measures a fixed block with `XCTCPUMetric` + `XCTMemoryMetric` [S44][S45] is **one** regression tool — convenient because it reports relative deltas and so needs no approved absolute budget — **not the only way** to guard against a "someone added a timer" regression, and not a complete one: CPU metrics for a bounded block measure work *inside that block* and may not surface idle wakeups or energy spent between test invocations, which are exactly the failure mode of interest here. Use it alongside the always-on idle observations in §7.3 (Activity Monitor, `powermetrics`/`dtrace` wakeup comparison), not instead of them.

---

## 8. Unknowns that require an on-host test

- **U1.** Combined effect of `canJoinAllSpaces` + `fullScreenAuxiliary` over a native full-screen app on the built-in display, on macOS 27. [S11]
- **U2.** Exact overlay placement over the camera housing (clipping, safe-area interaction at the very top edge), verified by screenshot. [S1][S2][S3]
- **U3.** Whether the island is genuinely absent from the lock/login surface (the `canBecomeVisibleWithoutLogin == false` default is documented for the *login window* [S15], but the modern Lock Screen is not documented in these terms).
- **U4.** Stage Manager behaviour (`.auxiliary` vs `.primary` vs `.canJoinAllApplications` and their mutual exclusion) [S11] — the host may or may not use Stage Manager.
- **U5.** Whether SwiftUI-on-`NSHostingView` costs measurable idle wakeups vs a pure-AppKit view on this host. [S28][S31]
- **U6.** Whether macOS 27 introduces any new overlay/notch-specific API or deprecation not visible in the documentation set read here. Apple's Activity Monitor guide *does* list **macOS 27 "Golden Gate"** as a version (so the OS is real and shipping) [S39], but no macOS 27 doc consulted here changed any of the APIs in §1–§4. Treat "nothing new" as unverified: check the macOS 27 AppKit/Foundation release notes on the actual SDK before implementing.
- **U7.** Cost of a continuous progress bar, if D4 chooses one; and whether the chosen media adapter is event-driven or polled (which decides whether progress adds data polling on top of display refresh). [S34][S21]
- **U8.** A hover-to-expand mechanism that coexists with a click-through compact pill — or a decision to keep the pill interactive and accept that it occludes clicks in its region. `ignoresMouseEvents = true` and hover-from-pointer-events are mutually exclusive as documented; if a click-through compact state is required, the hover trigger must be validated independently (no such mechanism was established here). [S19][S41]

---

## 9. Decisions for the human

- **D1 — Overlay content technology:** pure-AppKit `NSView` (fewest moving parts) vs `NSHostingView`-hosted SwiftUI (faster to build the expanded panel) — a maintainability choice; U5 can inform it but not settle it. [S31][S28]
- **D2 — Window level:** `mainMenu`/`statusBar` vs `floating` vs custom; and a rule that the island is never at or above `.screenSaver` to avoid the login surface (only the *login window* is documented here; the modern Lock Screen is U3). [S10][S15]
- **D3 — External-display behaviour:** when no notched screen is present, does the island hide, move to a status item, or anchor to an external display's menu bar? Not specified by Apple. [S4]
- **D4 — Progress:** compact indicator with no live progress (no periodic redraw) vs with live progress (accepts a periodic display refresh; whether it also adds source polling depends on the adapter — U7). [S34][S21]
- **D5 — Budget:** explicitly *not* set here. Any numeric idle CPU / battery / memory budget remains the human's decision (tickets 04–06). [S38]

---

## 10. Sources

All sources are Apple first-party. Numbering continues the convention of the sibling reports (`notifications.md`, `media.md`) but is self-contained for this file.

1. Apple, `NSScreen.safeAreaInsets` (macOS 12.0+) — camera-housing wording and "apply the specified insets to the screen's frame rectangle". https://developer.apple.com/documentation/appkit/nsscreen/safeareainsets
2. Apple, `NSScreen.auxiliaryTopLeftArea` (macOS 12.0+) — "unobscured portion of the top-left corner", global screen coordinates, outside the safe area, `nil` when the top isn't obscured. https://developer.apple.com/documentation/appkit/nsscreen/auxiliarytopleftarea-uglc
3. Apple, `NSScreen.auxiliaryTopRightArea` (macOS 12.0+). https://developer.apple.com/documentation/appkit/nsscreen/auxiliarytoprightarea-gr2n
4. Apple, `NSScreen` — `frame`, `visibleFrame`, `screens` (menu-bar screen is index 0), `main` (screen with keyboard focus). https://developer.apple.com/documentation/appkit/nsscreen
5. Apple, `NSPanel` — "auxiliary to the main window". https://developer.apple.com/documentation/appkit/nspanel
6. Apple, `NSPanel.isFloatingPanel` — when a panel should float; "hides when the app is deactivated (the default behavior for panels)". https://developer.apple.com/documentation/appkit/nspanel/isfloatingpanel
7. Apple, `NSPanel.becomesKeyOnlyIfNeeded` — non-activating panels and `needsPanelToBecomeKey`. https://developer.apple.com/documentation/appkit/nspanel/becomeskeyonlyifneeded
8. Apple, `NSWindow.StyleMask.nonactivatingPanel` — "does not activate the owning app". https://developer.apple.com/documentation/appkit/nswindow/stylemask-swift.struct/nonactivatingpanel
9. Apple, `NSWindow.hidesOnDeactivate` — default `false` for `NSWindow`, `true` for `NSPanel`. https://developer.apple.com/documentation/appkit/nswindow/hidesondeactivate
10. Apple, `NSWindow.Level` — documented stacking levels `normal`…`screenSaver`. https://developer.apple.com/documentation/appkit/nswindow/level-swift.struct
11. Apple, `NSWindow.CollectionBehavior` and members — mutual exclusion of `primary`/`auxiliary`/`canJoinAllApplications`; `canJoinAllSpaces` ("The menu bar behaves this way"), `stationary`, `fullScreenAuxiliary`, `fullScreenNone`, `transient`, `managed`, `ignoresCycle`. https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct and https://developer.apple.com/documentation/appkit/nswindow/collectionbehavior-swift.struct/canjoinallspaces and .../stationary and .../fullscreenauxiliary and .../auxiliary
12. Apple, `NSApplication.ActivationPolicy.accessory` — no Dock, no menu bar, "corresponds to `LSUIElement` being `1`". https://developer.apple.com/documentation/appkit/nsapplication/activationpolicy-swift.enum/accessory
13. Apple, `LSUIElement` — "agent app that runs in the background and doesn't appear in the Dock". https://developer.apple.com/documentation/bundleresources/information-property-list/lsuielement
14. Apple, `NSApplication.PresentationOptions` — intended "for fullscreen apps such as games or kiosks"; mutual-exclusion rules; not an overlay lever. https://developer.apple.com/documentation/appkit/nsapplication/presentationoptions-swift.struct
15. Apple, `NSWindow.canBecomeVisibleWithoutLogin` — default `false` (login-window visibility). https://developer.apple.com/documentation/appkit/nswindow/canbecomevisiblewithoutlogin
16. Apple, `NSApplication.didChangeScreenParametersNotification` — posted on the main actor. https://developer.apple.com/documentation/appkit/nsapplication/didchangescreenparametersnotification
17. Apple, `NSWorkspace.activeSpaceDidChangeNotification` — "use `notificationCenter` to register… otherwise you won't receive the notification". https://developer.apple.com/documentation/appkit/nsworkspace/activespacedidchangenotification
18. Apple, `NSWindow.screen` — screen the window is mostly on; `nil` when offscreen. https://developer.apple.com/documentation/appkit/nswindow/screen
19. Apple, `NSTrackingArea` — pointer enter/exit events (AppKit hover). https://developer.apple.com/documentation/appkit/nstrackingarea
20. Apple, *Scheduling CPU work efficiently* — "Prefer an event-driven design… instead of polling"; lowest viable QoS; 10–100 ms concurrent granularity; BackgroundTasks. https://developer.apple.com/documentation/xcode/scheduling-cpu-work-efficiently
21. Apple, `Timer.tolerance` — tolerance increases the system's ability to save power; default `0`. https://developer.apple.com/documentation/foundation/timer/tolerance
22. Apple, `NSBackgroundActivityScheduler` — deferrable tasks, ≥10-minute intervals, `shouldDefer`. https://developer.apple.com/documentation/foundation/nsbackgroundactivityscheduler
23. Apple, *Responding to power notifications* — Low Power Mode and thermal-state reactions ("Reducing display updates", "Minimizing animations"). https://developer.apple.com/documentation/xcode/responding-to-power-notifications
24. Apple, `ProcessInfo.isLowPowerModeEnabled` (macOS 12.0+) and `NSProcessInfoPowerStateDidChange`. https://developer.apple.com/documentation/foundation/processinfo/islowpowermodeenabled
25. Apple, `ProcessInfo.thermalState` (macOS 10.10.3+) and `thermalStateDidChangeNotification`. https://developer.apple.com/documentation/foundation/processinfo/thermalstate-swift.property
26. Apple, *Reducing your app's battery use* — three-stage process; "update its state less frequently"; "remove activity altogether". https://developer.apple.com/documentation/xcode/reducing-your-app-s-battery-use
27. Apple, *Improving your app's rendering efficiency* — "Limit the frame rate and duration of animations… pause or stop them when the update completes"; `setNeedsDisplay(_:)` smallest region; Liquid Glass cost. https://developer.apple.com/documentation/xcode/improving-your-app-s-rendering-efficiency
28. Apple, *Understanding and improving SwiftUI performance* — view-update frequency/length, Instruments SwiftUI template, Update Groups, Hitches. https://developer.apple.com/documentation/xcode/understanding-and-improving-swiftui-performance
29. Apple, *Analyzing your app's battery use* — Xcode Energy Impact gauge, Metrics, `XCTCPUMetric`, MetricKit, Instruments Power Profiler. https://developer.apple.com/documentation/xcode/analyzing-your-app-s-battery-use
30. Apple, *Measuring your app's power use with Power Profiler* — Power Profiler is documented for iPhone/iPad (iOS 26+); cross-model metric values are not comparable. https://developer.apple.com/documentation/xcode/measuring-your-app-s-power-use-with-power-profiler
31. Apple, `NSHostingView` (macOS 10.15+) — "integrate SwiftUI views into your AppKit view hierarchies"; `safeAreaRegions`, `sizingOptions`, event forwarding. https://developer.apple.com/documentation/swiftui/nshostingview
32. Apple, SwiftUI `Canvas` — immediate-mode drawing; "might provide better performance for a complex drawing that involves dynamic data"; no per-element interactivity/accessibility. https://developer.apple.com/documentation/swiftui/canvas
33. Apple, SwiftUI `drawingGroup(opaque:colorMode:)` — offscreen composite. https://developer.apple.com/documentation/swiftui/view/drawinggroup(opaque:colormode:)
34. Apple, SwiftUI `TimelineView` (macOS 12.0+) — scheduled redraws; system may coarsen cadence. https://developer.apple.com/documentation/swiftui/timelineview
35. Apple, `CADisplayLink` (macOS 14.0+) — "synchronize its drawing to the refresh rate"; frame-rate factors; Low Power Mode / thermal can lower it; `isPaused`, `invalidate()`. https://developer.apple.com/documentation/quartzcore/cadisplaylink
36. Apple, `NSAnimationContext`. https://developer.apple.com/documentation/appkit/nsanimationcontext
37. Apple, `AXIsProcessTrusted()` and `AXIsProcessTrustedWithOptions(_:)` with `kAXTrustedCheckOptionPrompt` (prompt is asynchronous and doesn't change the return value). https://developer.apple.com/documentation/applicationservices/1460720-axisprocesstrusted and https://developer.apple.com/documentation/applicationservices/1459186-axisprocesstrustedwithoptions
38. Apple, *Energy Efficiency Guide for Mac Apps* (archived) — "Monitor Usage Regularly": Activity Monitor energy/CPU columns ("When your app isn't active, the number in this column should be 0.0"), Xcode debug gauges ("When users aren't interacting with your app, it should have zero energy impact"), Instruments templates, and command-line tools `dtrace`, `fs_usage`, `pmset`, `powermetrics`. https://developer.apple.com/library/archive/documentation/Performance/Conceptual/power_efficiency_guidelines_osx/MonitoringEnergyUsage.html
39. Apple Support, *View energy consumption in Activity Monitor on Mac* — `Energy Impact`, `12 hr Power`, `App Nap`, `Preventing Sleep`, `Graphics Card`; page's version selector lists **macOS 27 Golden Gate**. https://support.apple.com/guide/activity-monitor/view-energy-consumption-actmntr43697/mac
40. Apple, `NSView.needsPanelToBecomeKey` — default `false`; interplay with non-activating panels. https://developer.apple.com/documentation/appkit/nsview/needspaneltobecomekey
41. Apple, `NSWindow.ignoresMouseEvents` — "transparent to mouse events". https://developer.apple.com/documentation/appkit/nswindow/ignoresmouseevents
42. Apple, `NSStatusBar` — status items "not guaranteed to be available at all times"; "use status items sparingly". https://developer.apple.com/documentation/appkit/nsstatusbar
43. Apple, SwiftUI `EnvironmentValues.accessibilityReduceMotion`. https://developer.apple.com/documentation/swiftui/environmentvalues/accessibilityreducemotion
44. Apple, `XCTCPUMetric` — CPU time / cycles / instructions retired. https://developer.apple.com/documentation/xctest/xctcpumetric
45. Apple, `XCTMemoryMetric` — memory delta across the measured block. https://developer.apple.com/documentation/xctest/xctmemorymetric

### Read but not cited for a substantive claim

Activity Monitor User Guide index [S39 base]; `NSWindow.isOpaque` / `hasShadow` / `NSWindow.OrderingMode`; `NSApplication.terminate(_:)`; `NSApplication.delegate`; `ProcessInfo`; `NSBackgroundActivityScheduler.init(identifier:)`; `NSApplication.ActivationPolicy`; `NSWindow.CollectionBehavior/primary`; `NSApplication.didChangeScreenParametersNotification` (already cited). These were read to confirm no contradicting documented behaviour and are not relied on above.
