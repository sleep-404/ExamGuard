# ExamGuard: Technical Architecture Plan

## Context

This is a hackathon entry for RTGS (Real Time Governance Society) — building an AI-enabled live exam proctoring system for the Dept. of School Education, Govt. of Andhra Pradesh. Performance, accuracy, and features win.

**Constraints:** 30 terminals, 10-min exams, 8/10 malpractice detection, 4G connectivity, ~₹30K/center budget, basic desktop CPUs (i5/Celeron), open-source only.

---

## System Architecture

```
┌──────────────────────────────┐       ┌──────────────────────────────┐
│  Terminal (x30)              │       │  Edge Server (Mini-PC + SSD) │
│                              │       │                              │
│  ┌────────────────────────┐  │  LAN  │  FastAPI + SSE + vanilla JS  │
│  │ Headless Python Agent  │  │──────→│                              │
│  │ (background process)   │  │ JSON  │  Persistence:                │
│  │ ** OWNS THE CAMERA **  │  │ episo │  • SQLite WAL (primary)      │
│  │                        │  │ des + │  • JSONL per-terminal (audit)│
│  │ AI Pipeline (threaded):│  │ JPEG  │  • Hash-chained events (P2)  │
│  │ T1: Webcam capture     │  │ snaps │                              │
│  │ T2: Detect→Track→Verify│  │  +    │  Logic:                      │
│  │ T3: Event/evidence push│  │ embed │  • 3 fusion rules            │
│  │ Bounded queue (size 1) │  │ dings │  • Cross-terminal seat-swap  │
│  │ — drop stale frames    │  │       │  • Common-mode audio CLASSIFY│
│  │                        │  │       │  • Time sync authority       │
│  │ Models (INT8 ONNX):    │  │       │  • Episode aggregation       │
│  │ • BlazeFace (detect)   │  │       │  • Evidence store            │
│  │ • MOSSE (track)        │  │       │  • All enrollment embeddings │
│  │ • MobileFaceNet (verify)│ │       │                              │
│  │ • PnP head pose (gaze) │  │       │  Dashboard (vanilla JS+SSE): │
│  │ • Silero VAD (audio)   │  │       │  • Terminal grid (30 tiles)  │
│  │ • Material heuristics  │  │       │  • Episode alert feed        │
│  │                        │  │       │  • Terminal detail view      │
│  │ Serves to browser:     │  │       │  • Evidence viewer           │
│  │ • /video_feed (MJPEG)  │  │       │  • Confusion matrix (P2)    │
│  │ • /api/enrollment/*    │  │       │  • Ground truth logger       │
│  │ • /api/exam/*          │  │       │  • Malpractice trigger app   │
│  │ • /api/webcam/status   │  │       │  • CCTV + motion heatmap    │
│  └───────┬────────────────┘  │       │  • Degradation status        │
│          │ MJPEG feed        │       │                              │
│  ┌───────▼────────────────┐  │       │  NO cloud dependency         │
│  │ Browser UI (HTML/JS)   │  │       │                              │
│  │ served by agent locally│  │       │  ┌────────────────────────┐  │
│  │ ** NO camera access ** │  │       │  │ CCTV Camera (RTSP/USB) │  │
│  │ reads /video_feed only │  │       │  │ + BackgroundSubtractor │  │
│  │                        │  │       │  │   MOG2 motion heatmap  │  │
│  │ • Enrollment flow      │  │       │  │ (AI on edge, no faces) │  │
│  │ • 10 MCQ exam interface│  │       │  └────────────────────────┘  │
│  │ • Timer + navigation   │  │       │                              │
│  │ • Webcam status light  │  │       │  Mobile-friendly pages:      │
│  └────────────────────────┘  │       │  • /admin/ground-truth       │
│                              │       │  • /admin/malpractice-trigger│
│  NO Electron, NO PyQt6       │       │                              │
│  NO YOLO, NO MediaPipe (P0)  │       │                              │
│  NO raw video leaves         │       │                              │
│  Episode-based events        │       │                              │
│  640x480 capture resolution  │       │                              │
└──────────────────────────────┘       └──────────────────────────────┘
```

**Terminal:** Headless Python agent runs as a background process. **The agent exclusively owns the webcam** via `cv2.VideoCapture(0)` and serves an MJPEG stream at `http://localhost:{PORT}/video_feed`. The browser exam UI consumes this feed via `<img>` tag — it never opens the camera directly. This solves Windows webcam exclusivity conflicts (only one process can access a camera). Exposes a local HTTP API (`/api/enrollment/*`, `/api/exam/*`, `/api/webcam/status`). No Electron, no PyQt6.

**Edge server:** Mini-PC with SSD (not Raspberry Pi — SD cards corrupt under write-heavy workloads). Runs FastAPI with SSE for dashboard streaming. Vanilla JS + HTMX dashboard — Streamlit rejected (full-page rerenders stutter with 30 streams), React rejected (build-chain complexity). Also runs CCTV motion analysis via `BackgroundSubtractorMOG2`.

**3-thread architecture:** Thread 1 (webcam capture → frame queue), Thread 2 (detect → track → verify inference → event queue), Thread 3 (event/API/evidence push). Uses `threading.Thread` + `queue.Queue` with **bounded queue of size 1** — stale frames are dropped, not queued. This prevents latency explosion on slow hardware. The API/event thread reads shared state atomically (`latest_frame`, `latest_alerts`) and never blocks on inference. No asyncio in the agent. No multiprocessing — duplicating the Python runtime costs ~200MB, unaffordable on 4GB machines.

