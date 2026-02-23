# ExamGuard: Complete Technical Architecture & Feature Plan

## Context

This is a hackathon entry for RTGS (Real Time Governance Society) — building an AI-enabled live exam proctoring system for the Dept. of School Education, Govt. of Andhra Pradesh. Performance, accuracy, and features win. This plan covers everything from pre-exam configuration to real-time detection to post-exam evidence.

**Constraints:** 30 terminals, 10-min exams, 8/10 malpractice detection, 4G connectivity, ~₹30K/center budget, basic desktop CPUs (i5/Celeron), open-source only.

---

## Phase 0: Pre-Exam Configuration (Before Any Student Arrives)

### 0.1 Center Setup (One-Time, per exam center)

1. **Install ExamGuard Agent** on each terminal (Python app or Electron app — NOT browser-only)
   - Downloads all AI models locally (~25MB total)
   - Native app ensures webcam data cannot be intercepted by browser extensions
   - Includes: MediaPipe BlazeFace (3MB), MobileFaceNet (4MB), MediaPipe Face Mesh (included with MediaPipe), YOLOv8-nano (6MB), WebRTC VAD (<1MB), ONNX Runtime

2. **Connect Edge Server** to the LAN
   - Raspberry Pi 4 (8GB) or mini-PC
   - Runs the aggregation service, dashboard backend, and tamper-proof log store
   - Connected to all terminals via Ethernet switch

3. **Mount & Configure CCTV**
   - Back-facing 1080p camera at rear of room
   - Connected to edge server via LAN (RTSP stream)
   - Edge server extracts 1 frame/sec and forwards to cloud (or processes locally if GPU available)

4. **Network Test**
   - Verify 4G uplink to cloud dashboard (if using cloud for CCTV processing)
   - Fallback: if no 4G, CCTV frames are queued locally on edge server

### 0.2 Exam Configuration (Before Each Exam Session)

1. **Create Exam Session** via dashboard
   - Exam ID, duration, question set, number of terminals
   - Assign terminal IDs (T-01 to T-40)

2. **Terminal Calibration**
   - Each terminal runs a quick check: webcam works, microphone works, AI models load successfully
   - Captures ambient noise baseline (for audio VAD thresholds)
   - Captures ambient light baseline (for face detection thresholds)

---

## Phase 1: Student Registration / Enrollment (5 min before exam)

### Flow:
```
Student arrives → Sits at assigned terminal → Registration screen appears
                                                    │
                                              ┌─────▼──────┐
                                              │ Enter ID /  │
                                              │ Hall Ticket │
                                              └─────┬──────┘
                                                    │
                                              ┌─────▼──────┐
                                              │ Privacy     │
                                              │ Notice      │
                                              │ (Telugu +   │
                                              │  English)   │
                                              └─────┬──────┘
                                                    │
                                              ┌─────▼──────────┐
                                              │ Face Capture    │
                                              │ (5 frames,      │
                                              │  guided UI)     │
                                              │                 │
                                              │ "Look straight" │
                                              │ "Turn left"     │
                                              │ "Turn right"    │
                                              │ "Look up"       │
                                              │ "Look down"     │
                                              └─────┬──────────┘
                                                    │
                                              ┌─────▼──────────┐
                                              │ Compute face   │
                                              │ embedding      │
                                              │ (MobileFaceNet)│
                                              │ Store in RAM   │
                                              └─────┬──────────┘
                                                    │
                                              ┌─────▼──────────┐
                                              │ Gaze           │
                                              │ Calibration    │
                                              │ "Look at the   │
                                              │  dot on screen"│
                                              │ (establishes   │
                                              │  baseline)     │
                                              └─────┬──────────┘
                                                    │
                                              ┌─────▼──────┐
                                              │ Ready!      │
                                              │ Wait for    │
                                              │ exam start  │
                                              └─────────────┘
```

### What gets captured during enrollment:
- **5 face images** from different angles → compute median face embedding (512D vector)
- **Baseline head pose** → mean yaw/pitch/roll during "look straight" phase
- **Ambient audio baseline** → background noise level for VAD threshold calibration
- **Terminal-to-student mapping** → sent to edge server as metadata

### What gets sent to edge server:
```json
{
  "event": "ENROLLMENT_COMPLETE",
  "terminal_id": "T-15",
  "student_id": "STU-042",
  "timestamp": "2026-03-15T10:00:00Z",
  "enrollment_quality": 0.95,
  "baseline_yaw": 2.3,
  "baseline_pitch": -1.5
}
```
NO face images or embeddings are sent. Only metadata.

---

## Phase 2: Real-Time Proctoring (During the 10-min Exam)

### 2.1 The Processing Pipeline (Per Terminal)

Every terminal runs this pipeline continuously:

```
Webcam (30fps) ─────────────────────────────────────────────────────────────
    │
    ├── EVERY FRAME (15fps effective):
    │   └── Face Detection (MediaPipe BlazeFace, ~15ms)
    │       ├── 0 faces → ALERT: CANDIDATE_ABSENT (critical)
    │       ├── 1 face → Continue to verification
    │       └── 2+ faces → ALERT: MULTIPLE_PERSONS (high)
    │
    ├── EVERY 2nd FRAME (~7.5fps):
    │   └── Head Pose + Gaze + Posture (MediaPipe Face Mesh, ~20ms)
    │       ├── Yaw > baseline+30° sustained > 30s → ALERT: GAZE_DEVIATION (medium, standalone)
    │       ├── Yaw > baseline+35° sustained > 3s → ALERT: GAZE_DEVIATION (high, standalone)
    │       ├── Yaw > baseline+45° any duration → ALERT: GAZE_DEVIATION (critical, standalone)
    │       └── Pitch > 30° downward sustained > 10s → ALERT: SUSPICIOUS_POSTURE (high, standalone)
    │           (proxy for phone-in-lap or notes-on-desk — no object detection needed)
    │
    ├── EVERY 5th FRAME (~3fps):
    │   └── Face Verification (MobileFaceNet, ~18ms)
    │       ├── Distance > 0.6 → ALERT: FACE_MISMATCH (critical)
    │       └── Distance < 0.4 → Confirmed match (log)
    │
    ├── EVERY 10th FRAME (~1.5fps):
    │   └── Object Detection (YOLOv8-nano, ~80ms)
    │       ├── Phone detected, confidence > 0.5, > 3 frames
    │       │   → ALERT: UNAUTHORIZED_OBJECT (high)
    │       └── Book/notes detected, confidence > 0.5, > 3 frames
    │           → ALERT: UNAUTHORIZED_OBJECT (high)
    │
    └── CONTINUOUS (audio stream, 20ms chunks):
        └── Audio VAD (Silero VAD, ~1ms per 30ms chunk)
            ├── Sustained speech > 10s → ALERT: SUSPICIOUS_AUDIO (high, standalone)
            └── Speech > 5s + concurrent GAZE_DEVIATION → POSSIBLE_CONSULTATION (fusion)

Microphone (16kHz) ─────────────────────────────────────────────────────────
```

