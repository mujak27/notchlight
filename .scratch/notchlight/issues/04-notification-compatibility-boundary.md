# Choose the notification compatibility boundary

Type: grilling
Labels: wayfinder:grilling
Parent: ../map.md
Blocked by: 01
Assigned to:
Status: open

## Question

Given the evidence from ticket 01, which notification compatibility boundary does the human choose for Notchlight?

Options to weigh (not exhaustive; the human may choose something else):

- **Full replacement**: capture from all apps and suppress all normal banners.
- **Constrained fallback**: capture and suppress only where it is reliable, and leave everything else to the system.
- **No-go**: notifications stay entirely with the system, and Notchlight is media-only or dropped.

The decision must weigh, against ticket 01's evidence:

- no duplicate notifications and no lost notifications;
- privacy, including lockscreen privacy and Focus and per-app notification settings;
- the permissions and private-API tradeoff, including OS-update fragility.

Do NOT assume the constrained fallback is approved; choosing the boundary is exactly what this ticket is for.

## Comments

- 2026-10-19 wayfinder charting: created as the human decision gate that consumes ticket 01's evidence. Blocked by 01; unblocked once 01 is `resolved`.
- No claim; unassigned. Resolve with the human in the loop.
