# ExamGuard Architecture: Final Validation Before Implementation

## What This Debate Is About

We are about to start implementation of ExamGuard — an AI-enabled live exam proctoring system for the School Education Department, Government of Andhra Pradesh. This is a hackathon competition (11 shortlisted startups, top 3 win pilot contracts). We have 7-10 days to build a working demo.

The architecture below is the result of 2 prior debate rounds (Claude Opus + GPT-5.2 + Gemini 3 Pro) that reached FULL consensus. **This final debate asks: is this architecture battle-ready to WIN the hackathon?** Specifically:

1. Are there any blind spots that could cause demo-day failure?
2. Is the scope right — are we cutting too much or too little?
3. Is the implementation schedule realistic for a small team?
4. What's our biggest risk and do we have adequate mitigation?
5. How do we differentiate from the other 10 competitors who likely use similar MediaPipe/MobileFaceNet stacks?

---

## The Converged Architecture

### System Overview

```
┌───────────────────────────┐       ┌──────────────────────────────┐
│  Terminal Agent (x30)     │       │  Edge Server (Mini-PC + SSD) │
│  Single Python Process    │       │                              │
│                           │  LAN  │  FastAPI + SSE + vanilla JS  │
│  PyQt6 UI:                │──────→│                              │
│  • 10 MCQ exam interface  │ JSON  │  Persistence:                │
│  • Enrollment flow        │ episo │  • SQLite WAL (primary)      │
│  • Timer + navigation     │ des + │  • JSONL per-terminal (audit)│
│                           │ JPEG  │                              │
│  AI Pipeline (threaded):  │ snaps │  Logic:                      │
│  T1: Webcam capture       │       │  • 3 fusion rules            │
│  T2: Detect→Track→Verify  │       │  • Common-mode audio reject  │
│  T3: Event/evidence push  │       │  • Time sync authority       │
│                           │       │  • Episode aggregation       │
│  Models (INT8 ONNX):      │       │  • Evidence store            │
│  • BlazeFace (detect)     │       │                              │
│  • MOSSE/CSRT (track)     │       │  Dashboard (vanilla JS+SSE): │
│  • MobileFaceNet (verify) │       │  • Terminal grid (30 tiles)  │
│  • Face Mesh (gaze+pitch) │       │  • Episode alert feed        │
│  • Silero VAD (audio)     │       │  • Evidence viewer           │
│                           │       │  • Confusion matrix          │
│  NO Electron              │       │  • Ground truth logger       │
│  NO YOLO                  │       │  • Degradation status        │
│  NO raw video leaves      │       │                              │
│  Episode-based events     │       │  NO cloud dependency         │
└───────────────────────────┘       └──────────────────────────────┘
```

### 6 Standalone Alert Types

1. **CANDIDATE_ABSENT** — 0 faces detected >5 seconds
2. **MULTIPLE_PERSONS** — 2+ faces detected >5 consecutive frames
3. **FACE_MISMATCH** — 3-zone hysteresis (match <0.45, uncertain 0.45-0.70, mismatch >0.70 sustained 3+ checks)
4. **GAZE_DEVIATION** — Yaw >baseline+30° sustained >30s (standalone); >35° sustained >3s (high); >45° any duration (critical)
5. **SUSPICIOUS_AUDIO** — Sustained speech >10s via Silero VAD (standalone)
6. **SUSPICIOUS_POSTURE** — Head pitch >30° downward sustained >10s (phone-in-lap/notes proxy)

### 3 Fusion Rules (Edge Server)

1. GAZE + FACE_MISMATCH within ±5s → CONFIRMED_IMPERSONATION
2. AUDIO + GAZE within ±5s → POSSIBLE_CONSULTATION
3. POSTURE + GAZE within ±5s → POSSIBLE_MATERIAL_USE

### Key Design Decisions

- **PyQt6 single-process** agent (no Electron — saves 300-500MB RAM on 4GB machines)
- **Detect → Track → Verify** pipeline: BlazeFace 3-5fps → MOSSE/CSRT tracker → MobileFaceNet every 2s or on track loss
- **CLAHE preprocessing** on every frame (~2ms, handles uneven lighting)
- **Episode-based events** (not per-frame spam): EPISODE_START/EPISODE_END with stable episode_id
- **FPS-based degradation**: ≥12fps = full; ≥7fps = shed gaze; ≥4fps = shed audio; <4fps = SURVIVAL mode
- **Store-and-forward**: JSONL spool on terminal, replay on reconnect
- **Push-based evidence**: terminal pushes encrypted JPEG to edge on high/critical alerts
- **Ground truth live logger**: `/admin/ground-truth` with big buttons per malpractice type

