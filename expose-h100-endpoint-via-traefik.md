# Runbook: Expose H100 AI Service Endpoint via Traefik (Network Access)

> **หลักคิด runbook ที่ดี:** เขียนสำหรับ "ตัวเองตอนตี 3 ครึ่งหลับครึ่งตื่น" หรือ "เพื่อนร่วมทีมที่เพิ่งเข้ามา 2 สัปดาห์" — command ต้อง copy-paste รันได้เลย, ไม่มี "ควรจะรู้อยู่แล้ว"

---

## Metadata

- **Service:** AI Chatbot Endpoint on H100 (NT-Production)
- **Severity:** P0
- **Owner:** DevOps / Infrastructure
- **Last verified:** 2026-07-13
- **Related alerts:** No alert rules

---

## 1. Symptom

ต้องการเปิดใช้งาน AI Service บนเครื่อง H100 (NT-Production) ให้เข้าถึงได้จากภายนอกผ่าน domain สำหรับทดสอบกับลูกค้า แต่ endpoint ยังเข้าไม่ได้ผ่าน Public IP/Domain (ติด Firewall/NAT/Traefik routing)

> หมายเหตุ: เดิม endpoint เดียวกันนี้เคยรันบน L40S ที่ตอบสนองช้ากว่า จึงย้าย workload มา H100 แล้ว (ดู `runbooks/gpu-workload-migration-l40s-to-h100.md` สำหรับขั้นตอนย้าย compute) — runbook นี้ครอบคลุม**เฉพาะการเปิด network access ให้เข้าถึง endpoint บน H100 ได้เท่านั้น**

---

## 2. Impact

- Endpoint ของ AI Service เข้าถึงไม่ได้จากภายนอก ทำให้ demo/นำเสนอลูกค้าติดขัด ไม่ smooth
- ต้องแก้ไขทันที เพราะไม่มีแผนสำรอง (fallback) รองรับกรณีนี้ในตอนนั้น

**Follow-up (ไม่ใช่ impact เฉพาะหน้า แต่ต้องทำต่อ):**
- ต้องกลับมาทำ Security Hardening เพิ่มเติมฝั่ง H100 หลังเปิด access แบบเร่งด่วนนี้ — ดู Section 6

---

## 3. Diagnosis

### 3.1 Service Health Check
- หากมี Dozzle ให้ดูการทำงานของ Services ว่ามี Error หรือไม่
- ตรวจสอบผ่าน SSH และรัน command:
```bash
sudo docker ps -a
```

### 3.2 Quick check จากเครื่องที่ต้องการเรียกใช้ API
```bash
# แทนที่ <H100_ENDPOINT_IP> และ <PORT> ด้วยค่าจริง (ดูใน internal wiki/vault ไม่ hardcode ใน runbook สาธารณะ)
curl http://<H100_ENDPOINT_IP>:<PORT>
telnet <H100_ENDPOINT_IP> <PORT>
```

**ถ้าเชื่อมต่อได้ (ปกติ):**
```
Trying <H100_ENDPOINT_IP>...
Connected to <H100_ENDPOINT_IP>.
Escape character is '^]'.
```
→ แปลว่าเชื่อมต่อผ่าน Public IP ได้แล้ว ปัญหาน่าจะอยู่ที่ layer อื่น (เช่น Traefik routing, DNS) ไปดู Section 4 Option A

**ถ้าเชื่อมต่อไม่ได้ (ค้างที่ "Trying..."):**
```
Trying <H100_ENDPOINT_IP>...
[ค้างไม่ต่อ]
```
→ ต้องเช็คเพิ่มหลายชั้น: Firewall OS (ufw/iptables), NAT ที่ Cloud Provider, Firewall ระดับ Network ไปดู Section 4 Option B

---

## 4. Mitigation

### Option A: Traefik Rule Configuration
ใช้เมื่อ: เชื่อมต่อ IP/port ได้แล้ว แต่ domain routing ยังไม่ทำงาน

1. เข้าไปที่เครื่อง Reverse Proxy
2. ตรวจสอบ `traefik.log` ว่าไม่มี Error ก่อนแก้ config
3. แก้ Traefik dynamic config ตามตัวอย่าง:
```yaml
http:
  routers:
    chat-ailegal-rtr:
      rule: "Host(`<domain-name>`)"
      entryPoints:
        - websecure
      # middlewares:
      #   - middlewares-cors@file
      service: chat-ailegal-svc

  services:
    chat-ailegal-svc:
      loadBalancer:
        servers:
          - url: "http://<internal-service-ip>:<port>"
```

> **⚠️ ห้าม restart Traefik** — Traefik อ่าน dynamic config ใหม่อัตโนมัติ (file watcher) การ restart อาจกระทบ router อื่นที่ใช้งานอยู่พร้อมกัน

### Option B: Firewall Configuration ที่ Cloud Provider
ใช้เมื่อ: telnet/curl ไปยัง Public IP ยังไม่ผ่านเลย

1. เข้าตรวจสอบผ่านเว็บ Cloud Provider Platform
2. ตรวจสอบ NAT rule
3. ตรวจสอบ Firewall rule
4. Allowlist IP/port ที่ต้องการใน NAT หรือ Firewall

---

## 5. Verification

- [ ] Sample request สำเร็จ (curl, telnet ผ่าน domain)
- [ ] ใช้งานได้ผ่าน AI Chatbot Platform จริง
- [ ] บันทึกไว้ว่า access นี้เปิดแบบเร่งด่วน — ต้องตามด้วย Security Hardening (ดู Section 6)

---

## 6. Root Cause & Long-term Fix

- **สาเหตุ:** Firewall/NAT rule เดิม whitelist เฉพาะ internal IP range เท่านั้น ทำให้ public IP ของ client ภายนอก (ลูกค้า) เข้าไม่ถึง endpoint บน H100
- **สิ่งที่ทำแบบเร่งด่วน (ต้อง revisit):** เปิด allowlist กว้างกว่าปกติเพื่อให้ทันเวลา demo — ยังไม่ได้ทำ hardening ที่เหมาะสม (เช่น จำกัด allowlist เฉพาะ IP ลูกค้าจริง, เพิ่ม auth layer หน้า Traefik)
- **Permanent fix ที่ควรทำ:**
  - [ ] จำกัด NAT/Firewall allowlist ให้เฉพาะ IP ที่จำเป็นจริง แทนการเปิดกว้าง
  - [ ] เพิ่ม middleware สำหรับ auth/rate-limit ที่ Traefik ก่อน production ใช้งานจริง
  - [ ] พิจารณาทำ VPN/WireGuard แทนการเปิด public endpoint ตรงๆ ถ้า security requirement สูง
- **Related ADR:** link ไปที่ `decisions/NNNN-h100-endpoint-exposure.md` (ยังไม่ได้เขียน — แนะนำเขียนเพราะเป็นการตัดสินใจ trade-off ระหว่างความเร็ว vs security)

---

## 7. Change Log

| Date | By | What changed |
|---|---|---|
| 2026-07-13 | @arm | Initial — split from GPU failover runbook, scoped to network/Traefik access only, redacted real IPs |