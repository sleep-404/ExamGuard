# ExamGuard Architecture — Final Pre-Implementation Review

## Objective
This is the FINAL architectural debate before we start coding. We must build a hackathon-winning AI-enabled live exam proctoring system. The jury evaluates: **accuracy (8/10 detections), performance (real-time on basic hardware), and innovation (wow factor).**

## The Full Architecture Plan

(See the detailed architecture below — debate every aspect for maximum hackathon competitiveness)

---

## Critical Questions for the Debate

### 1. ACCURACY — Can we reliably hit 8/10?

The current plan has 6 alert types + 3 fusion rules covering these scenarios:
- CANDIDATE_ABSENT (face gone >5s) — Very High confidence
- MULTIPLE_PERSONS (2+ faces >5 frames) — Very High confidence
- FACE_MISMATCH + SEAT_SWAP (3-zone hysteresis + cross-terminal) — High confidence
- GAZE_DEVIATION (sustained >30s at 30°+) — High confidence
- SUSPICIOUS_AUDIO (sustained speech >10s via Silero VAD) — High confidence
- SUSPICIOUS_POSTURE (pitch >30° >10s, fusion-only) — Medium-High

**Debate:** Are these thresholds correct? Are there blind spots? What malpractice scenarios could we miss? Is 8/10 realistic? Should we add any detection method we haven't considered?

### 2. PERFORMANCE — Will this run on Celeron/i5 with 4GB RAM?

Current plan: INT8 ONNX models (~13MB total), 640x480 capture, 3-thread architecture, detect-track-verify pipeline. Expected 12-15fps on i5, 8-10fps on Celeron. FPS degradation modes for graceful fallback.

**Debate:** Is the memory budget realistic (~430MB)? Will ONNX Runtime + MediaPipe + OpenCV actually fit? Is the 3-thread model correct or should we use multiprocessing? Any performance landmines we're not seeing?

### 3. INNOVATION — What makes us stand out?

Current differentiators:
- Cross-terminal seat-swap detection (edge compares all 30 enrollment embeddings)
- Episode-based event model with hash-chained tamper-proof audit trail
- Common-mode audio rejection (suppress room-wide noise from 30 microphones)
- FPS-based graceful degradation (FULL → SHED_GAZE → SHED_AUDIO → SURVIVAL)
- Confusion matrix against ground truth in real-time during demo
- Store-and-forward resilience for LAN failures
- Gaze baseline calibration per-student (median + MAD)

**Debate:** Is this enough innovation? What else could differentiate us? Are there any quick-win features (≤4 hours implementation) that would dramatically improve our demo? Any published research or techniques we should incorporate?

### 4. MODEL CHOICES — Are we using the best models?

Current: BlazeFace (detect), MOSSE/CSRT (track), MobileFaceNet+ArcFace (verify), Face Mesh/MediaPipe (gaze+pose), Silero VAD (audio).

**Debate:** Are there better INT8-quantizable alternatives for any of these? Should we use MediaPipe Face Detection instead of BlazeFace? Is MOSSE vs CSRT the right call? Any recent lightweight models (2024-2025) that outperform these?

### 5. DEMO STRATEGY — How do we maximize jury impact?

Current plan: Show detections in confidence order (absence → multi-face → seat-swap → fusion → confusion matrix). Ground truth logger + malpractice trigger app for controlled demonstrations.

**Debate:** Is this the right demo order? What would make the biggest impression on a government jury? Should we add visual elements to the dashboard? How do we handle the inevitable demo failure gracefully?

### 6. EDGE CASES & FAILURE MODES — What could go wrong?

Concerns: Government machine AV blocking ONNX, webcam driver issues, poor lighting in exam halls, students wearing glasses/masks, USB permissions, firewall blocking LAN communication.

**Debate:** What are the most likely failure modes on demo day? What contingencies should we prepare? Is the pre-flight diagnostic sufficient?

### 7. ARCHITECTURE GAPS — What's missing?