### What Was Cut

- CCTV processing (high demo-failure risk)
- YOLOv8-nano object detection (COCO domain mismatch, high CPU, AGPL licensing)
- Hash-chain tamper-proof logs
- PDF report generation
- Bias metrics dashboard
- Person Re-ID
- Adaptive scheduling
- Message brokers (NATS/MQTT/Redis)
- React frontend (build-chain complexity)
- Streamlit (full-page rerenders)

### Implementation Schedule (10 Days)

| Day | Focus | Exit Criterion |
|-----|-------|----------------|
| Day 0 | GO/NO-GO: WinPython portable + webcam + all models + mic on non-dev machine | Deployment confirmed or pivot |
| Day 1 | Webcam + 3 threads + BlazeFace + CLAHE + MOSSE tracker + exam_content.json | Face boxes on live feed with tracking |
| Day 2 | Enrollment (5 embeddings, quality gates) + face verification (3-zone) + edge server start | Enroll → verify + mock events on edge |
| Day 3 | Multi-face + absence + gaze (fusion-only, 1-2fps). FPS degradation modes. **TARGET: Vertical slice** | Real camera → real inference → real events → edge → dashboard tile |
| Day 4 | **HARD DEADLINE vertical slice.** Silero VAD. Time sync. Store-and-forward. | Events flow terminal → edge with time sync |
| Day 5 | Dashboard (terminal grid, alert feed, detail view). PyQt6 exam UI. Exam start/stop broadcast. Ground truth logger. | **Minimum viable demo works end-to-end** |
| Day 6 | Evidence snapshot push. Common-mode audio rejection. Evidence viewer. Fusion rules. | Dashboard shows evidence + fusion alerts |
| Day 7 | Confusion matrix (1:1 greedy matching). Post-exam summary. Multi-terminal testing (3-5). | Scoring works end-to-end |
| Day 8 | Multi-terminal stress test (10+). Threshold tuning on real hardware. | Stable under load |
| Day 9 | Calibration exam: ALL planned malpractice scenarios. Tune thresholds. Record fallback video. | Thresholds tuned, fallback ready |
| Day 10 | Demo rehearsal only. Practice 6-10 scenarios. Prepare backups. | Ready for demo |

### Expected Malpractice Coverage

| Scenario | Detection Method | Confidence |
|----------|-----------------|------------|
| Candidate leaves seat | CANDIDATE_ABSENT | Very High |
| Extra person enters | MULTIPLE_PERSONS | Very High |
| Impersonation / seat swap | FACE_MISMATCH | High |
| Talking to neighbor | SUSPICIOUS_AUDIO | High |
| Looking at neighbor's screen | GAZE_DEVIATION | High |
| Phone under desk / in lap | SUSPICIOUS_POSTURE | Medium-High |
| Notes on desk / lap | SUSPICIOUS_POSTURE | Medium-High |
| Passed note from outside | GAZE_DEVIATION + MULTIPLE_PERSONS | Medium |

**Reliable catches: 7-8/10. Target: 8/10.**

### POC Success Criteria

- 30 candidates, 10-minute sessions, 6-10 staged malpractice events
- Must detect ≥8/10 events
- Timestamps must match ground-truth logs from human monitors
- Confusion matrix with TP/FP/TN/FN rates

---

## Questions for the Debate Panel

1. **Demo-Day Risk Assessment**: What is the single most likely cause of demo-day failure, and is our mitigation sufficient?
2. **Scope Calibration**: Given 7-10 days for a small team (1-2 developers), is this scope achievable? Should anything else be cut or added?
3. **Competitive Differentiation**: 10 other startups are competing. Most will use similar MediaPipe/MobileFaceNet stacks. What makes this architecture stand out?
4. **The 8/10 Target**: Our analysis says 7-8/10 reliable catches. Is that realistic? What specific failure modes could drop us below 8?
5. **PyQt6 vs Alternatives**: Is PyQt6 the right choice for the exam UI + AI pipeline, or would a browser-based approach (Flask/FastAPI serving a web UI) be simpler and more reliable?
6. **Audio Strategy**: Silero VAD with 10s standalone threshold — is this too conservative? Too aggressive? Should we drop audio entirely and focus on visual detection?
7. **Deployment**: WinPython portable vs PyInstaller vs Docker — which is most likely to work on unknown exam center machines?
8. **Missing Features**: Is there anything we're NOT doing that the jury will expect to see? Any low-hanging fruit we're overlooking?
