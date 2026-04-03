# Product Requirement Document (PRD): ServiceNow CSDM Configuration Diff Tool
**Product Name:** ServiceNow CSDM Configuration Diff Tool  
**Document Version:** 2.0 (Updated to include edge-case handling, strict ACLs, and workflow automation)
**Target Platform:** ServiceNow
**Product Manager:** Nont Banditwong

---

## 1. Executive Summary & Objective
**The Problem:** Currently, verifying Configuration Items (CIs) across different environments (e.g., UAT vs. PROD) prior to deployment is a manual and error-prone process. Configuration drift is a leading cause of production incidents, often resulting from missing keys or lingering lower-environment variables (e.g., UAT URLs in PROD).
**The Solution:** Develop an On-Demand Impact Analysis tool within ServiceNow that performs a peer-to-peer, side-by-side comparison of CI configurations (Key-Value pairs). By mapping the correct CSDM infrastructure topology, the tool empowers Application Teams to quickly identify discrepancies and anti-patterns before executing a deployment.

### 1.1 Out of Scope (Non-Goals)
To ensure performance and focused delivery, V1 of this tool explicitly **excludes**:
* **Auto-remediation:** The tool identifies drift but will NOT automatically sync or push configurations from the Source to Target environments. It is read-only.
* **Deep OS/File Parsing:** The tool will not parse host-level `.conf`, `.yaml`, or `.xml` files. It only analyzes properties that are actively ingested and populated within standard ServiceNow CMDB tables.

---

## 2. Target Audience & Personas
* **Primary Users:** Software Engineers / DevOps / SREs (Executing deployments and checking configurations).
* **Secondary Users:** Tech Leads / Change Managers / QA (Reviewing and approving Change Requests based on accurate impact analysis).

## 3. Key Metrics (Success Criteria)
* **Incident Reduction:** Decrease P1/P2 incidents caused by configuration mismatches by [X]%.
* **Time Saved:** Reduce the time spent on manual configuration validation and Change Request (CR) preparation by [X] hours per deployment cycle.
* **Adoption Rate:** Achieve [X]% of Change Requests featuring a directly attached diff report as validation evidence.

---

## 4. User Stories
* **US1:** As a *Software Engineer*, I want to *compare Application Service configurations side-by-side between UAT and PROD* so that *I can ensure no configurations are missed prior to deployment.*
* **US2:** As a *Tech Lead*, I want *the system to contextually alert me (Anti-pattern Alert) if lower-environment values are found in the PROD environment* so that *I can prevent critical deployment errors before approving a CR.*
* **US3:** As a *DevOps Engineer*, I want to *explicitly select a specific PROD node from a cluster to compare against UAT, while the system automatically checks for intra-cluster consistency,* so that *I can guarantee cluster stability.*
* **US4:** As a *Change Manager*, I want to *seamlessly attach the comparison results directly to a Change Request* so that *I have an audit trail and risk assessment artifact without managing local file downloads.*
* **US5 (Data Quality):** As a *User*, I want *the system to clearly notify me if the CSDM topology is broken or missing child CIs* so that *I do not falsely assume a perfect configuration match when the underlying infrastructure simply wasn't found.*

---

## 5. Functional Requirements

### 5.1 System Scope & CSDM Integration
* **Trigger:** Real-time (On-demand) data fetching triggered by the user.
* **Data Source:** Directly queries CI standard tables/properties in ServiceNow.
* **Starting Context:** Searching begins at the `Application Service` CI level.
* **Topology Depth & Architecture (Peer-Level Structure):**
  * **Level 1:** VM Instance / Host (Hardware CI)
  * **Level 2 (Siblings under VM):** * Middleware (`cmdb_ci_app_server_tomcat`, `cmdb_ci_app_server_websphere`)
    * Database (`cmdb_ci_db_ora_instance`, `cmdb_ci_db_ora_catalog`)

### 5.2 Comparison Engine & Logic
* **Key-Value Matching:** Compares CI property fields.
* **Missing Key Identification:** Highlights keys present in Source but missing in Target (and vice versa).
* **Configuration Meta Data (2-Tier Architecture):**
  * **Tier 1 (Global Rules):** System-wide exclusions set by Admins (e.g., ignoring `sys_updated_on`, global anti-patterns).
  * **Tier 2 (App-Specific Rules):** Custom JSON/Key-value settings stored at the `Application Service` level, managed by App Teams (e.g., App-specific ignore lists).
