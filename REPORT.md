## Issue Summary

**Issue #13 context:** Validates that ServiceNow prevents resolving an incident without Resolution Notes.

**User request:** `SNOW_Login.xaml` was failing because the username selector broke after a ServiceNow update.

**Root cause:** Three compounding problems in the `NTypeInto 'User name'` target inside `SNOW_Login.xaml` (and its matching Object Repository entry):

1. **`type='text'` attribute in `FuzzySelectorArgument`** — In recent ServiceNow releases the username `<input>` element's `type` attribute was changed from `text` to `email`. Because the fuzzy selector included `type='text'` as an explicit constraint, it no longer matched the element and the activity failed immediately.

2. **No full-selector fallback** — `SearchSteps="FuzzySelector"` gave the engine only one strategy. A second strategy (`Selector`) using a stable `id`-based selector would have survived the `type` attribute change.

3. **Incorrect `BrowserURL` metadata** — The captured `BrowserURL` was `dev181290.service-now.com/navpage.do` (the post-login navigation page), not the login page URL. This is wrong metadata for a login-page element and misleads UiPath's healing agent when it tries to score selector candidates.

## Changes Made

| File | What changed |
|---|---|
| `Workflows/SNOW_Login.xaml` | Removed `type='text'` from `FuzzySelectorArgument` for the username target; added `FullSelectorArgument` using `id='user_name'`; changed `SearchSteps` from `FuzzySelector` to `Selector FuzzySelector`; corrected `BrowserURL` from `navpage.do` to `login.do` |
| `.objects/_70W/XuB5/4ynJ/9exx/.data/ObjectRepositoryTargetData/.content` | Same selector fixes applied to the canonical Object Repository entry so Studio does not revert the XAML inline data on next OR sync |

## UiPath Analysis

- The broken activity is `NTypeInto` (display name: "Type Into 'User name'") inside `SNOW_Login.xaml`, within the `NApplicationCard` "Login" body.
- The OR reference for the username element is `_BU5etMzk0KtDV4Hwc6ztw/9exx5vVOBkijpPs6hPFx7g`.
- The password selector (`type='password'`) and the "Log in" button selector (full selector `<webctrl tag='BUTTON' type='submit' aaname='Log in' />`) were not affected and required no changes.
- The `ScopeSelectorArgument` (`<html app='msedge.exe' title='Log in | ServiceNow' />`) remains in place; if the page title changes in a future ServiceNow release, this will need updating as well.

**Fixed selector (username):**
```xml
<!-- BEFORE -->
FuzzySelectorArgument="<webctrl tag='INPUT' type='text' aaname='User name' ... name='user_name' />"
SearchSteps="FuzzySelector"
BrowserURL="dev181290.service-now.com/navpage.do"

<!-- AFTER -->
FullSelectorArgument="<webctrl id='user_name' tag='INPUT' />"
FuzzySelectorArgument="<webctrl tag='INPUT' aaname='User name' ... name='user_name' />"
SearchSteps="Selector FuzzySelector"
BrowserURL="dev181290.service-now.com/login.do"
```

The engine now tries the `id='user_name'` full selector first (highly stable — ServiceNow has not changed this ID across releases) and falls back to the fuzzy name/aaname-based selector only if the ID lookup fails.

## Testing Notes

1. **Open the project in UiPath Studio** and navigate to `Workflows/SNOW_Login.xaml`.
2. Confirm the "Type Into 'User name'" target shows the updated `FullSelectorArgument` with `id='user_name'`.
3. **Run `SNOW_Login.xaml`** directly (Run > Run File) against the live `dev181290.service-now.com` PDI instance. Verify that:
   - Edge opens and navigates to the ServiceNow login page.
   - The username field is located and filled without a selector failure.
   - The password field is filled and "Log in" is clicked.
   - The workflow returns a valid `out_Ui_Element` handle (non-null).
4. **Run TC 01** (`TestCases/01_TC_CreateIncident.xaml`) as an end-to-end smoke test — it depends on `SNOW_Login.xaml` and exercises the full login → action → logout chain.
5. If the login page title has changed in the ServiceNow update (e.g., "Login | ServiceNow" instead of "Log in | ServiceNow"), also update the `ScopeSelectorArgument` on both the XAML and OR entries.

## Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| `id='user_name'` removed in a future ServiceNow update | Low — this ID is stable across all known ServiceNow releases | High | The fuzzy fallback (`name='user_name'`, `aaname='User name'`) still fires via `SearchSteps` ordering |
| Page title change breaking scope selector | Medium — ServiceNow occasionally renames its login page title | Medium | Update `ScopeSelectorArgument` title; or widen to `title='*ServiceNow'` for loose matching |
| OR resync reverting XAML changes | Low — both files updated in lockstep | Medium | Both the XAML inline target and the OR `.content` entry were updated; Studio resync will preserve the fix |
| No other workflows affected | Confirmed — password and button selectors were not modified; only the username `NTypeInto` target changed |  |  |
