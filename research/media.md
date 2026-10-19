# Notchlight — Media source capabilities (research findings)

Ticket: [Which media sources can Notchlight reliably control?](../.scratch/notchlight/issues/02-media-sources.md).
Type: evidence, not a decision. This report makes no decision.

## How to read this document

- **No compatibility claim here is a guarantee.** Every route below is qualified with a confidence and the reason.
- **The target host is unverified.** All conclusions are drawn from published documentation, first-party source repositories, and engineering threads. **Nothing was tested on macOS 27.0.1, an M5 Pro, or any Mac.** Where a source is older than the target OS, it is marked stale.
- **Stale-source rule applied:** macOS 27 is newer than most cited sources. A source that worked "up to macOS 26" is *evidence of fragility and direction*, not a statement about macOS 27. Where possible the newest available evidence (2026, incl. macOS 27 beta) is cited.
- Confidence labels: **High** (first-party API/doc + corroboration), **Medium** (first-party API/doc alone, or strong multi-source community evidence), **Low** (single community source or inference from mechanism), **Unverified** (mechanism reasoning only).

## Bottom line for the ask

1. **No documented public *general system-wide* now-playing reader was found in the reviewed Now Playing APIs.** The public Now Playing surfaces reviewed on macOS (`MPNowPlayingInfoCenter`, `MPRemoteCommandCenter`, and the new macOS 27 `NowPlaying` framework) are for *publishing your own app's* playback to the system, not for reading what other apps are playing. This does **not** mean no public route exists at all: **supported per-app public scripting** (e.g. Apple Events/AppleScript to Music or Spotify) can read and control *that named app*, but it is app-specific and is a different thing from a system-wide reader. [High for the reviewed APIs]
2. **The leading *broad-system* route found in reviewed evidence is the private `MediaRemote` framework via the `mediaremoted` daemon**, and since macOS 15.4 that route is gated by a private entitlement no third party can hold. The community workaround reaches it by invoking a system binary (`/usr/bin/perl`, bundle id `com.apple.perl…`) that *is* entitled, and loading a bundled helper framework from it. Reviewed evidence also documents a simpler private alternative, `MRNowPlayingRequest` (callable via `osascript`), which cannot load artwork and gives no real-time updates. Reviewed evidence did **not** establish `MediaRemote` as the *only* possible cross-app route. [Medium — strong, recent, multi-source community + reverse-engineering evidence]
3. **Control (play/pause/next/previous) is more available than read.** Sending commands through `MediaRemote` kept working after macOS 15.4, and per-app AppleScript / Web APIs give control directly. Reading title/artwork/state reliably is the hard part. [Medium]
4. **Per-source confidence is uneven.** Apple Music and Spotify have first-party-ish scriptable interfaces but are fragile and version-sensitive; browser playback exposes state through the standard Media Session API, which Safari/Chrome forward to system Now Playing, but reading that back still depends on `MediaRemote`. [Medium/Low per source]

---

## 1. Two layers, and why the distinction is the whole story

There are two different things people call "now playing":

- **The system Now Playing layer** (Control Center, media keys, Lock Screen-style surfaces). On macOS this is served by the daemon `mediaremoted` through the private `MediaRemote.framework`. In the reviewed sources this is the only layer that knows about *all* sources at once, and it is the layer the notch UI wants.
- **Per-app direct control APIs** (AppleScript/Apple Events, Spotify Web API, browser DOM/Media Session). These talk to one named app only, do not know which source is "currently playing", and cannot see other apps.

Notchlight's requirement ("prefer the currently playing source") is a *system-layer* requirement. The system layer is the one for which **no public general system-wide reader was found in the reviewed APIs**. That tension is the core feasibility finding.

## 2. System currently-playing read APIs

### 2.1 Public APIs reviewed — publish-only; no general system-wide reader

- **`MPNowPlayingInfoCenter`** (macOS 10.12.2+): documented as "An object for setting the Now Playing information **for media that your app plays**." Its `nowPlayingInfo` property is "The current Now Playing information **for the default Now Playing info center**" — i.e. your own app's entry. It is a publish/read-your-own surface, not a system-wide reader. [High]
  - Source: <https://developer.apple.com/documentation/mediaplayer/mpnowplayinginfocenter>, property: <https://developer.apple.com/documentation/mediaplayer/mpnowplayinginfocenter/nowplayinginfo>