* **Contextual Anti-Pattern & Risk Validation (Critical):**
  * **Environment-Aware Keyword Watchlist:** The engine must check the Target CI's `Environment` attribute first. If Target = PROD, triggers a severe warning if prohibited keywords (e.g., `dev`, `sit`, `test`) are detected. If Target = QA, the rule is bypassed.
  * **IP Range Validation:** Validates if IP addresses fall within the accepted subnet/range for that specific environment.

### 5.3 Cluster & High Availability (HA) Handling
* **Explicit Node Selection:** When comparing a single node (Source) to a cluster (Target), the UI presents a mandatory dropdown listing all Target nodes. The "Compare" button is disabled until a target node is chosen.
* **Intra-Cluster Consistency Check:** The system runs a background validation to ensure all nodes within the Target cluster share identical configurations. Drift within the cluster itself triggers a warning banner.

---

## 6. UI / UX Architecture
Built as a Single Page Application (SPA) within ServiceNow, divided into three main sections:

### 6.1 Context Selector (Top Bar)
* **App Selector:** Autocomplete search for `Application Service` ID.
* **Source CI:** Dropdown/Selector for the Source Environment.
* **Target CI:** Dropdown/Selector for the Target Environment (includes mandatory node selection for clusters).
* **Action Buttons:** "Compare", "Export to CSV/JSON", and **"Attach to Change Request"** (Includes a text input field for a `CHGXXXXXXX` record number).

### 6.2 Full CSDM Topology Tree (Left Panel)
* **Full Hierarchy Visibility:** Displays the complete tree structure regardless of differences to provide architectural context.
* **Visual Coding System:**
  * 🟢 **Green (Match):** Properties and relationships match perfectly.
  * 🔴 **Red (Mismatch):** Value differences or missing keys detected.
  * 🟡 **Yellow (Warning):** Values match, but anti-patterns/risks detected (e.g., wrong IP range).
  * ⚪ **Gray (Ignore):** Node skipped based on Meta Data rules.
  * 🟣 **Purple/Broken Link Icon (Incomplete Topology):** Highlights when an Application Service has no downstream infrastructure mapped, alerting the user to a CMDB data quality issue rather than a clean configuration match.
* **Recursive Status Propagation (Bubble-up):** If a child node has a Red/Warning status, the parent node will display a Warning/Red indicator to guide user attention.

### 6.3 Diff Viewer (Right Panel)
* **Git-Style Side-by-Side Table:** Source values (left), Target values (right).
* **Color Highlighting:** Green (Added), Red (Removed/Different), Yellow (Modified).
* **Controls:** `Show only differences` toggle switch, and a real-time text search bar to filter specific keys.

---

## 7. Backend Logic & Technical Architecture

### 7.1 Recursive Status Propagation Algorithm
The backend constructs the CSDM tree in memory and evaluates statuses bottom-up to generate the `self_status` and `tree_status`.
* **Logic Flow:**
  1. Verify Topology Completeness (Flag broken relationships).
  2. Check Ignore Lists (Meta Data).
  3. Calculate `self_status` (MATCH, WARNING, MISMATCH) based on diffs and context-aware anti-patterns.
  4. Recursively fetch `tree_status` of child nodes, bubbling up the highest severity level to the parent's `tree_status`.

### 7.2 Data Security & Access Control
* **Strict ACL Inheritance:** While standard fields should not contain secrets, users often misuse fields (e.g., pasting connection strings into "Notes"). Therefore, the tool will **strictly enforce ServiceNow Access Control Lists (ACLs)**. 
* **Visibility Rules:** The backend queries will run under the context of the logged-in user. If a user lacks read access to a specific CI or field, that data will be explicitly masked/hidden in the Diff Viewer.
* **Performance:** Peer-to-peer constraints guarantee lightweight database queries and fast rendering times without system timeouts.

---

## 8. Supporting Features & Workflow Automation
* **Native CR Attachment:** Users can enter a Change Request number (`CHGXXXXXXX`) directly in the UI. The tool will natively generate a JSON/CSV payload of the diff results and attach it to the target CR via API, eliminating manual file handling. 
* **Local Export Functionality:** Users can still fall back to manually exporting the diff results in CSV or JSON format if needed for external audits.