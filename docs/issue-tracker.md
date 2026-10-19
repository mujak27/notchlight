# Issue tracker configuration

Notchlight is a personal, native macOS project and is **planning only**: this repository holds no app code and authorizes no implementation.

The tracker is local Markdown. There is no issue server, no skills installer, and no additional framework or configuration. This file records the configuration already used by the repository; it does not add behavior.

## Layout and canonical sources

- Map: `.scratch/notchlight/map.md` — destination, notes, decisions so far, fog, and out of scope.
- Issues: `.scratch/notchlight/issues/NN-<slug>.md` — one file per ticket.
- Reports: `research/` — permanent evidence reports; they are not tickets.
- Glossary: `GLOSSARY.md` — the effort's domain language.

The map is canonical for decisions and fog; each ticket file is canonical for its own question, status, and answer.

## Labels

- `wayfinder:map` — the effort map.
- `wayfinder:research` — an evidence ticket.
- `wayfinder:grilling` — a human decision ticket.
- `wayfinder:prototype` — a bounded on-host feasibility ticket (`prototype HITL`).

Each file's `Labels:` line carries the matching label.

## Ticket metadata

Each ticket has a `Type:` of `research`, `grilling`, or `prototype HITL`, and a `Status:` of `open`, `claimed`, or `resolved`.

- `Assigned to:` names the owner once claimed. Leave it empty while the ticket is unclaimed.
- `Blocked by:` lists ticket numbers (`NN, NN`) that must be `resolved` first. Omit it when nothing blocks the ticket.
- Reference a ticket by number and name: `NN` plus the file stem or H1, e.g. `01-notification-banner-replacement` / "Can Notchlight safely replace macOS notification banners?".

## Workflow

1. **Claim before work.** Set `Assigned to:` and `Status: claimed`, then save before any work starts.
2. **Check blockers.** A ticket is unblocked only when every `Blocked by:` ticket is `resolved`.
3. **Frontier.** The next ticket is the open, unassigned ticket whose blockers are all resolved; scan `issues/` in filename order and take the first.
4. **Resolve.** Append the answer under `## Answer`, set `Status: resolved`, and add a short gist of the decision or evidence to the map's `## Decisions so far` as a title link to the ticket.

Do not mark a research ticket resolved without its report, and do not turn evidence into a decision. Human decision tickets stay open until the human chooses.
