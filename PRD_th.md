# Product Requirement Document (PRD)
**Product Name:** ServiceNow CSDM Configuration Diff Tool  
**Document Status:** Ready for Sprint Planning  
**Target Platform:** ServiceNow  

---

## 1. Executive Summary & Objective
**The Problem:** ปัจจุบันการตรวจสอบ Configuration Items (CIs) ระหว่าง Environment (เช่น UAT กับ PROD) ก่อนทำการ Deploy มักต้องทำแบบ Manual ซึ่งใช้เวลานานและเสี่ยงต่อ Human Error (Configuration Drift) ส่งผลให้เกิด Production Incidents จากการตั้งค่าผิดพลาด 
**The Solution:** พัฒนาระบบ On-Demand Impact Analysis ภายใน ServiceNow เพื่อเปรียบเทียบ CI Configuration (Key-Value) แบบ Peer-to-Peer ระหว่าง 2 Environments โดยเน้นที่ `Application Service` ลงลึกไปถึงระดับ Infrastructure เพื่อช่วยให้ Application Team ยืนยันความถูกต้องได้อย่างแม่นยำและรวดเร็ว

## 2. Target Audience
* **Primary Users:** Software Engineers / DevOps / SRE (ผู้เตรียมและ Execute การ Deploy)
* **Secondary Users:** Tech Lead / Change Manager / QA (ผู้ Review และ Approve Change Request)

## 3. Success Metrics (KPIs)
* **Incident Reduction:** ลดจำนวน P1/P2 Incidents ที่เกิดจาก Configuration Mismatch ลง [X]%
* **Time Saved:** ลดระยะเวลาการทำ Impact Analysis ลง [X] ชั่วโมงต่อรอบการ Deploy
* **Adoption Rate:** จำนวน Change Requests ที่มีการแนบไฟล์ Export ผลลัพธ์จาก Tool นี้เป็นสัดส่วน [X]% ของ CR ทั้งหมด

---

## 4. User Stories
1. **US1:** ในฐานะ *Software Engineer* ฉันต้องการ *เปรียบเทียบ Configuration ของ Application Service แบบ Side-by-Side ระหว่าง 2 Environments* เพื่อให้ *มั่นใจว่าไม่มีค่าคอนฟิกที่ตกหล่นก่อนนำขึ้นระบบจริง*
2. **US2:** ในฐานะ *Tech Lead* ฉันต้องการ *ให้ระบบแจ้งเตือน (Anti-pattern Alert) หากมีค่า UAT/SIT หลงเหลืออยู่ในฝั่ง PROD* เพื่อให้ *ป้องกันข้อผิดพลาดร้ายแรงก่อน Approve CR*
3. **US3:** ในฐานะ *DevOps* ฉันต้องการ *เปรียบเทียบ Node ต้นทาง กับ Primary Node ของปลายทาง (กรณีเป็น Cluster) พร้อมเลือก Node เป้าหมายได้เอง* เพื่อ *ประหยัดเวลาและรับประกันความถูกต้องของการกระจายคอนฟิก*
4. **US4:** ในฐานะ *Change Manager* ฉันต้องการ *Export หลักฐานการ Diff เป็นไฟล์* เพื่อใช้เป็น *ข้อมูลอ้างอิงประกอบ Change Request (CR)*

---

## 5. System Scope & Architecture
### 5.1 CSDM Integration & Relationship
* ระบบดึงข้อมูลแบบ **On-demand (Real-time)** จากฐานข้อมูล ServiceNow โดยตรง (ไม่ต้อง Parse ไฟล์ OS)
* **Starting Point:** เริ่มต้นค้นหาจากระดับ `Application Service`
* **Hierarchy & Depth (3 Levels):**
  1. **Level 1 (Host):** VM Instance / Hardware
  2. **Level 2 & 3 (Software Instances - Peer Level):** Middleware และ Database จะแสดงผลเป็น Sibling (พี่น้อง) ภายใต้ VM เดียวกันตามหลัก CSDM "Runs on"
* **Supported Technology Stack (Phase 1):**
  * Middleware: Tomcat (`cmdb_ci_app_server_tomcat`), WebSphere (`cmdb_ci_app_server_websphere`)
  * Database: Oracle (`cmdb_ci_db_ora_instance`, `cmdb_ci_db_ora_catalog`)

