# Agent Note: A dragged pet keeps its position through a restart

Status: implemented

## Problem

The pet's floating-window layout persists in `$DSH_HOME/pet.json`, and dragging
writes it there (`onDragEnd` -> `POST /api/pet/set-config` -> `pet.json`). Two
defects broke the round trip for every install whose pet row is renamed by an
aggregate bundle — which is the install the market hands out.

**The settings mirror addressed the wrong namespace.** `syncSettingsFromPet`
wrote a dragged layout into the settings document through
`settings.update(PET_SETTINGS_NAMESPACE, ...)`, a hardcoded `'pet'`. Under
`dsh-web-all` the row is inserted as `web-ui-pet`, so the write raised
`No configurable plugin entry "pet"`, the caller swallowed it as best-effort,
and the profile layer never learned a position at all. The browser half already
knew this: its `servedEntryId` resolves the same rename. Only the host half
assumed the package's own plugin name.

**A mount reset the layout to the schema defaults.** `applyImpl` applies the
row's own config at mount. A display field the profile never committed reads
back exactly like one committed to the schema default, so `petSettingsSection`
returned `right: 24, bottom: 20` — the bottom-right corner — and
`applySettingsSection` wrote that over both the running config and `pet.json`.
Every start therefore snapped the pet back to the corner, which is what users
reported, and it also undid the first drag of a fresh install.

## Decision

**The persisted layout is the fallback, and the settings namespace is the row's
own id.** Two shapes follow.

`petSettingsSection` gains `persisted` (what `pet.json` holds) and `committed`
(the profile layer's explicit values), and resolves each display field in that
order:

1. an explicit profile value, so a choice that happens to equal the schema
   default is still a choice;
2. a live value that differs from the schema default, i.e. a runtime commit;
3. the persisted value, so a field the profile never touched keeps the position
   the user dragged.

Step 3 is the fix. Steps 1 and 2 exist so the fallback cannot mask a real edit:
a committed value always outranks the persisted one, and a runtime commit at the
default is caught by the profile layer rather than by the default comparison.

`servedRow` resolves the settings row the way the browser half does: the first
of `web-ui-pet`, `ui-pet`, `pet` that the settings service actually describes.
`apply` hands the resolved id to `PetService.settingsNamespaceProvider`, and
`syncSettingsFromPet` addresses it. The lookup is lazy, so a mount that runs
before the settings service describes the row simply falls back to the persisted
layout — the safe direction — and every later write uses the real id.

`committed` comes from the settings service's own `describe()`, whose `user`
layer is the profile's explicit values (`projectForm` keeps only keys the layer
actually carries). The host half reaches `configEditor` nowhere.

## Alternatives considered

- **Drop the schema `.default()` on the display fields and let the persisted
  value be the only fallback.** Rejected. The Host derives the settings page from
  `Config`, and those defaults are the documented base of the form's
  inherit/reset behaviour; `petId` can omit one because it is not a number the
  page edits as a defaulted field.
- **Read the profile layer from `configEditor.configuration()` and match the row
  by `entry.fiber`.** Rejected as the primary mechanism: it reaches into a
  service this plugin does not inject and duplicates the rename knowledge the
  browser half already encodes in `servedEntryId`. Two halves reading one
  namespace through one rule is the property worth keeping.
- **Treat "the profile committed nothing" as "the section has no opinion" and
  skip the display fields entirely.** Rejected: the section still has to carry
  `visible` and `size`, and skipping fields makes an unset display read as
  whatever the ledger happens to hold even when the user did commit a value.
- **Re-clamp the restored position against the current viewport.** Rejected as
  out of scope. It is a real gap — a narrower window can still place the pet
  off-screen — but it is a different defect, and fixing it here would hide the
  failure mode this note is about.

## Consequences

- A drag survives a restart on an aggregate install, and a fresh install no
  longer loses its first drag.
- The settings document now learns the dragged position, so the settings card
  shows the layout the pet actually has instead of the schema defaults.
- Five tests were added: three pure precedence cases in `src/index.test.ts`, one
  mount-level case and one mirror-namespace case in
  `tests/pet-host-settings.spec.ts`. Verified failing against the unfixed source
  (five failures, the layout reading back as `right: 24 / bottom: 20`) and
  passing after it.
- Verification: `pnpm typecheck`, the full `pnpm test` suite and `pnpm build`
  pass, plus CI on the pull request head.
- A profile that commits a display value exactly equal to the schema default
  while `pet.json` holds a different one keeps the persisted value until the
  next settings announcement, because at mount only the default comparison is
  available. The next drag or settings edit resolves it; the layout is never
  lost.
