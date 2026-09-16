# ADR 0267: User-Configurable Automatic Update Checks

- Status: Accepted
- Date: 2026-09-16
- Deciders: PI-Desktop core
- Related: D433, ADR 0022, ADR 0197

## Context

ADR 0022 gives every packaged install an unconditional background schedule:
an initial GitHub Releases check 15s after boot and a 6h interval. Users on
metered or controlled networks, or who simply prefer to update on their own
schedule, have no way to stop the polling without losing the manual check
lane.

## Decision

1. `AppSettings.autoUpdate?: boolean` persists the preference through the
   existing host-owned settings path. Absent or `true` keeps the historical
   always-on schedule; `false` stops only the scheduled background checks.
2. Electron Main owns the gate. `AppUpdaterController.setAutoChecksEnabled`
   disposes the pending timers when disabled and restarts the delayed,
   time-bounded schedule when re-enabled; `startAutoCheck` refuses to
   schedule while disabled. The settings write path applies the toggle via
   `applyAutoUpdateSetting`; boot reads the persisted value before
   scheduling, and a failed read keeps the historical always-on behavior.
3. Manual checks are unaffected: the application menu item and the
   Settings → About row keep working with their usual status surfacing.
   Delivery modes (ADR 0022) are unchanged, and a downloaded update stays
   actionable until install or normal shutdown regardless of the toggle.
4. The renderer exposes one `role="switch"` row in Settings → About above
   the Updates row, hidden in development builds where the updater is
   disabled outright. Localized labels ship in all shipped locales.

## Consequences

- Users can silence background update polling without losing discovery or
  install capability; nothing about feeds, signing, or delivery changes.
- The preference rides the existing settings persistence, so no schema,
  migration, or new IPC surface is introduced.
- An in-flight check that overlaps a disable keeps its ambient failure
  semantics; the next schedule simply never starts until re-enabled.

## Alternatives

- Per-channel granular controls (check interval, download-only-off): rejected
  as speculative; a single boolean covers the actual request.
- Renderer-side suppression of the banner only: rejected because it leaves
  the network polling running, which is the thing being turned off.
- Reusing `UpdateMode` ("manual" delivery modes already skip downloads):
  rejected because delivery mode is platform-owned, not a user preference.
