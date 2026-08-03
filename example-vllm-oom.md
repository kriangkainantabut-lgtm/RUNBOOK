# Runbook: faster-whisper container crash / VRAM over-commit

## Metadata
- **Service:** nt-law-advisor / STT (faster-whisper)
- **Severity:** P1 (voice pipeline down = user เรียกใช้งานไม่ได้)
- **Owner:** @arm
- **Last verified:** 2026-07-13
- **Related alerts:** `GPUVramHigh`, `ContainerRestartLoop{service="stt"}`

## 1. Symptom
- STT container restart loop
- Log: `CUDA out of memory. Tried to allocate ... MiB`
- vLLM ยัง response ได้แต่ latency สูงขึ้น

## 2. Impact
- Voice input จาก user ล่ม → ระบบใช้งานไม่ได้ทั้ง flow
- GPU H100L-94C over-committed จาก vLLM + OCR + TTS + STT รวมกัน

## 3. Diagnosis

### 3.1 ดู VRAM ปัจจุบัน
```bash
nvidia-smi --query-gpu=memory.used,memory.free,memory.total --format=csv
```
**คาดว่าเห็น:** used > 90% ของ 94 GB

### 3.2 ดูว่า container ไหนกิน VRAM มากสุด
```bash
nvidia-smi --query-compute-apps=pid,process_name,used_memory --format=csv
docker ps --format "table {{.Names}}\t{{.ID}}" | head
# match PID กับ container
```

### 3.3 ดู log STT
```bash
docker compose -f /opt/nt-law-advisor/docker-compose.yml logs stt --tail 200 | grep -i "cuda\|memory"
```

## 4. Mitigation

### Option A: Restart STT อย่างเดียว (ถ้า VRAM แค่แน่นชั่วคราว)
```bash
cd /opt/nt-law-advisor
docker compose restart stt
```

### Option B: ลด vLLM `gpu_memory_utilization` ชั่วคราว
แก้ใน `docker-compose.yml`:
```yaml
vllm:
  command: >
    --gpu-memory-utilization 0.60   # ลดจาก 0.75
```
```bash
docker compose up -d vllm stt
```

### Option C: จำกัด VRAM ต่อ container ด้วย MIG/MPS
- ยังไม่ implement — ดู ADR-0003 (draft)

## 5. Verification
- [ ] `nvidia-smi` → memory.free > 5 GB
- [ ] `curl http://localhost:8XXX/health` STT ตอบ 200
- [ ] Test STT ด้วย sample WAV: `curl -F file=@test.wav http://localhost:8XXX/transcribe`

## 6. Root Cause & Long-term Fix
- **Root cause:** ไม่มี resource governance ระหว่าง service — vLLM warmup กิน VRAM แบบ elastic ชน STT
- **Permanent fix:** issue #XXX — implement MIG partitioning หรือ move STT ไป GPU ตัวอื่น
- **Related ADR:** `decisions/0003-gpu-resource-partitioning.md` (draft)

## 7. Change Log
| Date | By | What changed |
|---|---|---|
| 2026-07-13 | @arm | Initial from real incident |
