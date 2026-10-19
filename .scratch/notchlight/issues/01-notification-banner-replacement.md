# Can Notchlight safely replace macOS notification banners?

Type: research
Labels: wayfinder:research
Parent: ../map.md
Assigned to: mujak27 (research worker acting for dev)
Status: resolved

## Question

Can Notchlight capture other apps' notifications into the island and suppress their normal banners across all apps, safely and consistently? Answer with evidence and feasibility options, not with a build.

The answer must cover:

- **Other-app capture**: which mechanisms can read notifications produced by other apps (public APIs, the Notification Center database, Accessibility APIs, private APIs)? What does each expose, and what permission does each need?
- **Banner suppression independently**: is suppression separable from capture? Can Notchlight suppress a banner while still receiving the content, or does suppressing also remove capture (or the reverse)?
- **Content availability before suppression**: is the full content (app, title, body, thread, actions, attachments) available before the banner would appear, so suppression does not silently lose information?
- **Public vs private vs Accessibility**: what each mechanism permits, the permission it requires, and how tolerant it is to OS updates.
- **Focus / lock / per-app settings consistency**: does behaviour respect Focus, lockscreen privacy, and per-app notification settings, or does it bypass them?
- **Click / dismiss / Notification Center semantics**: after capture and suppression, what does clicking a preview do (open the source app) and what does dismissing do? Does the notification still reach Notification Center? Can anything be duplicated or lost?
- **Permission revocation**: what happens when the required permission is revoked mid-session?
- **Unsupported limits**: which apps or notification kinds cannot be handled.

Constraints: no SIP disabling and no system modifications. Report the feasibility options (full replacement, constrained fallback, no-go) with their evidence; do NOT make the human decision — that belongs to the ticket "Choose the notification compatibility boundary".

## Comments

- 2026-10-19 wayfinder charting: split from the map's notification thread. Target is capture-from-all-apps plus banner suppression; feasibility UNPROVEN. This ticket produces evidence, not a decision.
- 2026-10-19 claim: Assigned to mujak27 (research worker acting for dev); Status set to `claimed` and saved before research dispatch.
- 2026-10-19 context pointer (research dispatch): branch `research/notifications` (based on `origin/main`); findings report at [`research/notifications.md`](../../../research/notifications.md).
- 2026-10-19 research dispatched: findings-only report requested; no repo writer conflict. This records dispatch only; the ticket is neither complete nor resolved.
- 2026-10-19 resolved: research answered below; Status set to `resolved`, with the assignment and claim history above preserved. Evidence report: [`research/notifications.md`](../../../research/notifications.md). Research only — it approves no compatibility boundary, no fallback, and no private API.

## Answer

Reviewed sources found **no dedicated supported public cross-app notification API**; the public UserNotifications surface is own-app only. The public **Accessibility API (AX)** is the only no-SIP route found that reads other apps' notifications, and a shipping open-source notch app (Boring Notch) demonstrates it: public AX code captures **visible system banners** and then **moves the banner windows off-screen** to hide them.

That is **not proof of instant, no-flash replacement** — the banner exists before AX can act, and pre-emptive suppression (never showing a banner) was not demonstrated in the reviewed evidence. Nothing was tested on the exact host (macOS 27.0.1, build 26A434, Mac17,8), so no host behaviour is confirmed.

The remaining behaviours are unproven as well: whether AX returns **redacted text under the privacy/Show Previews settings**, whether off-screen parking preserves **Notification Center history**, and whether a parked banner is **restored after a helper crash or permission loss**.

**Focus- and per-app-silenced events are intentionally not surfaced**: they never present as banners, so AX never sees them, which is consistent with the intent to honour Focus and per-app preferences.

The Notification Center **database route is not an approved path** — its privacy, permissions, and write timing versus a banner are unverified, and macOS 27 denies cross-container access by default.

**No fallback and no private API were approved by this research.** The evidence supports only a candidate boundary (mirror visible banners, hide off-screen); choosing the boundary is the human decision in "Choose the notification compatibility boundary".

Sources of record: Apple `UNUserNotificationCenter` and `AXUIElement` documentation; Boring Notch v2.8-rc source and release notes.