### 2.2 Adaptive Scheduling

When risk is LOW (no alerts in last 30s):
- Face detection: every frame
- Gaze: every 4th frame
- Face verify: every 8th frame
- Object detect: every 15th frame
- **CPU usage: ~40%**

When risk is MEDIUM (alerts in last 30s):
- Face detection: every frame
- Gaze: every 2nd frame
- Face verify: every 5th frame
- Object detect: every 10th frame
- **CPU usage: ~60%**

When risk is HIGH (critical alert in last 10s):
- ALL models: every frame or every 2nd frame
- Evidence snapshot captured
- **CPU usage: ~85%**

### 2.3 CPU Budget Analysis (Per Frame @ 15fps = 66ms budget)

| Model | Frequency | Time per run | Amortized per frame |
|-------|-----------|-------------|-------------------|
| BlazeFace | every frame | 15ms | 15ms |
| Face Mesh (gaze) | every 2 frames | 20ms | 10ms |
| MobileFaceNet (verify) | every 5 frames | 18ms | 3.6ms |
| YOLOv8-nano (objects) | every 10 frames | 80ms | 8ms |
| WebRTC VAD | continuous | <1ms | <1ms |
| **TOTAL** | | | **~37.6ms** |

**Budget remaining: 28.4ms** for preprocessing, postprocessing, event logging, and network communication. This fits within the 66ms budget with headroom.

**On Celeron (2x slower):** Amortized total ~75ms. Exceeds 66ms budget. Solution: reduce to 10fps effective rate (100ms budget) or skip object detection on Celeron.

### 2.4 CCTV Processing Pipeline (Edge Server or Cloud)

```
CCTV Camera (RTSP, 15fps) ──→ Edge Server
                                   │
                              Frame Extraction
                              (1 frame per second)
                                   │
                              ┌────▼────────────────────┐
                              │ Person Detection         │
                              │ (YOLOv8-medium on GPU    │
                              │  or YOLOv8-nano on CPU)  │
                              │                          │
                              │ Output: bounding boxes   │
                              │ for each person in room  │
                              └────┬────────────────────┘
                                   │
                              ┌────▼────────────────────┐
                              │ Seat Assignment Mapping  │
                              │                          │
                              │ At exam start:           │
                              │ Person 1 → Zone A (T-01) │
                              │ Person 2 → Zone B (T-02) │
                              │ ...                      │
                              │                          │
                              │ During exam:             │
                              │ Track person movement    │
                              │ between zones            │
                              └────┬────────────────────┘
                                   │
                              ┌────▼────────────────────┐
                              │ Movement Alerts          │
                              │                          │
                              │ Person left Zone A       │
                              │ → ALERT: SEAT_DEPARTURE  │
                              │                          │
                              │ Person moved A → B       │
                              │ → ALERT: SEAT_SWAP       │
                              │                          │
                              │ New person entered room  │
                              │ → ALERT: UNAUTHORIZED    │
                              │   ENTRY                  │
                              │                          │
                              │ Person count changed     │
                              │ → ALERT: COUNT_MISMATCH  │
                              └─────────────────────────┘
```

### 2.5 Data Flow Summary

```
┌─────────────────┐          ┌──────────────────┐         ┌──────────────┐
│ Terminal (x30)   │   LAN   │ Edge Server      │   4G    │ Cloud/State  │
│                  │────────→│                  │────────→│ Dashboard    │
│ Webcam AI:       │ JSON    │ Aggregation:     │ JSON +  │              │
│ • Face detect    │ events  │ • Correlate      │ CCTV    │ • Live view  │
│ • Face verify    │ only    │   terminal +     │ frames  │ • Alert feed │
│ • Gaze track     │         │   CCTV alerts    │         │ • Evidence   │
│ • Object detect  │         │ • Hash-chain log │         │ • Confusion  │
│ • Audio VAD      │         │ • CCTV process   │         │   matrix     │
│                  │         │ • Evidence store  │         │              │
│ NO raw video     │         │                  │         │              │
│ leaves terminal  │         │ Evidence snapshots│         │              │
│                  │         │ stored locally    │         │              │
└─────────────────┘         └──────────────────┘         └──────────────┘
```

---

## Phase 3: Alert Types & Severity Matrix

### Complete Alert Taxonomy

| Alert Type | Trigger | Severity | Persistence Required | Standalone? | False Positive Mitigation |
|-----------|---------|----------|---------------------|-------------|--------------------------|
| `CANDIDATE_ABSENT` | 0 faces detected | Critical | >5 seconds | Yes | Ignore brief frame drops, occlusions |
| `FACE_MISMATCH` | Face embedding distance > 0.70 sustained | Critical | >3 consecutive checks | Yes | 3-zone hysteresis (match <0.45, uncertain 0.45-0.70, mismatch >0.70); compare against closest of 5 enrollment embeddings |
| `MULTIPLE_PERSONS` | >1 face detected at terminal | High | >5 consecutive frames | Yes | Ignore reflections (check face size ratios); ignore faces at screen edges |
| `GAZE_DEVIATION` | Yaw > baseline+30° sustained | Medium | >30 seconds | **Yes (standalone)** | Baseline-adjusted per student; 30s threshold eliminates natural looking-around |
| `GAZE_DEVIATION` | Yaw > baseline+35° sustained | High | >3 seconds | **Yes (standalone)** | Only flag when clearly looking at neighbor's screen direction |
| `GAZE_DEVIATION` | Yaw > baseline+45° | Critical | Any duration | **Yes (standalone)** | Extreme angle — unambiguous |
| `SUSPICIOUS_POSTURE` | Head pitch >30° downward sustained | High | >10 seconds | **Yes (standalone, NEW)** | Proxy for phone-in-lap or notes-on-desk; baseline-adjusted; 10s threshold filters natural head dips |
| `SUSPICIOUS_AUDIO` | Sustained speech detected | High | >10 seconds | **Yes (standalone)** | Silero VAD with common-mode rejection (3+ terminals in 500ms → suppress as ambient); 10s threshold = very likely conversation |