**Debate:** Review the full architecture plan critically. Is there anything important that's missing, over-engineered, or under-specified? Any changes needed for maximum accuracy/performance/innovation?

---

## Full Architecture Plan

(Included below for reference — this is the complete architecture-plan.md)

### System Architecture

```
Terminal (x30): Headless Python Agent (background process)
  - 3 threads: webcam capture, detect→track→verify inference, event/evidence push
  - Models (INT8 ONNX): BlazeFace, MOSSE/CSRT, MobileFaceNet, Face Mesh, Silero VAD
  - Local HTTP/WS API serving browser UI (HTML/JS)
  - NO Electron, NO PyQt6, NO YOLO
  - Episode-based events, 640x480 capture, store-and-forward

Edge Server (Mini-PC + SSD):
  - FastAPI + SSE + vanilla JS dashboard
  - SQLite WAL (primary) + JSONL per-terminal (audit) + hash-chained events
  - 3 fusion rules, cross-terminal seat-swap, common-mode audio rejection
  - Time sync authority, episode aggregation, evidence store
  - Dashboard: 30-tile grid, alert feed, evidence viewer, confusion matrix
  - Ground truth logger + malpractice trigger app (mobile-friendly)
  - CCTV display stub (RTSP/USB + motion overlay, no AI)
```

### Enrollment Flow
- Student enters ID → Privacy notice (Telugu + English) → 5 face captures (guided poses) → 5 embeddings via MobileFaceNet → Gaze baseline calibration (10s median+MAD) → Push to edge → Ready
- Quality gates: confidence >0.9, face >80px, blur/brightness checks, 3 failures → Force Proceed → DEGRADED

### Processing Pipeline
- CLAHE preprocessing → BlazeFace detect (3-5fps) → MOSSE/CSRT track → MobileFaceNet verify (every 2s or track loss)
- 3-zone face verification hysteresis: <0.45 match, 0.45-0.70 uncertain, >0.70 sustained 3+ → FACE_MISMATCH
- Gaze: Face Mesh 478 landmarks, yaw thresholds at 30°/35°/45° relative to calibrated baseline
- Audio: Silero VAD, standalone at >10s sustained speech, fusion at >5s + gaze

### Alert Types
- CANDIDATE_ABSENT (critical), MULTIPLE_PERSONS (high), FACE_MISMATCH (critical)
- GAZE_DEVIATION (medium/high/critical based on angle+duration), SUSPICIOUS_AUDIO (high)
- SUSPICIOUS_POSTURE (fusion-only with gaze)
- 3 fusion rules: CONFIRMED_IMPERSONATION, POSSIBLE_CONSULTATION, POSSIBLE_MATERIAL_USE

### Cross-Terminal Seat-Swap (Edge)
All 30 enrollment embeddings on edge. On FACE_MISMATCH, compare against ALL enrolled → CONFIRMED_SEAT_SWAP with both terminal IDs.

### Common-Mode Audio Rejection (Edge)
Buffer VAD events 500ms. If ≥3 terminals trigger → suppress all but highest-amplitude.

### FPS Degradation
≥12 FULL | ≥7 SHED_GAZE | ≥4 SHED_AUDIO | <4 SURVIVAL (presence + multi-face only)

### Deployment
Plan A: WinPython portable on USB. Plan B: PyInstaller --onedir. VC++ Redist + DirectX on USB.
Memory budget: ~430MB on 4GB terminal.

### Implementation Phases
Phase 0: Pre-flight (go/no-go). Phase 1: Core pipeline. Phase 2: Detection + 3-terminal vertical slice.
Phase 3: Dashboard + exam UI. Phase 4: Advanced features. Phase 5: Validation + demo prep.
Milestone gate: If Phase 0-2 not done by mid-sprint, cut ALL Phase 4.

### Demo Choreography
1. CANDIDATE_ABSENT (dramatic, reliable)
2. MULTIPLE_PERSONS (visually obvious)
3. FACE_MISMATCH + SEAT_SWAP (wow moment)
4. GAZE + AUDIO fusion (sophisticated)
5. Confusion matrix live (quantitative proof)
