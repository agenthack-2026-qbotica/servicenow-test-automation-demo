# Report — Issue #66 (Test28)

## Issue Summary

The issue title and acceptance criteria describe a negative test for ServiceNow
incident resolution: an incident should **not** be resolvable while the
"Resolution Notes" field is empty, and the system should show a validation
message and keep the incident in its prior state. However, the actual user
request pivoted to a different, concrete problem report from a **test run
failure**, not a missing feature:

- `TestCases/01_TC_CreateIncident.xaml` failed because the robot could not
  find a required UI element (a selector/synchronization/application-state
  issue).
- `TestCases/04_TC_HighPriorityWAssignGroup.xaml` (the closest match in this
  repo to the "ResolveHighPriorityWAssignGroup" test case named in the
  request — file naming has since been renamed, see note below) failed
  because the robot attempted to interact with a control (the **Resolve**
  button) while it was still disabled.

**Root cause:** both test cases invoke thin `TestCases/*.xaml` wrappers that
delegate the actual UI interaction to `Workflows/01_CreateIncident.xaml` and
`Workflows/04_HighPriorityWAssignGroup.xaml`. In both workflows, UI-state
transitions (page navigation after clicking "CreateNew", and the ServiceNow
client-side field validation that enables the "Resolve" button after
Resolution Code/Resolution Notes are populated) were not given enough time to
settle before the next activity ran:

1. In `01_CreateIncident.xaml`, clicking "CreateNew" triggers a full page
   navigation to the New Incident form (ServiceNow reloads the `gsft_main`
   iframe). The very next activity, "Get Text 'Text_Number'", only had a 2s
   `DelayBefore` and no `DelayAfter` was set on the preceding click, so on a
   slow-loading form the field lookup could race the page load and fail with
   "element not found."
2. In `04_HighPriorityWAssignGroup.xaml`, the "Resolve" button only becomes
   enabled once ServiceNow's client-side validation has run after the
   Resolution Code and Resolution Notes fields are populated (a blur/onchange
   event). The workflow typed the notes and clicked "Resolve" only 1 second
   later with no verification that the button had actually become enabled —
   so on a slow validation cycle the click landed on a still-disabled button.

**Note on file naming:** the current repository does not contain files named
`02_TC_UpdateIncidentPriorityToHigh.xaml` in the "resolve without resolution
notes" flavor described in the issue, nor a
`04_TC_ResolveHighPriorityWAssignGroup.xaml`. The closest, currently-existing
equivalent that performs the "Resolve" action is
`TestCases/04_TC_HighPriorityWAssignGroup.xaml` /
`Workflows/04_HighPriorityWAssignGroup.xaml`, which is what was fixed here.
`CLAUDE.md`'s file inventory is stale relative to the current file set and
should be refreshed separately.

## Changes Made

### `Workflows/01_CreateIncident.xaml`
- Added `DelayAfter="3"` to both "Click 'CreateNew'" activities (the
  `IfExists`/`IfNotExists` branches of the "Check App State 'Button_CreateNew'"
  check), giving the New Incident page time to begin rendering before the
  next activity starts.
- Increased "Get Text 'Text_Number'" `DelayBefore` from `2` to `3` seconds and
  its `Timeout` from `60` to `90` seconds, giving the Number field more margin
  to become interactive after the New Incident form finishes loading.

### `Workflows/04_HighPriorityWAssignGroup.xaml`
- Added `DelayAfter="2"` to "Type Into 'Resolution notes'" so ServiceNow's
  client-side field validation (triggered on blur/change) has time to run
  before the workflow checks the Resolve button's state.
- Wrapped the "Click 'Resolve'" activity in a new
  `uix:NCheckState DisplayName="Check App State 'Button_Resolve'"` gate
  (`Timeout="30"`), reusing the same Object Repository selector already
  recorded for the Resolve button (`Reference="_BU5etMzk0KtDV4Hwc6ztw/ZOumMXmFJE66n1OD_SkPew"`).
  This mirrors the existing "Check App State 'Button_CreateNew'" pattern
  already used in `01_CreateIncident.xaml`:
  - `IfExists` (button reached the `Interactive`/enabled state within 30s):
    performs the original "Click 'Resolve'" click.
  - `IfNotExists` (button never became enabled): logs a `Warn`-level message
    instead of blindly clicking a disabled control, so the test fails with a
    clear diagnostic instead of an opaque "element disabled" automation error.