- **`MPNowPlayingSession`**: manages Now Playing info/commands for *your* `AVPlayer` instances. Apple's availability metadata lists it for iOS 16.0 / iPadOS 16.0 / Mac Catalyst 16.0 / tvOS 14.0 / visionOS 1.0 — **macOS is not listed**, so it is not even a macOS option. [High]
  - Source: <https://developer.apple.com/documentation/mediaplayer/mpnowplayingsession> (platform list in the doc's metadata)
- **`MPRemoteCommandCenter`**: receives remote commands *your app* registered for when *your app* is the Now Playing app. It does not enumerate or read other apps. [High]
- **New `NowPlaying` framework (introduced 27.0 — iOS/iPadOS/macOS/Mac Catalyst/tvOS/visionOS/watchOS 27.0)**. WWDC26 session 312 and the framework reference describe it entirely as *publishing* your app's media — local sessions (`MediaSession` + `MediaSessionRepresentable`) and remote sessions (`RemoteMediaSession`) broadcast to Lock Screen / Control Center / Dynamic Island / CarPlay. There is **no query/enumeration/read surface for other apps' playback** in the documented API. The framework reference explicitly warns not to mix it with `MPNowPlayingInfoCenter`/`MPRemoteCommandCenter` for local playback. [High]
  - Sources: <https://developer.apple.com/videos/play/wwdc2026/312/>, <https://developer.apple.com/documentation/nowplaying>
- Taken together, **no public *general system-wide* reader for the currently playing source was found in the reviewed Now Playing APIs, and macOS 27 does not add one.** The reviewed public APIs reinforce the publish-only model. This does **not** rule out public *per-app* routes: a supported app's own scripting (Apple Events/AppleScript) can expose that one app's playback, as covered in §4. This distinction — no system-wide public reader vs. supported per-app public scripting — must not be collapsed. This is the single most important finding for the ticket. [High for the reviewed APIs]

### 2.2 Private API — `MediaRemote.framework` (a leading broad-system route)

- **What it is:** a private framework that talks to the media server (`mediaserverd` historically, `mediaremoted` on modern macOS). It exposes functions such as `MRMediaRemoteGetNowPlayingInfo`, `MRMediaRemoteSendCommand`, and now-playing notifications. [Medium]
  - Source: <https://theapplewiki.com/wiki/Dev:MediaRemote.framework> (community reverse-engineering wiki)
- **What it returns (rich):** a metadata dictionary. A mature adapter's documented key set includes: `bundleIdentifier`, `parentApplicationBundleIdentifier`, `playing`, `title`, `artist`, `album`, `duration`, `elapsedTime`, `timestamp`, `artworkMimeType`, `artworkData`, `playbackRate`, `repeatMode`, `shuffleMode`, `mediaType`, `uniqueIdentifier`, `queueIndex`, `totalQueueCount`, and more. This is everything the notch feature needs: title, artist, album, artwork (base64), playing state, elapsed time. [Medium]
  - Source: `get`/`stream` command docs, <https://github.com/ungive/mediaremote-adapter> (README)
- **Commands it can send:** play, pause, toggle, stop, next, previous, seek, skip ±15 s, shuffle/repeat, speed. [Medium] (same README `send COMMAND` table)
- **Real-time updates:** the adapter's `stream` command pushes diffed JSON updates, so a long-lived observer is possible without polling; the framework-based streaming path can also deliver artwork. [Medium]
- **A simpler private alternative in reviewed evidence — `MRNowPlayingRequest`:** callable via `osascript`, it can fetch now-playing info without the framework-loading trick, but cannot load artwork and gives no real-time updates (requires polling). It is a second private route, though weaker for this feature. [Medium]
- **The macOS 15.4 gate (critical):** reverse-engineering in the LyricFever issue shows `mediaremoted` began enforcing entitlements in 15.4. The client must carry `com.apple.mediaremote.now-playing-read-access` and/or `com.apple.mediaremote.full-now-playing-read-access`; the 15.4 `-[MRDMediaRemoteClient isAllowedAccessToDataFromPlayerPath:]` path now returns "Operation not permitted" for ordinary apps. Before 15.4 the entitlement was not enforced for the local playback path. [Medium — reverse-engineered disassembly + reproduced error `kMRMediaRemoteFrameworkErrorDomain Code=3 "Operation not permitted"`]
  - Source: <https://github.com/aviwad/LyricFever/issues/94> (comments by Mx-Iris, spbgzh, ungive)
  - Corroboration in a real notch app: <https://github.com/TheBoredTeam/boring.notch/issues/417>
