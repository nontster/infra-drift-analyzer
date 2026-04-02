# Epic: ServiceNow CSDM Configuration Diff Tool

## Epic 1: Backend & Data Architecture

### Story 1.1: 2-Tier Configuration Meta Data Model
**Type:** User Story
**Description:** ในฐานะ System Admin และ Application Team ฉันต้องการระบบจัดการ Meta Data แบบ 2 ระดับ (Global และ App-specific) เพื่อกำหนดข้อยกเว้นและตัวแปรของ Environment ในการทำ Diff

**Acceptance Criteria (AC):**
* **AC1:** สามารถตั้งค่า Tier 1 (Global Rules) ในระดับ System Properties หรือ Custom Table เพื่อกำหนด Ignore Fields เริ่มต้น (เช่น `sys_updated_on`) และ Global Anti-patterns
* **AC2:** สามารถตั้งค่า Tier 2 (App-Specific Rules) โดยเพิ่ม Field (เช่น JSON format) ที่ `Application Service` CI เพื่อให้ App Team กำหนด Ignore List และ Environment Variable Mapping ของตัวเองได้
* **AC3:** Backend logic สามารถนำข้อมูล Tier 1 และ Tier 2 มา Merge กันเพื่อใช้เป็น Rule Set ในการทำ Compare ได้อย่างถูกต้อง

### Story 1.2: CSDM Data Fetching & Tree Construction
**Type:** User Story
**Description:** ในฐานะ Backend System ฉันต้องการดึงข้อมูล CI และความสัมพันธ์จากฐานข้อมูล ServiceNow เพื่อสร้างโครงสร้าง Tree สำหรับเปรียบเทียบ

**Acceptance Criteria (AC):**
* **AC1:** ระบบสามารถเริ่มต้นค้นหาจาก `Application Service` CI และดึงข้อมูลความสัมพันธ์ลงมาได้ 3 ระดับ
* **AC2:** โครงสร้างที่ได้ต้องเป็น Peer-level Structure โดยให้ Middleware (Tomcat/WebSphere) และ Database (Oracle) อยู่ในระดับเดียวกัน (Siblings) ภายใต้ VM (Host) เดียวกัน
* **AC3:** สามารถดึงค่า Properties ทั้งหมดของ CI แต่ละตัวออกมาเป็นรูปแบบ Key-Value ได้แบบ Real-time

### Story 1.3: Diff Engine & Recursive Status Propagation
**Type:** User Story
**Description:** ในฐานะ Backend System ฉันต้องการเปรียบเทียบข้อมูล CI แบบ Peer-to-Peer และประมวลผลสถานะจากล่างขึ้นบน (Bottom-Up) เพื่อส่งให้ Frontend

**Acceptance Criteria (AC):**
* **AC1:** ระบบสามารถเปรียบเทียบ Key-Value ระหว่าง Source และ Target และระบุได้ว่า Key ใดหายไป หรือ Value ใดแตกต่างกัน
* **AC2:** ระบบสามารถคำนวณ `self_status` ของแต่ละ Node ได้ (Match, Mismatch, Warning, Ignore)
* **AC3:** ระบบสามารถทำ Recursive Logic เพื่อดึงสถานะที่ร้ายแรงที่สุดของ Node ลูกขึ้นมาเป็น `tree_status` ของ Node แม่ได้
* **AC4:** API Response ส่งกลับมาเป็น JSON structure ที่ครอบคลุมทั้ง Topology tree, diff data, และ status

### Story 1.4: Anti-Pattern & Cluster Consistency Validation
**Type:** User Story
**Description:** ในฐานะ Tech Lead ฉันต้องการให้ระบบตรวจสอบ Anti-pattern และความสม่ำเสมอของ Cluster เพื่อป้องกันความเสี่ยง