### Multi-Modal Fusion Rules

Standalone alerts fire independently. Fusion rules fire at **lower thresholds** when signals corroborate each other:

| Combination | Result | Confidence |
|------------|--------|------------|
| GAZE_DEVIATION (>5s) + FACE_MISMATCH within ±5s | CONFIRMED_IMPERSONATION | Very High |
| SUSPICIOUS_AUDIO (>5s) + GAZE_DEVIATION (>5s) within ±5s | POSSIBLE_CONSULTATION | High |
| SUSPICIOUS_POSTURE (>5s) + GAZE_DEVIATION within ±5s | POSSIBLE_MATERIAL_USE | High |
| GAZE_DEVIATION + MULTIPLE_PERSONS | LIKELY_COLLABORATION | High |

**Key design:** Gaze and audio have two threshold tiers — a conservative standalone threshold (long duration) and a shorter fusion threshold (when corroborated by another signal). This means they catch obvious standalone cheating AND boost detection of subtle multi-signal cheating.

---

## Phase 4: Evidence & Logging

### 4.1 Event Log Format (Hash-Chained)

```json
{
  "event_id": "evt_00147",
  "session_id": "EXAM-2026-03-15-HALL-A",
  "terminal_id": "T-15",
  "student_id": "STU-042",
  "timestamp": "2026-03-15T10:05:32.123Z",
  "event_type": "FACE_MISMATCH",
  "severity": "critical",
  "payload": {
    "confidence": 0.92,
    "embedding_distance": 0.73,
    "threshold": 0.60,
    "face_count": 1
  },
  "corroborating_events": ["evt_00145"],
  "previous_hash": "a3f2b8c9d1e4f7...",
  "event_hash": "d7e1f4a2b5c8e3..."
}
```

### 4.2 Evidence Snapshot (Triggered on High/Critical Alerts)

When a high or critical alert fires:
1. Capture the current webcam frame
2. Draw bounding boxes on detected faces / objects
3. Annotate with alert type, timestamp, confidence
4. Store locally on the terminal (encrypted)
5. Index in the event log (reference to file path, not the image data)

Evidence snapshots are NEVER transmitted over the network. They stay on the terminal. The dashboard can request a specific snapshot via authenticated API call to the terminal.

### 4.3 Confusion Matrix Generation

At the end of each exam session, auto-generate:
```
                    Predicted Positive    Predicted Negative
Actual Positive     True Positive (TP)    False Negative (FN)
Actual Negative     False Positive (FP)   True Negative (TN)

Precision = TP / (TP + FP)
Recall    = TP / (TP + FN)
F1 Score  = 2 * (Precision * Recall) / (Precision + Recall)
```

For the POC demo, the jury provides ground truth. The system compares its alerts (with timestamps) against ground truth to compute the matrix.

---

## Phase 5: Dashboard (Monitoring Interface)

### 5.1 Dashboard Views

**View 1: Room Overview (Default)**
```
┌──────────────────────────────────────────────────────────────┐
│  ExamGuard Dashboard — EXAM-2026-03-15-HALL-A    [LIVE]      │
│                                                               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐          │
│  │ T-01│ │ T-02│ │ T-03│ │ T-04│ │ T-05│ │ T-06│          │
│  │ 🟢  │ │ 🟢  │ │ 🔴  │ │ 🟢  │ │ 🟡  │ │ 🟢  │          │
│  │ OK  │ │ OK  │ │ALERT│ │ OK  │ │ GAZE│ │ OK  │          │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘          │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐          │
│  │ T-07│ │ T-08│ │ T-09│ │ T-10│ │ T-11│ │ T-12│          │
│  │ 🟢  │ │ 🟢  │ │ 🟢  │ │ ⚫  │ │ 🟢  │ │ 🟢  │          │
│  │ OK  │ │ OK  │ │ OK  │ │EMPTY│ │ OK  │ │ OK  │          │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘          │
│  ...                                                         │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ CCTV Live Feed (back-facing, 1fps refresh)               ││
│  │ [person boxes overlaid on room view]                     ││
│  └──────────────────────────────────────────────────────────┘│
│                                                               │
│  Alert Feed (last 60 seconds):                                │
│  🔴 10:05:32 T-03 FACE_MISMATCH (conf: 0.92) — CRITICAL     │
│  🟡 10:05:28 T-05 GAZE_DEVIATION (yaw: 32°, 4.1s) — MEDIUM  │
│  🟡 10:04:15 T-10 CANDIDATE_ABSENT (8s) — CRITICAL           │
│                                                               │
│  Stats: 30 terminals | 28 active | 1 alert | 1 absent       │
│  Detection FPS: avg 14.2 | Alerts today: 7 | FP rate: 3%    │
└──────────────────────────────────────────────────────────────┘
```

**View 2: Terminal Detail (Click on a terminal)**
- Live alert timeline for that terminal
- Risk score graph over time
- All events logged for that student
- Evidence snapshots (loaded on-demand from terminal)
- Head pose / gaze visualization

**View 3: Exam Summary (Post-exam)**
- Confusion matrix
- All flagged events with timestamps
- Evidence package for each flagged event
- Per-student risk score summary
- Export to PDF for documentation

### 5.2 Tech Stack for Dashboard

| Component | Technology | Why |
|-----------|-----------|-----|
| Backend | FastAPI (Python) | Same language as AI models; async WebSocket support |
| Frontend | React + Tailwind | Fast development; responsive; real-time updates via WebSocket |
| Real-time | WebSocket | Bidirectional; low latency for alerts |
| Database | SQLite (edge) + PostgreSQL (cloud) | SQLite for local, PostgreSQL for dashboard persistence |
| Charts | Recharts or Chart.js | Confusion matrix, risk timeline, FPS graphs |

### 5.3 Communication Protocol

```
Terminal ←──WebSocket──→ Edge Server ←──WebSocket──→ Dashboard
                              │
                         REST API (for enrollment,
                         session management,
                         evidence retrieval)
```

- **Terminal → Edge:** WebSocket for real-time events. JSON messages.
- **Edge → Dashboard:** WebSocket for live alert feed. REST for historical queries.
- **Dashboard → Terminal:** REST API for evidence snapshot retrieval (authenticated).
- **Heartbeat:** Every 5 seconds. Terminal sends FPS, face count, risk score. Edge responds with ack + any configuration updates.

---

## Phase 6: Post-Exam Analysis

### 6.1 Session Summary Generation

