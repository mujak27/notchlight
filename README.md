# Notchlight

Personal, native macOS notch-island app for notification replacement and compact media control.

**Planning only.** This repository holds planning artifacts and no app code; nothing here describes a finished implementation. The foundational research (notifications, media, overlay) is complete and reviewed, but none of it was tested on the target Mac: compatibility remains unverified and feasibility is unproven, so no implementation is authorized.

## Planning artifacts

- [Effort map](.scratch/notchlight/map.md) — destination, notes, decisions so far, fog, and out of scope.
- [Tickets](.scratch/notchlight/issues/) — one file per ticket, `.scratch/notchlight/issues/NN-<slug>.md`.
- [Research](research/) — [notification replacement](research/notifications.md), [media sources](research/media.md), and [native overlay](research/overlay.md) findings.
- [Glossary](GLOSSARY.md) — this effort's domain language.
- [Issue tracker configuration](docs/issue-tracker.md) — the local Markdown tracker's layout, metadata, and workflow.

## Local issue tracker

Tickets are local Markdown files; there is no issue server. The full configuration is in [docs/issue-tracker.md](docs/issue-tracker.md).

- **Claim**: a ticket is claimed by recording `Assigned to:` and `Status: claimed` and saving before any work starts.
- **Dependencies**: a `Blocked by: NN, NN` line near the top lists the tickets that must be `resolved` first; a ticket is unblocked when every listed ticket is resolved.
- **Frontier**: the next ticket to work is the open, unassigned ticket whose blockers are all resolved — scan `issues/` in filename order and take the first.
- **Resolve**: append the answer under `## Answer`, set `Status: resolved`, and add a short gist with a title link to the map's Decisions so far.

## Research

The three research reports are permanent project documentation. They are evidence, not decisions: they describe documented mechanisms, confidence, and open questions, and they do not approve a compatibility boundary, a source list, a resource budget, or any implementation.
