# ExamGuard Architecture — Final Pre-Coding Debate

## Objective

We are about to START CODING the ExamGuard prototype for a competitive hackathon (RTGS, Dept. of School Education, Govt. of Andhra Pradesh). **We need to win.** 11 teams are competing. The jury evaluates:

1. **Accuracy** — detect at least 8/10 staged malpractice events with minimal false positives
2. **Performance** — real-time processing on basic desktop CPUs (i5/Celeron, 4GB RAM) with 4G connectivity
3. **Innovation** — novel approaches that differentiate us from 10 other competing teams

This architecture has already been through 2 prior multi-LLM debates. The debates produced 15 consensus points and 6 convergence points. This is the FINAL debate before coding begins — focus on **implementation-critical issues** that could make or break our hackathon entry.

## Current Architecture (Post-Debate Consensus)

### Terminal Side (x30 desktops)
- **Headless Python agent** exclusively owns the webcam via `cv2.VideoCapture(0)`, serves MJPEG to browser
- 3-thread architecture: capture → inference → event push (bounded queue size 1, frame dropping)
- **Models (all INT8 ONNX, ~11MB total):**
  - BlazeFace (face detection + 6 keypoints for PnP, ~15ms/frame)
  - MOSSE tracker (<1ms between detections, re-verified by MobileFaceNet every 2s)
  - MobileFaceNet + ArcFace (face verification 512D embeddings, ~18ms/run, every 2s or track loss)
  - PnP Head Pose from BlazeFace keypoints via `cv2.solvePnP` (gaze, ~2ms/frame) — NO Face Mesh in P0/P1
  - Silero VAD (audio voice activity detection, 2MB ONNX, ~1ms/30ms chunk)
  - Material Heuristics: phone screen glow (bright rectangular region in lower frame) + skin-tone blob in desk ROI (~1-2ms, no model)
- CLAHE preprocessing on every frame before face detection (~2ms)
- Per-frame brightness tracking to prevent false CANDIDATE_ABSENT from poor lighting
- Browser UI served locally by agent: enrollment flow, 10 MCQ exam, timer, webcam status light
- NO Electron, NO PyQt6, NO MediaPipe (P0/P1), NO YOLO

### Edge Server (Mini-PC + SSD on LAN)
- FastAPI + SSE + vanilla JS/HTMX dashboard (no React, no Streamlit)
- SQLite WAL mode with single async writer queue (batch flush every 250ms or 50 events)
- JSONL per-terminal audit trail + hash-chained events (SHA-256)
- 3 fusion rules on edge:
  - GAZE_DEVIATION + FACE_MISMATCH within ±5s → CONFIRMED_IMPERSONATION
  - SUSPICIOUS_AUDIO + GAZE_DEVIATION within ±5s → POSSIBLE_CONSULTATION
  - SUSPICIOUS_POSTURE + GAZE_DEVIATION within ±5s → POSSIBLE_MATERIAL_USE
- Cross-terminal seat-swap: On FACE_MISMATCH, compare embedding against ALL 30 enrolled → CONFIRMED_SEAT_SWAP
- Common-mode audio CLASSIFICATION (not suppression): ≥3 terminals → ROOM_NOISE (logged); 2 adjacent → POSSIBLE_CONSULTATION; 1 → SUSPICIOUS_AUDIO
- CCTV: BackgroundSubtractorMOG2 + motion heatmap overlay (AI on edge, no face processing)
- App-level time sync: RTT/2 offset estimation, median of last 5, edge is time authority
- Ground truth logger + malpractice trigger app (mobile-friendly, with "skip to next" button)

### 7 Alert Types + 3 Fusion Rules

| Alert | Mode | Threshold | Confidence |
|-------|------|-----------|------------|
| CANDIDATE_ABSENT | Standalone | >5s no face (with brightness gate) | Very High |
| MULTIPLE_PERSONS | Standalone | >5 consecutive frames, 2+ faces | Very High |
| FACE_MISMATCH | Standalone | 3-zone hysteresis, >0.70 cosine sustained 3+ checks | Critical |
| GAZE_DEVIATION | Standalone + Fusion | >30s at +30°, >3s at +35°, any at +45° (PnP head pose) | Medium-High |
| SUSPICIOUS_AUDIO | Standalone + Fusion | >10s sustained speech (standalone), >5s + gaze (fusion) | High |
| SUSPICIOUS_POSTURE | Fusion-only | Pitch >30° down >10s + GAZE_DEVIATION | Medium |
| POSSIBLE_MATERIAL_USE | Heuristic standalone | Phone glow OR skin-tone blob in desk region | Medium-High |

### Expected Coverage
8/10 malpractice scenarios with <5% false positive rate. Weakest: subtle phone-under-desk (glow not visible) and brief whisper exchanges (<5s).

