# Agent Note: The 2026-09-29 pet intake round

Status: implemented

## Problem

The 2026-09-29 maintenance round reviewed the one open pull request in this
repository, #2 (`fix(pet): keep a dragged position through a restart`). The pull
request is a bug fix with a diagnosed two-part root cause and five new test
cases, and it arrived with green CI. What CI cannot establish is whether the new
cases actually gate the fix or merely describe it, and whether the described
blast radius is real - both of which decide whether the fix may merge.

## Decision

**#2 was merged.** The fix was reproduced rather than read. At the pull request
head, `src/index.test.ts` and `tests/pet-host-settings.spec.ts` pass 18/18, the
full `vitest run` is 563 passed, `tsc -p tsconfig.json --noEmit` is clean, and
`node --test scripts/dsh-pet-migrate-v2.test.mjs scripts/dsh-pet.test.mjs` is 22
passed. As a control, only `src/index.ts` was reverted to its pre-pull-request
content while keeping the five new cases: the result is 5 failed | 13 passed,
with the layout reading back as `right: 24 / bottom: 20` and the settings mirror
writing to `pet`. That is exactly the failure the description predicts, so the
new tests are a gate rather than a narration.

The root cause is correctly identified on both branches. The settings mirror
addressed a hardcoded `pet` namespace while an aggregate bundle serves the row
as `web-ui-pet`, so the write raised and was swallowed as best-effort; and a
mount could not distinguish a display field the profile never committed from one
committed to the schema default, so every start rewrote the layout to the
corner. Resolving the serving row lazily and ranking `committed` above a
differing live value above `persisted` fixes both without reaching into
`configEditor`.

The change touches `src/`, `tests/` and `.agents/notes/` only. Nothing under
`assets/`, which is this repository's market input, so no content re-pin
accompanies it.

## Alternatives considered

- **Accept the description's claim that the five cases fail against the old
  source.** Rejected. "The tests are new" and "the tests would catch the bug" are
  different claims; the revert-and-rerun control is what separates them, and it
  cost one file swap.
- **Treat the un-clamped restored position as part of this fix.** Rejected. A
  narrower window can still place the pet off-screen, but that is a distinct
  defect with its own reproduction, and folding it in would have hidden the
  round-trip failure this pull request is about. The author disclosed it.
- **Require a re-pin because `src/` changed.** Rejected. The market reads
  `assets/`, so plugin source and tests are not store content; moving the
  gitlink would publish a pin whose diff the build cannot see.

## Consequences

- A drag survives a restart on an aggregate install, and a fresh install keeps
  its first drag.
- The five new cases are now the regression gate for this area, and the
  revert-and-rerun control is recorded here as the check that established it.
- The un-clamped position remains a known, separate gap; the next round can
  start from the author's own scope note rather than re-deriving it.
