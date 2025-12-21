# CapEx Project Init App (Power Apps)

Canvas app used as the front-end for creating and consulting CapEx projects in **ALLOPS**.  
It sits on top of the same SharePoint lists used by the Power Automate flows and exposes links, dashboards and documents in a user-friendly way.

---

## 1. Purpose

- Guide Project Managers through the **creation** of a new CapEx project with only a few required fields.
- Avoid overwhelming the user by splitting information across **tabs/sections**.
- Centralize access to:
  - Project metadata (who / where / when / status).
  - Project documents (root folder, “blue” folder, cost tracking, etc.).
  - Gantt / task lists.
  - Power BI dashboards filtered by **CPX ID**.
  - Teams channel and other collaboration assets.

The app is optional: flows can run using a plain SharePoint form, but the app provides a better UX and a single place to work.

---

## 2. Architecture overview

- **Type**: Canvas app (tablet layout).
- **File**: `CapexProjectInitApp.msapp`
- **Primary data source**: SharePoint **Projects Master List**  
  (e.g. `AR Projects Master list`, same list used by the flows).
- **Other resources**:
  - SharePoint document libraries where project folders and templates live.
  - Power BI dashboards/tiles embedded via URL.
  - Links (URLs) stored in list columns and opened with `Launch()`.

High-level flow from the app perspective:

1. User opens the app and either:
   - Creates a new project, or
   - Selects an existing project.
2. The app writes/reads to the **Projects Master List**.
3. Power Automate flows provision the project (folders, lists, Teams, etc.).
4. The app shows the updated links, Gantt/tasks and dashboards for that project.

---

## 3. Screens and navigation

The app is organized into screens/sections, navigated through a **horizontal tab bar** (horizontal gallery):

- **New / Edit Project**
  - SharePoint form connected to the Projects Master List.
  - First “tab” only shows a **minimal set of required fields** (for example):
    - Project name
    - Organizational area
    - Fiscal year
    - Plant
  - Additional fields are distributed in other sections (details, financials, etc.).
  - Visibility of DataCards is controlled so that the “New” view stays simple.

- **Project Overview**
  - Read-only view of key metadata for the currently selected project.
  - Uses the same list item as the form (no duplicated data source).

- **Links & Documents**
  - Icons / buttons that call `Launch()` with URLs stored in the master list:
    - Project root folder
    - Shared / “blue” folder
    - Cost tracking workbook
    - Gantt / tasks list
    - Teams channel
  - Some SharePoint columns may hold *dummy* values (e.g. `"."`) for JSON formatting;  
    in the app, the user interacts with **custom icons/buttons**, not with the dummy text.

- **Analytics (Power BI)**
  - Embeds a **Power BI tile** that shows project KPIs (committed/spent, status, etc.).
  - The tile is filtered by the current project’s **CPX ID** using URL parameters.

> Exact screen/tab names can be adjusted to your environment; the .msapp is a reference implementation.

---

## 4. Data sources

Main expected connections:

- **SharePoint**
  - Site: ALLOPS (or equivalent in your tenant).
  - List: Projects master (e.g. `AR Projects Master list`).
  - The app assumes this list exposes:
    - Basic metadata (name, country, plant, FY, owner, etc.).
    - URL columns for:
      - ProjectFolderUrl
      - CostTrackingUrl
      - DashboardUrl
      - TeamsChannelUrl
      - TaskListUrl / GanttUrl
      - BlueFolderUrl (if used)

- **Power BI**
  - One or more dashboards already published to the Power BI service.
  - At least one **tile** with a field that can be filtered by CPX ID (e.g. `CPX ID#`).

If column names differ, the formulas inside the app must be adjusted accordingly.

---

## 5. Power BI integration and filtering

The app embeds a Power BI **dashboard tile** using an URL of the form:

```text
https://app.powerbi.com/embed
  ?dashboardId=<DashboardId>
  &tileId=<TileId>
  &filter=<TableName>/<ColumnName> eq '<CPX_ID>'