After exam ends:
1. Seal the hash chain (final event = "EXAM_END")
2. Compute confusion matrix against ground truth (if available)
3. Generate per-student risk report
4. Package all evidence snapshots
5. Generate PDF summary report

### 6.2 Ground Truth Comparison (For POC Demo)

The jury will provide a list of planted malpractice events with timestamps. Our system:
1. Takes the jury's ground truth log
2. Matches each ground truth event to our nearest detected event (within ±15 second window)
3. Computes: TP (matched), FN (missed), FP (flagged but no ground truth match)
4. Outputs the confusion matrix and detection accuracy

---

## Feature Priority for Implementation

### Tier 1: Must Ship (Core — wins or loses the hackathon)

| # | Feature | Effort | Impact |
|---|---------|--------|--------|
| 1 | Face enrollment + verification | 2 days | Critical — detects seat swaps, impersonation |
| 2 | Multi-face detection | 0.5 days | Critical — detects unauthorized assistance |
| 3 | Candidate absence detection | 0.5 days | Critical — detects person leaving |
| 4 | Gaze deviation tracking | 1 day | High — detects looking at neighbor |
| 5 | Audio VAD | 1 day | High — detects whispering/talking |
| 6 | Real-time alert dashboard | 2 days | Critical — the jury needs to SEE the system working |
| 7 | Event logging with timestamps | 0.5 days | Critical — timestamps must match ground truth |
| 8 | Confusion matrix generator | 0.5 days | Required — Anil sir explicitly asked for this |

**Total Tier 1: ~8 days**

### Tier 2: Should Ship (Differentiators — puts us in top 3)

| # | Feature | Effort | Impact |
|---|---------|--------|--------|
| 9 | Object detection (phones, books) | 1 day | High — catches material-based cheating |
| 10 | CCTV room monitoring | 1.5 days | Medium-High — room-level corroboration |
| 11 | Hash-chain tamper-proof logs | 0.5 days | Medium — demonstrates evidence integrity |
| 12 | Multi-modal fusion (combining signals) | 1 day | High — reduces false positives dramatically |
| 13 | Evidence snapshot capture | 0.5 days | Medium — provides visual proof for reviewers |
| 14 | Post-exam PDF report | 0.5 days | Medium — professional output for jury |

**Total Tier 2: ~5.5 days**

### Tier 3: Nice to Have (Wow Factor — makes us memorable)

| # | Feature | Effort | Impact |
|---|---------|--------|--------|
| 15 | Bias metrics dashboard | 1 day | High for credibility — shows awareness |
| 16 | Person Re-ID across webcam + CCTV | 2 days | Medium — cross-camera correlation |
| 17 | Adaptive scheduling (risk-based) | 0.5 days | Medium — smart resource management |
| 18 | Offline mode (4G drops) | 0.5 days | Medium — realistic deployment scenario |
| 19 | Scalability calculator | 0.5 days | Medium — shows 10K center math |
| 20 | DPDP compliance badges in UI | 0.5 days | Low-Medium — regulatory awareness |

---

## Tech Stack Summary

| Layer | Technology | Role |
|-------|-----------|------|
| **Terminal Agent** | Python 3.10+ | AI inference, webcam capture, local processing |
| **AI Models** | ONNX Runtime (CPU) | Model inference engine (quantized int8) |
| **Face Detection** | MediaPipe BlazeFace | Real-time face detection (3MB, 70fps) |
| **Face Verification** | MobileFaceNet + ArcFace | Identity verification (4MB, 99.55% LFW) |
| **Gaze Tracking** | MediaPipe Face Mesh | Head pose + iris tracking (478 landmarks) |
| **Object Detection** | YOLOv8-nano (Ultralytics) | Phone/book detection (6MB, 10-25fps) |
| **Audio** | WebRTC VAD + librosa | Speech detection + MFCC features |
| **Edge Server** | FastAPI + SQLite | Aggregation, logging, CCTV processing |
| **Dashboard Backend** | FastAPI + WebSocket | Real-time API + alert streaming |
| **Dashboard Frontend** | React + Tailwind + Recharts | Monitoring UI |
| **Communication** | WebSocket + REST | Real-time + request-response |
| **Evidence Store** | Encrypted SQLite + filesystem | Local tamper-proof storage |

---

## Implementation Order (7-10 Day Sprint)

| Day | Tasks |
|-----|-------|
| **Day 1** | Project setup: Python env, ONNX Runtime, MediaPipe, model downloads. Basic webcam capture + face detection working. |
| **Day 2** | Face enrollment flow (5-frame capture, embedding extraction). Face verification (cosine distance matching). |
| **Day 3** | Gaze tracking (MediaPipe Face Mesh, head pose estimation, yaw/pitch thresholds). Multi-face detection. |
| **Day 4** | Audio pipeline (WebRTC VAD, speech ratio tracking). Event logging system (SQLite + hash chain). |
| **Day 5** | Dashboard backend (FastAPI, WebSocket, REST endpoints). Dashboard frontend scaffold (React + Tailwind). |
| **Day 6** | Dashboard frontend (room overview, terminal grid, alert feed, CCTV view). Real-time WebSocket integration. |
| **Day 7** | Object detection (YOLOv8-nano for phones/books). Multi-modal fusion logic. |
| **Day 8** | CCTV processing pipeline. Evidence snapshot capture. Adaptive scheduling. |
| **Day 9** | Confusion matrix generator. Post-exam report. End-to-end integration testing. |
| **Day 10** | Bug fixes, performance tuning, demo rehearsal. Stress test with 30 simulated terminals. |

---

## Expected Performance Metrics

| Metric | Target | Basis |
|--------|--------|-------|
| Face detection accuracy | >95% | MediaPipe BlazeFace benchmarks |
| Face verification accuracy | >95% | MobileFaceNet 99.55% LFW, conservative for Indian demographics |
| Gaze deviation detection | 80-85% | MediaPipe Face Mesh with calibration |
| Multi-face detection | 90-95% | MediaPipe multi-face, persistence filtering |
| Audio speech detection | 75-85% | WebRTC VAD, depends on ambient noise |
| Phone/book detection | 85-90% | YOLOv8-nano with persistence filtering |
| Overall 8/10 target | ACHIEVABLE | Multi-modal fusion boosts weakest detectors |
| False positive rate | <5% | Persistence filtering + multi-signal corroboration |
| Processing FPS | 12-15fps (i5), 8-10fps (Celeron) | Adaptive scheduling with frame skipping |
| Alert latency | <2 seconds | Local processing, no network dependency |

---