---

## Pre-Exam Setup

### Center Setup (One-Time)

1. **Install ExamGuard Agent** on each terminal (headless Python agent)
   - Downloads all AI models locally (~11MB total)
   - Models: BlazeFace (3MB), MobileFaceNet (4MB), Silero VAD (2MB ONNX)
   - Gaze: PnP head pose from BlazeFace keypoints (no separate model)
   - Material detection: skin-tone + phone glow heuristics (no model)
   - Requires ONNX Runtime (CPU) + OpenCV
   - **NO MediaPipe in P0/P1** — avoids DLL/protobuf conflicts with ONNX Runtime on Windows

2. **Connect Edge Server** to LAN
   - Mini-PC with SSD
   - Runs aggregation service, dashboard, and tamper-proof log store
   - Connected to all terminals via Ethernet switch

3. **Mount CCTV** (with motion AI on edge)
   - RTSP or USB camera at rear of room (privacy-preserving — faces not visible from back)
   - Edge server ingests feed → `BackgroundSubtractorMOG2` → motion heatmap overlay on dashboard
   - Detects "high commotion" (running, fighting, mass movement) — cheap, no face AI
   - Satisfies the "integrate CCTV feeds with AI" jury requirement

4. **Run Pre-Flight Diagnostic** on every terminal (**Phase 0 gating deliverable**)
   - Single script, runnable from USB, outputs PASS/FAIL with specific failure reasons
   - Tests: webcam open/close, microphone access
   - Tests: ONNX Runtime import + all model loads
   - Tests: OpenCV import + basic frame processing
   - Tests: **available RAM measurement after Chrome + AV are running** (critical — real conditions)
   - Tests: sustained 30-second inference loop reporting actual FPS
   - Tests: network connectivity to edge server + clock offset measurement
   - Tests: disk write permissions to working directory
   - **Go/No-Go decisions based on results:**
     - If available RAM < 350MB after Chrome + AV → PnP-only gaze, heuristics-only material, no optional models
     - If ONNX Runtime + OpenCV have DLL conflicts → pivot packaging strategy
     - If sustained FPS < 5 → reduce resolution to 480x360 or cut audio from terminal
   - Run in first 15 minutes of setup; **no feature coding begins until Phase 0 passes**

### Exam Configuration (Before Each Session)

1. **Create Exam Session** via dashboard — exam ID, duration, terminal count
2. **Terminal Calibration** — webcam check, mic check, AI model load, ambient noise baseline, ambient light baseline
3. **Model Pre-Loading** — all ONNX models loaded during calibration, stay in memory through enrollment and exam. Display progress bar.

---

## Enrollment Flow (5 min before exam)

```
Student arrives → Sits at terminal → Enrollment screen in browser
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
                                    │ Compute 5 face │
                                    │ embeddings     │
                                    │ (MobileFaceNet)│
                                    │ Store all 5    │
                                    └─────┬──────────┘
                                          │
                                    ┌─────▼──────────┐
                                    │ Gaze Baseline  │
                                    │ Calibration    │
                                    │ (10s median+MAD│
                                    │  of head pose) │
                                    └─────┬──────────┘
                                          │
                                    ┌─────▼──────┐
                                    │ Push to Edge│
                                    │ (embeddings │
                                    │ + metadata) │
                                    └─────┬──────┘
                                          │
                                    ┌─────▼──────┐
                                    │ Ready!      │
                                    └─────────────┘
```

### Enrollment Quality Gates
- Face confidence > 0.9
- Face size > 80px
- Blur and brightness checks
- Auto-retry with guidance on failure
- After 3 failures → "Force Proceed" → terminal flagged DEGRADED

### What Gets Sent to Edge Server
- All 5 enrollment embeddings (for cross-terminal seat-swap matching)
- Terminal-to-student mapping metadata
- Enrollment quality score
- Baseline head pose (yaw, pitch)

---

## Processing Pipeline (Per Terminal, During Exam)

Every frame passes through CLAHE preprocessing (~2ms) before any detection. Capture resolution: 640x480. **Bounded queue of size 1** between capture → inference — stale frames are dropped, not accumulated.

**Detect → Track → Verify** is the baseline architecture (not optimization):
- BlazeFace runs at 3-5fps for detection
- MOSSE lightweight tracker fills inter-detection frames (MOSSE over CSRT — 5-10x faster, sufficient since MobileFaceNet re-verifies every 2s)
- MobileFaceNet verifies every 2s or on track loss

