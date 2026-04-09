# Product Requirement Document (PRD): Infrastructure Drift Analyzer
**Product Name:** Infrastructure Drift Analyzer
**Document Version:** 3.0 (Updated Scope: Host-Level Configuration Comparison)
**Target Platform:** ServiceNow
**Product Manager:** Nont Banditwong

---

## 1. Executive Summary & Objective
**The Problem:** Currently, verifying Configuration Items (CIs) across different environments (e.g., UAT vs. PROD) prior to deployment is a manual and error-prone process. Configuration drift is a leading cause of production incidents, often resulting from missing keys or lingering lower-environment variables (e.g., UAT URLs in PROD).
**The Solution:** Develop an On-Demand Impact Analysis tool within ServiceNow that performs a peer-to-peer, side-by-side comparison of CI configurations (Key-Value pairs) at the **Host (Server) level**. By filtering hosts via Application ID/Acronym and Environment (`used_for`), the tool empowers Application Teams to quickly identify server-level discrepancies and anti-patterns before executing a deployment.

### 1.1 Out of Scope (Non-Goals)
To ensure performance and focused delivery, V1 of this tool explicitly **excludes**:
* **Auto-remediation:** The tool identifies drift but will NOT automatically sync or push configurations from the Source to Target environments. It is read-only.
* **Deep OS/File Parsing:** The tool will not parse host-level `.conf`, `.yaml`, or `.xml` files. It only analyzes properties that are actively ingested and populated within standard ServiceNow CMDB tables (e.g., `cmdb_ci_server`, `cmdb_ci_linux_server`).

---

## 2. Target Audience & Personas
* **Primary Users:** Software Engineers / DevOps / SREs (Executing deployments and checking server configurations).
* **Secondary Users:** Tech Leads / Change Managers / QA (Reviewing and approving Change Requests based on accurate impact analysis).

## 3. Key Metrics (Success Criteria)
* **Incident Reduction:** Decrease P1/P2 incidents caused by configuration mismatches by [X]%.
* **Time Saved:** Reduce the time spent on manual configuration validation and Change Request (CR) preparation by [X] hours per deployment cycle.
* **Adoption Rate:** Achieve [X]% of Change Requests featuring a directly attached diff report as validation evidence.

---

## 4. User Stories
* **US1:** As a *Software Engineer*, I want to *compare Host/Server configurations side-by-side between lower environments (e.g., NFT/UAT) and PROD* so that *I can ensure no infrastructure configurations (like CPU, RAM, OS versions, or IP structures) are missed prior to deployment.*
* **US2:** As a *Tech Lead*, I want *the system to contextually alert me (Anti-pattern Alert) if lower-environment values are found in the PROD server attributes* so that *I can prevent critical deployment errors before approving a CR.*
* **US3:** As a *DevOps Engineer*, I want to *explicitly select a specific PROD node from a cluster to compare against a UAT node, while the system automatically checks for intra-cluster consistency across other PROD nodes,* so that *I can guarantee cluster stability.*
* **US4:** As a *Change Manager*, I want to *seamlessly attach the server comparison results directly to a Change Request* so that *I have an audit trail and risk assessment artifact without managing local file downloads.*

---

## 5. Functional Requirements

### 5.1 System Scope & CSDM Integration
* **Trigger:** Real-time (On-demand) data fetching triggered by the user.
* **Data Source:** Directly queries CI standard tables/properties in ServiceNow (`cmdb_ci_server` and extended tables).
* **Starting Context (Host-Centric):** Searching begins strictly at the Host CI level.
* **Query Parameters:** Utilizes User Claim data (`u_app_id`, `u_app_acronym`) and operational state (`operational_status=1`) to securely boundary the data scope.
* **Topology Depth (Host to Child Components):**
  * **Level 1 (Base):** VM Instance / Host (e.g., `cmdb_ci_linux_server`, `cmdb_ci_win_server`).
  * **Level 2 (Software Instances on Host):** Middleware (`cmdb_ci_app_server_tomcat`) or Database (`cmdb_ci_db_ora_instance`) running specifically on that matched host.

### 5.2 Comparison Engine & Logic
* **Key-Value Matching:** Compares CI property fields returned by the ServiceNow API.
* **Missing Key Identification:** Highlights keys present in Source but missing in Target (and vice versa).
* **Configuration Meta Data (2-Tier Architecture):**
  * **Tier 1 (Global Rules):** System-wide exclusions set by Admins (e.g., ignoring `sys_updated_on`, `sys_id`, `last_discovered`).
  * **Tier 2 (App-Specific Rules):** Custom JSON/Key-value settings tagged to the specific `u_app_id` to ignore dynamic fields expected to change between hosts.
