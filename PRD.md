# Product Requirement Document (PRD): ServiceNow CSDM Configuration Diff Tool

**Document Version:** 1.0
**Target Platform:** ServiceNow
**Product Manager:** Gemini

---

## 1. Executive Summary & Objective
**The Problem:** Currently, verifying Configuration Items (CIs) across different environments (e.g., UAT vs. PROD) prior to deployment is a manual and error-prone process. Configuration drift is a leading cause of production incidents, often resulting from missing keys or lingering lower-environment variables (e.g., UAT URLs in PROD).
**The Solution:** Develop an On-Demand Impact Analysis tool within ServiceNow that performs a peer-to-peer, side-by-side comparison of CI configurations (Key-Value pairs). By mapping the correct CSDM infrastructure topology, the tool empowers Application Teams to quickly identify discrepancies and anti-patterns before executing a deployment.

## 2. Target Audience & Personas
* **Primary Users:** Software Engineers / DevOps / SREs (Executing deployments and checking configurations).
* **Secondary Users:** Tech Leads / Change Managers / QA (Reviewing and approving Change Requests based on accurate impact analysis).

## 3. Key Metrics (Success Criteria)
* **Incident Reduction:** Decrease P1/P2 incidents caused by configuration mismatches by [X]%.
* **Time Saved:** Reduce the time spent on manual configuration validation and Change Request (CR) preparation by [X] hours per deployment cycle.
* **Adoption Rate:** Achieve [X]% of CRs attaching exported diff reports as validation evidence.

## 4. User Stories
* **US1:** As a *Software Engineer*, I want to *compare Application Service configurations side-by-side between UAT and PROD* so that *I can ensure no configurations are missed prior to deployment.*
* **US2:** As a *Tech Lead*, I want *the system to alert me (Anti-pattern Alert) if lower-environment values are found in the PROD environment* so that *I can prevent critical deployment errors before approving a CR.*
* **US3:** As a *DevOps Engineer*, I want to *explicitly select a specific PROD node from a cluster to compare against UAT, while the system automatically checks for intra-cluster consistency,* so that *I can guarantee cluster stability.*
* **US4:** As a *Change Manager*, I want to *export the comparison results* so that *I have an audit trail and risk assessment artifact for the Change Request.*

---

## 5. Functional Requirements

### 5.1 System Scope & CSDM Integration
* **Trigger:** Real-time (On-demand) data fetching triggered by the user.
* **Data Source:** Directly queries CI standard tables/properties in ServiceNow (No external OS file parsing required).
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
  * **Tier 2 (App-Specific Rules):** Custom JSON/Key-value settings stored at the `Application Service` level, managed by App Teams (e.g., App-specific ignore lists, environment-specific variable mappings).
* **Anti-Pattern & Risk Validation (Critical):**
  * **Keyword Watchlist:** Triggers a severe warning if prohibited keywords (e.g., `dev`, `sit`, `test`) are detected in PROD values.
  * **IP Range Validation:** Validates if IP addresses fall within the accepted subnet/range for that specific environment.

### 5.3 Cluster & High Availability (HA) Handling
* **Explicit Node Selection (No Auto-Select):** When comparing a single node (Source) to a cluster (Target), the UI will present a mandatory dropdown listing all Target cluster nodes.
* **Validation Control:** The "Compare" button remains disabled until the user explicitly selects a target node.
* **Intra-Cluster Consistency Check:** The system runs a background validation to ensure all nodes within the Target cluster share identical configurations. If drift is detected within the cluster itself, a warning is displayed.

---

## 6. UI / UX Architecture
Built as a Single Page Application (SPA) within ServiceNow, divided into three main sections:

### 6.1 Context Selector (Top Bar)
* **App Selector:** Autocomplete search for `Application Service` ID.
* **Source CI:** Dropdown/Selector for the Source Environment (e.g., UAT VM).
* **Target CI:** Dropdown/Selector for the Target Environment (e.g., PROD VM/Cluster). Includes the mandatory node selection dropdown if a cluster is detected.
* **Action Buttons:** "Compare" (Disabled until contexts are set) and "Export".

### 6.2 Full CSDM Topology Tree (Left Panel)
* **Full Hierarchy Visibility:** Displays the complete tree structure (App Service -> VM -> [Middleware & DB]) regardless of whether differences exist, providing full architectural context.
* **Visual Coding System:**
  * 🟢 **Green (Match):** Properties and relationships match perfectly.
  * 🔴 **Red (Mismatch):** Value differences or missing keys/CIs detected.
  * 🟡 **Yellow (Warning):** Values match, but anti-patterns/risks detected (e.g., wrong IP range).
  * ⚪ **Gray (Ignore):** Node skipped based on Tier 1/Tier 2 Meta Data rules.
* **Recursive Status Propagation (Bubble-up):** If a child node (e.g., DB) has a Red status, the parent node (e.g., VM) will display a Warning/Red indicator to guide the user's attention.

### 6.3 Diff Viewer (Right Panel)
* **Git-Style Side-by-Side Table:** Source values on the left, Target values on the right.
* **Color Highlighting:** Green (Added), Red (Removed/Different), Yellow (Modified).
* **Controls:** * `Show only differences` toggle switch to reduce visual noise.
  * Real-time text search bar to filter specific property keys.

---

## 7. Backend Logic & Technical Architecture

### 7.1 Recursive Status Propagation Algorithm
The backend will construct the CSDM tree in memory and evaluate statuses from the bottom up to generate the `self_status` and `tree_status` for UI rendering.
* **Logic Flow:**
  1. Check Ignore Lists (Meta Data).
  2. Calculate `self_status` (MATCH, WARNING, MISMATCH) based on property diffs and anti-patterns.
  3. Recursively fetch `tree_status` of all child nodes.
  4. Assign the highest severity level among the node and its children as the current node's `tree_status`.
* **API Payload Structure:** Returns a JSON object containing `ci_name`, `self_status`, `tree_status`, `diff_data`, and a nested array of `children`.

### 7.2 Data Security & Performance
* **No Data Masking Required:** Since sensitive data (e.g., passwords, secrets) is not ingested into ServiceNow standard CI properties, no masking or encryption overhead is required during rendering.
* **Performance:** Peer-to-peer constraints (max 2 VMs/Clusters and their direct peers) guarantee lightweight database queries and fast rendering times without risk of system timeouts.

---

## 8. Supporting Features
* **Export Functionality:** Users can export the diff results in CSV or JSON format. This artifact is intended to be attached to ServiceNow Change Requests as validation evidence.