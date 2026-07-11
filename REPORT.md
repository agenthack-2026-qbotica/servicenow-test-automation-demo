# Issue #61 (Test26) — Fix Report

## Issue Summary

Issue #61 ("Test26") requests validating that a ServiceNow incident **cannot** be resolved
while the Resolution Notes field is empty. However, the actionable content of the issue
(the "USER REQUEST" section) is a root-cause report from a **prior automated test run**,
not a request to build that new validation scenario. It states that the run failed because
of UI automation reliability problems, not a confirmed application defect:

- **`TestCases/01_TC_CreateIncident.xaml`** — the robot could not find a required UI element
  (selector/synchronization/window-state/page-rendering issue).
- **`TestCases\04_TC_ResolveHighPriorityWAssignGroup.xaml`** (closest match in this repo:
  `TestCases/04_TC_HighPriorityWAssignGroup.xaml`, which invokes
  `Workflows/04_HighPriorityWAssignGroup.xaml`) — the robot attempted to interact with a
  **disabled** target element, meaning it clicked before the control became enabled.

**Root cause identified after inspecting both XAML files:**

1. **`Workflows/01_CreateIncident.xaml`** — the very first UI interaction after the browser
   session is handed off from `SNOW_Login.xaml` is `NClick "Click 'Favorites'"` with **no
   delay and no readiness check**. Unlike the later `NCheckState "Check App State
   'Button_CreateNew'"` (which properly waits up to `Timeout="30"`), this first click has
   nothing guarding it. If the ServiceNow Polaris UI is still rendering right after
   login/navigation, this click can miss the element entirely — matching the "could not
   find the required UI element" symptom.

2. **`Workflows/04_HighPriorityWAssignGroup.xaml`** — two synchronization gaps:
   - `NGetText "Get Text 'Priority'"` runs with only `DelayBefore="1"` right after the
     incident is selected from search results (`DelayAfter="5"` on that click), with no
     `Timeout` for retry — the incident form can still be finishing its load.
   - The `NKeyboardShortcuts "Keyboard Shortcuts 'Button_ResolutionCode'"` activity selects
     a Resolution Code via repeated Down-arrow key events and had **no delay after it**
     before the workflow types Resolution Notes and clicks **Resolve**. ServiceNow's
     Polaris UI re-validates mandatory fields and enables/disables the **Resolve** button
     via a client-side `onchange` handler that fires after the dropdown value changes —
     without a pause for that handler to run, the subsequent `NClick "Click 'Resolve'"`
     (previously `DelayBefore="1"`) can fire while the button is still disabled. This
     matches the "attempted to interact with a disabled target element" symptom exactly.

## Changes Made

- **`Workflows/01_CreateIncident.xaml`**
  - `NClick "Click 'Favorites'"` (`IdRef="NClick_4"`): added `DelayBefore="5"` so the robot
    waits for the ServiceNow page to finish rendering after the login handoff before
    interacting with it.

- **`Workflows/04_HighPriorityWAssignGroup.xaml`**
  - `NGetText "Get Text 'Priority'"` (`IdRef="NGetText_2"`): increased `DelayBefore` from
    `1` to `3` and added `Timeout="30"` so the incident form has more time to finish
    loading, with a retry window if it hasn't.
  - `NKeyboardShortcuts "Keyboard Shortcuts 'Button_ResolutionCode'"`
    (`IdRef="NKeyboardShortcuts_1"`): added `DelayAfter="2"` so ServiceNow's client-side
    validation/enable logic has time to run after the Resolution Code dropdown value
    changes.
  - `NClick "Click 'Resolve'"` (`IdRef="NClick_4"`): increased `DelayBefore` from `1` to
    `3`, giving the Resolve button time to become enabled before the click is attempted.

No selectors, targets, Object Repository references, or business logic were changed —
only timing/synchronization attributes on existing activities. The empty-Resolution-Notes
validation scenario described in the issue's acceptance criteria was **not** implemented;
it is out of scope for this fix (see Testing Notes).

## UiPath Analysis

Findings from analysis via the `uipath-rpa` skill/conventions:

- Both files follow the project's `NApplicationCard` + `uix:N*` UI Automation pattern
  correctly; the defects were purely missing/insufficient synchronization, not structural
  or selector problems.
