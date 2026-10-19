# Issue tracker configuration

Notchlight is a personal, native macOS project and is **planning only**: this repository holds no app code and authorizes no implementation.

Tickets are GitHub Issues in `mujak27/notchlight`. Native GitHub sub-issues and dependencies are the tracker structure; body metadata conventions from the former local tracker are retired. There is no local scratch tracker, skills installer, or additional framework.

## Canonical sources

- Map: [Notchlight — Wayfinding Map](https://github.com/mujak27/notchlight/issues/1) (issue #1) — destination, notes, decisions so far, fog, and out of scope.
- Issues: native child issues of the map — one issue per ticket, under [All issues](https://github.com/mujak27/notchlight/issues).
- Reports: `research/` — permanent evidence reports; they are not tickets.
- Glossary: `GLOSSARY.md` — the effort's domain language.

The map is canonical for decisions and fog; each issue is canonical for its own question and resolution.

## Labels and types

- `wayfinder:map` — the effort map.
- `wayfinder:research` — an evidence ticket.
- `wayfinder:grilling` — a human decision ticket.
- `wayfinder:prototype` — a bounded on-host feasibility ticket (`prototype HITL`).

All eight tickets are child issues of the map: three `wayfinder:research`, four `wayfinder:grilling`, and one `wayfinder:prototype`. GitHub issue types are not used; the type label is authoritative, and the issue body names the type.

## Hierarchy and dependencies

- Every ticket is a native sub-issue of the map. The map's sub-issue list is the hierarchy and priority order; filenames carry no order.
- A blocker is a native GitHub blocked-by link, not a `Blocked by:` body line. Completed blockers remain linked.
- Native sub-issues and dependencies are visible in the GitHub issue UI; no parallel body convention is maintained.

Inspect the hierarchy and dependency graph with:

```sh
gh api repos/mujak27/notchlight/issues/1/sub_issues
gh api repos/mujak27/notchlight/issues/<number>/dependencies/blocked_by
```

## Claim and workflow

1. **Claim before work.** Assign `mujak27` before any work starts: `gh issue edit <number> --repo mujak27/notchlight --add-assignee mujak27`.
2. **Check blockers.** A ticket is unblocked only when every native blocked-by issue is closed.
3. **Frontier.** The next ticket is the first open, unassigned child in the map's sub-issue priority order whose blocked-by issues are all closed.
4. **Work.** Update the issue with findings, evidence, and any resolution comment; do not rewrite the original question.
5. **Resolve research.** Add a resolution comment containing the answer and a link to its report under `research/`, then close the issue as completed.
6. **Resolve human decisions.** Leave the issue open until the human chooses; record the decision in a resolution comment and close it as completed.
7. **Update the map.** Add only closed decisions, as one-line title links under `## Decisions so far`. Do not list open child issues on the map.

Wayfinding, grilling, and domain-modeling remain the working skills for this effort; this repository adds no installer or agent configuration.