- **The community workaround (the current state of the art):** because processes whose bundle id begins `com.apple.` are granted access, invoke a privileged *system* binary — `/usr/bin/perl` (reported bundle id `com.apple.perl…`) — and have it dynamically load a bundled helper framework that performs the `MediaRemote` calls and streams results to stdout. This survives macOS 15.4+ without disabling SIP, **and is reported working on macOS 26 Tahoe and macOS 27.0 beta** (the project's badge lists `macOS 27.0 (26A5425a)`, last tested Sep 2026). [Medium]
  - Sources: <https://github.com/ungive/mediaremote-adapter> (README: mechanism, `test` self-check command, macOS 27 testing); the README points to a maintained Swift fork at <https://github.com/ejbills/mediaremote-adapter>
- **SIP-based bypass is out of scope.** Code-injecting `mediaremoted` with SIP disabled is one documented way to bypass the gate, but the map explicitly forbids disabling SIP and system modifications, so it is excluded. Reviewed evidence did **not** establish it as the only alternative. [High for the exclusion]
- **This route is inherently fragile.** Its own author added a `test` command so apps can detect breakage and *fall back to AppleScript* — an explicit admission that a future macOS update may break it again. One adapter revision already needed a fork to keep seeking working on 26.2. [Medium]

### 2.3 Control vs read asymmetry (post-15.4)

- Multiple independent reports state **reading stopped but commands kept working**: "MRMediaRemoteCommand still work … still can pause and play music through MRMediaRemote", and boring.notch users saw "next, previous and pause controls … working but the music slider, album poster is not showing". So a source that can be *controlled* via `MediaRemote` may still be *unreadable*. [Medium]
  - Sources: LyricFever #94 (issue text), boring.notch #417 (issue text).

## 3. Permissions and the permission model

There are three separate permission worlds, and conflating them is a trap:

- **`MediaRemote` (private):** needs the private entitlement above (not user-grantable). The practical path in reviewed evidence is the entitled-system-binary workaround (plus the weaker `MRNowPlayingRequest` alternative). **No TCC prompt, no user setting.** [Medium]
- **Apple Events / Automation (for AppleScript to Spotify or Music):** the first Apple Event to each target app raises a TCC prompt; approval appears in System Settings → Privacy & Security → Automation. The declarations differ by distribution and should be stated precisely: a hardened-runtime (non-sandboxed) app declares the `com.apple.security.automation.apple-events` entitlement plus an `NSAppleEventsUsageDescription` Info.plist string; a sandboxed app uses sandbox entitlements, and the App Store path uses `com.apple.security.scripting-targets` for scripting targets. **Verification needed:** the exact entitlement/Info.plist combination for a personal, non-sandboxed hardened-runtime app was not confirmed against Apple documentation here, so this report does not treat any single entitlement as "the" one. It is *per target app* and reported flaky (prompts that never reappear if the target app hasn't run recently). [Medium for mechanism; precise entitlement mapping marked verification-needed]
  - Sources: AppleScript docs and the automation-permission failure report: <https://developer.apple.com/forums/thread/792157>
- **Accessibility:** the ticket notes accessibility APIs are permitted for notifications, but **media reading/control does not require Accessibility** — the routes here are `MediaRemote`, Apple Events, or per-app APIs. Accessibility would only enter if one tried to drive another app's UI (not proposed here). [High]

The map's permission stance (map §Notes) is: Accessibility APIs are permitted; **private APIs MAY be investigated, with OS-update fragility acknowledged as a research concern, but private APIs are NOT yet accepted for the final implementation**; no SIP disabling and no system modifications. Read that way, the private `MediaRemote` + entitled-system-binary route is a **research candidate only, not an approved implementation path**, and this document does not endorse it. Any "only route" phrasing must not be read as a decision.

## 4. Per-source findings

### 4.1 Apple Music (`Music.app`) — native, macOS

- **Read via `MediaRemote`:** works in principle as a system source; historically Apple Music's own playback is the most consistently represented source (`com.apple.Music` is itself entitled, so its notifications are always produced). [Medium/Low]
- **Read/control via AppleScript:** `Music.app` is scriptable (`tell application "Music"`), exposing `player state`, `current track` (name/artist/album/artwork), play/pause/next/previous. **But on macOS 26 Tahoe the `current track` event broke for non-library and autoplay tracks** — `Can't get name of current track` (-1728) — while library/"songs"-tab playback still works. This is a first-party developer-forum report reproducing on Tahoe, i.e. the *most recent* Apple Music evidence available and a direct warning that even first-party scripting is unstable. [High for the report; the breakage pattern is version-specific]
  - Source: <https://developer.apple.com/forums/thread/798267> ("Apple Script for Music app no longer supports current track event")
  - Corroboration: boring.notch issues where Apple Music media "playback not working" / "List of Song in Apple Music not visible in notch".
- **MusicKit** is for playing Apple Music *in your own app*; it is not a reader for the Music.app's playback. Do not treat it as a source for this feature. [Medium]

### 4.2 Spotify — third-party native, macOS

- **Read/control via AppleScript:** Spotify's macOS app exposes a scripting dictionary. Commonly used terms include `player state`, `current track` (`name`, `artist`, `album`, `artwork url`), `player position`, `playpause`, `next track`, `previous track`. This is community-documented (Spotify does not publish AppleScript docs); dictionaries and behaviours have changed across releases, and users report breakage that comes and goes. [Medium]
  - Sources: <https://github.com/simonbs/spotify-controls-api/blob/master/Spotify.applescript> (working script), Spotify community threads on `current track` breaking.
- **Control via Spotify Web API:** the Player endpoints control playback and read `currently-playing`, but (a) need OAuth + **Spotify Premium**, (b) operate on the user's Spotify Connect playback regardless of which local window is focused, (c) since 2025 Spotify tightened API access for small/indie apps. This is heavier and account-coupled than the "no accounts" scope allows. [High for the mechanics; it is a poor fit for this feature]
  - Sources: <https://developer.spotify.com/documentation/web-api>, <https://engineering.atspotify.com/2022/04/spotifys-player-api>
- **Read via `MediaRemote`:** Spotify does publish to system Now Playing, so the adapter path can see it — subject to the 15.4 gate. boring.notch shipped dedicated "media controllers for Spotify, YouTube Music, and Apple Music" precisely because a single `MediaRemote` path was not enough. [Medium]

### 4.3 Browser playback (Safari, Chrome) — the Media Session API

- **What the browser exposes:** standards-based **Media Session API** (`navigator.mediaSession`, `MediaMetadata`, action handlers) lets a page publish title/artist/album/artwork and handle play/pause/next/previous/seek. [High]
  - Source: <https://developer.mozilla.org/en-US/docs/Web/API/Media_Session_API>, <https://web.dev/articles/media-session>
- **Browser → system Now Playing:** Apple states that **"Safari will use the information provided through Media Session to populate Now Playing with your content's title, duration, play state, artwork, and more."** Chrome likewise integrates Media Session with the OS media keys/hub; the same API drives the macOS "Now Playing" widget. So *the browser* publishes browser playback into the system layer. [High for Safari (first-party Apple statement); Medium for Chrome]
  - Source: WWDC21 session 10189 "Coordinate media playback in Safari with Group Activities": <https://developer.apple.com/videos/play/wwdc2021/10189/>
- **Reading it back for Notchlight:** no reviewed source documents a per-browser API a third-party app can call to read the page's Media Session state. The practical read is again `MediaRemote` (subject to the 15.4 gate and the adapter). Identifying "this is Safari" vs "this is Chrome" vs a tab is limited to what `MediaRemote` reports (`bundleIdentifier`), not which site/tab beyond what the page chose to publish. [Medium/Low]
- **Controlling browser playback:** media keys/the system layer route to whatever the browser declares as now playing, so `MediaRemote` commands generally work. But if a page sets no action handlers, controls may be absent — the web.dev article notes controls "won't be shown unless the proper action handler is set". Not every site implements next/previous. [Medium]
- **Browser extension / helper options** (a Notchlight-controlled extension in Safari/Chrome, or a helper): would give direct, reliable access to the page's Media Session state and control *for that browser*, bypassing the `MediaRemote` read gate — but it is browser-specific, needs per-browser code, and is a materially larger surface. No primary source mandates it; it is an option, not a documented requirement. [Low — options space, not evidence of a working route]
- **Net:** browser playback is *probably* visible and controllable through the system layer, but no reviewed public **system-wide** API lets Notchlight read a browser's playback; that read would depend on the private route or a per-browser helper.

### 4.4 Other native media apps (VLC, IINA, TIDAL, podcasts, etc.)

- **Generic case:** any app that publishes to system Now Playing is visible through `MediaRemote` (subject to the 15.4 gate). This is the appeal of the system-layer approach: broad source coverage from one route. [Medium]
- **Known per-app irregularities:** the adapter documents a TIDAL-specific quirk where metadata transiently mixes old and new track, requiring a debounce workaround — evidence that even within the system layer, per-app metadata quality varies. [Low/Medium]
  - Source: <https://github.com/ungive/mediaremote-adapter> (`--experimental-peculiar-debounce:BUNDLE_ID`)

## 5. Artwork / state / previous–next matrix (capability, per source)

Legend: ✓ available · ~ partial/conditional · ✗ not available · ? unverified on macOS 27.

| Source | Read title/artist | Read artwork | Reliable play state | prev/next | Route(s) | Confidence |
|---|---|---|---|---|---|---|
| Apple Music (`Music.app`) | ✓ (vulnerable: Tahoe `current track` bug on non-library/autoplay) | ~ (via `MediaRemote`; AppleScript artwork unreliable) | ~ | ✓ | `MediaRemote`; AppleScript | Medium |
| Spotify | ✓ | ~ (`artwork url` via AppleScript; `MediaRemote` artwork may lag) | ~ (AppleScript `player state`) | ✓ | `MediaRemote`; AppleScript; Web API | Medium |
| Safari playback | ~ (what the page publishes) | ~ (page-provided) | ~ | ~ (only if the site sets handlers) | System layer only (browser publishes via Media Session) | Medium/Low |
| Chrome playback | ~ | ~ | ~ | ~ | System layer only | Low/Medium |
| Other native apps | ✓ (varies) | ~ (varies; some unload artwork transiently) | ~ | ~ | `MediaRemote` | Low/Medium |
| **Cross-app / system-wide, public API only** | ✗ | ✗ | ✗ | ✗ | no public system-wide reader found | High (for the reviewed APIs) |
| **One named app via its own supported public scripting** | ✓ (that app) | ~ (that app) | ~ (that app) | ✓ (that app) | e.g. Apple Events/AppleScript to Music or Spotify | Medium |

The two final rows are the point: **no public *system-wide* reader was found**, but **supported per-app public scripting can read that one app** — these must not be collapsed. Every row above the per-app-scripting row relies on the private `MediaRemote` route or a per-app API.

## 6. Sleep/wake and multiple sources

### 6.1 Multiple simultaneous sources

- **Apple documents *eligibility* heuristics, not a full cross-app election rule.** WWDC22 explains the criteria for an app to become the Now Playing app: it must register a handler for at least one remote command, and its audio session must be a non-mixable category (mixable playback is treated as not a candidate). Being the Now Playing app means it populates Control Center and receives remote controls. **How the system picks among multiple eligible apps is not documented in the reviewed sources.** [High for the eligibility heuristics; the doc predates macOS 27 and does not state the cross-app arbitration rule]
  - Source: WWDC22 session 110338, <https://developer.apple.com/videos/play/wwdc2022/110338/>
- **Observation, not established macOS policy:** a recent (2026) Apple developer-forum post on iOS 27 describes one app with two Now Playing *clients* and states "MediaRemote elects between them correctly — the client whose process is making sound becomes active". That is *iOS / within-app* evidence. Extending it to *cross-app macOS* arbitration ("the active source is whichever app is making sound") is **speculation**; no reviewed source documents the cross-app macOS election rule, and it must not be presented as policy. What can be said safely: a system "currently playing" notion exists and it can change as sources start/stop. [Low — inference/speculation, explicitly not evidence of macOS policy]
  - Source: <https://developer.apple.com/forums/thread/847151>
- **Ambiguity when several sources play at once** is a foreseeable product problem, and the system-layer answer ("whatever `mediaremoted` reports as now playing") is a system heuristic, not a Notchlight-chosen policy. [Low/Medium — the underlying macOS cross-app rule is not documented in reviewed sources]
- **Practical consequence:** boring.notch adding per-app controllers, and the adapter's need to disambiguate by `bundleIdentifier`, both show that "which source is playing" is not answered cleanly by one call in practice.

### 6.2 Sleep and wake

- **No primary source found that documents `MediaRemote`/`mediaremoted` behaviour across sleep/wake on macOS. This is simply Unverified.**
  - By mechanism only: `mediaremoted` is a per-user daemon and the adapter's `stream` is a separate long-lived process, so a long-lived observer *may* need to be re-established after wake. Whether it does, and whether the system's now-playing state persists, is cleared, or is rebuilt on wake, is **not documented in reviewed sources** and is not asserted here.
  - Treat this as a human testing item on the target host, not an assumption in the spec.
- **Recommendation to carry into the spec as an unknown:** define recovery behaviour for "stream died after wake" and "no source after wake" before any implementation, and verify on the target host.

## 7. Per-ticket confidence summary

- **Public *system-wide* reader for other apps' now playing: not found** in the reviewed Now Playing APIs (own-app publish only, incl. macOS 27 Now Playing framework); this does not preclude supported per-app public scripting. Confidence: High for the reviewed APIs.
- **Private `MediaRemote` read: conditional and fragile; the leading broad-system route in reviewed evidence (not proved exclusive).** Reachable via the entitled-system-binary workaround; gated since 15.4; works on 26/27 per community evidence. A weaker private alternative (`MRNowPlayingRequest`) also exists. Confidence: Medium.
- **Apple Music: readable/controllable, but first-party scripting broke on Tahoe for common cases.** Confidence: Medium.
- **Spotify: readable/controllable, AppleScript-fragile; Web API is Premium/account-coupled.** Confidence: Medium.
- **Browser (Safari strong, Chrome likely): control generally works; read depends on the private route.** Confidence: Medium/Low.
- **Multiple sources: a system "now playing" notion exists, but the cross-app selection rule is not documented in reviewed sources; treat it as unverified, not as a stable chosen source.** Confidence: Unverified for the rule.
- **Sleep/wake: Unverified.** Confidence: Unverified.
- **macOS 27 / M5 Pro specifically: Unverified** — no hardware testing was in scope.

## 8. Uncertainties and proposed human decision questions (not decisions)

Uncertainties to record:

1. Whether the `MediaRemote` + `/usr/bin/perl` adapter still works on **macOS 27.0.1 (build 26A434) specifically** (evidence is macOS 27.0 beta `26A5425a`, Sep 2026). Unverified on the target build.
2. Whether the system now-playing state **survives sleep/wake**, and whether a streaming observer must be restarted after wake. Unverified.
3. How `mediaremoted` arbitrates the active source **when multiple macOS apps play at once** — the cross-app macOS rule is not documented in reviewed sources — and whether it matches any policy Notchlight would want. Unverified/speculative.
4. Whether Safari/Chrome browser playback on macOS 27 exposes enough metadata (esp. artwork) through the system layer to render the notch without a per-browser helper.
5. Whether `Music.app`'s Tahoe `current track` regression persists in macOS 27 (it affects the AppleScript fallback that the adapter's own author recommends).
6. Whether an AppleScript fallback or a browser extension would be needed at all, given the private route covers more sources.

Proposed decision questions for the human (these are options, not recommendations):

- **D1. Which read route is acceptable for a personal, no-SIP-change app, given the map currently authorizes private APIs for *investigation* only and not for final implementation?** (a) private `MediaRemote` via entitled-system-binary adapter (would have to meet the map's implementation bar); (b) supported per-app AppleScript/Apple Events only (narrower, each source a separate integration); (c) browser-extension/helper additions. Each has a different failure story.
- **D2. What is the accepted fragility posture?** Private API means a macOS update can break media read at any time; the spec needs an explicit "media unavailable" behaviour and a self-test/fallback policy (the adapter ships a `test` command for exactly this).
- **D3. What is the supported-source list for v1?** e.g. Apple Music + Spotify only, or "whatever the system layer reports". Broad coverage is *possible* via the system layer but *unproven per source*.
- **D4. What is the multi-source policy?** Follow whatever `mediaremoted` reports as now playing (simple, but the cross-app rule is undocumented and may flip or conflict when several sources play), or add explicit per-app preference/priority (more work, more defined).
- **D5. What is the sleep/wake requirement?** Must the indicator self-heal after wake, and is a brief "no media" gap acceptable?
- **D6. Is the target macOS 27 build to be verified by hand before the spec is closed?** (Recommend yes, given every route above is version-sensitive.)

## 9. Sources

Primary (Apple / standards):

- `MPNowPlayingInfoCenter` — <https://developer.apple.com/documentation/mediaplayer/mpnowplayinginfocenter>; property <https://developer.apple.com/documentation/mediaplayer/mpnowplayinginfocenter/nowplayinginfo>
- `MPNowPlayingSession` (no macOS) — <https://developer.apple.com/documentation/mediaplayer/mpnowplayingsession>
- `MPRemoteCommandCenter` — <https://developer.apple.com/documentation/mediaplayer/mpremotecommandcenter>
- Be a now playable app — <https://developer.apple.com/documentation/mediaplayer/becoming-a-now-playable-app>
- Now Playing framework overview (27.0) — <https://developer.apple.com/documentation/nowplaying>
- WWDC26 312, "Meet the Now Playing framework" — <https://developer.apple.com/videos/play/wwdc2026/312/>
- WWDC22 110338, "Explore media metadata publishing and playback interactions" (Now Playing eligibility heuristics) — <https://developer.apple.com/videos/play/wwdc2022/110338/>
- WWDC21 10189, Safari + Media Session populates Now Playing — <https://developer.apple.com/videos/play/wwdc2021/10189/>
- Apple Developer Forums 798267, Music.app `current track` broken on Tahoe — <https://developer.apple.com/forums/thread/798267>
- Apple Developer Forums 792157, automation-permission flakiness — <https://developer.apple.com/forums/thread/792157>
- Apple Developer Forums 847151, MediaRemote client election ("process making sound becomes active") — <https://developer.apple.com/forums/thread/847151>
- MDN Media Session API — <https://developer.mozilla.org/en-US/docs/Web/API/Media_Session_API>
- web.dev, Media Session API (controls only when handlers set; macOS Now Playing widget) — <https://web.dev/articles/media-session>
- Spotify Web API — <https://developer.spotify.com/documentation/web-api>; Player API notes — <https://engineering.atspotify.com/2022/04/spotifys-player-api>

First-party / reverse-engineering / source repositories:

- `ungive/mediaremote-adapter` — mechanism, macOS 15.4 gate, commands, keys, `test`, macOS 26/27 testing — <https://github.com/ungive/mediaremote-adapter>
- `ejbills/mediaremote-adapter` (Swift fork the adapter README points to as maintained) — <https://github.com/ejbills/mediaremote-adapter>
- `aviwad/LyricFever` issue #94 — the 15.4 entitlement change, disassembly, workarounds, `MRNowPlayingRequest` limits — <https://github.com/aviwad/LyricFever/issues/94>
- `TheBoredTeam/boring.notch` issue #417 — a real notch app losing now-playing on 15.4, and shipping per-app Spotify/YouTube-Music/Apple-Music controllers — <https://github.com/TheBoredTeam/boring.notch/issues/417>
- `simonbs/spotify-controls-api` — working Spotify AppleScript dictionary usage — <https://github.com/simonbs/spotify-controls-api/blob/master/Spotify.applescript>
- The Apple Wiki, `Dev:MediaRemote.framework` — private framework overview and functions — <https://theapplewiki.com/wiki/Dev:MediaRemote.framework>
