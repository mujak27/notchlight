# Notchlight research — capturing and suppressing other apps' notifications on macOS 27

Ticket: [Can Notchlight safely replace macOS notification banners?](../.scratch/notchlight/issues/01-notification-banner-replacement.md) (research); this report is evidence only and makes no decision — the decision belongs to the grilling ticket [Choose the notification compatibility boundary](../.scratch/notchlight/issues/04-notification-compatibility-boundary.md).
Host under study: personal Mac Apple M5 Pro 16", macOS 27.0.1 (build 26A434), Mac17,8, arm64. One host; portability not required.
Constraints assumed: no SIP disabling, no system-modification, no OS preference changes, **no new permission prompts** for this research.
Method: read-only primary-source and first-party-source research (Apple documentation/release notes/support docs; a shipping open-source notch app; security-research write-ups; community source code). No code was run against the host and no system state was changed.
Written: 2026-10-19. Research branch: `research/notifications` (based on `origin/main`).

---

## 0. Headline: three separate verdicts

The question asks for capture and suppression. They must be answered separately, and the privacy requirement bears on both.

| Axis | Verdict | Confidence |
|---|---|---|
| **Ingestion — read other apps' notifications** | **Partly demonstrated.** No documented public **dedicated cross-app notification API** was found in the reviewed sources, and the public `UNUserNotificationCenter` surface is app-scoped. The **Accessibility API (AX)** is itself public and is the only no-SIP mechanism found in the reviewed evidence that reads other apps' notifications — and it sees **visible banners only** (title/subtitle/body + app display name). The Notification Center SQLite DB holds *stored records* (possibly incl. attachments); its permission path on this host is unverified, and macOS 27 restricts cross-container access by default. | Medium–High for AX; Low–Medium for DB permission and completeness (unverified) |
| **Suppression — hide the normal banners** | **Constrainable, not solved.** No documented public API for suppressing other apps' banners was found. **Pre-emptive** suppression (preventing the banner from ever appearing) was **not demonstrated** in the reviewed evidence under the given constraints — that is not proof it is impossible, only that no public or in-scope route was established. The demonstrated non-destructive method is to **move the banner window off-screen via AX** (the banner appears, then is hidden). Pressing the banner's Close/Clear is also reported to suppress, but its exact effect on Notification Center history is unverified. | Medium for AX off-screen hide; **not demonstrated** for pre-emptive; Medium for unverified close semantics |
| **Privacy — lockscreen, Focus, per-app** | **Unresolved; both routes need validation.** AX reads only what is *rendered*, so content is bounded by the system's presentation — but whether AX returns redacted text under "Show Previews = Never/When Unlocked" is **unverified**. The DB stores records independent of what was displayed, and a DB-based implementation could still choose to filter, but no proven, reliable way to integrate the user's Focus/lockscreen/per-app policy for that route was found. Neither route was shown to enforce the map's privacy policy correctly, and neither was shown to violate it. | Low–Medium; needs a target-host validation step |

The map's destination — "capture notifications from **all** apps into the island and suppress their normal banners" — is **not established by the reviewed evidence**. What the evidence does support is a narrower shape: *mirror visible banners via the public AX API, hide them off-screen, and show them in the island* — with an explicit "visible banners only" boundary. Whether that boundary is acceptable is a human decision; **no boundary or fallback has been approved, and a narrowed "selected apps" scope must not be treated as user-approved.**

---

## 1. Mechanism inventory (what each exposes, and what it costs)

### 1.1 UserNotifications / `UNUserNotificationCenter` (public) — own app only

- `UNUserNotificationCenter` is documented as "The central object for managing notification-related activities **for your app or app extension**". [1]
- `getDeliveredNotifications(completionHandler:)`: "Fetches **all of your app's** delivered notifications that are still present in Notification Center… If none of **your app's** notifications are visible in Notification Center, the array is empty." [2]
- `UNUserNotificationCenterDelegate.willPresent` fires only for the delegating app's notifications. [3]
- `com.apple.developer.usernotifications.filtering` ("Enable receiving notifications without displaying the notification to the user") lets a **Notification Service Extension suppress its own app's remote push** after Apple approves the entitlement; it has no effect on other apps. [4]

