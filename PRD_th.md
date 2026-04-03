# Product Requirement Document (PRD)
**Product Name:** ServiceNow CSDM Configuration Diff Tool  
**Document Status:** Version 2.0 (Updated Edge Cases, Security & Workflow Automation)  
**Target Platform:** ServiceNow  
**Product Manager:** Nont Banditwong

---

## 1. Executive Summary & Objective
**The Problem:** ปัจจุบันการตรวจสอบ Configuration Items (CIs) ระหว่าง Environment (เช่น UAT กับ PROD) ก่อนทำการ Deploy มักต้องทำแบบ Manual ซึ่งใช้เวลานานและเสี่ยงต่อ Human Error (Configuration Drift) ส่งผลให้เกิด Production Incidents จากการตั้งค่าผิดพลาด 
**The Solution:** พัฒนาระบบ On-Demand Impact Analysis ภายใน ServiceNow เพื่อเปรียบเทียบ CI Configuration (Key-Value) แบบ Peer-to-Peer ระหว่าง 2 Environments โดยเน้นที่ `Application Service` ลงลึกไปถึงระดับ Infrastructure เพื่อช่วยให้ Application Team ยืนยันความถูกต้องได้อย่างแม่นยำและรวดเร็ว

### 1.1 Out of Scope (Non-Goals / นอกเหนือขอบเขต)
เพื่อควบคุม Scope และประสิทธิภาพการทำงานของ V1 เครื่องมือนี้จะ**ไม่รวมถึง**:
* **Auto-remediation:** เครื่องมือนี้ใช้สำหรับระบุ Configuration Drift เท่านั้น จะไม่มีการ Auto-sync หรือ Push คอนฟิกจาก Source ไป Target อัตโนมัติ (ระบบทำงานแบบ Read-only)
* **Deep OS/File Parsing:** จะไม่มีการเข้าไป Parse ไฟล์ระดับ OS (เช่น `.conf`, `.yaml`, `.xml`) ที่อยู่บนเครื่อง Host แต่จะวิเคราะห์เฉพาะ Properties ที่ถูกดึงเข้ามาใน Standard Tables ของ ServiceNow CMDB เท่านั้น

---

## 2. Target Audience
* **Primary Users:** Software Engineers / DevOps / SRE (ผู้เตรียมและ Execute การ Deploy)
* **Secondary Users:** Tech Lead / Change Manager / QA (ผู้ Review และ Approve Change Request)

## 3. Success Metrics (KPIs)
* **Incident Reduction:** ลดจำนวน P1/P2 Incidents ที่เกิดจาก Configuration Mismatch ลง [X]%
* **Time Saved:** ลดระยะเวลาการทำ Impact Analysis ลง [X] ชั่วโมงต่อรอบการ Deploy
* **Adoption Rate:** จำนวน Change Requests ที่มีการแนบไฟล์ผลลัพธ์จากระบบการ Diff เข้าไปโดยตรง เป็นสัดส่วน [X]% ของ CR ทั้งหมด

---

## 4. User Stories
1. **US1:** ในฐานะ *Software Engineer* ฉันต้องการ *เปรียบเทียบ Configuration ของ Application Service แบบ Side-by-Side ระหว่าง 2 Environments* เพื่อให้ *มั่นใจว่าไม่มีค่าคอนฟิกที่ตกหล่นก่อนนำขึ้นระบบจริง*
2. **US2:** ในฐานะ *Tech Lead* ฉันต้องการ *ให้ระบบวิเคราะห์ Context และแจ้งเตือน (Anti-pattern Alert) หากมีค่า Lower-Environment หลงเหลืออยู่ในฝั่ง PROD* เพื่อให้ *ป้องกันข้อผิดพลาดร้ายแรงก่อน Approve CR*
3. **US3:** ในฐานะ *DevOps* ฉันต้องการ *เปรียบเทียบ Node ต้นทาง กับ Node ปลายทาง (กรณีเป็น Cluster) พร้อมเลือก Node เป้าหมายได้เองในขณะที่ระบบตรวจสอบความสอดคล้องกันของทั้ง Cluster อัตโนมัติ* เพื่อ *รับประกันความเสถียรของ Cluster*
4. **US4:** ในฐานะ *Change Manager* ฉันต้องการ *แนบผลลัพธ์การเปรียบเทียบเข้ากับ Change Request (CR) ได้อย่างราบรื่นจากระบบโดยตรง* เพื่อให้ *มีหลักฐานอ้างอิงและประเมินความเสี่ยงโดยไม่ต้องจัดการดาวน์โหลดไฟล์ลงเครื่องตัวเอง*
5. **US5 (Data Quality):** ในฐานะ *ผู้ใช้งาน* ฉันต้องการ *ให้ระบบแจ้งเตือนอย่างชัดเจนหากโครงสร้าง CSDM (Topology) แตกหักหรือขาดหาย* เพื่อ *ป้องกันการทึกทักไปเองว่าคอนฟิกตรงกันทั้งหมด ทั้งๆ ที่แท้จริงแล้วระบบหา Infrastructure ปลายทางไม่พบ*