```
Webcam (640x480) ──→ CLAHE preprocessing ──→ Brightness check ──→ Pipeline
    │
    ├── Brightness Gate (per-frame, ~0.1ms)
    │   └── Mean brightness < threshold for >2s → POOR_LIGHTING (not ABSENT)
    │       Prevents false CANDIDATE_ABSENT from auto-exposure hunting,
    │       flickering lights, or hand-over-camera scenarios
    │
    ├── Face Detection (BlazeFace, ~15ms)
    │   ├── 0 faces → CANDIDATE_ABSENT (critical, >5s)
    │   ├── 1 face → Continue to track/verify
    │   └── 2+ faces → MULTIPLE_PERSONS (high, >5 consecutive frames)
    │
    ├── Tracking (MOSSE between detections, <1ms)
    │
    ├── Face Verification (MobileFaceNet, every 2s or track loss)
    │   └── 3-zone hysteresis (configurable thresholds, cosine similarity):
    │       ├── <0.45 → match (log silently)
    │       ├── 0.45-0.70 → uncertain (increase verify frequency, NO alert)
    │       └── >0.70 sustained 3+ checks → FACE_MISMATCH (critical)
    │       Compare against closest of 5 enrollment embeddings
    │
    ├── Gaze + Posture (PnP head pose from BlazeFace keypoints, ~2ms)
    │   ├── Uses 5-6 face detection keypoints + cv2.solvePnP for yaw/pitch
    │   ├── NO Face Mesh in P0/P1 — saves ~30-40ms/frame on Celeron
    │   ├── Yaw > baseline+30° sustained >30s → GAZE_DEVIATION (medium)
    │   ├── Yaw > baseline+35° sustained >3s → GAZE_DEVIATION (high)
    │   ├── Yaw > baseline+45° any duration → GAZE_DEVIATION (critical)
    │   └── Pitch >30° down >10s → SUSPICIOUS_POSTURE (fusion-only)
    │       └── NEVER fires standalone — too many FPs from normal reading
    │   Thresholds are configurable (default 30° relative to calibrated baseline;
    │   fall back to 35° absolute if calibration proves unreliable)
    │
    ├── Material Heuristics (no model, ~1-2ms, runs even in SHED_GAZE mode)
    │   ├── Phone screen glow: bright rectangular region in lower frame half
    │   ├── Hand-near-face: skin-tone blob in lower-face / desk ROI
    │   └── Either trigger → POSSIBLE_MATERIAL_USE with evidence snapshot
    │       Independent of gaze state — active even during FPS degradation
    │
    └── Audio (Silero VAD, 2MB ONNX, ~1ms per 30ms chunk)
        ├── Sustained speech >10s → SUSPICIOUS_AUDIO (high, standalone)
        └── Speech >5s + GAZE_DEVIATION → POSSIBLE_CONSULTATION (fusion)

Microphone (16kHz) ──────────────────────────────────────────────────

P2 UPGRADE PATH (conditional on Phase 0 RAM results):
    ├── Duty-cycled Face Mesh at 1-2fps for iris-level gaze accuracy
    │   (prefer small ONNX landmark model over MediaPipe if DLL issues)
    └── Conditional MobileNet-SSD INT8 at 0.5-1fps for phone/paper detection
        (independent trigger, not gated by gaze state)
```

---

## Alert Types & Severity

| Alert Type | Mode | Threshold | Severity | Notes |
|------------|------|-----------|----------|-------|
| POOR_LIGHTING | **Informational** | Mean brightness < threshold >2s | Info | Prevents false ABSENT from auto-exposure/flickering |
| CANDIDATE_ABSENT | **Standalone** | >5s no face (with brightness gate) | Critical | Near-certain detection; brightness check prevents false triggers |
| MULTIPLE_PERSONS | **Standalone** | >5 consecutive frames | High | Ignore reflections (check face size ratios) |
| FACE_MISMATCH | **Standalone** | 3-zone hysteresis (>0.70 sustained 3+ checks) | Critical | + cross-terminal seat-swap on edge |
| GAZE_DEVIATION | **Standalone + Fusion** | Standalone: >30s at +30°, >3s at +35°, any at +45°; Fusion: >5s with other signal | Medium/High/Critical | PnP head pose; baseline calibrated per-student (10s median+MAD); configurable thresholds |
| SUSPICIOUS_AUDIO | **Standalone + Fusion** | Standalone: >10s sustained; Fusion: >5s with gaze | High | Common-mode **classification** at edge (not suppression) |
| SUSPICIOUS_POSTURE | **Fusion-only** | Head pitch >30° down >10s + GAZE_DEVIATION | High | NEVER standalone — too noisy from reading/thinking |
| POSSIBLE_MATERIAL_USE | **Heuristic (standalone)** | Phone glow OR skin-tone blob in desk region | Medium | No model; runs even in SHED_GAZE mode; evidence snapshot to dashboard for human confirmation |

### Fusion Rules (3, on edge server)

| Combination | Result | Confidence |
|-------------|--------|------------|
| GAZE_DEVIATION + FACE_MISMATCH within ±5s | CONFIRMED_IMPERSONATION | Very High |
| SUSPICIOUS_AUDIO + GAZE_DEVIATION within ±5s | POSSIBLE_CONSULTATION | High |
| SUSPICIOUS_POSTURE + GAZE_DEVIATION within ±5s | POSSIBLE_MATERIAL_USE | High |

### Cross-Terminal Seat-Swap Detection (Edge Server)

Enrollment embeddings from all 30 terminals are stored on edge. On FACE_MISMATCH at any terminal, edge compares the mismatched embedding against ALL enrolled candidates. If a match is found at another terminal, emit CONFIRMED_SEAT_SWAP with both terminal IDs. ~2 hours implementation, potential demo-winning moment.

### Common-Mode Audio Classification (Edge Server)

**Classify, don't suppress** — suppression can hide coordinated collusion (e.g., proctor dictating answers).

Buffer VAD events for 500ms:
- If ≥3 terminals trigger simultaneously → reclassify as `ROOM_NOISE` (low severity, **logged, not discarded**)
- If exactly 2 adjacent terminals trigger → escalate to `POSSIBLE_CONSULTATION` (elevated severity)
- Single terminal speech → keep as `SUSPICIOUS_AUDIO` (original severity)

This preserves evidence of room-wide events while differentiating from individual whispering. Addresses the 30-microphone physics problem without creating false negatives.

---

## Event Model

**Episode-based** (not per-frame spam). Terminals emit EPISODE_START/EPISODE_END with stable episode_id.