No selectors were re-recorded/re-indicated, since this environment has no
live ServiceNow/Edge session to interact with — see Testing Notes below.

## UiPath Analysis

- Both workflows already follow the project's Given/When/Then and
  `uix:N*` (UI Automation Next) activity conventions; no structural or
  naming-convention violations were found.
- Existing selectors already use `WaitForReadyArgument="Interactive"` on
  their targets, which does wait for elements to become visible+enabled
  before acting — but this built-in wait has an implicit, comparatively short
  timeout that isn't a substitute for an explicit application-state check
  when the underlying condition is a multi-second client-side JS validation
  (the Resolve button) or a full page reload (the New Incident form).
- The `01_CreateIncident.xaml` workflow already contains a good example of
  the recommended pattern — `uix:NCheckState` ("Check App State
  'Button_CreateNew'") gating on menu-item availability before clicking. The
  fix for `04_HighPriorityWAssignGroup.xaml` follows this same established,
  in-repo pattern rather than introducing a new synchronization idiom.
- `uip rpa get-errors` could not be run in this sandbox
  (`HelmFeatureNotAvailableException: Helm requires a signed-in user`), so
  validation here was limited to confirming both XAML files are well-formed
  XML with no duplicate `WorkflowViewState.IdRef` values (checked via
  `xml.etree.ElementTree` and a `grep`-based uniqueness scan).

## Testing Notes

After this fix, verify in UiPath Studio (with a live connection to
`dev181290.service-now.com` and Orchestrator `SNOW_Credentials` asset):

1. Run `TestCases/01_TC_CreateIncident.xaml` via *Run Test Case* — confirm
   the "Get Text 'Text_Number'" step no longer intermittently reports
   "element not found" immediately after the New Incident form loads.
2. Run `TestCases/04_TC_HighPriorityWAssignGroup.xaml` against a high
   priority incident with an assignment group set — confirm the new "Check
   App State 'Button_Resolve'" step passes (`IfExists` branch) and the
   incident is successfully resolved.
3. As a regression check for the `IfNotExists` branch, temporarily point the
   test at an incident where a required resolution field is intentionally
   left blank and confirm the workflow logs the new `Warn` message instead of
   throwing a raw "element disabled" automation exception.
4. If either activity still occasionally fails after these timing changes,
   the next step is to re-indicate the affected selectors from a live
   ServiceNow session (Studio's selector repair / Object Repository update),
   since a stale selector — rather than timing — would be the remaining
   candidate root cause.
5. Separately, since the underlying issue title/acceptance criteria describe
   validating that ServiceNow itself rejects resolving an incident with empty
   Resolution Notes (a distinct feature from the timing fix above), confirm
   whether a dedicated negative test case for that scenario is still wanted;
   none currently exists in this repository under any name.

## Risk Assessment

- **Low risk.** All changes are additive timing/synchronization adjustments
  (`DelayAfter`/`DelayBefore`/`Timeout` tuning) plus one new conditional gate
  (`uix:NCheckState`) that reuses an already-recorded selector. No existing
  selectors, click targets, or business logic branches were removed or
  altered.
- The added `Timeout="30"` on the new Resolve-button check adds up to 30
  seconds to worst-case runtime for `04_TC_HighPriorityWAssignGroup.xaml`
  only in the failure path (button never enables); the success path is
  unaffected once the button becomes interactive.
- The `IfNotExists` branch changes failure behavior for the disabled-Resolve
  case from an unhandled automation exception to a logged warning followed by
  workflow completion without clicking Resolve. This means the incident will
  remain unresolved and downstream `Then` assertions (if any check for a
  "Resolved" state) will now fail with a clear, attributable log message
  rather than the run aborting on an automation error — this is intentional
  and improves diagnosability, but should be reviewed if any caller depends
  on an exception being thrown in this scenario.
- No changes were made to `TestCases/*.xaml` files, `applicationsMetadata.json`,
  or the Object Repository (`.objects/`) — all fixes are contained to the two
  `Workflows/*.xaml` files named in the request.
