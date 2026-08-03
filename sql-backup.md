# Runbook: Backup & Restore SQL Server

## Metadata
- **Type:** Operational (Recurring / On-demand)
- **Frequency:** ตามที่ Dev Request หรือก่อนทำการ Rollback ที่กระทบ Production
- **Service:** SQL Server (Docker container)
- **Owner:** DevOps
- **Last verified:** 2026-07-31

## 1. Purpose

กรณีที่ทีมต้องการ Backup เพื่อเตรียมพร้อมสำหรับการแก้ไขปัญหาที่ส่งผลกระทบต่อ Production และต้องมี Rollback point ไว้ใช้งาน

## 2. Prerequisites

- [ ] Disk space เพียงพอที่ backup destination — มากกว่าขนาด database ปัจจุบัน x 1.5 เท่า (เผื่อ compression/temp file)
- [ ] Credential/permission ของ Database (SA หรือ account ที่มีสิทธิ์ BACKUP/RESTORE)
- [ ] Backup destination พร้อมใช้งาน (local disk / S3 / network share)
- [ ] ถ้าจะ Restore: แจ้งทีมที่เกี่ยวข้องล่วงหน้า (ยกเว้น incident เร่งด่วน) เพราะจะต้องตัดการเชื่อมต่อผู้ใช้ทั้งหมด

---

## 3. Steps

### A ผ่าน SSMS (SQL Server Management Studio)

**Backup:**
1. Connect to SQL Server
2. เลือก Database ที่ต้องการ
3. Tasks → Backup
4. กำหนด File name (ตรวจสอบว่า SQL Server มีการ Mount folder ไว้หรือไม่ ถ้าใช้ Docker Compose)
5. OK แล้ว Verify ว่า Backup File ถูกสร้างสำเร็จ

**Restore:**
1. Right-click ที่ Databases
2. เลือก **Restore File and Filegroup** (สำหรับ restore จากไฟล์ .bak โดยตรง) หรือ **Restore Database** (สำหรับ restore จาก backup ที่มีอยู่ในเครื่อง)
3. เลือก **Close existing connections**

   > **⚠️ คำเตือน:** ตัวเลือกนี้จะตัดการเชื่อมต่อของทุกคนที่กำลังใช้ database อยู่ทันที ต้องแจ้งทีมที่เกี่ยวข้องล่วงหน้าก่อนเสมอ ยกเว้นกรณี incident เร่งด่วนที่ต้อง restore ทันที

4. Restore แล้ว Validate ข้อมูลว่าถูกต้อง

---

### B ผ่าน SQL Server Docker CLI

> **⚠️ Security:** ห้ามพิมพ์ password ตรงใน command โดยตรง ให้ตั้งเป็น environment variable ก่อนเสมอ เพื่อกัน password หลุดเข้า `bash history` หรือถูกเห็นผ่าน `ps aux` ระหว่าง process กำลังรัน
>
> หลังใช้งานผ่าน CLI เสร็จ แนะนำให้ลบ history ที่เกี่ยวข้องด้วย (`history -d <line_number>` หรือ `unset MSSQL_SA_PASSWORD` ทันทีหลังใช้เสร็จ)

**ตั้งค่า credential ก่อนเริ่ม (ทำครั้งเดียวต่อ session):**
```bash
read -s -p "SA Password: " MSSQL_SA_PASSWORD
export MSSQL_SA_PASSWORD
```
*(`read -s` จะไม่แสดงตัวอักษรที่พิมพ์บนหน้าจอ และไม่ทิ้ง password ไว้ใน history)*

**Backup — สร้างไฟล์ .bak ใน Container ก่อน แล้วคัดลอกออกมาที่ Host:**
```bash
# Step 1: สร้าง backup file ภายใน container
sudo docker exec -it sql-server /opt/mssql-tools/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_SA_PASSWORD" \
  -Q "BACKUP DATABASE [<ชื่อฐานข้อมูล>] TO DISK = '/var/opt/mssql/data/<backup_db_name>.bak'"

# Step 2: คัดลอกไฟล์จาก Container -> Host
sudo docker cp sql-server:/var/opt/mssql/data/<backup_db_name>.bak ./<backup_db_name>.bak
```

**Restore — คัดลอกไฟล์จาก Host เข้า Container ก่อน แล้วจึงสั่ง restore:**
```bash
# Step 1: คัดลอกไฟล์จาก Host -> Container
sudo docker cp ./<backup_db_name>.bak sql-server:/var/opt/mssql/data/<backup_db_name>.bak

# Step 2: สั่ง restore (WITH REPLACE จะเขียนทับ database เดิม)
sudo docker exec -it sql-server /opt/mssql-tools/bin/sqlcmd \
  -S localhost -U SA -P "$MSSQL_SA_PASSWORD" \
  -Q "RESTORE DATABASE [<ชื่อฐานข้อมูล>] FROM DISK = '/var/opt/mssql/data/<backup_db_name>.bak' WITH REPLACE"
```

**เมื่อทำเสร็จแล้ว ล้าง credential ออกจาก session:**
```bash
unset MSSQL_SA_PASSWORD
```

---

## 4. Verification

- [ ] Backup file มีขนาด > 0 และใกล้เคียงกับ backup ครั้งก่อน (ไม่เล็กผิดปกติ)
- [ ] Restore test บน instance ทดสอบ (ทำเป็นระยะ ไม่ต้องทุกครั้ง)
- [ ] Log แสดงผล backup/restore สำเร็จ ไม่มี error

---

## 5. Retention & Cleanup

- Backup ที่สร้างก่อนทำ Rollback: เก็บอย่างน้อย 7 วันหลังยืนยันว่าระบบเสถียรแล้ว
- Backup ทั่วไปที่ไม่มี incident ผูกอยู่ และเก็บมานานกว่า 1 ปี: ลบได้
- **ก่อนลบทุกครั้ง ตรวจสอบว่าไม่มี incident/issue ใดอ้างอิงถึง backup นี้อยู่**

---

## 6. Failure Handling

**ถ้า backup fail จะทำยังไง:**
- [ ] เช็ค disk space ก่อน (สาเหตุที่พบบ่อยที่สุด)
- [ ] เช็ค permission/credential หมดอายุไหม
- [ ] Alert ทีมที่เกี่ยวข้องถ้า backup fail ติดกัน N ครั้ง

---

## 7. Change Log

| Date | By | What changed |
|---|---|---|
| 2026-07-31 | Arm-DevOps | Initial |