```json
{
  "episode_id": "ep_00023",
  "terminal_id": "T-15",
  "student_id": "STU-042",
  "alert_type": "FACE_MISMATCH",
  "severity": "critical",
  "started_at": "2026-03-15T10:05:30.000Z",
  "ended_at": "2026-03-15T10:05:38.500Z",
  "peak_confidence": 0.92,
  "trigger_count": 4,
  "corroborating_episodes": ["ep_00021"],
  "previous_hash": "a3f2b8c9d1e4f7...",
  "event_hash": "d7e1f4a2b5c8e3..."
}
```

**Hash-chaining:** `hash_i = sha256(hash_{i-1} || canonical_json(event_i))` in JSONL. Directly addresses "tamper-proof logs" requirement.

---

## Data Flow

```
┌─────────────────┐          ┌──────────────────┐
│ Terminal (x30)   │   LAN   │ Edge Server      │
│                  │────────→│ (Mini-PC + SSD)  │
│ Sends:           │         │                  │
│ • JSON episodes  │         │ Receives:        │
│ • JPEG evidence  │         │ • Episodes       │
│   snapshots      │         │ • Evidence       │
│ • Enrollment     │         │ • Embeddings     │
│   embeddings     │         │                  │
│                  │         │ Stores:          │
│ NO raw video     │         │ • SQLite WAL     │
│ leaves terminal  │         │ • JSONL audit    │
│                  │         │ • Hash-chained   │
│ Store-and-forward│         │   event log      │
│ with JSONL spool │         │                  │
│ for resilience   │         │ Serves:          │
│                  │         │ • Dashboard (SSE)│
│                  │         │ • CCTV display   │
└─────────────────┘         └──────────────────┘
```

**Evidence push:** Terminal pushes encrypted JPEG to edge via HTTP POST on high/critical episodes only. Evidence snapshots include bounding boxes, alert type, timestamp, and confidence overlaid on frame.

**Store-and-forward:** Events always written to local JSONL first, then forwarded to edge. If edge is unreachable, queue locally and replay on reconnect.

---

## Dashboard

**Tech:** FastAPI + SSE + vanilla JS/HTMX. No React, no Streamlit.

SSE sends **delta messages only** — episode start/end + periodic heartbeat per terminal.

**SQLite WAL mode**, single async writer queue. Batch flush every 250ms or 50 pending events. Prevents SQLITE_BUSY under 30-terminal concurrency.

### Dashboard Views

**Terminal Grid (Default):**
```
┌──────────────────────────────────────────────────────────────┐
│  ExamGuard Dashboard — EXAM-2026-03-15-HALL-A    [LIVE]      │
│                                                               │
│  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐          │
│  │ T-01│ │ T-02│ │ T-03│ │ T-04│ │ T-05│ │ T-06│          │
│  │ OK  │ │ OK  │ │ALERT│ │ OK  │ │ GAZE│ │ OK  │          │
│  └─────┘ └─────┘ └─────┘ └─────┘ └─────┘ └─────┘          │
│  ...  (30 tiles total)                                       │
│                                                               │
│  ┌──────────────────────────────────────────────────────────┐│
│  │ CCTV Live Feed + Motion Heatmap (AI on edge)            ││
│  └──────────────────────────────────────────────────────────┘│
│                                                               │
│  Episode Alert Feed (last 60 seconds):                       │
│  CRITICAL 10:05:32 T-03 FACE_MISMATCH (conf: 0.92)          │
│  MEDIUM   10:05:28 T-05 GAZE_DEVIATION (yaw: 32°, 31s)      │
│  CRITICAL 10:04:15 T-10 CANDIDATE_ABSENT (8s)                │
│                                                               │
│  Stats: 30 terminals | 28 active | 1 alert | 1 absent       │
│  Degradation: T-22 [SHED_GAZE] | T-29 [SURVIVAL]            │
└──────────────────────────────────────────────────────────────┘
```

### Dashboard Pages

| Page | Purpose |
|------|---------|
| Terminal grid (30 tiles) | Room overview with color-coded status per terminal |
| Episode alert feed | Chronological stream of all episode events |
| Terminal detail | Per-student timeline, risk graph, head pose visualization |
| Evidence viewer | On-demand JPEG evidence with annotations |
| Confusion matrix | Real-time TP/FP/FN/TN against ground truth |
| Ground truth logger | `/admin/ground-truth` — big buttons per malpractice type, captures `server_now()` on click (no manual time entry) |
| Malpractice trigger app | `/admin/malpractice-trigger` — mobile web page, 6 buttons + terminal selector for actors staging events |
| CCTV feed + motion heatmap | RTSP/USB ingest → `BackgroundSubtractorMOG2` → motion heatmap overlay + commotion index (AI on edge, no face processing) |
| Degradation status | FPS mode per terminal (full / shed_gaze / shed_audio / survival) |
| Post-exam summary | Per-student risk report, all flagged episodes, confusion matrix |

---

## FPS Degradation Modes

| FPS | Mode | Active Models | Material Heuristics |
|-----|------|---------------|---------------------|
| ≥12 | FULL | All models active | Active |
| ≥7 | SHED_GAZE | Drop PnP gaze tracking | **Still active** (independent) |
| ≥4 | SHED_AUDIO | Drop gaze + audio | **Still active** (independent) |
| <4 | SURVIVAL | Presence + multi-face only | **Still active** (independent) |

**Material heuristics (phone glow + skin-tone) run independently of all degradation modes.** They cost ~1-2ms/frame and must never be shed — they are the only material detection capability in P0/P1.

