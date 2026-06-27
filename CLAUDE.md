# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UiPath test suite that automates ServiceNow incident lifecycle testing via browser automation against a developer PDI instance (`https://dev181290.service-now.com`). All workflows are XAML-only (no coded .cs files). Integrated with UiPath Test Manager on the `hackathon26_795` staging tenant.

## Common Commands

```bash
# Validate project for errors
uip rpa get-errors --project-dir "."

# Run a specific test case via UiPath CLI
uip rpa run --project-dir "." --entry-point "TestCases/01_TC_CreateIncident.xaml"

# Publish project package
uip rpa pack --project-dir "." --output-dir "./output"
```

Tests are primarily run from **UiPath Studio** (Run > Run Test Case) or **UiPath Test Manager** — the CLI commands above require `uip` (UiPath CLI) to be installed.

## Architecture

### Pattern: Given / When / Then

Each test case in `TestCases/` follows a strict three-sequence structure:
- **Given**: Login via `SNOW_Login.xaml` → receives a `UiElement` browser session handle
- **When**: Invoke one or more action workflows from `Workflows/`, passing the `UiElement` handle
- **Then**: Assert results (often: export data → ReadRange → VerifyExpressionWithOperator) → Logout

### Browser Session Sharing

`SNOW_Login.xaml` opens Edge, navigates to ServiceNow, reads the `SNOW_Credentials` asset from Orchestrator (`Shared` folder), and returns a `UiElement` session handle. This handle is passed as `in_Ui_Element` into every subsequent workflow — no re-login per step. `SNOW_Logout.xaml` closes the session.

### Data Verification Pattern

When test cases need to verify incident data, they use `01_RetrieveIncident.xaml` to export the incident to a local file resource, then read it with `ReadRange` (Excel activities) into a DataTable, and assert on `RowCount`.

### Object Repository

UI elements are stored in `.objects/` as an Object Repository app (app ID: `_BU5etMzk0KtDV4Hwc6ztw`) representing the ServiceNow application in Edge. Elements are referenced inline via `Reference` attributes in XAML — there is no generated typed Descriptors class.

## Naming Conventions

| Scope | Convention | Example |
|-------|-----------|---------|
| Variables | `typePrefix_Name` | `str_Number`, `bln_Flag`, `dt_IncidentData`, `file_LocalResource`, `Ui_Element` |
| Arguments | `in_` / `out_` prefix + type prefix | `in_str_IncidentNumber`, `out_bln_HighPriorityWithAssignmentGroup` |
| Test case files | `NN_TC_ActionNoun.xaml` | `01_TC_CreateIncident.xaml` |
| Reusable workflows | `NN_ActionNoun.xaml` or `SNOW_Verb.xaml` | `03_AddAssignmentGroup.xaml`, `SNOW_Login.xaml` |
| x:Class names | Match display name | `TC_CreateIncident`, `SNOW_Login` |

## Incomplete Test Cases

TC 02 (`02_TC_UpdateIncidentPriorityToHigh.xaml`), TC 06 (`06_TC_CloseIncidentWoResolutionNotes.xaml`), and TC 07 (`07_TC_ChangePriorityWoAssignmentGroup.xaml`) are stubs with empty Given/When/Then sequences.

## Key Files

- `project.json` — dependencies, target framework, Studio version
- `Workflows/SNOW_Login.xaml` — session setup; credential retrieval from Orchestrator
- `TestCases/01_TC_CreateIncident.xaml` — most complete test case; reference implementation for the full pattern
- `applicationsMetadata.json` — browser/URL config for the ServiceNow app
- `.tmh/config.json` — Test Manager Hub connection (staging.uipath.com)