* **Contextual Anti-Pattern & Risk Validation (Critical):**
  * **Environment-Aware Keyword Watchlist:** The engine must check the Target Host's environment (via `used_for` attribute). If Target = PRD, triggers a severe warning if prohibited keywords (e.g., `dev`, `sit`, `nft`) are detected in critical fields like `fqdn`, `name`, or `short_description`.
  * **Network Range Validation:** Validates if IP addresses (`ip_address`, `default_gateway`) fall within the accepted subnet/range for that specific environment.

### 5.3 Cluster & High Availability (HA) Handling
* **Explicit Node Selection:** The UI mandates selecting a specific Target Server (e.g., `snftwwsaapp01`) from a list of available hosts in that environment. 
* **Intra-Cluster Consistency Check:** The system runs a background validation across all hosts sharing the same `u_app_id` and `used_for` (Environment) to ensure nodes within the same cluster share identical baseline configurations (e.g., same RAM, CPU Count, OS Version). Drift within the cluster itself triggers a warning banner.

---

## 6. UI / UX Architecture
Built as a Single Page Application (SPA) within ServiceNow, divided into three main sections:

### 6.1 Context Selector (Top Bar)
* **App Context (Auto-filled):** Displays logged-in user's `u_app_id` (e.g., 808) and `u_app_acronym` (e.g., WSA).
* **Source Environment Selector:** Dropdown populated by unique `used_for` values (e.g., DEV, NFT, SIT, UAT).
* **Source Host Selector:** Dropdown listing specific server names (`fqdn` or `name`) matching the selected App Context and Source Environment.
* **Target Environment Selector:** Dropdown for the target (e.g., PRD).
* **Target Host Selector:** Dropdown listing specific server names for the Target Environment.
* **Action Buttons:** "Compare", "Export to CSV/JSON", and **"Attach to Change Request"** (Includes a text input field for a `CHGXXXXXXX` record number).

### 6.2 Host Components Tree (Left Panel)
* **Hierarchy Visibility:** Displays the Host as the root node, with any discovered child software instances (Middleware/DB) branching below it.
* **Visual Coding System:**
  * 🟢 **Green (Match):** Properties match perfectly.
  * 🔴 **Red (Mismatch):** Value differences or missing keys detected.
  * 🟡 **Yellow (Warning):** Values match, but anti-patterns/risks detected (e.g., PRD server contains 'nft' in description).
  * ⚪ **Gray (Ignore):** Property skipped based on Meta Data rules.

### 6.3 Diff Viewer (Right Panel)
* **Git-Style Side-by-Side Table:** Source Host values (left), Target Host values (right).
* **Color Highlighting:** Green (Added), Red (Removed/Different), Yellow (Modified).
* **Controls:** `Show only differences` toggle switch, and a real-time text search bar to filter specific keys (e.g., searching for `cpu` or `ram`).

---

## 7. Backend Logic & Technical Architecture

### 7.1 Status Propagation Algorithm
The backend fetches the CI payload and evaluates statuses to generate the UI indicators.
* **Logic Flow:**
  1. Fetch JSON payloads for Source Host and Target Host via `cmdb_ci_server` API.
  2. Filter out ignored keys (e.g., `sys_updated_on`).
  3. Calculate Diff status (MATCH, WARNING, MISMATCH) based on exact string comparison and context-aware anti-patterns (checking the `used_for` flag).
  4. Pass the analyzed JSON payload to the Diff Viewer component.

### 7.2 Data Security & Access Control
* **Strict ACL Inheritance:** While standard fields should not contain secrets, the tool will **strictly enforce ServiceNow Access Control Lists (ACLs)**. 
* **Visibility Rules:** The backend queries will run under the context of the logged-in user (passing User Claims to the API). If a user lacks read access to a specific CI or field, that data will not be returned by the ServiceNow API.
* **Performance:** Direct host-to-host querying (`sysparm_query`) ensures extremely lightweight database queries and fast rendering times.

---

## 8. Supporting Features & Workflow Automation
* **Native CR Attachment:** Users can enter a Change Request number (`CHGXXXXXXX`) directly in the UI. The tool will natively generate a JSON/CSV payload of the diff results and attach it to the target CR via API, eliminating manual file handling. 
* **Local Export Functionality:** Users can still fall back to manually exporting the diff results in CSV or JSON format if needed for external audits.