Dashboard shows degradation mode per terminal. Degradation is visible to the operator.

---

## Models (All INT8 ONNX, Local on Terminal)

### P0/P1 Models (always loaded)

| Model | Purpose | Size | Speed |
|-------|---------|------|-------|
| BlazeFace | Face detection + keypoints for PnP | 3MB | ~15ms/frame |
| MOSSE | Lightweight face tracking | N/A (OpenCV) | <1ms/frame |
| MobileFaceNet + ArcFace | Face verification (512D embeddings) | 4MB | ~18ms/run |
| PnP Head Pose | Gaze via `cv2.solvePnP` from BlazeFace keypoints | N/A (OpenCV) | ~2ms/frame |
| Silero VAD | Audio voice activity detection | 2MB ONNX | ~1ms/30ms chunk |
| Material Heuristics | Phone glow + skin-tone blob detection | N/A (OpenCV) | ~1-2ms/frame |

**Total P0/P1 model footprint: ~11MB** (down from ~13MB with Face Mesh)

### P2 Conditional Models (loaded only if Phase 0 shows RAM headroom)

| Model | Purpose | Size | Speed | Condition |
|-------|---------|------|-------|-----------|
| Face Mesh (ONNX, not MediaPipe) | Iris-level gaze accuracy (478 landmarks) | ~2MB | ~20ms/run | Duty-cycled at 1-2fps; only if available RAM > 400MB |
| MobileNet-SSD INT8 | Phone/paper object detection | ~15-25MB | ~15ms/run | At 0.5-1fps; only if available RAM > 450MB |

### NOT included

- **YOLOv8-nano** — cut entirely: COCO domain mismatch, high CPU cost (~80ms/frame), AGPL licensing risk
- **MediaPipe (as a package)** — avoided in P0/P1 due to known DLL/protobuf conflicts with ONNX Runtime + OpenCV on Windows portable Python. If Face Mesh is needed in P2, prefer a standalone ONNX export over the MediaPipe pip package.
- **CSRT tracker** — replaced by MOSSE (5-10x faster). CSRT's higher accuracy is unnecessary since MobileFaceNet re-verifies every 2 seconds anyway.

**CLAHE preprocessing** applied to every frame before face detection (~2ms overhead). Dramatically improves detection under uneven exam hall lighting (window glare, fluorescent flicker).

---

## Deployment

### Packaging (Both Prepared, Decide On-Site)

| Option | Method | Pros | Cons |
|--------|--------|------|------|
| Plan A (primary) | WinPython portable on USB | No admin needed, no install | Larger USB payload |
| Plan B (fallback) | PyInstaller `--onedir` | Single directory copy | May hit AV false positives |

Decide after pre-flight test on target machine. Include VC++ Redistributable + DirectX runtime on USB as insurance.

### Pre-Flight Diagnostic Script

Run on every terminal before exam:
- Camera accessible and returning valid frames
- Microphone accessible
- ONNX models load successfully
- Write permissions to working directory
- LAN connectivity to edge server

### Memory Budget (4GB Terminal) — Realistic Estimate

**WARNING:** The original ~430MB estimate was dangerously optimistic. Chrome on government machines + AV can consume significantly more than estimated. Phase 0 must **measure empirically** on real hardware.

| Component | Optimistic | Realistic (with AV + Chrome) |
|-----------|-----------|------------------------------|
| Windows 10 base | ~1.5GB | ~1.5GB |
| Antivirus (mandatory on govt machines) | — | ~200MB |
| Chrome/Edge (exam UI via MJPEG) | ~150MB | ~500MB–1.5GB |
| Python + ONNX Runtime | ~150MB | ~150MB |
| P0/P1 Models loaded (INT8) | ~30MB | ~30MB |
| OpenCV + frame buffers | ~80MB | ~80MB |
| **Total** | ~1.91GB | ~2.46–3.46GB |
| **Remaining headroom on 4GB** | ~2GB | **~550MB–1.5GB** |

**Phase 0 Go/No-Go thresholds:**
- Available RAM after Chrome + AV < 350MB → P2 models forbidden, PnP-only gaze, heuristics-only material
- Available RAM 350-500MB → P0/P1 models only, no upgrades
- Available RAM > 500MB → P2 conditional models may be loaded

**Mitigations:** `OMP_NUM_THREADS=2` to limit ONNX parallelism. INT8 quantized models only. 640x480 capture resolution. Bounded queue size 1 (no frame accumulation). No multiprocessing (saves ~200MB from duplicated Python runtime).

### Time Synchronization

App-level sync: RTT/2 offset estimation on every heartbeat, median of last 5 readings, monotonic clocks for interval measurement. Edge server is the time authority.

**Ground truth logger must also sync.** At startup, the ground truth logger (mobile web page) performs a single HTTP timestamp exchange with the edge server, stores the offset, and applies it to all ground truth timestamps. Without this, even 3-5 seconds of clock drift will corrupt the confusion matrix — making accurate detections appear as false negatives. Budget: 1 hour, P1 priority.

---

## Feature Priority Table

**Write this as a physical/visible artifact before any code. This is the single highest-leverage action from the multi-LLM debate.**

