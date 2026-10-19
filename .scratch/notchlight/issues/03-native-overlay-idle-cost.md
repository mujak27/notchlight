# What native overlay design can stay idle cheaply?

Type: research
Labels: wayfinder:research
Parent: ../map.md
Assigned to: mujak27 (research worker acting for dev)
Status: resolved

## Question

What native overlay design lets the island stay cheap at idle while still showing playback and notifications correctly across the real desktop? Answer with evidence and a measurable test plan, not with a build and not with fabricated numbers.

The answer must cover:

- **Notch geometry and safe area**: how to place and size the surface relative to the notch and menu bar on Mac17,8, and how the safe area behaves.
- **Panel / window and app lifecycle**: which native window or panel type and app lifecycle (accessory/agent app, activation policy, launch behaviour) suits a notch island.
- **Fullscreen / Spaces / external displays**: how the surface should behave under fullscreen apps, across Spaces, and on external displays.
- **Event vs polling and energy guidance**: which effects need an event-driven path versus a timer, where polling is unavoidable, and the platform energy guidance that applies.
- **Measurable test plan**: the concrete workload, instruments, and observations that would let a human later measure idle CPU, battery, and memory — described as a method, with NO invented benchmark values.

## Comments

- 2026-10-19 wayfinder charting: split from the map's overlay thread. Resource priority is idle CPU and battery first, memory next, with no numeric budget approved yet. This ticket produces evidence and a measurement method, not a budget decision.
- 2026-10-19 claim: Assigned to mujak27 (research worker acting for dev); Status set to `claimed` and saved before research dispatch.
- 2026-10-19 context pointer (research dispatch): branch `research/overlay` (based on `origin/main`); findings report at [`research/overlay.md`](../../../research/overlay.md).
- 2026-10-19 research dispatched: findings-only report requested; no repo writer conflict. This records dispatch only; the ticket is neither complete nor resolved.
- 2026-10-19 resolved: research answered below; Status set to `resolved`, with the assignment and claim history above preserved. Evidence report: [`research/overlay.md`](../../../research/overlay.md). Research only — it approves no resource budget and no overlay behaviour.

## Answer

A **documented public AppKit candidate** exists for the overlay surface: a borderless, non-activating `NSPanel` in an accessory/agent app, anchored to the notch via `NSScreen.safeAreaInsets` and the auxiliary top-left/top-right areas, with window `collectionBehavior` for Spaces and event-driven updates instead of an idle timer.

The **presentation layer does not settle ingress cost**. Whether the combined collection behaviours keep the island over a native full-screen app, how it behaves across Spaces, on external displays, at lock/login, and how hover-to-expand coexists with a compact pill all remain **on-host tests**. Framework overhead (pure AppKit versus `NSHostingView`-hosted SwiftUI) is likewise unmeasured.

**Event-driven design with minimal redraw is preferred**; a live progress bar is the one effect that forces periodic display refresh. No resource benchmarks are stated, **no numeric budget is approved**, and zero energy at idle is not guaranteed.

Sources of record: Apple AppKit documentation (`NSPanel`, `NSScreen`, `NSWindow.CollectionBehavior`) and the archived *Energy Efficiency Guide for Mac Apps*.
