# ADR-00XX - {decision in present tense}

One decision per file. Filename: `ADR-NNN-short-kebab-title.md` (template is `ADR-000-template.md`). Proposed records may be reordered so the index matches dependency order. Do not rewrite or renumber an Accepted record - supersede it.

Copied from [The Kata Log](https://github.com/TheKataLog) templates (Five Nines, Pragmatic, BluzBrothers, CELUS Ceals) and trimmed to what this team will actually fill.

## Date

YYYY-MM-DD

## Status

Proposed | Accepted | Superseded by ADR-NNNN | Deprecated | Rejected

## Context

Why a decision is needed now. Assumptions, volumes, and constraints. Mark unknowns as TBD - do not invent numbers. Say what this record does **not** decide.

## Evaluation criteria

What we will score the options against (architecture characteristics or hard constraints). Mark the driving ones.

- **{criterion}** - {measurable target or TBD}

## Options

- **{Option A}**: {one or two sentences}
- **{Option B}**: {one or two sentences}

## Decision

The chosen option, in present tense, and the one or two criteria that decided it. Why the runners-up lost on those criteria.

## Key differentiators

Why this option fits the criteria (Five Nines). Bullet the few things that are not true of the rejected options.

- **{point}**

## Architecture characteristics

Which named characteristics this decision moves, in both directions. A decision that improves everything and weakens nothing has not been analysed. Mark the driving ones.

| Characteristic | Effect | Why |
|---|---|---|
| {e.g. availability, testability, data integrity, simplicity, performance, portability, operability} | Improved (driving) / Improved / **Weakened** | {one line} |

**Deliberately downplayed: {characteristic}.** Why it is acceptable to lose it here, and where the opposite call would be correct.

**Fit with the existing architecture.** How this sits with decisions already in the repository - same offline story, same event backbone, same human-in-the-loop posture - rather than introducing a second philosophy. This is a kata judging criterion.

## Consequences

### Positive

- {what improves, tied to a criterion}

### Negative

- {what gets worse, harder, or more expensive - never leave this empty}

### Strengthened characteristics

- {characteristic name from the funnel} ({brief reason tied to the option chosen})

### Weakened characteristics

- {characteristic name from the funnel} ({what the decision costs in that dimension - never leave empty})

## Risks & trade-offs

| Risk area | Description | Mitigation |
|---|---|---|
| {area} | {what can go wrong} | {what we do about it} |

## Verification

How we will know the decision is holding (fitness function, CI test, ops checklist). Omit only if the risk table already names the check.

## Conclusion

One or two sentences. Related ADRs by number, not a second decision.