### Implementation Phases
- **Phase 0:** Pre-flight diagnostic (go/no-go gate — RAM, DLL, FPS)
- **Phase 1:** Core pipeline (agent + camera + MJPEG + enrollment + face verify + PnP gaze)
- **Phase 2:** All detections + 3-terminal vertical slice (hard milestone)
- **Phase 3:** Dashboard + CCTV + demo infrastructure + fusion rules
- **Phase 4:** Conditional P2 upgrades (only if P0+P1 pass 3-terminal test)
- **Phase 5:** Validation + demo prep + threshold tuning

## What We Need You to Debate

### 1. ACCURACY — Can We Reliably Hit 8/10?

- **Material detection relies ONLY on heuristics** (phone glow + skin-tone). No object detection model in P0/P1. If the jury stages "student uses phone under desk" or "student reads notes on lap" — can heuristics catch this? What percentage of phone/notes scenarios will we miss?
- **PnP head pose from 5-6 BlazeFace keypoints** gives coarse yaw/pitch, NOT iris-level gaze. Is the 30° threshold with 30s persistence enough? What about subtle sideways glances while head stays mostly forward?
- **Audio: 10s sustained speech for standalone** — this catches dictation or conversation but misses brief whispered exchanges. Is 5s+gaze fusion enough?
- **Enrollment has no liveness detection in P0** — could a photo taped to the screen fool BlazeFace + MobileFaceNet?
- **What specific malpractice events are we most likely to MISS?** Be concrete.
- **What's our realistic score: 7/10? 8/10? 9/10?**

### 2. PERFORMANCE — Will This Actually Run?

- Realistic RAM: Windows 10 (~1.5GB) + AV (~200MB) + Chrome (~500MB-1.5GB) + Python+ONNX (~150MB) + models (~30MB) + OpenCV (~80MB) = 2.46-3.46GB. On 4GB that's **550MB-1.5GB headroom**. Is this enough?
- Per-frame cost: CLAHE (2ms) + BlazeFace (15ms) + PnP (2ms) + material heuristics (2ms) = ~21ms. Plus MOSSE (<1ms). That's ~45fps theoretical. With MobileFaceNet every 2s (~18ms) and Silero VAD (~1ms/30ms chunk), can we sustain 12+ FPS on Celeron?
- **Bounded queue size 1 + frame dropping** — could we miss a 2-second malpractice event if inference is slow?
- **Windows-specific concerns:** DLL conflicts between ONNX Runtime and OpenCV? Antivirus blocking Python/ONNX? Webcam driver exclusivity?
- What optimizations should we add that we haven't considered?

### 3. INNOVATION — What Wins the Hackathon?

Current differentiators:
- Agent-as-camera-owner MJPEG architecture (novel for proctoring)
- Cross-terminal seat-swap via embedding comparison (demo "wow moment")
- Common-mode audio classification (technically sophisticated)
- Hash-chained tamper-proof audit trail
- Adaptive FPS degradation modes
- Live confusion matrix against ground truth

**Is this enough to beat 10 other teams?** What can we add that is:
- Implementable in the coding window
- Low engineering risk
- High demo/visual impact
- Genuinely differentiating

Specific questions:
- Should we add a **risk score per student** (weighted combination of all alerts) for the dashboard?
- Should we add **enrollment liveness** (blink detection via eye aspect ratio from BlazeFace keypoints)?
- Should we add a **post-exam summary report** with per-student risk timelines?
- Is there a **novel visualization** for the dashboard that would impress a government jury?
- Any **recent research (2024-2026)** we should be incorporating?

### 4. IMPLEMENTATION RISKS — What Could Derail Us?

- What's the single highest-risk component in the architecture?
- What's most likely to fail on demo day?
- Is the Phase 0 pre-flight diagnostic comprehensive enough?
- Should we change the phase ordering for hackathon optimization?
- What's the minimum viable demo that's already impressive?

### 5. CRITICAL GAPS — What Are We Missing?

- **Exam UI:** 10 MCQ interface — how critical is the UI quality? Should we invest time in making it polished?
- **Evidence presentation to jury:** How should evidence snapshots be displayed? Annotations? Side-by-side with ground truth?
- **Network resilience:** Store-and-forward is P2. Should it be P1?
- **Configuration management:** All thresholds in config.yaml — should we add a settings UI?
- **Anything else we haven't thought of?**

## Debate Rules

- This is a **hackathon**, not a production system. Optimize for demo impact and jury impression.
- Be specific and actionable. No generic advice.
- Consider that **10 other teams** are building similar systems. Most will use MediaPipe + YOLO + basic dashboards. How do we stand out?
- Challenge assumptions. If something in the architecture is wrong, say so.
- Suggest concrete improvements with estimated implementation time.
