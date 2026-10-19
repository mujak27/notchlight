# Which media sources can Notchlight reliably control?

Type: research
Labels: wayfinder:research
Parent: ../map.md
Assigned to: mujak27 (research worker acting for dev)
Status: resolved

## Question

Which media sources can Notchlight reliably read and control — track info, artwork, play/pause, and previous/next — and under what conditions? Answer with evidence and feasibility options, not with a build.

The answer must cover:

- **System currently-playing APIs**: what the system exposes for now-playing state (for example the MediaRemote layer and anything public), what it returns, and what permission it needs.
- **Public vs private APIs and permissions**: what each public and private route actually allows and the permission each requires.
- **Per-source behaviour**: what is achievable for Spotify, Apple Music, native media apps, and browser playback (browser-native controls and any browser extension or helper), including where control is impossible.
- **Artwork / state / previous-next limitations**: which sources expose artwork, reliable playback state, and working previous/next, and which do not.
- **Sleep and multiple-source behaviour**: what happens when the Mac sleeps, and how the APIs behave when several sources play at once (does a "currently playing" source exist and is it stable?).

Do NOT report any source as guaranteed support; report per-source confidence and the reason.

## Comments

- 2026-10-19 wayfinder charting: split from the map's media thread. Currently-playing source is preferred and broad source coverage is unproven. This ticket produces evidence, not a decision.
- 2026-10-19 claim: Assigned to mujak27 (research worker acting for dev); Status set to `claimed` and saved before research dispatch.
- 2026-10-19 context pointer (research dispatch): branch `research/media` (based on `origin/main`); findings report at [`research/media.md`](../../../research/media.md).
- 2026-10-19 research dispatched: findings-only report requested; no repo writer conflict. This records dispatch only; the ticket is neither complete nor resolved.
- 2026-10-19 resolved: research answered below; Status set to `resolved`, with the assignment and claim history above preserved. Evidence report: [`research/media.md`](../../../research/media.md). Research only — no source is guaranteed and no read route is approved.

## Answer

No documented **public general system-wide reader** for other apps' now-playing was found: the reviewed Now Playing APIs (`MPNowPlayingInfoCenter`, `MPRemoteCommandCenter`, and the macOS 27 `NowPlaying` framework) publish an app's own media. **Supported per-app public scripting** (Apple Events / AppleScript to a named app such as Music or Spotify) is a separate public option that reads and controls that one app.

The leading **broad-system route is private**: `MediaRemote` via `mediaremoted`, reachable through a system-binary helper after the macOS 15.4 entitlement gate. It is fragile, unverified on the target host, and **not an approved implementation path**.

Compatibility scope therefore **depends on the adapter, the site, and the source**, and on testing: Apple Music and Spotify are scriptable but version-fragile, browser playback publishes through the Media Session API into the system layer while reading it back depends on the private route, and multiple-source arbitration and sleep/wake behaviour are undocumented.

Sources of record: Apple MediaPlayer/Now Playing documentation; the `mediaremote-adapter` project; Boring Notch issue #417.