- `WaitForReadyArgument="Interactive"` is already set on nearly every target in both files,
  which normally causes UiPath to wait for an element to become interactive. This is not
  sufficient by itself for ServiceNow's Polaris UI: many controls live inside Shadow DOM /
  custom web components (`shadowhostid=''` selectors) where an element can report as
  "Interactive" (attached, visible) before the page's own JS has finished enabling it for
  business rules (e.g., mandatory-field validation gating the Resolve button). Explicit
  `DelayBefore`/`DelayAfter`/`Timeout` attributes are the safe, minimal way to add slack for
  this without needing to re-capture selectors or Object Repository targets.
- `Workflows/01_CreateIncident.xaml` already contains a good synchronization example
  (`NCheckState "Check App State 'Button_CreateNew'"`, `Timeout="30"`) — this pattern should
  be the template for future workflows that need to branch based on whether an element has
  appeared yet, rather than relying on fixed delays alone.
- Confirmed no `.xaml` file in this repo currently implements the "empty Resolution Notes
  blocks Resolve" scenario described in the issue's acceptance criteria — there is no test
  case or workflow referencing resolution-notes validation messaging. Implementing that
  scenario is a separate, larger task (new test case + workflow + Object Repository
  targets for the validation message) and was intentionally left out of this fix per the
  user request, which scoped the work to the reliability root cause only.
- `uip rpa validate` could not be run to full completion in this environment because it
  requires an authenticated UiPath session (`uip login` / signed-in Assistant/Robot), which
  isn't available here. Both edited files were verified to be well-formed XML.

## Testing Notes

After these changes, verify (ideally from UiPath Studio or Test Manager, against the live
`dev181290.service-now.com` PDI):

1. Run `uip rpa validate --file-path "Workflows/01_CreateIncident.xaml" --project-dir "."`
   and the same for `Workflows/04_HighPriorityWAssignGroup.xaml` in an authenticated
   session — confirm 0 errors, then `uip rpa build "."` for the whole project.
2. Run `TestCases/01_TC_CreateIncident.xaml` end-to-end and confirm the Favorites click no
   longer intermittently fails to find its element right after login.
3. Run `TestCases/04_TC_HighPriorityWAssignGroup.xaml` against an incident with High
   Priority and a non-empty Assignment Group; confirm the Resolve button is clicked
   successfully (no "element disabled" failure) and the incident actually transitions to
   Resolved.
4. Since these are timing-based fixes (not deterministic waits on a specific condition),
   re-run each test case a few times to confirm the failures were actually eliminated and
   not just made less frequent — if flakiness persists, consider replacing the fixed
   `DelayBefore`/`DelayAfter` on the Resolve click with an explicit `NCheckState`/element-
   enabled check (following the `Button_CreateNew` pattern in `01_CreateIncident.xaml`).
5. The empty-Resolution-Notes acceptance criteria from the issue description are still
   unimplemented — if that feature is still wanted, it should be tracked as separate,
   explicit follow-up work (new test case + workflow + captured selectors for the
   validation message and the disabled/blocked Resolve state).

## Risk Assessment

- **Low risk / low blast radius.** Changes are limited to timing attributes
  (`DelayBefore`, `DelayAfter`, `Timeout`) on existing activities in two workflow files.
  No selectors, Object Repository references, control flow, or assertions were touched.
- **Slower execution time.** Total added delay is roughly 5s (`01_CreateIncident.xaml`) and
  roughly 4s net (`04_HighPriorityWAssignGroup.xaml`) per run. Negligible for a test suite
  running a handful of incident lifecycle test cases.
- **Fixed delays are a mitigation, not a guarantee.** If the ServiceNow dev instance is
  under unusually heavy load or has changed further, these fixed waits could still be
  insufficient. The more robust long-term fix is an explicit `NCheckState`/element-enabled
  check before the Resolve click (as already used for `Button_CreateNew`), but that
  requires live Studio access to capture the correct target/state, which is out of scope
  for this environment.
- **No behavior change to already-passing scenarios.** Since only wait times increased
  (never decreased) and no logic changed, previously-passing runs of these two test cases
  should be unaffected other than running slightly slower.