| Priority | Features | Cut trigger |
|----------|----------|-------------|
| **P0 (demo fails without)** | Agent camera ownership + MJPEG feed, CANDIDATE_ABSENT (with lighting check), MULTIPLE_PERSONS, FACE_MISMATCH, enrollment flow (5 embeddings + quality gates), edge dashboard with alert feed, pre-flight diagnostic script | Never cut |
| **P1 (8/10 at risk without)** | GAZE_DEVIATION (PnP-based, configurable 30°-35°), SUSPICIOUS_AUDIO (Silero VAD), cross-terminal SEAT_SWAP, unauthorized material heuristics (skin-tone + phone glow), common-mode audio classification, clock sync for ground truth logger, FPS degradation modes | Cut individual items if behind by >4 hours on P0 |
| **P2 (innovation/wow)** | Live confusion matrix, CCTV motion heatmap, duty-cycled Face Mesh gaze upgrade, conditional MobileNet-SSD material detector, hash-chained audit trail, store-and-forward, timeline visualization, enrollment liveness (blink detection), scalability one-pager | Cut entirely if P0+P1 not passing 3-terminal test |

**Hard rule:** No P2 work begins until ALL P0+P1 items pass a live 3-terminal integration test.

---

## Implementation Phases

### Phase 0: Pre-Flight (Gating Milestone)

**This is a first-class deliverable, not a checklist.** Build a single diagnostic script that is the go/no-go gate for the entire project.

- Pre-flight diagnostic on a non-dev Windows machine (or closest approximation)
- Test WinPython portable and PyInstaller `--onedir`
- Confirm target OS, include VC++ Redist + DirectX on USB
- Validate ONNX Runtime + OpenCV coexistence (watch for DLL/protobuf conflicts)
- **Measure available RAM empirically** after Chrome + AV are running
- Run sustained 30-second inference, report actual FPS
- **Go/No-Go decisions:**
  - RAM < 350MB → PnP-only gaze, heuristics-only material, no P2 models
  - DLL conflicts with any package → eliminate that package, pivot
  - FPS < 5 → reduce resolution or cut audio

**Exit criterion:** Deployment method confirmed, RAM budget known, all packages validated. No feature coding begins until this passes.

### Phase 1: Core Pipeline (P0 Features)

- **Agent owns the camera** via `cv2.VideoCapture(0)`, serves MJPEG at `localhost:{PORT}/video_feed`
- Browser exam UI consumes video via `<img src="/video_feed">` — NO direct camera access
- 3-thread architecture with **bounded queue size 1** (capture, inference, event/API)
- BlazeFace + CLAHE + `--source` flag + MOSSE tracker, 640x480 capture
- Create `exam_content.json` (10 MCQs), episode event schema
- Enrollment (5 embeddings, quality gates, Force Proceed after 3 failures)
- Face verification (3-zone hysteresis, **cosine similarity** explicitly validated, configurable thresholds)
- **PnP head pose** from BlazeFace keypoints (no Face Mesh) — gaze threshold configurable
- **Per-frame brightness tracking** — POOR_LIGHTING guard for CANDIDATE_ABSENT
- Push enrollment embeddings to edge
- Edge server start (FastAPI, SQLite WAL, async single-writer queue)
- Browser enrollment UI + exam UI (MCQs, timer)

**Exit criterion:** Enroll → verify working, MJPEG serving, embeddings on edge, PnP gaze active.

### Phase 2: Detection & Vertical Slice (P1 Features — Hard Deadline)

- All P0 alerts: CANDIDATE_ABSENT (with lighting gate), MULTIPLE_PERSONS, FACE_MISMATCH
- P1 alerts: GAZE_DEVIATION (PnP-based), SUSPICIOUS_AUDIO (Silero VAD, 10s threshold)
- **Unauthorized material heuristics** (independent of gaze state, runs even in SHED_GAZE):
  - Bright rectangular region in lower frame half (phone screen glow)
  - Skin-tone blob in lower-face / desk ROI (hand holding object)
  - Triggers POSSIBLE_MATERIAL_USE with evidence snapshot
- Cross-terminal seat-swap on edge
- **Common-mode audio classification** on edge (≥3 terminals → ROOM_NOISE; 2 adjacent → POSSIBLE_CONSULTATION)
- FPS degradation modes (FULL → SHED_GAZE → SHED_AUDIO → SURVIVAL)
- App-level time sync + ground truth logger clock sync
- Episode events to edge
- **Vertical slice with 3 terminals** — all P0+P1 detections flowing terminal → edge → dashboard

**Exit criterion:** 3 terminals → edge → dashboard with all P0+P1 detections working reliably. **This is the milestone gate.**

### Phase 3: Dashboard + CCTV + Demo Infrastructure

- Dashboard: terminal grid (30 tiles via SSE), alert feed, terminal detail, degradation status, evidence viewer
- **CCTV tile with `BackgroundSubtractorMOG2`** + motion heatmap overlay on edge (satisfies "AI on CCTV")
- Exam start/stop broadcast
- Ground truth logger + malpractice trigger app (mobile-friendly, with **"skip to next scenario" button**)
- Demo choreography script with **90-second hard limits** per detection type
- 3 fusion rules on edge (CONFIRMED_IMPERSONATION, POSSIBLE_CONSULTATION, POSSIBLE_MATERIAL_USE)
- Evidence snapshot push (JPEG to edge on high/critical)

**Exit criterion:** End-to-end demo works with all detection types, CCTV motion, and controlled choreography.

### Phase 4: Conditional Upgrades (P2 — only if time + RAM permit)

**Gate:** ALL P0+P1 items passing 3-terminal live test. RAM headroom confirmed in Phase 0.

