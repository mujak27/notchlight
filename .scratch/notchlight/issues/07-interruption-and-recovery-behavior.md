# Choose interruption and recovery behavior

Type: grilling
Labels: wayfinder:grilling
Parent: ../map.md
Blocked by: 04, 05
Assigned to:
Status: open

## Question

Once the notification compatibility boundary and the first supported media sources are chosen, what interruption and recovery behaviour does the human require for Notchlight?

The decision must define:

- **Notification interruption**: how a notification interrupts media in the island — queueing, display timeout, dismissal, and how media is restored afterwards.
- **Permission loss**: behaviour when a required permission is denied at launch or revoked mid-session, including restoring anything that was hidden before the permission was lost.
- **Adapter breakage**: behaviour when a notification or media mechanism stops working, including after an OS update, and whether the system's own path is the fallback.
- **Lock, sleep, and wake**: behaviour across lock and unlock and across sleep and wake.
- **No hidden or lost notifications**: how Notchlight avoids leaving a notification hidden and never shown, and preserves the system's own fallbacks.

This fixes the failure and recovery story that feasibility and the compatibility boundary leave open.

## Comments

- 2026-10-19 wayfinder charting: graduated from the map's now-sharp failure/recovery fog, now bounded by the notification boundary and media source decisions. Blocked by those two human decision tickets; unblocked when both are resolved.
- No claim; unassigned. Resolve with the human in the loop.