**Acceptance Criteria (AC):**
* **AC1:** ระบบสามารถตรวจสอบ Keyword ใน Target Values (เช่น คำว่า `dev`, `uat` บน `prod`) หากพบให้ตั้งค่า Status เป็น Warning/Critical
* **AC2:** ระบบสามารถ Validate IP Range ตามที่กำหนดไว้ใน Environment นั้นๆ ได้
* **AC3:** หาก Target เป็น Cluster ระบบจะทำการดึงข้อมูล Node ทั้งหมดใน Cluster มาเปรียบเทียบกันเอง (Intra-Cluster) แบบ Background
* **AC4:** หากพบ Configuration Drift ภายใน Target Cluster เดียวกัน ระบบต้องแนบ Warning Message ไปใน API Response

---

## Epic 2: Frontend UI & UX

### Story 2.1: Context Selector (Top Bar)
**Type:** User Story
**Description:** ในฐานะ User ฉันต้องการหน้าต่างควบคุมเพื่อระบุ Application Service และ Environment ที่ต้องการเปรียบเทียบ

**Acceptance Criteria (AC):**
* **AC1:** มีช่อง Autocomplete ค้นหา `Application Service`
* **AC2:** มี Dropdown เลือก Source CI และ Target CI
* **AC3 (Cluster Logic):** หาก Target CI เป็น Cluster ระบบต้องแสดง Dropdown บังคับให้ User เลือก Target Node แบบเจาะจง โดยไม่มีการ Auto-select ค่าเริ่มต้น
* **AC4:** ปุ่ม "Compare" จะถูก Disable จนกว่า User จะกรอก/เลือกข้อมูลบังคับทั้งหมดครบถ้วน

### Story 2.2: CSDM Topology Tree View (Left Panel)
**Type:** User Story
**Description:** ในฐานะ User ฉันต้องการเห็นโครงสร้างสถาปัตยกรรมทั้งหมดพร้อม Visual Indicator เพื่อให้รู้ว่ามีจุดไหนที่มีความแตกต่าง

**Acceptance Criteria (AC):**
* **AC1:** แสดงผล Tree View แบบเต็ม (Full Hierarchy) โดยอิงตาม API Response
* **AC2:** หน้า UI แสดงวงกลมสี (Indicator) หน้าชื่อ CI ตาม `tree_status`: สีเขียว (Match), สีแดง (Mismatch), สีเหลือง (Warning), สีเทา (Ignore)
* **AC3:** สามารถกด Expand/Collapse กิ่งของ Tree ได้
* **AC4:** เมื่อ User คลิกที่ Node ใดๆ ระบบจะนำข้อมูลของ Node นั้นไปแสดงใน Diff Viewer ทางด้านขวา

### Story 2.3: Side-by-Side Diff Viewer (Right Panel)
**Type:** User Story
**Description:** ในฐานะ User ฉันต้องการดูรายละเอียดความแตกต่างของ Configuration แบบ Side-by-Side เพื่อการวิเคราะห์ที่แม่นยำ

**Acceptance Criteria (AC):**
* **AC1:** แสดงตารางเปรียบเทียบ 2 คอลัมน์ (Source ด้านซ้าย, Target ด้านขวา)
* **AC2:** มีการใช้สี Highlight แบบ Git-style (เพิ่มเข้ามา = เขียว, ถูกลบ/ต่างกัน = แดง, เปลี่ยนแปลง = เหลือง)
* **AC3:** มี Toggle Switch `Show only differences` เมื่อเปิดใช้งานจะซ่อนแถวที่ข้อมูลเหมือนกัน
* **AC4:** มี Search Bar เพื่อค้นหา Key ภายในตารางแบบ Real-time

---

## Epic 3: Supporting Features

### Story 3.1: Export Diff Results
**Type:** User Story
**Description:** ในฐานะ Change Manager ฉันต้องการ Export ผลการเปรียบเทียบเพื่อนำไปแนบเป็นหลักฐานใน Change Request

**Acceptance Criteria (AC):**
* **AC1:** มีปุ่ม "Export" บน UI
* **AC2:** สามารถเลือก Export รูปแบบไฟล์ได้ (CSV และ JSON)
* **AC3:** ไฟล์ Export ต้องระบุข้อมูล Context (App Service, Source CI, Target CI, วันที่ทำการ Diff) และ List ของ Properties ที่มีความแตกต่าง หรือติด Anti-pattern