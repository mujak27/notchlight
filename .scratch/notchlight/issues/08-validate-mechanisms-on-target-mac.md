# Validate selected mechanisms on the target Mac

Type: prototype HITL
Labels: wayfinder:prototype
Parent: ../map.md
Blocked by: 04, 05, 06, 07
Assigned to:
Status: open

## Question

Do the selected notification, media, and overlay mechanisms meet the accepted compatibility boundaries and lightweight criteria on macOS 27.0.1 (build 26A434), Mac17,8 (Apple M5 Pro) — or must the specification revise its scope?

This is a bounded evidence question, not an implementation deliverable. It contemplates a future, small, throwaway on-host spike run with a live human at the machine. **No code, no spike, and no permission changes happen in this planning session.**

Evidence the spike must produce:

- **Notification capture ordering**: whether capture happens before a banner is hidden, and what the no-flash expectation really looks like — including how a fraction-of-a-second visible banner behaves.
- **Preview privacy**: what is actually captured under Preview, Focus, and lock settings, especially any redaction.
- **History and recovery**: whether Notification Center history survives suppression, whether hidden banners are restored after permission revocation or a crash, and how each mechanism fails when revoked.
- **Media sources**: real metadata, artwork, and transport control per candidate source, plus sleep/wake recovery and multiple-source behaviour.
- **Overlay**: hover expand/collapse, full-screen and Spaces behaviour, and the resource measurement workload for idle CPU, energy, and memory, with no pre-set numbers.

The specific mechanism unknowns to check are listed in the research reports, which this ticket points to rather than restating: the [notification](../../../research/notifications.md), [media](../../../research/media.md), and [overlay](../../../research/overlay.md) reports. Permission prompting is expected, so a human must run it.

## Comments

- 2026-10-19 wayfinder charting: created as a bounded prototype whose only purpose is feasibility evidence for the selected mechanisms; it authorizes no app build. Blocked by the decisions it must be guided by — the notification boundary, the media sources, the acceptance criteria, and interruption/recovery — so the human's choices bound the spike.
- No claim; unassigned. Resolve with the human in the loop.
