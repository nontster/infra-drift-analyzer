# ServiceNow CSDM Configuration Diff Tool

![Version](https://img.shields.io/badge/version-2.0-blue)
![Platform](https://img.shields.io/badge/platform-ServiceNow-green)
![Status](https://img.shields.io/badge/status-Active-brightgreen)

## 🚀 Overview
The **ServiceNow CSDM Configuration Diff Tool** is an on-demand impact analysis engine designed to eliminate production incidents caused by configuration drift. By providing a side-by-side comparison of Configuration Items (CIs) across environments (e.g., UAT vs. PROD), it empowers DevOps and SRE teams to validate infrastructure topology and property accuracy before deployment.

### The Problem
Manual verification of CIs is error-prone. Missing keys or lingering lower-environment variables (like UAT URLs in PROD) are leading causes of P1/P2 incidents.

### The Solution
A read-only, peer-to-peer comparison tool that maps CSDM infrastructure and identifies anti-patterns, ensuring your production environment remains stable and consistent.

---

## ✨ Key Features

### 🔍 Side-by-Side Comparison (Git-style)
* **Property Matching:** Real-time Key-Value pair analysis between source and target CIs.
* **Missing Key Identification:** Instant highlighting of discrepancies in keys present in one environment but missing in the other.
* **Visual Highlighting:** Green (Added), Red (Removed/Different), and Yellow (Modified/Warning).

### 🌳 CSDM Topology Visualization
* **Full Hierarchy Visibility:** A recursive tree view from the `Application Service` level down to Middleware and Databases.
* **Recursive Status Propagation:** A "bubble-up" logic where issues in child nodes (e.g., a DB mismatch) trigger a warning on the parent Application Service.
* **Data Quality Alerts:** Visual indicators (Purple/Broken Link) if the CSDM topology is missing child CIs.

### ⚠️ Smart Risk & Anti-Pattern Detection
* **Environment-Aware Watchlist:** Automatically flags PROD configurations containing prohibited keywords (e.g., `dev`, `test`, `sit`).
* **IP Range Validation:** Ensures target IP addresses align with the expected environment subnets.
* **Intra-Cluster Consistency:** Validates that all nodes within a target cluster share identical configurations to prevent HA instability.

### 📋 Workflow Automation
* **Native Change Request (CR) Attachment:** Seamlessly attach diff reports (JSON/CSV) to a `CHGXXXXXXX` record directly from the UI for audit trails.
* **2-Tier Rule Engine:** * **Global:** System-wide exclusions (e.g., `sys_updated_on`).
    * **App-Specific:** Custom ignore lists managed by individual Application Teams.

---

## 🛠 Architecture & Logic

### Recursive Status Algorithm
The tool constructs the CSDM tree in-memory and evaluates statuses bottom-up:
1. **Topology Check:** Is the infrastructure mapped correctly?
2. **Metadata Filtering:** Apply Global and App-specific ignore lists.
3. **Self-Status Evaluation:** Compare values and check for anti-patterns.
4. **Bubble-Up:** Pass the highest severity status up to the parent node.

### Security & Compliance
* **ACL Inheritance:** The tool strictly honors ServiceNow Access Control Lists. If a user cannot view a field in the native UI, it is masked in the Diff Viewer.
* **Read-Only:** The tool does not perform auto-remediation, ensuring zero risk of accidental configuration overrides.

---

## 💻 UI Guide

| Section | Purpose |
| :--- | :--- |
| **Context Selector** | Search for `Application Service`, select Source/Target nodes, and trigger comparison. |
| **Topology Tree** | Left-hand navigation showing the health of the entire infrastructure stack. |
| **Diff Viewer** | Right-hand Git-style table with "Show only differences" and real-time search filtering. |

---

## 🚦 Getting Started

### Prerequisites
* ServiceNow Vancouver or later.
* CSDM 4.0+ data model implementation.
* Read access to `cmdb_ci_*` tables for the end-user.

### Installation
1. Import the Update Set into your ServiceNow instance.
2. Configure **Global Metadata Rules** in the `x_diff_tool_metadata` table.
3. Ensure **Application Services** have the `Environment` attribute correctly populated.

---

## 📈 Success Metrics
* **Incident Reduction:** Targeted [X]% decrease in P1/P2 incidents related to config drift.
* **Efficiency:** Reduced manual validation time per deployment by [X] hours.
* **Compliance:** [X]% of Change Requests verified with attached diff reports.

---

## 🤝 Contribution
**Product Manager:** Nont Banditwong  
**Team:** Platform Infrastructure / DevOps Tools

---
*This tool is intended for internal configuration validation and does not parse host-level OS files (.yaml, .xml).*