- If RAM allows: duty-cycled Face Mesh at 1-2fps for iris-level gaze (prefer ONNX export over MediaPipe)
- If RAM allows: conditional MobileNet-SSD INT8 at 0.5-1fps for phone/paper detection
- Live confusion matrix (1:1 greedy matching, ±15s window)
- Hash-chained audit trail with periodic anchoring to edge
- Timeline visualization per terminal (green/yellow/red segments with evidence thumbnails)
- Enrollment-time liveness check (blink detection via eye aspect ratio)
- Store-and-forward (JSONL spool for LAN failures)
- Post-exam summary report
- Scalability one-pager for jury

**Exit criterion:** Dashboard shows evidence + P2 features + scoring.

### Phase 5: Validation & Demo Prep

- Multi-terminal stress test (10+ terminals simulated)
- **Tune all configurable thresholds** (gaze angle, face verification zones, duration windows) on actual hardware using a 30-60 minute capture session
- Calibration exam: run ALL planned malpractice scenarios, record raw scores, test degradation modes
- **Rehearse full demo choreography at least twice** with timing (90s per scenario)
- **Record "golden demo" fallback video** (3-min walkthrough — non-optional insurance against hardware failures)
- Package validation: test final deployment artifact on a clean machine
- Verify confusion matrix accuracy by checking clock alignment
- Prepare USB backups of both packaging options

**Exit criterion:** Thresholds tuned, demo rehearsed, fallback video recorded, ready for demo.

**Milestone Gate:** If Phase 0–2 items are not complete by mid-sprint, cut ALL Phase 4 items and focus exclusively on making P0+P1 detections rock-solid.

---

## Demo Choreography

**Scripted, rehearsed, time-boxed.** Each scenario gets ~90 seconds with a "skip to next" button on the ground truth logger. No improvisation — a 10-minute window with 6 detection types leaves zero room for delays.

Show detections in confidence order (highest-reliability first):

1. **CANDIDATE_ABSENT** (~90s) — student leaves seat, alert fires in 5s. Easy, reliable, dramatic.
2. **MULTIPLE_PERSONS** (~90s) — second person enters frame. Visually obvious to jury.
3. **FACE_MISMATCH + CONFIRMED_SEAT_SWAP** (~90s) — two students swap seats. Edge cross-references embeddings, identifies both terminals. **Wow moment.**
4. **POSSIBLE_MATERIAL_USE** (~90s) — student holds phone below desk. Heuristic detects glow/hand in desk region. Evidence snapshot appears on dashboard.
5. **GAZE + AUDIO fusion** (~90s) — student looks sideways while whispering. Fusion rule fires POSSIBLE_CONSULTATION.
6. **Confusion matrix live** (~60s) — show TP/FP/FN computed against ground truth in real-time. Quantitative proof.

### Demo Day Checklist

- [ ] 2-3 backup USB webcams + USB ring lights
- [ ] WinPython portable AND PyInstaller binary on USB drives, tested on target OS
- [ ] VC++ Redistributable + DirectX runtime installers on USB
- [ ] Pre-flight diagnostic script tested and ready (the Phase 0 script)
- [ ] **"Golden demo" fallback video** (3-min walkthrough, queued to play within 30s if hardware fails)
- [ ] Edge server mini-PC with SSD + Ethernet switch + LAN cables (NO WiFi dependency)
- [ ] Portable WiFi router (~₹500-1,000) for malpractice trigger app on mobile
- [ ] USB webcam (~₹1,500) as CCTV camera fallback
- [ ] 10 MCQ questions loaded in `exam_content.json`
- [ ] Ground truth logger + malpractice trigger app tested (with **skip button** and **clock sync**)
- [ ] Dashboard on large screen/projector connected to edge server
- [ ] Windows Defender exclusion configured on all terminals
- [ ] **All 6 scenarios rehearsed** in confidence order with **90-second time limits**
- [ ] `config.yaml` with tuned thresholds from calibration runs (gaze angle, face verify zones, durations)
- [ ] Scalability one-pager printed for jury
- [ ] **Demo choreography script printed** with scenario order, timing, and skip triggers

---

## Scenario Coverage

| Malpractice Scenario | Detection Method | Confidence |
|----------------------|-----------------|------------|
| Candidate leaves seat | CANDIDATE_ABSENT (face gone >5s, with brightness gate) | Very High |
| Extra person enters frame | MULTIPLE_PERSONS (2+ faces >5 frames) | Very High |
| Impersonation / seat swap | FACE_MISMATCH (3-zone) + CONFIRMED_SEAT_SWAP (edge) | High |
| Talking to neighbor | SUSPICIOUS_AUDIO standalone (>10s) | High |
| Looking at neighbor's screen | GAZE_DEVIATION standalone (>30s sustained, PnP head pose) | High |
| **Phone on desk / in lap** | **Material heuristic (phone glow)** + SUSPICIOUS_POSTURE fusion | **Medium-High** |
| **Notes/cheat sheet on desk** | **Material heuristic (skin-tone blob)** + GAZE_DEVIATION | **Medium-High** |
| Phone under desk (hidden) | SUSPICIOUS_POSTURE + GAZE → POSSIBLE_MATERIAL_USE (fusion) | Medium |
| Passed note from outside | GAZE_DEVIATION + possible MULTIPLE_PERSONS | Medium |

**Reliable catches: 8/10. Precision over recall.** The addition of material heuristics (phone glow + skin-tone detection) plugs the critical gap identified by all 3 debate models. Without material detection, the system could only reliably catch 6-7/10 scenarios. With it: 8/10. The heuristics may produce some false flags, but evidence snapshots go to the dashboard for human confirmation — the system flags, the proctor decides.

It is better to detect 8/10 events with <5% FP than to detect 10/10 events with 15% FP. The jury will penalize false positives as much as missed detections.

