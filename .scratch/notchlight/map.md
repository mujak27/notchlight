# Notchlight — Wayfinding Map

Labels: wayfinder:map

The planning map for the Notchlight effort: a personal, native macOS notch-island app. It records the destination, the charting notes, the decisions so far, the remaining fog, and what is out of scope. Child tickets live in `issues/`; this map does not index them.

**Planning only.** No app code exists and no implementation is described as finished. The foundational research is complete and reviewed; feasibility of the core premise remains **UNPROVEN**, and is still conditional on the compatibility boundary and the resource budgets, which the human has not yet chosen. The all-app ambition is not narrowed here.

## Destination

An implementation-ready specification for Notchlight — a personal, native macOS notch-island app with exactly two features:

- **Notification replacement**: capture notifications from other apps into the island and suppress their normal banners.
- **Media control**: a compact now-playing indicator in the island with track info, artwork, play/pause, and previous/next.

The spec is done when it can be handed to an implementer with no remaining feasibility discovery: every compatibility boundary, supported-source list, failure/recovery behaviour, and acceptance criterion is decided and written down.

## Notes

Human-approved charting notes. These shaped the map; they are not resolved tickets and are not decisions-so-far.

- Destination is an implementation-ready spec, not an app implementation.
- Target host: personal Mac — Apple M5 Pro 16-inch, macOS 27.0.1 (build 26A434), Mac17,8, arm64, 24 GiB. One known host; portability is NOT a requirement.
- Scope is exactly two features: notification replacement and media control.
- Notifications: capture notifications from all apps into the island and suppress the normal banners. Feasibility is UNPROVEN.
- Accessibility APIs are permitted. Private APIs MAY be investigated; OS-update fragility is acknowledged as a research concern, but private APIs are NOT yet accepted for the final implementation. No SIP disabling and no system modifications.
- Preserve lockscreen privacy, and honour Focus and per-app notification preferences.
- Notification previews: clicking opens the source app; dismissing removes the preview. No inline reply and no custom history; leave history in Notification Center if feasible.
- Media: track info, artwork, play/pause, previous/next, preferring the currently playing source. Broad source coverage is unproven.
- Media explicitly excludes playlists, browsing, and accounts.
- Island behaviour: compact playback indicator; hover expands it; notifications temporarily take priority, then media is restored.
- Native only. No Electron and no embedded browser.
- Resource priority: negligible idle CPU and battery first, memory next. Do NOT invent measured performance numbers or numeric budgets; any budget is a human decision.
- Only planning, not implementation.
- A bounded, throwaway on-host prototype is the only further feasibility evidence contemplated; it authorizes no app build.
- Wayfinder grilling and domain-modeling skills are ongoing for this effort.

## Decisions so far

- No dedicated public cross-app notification API exists; the demonstrated public path is Accessibility mirroring of *visible* banners hidden off-screen — [Can Notchlight safely replace macOS notification banners?](issues/01-notification-banner-replacement.md)
- No public general system-wide now-playing reader was found; broad source coverage runs on the private `MediaRemote` route — [Which media sources can Notchlight reliably control?](issues/02-media-sources.md)
- A public AppKit panel is a candidate overlay surface, but ingress cost, Spaces/full-screen behaviour, and energy are unverified — [What native overlay design can stay idle cheaply?](issues/03-native-overlay-idle-cost.md)

## Not yet specified

Fog — fuzzy requirements that are not yet sharp enough to ticket:

- Final specification packaging and readiness: how the spec is assembled, reviewed, and verified once the compatibility boundary, the declared behaviours, and the resource budgets are settled. No spec is written now.

Known, exact questions are already ticketed in `issues/`; they are not fog.

## Out of scope

- App implementation in this effort.
- Public distribution and the App Store.
- Extra widgets.
- Playlists.
- Custom notification history and inline replies.
- System security changes (no SIP disabling, no system modifications).