## Critique Synthesis (3 Adversarial Reviews)

Three expert perspectives reviewed this plan: **AI/ML Engineer** (accuracy & detection), **Hackathon Strategist** (scope & demo), and **Systems Engineer** (reliability & deployment). Key findings and revisions below.

### Critical Revisions Based on Critique

#### R1. Replace WebRTC VAD with Silero VAD
**Problem:** WebRTC VAD is designed for teleconferencing. In a 30-student exam hall with keyboard clicks, fans, shuffling papers, it will produce **15-30% false positive rate** on speech detection. This directly hurts the confusion matrix.

**Fix:** Use **Silero VAD** (2MB ONNX model, ~1ms per 30ms chunk). Far better at rejecting non-speech sounds. Drop-in replacement. Still runs on CPU with negligible overhead.

Update in tech stack: `WebRTC VAD` → `Silero VAD (ONNX, 2MB)`

#### R2. Revise Face Verification to 3-Zone Hysteresis
**Problem:** Single threshold (distance > 0.6 = mismatch) is too brittle. Lighting changes, head tilt, or glasses can push a genuine match to 0.55-0.65, causing spurious critical alerts.

**Fix:** Three zones:
- **Match:** distance < 0.45 → confirmed, log silently
- **Uncertain:** 0.45 - 0.70 → increase verification frequency, do NOT alert
- **Mismatch:** distance > 0.70 sustained for 3+ consecutive checks → ALERT: FACE_MISMATCH

Also: store **all 5 enrollment embeddings** (not just median). Compare against closest of 5 — handles pose variation better.

#### R3. Revise Gaze Deviation — Promote to Standalone with Conservative Thresholds
**Problem:** Original plan made gaze fusion-only, meaning it could never fire on its own. But a student staring at their neighbor's screen for 30 seconds straight is cheating — period. Keeping it fusion-only meant missing these obvious scenarios.

**Fix:** Baseline-relative thresholds with **dual-mode (standalone + fusion)**:
- **Standalone Tier 1 (Medium):** Yaw > baseline + 30° sustained > **30 seconds** → fires independently
- **Standalone Tier 2 (High):** Yaw > baseline + 35° sustained > **3 seconds** → fires independently
- **Standalone Tier 3 (Critical):** Yaw > baseline + 45° for any duration → fires independently
- **Fusion mode:** Yaw > baseline + 25° sustained > **5 seconds** + concurrent signal → fires as fusion

Standalone thresholds are deliberately conservative (30s for medium) to prevent false positives from natural looking-around. Fusion thresholds remain aggressive.

Continuously adapt baseline during exam (slow-moving average of median head position).

#### R3b. Add Head-Pitch Posture Detection (SUSPICIOUS_POSTURE) — NEW
**Problem:** Cutting YOLO left zero detection for phones and cheat sheets — the #1 and #2 most common cheating methods. This is an existential gap.

**Fix:** Use **Face Mesh pitch angle** (already computed, ~0ms additional cost) as a posture proxy:
- Head pitch > 30° downward sustained > 10 seconds → ALERT: SUSPICIOUS_POSTURE
- Baseline-adjusted per student during enrollment (some students naturally look slightly down)
- This catches: phone in lap, notes on desk, cheat sheet under keyboard
- It's not object detection, but it covers the observable behavior that accompanies object use
- 10-second threshold filters natural head dips (reading questions, typing)

#### R3c. Promote Audio to Standalone — NEW
**Problem:** Audio as fusion-only meant a student clearly having a conversation could never trigger an alert unless they were also looking sideways. That's leaving easy detections on the table.

**Fix:** Dual-mode audio thresholds:
- **Standalone:** Sustained speech > **10 seconds** → SUSPICIOUS_AUDIO (high) — fires independently
- **Fusion:** Speech > **5 seconds** + concurrent GAZE_DEVIATION → POSSIBLE_CONSULTATION
- Common-mode rejection still applies (3+ terminals in 500ms → suppress)
- 10-second standalone threshold is very conservative — filters coughs, sneezes, brief muttering

#### R4. Add CLAHE Preprocessing for Face Detection
**Problem:** Exam halls have uneven lighting (window glare, fluorescent flicker). Face detection and verification accuracy degrades significantly with poor contrast.

**Fix:** Apply **CLAHE (Contrast Limited Adaptive Histogram Equalization)** to every frame before face detection. ~2ms overhead per frame. Dramatically improves detection in low-light or uneven-light conditions. OpenCV has this built-in.

#### R5. Fine-Tune YOLOv8 or Supplement with Targeted Data
**Problem:** YOLOv8-nano trained on COCO detects phones in ideal conditions but fails on: phones held under desk, cheat sheets (not a COCO class), phones face-down. COCO "cell phone" class has low representation in exam-like scenarios.

**Fix for hackathon:** Collect 100-200 images of phones-in-hand at desk angles + cheat sheets. Fine-tune YOLOv8-nano for 50 epochs (~2 hours on GPU). Alternatively, skip fine-tuning and accept lower phone detection accuracy — **focus on face-based detections which are higher impact**.

#### R6. Object Detection is Tier 2, Not Tier 1
**Agreed by all 3 reviewers.** Phone/book detection with COCO-trained YOLO will produce false positives (water bottles, pencil cases flagged as phones). For the hackathon demo, face verification + gaze + multi-face + absence detection are far more reliable signals. Object detection is nice-to-have.

---

### Scope Reduction for Hackathon (Strategist's Recommendation)

**Original plan: 20 features across 3 tiers = too ambitious for 7-10 days.**

Revised focus — **6 core features that will win the demo:**

| # | Feature | Why It Wins |
|---|---------|------------|
| 1 | Face enrollment + verification (with 3-zone hysteresis) | Detects impersonation/seat swaps — the #1 demo scenario |
| 2 | Multi-face detection | Visually dramatic — jury sees a second face appear |
| 3 | Candidate absence detection | Easy win, reliable, no false positives |
| 4 | Gaze deviation tracking (revised thresholds) | Looking at neighbor's screen — classic cheating |
| 5 | Real-time dashboard with terminal grid + alert feed | The jury SEES the system working in real-time |
| 6 | Confusion matrix generator + ground truth comparison | Anil sir explicitly asked for this; quantifies accuracy |

