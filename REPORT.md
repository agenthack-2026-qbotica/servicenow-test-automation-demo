# Issue #64 (Test27) — Fix Report

## Issue Summary

Test27 requires the automation suite to validate that ServiceNow blocks resolving an
incident when the **Resolution Notes** field is left empty (a mandatory-field business
rule), and that the incident stays in its prior state with a validation message shown.

The failure report attached to this issue, however, described two **automation
reliability defects** discovered while attempting to exercise that flow, not a business
logic violation in ServiceNow itself:

1. `TestCases/01_TC_CreateIncident.xaml` — the robot could not find a required UI
   element (selector/state/synchronization problem).
2. `TestCases/04_TC_ResolveHighPriorityWAssignGroup.xaml` (the file in this repo with
   equivalent behavior is `TestCases/04_TC_HighPriorityWAssignGroup.xaml`, which drives
   `Workflows/04_HighPriorityWAssignGroup.xaml`) — the robot attempted to interact with a
   target element before it was actually enabled/interactive.

**Root cause confirmed:** both symptoms trace back to the same defect pattern —
individual UI-automation activities configured with `WaitForReadyArgument="None"`,
meaning UiPath does not wait for the target element to become interactive (visible +
enabled) before acting on it. Every sibling activity in these same workflows correctly
uses `WaitForReadyArgument="Interactive"`; the affected activities were outliers left
over from an earlier authoring pass, and under normal ServiceNow page-load/AJAX latency
they race ahead of the DOM and either fail to find the element or hit it while it's still
disabled.

- In `Workflows/01_CreateIncident.xaml`, the `Keyboard Shortcuts 'Input_Caller'` activity
  (sends Enter to confirm the autocomplete suggestion in the Caller reference field)
  fired with no readiness check. If ServiceNow's autocomplete AJAX call for the Caller
  field hasn't populated yet, the Enter keypress lands on the wrong state, the Caller
  field doesn't resolve correctly, and downstream steps (reading the incident `Number`
  field, etc.) then fail with "element not found."
- In `TestCases/04_TC_HighPriorityWAssignGroup.xaml`, the entire inline `... Then`
  verification block (Click Favorites → Click Home → Type incident number → click list
  item → click "Resolution Information" tab → read "Resolved" field) had
  `WaitForReadyArgument="None"` on every target, unlike the reliable canonical workflow
  `Workflows/04_HighPriorityWAssignGroup.xaml`, which uses `"Interactive"` throughout.
  Clicking the "Resolution Information" tab or subsequent controls before ServiceNow's
  SPA has finished rendering/enabling that region manifests exactly as "attempted to
  interact with a disabled target element."

## Changes Made

- **`Workflows/01_CreateIncident.xaml`** — changed `WaitForReadyArgument` from `None` to
  `Interactive` on the `NKeyboardShortcuts` target for `Input_Caller` (the Enter-key
  confirmation of the Caller reference field), so the shortcut only fires once the field
  is genuinely interactive.
- **`TestCases/04_TC_HighPriorityWAssignGroup.xaml`** — changed `WaitForReadyArgument`
  from `None` to `Interactive` on all 6 affected targets in the `... Then` sequence
  (`Button_Favorites`, `Home`, `Input_HomeSearch`, `Link_SelectIncident`,
  `Button_ResolutionInformation`, and the `Resolved` field read), matching the
  synchronization pattern already used everywhere else in this file and in the
  equivalent canonical workflow.

No selectors, references, or Object Repository entries were changed — only the
synchronization attribute already exposed by these activities for exactly this purpose.

## UiPath Analysis

- Both affected files use the `uix:` (UI Automation Next / modern) activity set with
  `TargetAnchorable` selectors resolved against the Object Repository app
  `_BU5etMzk0KtDV4Hwc6ztw`. `WaitForReadyArgument="Interactive"` is the built-in
  mechanism for waiting until an element is visible **and** enabled before the activity
  acts on it; `"None"` skips that wait entirely.
- `Workflows/02_UpdateIncidentPriorityToHigh.xaml` and `Workflows/04_HighPriorityWAssignGroup.xaml`
  (the reusable business-logic workflows invoked by the test cases) were already
  consistent, using `"Interactive"` throughout — confirming the defect was isolated to
  the two files named in the issue and not systemic.
- **Follow-up not in scope of this fix:** `TestCases/02_TC_UpdateIncidentPriorityToHigh.xaml`
  contains 5 more occurrences of the same `WaitForReadyArgument="None"` pattern in its
  own inline verification block. It wasn't named in this issue and wasn't touched here,
  but it carries the same latent risk and should be corrected in a follow-up pass.
- No dedicated negative test workflow for "resolve with empty Resolution Notes" (the
  actual Test27 acceptance criteria) exists yet in `TestCases/` or `Workflows/`. Building
  one requires recording a live selector for ServiceNow's mandatory-field validation
  banner/message against the real PDI in UiPath Studio — that cannot be done from this
  environment (no interactive browser/Studio access here), so it is called out as
  required follow-up rather than guessed at with fabricated Object Repository
  references.

## Testing Notes

After this fix, verify from UiPath Studio (or Test Manager) against the
`dev181290.service-now.com` PDI:

1. Run `TestCases/01_TC_CreateIncident.xaml` — confirm the Caller field resolves
   correctly before the Enter key fires, the incident is created, and the `Number` field
   read-back succeeds without an "element not found" fault.
2. Run `TestCases/04_TC_HighPriorityWAssignGroup.xaml` — confirm the inline "Then"
   verification (favorites → home → search → select incident → Resolution Information
   tab → Resolved field) no longer intermittently fails on a not-yet-interactive
   control.
3. Re-run both a handful of times under normal network latency, since the original
   defect was a race condition — a single clean pass does not fully prove the fix under
   variable ServiceNow response times.
4. `uip rpa get-errors --project-dir "."` could not be run to completion in this sandbox
   (the `uip` CLI requires cloud authentication not configured here); both edited files
   were confirmed to be well-formed XML via `xml.etree.ElementTree`. Re-run
   `uip rpa get-errors` in an authenticated environment before merging.
5. Author and validate the actual Test27 negative scenario (empty Resolution Notes
   blocks Resolve, shows validation message, incident state unchanged) as a follow-up —
   this requires live selector recording in Studio against the real PDI.

## Risk Assessment

- **Low risk.** The change only tightens synchronization (wait-for-interactive) on
  activities that already targeted the correct elements; it does not alter selectors,
  business logic, or test assertions.
- Adding a readiness wait can very slightly increase execution time per activity (bounded
  by each activity's existing timeout), but this trades a small, predictable wait for
  eliminating a race-condition failure mode — a net reliability improvement.
- Because `TestCases/02_TC_UpdateIncidentPriorityToHigh.xaml` still has the same
  `WaitForReadyArgument="None"` pattern (not fixed here, out of scope), it remains
  exposed to the identical class of intermittent failure and should be prioritized next.
- The actual Test27 business-rule scenario (empty Resolution Notes blocking Resolve) is
  still not automated. Until that negative test exists, this issue's acceptance criteria
  are only partially covered by this change — this fix addresses the automation
  reliability defects reported, not the underlying feature test itself.