**Verdict: the public UserNotifications surface does not expose other apps' notifications and does not provide a way to suppress them.** Community consensus matches: "`UNUserNotificationCenter` appears to be restricted to notifications generated by the current application." [5] This is a statement about the documented UserNotifications API surface found; the Accessibility API (§1.3) is a separate, public mechanism that does read other apps' banners.

### 1.2 Notification Center database (SQLite) — stored records, unresolved permission path

- **Location.** Before macOS 15: `$(getconf DARWIN_USER_DIR)com.apple.notificationcenter/db2/db` (plus an `attachments/` folder). [6][7] macOS 15 (Sequoia) moved it to `~/Library/Group Containers/group.com.apple.usernoted/db2/db`. [8]
- **Contents.** The `record` table holds each notification's binary-plist `data`, which contains the request with `titl` (title), `body`, and attachments (`atta`), keyed by an `app` table row with the source bundle identifier; also `delivered_date`, whether the notification was presented, and whether it was snoozed. [6][7] People have read `Slack`/`Messages` bodies out of it with `strings`/`plutil`. [6]
- **Permission before macOS 15.** "we don't need special permissions to read this database." [6]
- **Permission macOS 15+.** Apple: "In macOS 15 and later, app group containers offer System Integrity Protection… These app group containers limit access by apps that aren't in the app group. Apps not in the app group that attempt to access locations within an app group or app data container, **result in a user prompt** that requests their authorization." [9] A real-world log confirms reads of a *group container* trigger `kTCCServiceSystemPolicyAppData`. [10]
- **Permission macOS 27 (this host's major version).** Apple's macOS 27 release notes tighten this further:
  > "Accessing files in other developer teams' app data containers and app group containers **no longer prompts the user for authorization; such accesses are denied by default** and can be managed by the user in Privacy & Security settings. (161835690)" [11]
  > "XProtect may now restrict access to app data that is commonly targeted by malicious software. (178668601)" [11]
  A **separate** macOS 27 note concerns a different case — a process **without a bundle ID** — and gives the workaround "grant Full Disk Access to the process for the duration of the requirement. (184660124)" [11]. That note is about a non-bundled process; it does **not** establish that a normal bundled Notchlight.app would require blanket Full Disk Access, nor that a per-container grant is available to it. The exact permission path for a bundled app reading this container on macOS 27 is **unverified**.
- Apple is publicly moving to tighten Full Disk Access generally. [12]

**Verdict:** the DB is a **stored record** of notifications — a source that may include entries for notifications that were not shown as banners, and may include attachments — but it is **not** established to be complete or to contain every notification a user receives, and the ordering of its writes relative to a banner appearing is **not established**. On macOS 15+ reading it needs a user grant; on macOS 27 cross-container access is denied by default. Reading it from an app is one implementation choice (e.g. polling) and does not rule out other filesystem-event approaches on the DB/WAL. The container is also the Notification Center store — see §3 on the dismiss relationship. **UNVERIFIED on this host:** whether `group.com.apple.usernoted` is denied without a grant on macOS 27; what grant (if any) a bundled app can be given; whether that grant restores the read; and whether the stored records equal the set of notifications delivered.

### 1.3 Accessibility API (AXObserver + AX tree) — the demonstrated public mechanism

This is not speculation: a **shipping** open-source notch app implements essentially the requested feature. Boring Notch (TheBoredTeam/boring.notch) v2.8-rc.0 (2026-09-23) added:

> "**Notifications in the notch** – Mirrors visible notification banners in the notch with the sending application's icon. **Notification reliability** – Improved banner discovery, XPC delivery, focus handling, stale-banner handling, and Notification Center interaction. **Accessibility-aware mirroring** – Notification mirroring now checks Accessibility authorization in real time…" [13]

Its source (v2.8-rc.2) shows exactly how:

- A **non-sandboxed XPC helper** (`com.apple.security.app-sandbox = false`) does the work. [14][15]
- It checks `AXIsProcessTrusted()` and attaches to `com.apple.notificationcenterui` via `AXUIElementCreateApplication`. [16]
- It installs an `AXObserver` (`AXObserverCreate` + `AXObserverAddNotification`) for `AXWindowCreated`, `AXCreated`, `AXUIElementDestroyed`, `AXChildrenChanged`, `AXLayoutChanged`, and then walks the AX tree. [16]
- It detects **banners** by subrole: `AXNotificationCenterBanner`, `AXNotificationCenterAlert`, `AXNotificationCenterNotification`, `AXNotificationCenterBannerWindow`, `AXNotificationCenterAlertStack` (excluding `AXNotificationCenterPanel`, desktop widgets). [17]
- It reads content from AX element **identifiers** `title` / `header` / `subtitle` / `body`, and the app's display name from the `AXAttributedDescription` attribute. [16]
- The bundle identifier is **not** provided by AX; the helper derives it heuristically by matching the display name against running apps and `/Applications`. [16]
- The UI footer states the boundary explicitly: **"Requires [Accessibility]. Only visible banners are mirrored."** [18]
- macOS 27 compatibility note (their changelog): "Accessibility permission references now use the current macOS terminology (**Device Control and Data Access**)…". [13]

Apple treats this API as public: `AXUIElement` is "A structure used to refer to an accessibility object… Accessibility objects respond to messages sent by assistive applications and send notifications that describe state changes." [19] Accessibility is a TCC-granted permission with a user alert: "When a third-party app tries to access and control your Mac through accessibility features, you receive an alert, and you must specifically grant the app access." [20]

**Verdict:** AX gives **ingestion of visible banners** with **no private framework and no SIP change**, at the cost of the Accessibility permission and a hard "visible banners only" boundary. Fragility is real but explicitly managed by the app (dedicated subrole constants + unit tests in `NotificationPanelDetectionTests.swift` [21]). AX hierarchies and subroles shift between macOS releases (community reports and Apple's own release notes both show Notification Center AX/dispatch churn). [22][11]

### 1.4 Screen capture + OCR — heavy, low-fidelity, not found in practice

ScreenCaptureKit/the window server can read banner *pixels*, but requires Screen Recording, yields no structured fields, no actions, no attachments, and cannot suppress; it would also carry an ongoing capture/OCR cost that has **not been measured** (a potential cost, not a benchmark). No shipping notch app in the sources found uses it as the primary path; note several notch apps target a **known/selected app set** rather than all banners (e.g. DynamicLake markets "notifications from iMessages, WhatsApp, Telegram, Slack and more"). [23] Treat as a last-resort option only.

### 1.5 Private API / code injection — a route to *pre-emptive* suppression that is out of scope (not shown to be the only route)

To stop a banner from ever appearing (rather than chase it), a route would have to intervene inside Apple's notification pipeline (e.g. `usernoted` / `NotificationCenter`), via code injection or an Apple-private Mach/XPC client. The author of QuietYou (a shipping *dismisser*) reports the public ceiling that project hit and lists injection as *future* work: "Unfortunately there's no good way to stop the notification from appearing at all, but this way it's at least only on screen for a split second," and "Figuring out how to inject code into the Notification Center to actually prevent unwanted notifications from ever opening." [24] That is a report that *this* project found no public pre-emptive route — it is **not** evidence that every non-injection option is impossible. Injection itself would require defeating library validation / code signing and is excluded by the "no SIP disabling / no system modification" constraint. No entitlement granting a third party cross-app notification read or suppression was found in the reviewed sources; the only documented notification filtering entitlement is own-app and Apple-gated. [4]

**Verdict: pre-emptive suppression was not demonstrated in the reviewed evidence under the given constraints. Whether it is achievable at all — with or without injection — is unverified.**

---

## 2. Capture vs suppression: are they separable?

- **Public API for notifications:** no documented public *dedicated* cross-app notification API was found; the public UserNotifications surface is app-scoped (§1.1). The Accessibility API is itself public and provides capture *and* UI-level hide (§1.3), but not pre-emptive suppression.
- **DB route:** a stored-record read. It does not by itself hide anything, and no public "suppress this delivered notification" call was found. (Writing to the DB or to per-app preferences would be a system change, excluded.)
- **AX route:** capture and UI-level suppression are co-timed, not separable: the banner must exist before AX can see it. Options on an observed banner are (a) **press** an action, (b) **hide** it, or (c) do nothing.
- **Pre-emptive suppression (never shown) was not demonstrated** in the reviewed evidence under the given constraints; the banner is on screen for at least one observation/settle interval. Boring Notch's own wording is "only visible banners are mirrored." [18]

---

## 3. Suppression and Notification Center history — expected behaviour from source, not yet tested on the target

Two AX methods were found, with what appear to be opposite consequences for the map's "leave history in Notification Center if feasible" requirement. Both are read from source and documentation; **neither was exercised on the target host**.

1. **Press the notification's Close/Clear action (QuietYou's method):** the app "finds all the labels in the notification and checks to see if they contain matching text, and upon finding a match sends the annoyingly hard to click X button on the notification a 'press' message." [24] A community developer of Notification Center DB tooling reports that a notification which is closed/checked is "gone for good" from the store. [5] **Expected consequence:** dismissing likely removes the item from Notification Center — but the **exact** AX action, its name, and whether it clears the item on this macOS/app version are **unverified**. *If* it does clear the item, suppress-by-dismiss would conflict with the history requirement.

2. **Hide the banner window (Boring Notch's method):** it records the banner window's origin and sets its `kAXPositionAttribute` to `(-10000, -10000)` ("park"), then restores the origin when the mirrored notification goes away or when watching stops. [16] **Expected consequence:** the banner leaves the screen but is **not dismissed**, so the item stays in Notification Center. This is the only demonstrated non-destructive suppression found — but it is **expected behaviour read from shipping source, not tested on the target**.

   **Risks / unknowns with parking:**
   - Whether a parked window is recoverable if the helper dies while banners are parked is **unknown**; the helper calls `restoreAllWindows()` on stop, so behaviour when stop does not run is untested. [16]
   - It mutates a live system window's position; interactions with opening Notification Center, fullscreen, Stage Manager, multiple displays and animations are not covered by the code read and are **unverified**.
   - The banner may still be briefly visible before parking; macOS also animates it in.

   The changelog lists "stale-banner handling" and "Notification Center interaction" as work items — a signal that these paths had known issues at some point, though that is **not** evidence about which sharp edges remain. [13]

---

## 4. Content availability *before* suppression

- **AX route:** content is available only **as rendered and only after the banner exists**. Fields: title/subtitle/body + app display name. **No bundle identifier** (heuristic resolution), **no attachments**, **no rich content**. Action buttons exist as AX elements and can be pressed (Boring Notch's helper is the process that sends the Apple event used to reply to an iMessage notification [15]), but using arbitrary per-app actions is bespoke AX work, not a general API.
- **Show Previews interaction (requires validation):** because AX reads what is drawn, "Show Previews = Never/When Unlocked" would be expected to yield the *redacted* placeholder rather than the body. Apple documents the setting ("Show a preview in notifications… Never: Never show previews"; also per-app). [25] This is an expectation about what AX returns under that setting — **not verified**, and worth a one-off check on the real host.
- **DB route:** stored records that may include attachments; a DB-based implementation is one choice (e.g. polling) and it needs the unverified permission grant described in §1.2.
- **Ordering.** Whether stored records are available (`delivered_date`) before, at, or after a banner would be shown is **not established**; "full content *before* the banner appears" is therefore **unverified** for every route reviewed.

---

## 5. Focus, lock screen, and per-app settings consistency

- **Focus.** Apple: when a Focus is active, notifications "will go directly to Notification Center by default" unless the user opts to show them on the lock screen. [26] Nothing in the AX approach bypasses Focus. Consequence: Focus-silenced notifications do not present as banners, so an AX banner-capture would not surface them. Given the map's requirement to *honour* Focus and per-app settings, **not surfacing them is the expected, correct behaviour**, not a conflict with the requirement — provided the human decides the island's scope is *system banners eligible for presentation*, not *every event an app sends*. That is a real distinction: an app may emit events that the system suppresses or never delivers as banners, and "all apps" does not imply "every event". This boundary is a human decision.
- **Lock screen / display sleeping.** Apple exposes per-app toggles "Show Notifications: when screen is locked" and "when display is sleeping", plus a global "when mirroring or sharing the display" choice; if paused, no banner appears. [25] Not bypassed, and again absent from AX banner capture — expected given the respect-settings intent.
- **Lockscreen privacy.** Two unverified points: whether AX returns redacted text under "Show Previews = Never/When Unlocked" (likely, since AX reads what is drawn, but **not verified**), and what a DB-based implementation would do with full bodies the OS chose not to display. A DB read does not by itself force a privacy violation — an implementation could filter or hide — but a proven, reliable way to integrate the user's lockscreen/Focus/per-app policy for that route was **not found**. Both routes need a validation step before any privacy claim is made.
- **Persistent "Alerts" vs "Banners".** The AX code distinguishes alert subroles; persistent alerts stay on screen until acted on, so parking/dismissal semantics could differ from transient banners. [17] Per-app alert style (None/Banners/Alerts) is a user setting; "None" should mean no on-screen alert, but whether such notifications are still delivered to Notification Center (and therefore stored) depends on the separate "Show in Notification Center"/allow settings and is **not established** here. [25]
- **Critical / Time Sensitive.** These require an explicit per-app user decision the first time, so Notchlight cannot assume it can handle them. [27]

**Verdict:** the AX route **reads only what the system decided to show**, so it *inherits* Focus/per-app decisions rather than enforcing or bypassing them — which respects the settings' intent, but by itself does not cover "all apps / every event".

---

## 6. Click, dismiss, and Notification Center semantics

- **Click to open.** AX can press a banner (default action opens the source app). Boring Notch instead emulates open by activating the resolved bundle identifier via `NSWorkspace`, falling back to `NSWorkspace.urlForApplication`. [16][28] That means the *notification's own* default action (e.g. a deep link) is **not guaranteed** — it opens the app, which matches the map's "clicking opens the source app."
- **Dismiss.** As in §3: pressing Close is expected to remove the item everywhere (exact action/macOS effect unverified); parking hides the banner and is expected to keep the item.
- **Notification Center history.** Expected preserved by parking; expected destroyed by Close/checking. Both rest on source and community reports, not target testing. [16][5]
- **Duplicates / loss.**
  - If captured notifications are **re-presented as Notchlight's own `UNNotification`**, they become a *second* record in Notification Center under Notchlight's identity. There is no public API to attribute a notification to another app, and no way to reuse the original app's icon/identity (the app can only set its own content). Clicking that duplicate opens Notchlight unless it re-activates the source app. So re-posting → **duplicates**; and if you don't re-post, the island's content is only in the island.
  - If the app is **not running** or Accessibility is not granted, banners appear normally — a behavioural split, not a duplicate.
  - **Quiet / Focus-silenced / locked / sleeping-paused notifications:** these do not present as banners, so they are absent from AX banner capture (expected, given the respect-Focus/per-app intent). Whether each one still appears in Notification Center is a separate question and is **not established** for every case here.
  - An app whose alert is custom-drawn (not posted as a system notification) would be invisible to every mechanism here.
  - **Dedup key:** the banner `AXIdentifier`/`AXStackingIdentifier` is used as the token; AX identifiers are undocumented and version-dependent, so token stability across releases is a risk. [16]

---

## 7. Permission revocation mid-session

- **Accessibility revoked** (e.g. user toggles it off): AX calls fail; Boring Notch re-checks in real time and exposes a grant path, and its watcher `start()` returns `false` if not trusted. [13][16][18] The state to plan for is **parked banners with a dead helper**: if `stop()`/restore does not run, notification windows could be left off-screen. Whether macOS recovers such a strayed window on its own is **unknown** (not asserted either way here).
- **Full Disk Access revoked** (DB route): reads would be expected to fail. On macOS 27 this is a **silent denial** ("no longer prompts… denied by default"), not a prompt, so the app would need to detect the failure and degrade. [11]
- **Screen Recording revoked** (OCR route): capture stops; no suppression existed anyway.
- Neither permission can be requested programmatically without a prompt; the map's research constraint ("no permission prompts") applies to this research, not necessarily to the eventual product — but every viable mechanism here **requires** a user-granted permission, so the product will need at least one prompt/settings trip.

---

## 8. Concrete unsupported limits (what cannot be handled)

- Notifications that do not present as a system banner and are therefore outside AX capture: Focus-silenced, "deliver quietly", screen-locked / display-sleeping paused, and "Notifications Off" while mirroring/sharing. [25][26]
- Anything arriving while the notch app/helper is not running or Accessibility is not granted.
- **Attachments and rich content** (AX carries no attachments).
- **Arbitrary action buttons / quick reply** as a general feature (per-element AX only; map already excludes inline reply).
- **Critical alerts / Time Sensitive** without the user's explicit per-app consent. [27]
- System dialogs (crash reports, Gatekeeper, software-update prompts) which are not Notification Center banners.
- Apps that draw their own alert UI instead of posting a notification.
- Reliable **bundle identity**: AX gives a display name, not a bundle ID; resolution is heuristic and can miss helper processes. [16]

---

## 9. macOS 27 specific findings (and what remains unverified)

- **No new cross-app notification API was found in the reviewed macOS 27 release notes.** The only notifications-related entry is: "Notification Center — Fixed: You might be unable to dismiss certain notifications. (181736751)". [11] (Also seen paraphrased in secondary coverage. [29]) No new capture/suppression surface appears.
- **App-data access got tighter, not looser** (§1.2): cross-team container access is denied by default, XProtect can restrict app data, the local TCC database is no longer directly readable, and Full Disk Access is being re-scoped in public announcements. [11][12]
- **Accessibility permission naming:** Boring Notch claims macOS 27 renamed the Accessibility permission to "Device Control and Data Access". [13] We could **not** confirm this term on Apple's macOS 27 support page, which still reads "Accessibility." [20] Treat the rename claim as **unverified**.
- **UNVERIFIED on the target host (macOS 27.0.1, 26A434):**
  1. Whether the Notification Center DB under `group.com.apple.usernoted` is actually denied without a grant on macOS 27 (Apple's notes speak of "other developer teams' app… containers"; Apple's own container is not explicitly addressed).
  2. What grant (if any) a bundled app can be given for that container, and whether it restores the read.
  3. Whether the AX banner subroles and identifiers that Boring Notch relies on are unchanged on this exact build.
  4. Whether parking the banner window behaves correctly with Notification Center open, in fullscreen, and across multiple displays.
  5. Whether AX exposes redacted text under "Show Previews: Never" (see §4).
  These would require a small, on-host verification spike (permission-prompting, so a human-run step), which was out of scope for this read-only research. Until then, no statement here about behaviour on the exact host should be treated as confirmed.

---

## 10. Is the issue provable as stated?

The research ticket asked whether the app can "capture other apps' notifications **and suppress their normal banners** across **all** apps, safely and consistently." On the evidence:

- **"Capture other apps' notifications":** demonstrated for *visible banners* via the public Accessibility API (shipping example [13][16]); **not demonstrated** for quiet/Focus/locked notifications by any reviewed route — the DB is a stored-record source whose completeness and permission path on this host are unverified. [11]
- **"Suppress their normal banners":** only *post-appearance* UI-level suppression (hide the window) was demonstrated; Close/dismiss may remove the item, with unverified exact effect on Notification Center. **Pre-emptive suppression was not demonstrated** in the reviewed evidence under the constraints; injection is one possible route and is out of scope, but the reviewed evidence does not prove it is the only one. [24]
- **"Across all apps / safely and consistently":** **UNVERIFIED / not established.** "All apps" fails on any notification that does not present as a banner; "consistently" rests on undocumented, release-variable AX subroles [22][21] and on the untested parked-window behaviour [16].

So the **strongest supportable statement is:** *a narrow Accessibility-based shape — mirror visible banners, hide them off-screen, open the source app on click — is demonstrated by a shipping app; whether this also preserves Notification Center history on the target must be validated; the boundary "visible system banners eligible for presentation" (not every event an app emits) is a human decision; a database-based route that could cover stored but unshown notifications exists but its permission path and completeness on this host are unverified; and no reviewed evidence establishes a public, in-scope route to pre-emptive suppression.*

---

## 11. Open questions for the human decision (grilling ticket "Choose the notification compatibility boundary")

Recommended framing — **questions, not answers**; the human decides.

1. **Boundary.** Is "visible banners only" (i.e. banners the system chose to present) an acceptable boundary, or is capturing stored-but-unshown records also required? (If the latter, the database route is the candidate — but its permission path and completeness on this host are unverified.)
2. **Permission appetite.** Whatever grant the database route turns out to need (its exact path is unverified; Apple is actively restricting this area), is that grant acceptable, and how should the app behave if it is denied or later revoked? [12][11]
3. **Transient banner.** Is a banner that is visible for a fraction of a second before being hidden acceptable, versus "should never appear" (for which no in-scope route was demonstrated)?
4. **History.** Confirm "hide off-screen, expected to keep the Notification Center item" is the intended trade, versus "clear it" (which likely loses the item). What should happen on helper crash or revocation while banners are parked — auto-restore, or fail safe? (Recovery behaviour is currently unknown.)
5. **Identity.** Is heuristic app-name→bundle-ID resolution acceptable for click-to-open, given AX provides no bundle ID?
6. **Actions.** Confirm open-app-only is the target (no notification's own action buttons / deep links), matching the map's exclusion of inline reply.
7. **Duplicates.** If the island *re-posts* notifications as its own, that creates duplicate Notification Center entries; confirm the island does not re-post, or accept the duplication.
8. **Fragility budget.** Is depending on undocumented AX subroles — which Apple changes between releases — acceptable, and what regression testing/fallback is required when it breaks?
9. **Narrowed scope.** If a "selected apps only" fallback is chosen, record it as an **explicit human decision**; it must not be treated as already approved.

---

## 12. Sources

Primary Apple documentation and release notes:

1. `UNUserNotificationCenter` — "central object for managing notification-related activities for your app or app extension." https://developer.apple.com/documentation/usernotifications/unusernotificationcenter
2. `getDeliveredNotifications(completionHandler:)` — "all of your app's delivered notifications." https://developer.apple.com/documentation/usernotifications/unusernotificationcenter/getdeliverednotifications(completionhandler:)
3. `UNUserNotificationCenterDelegate` / `UNNotificationPresentationOptions` (own-app presentation): https://developer.apple.com/documentation/usernotifications/unusernotificationcenterdelegate
4. `com.apple.developer.usernotifications.filtering` entitlement — own-app push silencing, Apple-gated: https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.developer.usernotifications.filtering
5. Apple Developer Forums / Late Night Software forum thread "Parsing Notifications in macOS Sequoia" — "`UNUserNotificationCenter` appears to be restricted to notifications generated by the current application"; "If you click the 'Close' button or check the notification, then it's gone for good." https://forum.latenightsw.com/t/parsing-notifications-in-macos-sequoia/5001
6. SentinelLabs, "macOS Incident Response | Part 2" — notification DB readable without special permissions (pre-macOS 15); `strings` of the DB yields Slack notices. https://www.sentinelone.com/labs/macos-incident-response-part-2-user-data-activity-and-behavior
7. K. Kieczkowska, "macOS Notification Center Forensics" (2020) — `record`/`app` tables; `data` plist with `req`, `titl`, `body`, `atta`, `delivered_date`, presented/snoozed. https://kieczkowska.wordpress.com/2020/05/20/macos-notifications-forensics
8. Csaba Fitzl (@theevilbit) — macOS Sequoia moved the DB to `~/Library/Group Containers/group.com.apple.usernoted/db2/db`. https://x.com/theevilbit/status/1811758367045537990 ; coverage: https://9to5mac.com/2024/09/01/security-bite-apple-addresses-privacy-concerns-around-notification-center-database-in-macos-sequoia
9. Apple, "Accessing app group containers in your existing macOS app" — macOS 15+ app/data-container access "result in a user prompt that requests their authorization." https://developer.apple.com/documentation/xcode/accessing-app-group-containers
10. Cyberduck issue #16844 — real log of `kTCCServiceSystemPolicyAppData` prompt when reading a group container: https://github.com/iterate-ch/cyberduck/issues/16844
11. Apple, "macOS 27 Golden Gate Release Notes" — System Integrity Protection / app-data entries (161835690, 178668601, 184660124, 90775556) and Notification Center entry (181736751). https://developer.apple.com/documentation/macos-release-notes/macos-27-release-notes
12. Apple Developer News, "Updates to Full Disk Access in macOS" (2026-10-02): https://developer.apple.com/news/?id=p6zjojqw
13. Boring Notch v2.8-rc.0 release notes (2026-09-23) — notification mirroring, "only visible banners", XPC delivery, focus handling, macOS 27 accessibility terminology. https://github.com/TheBoredTeam/boring.notch/releases/tag/v2.8-rc.0
14. `BoringNotchXPCHelper.entitlements` (non-sandboxed helper), v2.8-rc.2: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/BoringNotchXPCHelper/BoringNotchXPCHelper.entitlements
15. `BoringNotchXPCHelper/Info.plist` — `NSAppleEventsUsageDescription` "boring.notch sends your replies to iMessage conversations… the helper is the process that sends the Apple event", v2.8-rc.2: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/BoringNotchXPCHelper/Info.plist
16. `BoringNotchXPCHelper/NotificationWatcher.swift` — `AXIsProcessTrusted`, `AXUIElementCreateApplication("com.apple.notificationcenterui")`, `AXObserverCreate`/`AXObserverAddNotification`, tree walk, `title`/`header`/`subtitle`/`body`/`AXAttributedDescription`, bundle-ID heuristics, and the park/restore via `kAXPositionAttribute` `(-10000,-10000)`: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/BoringNotchXPCHelper/NotificationWatcher.swift
17. `Shared/NotificationPanelDetection.swift` — banner subroles (`AXNotificationCenterBanner`, `AXNotificationCenterAlert`, `AXNotificationCenterNotification`, `AXNotificationCenterBannerWindow`, `AXNotificationCenterAlertStack`), panel exclusion, observed AX notifications: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/Shared/NotificationPanelDetection.swift
18. `boringNotch/components/Settings/Views/NotificationSettingsView.swift` — footer: "Requires [Accessibility]. Only visible banners are mirrored."; real-time authorization check: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/boringNotch/components/Settings/Views/NotificationSettingsView.swift
19. Apple, `AXUIElement` class reference — "A structure used to refer to an accessibility object… respond to messages sent by assistive applications and send notifications that describe state changes." https://developer.apple.com/documentation/applicationservices/axuielement
20. Apple Support, "Allow accessibility apps to access your Mac" — "you receive an alert, and you must specifically grant the app access." https://support.apple.com/guide/mac-help/allow-accessibility-apps-to-access-your-mac-mh43185/mac
21. `boringNotchTests/NotificationPanelDetectionTests.swift` (versioned fixture of the AX contract): https://github.com/TheBoredTeam/boring.notch/blob/v2.8-rc.2/boringNotchTests/NotificationPanelDetectionTests.swift
22. AeroSpace issue #1672 — AX approach to notification banners/Notification Center, and "banner notifications and notification center items have different AX hierarchies, and they shift between macOS versions": https://github.com/nikitabobko/AeroSpace/issues/1672
23. DynamicLake product page — notified app set (Messages/WhatsApp/Telegram/Slack etc.) as a narrowed scope: https://www.dynamiclake.com
24. QuietYou README (briankendall/QuietYou) — AX observers + pressing the X button; "there's no good way to stop the notification from appearing at all"; injection listed as future work: https://github.com/briankendall/QuietYou
25. Apple Support, "Notifications settings on Mac" — "Show previews: Always/When Unlocked/Never" (and per-app); "Show Notifications: when display is sleeping / when screen is locked"; alert style None/Banners/Alerts: https://support.apple.com/guide/mac-help/mh40583/mac and https://support.apple.com/guide/mac-help/change-notifications-settings-on-mac-mchl205da693/mac
26. Apple Support (Apple's own video), Focus behaviour — "When your Focus is active, your notifications will go directly to Notification Center by default" (iPhone wording; Mac guide: silenced/allow per-person and per-app): https://support.apple.com/guide/mac-help/set-up-a-focus-to-stay-on-task-mchl613dc43f/mac
27. Apple Support, "View app notifications on Mac" — time-sensitive/critical notifications ask permission the first time: https://support.apple.com/guide/mac-help/view-app-notifications-mh40609/mac
28. `boringNotch/managers/SystemNotificationManager.swift` — click-to-open via `NSWorkspace.activate`/`urlForApplication`: https://raw.githubusercontent.com/TheBoredTeam/boring.notch/v2.8-rc.2/boringNotch/managers/SystemNotificationManager.swift
29. Secondary corroboration of macOS 27 notification notes and build identity (macOS 27 "Golden Gate", 26A428/26A434): https://blakecrosley.com/blog/macos-27-release-notes ; https://mrmacintosh.com/macos-tahoe-27-0-1-update-everything-you-need-to-know

Additional background read (not cited for a substantive claim above): Apple release notes for the Full Disk Access re-scoping context (12); Objective-See's notification-DB write-up and AuRevoir tool (the pre-Sequoia `$TMPDIR` DB path) https://objective-see.org/blog/blog_0x2E.html ; `ncprefs.py` for the per-app `com.apple.ncprefs.plist` settings (a system-preference change, out of scope): https://github.com/drewdiver/ncprefs.py