**Cut from hackathon scope:**
- ~~CCTV processing~~ (complex, requires separate camera setup, high risk of demo failure)
- ~~Hash-chain tamper-proof logs~~ (nice on paper, jury won't verify it)
- ~~PDF report generation~~ (jury sees the dashboard live, doesn't need a PDF)
- ~~Bias metrics dashboard~~ (important for production, not for hackathon scoring)
- ~~Person Re-ID~~ (2 days effort, marginal value for 10-min demo)
- ~~Adaptive scheduling~~ (optimize later, not visible to jury)

**Added to scope:**
- **Simple exam interface** (10 MCQ questions displayed to student — we need actual exam content for the demo to look real)
- **Ground truth input interface** (admin page where jury can log planted malpractice events for confusion matrix comparison)
- **Demo rehearsal script** (practice the 6-10 staged malpractice events ourselves before demo day)

### Audio VAD: Include but Set Conservative Thresholds
**Strategist says deprioritize, AI/ML says include with Silero.** Compromise: include Silero VAD but set **very conservative thresholds** (sustained speech > 10 seconds only, ignore brief sounds). Better to miss whispers than flood dashboard with false audio alerts during demo.

---

### Systems Engineering Fixes

#### S1. Threading Architecture
**Problem:** Python GIL means webcam capture + model inference on same thread = frame drops.

**Fix:** Three threads with queues:
```
Thread 1: Webcam capture → frame queue (producer)
Thread 2: Model inference → reads from frame queue, writes to event queue (consumer/producer)
Thread 3: Network sender → reads from event queue, sends to edge server (consumer)
```
Use `threading.Thread` + `queue.Queue`. Keep it simple — no asyncio in the agent.

#### S2. Model Pre-Loading
**Problem:** Loading 4 ONNX models takes 20-30 seconds on Celeron. If done at exam start, students wait.

**Fix:** Load all models during terminal calibration (Phase 0.2). Models stay in memory through enrollment and exam. Display progress bar during load.

#### S3. Webcam Reliability (Highest Risk)
**Problem:** Webcam capture is the #1 point of failure. USB webcams disconnect, drivers crash, `cv2.VideoCapture` silently returns black frames.

**Fix:**
- Retry logic with exponential backoff on capture failure
- Black frame detection (check if frame mean < 5)
- Heartbeat: if no valid frame for 10 seconds, raise WEBCAM_FAILURE alert to dashboard
- **Bring 2-3 backup USB webcams to the demo**

#### S4. Store-and-Forward for Events
**Problem:** If edge server is temporarily unreachable (LAN switch hiccup), events are lost.

**Fix:** Events are always written to local SQLite first, then forwarded to edge server. If edge is down, queue locally. Replay on reconnect. Simple and bulletproof.

#### S5. Memory Budget on 4GB Terminals
Estimated memory usage:
- Python + ONNX Runtime: ~150MB
- 4 models loaded: ~100MB (INT8 quantized)
- OpenCV + frame buffers: ~80MB
- Exam browser (if Electron): ~200MB
- **Total: ~530MB**
- Windows 10: ~1.5GB base
- **Remaining: ~2GB headroom on 4GB machine** — acceptable

Mitigation: Set `OMP_NUM_THREADS=2` to limit ONNX parallelism. Use INT8 quantized models only.

#### S6. Deployment via PyInstaller
**Problem:** Installing Python + dependencies on 30 terminals manually is error-prone.

**Fix:** Build PyInstaller single-file executable. Copy to USB drive. Plug and run on each terminal. **Test the PyInstaller build on Day 2** — don't wait until demo day to discover dependency issues.

#### S7. Determine Target OS Immediately
The demo terminals' OS (Windows 10/11 vs Ubuntu) affects:
- Webcam driver behavior (DirectShow vs V4L2)
- PyInstaller build target
- Kiosk mode setup
- Audio capture library

**Action: Ask the department what OS the demo terminals run.** Build and test for that OS.

---

### Revised Implementation Schedule (10 Days)

| Day | Tasks | Deliverable |
|-----|-------|-------------|
| **Day 1** | Project setup. Python env, install MediaPipe + ONNX Runtime + OpenCV. Basic webcam capture with threading. Face detection (BlazeFace) working. **Build PyInstaller test binary.** | Webcam shows face bounding boxes |
| **Day 2** | Face enrollment (5-frame capture, 5 embeddings stored). Face verification with 3-zone hysteresis. CLAHE preprocessing. | Enroll → verify working end-to-end |
| **Day 3** | Gaze tracking (Face Mesh, head pose, revised 3-tier thresholds). Multi-face detection with persistence filtering. Absence detection. | All 4 core detections working |
| **Day 4** | Audio pipeline (Silero VAD, conservative thresholds). Event logging (SQLite, hash chain). Store-and-forward pattern. | Audio + logging complete |
| **Day 5** | Edge server backend (FastAPI, WebSocket). Dashboard frontend scaffold (React + Tailwind + Vite). Terminal grid component. | Backend serves events, frontend shows grid |
| **Day 6** | Dashboard: alert feed, terminal detail view, WebSocket real-time updates. Simple exam interface (10 MCQs). | Full dashboard with live alerts |
| **Day 7** | Object detection (YOLOv8-nano, phones/books). Multi-modal fusion (weighted risk scoring). Ground truth input interface. | Object detection + fusion + ground truth UI |
| **Day 8** | Confusion matrix generator. Post-exam summary view. Evidence snapshot capture on high/critical alerts. | End-to-end flow complete |
| **Day 9** | Integration testing with simulated malpractice. Fix bugs. Performance tuning (frame rates, thresholds). | System stable under test |
| **Day 10** | Demo rehearsal. Practice 6-10 malpractice scenarios. Fine-tune thresholds based on rehearsal. Prepare backup plan. | Ready for demo |

---

### Demo Day Checklist

- [ ] Backup USB webcams (2-3 extra)
- [ ] USB ring lights for consistent face illumination
- [ ] PyInstaller binary on USB drives
- [ ] Pre-recorded fallback video (in case webcam fails during demo)
- [ ] 10 MCQ questions loaded in exam interface
- [ ] Ground truth logging sheet for jury
- [ ] Practice the 6-10 malpractice scenarios beforehand
- [ ] Test on actual demo terminals (not just dev laptop)
- [ ] Bring Ethernet switch + cables (don't rely on WiFi)
- [ ] Have the dashboard running on a large screen/projector for jury visibility

---

### Revised Expected Performance (Post-Critique)

| Metric | Original Target | Revised Target | Notes |
|--------|----------------|----------------|-------|
| Face detection | >95% | >95% | BlazeFace with CLAHE — solid |
| Face verification | >95% | >90% | 3-zone hysteresis reduces FP but may miss some mismatches |
| Gaze deviation (standalone) | fusion-only | 70-80% | **Promoted:** 30s sustained threshold = high precision, catches obvious looking-away |
| Multi-face detection | 90-95% | 90-95% | Unchanged — reliable |
| Audio speech detection (standalone) | fusion-only | 60-70% | **Promoted:** 10s sustained speech = high confidence conversation detection |
| Head-pitch posture (phone/notes proxy) | N/A | 65-75% | **NEW:** pitch >30° for >10s catches phone-in-lap and notes-on-desk scenarios |
| Phone/book detection | 85-90% | **CUT** | YOLO removed; head-pitch posture proxy replaces as indirect detection |
| **Overall 8/10 target** | **ACHIEVABLE** | **STRONG** | 6 standalone alerts + 3 fusion rules cover 7-8/10 scenarios reliably |
| False positive rate | <5% | <3% | Conservative standalone thresholds + persistence = fewer false alarms |
| Processing FPS | 12-15fps | 12-15fps | Head pitch adds ~0ms (Face Mesh already running); i5 with INT8 models |

### Scenario Coverage Analysis (Post-Revision)

| Malpractice Scenario | Detection Method | Confidence |
|----------------------|-----------------|------------|
| Candidate leaves seat | CANDIDATE_ABSENT (face gone >5s) | Very High |
| Extra person enters frame | MULTIPLE_PERSONS (2+ faces >5 frames) | Very High |
| Impersonation / seat swap | FACE_MISMATCH (3-zone hysteresis) | High |
| Talking to neighbor | SUSPICIOUS_AUDIO standalone (>10s) | High |
| Looking at neighbor's screen | GAZE_DEVIATION standalone (>30s sustained) | High |
| Phone under desk / in lap | SUSPICIOUS_POSTURE (head pitch down >10s) | Medium-High |
| Notes on desk / lap | SUSPICIOUS_POSTURE (head pitch down >10s) | Medium-High |
| Passed note from outside | GAZE_DEVIATION + possible MULTIPLE_PERSONS | Medium |

**Reliable catches: 7/10. With fusion boosting marginal signals: 8/10.**

**Key insight:** It's better to detect 8/10 events with <3% FP than to detect 10/10 events with 15% FP. The jury will penalize false positives as much as missed detections. Tune for precision over recall.

**Post-innovation-critique insight:** The original 3-standalone-alert architecture was "competent but generic" — the same MediaPipe+MobileFaceNet stack that 80% of competitors will build. Promoting gaze and audio to standalone and adding head-pitch posture detection expands coverage from ~3/10 to ~8/10 malpractice scenarios with **zero additional CPU cost** (Face Mesh is already running, Silero is already running). This is the difference between mid-pack and winning.

---

## Multi-LLM Debate Consensus (Claude Opus + GPT-5.2 + Gemini 3 Pro)

Full debate reports: `debate-report.md` (Round 1, 3 rounds) and `debate-report-r2.md` (Round 2, 5 rounds — **CONVERGED**).

**Consensus Level: FULL AGREEMENT** — All 3 models reached AGREE by Round 5 (auto-stopped, skipped Round 6).

### 29 Points of Unanimous Agreement

**Architecture:**
1. Edge-first, on-terminal inference, no cloud dependency for core functionality
2. **Single-process Python agent with PyQt6** — NO Electron (saves 300-500MB RAM, mandatory on 4GB Celeron)
3. **3-thread architecture:** Thread 1 (webcam capture → frame queue), Thread 2 (model inference → event queue), Thread 3 (network sender)
4. **Detect → Track → Verify** as the BASELINE architecture (not optimization): BlazeFace 3-5fps → lightweight tracker (MOSSE/CSRT) → MobileFaceNet every 2s or on track loss
5. **CLAHE preprocessing** on every frame before face detection (~2ms, handles uneven lighting)

**Scope Cuts:**
6. **YOLOv8-nano CUT ENTIRELY** — COCO domain mismatch, high CPU cost, FP risk, AGPL licensing
7. **CCTV, Person Re-ID, hash-chain, PDF reports, bias metrics, adaptive scheduling ALL CUT**
8. No message broker (NATS/MQTT/Redis) — in-memory queues + HTTP/SSE sufficient

**Alert Strategy (Precision Over Recall):**
9. **6 standalone alert types:**
   - CANDIDATE_ABSENT (>5s no face detected)
   - MULTIPLE_PERSONS (>5 consecutive frames with 2+ faces)
   - FACE_MISMATCH (3-zone hysteresis: match <0.45, uncertain 0.45-0.70, mismatch >0.70 sustained 3+ checks)
   - GAZE_DEVIATION (sustained >30s off-center beyond baseline + 30°) — **promoted from fusion-only** with conservative standalone threshold
   - SUSPICIOUS_AUDIO (sustained speech >10s via Silero VAD) — **promoted from fusion-only** with conservative standalone threshold
   - SUSPICIOUS_POSTURE (head pitch >30° downward sustained >10s) — **NEW**, detects phone-in-lap/notes-on-desk via Face Mesh pitch angle, ~2ms/frame cost (already running Face Mesh)
10. **Gaze remains a fusion signal too** — standalone fires only at 30s sustained; fusion fires at 5s when combined with other signals
11. **Audio remains a fusion signal too** — standalone fires only at 10s sustained speech; fusion fires at 5s when combined with gaze
12. **3 hardcoded fusion rules:** (1) GAZE + FACE_MISMATCH within ±5s → CONFIRMED_IMPERSONATION; (2) AUDIO + GAZE within ±5s → POSSIBLE_CONSULTATION; (3) SUSPICIOUS_POSTURE + GAZE_DEVIATION within ±5s → POSSIBLE_MATERIAL_USE
13. Every false alarm during demo destroys jury credibility — tune for precision
14. **Rationale for promotion:** With only 3 standalone alerts, the system could reliably catch ~3/10 malpractice scenarios. Promoting gaze and audio to standalone (with conservative thresholds) and adding head-pitch posture detection brings reliable catches to 7-8/10 — the difference between mid-pack and winning.

**Event Model:**
14. **Episode-based events** (not per-frame spam): terminals emit EPISODE_START/EPISODE_END with stable episode_id. Schema: `{episode_id, terminal_id, student_id, alert_type, severity, started_at, ended_at, peak_confidence, trigger_count, corroborating_episodes}`
15. **1:1 greedy matching** for confusion matrix: each ground truth event matches at most one episode within ±15s
16. **Common-mode audio rejection at edge:** if >30% of terminals trigger simultaneously → suppress as ambient noise

**Dashboard:**
17. **FastAPI + SSE + vanilla JS/HTMX** — Streamlit rejected (full-page rerenders stutter with 30 streams), React rejected (build-chain complexity)
18. SSE sends **delta messages only** (episode start/end + periodic heartbeat per terminal)
19. **LAN-first**, hosted on edge server, cloud optional
20. **Ground truth live logger:** `/admin/ground-truth` page with big buttons per malpractice type, capturing `server_now()` on click (NO manual time entry)

**Infrastructure:**
21. **Mini-PC with SSD** for edge server (not Raspberry Pi + SD card)
22. **SQLite WAL mode** on edge as primary persistence, single async writer queue
23. Terminal-side **JSONL files** as store-and-forward buffer for edge-down resilience
24. **Push-based evidence:** terminal pushes encrypted JPEG to edge via HTTP POST on high/critical alerts only

**Deployment:**
25. **Day-0 packaging spike = existential go/no-go gate.** Test on non-dev target machine.
26. **Deployment decision tree:** Plan A = WinPython portable on USB (no admin needed); Plan B = PyInstaller; Plan C = bootable Linux (only if BIOS confirmed)
27. Include **VC++ Redistributable + DirectX runtime** on USB as insurance

**Reliability:**
28. **App-level time sync:** RTT/2 offset estimation every heartbeat, median of last 5, monotonic clocks for intervals
29. **FPS-based degradation modes:** ≥12fps = full; ≥7fps = shed gaze; ≥4fps = shed audio; <4fps = presence + multi-face only (SURVIVAL mode). Dashboard shows mode per terminal.
30. **Enrollment quality gates:** face confidence >0.9, face >80px, blur/brightness checks, auto-retry with guidance. After 3 failures → "Force Proceed" → terminal flagged DEGRADED
31. **Video file fallback:** `--source` CLI flag (device index or file path)
32. Prepare **10 MCQ questions** as hardcoded JSON on Day 1
33. **Exam start/stop synchronization** via edge broadcast to all terminals

### Remaining Disagreements (Minor — All Within Narrow Band)

| Topic | Status |
|-------|--------|
| Audio: include or cut entirely | Claude + GPT-5.2: include as fusion-only, cut if unstable by Day 4. Gemini: prefers cutting. **Resolution: attempt it, Day 4 cut decision.** |
| Vertical slice day | Day 3 target (Gemini), Day 4 hard deadline (all agree). **Not an architectural disagreement.** |
| Hash-chain logging | Implicitly dropped from scope by all. Not worth debating. |
| JSONL audit trail on edge | Claude + GPT-5.2: yes (one `file.write()` call). Gemini: neutral. **Trivial, include it.** |

### Final Converged Architecture

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
│  T1: Webcam capture       │       │  • 2 fusion rules            │
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

### Final Implementation Schedule (Converged)

| Day | Focus | Exit Criterion |
|-----|-------|----------------|
| **Day 0** | **GO/NO-GO:** WinPython portable on USB + webcam + all models + mic on non-dev Windows machine. FastAPI+SSE hello-world. Confirm target OS. Include VC++ Redist + DirectX on USB. | Deployment method confirmed or pivot |
| **Day 1** | Webcam capture + 3 threads + BlazeFace + CLAHE + `--source` flag + MOSSE tracker. Create `exam_content.json` (10 MCQs). Write episode event schema. GIL-awareness test (PyQt6 label + background ONNX = no stutter). | Face boxes on live feed with tracking |
| **Day 2** | Enrollment (5 embeddings, quality gates, Force Proceed after 3 failures). Face verification (3-zone hysteresis). Mock agent sending synthetic episodes to edge. Start edge server (FastAPI, SQLite WAL, writer queue). | Enroll → verify + mock events on edge |
| **Day 3** | Multi-face + absence + gaze (fusion-only, 1-2fps). FPS degradation modes in inference loop. **TARGET: Real camera → real inference → real episodes → edge → basic dashboard tile.** | **Vertical slice working (target)** |
| **Day 4** | **HARD DEADLINE for vertical slice** (if not working, stop all features, debug integration). Silero VAD (fusion-only, 10s threshold) — **cut if unstable**. App-level time sync. Store-and-forward (JSONL spool, idempotent replay). | Events flow terminal → edge with time sync |
| **Day 5** | Dashboard: terminal grid (30 tiles via SSE), episode alert feed, terminal detail view. PyQt6 exam UI (MCQs, timer, navigation). Exam start/stop broadcast. Ground truth live logger (`/admin/ground-truth` with big buttons). | **Minimum viable demo works end-to-end** |
| **Day 6** | Evidence snapshot push (JPEG to edge on high/critical episodes). Common-mode audio rejection. Evidence viewer in dashboard. 2 fusion rules on edge. | Dashboard shows evidence + fusion alerts |
| **Day 7** | Confusion matrix (1:1 greedy matching). Post-exam summary view. Multi-terminal testing (3-5 terminals). | Scoring works end-to-end |
| **Day 8** | Multi-terminal stress test (10+ terminals simulated). Threshold tuning on real/closest hardware. Detect-track-verify optimization if FPS insufficient. | Stable under load |
| **Day 9** | **Calibration exam:** run ALL planned malpractice scenarios. Record raw scores. Tune thresholds empirically. Test degradation modes. Pre-record demo video as insurance. | Thresholds tuned, fallback video recorded |
| **Day 10** | **Demo rehearsal only.** Practice 6-10 scenarios. Verify confusion matrix output. Fix doc inconsistencies. Prepare backups (webcams, ring lights, video fallback, USB drives). | Ready for demo |

### Demo Day Checklist (Converged)

- [ ] 2-3 backup USB webcams + USB ring lights
- [ ] WinPython portable (or PyInstaller binary) on USB drives, tested on target OS
- [ ] VC++ Redistributable + DirectX runtime installers on USB
- [ ] Pre-recorded demo video as fallback
- [ ] Edge server mini-PC with SSD + Ethernet switch + LAN cables (NO WiFi dependency)
- [ ] 10 MCQ questions loaded in `exam_content.json`
- [ ] Ground truth logger tested and ready for jury
- [ ] Dashboard on large screen/projector connected to edge server
- [ ] Windows Defender exclusion configured on all terminals
- [ ] All 6-10 malpractice scenarios rehearsed by team
- [ ] `config.yaml` with tuned thresholds from Day 9 calibration
- [ ] Document reconciled: evidence flow, privacy claims, terminal count consistent
