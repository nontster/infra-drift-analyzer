# Product Requirement Document (PRD): Infrastructure Drift Analyzer
* **Product Name:** Infrastructure Drift Analyzer
* **Document Version:** 3.1 (Updated Scope: Standalone App / Backstage Plugin Integration)
* **Target Platform:** Standalone Web Application / Spotify Backstage Plugin
* **Data Source (System of Record):** ServiceNow CMDB
* **Product Manager:** Nont Banditwong

---

## 1. Executive Summary & Objective
* **The Problem:** Currently, verifying Configuration Items (CIs) across different environments (e.g., UAT vs. PROD) prior to deployment is a manual and error-prone process. Configuration drift is a leading cause of production incidents, often resulting from missing keys or lingering lower-environment variables (e.g., UAT URLs in PROD).
* **The Solution:** Develop an On-Demand Impact Analysis tool as a **Standalone Web Application or Spotify Backstage Plugin** that performs a peer-to-peer, side-by-side comparison of CI configurations (Key-Value pairs) at the **Host (Server) level**. By integrating with ServiceNow REST APIs and filtering hosts via Application ID/Acronym and Environment (`used_for`), the tool empowers Application Teams to quickly identify server-level discrepancies and anti-patterns directly from their developer portal before executing a deployment.

### 1.1 Out of Scope (Non-Goals)
To ensure performance and focused delivery, V1 of this tool explicitly **excludes**:
* **Auto-remediation:** The tool identifies drift but will NOT automatically sync or push configurations from the Source to Target environments. It is read-only.
* **Deep OS/File Parsing:** The tool will not parse host-level `.conf`, `.yaml`, or `.xml` files. It only analyzes properties that are actively ingested and populated within standard ServiceNow CMDB tables (e.g., `cmdb_ci_server`, `cmdb_ci_linux_server`).

---

## 2. Target Audience & Personas
* **Primary Users:** Software Engineers / DevOps / SREs (Executing deployments and checking server configurations directly from their Backstage portal/Web App).
* **Secondary Users:** Tech Leads / Change Managers / QA (Reviewing and approving Change Requests based on accurate impact analysis).

## 3. Key Metrics (Success Criteria)
* **Incident Reduction:** Decrease P1/P2 incidents caused by configuration mismatches by [X]%.
* **Time Saved:** Reduce the time spent on manual configuration validation and Change Request (CR) preparation by [X] hours per deployment cycle.
* **Adoption Rate:** Achieve [X]% of Change Requests featuring a directly attached diff report as validation evidence.
* **Portal Engagement:** Achieve [X] monthly active users (MAU) utilizing the plugin within Spotify Backstage.

---

## 4. User Stories
* **US1:** As a *Software Engineer*, I want to *compare Host/Server configurations side-by-side between lower environments (e.g., NFT/UAT) and PROD via our Backstage portal* so that *I can ensure no infrastructure configurations (like CPU, RAM, OS versions, or IP structures) are missed prior to deployment.*
* **US2:** As a *Tech Lead*, I want *the system to contextually alert me (Anti-pattern Alert) if lower-environment values are found in the PROD server attributes* so that *I can prevent critical deployment errors before approving a CR.*
* **US3:** As a *DevOps Engineer*, I want to *explicitly select a specific PROD node from a cluster to compare against a UAT node, while the system automatically checks for intra-cluster consistency across other PROD nodes,* so that *I can guarantee cluster stability.*
* **US4:** As a *Change Manager*, I want to *seamlessly attach the server comparison results directly to a Change Request from the web app* so that *I have an audit trail and risk assessment artifact without managing local file downloads.*

---

## 5. Functional Requirements

### 5.1 System Scope & Integration
* **Trigger:** Real-time (On-demand) data fetching triggered by the user via the frontend UI.
* **Integration System:** Connects to ServiceNow via standard REST APIs (e.g., Table API, CMDB Instance API).
* **Starting Context (Host-Centric):** Searching begins strictly at the Host CI level.
* **Query Parameters:** Utilizes User Claims from the Identity Provider (IdP) to map `u_app_id` and `u_app_acronym`, filtering with operational state (`operational_status=1`) to securely boundary the data scope.
* **Topology Depth (Host to Child Components):**
  * **Level 1 (Base):** VM Instance / Host (e.g., `cmdb_ci_linux_server`, `cmdb_ci_win_server`).
  * **Level 2 (Software Instances on Host):** Middleware (`cmdb_ci_app_server_tomcat`) or Database (`cmdb_ci_db_ora_instance`) running specifically on that matched host.

### 5.2 Comparison Engine & Logic
* **Key-Value Matching:** Compares CI property fields returned by the API payload.
* **Missing Key Identification:** Highlights keys present in Source but missing in Target (and vice versa).
* **Configuration Meta Data (2-Tier Architecture):**
  * **Tier 1 (Global Rules):** System-wide exclusions (e.g., ignoring `sys_updated_on`, `sys_id`, `last_discovered`) managed in the application's configuration.
  * **Tier 2 (App-Specific Rules):** Custom JSON/Key-value settings tagged to the specific `u_app_id` to ignore dynamic fields expected to change between hosts.
