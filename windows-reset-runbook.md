# Runbook: Windows PC Reset (Remove All Files)

## Scope
ใช้สำหรับ reset เครื่อง Windows ก่อนส่งต่อ/ขาย/รีไซเคิล หรือแก้ปัญหาระบบร้ายแรง โดยลบไฟล์และแอปทั้งหมด

## Pre-checks
- [ ] สำรองข้อมูลสำคัญ (เอกสาร, รูป, ไฟล์งาน) ไปยัง external drive / cloud
- [ ] จด license key ของซอฟต์แวร์ที่ซื้อแยก (Office, etc.)
- [ ] Sign out จากบัญชีทั้งหมด: Microsoft account, OneDrive, Google, Adobe, Steam ฯลฯ
- [ ] เครื่องเสียบปลั๊กไฟ หรือแบตเตอรี่ > 50%
- [ ] มีสาย LAN/Wi-Fi พร้อมใช้ (บางขั้นตอนต้องดาวน์โหลดไฟล์)

## ขั้นตอน: Factory Reset (built-in, แนะนำสำหรับเครื่องทั่วไป)

1. เปิด **Settings** → **System** → **Recovery**
2. กด **Reset this PC**
3. เลือก **Remove everything**
4. เลือกวิธีลง Windows ใหม่:
   - **Cloud download** – ดาวน์โหลด Windows ใหม่จาก Microsoft (แนะนำถ้าไฟล์ระบบเสียหาย)
   - **Local reinstall** – ใช้ไฟล์ในเครื่อง (เร็วกว่า, ต้อง disk ปกติ)
5. เลือก **Change settings** → เปิด **Clean data** = **Yes** (ลบข้อมูลแบบเขียนทับ ป้องกันกู้คืน)
6. กด **Reset** แล้วรอ (30 นาที - 2 ชม. ขึ้นกับ disk)
7. ตั้งค่า Windows ใหม่ตาม out-of-box setup

> หมายเหตุ: วิธีนี้ลบทุกอย่างรวมถึง recovery partition เดิม เหมาะกับกรณีเปลี่ยนมือเครื่องหรือสงสัยมัลแวร์ฝังลึก

## Post-reset Checklist
- [ ] ติดตั้ง driver ที่จำเป็น (ผ่าน Windows Update หรือเว็บผู้ผลิต)
- [ ] ติดตั้ง Windows Update ล่าสุด
- [ ] ติดตั้งโปรแกรมที่จำเป็น
- [ ] กู้คืนไฟล์จาก backup (ถ้ามี)

## Rollback / ถ้าเกิดปัญหา
- หากติดตั้งไม่สำเร็จกลางคัน ให้บูตซ้ำจาก USB installer แล้วเริ่มใหม่ตั้งแต่ขั้นตอน Custom install
- หากลืม backup ก่อน reset และเลือก Local reinstall (ไม่ใช่ wipe drive) อาจกู้ไฟล์บางส่วนคืนได้ด้วยโปรแกรม data recovery ก่อนที่จะเขียนทับข้อมูลเพิ่ม
