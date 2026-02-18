# ALLOPS – CapEx Project Lifecycle Automation (Power Apps + Power Automate)

This solution combines **Power Automate**, **Power Apps**, **SharePoint** and **Power BI** to:

- Create the project’s base document structure.
- Centralize project metadata in a master list.
- Automatically enforce business rules (status, complexity, closure logic).
- Monitor budget deviations (110% committed threshold).
- Provide the Project Manager (PM) with key links (documents, Gantt, cost tracking, dashboard, shared folder).
- Facilitate communication through a Microsoft Teams channel.
- Offer a user-friendly interface to create and browse projects.

Originally developed for a multinational company for use in the LATAM region.

> **Note:** In this repository, **ALLOPS** refers to the corporate SharePoint site where CapEx projects are centralized.  
> The flows can be adapted to any similar environment by changing the SharePoint lists and paths.

---

## Goal

Reduce manual work when creating a new CapEx project:

- Standardize folder structure, initial files and related lists.
- Centralize project information in a SharePoint master list.
- Provide the end user with all key links in a single automated email.
- Make cost tracking and planning easier and enable quick visual overview (Gantt / MS Project / dashboard).
- Centralize project communication through a dedicated Teams channel.

---

## Solution components

### 1. Power Automate flows (`flows/`)

The solution includes multiple flows covering provisioning, governance and financial monitoring:

#### 🔹 CPX Initialization (Parent + Child)

- **Parent – `CPXinitializationARParent-*.json`**  
  - Triggered when a new item is created in the projects list.  
  - Validates basic data, assigns a **unique ID** to the project and calls the CHILD flow.

- **Child – `CPXProvisioningChild-*.json`**  
  - Creates the project document structure in SharePoint (folders and files from templates).  
  - Updates the master projects list with the generated links.  
  - (Optionally) creates a Teams channel and a welcome post.  
  - Sends an automatic email to the project owner with all key links.

---
#### 🔹 CPX Closure Automation – `CPX-closure.json`

Ensures reporting consistency and correct eFTE calculation.

- When **CPX Status** changes to **Done** or **Closed**:
  - Automatically sets **CPX Complexity = "All (CPX Closure)"**.
- If a project moves back from **Done/Closed → In progress**:
  - Sets **CPX Complexity = "Low"**.
- For any other transition:
  - The flow does not override complexity.

This guarantees alignment between operational status and reporting logic.

---

#### 🔹 CPX Warning – 110% Budget Rule – `CPXWarning110.json`

Monitors committed amounts (COMM) against the approved budget.

- Dynamically calculates total committed values:
  - Committed till PFY
  - COMMPFY P1–P12
  - COMMCFY P1–P12
- Compares against approved budget.
- If committed exceeds **110%**, the flow:
  - Sends a formatted HTML alert to the PM.
  - Suggests review of open POs/contracts.
  - Recommends fund extension if applicable.

This flow implements an automated financial governance control.

---

Technologies used in the flows:

- **Power Automate (cloud)**
- **SharePoint Online**
- **Outlook 365** (email sending)
- **SharePoint REST API** (`CreateCopyJobs`) to copy folder/file templates.
- **Microsoft Graph REST API** to create channels and welcome posts.

More technical detail in [`flows/flows_info.md`](flows/flows_info.md).

---

### 2. Power Apps application (`powerapps/`)

Front end for PMs and the engineering team:

- Screens focused on:
  - **Project creation** with a small set of required fields (simplified view).
  - **Project browsing** and navigation to key links.
  - Embedded **Power BI dashboards** filtered by project ID.
  - Shortcuts to the Gantt / task list, cost tracking file and other resources.

Files:

- `powerapps/CapexProjectInitApp.msapp` – exported Power Apps app.  
- `powerapps/app_info.md` – functional description and import guide (to be completed).

---

### 3. SharePoint Online

The solution assumes:

- A **master projects list** (e.g. `AR Projects Master list`):
  - Stores project metadata (country, company, internal ID, name, complexity, PM, etc.).
  - Stores links to:
    - Main document folder
    - Gantt / task list
    - Cost tracking file
    - Power BI dashboard
    - Shared / “blue” folder

- **Libraries and templates**:
  - Location of the folder/file templates that the flow copies when creating a project.

---

### 4. Power BI

- Dashboards showing:
  - Cost progress (committed / spent).
  - Global CapEx status.
  - Consolidated information per project.

- Some tiles are **embedded in Power Apps** and filtered by project ID (CPX ID) using URL parameters on the tile.

---
## Governance & Control Layer

Beyond project provisioning, this solution introduces automated governance controls:

- Status-to-complexity enforcement.
- Budget threshold monitoring (110% rule).
- Automatic stakeholder notification.
- Centralized reporting consistency across AR, PE and Master lists.

This transforms the tool from a provisioning utility into a structured CapEx control framework.


## High-level architecture

The high-level process is:

1. **New project creation**
   - The PM fills in a form (SharePoint list or Power Apps app).
   - An item with the minimum required project data is created.

2. **PARENT flow execution (Power Automate)**
   - Assigns a unique ID to the item.
   - Calls the CHILD flow, passing project data.

3. **Provisioning (CHILD flow)**
   - Creates the document structure from templates.
   - Generates lists / Gantt / cost tracking as needed.
   - Updates the master list with all links.

4. **Notification to the PM**
   - Sends an automatic email with:
     - Project summary.
     - Main link to the project in ALLOPS.
     - Links to user guides and dashboards.

5. **Front-end for browsing**
   - The PM and team use the Power Apps app to:
     - View project details.
     - Access all related resources.
     - Open dashboards already filtered by project.

See the diagram and technical details in [`docs/architecture.md`](docs/architecture.md).

---

## Repository structure

```text
.
├── README.md                      # High-level overview of the whole solution
├── flows/
│   ├── CPXProvisioningChild-292B8380-6DC1-F011-BBD2-6045BD9F321D.json       # Export of CHILD Power Automate flow
│   ├── CPXinitializationARParent-CFFF159C-D4CA-F011-8544-7CED8D592B6B.json  # Export of PARENT Power Automate flow
│   ├── CPX-closure.json    # Export of the additional flow
│   ├── CPXWarning110.json  # Export of the additional flow
│   └── flows_info.md                                                       # Detailed description of the flows
├── powerapps/
│   ├── CapexProjectInitApp.msapp          # Export of the Power Apps app
│   └── app_info.md                        # App details and import/config guide
├── assets/
│   └── screenshots/                       # PNG/JPG screenshots used in the docs
└── docs/
    └── architecture.md                    # Technical architecture documentation
    └── solution_export/
