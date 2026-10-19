# Notchlight

Personal, native macOS notch-island app for notification replacement and compact media control.

**Planning only.** This repository holds planning artifacts and no app code; nothing here describes a finished implementation. The foundational research (notifications, media, overlay) is complete and reviewed, but none of it was tested on the target Mac: compatibility remains unverified and feasibility is unproven, so no implementation is authorized.

## Planning artifacts

- [Effort map](https://github.com/mujak27/notchlight/issues/1) — destination, notes, decisions so far, fog, and out of scope.
- [Issues](https://github.com/mujak27/notchlight/issues) — the map's native GitHub child issues.
- [Research](research/) — [notification replacement](research/notifications.md), [media sources](research/media.md), and [native overlay](research/overlay.md) findings.
- [Glossary](GLOSSARY.md) — this effort's domain language.
- [Issue tracker configuration](docs/issue-tracker.md) — the GitHub Issues tracker's labels, hierarchy, and workflow.

## GitHub issue tracker

Tickets are GitHub Issues on `mujak27/notchlight`; the [effort map](https://github.com/mujak27/notchlight/issues/1) is issue #1. The full configuration is in [docs/issue-tracker.md](docs/issue-tracker.md).

- **Claim**: assign `mujak27` to the issue before any work starts.
- **Dependencies**: native GitHub blocked-by links, not body conventions. A ticket is unblocked only when every blocked-by issue is closed.
- **Frontier**: the first open, unassigned child in native sub-issue priority order whose blocked-by issues are all closed.
- **Resolve**: add a resolution comment, then close the issue as completed; research closes only with its evidence report linked.
- **Map**: add only closed decisions, as one-line title links under Decisions so far; the map does not index open children.

## Research

The three research reports are permanent project documentation. They are evidence, not decisions: they describe documented mechanisms, confidence, and open questions, and they do not approve a compatibility boundary, a source list, a resource budget, or any implementation.
