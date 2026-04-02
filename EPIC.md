# Epic: ServiceNow CSDM Configuration Diff Tool

## Epic 1: Backend & Data Architecture

### Story 1.1: 2-Tier Configuration Meta Data Model
**Type:** User Story
**Description:** As a System Admin and Application Team member, I want a 2-tier metadata management system (Global and App-specific) to define exceptions and environment variables for the diff process.

**Acceptance Criteria (AC):**
* **AC1:** Able to configure Tier 1 (Global Rules) at the System Properties or Custom Table level to define default ignored fields (e.g., `sys_updated_on`) and global anti-patterns.
* **AC2:** Able to configure Tier 2 (App-Specific Rules) by adding a custom field (e.g., JSON format payload) to the `Application Service` CI, allowing App Teams to define their own ignore lists and environment-specific variable mappings.
* **AC3:** Backend logic must successfully merge Tier 1 and Tier 2 rulesets to use as the baseline logic during the comparison process.

### Story 1.2: CSDM Data Fetching & Tree Construction
**Type:** User Story
**Description:** As a Backend System, I want to fetch CI data and relationships from the ServiceNow database to construct a correct tree structure for comparison.

**Acceptance Criteria (AC):**
* **AC1:** The system can initiate a search from the `Application Service` CI and fetch downstream relationships up to 3 levels deep.
* **AC2:** The resulting structure must follow a Peer-Level hierarchy, placing Middleware (Tomcat/WebSphere) and Database (Oracle) at the same level (siblings) operating under the same VM (Host).
* **AC3:** Capable of fetching all required CI properties as Key-Value pairs in real-time.

### Story 1.3: Diff Engine & Recursive Status Propagation
**Type:** User Story
**Description:** As a Backend System, I want to execute a peer-to-peer CI data comparison and process the status bottom-up (recursive) to deliver accurate indicators to the frontend.

**Acceptance Criteria (AC):**
* **AC1:** The system can compare Key-Value pairs between Source and Target, accurately identifying missing keys or differing values.
* **AC2:** The system can calculate the `self_status` of each individual node (Match, Mismatch, Warning, Ignore).
* **AC3:** The system executes recursive logic to bubble up the highest severity status among child nodes, setting it as the `tree_status` of the parent node.
* **AC4:** The API Response must return a properly structured JSON payload encompassing the topology tree, diff data, and all status indicators.

### Story 1.4: Anti-Pattern & Cluster Consistency Validation
**Type:** User Story
**Description:** As a Tech Lead, I want the system to validate anti-patterns and cluster consistency to prevent high-risk deployment errors.

**Acceptance Criteria (AC):**
* **AC1:** The system checks for restricted keywords in Target Values (e.g., finding `dev` or `uat` on `prod`). If detected, it sets the status to Warning/Critical.
* **AC2:** The system can validate if IP addresses fall within the defined subnet/range for the target environment.
* **AC3:** If the Target is a Cluster, the system performs a background intra-cluster comparison across all nodes within that cluster.
* **AC4:** If intra-cluster configuration drift is detected, the system must append a Warning Message to the API Response.

---

## Epic 2: Frontend UI & UX

### Story 2.1: Context Selector (Top Bar)
**Type:** User Story
**Description:** As a User, I want a control panel to specify the Application Service and the specific environments/nodes I want to compare.

**Acceptance Criteria (AC):**
* **AC1:** Contains an autocomplete search input for the `Application Service`.
* **AC2:** Contains dropdown selectors for Source CI and Target CI.
* **AC3 (Cluster Logic):** If the Target CI is identified as a cluster, the UI must display a mandatory dropdown forcing the user to explicitly select a specific Target Node (Auto-select/default selection is strictly prohibited).
* **AC4:** The "Compare" action button remains disabled until the user has fully populated all mandatory fields and explicit selections.

### Story 2.2: CSDM Topology Tree View (Left Panel)
**Type:** User Story
**Description:** As a User, I want to see the full architectural structure alongside visual indicators so I can instantly locate configuration discrepancies.

**Acceptance Criteria (AC):**
* **AC1:** Renders the full tree hierarchy based on the provided API response.
* **AC2:** Displays color-coded indicator circles next to the CI name based on the `tree_status`: Green (Match), Red (Mismatch), Yellow (Warning), Gray (Ignore).
* **AC3:** Allows users to interactively expand and collapse tree branches.
* **AC4:** Clicking on any node within the tree automatically renders that node's detailed comparison data in the right panel.

### Story 2.3: Side-by-Side Diff Viewer (Right Panel)
**Type:** User Story
**Description:** As a User, I want to view detailed configuration differences side-by-side for accurate analysis.

**Acceptance Criteria (AC):**
* **AC1:** Displays a two-column comparison table (Source context on the left, Target context on the right).
* **AC2:** Applies Git-style color highlighting to rows (Added = Green, Removed/Mismatch = Red, Modified = Yellow).
* **AC3:** Includes a `Show only differences` toggle switch; when active, it hides all rows with matching data.
* **AC4:** Includes a real-time search bar to filter specific keys within the rendered table.

---

## Epic 3: Supporting Features

### Story 3.1: Export Diff Results
**Type:** User Story
**Description:** As a Change Manager, I want to export the comparison results to attach as validation evidence in a Change Request (CR).

**Acceptance Criteria (AC):**
* **AC1:** An "Export" button is accessible on the UI after a comparison is run.
* **AC2:** Allows the user to select the export file format (CSV or JSON).
* **AC3:** The exported document must include metadata context (App Service ID, Source CI, Target CI, Date/Time of execution) along with the isolated list of properties containing differences or anti-pattern warnings.