---

## 5. System Scope & Architecture
### 5.1 CSDM Integration & Relationship
* ระบบดึงข้อมูลแบบ **On-demand (Real-time)** จากฐานข้อมูล ServiceNow โดยตรง
* **Starting Point:** เริ่มต้นค้นหาจากระดับ `Application Service`
* **Hierarchy & Depth (3 Levels):**
  1. **Level 1 (Host):** VM Instance / Hardware
  2. **Level 2 & 3 (Software Instances - Peer Level):** Middleware และ Database จะแสดงผลเป็น Sibling ภายใต้ VM เดียวกันตามหลัก "Runs on"

### 5.2 2-Tier Configuration Meta Data
ระบบจัดการตัวแปรและข้อยกเว้นผ่าน Meta Data 2 ระดับ:
* **Tier 1: Global Rules:** กฎระดับระบบ เช่น System fields ที่ต้อง Ignore (`sys_updated_on`) 
* **Tier 2: App-Specific Rules:** ข้อมูล JSON/Key-Value ที่ฝังอยู่ระดับ `Application Service` เพื่อให้ทีม App กำหนด Ignore List ของตัวเองได้

### 5.3 Comparison Engine & Logic
* **Missing Keys:** ตรวจจับ Key ที่มีใน Source แต่ไม่มีใน Target (และสลับกัน)
* **Value Matching:** เปรียบเทียบความแตกต่างของข้อมูล
* **Contextual Anti-Pattern & Risk Validation (Critical):**
  * **Environment-Aware Keyword Watchlist:** ระบบต้องตรวจสอบฟิลด์ `Environment` ของ Target CI ก่อน หาก Target = PROD จะทำการแจ้งเตือนขั้นร้ายแรงหากพบคำต้องห้าม (เช่น `dev`, `sit`, `test`) แต่หาก Target เป็นเพียง QA กฎนี้จะถูกข้ามไปเพื่อลด False Positive
  * **IP Range Validation:** ตรวจสอบว่า IP Address อยู่ใน Range ที่อนุญาตของ Environment นั้นๆ หรือไม่

### 5.4 Cluster & High Availability Handling
* **Explicit Node Selection:** หาก Target เป็น Cluster ระบบจะแสดง Dropdown บังคับให้ผู้ใช้เลือก Target Node ที่ต้องการเปรียบเทียบด้วยตัวเอง (ปุ่ม Compare จะถูก Disable จนกว่าจะเลือก)
* **Intra-Cluster Consistency Check:** ระบบจะตรวจสอบ Node อื่นๆ ใน Target Cluster แบบเบื้องหลัง หากคอนฟิกภายใน Cluster ไม่ตรงกันเอง จะแสดง Warning Banner ทันที

---

## 6. UI / UX Design
ออกแบบเป็น Single Page Application (SPA) แบ่งเป็น 3 ส่วนหลัก:

### 6.1 Context Selector (Top Bar)
* ช่องค้นหา/เลือก `Application Service`
* ช่องเลือก Source Environment / Target Environment (รวมถึง Mandatory Dropdown หากปลายทางเป็น Cluster)
* ปุ่ม Action: "Compare", "Export to CSV/JSON", และ **"Attach to Change Request"** (พร้อมช่อง Input ให้กรอกหมายเลข `CHGXXXXXXX`)