* **Contextual Anti-Pattern & Risk Validation (Critical):**
  * **Environment-Aware Keyword Watchlist:** The engine must check the Target Host's environment (via `used_for` attribute). If Target = PRD, triggers a severe warning if prohibited keywords (e.g., `dev`, `sit`, `nft`) are detected in critical fields like `fqdn`, `name`, or `short_description`.
  * **Network Range Validation:** Validates if IP addresses (`ip_address`, `default_gateway`) fall within the accepted subnet/range for that specific environment.

### 5.3 Cluster & High Availability (HA) Handling
* **Explicit Node Selection:** The UI mandates selecting a specific Target Server (e.g., `snftwwsaapp01`) from a list of available hosts in that environment. 
* **Intra-Cluster Consistency Check:** The backend service runs a background validation query across all hosts sharing the same `u_app_id` and `used_for` (Environment) to ensure nodes within the same cluster share identical baseline configurations. Drift within the cluster itself triggers a warning banner in the UI.

---

## 6. UI / UX Architecture
Built as a modern Single Page Application (SPA) (e.g., React, Vue) or an embedded component within Spotify Backstage, divided into three main sections:

### 6.1 Context Selector (Top Bar)
* **App Context (Auto-filled):** Displays logged-in user's `u_app_id` (e.g., 808) and `u_app_acronym` (e.g., WSA), fetched via SSO/IdP claims.
* **Source Environment Selector:** Dropdown populated by unique `used_for` values (e.g., DEV, NFT, SIT, UAT).
* **Source Host Selector:** Dropdown listing specific server names (`fqdn` or `name`) matching the selected App Context and Source Environment.
* **Target Environment Selector:** Dropdown for the target (e.g., PRD).
* **Target Host Selector:** Dropdown listing specific server names for the Target Environment.
* **Action Buttons:** "Compare", "Export to CSV/JSON", and **"Attach to Change Request"** (Includes a text input field for a `CHGXXXXXXX` record number).

### 6.2 Host Components Tree (Left Panel)
* **Hierarchy Visibility:** Displays the Host as the root node, with any discovered child software instances branching below it.
* **Visual Coding System:**
  * 🟢 **Green (Match):** Properties match perfectly.
  * 🔴 **Red (Mismatch):** Value differences or missing keys detected.
  * 🟡 **Yellow (Warning):** Values match, but anti-patterns/risks detected (e.g., PRD server contains 'nft').
  * ⚪ **Gray (Ignore):** Property skipped based on Meta Data rules.

### 6.3 Diff Viewer (Right Panel)
* **Git-Style Side-by-Side Table:** Source Host values (left), Target Host values (right).
* **Color Highlighting:** Green (Added), Red (Removed/Different), Yellow (Modified).
* **Controls:** `Show only differences` toggle switch, and a real-time text search bar to filter specific keys (e.g., searching for `cpu` or `ram`).

---

## 7. Backend Logic & Technical Architecture

### 7.1 Status Propagation & Aggregation Layer
The external backend (or Backstage backend plugin) acts as an aggregation layer.
* **Logic Flow:**
  1. Receive request from frontend and query ServiceNow REST API (`cmdb_ci_server` tables) for Source and Target hosts.
  2. Parse the JSON response and filter out ignored keys.
  3. Calculate Diff status (MATCH, WARNING, MISMATCH) using the comparison engine logic.
  4. Return the processed, UI-ready JSON payload to the frontend.

### 7.2 Security, Authentication & Access Control
* **Identity Provider (IdP) Integration:** The application relies on the organization's SSO (e.g., OAuth 2.0 / OIDC) to authenticate users and establish their Application contexts (`u_app_id`).
* **ServiceNow Authorization (API Context):** * Option A (User-Context): Pass user tokens to ServiceNow via OAuth to enforce native ServiceNow Access Control Lists (ACLs) directly.
  * Option B (Service Account): Use a restricted Integration Service Account for read-only API calls, applying filtering logic at the application backend based on the user's IdP claims to ensure they only view permitted CIs.
* **Performance:** Utilizing focused REST queries (`sysparm_query`) and caching relatively static topology metadata (where applicable) to prevent overwhelming the ServiceNow instance.

---

## 8. Supporting Features & Workflow Automation
* **REST-Driven CR Attachment:** Users can enter a Change Request number (`CHGXXXXXXX`) in the tool. The application backend will generate a diff artifact (JSON/PDF/CSV) and utilize the ServiceNow Attachment API (`/api/now/attachment`) to upload it directly to the target CR record.
* **Local Export Functionality:** Users can export the diff results locally in CSV or JSON format for external audits or offline review.