## Expected Performance

| Metric | Target | Basis |
|--------|--------|-------|
| Face detection accuracy | >95% | BlazeFace with CLAHE preprocessing |
| Face verification accuracy | >90% | MobileFaceNet 99.55% LFW; 3-zone hysteresis reduces FP; configurable thresholds |
| Gaze deviation (PnP standalone) | 65-75% | PnP head pose from 5-6 keypoints; less precise than Face Mesh but 30s duration threshold compensates; configurable 30°-35° |
| Multi-face detection | 90-95% | BlazeFace multi-face with persistence filtering |
| Audio speech detection (standalone) | 60-70% | 10s sustained speech = high confidence; common-mode classification prevents false suppression |
| Material detection (heuristic) | 50-65% | Phone glow + skin-tone heuristics are noisy but catch visible phone screens; evidence snapshot for human review |
| Head-pitch posture (fusion-only) | 65-75% | Pitch >30° for >10s catches phone-in-lap and notes-on-desk |
| **Overall 8/10 target** | **STRONG** | 7 alert types + material heuristics + 3 fusion rules cover 8/10 scenarios |
| False positive rate | <5% | Conservative standalone thresholds + persistence filtering + human confirmation via dashboard for heuristic alerts |
| Processing FPS | 12-15fps (i5), 8-12fps (Celeron) | INT8 models, PnP gaze (~2ms vs ~20ms Face Mesh), 640x480, bounded queue frame-dropping |
| Alert latency | <2 seconds | Local processing, no network dependency for detection |

---

## Multi-LLM Debate Consensus

### Debate History

Two debates have been conducted, with findings incorporated into this architecture:

1. **Prior debate** (`debate-report.md`): 3 rounds, Claude Opus + GPT-5.2 + Gemini 3 Pro. Established the 49 core consensus points (edge-first architecture, 6 alert types, episode model, detect-track-verify pipeline, etc.). Full agreement.

2. **Final pre-implementation debate** (`debate-report-final.md`): 3 rounds, same 3 models. Focused on accuracy (8/10), performance (4GB Celeron), and innovation (hackathon-winning). Near-consensus with critical architecture changes.

### Final Debate: 15 Common Ground Points (All 3 Models Agree)

1. **Unauthorized material/phone detection is a critical gap** that must be addressed. Without it, 8/10 is at serious risk if any test event involves phones/notes. Added as material heuristics (P1).
2. **Agent must exclusively own the webcam** and serve MJPEG to browser. Solves Windows camera exclusivity. P0 architectural requirement.
3. **Common-mode audio must classify, not suppress.** Suppression can hide coordinated collusion. ≥3 terminals → ROOM_NOISE (logged); 2 adjacent → POSSIBLE_CONSULTATION.
4. **Threading over multiprocessing on 4GB.** Multiprocessing costs ~200MB. Bounded queues of size 1 with frame dropping.
5. **Phase 0 is a first-class gating deliverable.** Empirical RAM measurement, DLL conflict validation, sustained FPS test. No coding until it passes.
6. **~430MB memory budget is dangerously thin.** Must be measured empirically under real conditions (Windows + AV + Chrome).
7. **Face Mesh every frame is too expensive on Celeron.** Must be replaced (PnP head pose) or heavily duty-cycled (P2 upgrade).
8. **CCTV cannot be display-only.** BackgroundSubtractorMOG2 + motion heatmap on edge satisfies "AI on CCTV." Promoted to Phase 3.
9. **Lighting check prevents false CANDIDATE_ABSENT.** Track brightness, emit POOR_LIGHTING if dark >2s. 30-minute fix.
10. **Explicit P0/P1/P2 priority table** with hard cut rules, created before coding. No P2 until P0+P1 pass 3-terminal test.
11. **Scope is too ambitious** — 4 rock-solid detectors beat 6 flaky ones. The priority table enforces focus.
12. **All thresholds must be configurable** (settings file) — gaze angle, face verify zones, durations. Tune during validation.
13. **Ground truth logger must clock-sync** to edge server. Even 3-5s drift corrupts confusion matrix.
14. **"Golden demo" fallback** — pre-recorded video as insurance against hardware failures.
15. **Demo must be scripted** with 90-second limits per scenario and "skip to next" buttons.

### Final Debate: 6 Convergence Points

1. **Multiprocessing → threading**: Gemini initially advocated separate process. Conceded after Claude/GPT-5.2 showed 200MB memory cost is fatal on 4GB.
2. **Material detection approach**: Converged on heuristics first (P1), model-based upgrade (P2) only if RAM allows.
3. **Face Mesh handling**: Converged on PnP head pose as default, duty-cycled Face Mesh as conditional P2 upgrade.
4. **CCTV treatment**: All independently flagged stub as insufficient. Converged on background subtraction + motion heatmap.
5. **Audio classification**: GPT-5.2 identified flaw, others adopted. Full convergence on reclassify-not-suppress.
6. **Webcam ownership**: Claude raised, others endorsed. All converged on agent-as-camera-owner with MJPEG.

### 3 Remaining Disagreements (Narrow, Resolved by Configuration)

1. **Gaze threshold 30° vs 35°**: Made configurable. Default 30° relative to baseline; fallback 35° absolute.
2. **MediaPipe vs avoid entirely**: Avoided in P0/P1. If Face Mesh needed in P2, prefer standalone ONNX export.
3. **Object detector trigger strategy**: Moot unless RAM allows model. Heuristics are the primary path regardless.