### 6.2 Full CSDM Topology Tree (Left Panel)
* แสดงโครงสร้าง Tree View ทั้งหมดแบบกางออก เพื่อให้เห็นภาพรวม Architecture เสมอ
* **Visual Status Indicators:** * 🟢 **Green (Match):** Properties ตรงกัน
  * 🔴 **Red (Mismatch/Critical):** มีความแตกต่าง หรือมี CI หายไป
  * 🟡 **Yellow (Warning):** ค่าตรงกัน แต่พบ Anti-pattern/Risk
  * ⚪ **Gray (Ignore):** ข้ามการตรวจสอบตาม Meta-data
  * 🟣 **Purple/Broken Link Icon (Incomplete Topology):** ไฮไลท์แจ้งเตือนเมื่อ Application Service ไม่มี Infrastructure ปลายทางผูกอยู่ (เป็นปัญหาด้าน Data Quality ไม่ใช่ Configuration Match)
* **Recursive Status Propagation (Bubble-up):** หาก Node ลูกมีสถานะ Red/Warning ตัว Parent Node จะแสดงสถานะแจ้งเตือนตามไปด้วยเพื่อดึงความสนใจของผู้ใช้

### 6.3 Side-by-Side Diff Viewer (Right Panel)
* ตารางเปรียบเทียบ Key-Value สไตล์ Git (Source ซ้าย, Target ขวา) 
* **Color Coding:** เขียว (Added), แดง (Removed/Different), เหลือง (Modified)
* **Controls:** Toggle "Show only differences" และ Search Bar ค้นหา Key แบบ Real-time

---

## 7. Backend Logic & Technical Architecture

### 7.1 Recursive Status Propagation Algorithm
ระบบ Backend จะสร้าง CSDM Tree ในหน่วยความจำและประมวลผลจากล่างขึ้นบน (Bottom-up)
* **Logic Flow:**
  1. ตรวจสอบความสมบูรณ์ของ Topology (ตั้งค่า Flag หากพบ Broken relationships)
  2. ประมวลผล Ignore Lists (Meta Data)
  3. คำนวณ `self_status` (MATCH, WARNING, MISMATCH) โดยอิงตามการ Diff และ Environment-Aware Anti-patterns
  4. ดึงสถานะ `tree_status` ของ Node ลูกทั้งหมด และดึงสถานะขั้นสูงสุด (Highest Severity) มาเป็น `tree_status` ของ Node ปัจจุบัน

### 7.2 Data Security & Access Control
* **Strict ACL Inheritance:** แม้ว่า Standard Fields ใน ServiceNow ไม่ควรมีข้อมูลความลับ (Secrets) แต่ในทางปฏิบัติผู้ใช้อาจใส่ข้อมูลผิดประเภทลงไป ดังนั้นระบบจะต้อง **บังคับใช้ Access Control Lists (ACLs) ของ ServiceNow อย่างเคร่งครัด**
* **Visibility Rules:** การดึงข้อมูล Backend จะทำงานภายใต้สิทธิ์ (Context) ของผู้ใช้ที่กำลัง Login หากผู้ใช้ไม่มีสิทธิ์เข้าถึง (Read Access) ใน CI หรือ Field ใด ข้อมูลนั้นจะถูกซ่อน (Masked/Hidden) ใน Diff Viewer โดยอัตโนมัติ
* **Performance:** การจำกัดขอบเขตแบบ Peer-to-peer ช่วยรับประกันว่า Database Queries จะใช้ทรัพยากรน้อยและ Render ได้รวดเร็วโดยไม่เกิด Timeout

---

## 8. Supporting Features & Workflow Automation
* **Native CR Attachment:** ผู้ใช้สามารถกรอกหมายเลข Change Request (`CHGXXXXXXX`) ใน UI ระบบจะสร้าง Payload JSON/CSV ของผลการเปรียบเทียบและแนบเข้าไปที่ CR นั้นผ่าน API อัตโนมัติ เพื่อลดการจัดการไฟล์แบบ Manual
* **Local Export Functionality:** ยังคงรองรับการ Export ผลลัพธ์เป็นไฟล์ CSV หรือ JSON ลงเครื่อง เพื่อใช้เป็นหลักฐานประกอบการ Audit ภายนอกหากจำเป็น