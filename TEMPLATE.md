# Runbook: <ชื่อ symptom/alert>

> **หลักคิด runbook ที่ดี:** เขียนสำหรับ "ตัวเองตอนตี 3 ครึ่งหลับครึ่งตื่น" หรือ "เพื่อนร่วมทีมที่เพิ่งเข้ามา 2 สัปดาห์" — command ต้อง copy-paste รันได้เลย, ไม่มี "ควรจะรู้อยู่แล้ว"

---

## Metadata

- **Service:** ระบบที่เกี่ยวข้อง
- **Severity:** P0 / P1 / P2
- **Owner:** ทีม/คนที่ดูแล
- **Last verified:** YYYY-MM-DD (ต้อง re-verify ทุก 6 เดือน)
- **Related alerts:** ชื่อ alert rule ใน Grafana/Alertmanager

---

## 1. Symptom (อาการ)

อาการที่เห็นคืออะไร — ตรงกับ alert message หรือคำที่ user มักบ่น

**Example alert:**
```
HighVramUsage on gpu-01: usage=95% threshold=90%
```

---

## 2. Impact (ผลกระทบ)

- ใครโดน
- ระบบไหน degrade
- ต้อง escalate ทันทีไหม

---

## 3. Diagnosis (ตรวจอะไรก่อน)

ทำตามลำดับ **จากถูกที่สุด → แพงที่สุด**:

### 3.1 Quick check
```bash
# command จริงที่ copy-paste ได้
docker ps --filter "name=xxx"
```
**คาดว่าจะเห็น:** ...
**ถ้าเห็นแบบนี้แปลว่า:** ...

### 3.2 Deeper check
```bash
docker logs xxx --tail 100
```

### 3.3 Dashboard link
- Grafana: https://grafana.internal/d/xxx
- ดูที่ panel: "VRAM Usage per Container"

---

## 4. Mitigation (แก้ยังไงให้หายก่อน)

**ตัดสินใจก่อน:** restart ปลอดภัยไหม? มี user กำลังใช้อยู่ไหม?

### Option A: Quick restart (ถ้า service ยอมให้ downtime สั้นๆ)
```bash
docker compose restart <service>
```

### Option B: Graceful failover
```bash
# steps ...
```

### Option C: Escalate ถ้าทำ A/B ไม่ได้
- ติดต่อ: ...
- Info ที่ต้องเตรียม: log จาก step 3, timeline

---

## 5. Verification (ยืนยันว่าหายจริง)

- [ ] Alert clear ใน Grafana
- [ ] Health endpoint 200: `curl -f https://xxx/health`
- [ ] Sample request สำเร็จ

---

## 6. Root Cause & Long-term Fix

- **สาเหตุที่เกิดบ่อย:**
- **Permanent fix ที่ควรทำ:** link ไปที่ issue #xxx
- **Related ADR:** link ไปที่ `decisions/NNNN-xxx.md`

---

## 7. Change Log

| Date | By | What changed |
|---|---|---|
| YYYY-MM-DD | @arm | Initial |