### 5.2 2-Tier Configuration Meta Data
ระบบจัดการตัวแปรและข้อยกเว้น (Environment-specific variables) ผ่าน Meta Data 2 ระดับ:
* **Tier 1: Global Rules (System Admin):** กฎที่บังคับใช้ทั้งหมด เช่น System fields ที่ต้อง Ignore (`sys_updated_on`) หรือ Global Anti-pattern 
* **Tier 2: App-Specific Rules (App Team):** ข้อมูล JSON/Key-Value ที่ฝังอยู่ระดับ `Application Service` CI เพื่อให้ทีม App กำหนด Ignore List หรือ Mapping Rule ของตัวเองได้

---

## 6. Functional Requirements
### 6.1 Comparison Engine (Diff Logic)
เปรียบเทียบ Properties ภายใน CI Class ทั้งสองฝั่ง โดยมีเงื่อนไขดังนี้:
* **Missing Keys:** ตรวจจับ Key ที่มีใน Source แต่ไม่มีใน Target (และสลับกัน)
* **Value Matching:** ค่าที่ควรเหมือนต้องเหมือน หากต่างให้แสดงสถานะ Mismatch
* **Anti-Pattern & Risk Validation (Critical):**
  * **Keyword Watchlist:** ตรวจสอบคำต้องห้ามในฝั่ง PROD (เช่น `dev`, `sit`, `uat`)
  * **IP Range Validation:** ตรวจสอบว่า IP Address อยู่ใน Range ที่อนุญาตของ Environment นั้นๆ หรือไม่

### 6.2 Cluster & High Availability Handling
* **Explicit Node Selection:** หาก Target เป็น Cluster ระบบจะแสดง Dropdown ให้ผู้ใช้เลือก Target Node ที่ต้องการเปรียบเทียบด้วยตัวเอง (Mandatory) จะไม่มีการ Auto-select ใดๆ และปุ่ม Compare จะกดไม่ได้จนกว่าจะเลือก
* **Intra-Cluster Consistency Check:** ระบบตรวจเช็ค Node อื่นๆ ใน Cluster ปลายทางเบื้องหลัง หากคอนฟิกไม่ตรงกันเองให้แสดง Warning 

---

## 7. UI / UX Design
ออกแบบเป็น Single Page Application แบ่งเป็น 3 ส่วนหลัก:

### 7.1 Context Selector (Top Bar)
* ช่องค้นหา/เลือก Application ID (`Application Service`)
* ช่องเลือก Source Environment / Target Environment
* **Target Node Dropdown:** สำหรับเลือก Node เปรียบเทียบ (Required Field)
* ปุ่ม Action: "Compare" และ "Export" (CSV/JSON)

### 7.2 Full CSDM Topology Tree (Left Panel)
* แสดงโครงสร้าง Tree View ทั้งหมดแบบกางออก เพื่อให้เห็นภาพรวม Architecture 
* **Visual Status Indicators:** * 🟢 **Green (Match):** Properties ตรงกัน
  * 🔴 **Red (Mismatch/Critical):** มีความแตกต่าง หรือมี CI หายไป
  * 🟡 **Yellow (Warning):** ค่าตรงกัน แต่พบ Anti-pattern/Risk
  * ⚪ **Gray (Ignore):** ข้ามการตรวจสอบตาม Meta-data

### 7.3 Side-by-Side Diff Viewer (Right Panel)
* แสดงตารางเปรียบเทียบ Key-Value (Source ซ้าย, Target ขวา) เมื่อคลิกที่ Node ใน Tree
* **Git-style Color Coding:** ไฮไลท์สีแยกความแตกต่างชัดเจน (เขียว=เพิ่ม, แดง=ลบ/เปลี่ยน)
* **Controls:**
  * Toggle "Show only differences" (ซ่อนค่าที่เหมือนกัน)
  * Search Bar สำหรับค้นหา Key ภายในตาราง

---

## 8. Backend Logic (Status Propagation)
ใช้หลักการ **Bottom-Up Recursive Algorithm** เพื่อประมวลผลสถานะจาก Node ลูกขึ้นไปหา Parent Node

* **Severity Levels:** Level 3 (🔴 Mismatch) > Level 2 (🟡 Warning) > Level 1 (🟢 Match) > Level 0 (⚪ Ignore)
* **Propagation Logic:**
  1. ประมวลผล Self-Status ของ Node ตัวเอง
  2. ดึงสถานะของ Node ลูกทั้งหมด (Children Status)
  3. เปรียบเทียบหา "Highest Severity" ระหว่างตัวเองและลูกๆ เพื่อกำหนดเป็น `Tree_Status` (สถานะที่จะโชว์บนวงกลมใน Tree View)
  4. ผลลัพธ์ (Payload) จะส่งกลับมาเป็น JSON ที่ระบุทั้ง `self_status` และ `tree_status` เพื่อลดภาระการคำนวณของ UI