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
│  │                        │  │ episo │  • SQLite WAL (primary)      │
│  │ AI Pipeline (threaded):│  │ des + │  • JSONL per-terminal (audit)│
│  │ T1: Webcam capture     │  │ JPEG  │  • Hash-chained events       │
│  │ T2: Detect→Track→Verify│  │ snaps │                              │
│  │ T3: Event/evidence push│  │  +    │  Logic:                      │
│  │                        │  │ embed │  • 3 fusion rules            │
│  │ Models (INT8 ONNX):    │  │ dings │  • Cross-terminal seat-swap  │
│  │ • BlazeFace (detect)   │  │       │  • Common-mode audio reject  │
│  │ • MOSSE/CSRT (track)   │  │       │  • Time sync authority       │
│  │ • MobileFaceNet (verify)│ │       │  • Episode aggregation       │
│  │ • Face Mesh (gaze+pitch)│ │       │  • Evidence store            │
│  │ • Silero VAD (audio)   │  │       │  • All enrollment embeddings │
│  │                        │  │       │                              │
│  │ Local HTTP/WS API:     │  │       │  Dashboard (vanilla JS+SSE): │
│  │ • /api/enrollment/*    │  │       │  • Terminal grid (30 tiles)  │
│  │ • /api/exam/*          │  │       │  • Episode alert feed        │
│  │ • /api/webcam/status   │  │       │  • Terminal detail view      │
│  └───────┬────────────────┘  │       │  • Evidence viewer           │
│          │ local WS          │       │  • Confusion matrix          │
│  ┌───────▼────────────────┐  │       │  • Ground truth logger       │
│  │ Browser UI (HTML/JS)   │  │       │  • Malpractice trigger app   │
│  │ served by agent locally│  │       │  • CCTV live feed stub       │
│  │                        │  │       │  • Degradation status        │
│  │ • Enrollment flow      │  │       │                              │
│  │ • 10 MCQ exam interface│  │       │  NO cloud dependency         │
│  │ • Timer + navigation   │  │       │                              │
│  │ • Webcam status light  │  │       │  ┌────────────────────────┐  │
│  └────────────────────────┘  │       │  │ CCTV Camera (RTSP/USB) │  │
│                              │       │  │ Display-only, no AI    │  │
│  NO Electron, NO PyQt6       │       │  │ + motion overlay        │  │
│  NO YOLO                     │       │  └────────────────────────┘  │
│  NO raw video leaves         │       │                              │
│  Episode-based events        │       │  Mobile-friendly pages:      │
│  640x480 capture resolution  │       │  • /admin/ground-truth       │
│                              │       │  • /admin/malpractice-trigger│
└──────────────────────────────┘       └──────────────────────────────┘
```

**Terminal:** Headless Python agent runs as a background process. Exposes a local HTTP/WS API (`/api/enrollment/*`, `/api/exam/*`, `/api/webcam/status`). Exam UI is HTML/JS served by the agent and opened in the system browser. No Electron, no PyQt6 — eliminates dependency chains and deployment risk on unknown government machines.

**Edge server:** Mini-PC with SSD (not Raspberry Pi — SD cards corrupt under write-heavy workloads). Runs FastAPI with SSE for dashboard streaming. Vanilla JS + HTMX dashboard — Streamlit rejected (full-page rerenders stutter with 30 streams), React rejected (build-chain complexity).

**3-thread architecture:** Thread 1 (webcam capture → frame queue), Thread 2 (detect → track → verify inference → event queue), Thread 3 (event/evidence push to edge). Uses `threading.Thread` + `queue.Queue`. No asyncio in the agent.

---

## Pre-Exam Setup

### Center Setup (One-Time)

1. **Install ExamGuard Agent** on each terminal (headless Python agent)
   - Downloads all AI models locally (~13MB total)
   - Models: BlazeFace (3MB), MobileFaceNet (4MB), Face Mesh (included with MediaPipe), Silero VAD (2MB ONNX)
   - Requires ONNX Runtime (CPU)

2. **Connect Edge Server** to LAN
   - Mini-PC with SSD
   - Runs aggregation service, dashboard, and tamper-proof log store
   - Connected to all terminals via Ethernet switch

3. **Mount CCTV** (display-only stub)
   - RTSP or USB camera at rear of room
   - Edge server ingests feed → displays on dashboard with background subtraction motion overlay
   - NO AI processing on CCTV feed

4. **Run Pre-Flight Diagnostic** on every terminal
   - Camera, mic, ONNX model loading, write permissions, LAN connectivity
   - Run in first 15 minutes of setup

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

Every frame passes through CLAHE preprocessing (~2ms) before any detection. Capture resolution: 640x480.

**Detect → Track → Verify** is the baseline architecture (not optimization):
- BlazeFace runs at 3-5fps for detection
- MOSSE/CSRT lightweight tracker fills inter-detection frames
- MobileFaceNet verifies every 2s or on track loss

```
Webcam (640x480) ──→ CLAHE preprocessing ──→ Pipeline
    │
    ├── Face Detection (BlazeFace, ~15ms)
    │   ├── 0 faces → CANDIDATE_ABSENT (critical, >5s)
    │   ├── 1 face → Continue to track/verify
    │   └── 2+ faces → MULTIPLE_PERSONS (high, >5 consecutive frames)
    │
    ├── Tracking (MOSSE/CSRT between detections)
    │
    ├── Face Verification (MobileFaceNet, every 2s or track loss)
    │   └── 3-zone hysteresis:
    │       ├── <0.45 → match (log silently)
    │       ├── 0.45-0.70 → uncertain (increase verify frequency, NO alert)
    │       └── >0.70 sustained 3+ checks → FACE_MISMATCH (critical)
    │       Compare against closest of 5 enrollment embeddings
    │
    ├── Gaze + Posture (Face Mesh, 478 landmarks, ~20ms)
    │   ├── Yaw > baseline+30° sustained >30s → GAZE_DEVIATION (medium)
    │   ├── Yaw > baseline+35° sustained >3s → GAZE_DEVIATION (high)
    │   ├── Yaw > baseline+45° any duration → GAZE_DEVIATION (critical)
    │   └── Pitch >30° down >10s → SUSPICIOUS_POSTURE (fusion-only)
    │       └── NEVER fires standalone — too many FPs from normal reading
    │
    └── Audio (Silero VAD, 2MB ONNX, ~1ms per 30ms chunk)
        ├── Sustained speech >10s → SUSPICIOUS_AUDIO (high, standalone)
        └── Speech >5s + GAZE_DEVIATION → POSSIBLE_CONSULTATION (fusion)

Microphone (16kHz) ──────────────────────────────────────────────────
```

---

## Alert Types & Severity

| Alert Type | Mode | Threshold | Severity | Notes |
|------------|------|-----------|----------|-------|
| CANDIDATE_ABSENT | **Standalone** | >5s no face | Critical | Near-certain detection |
| MULTIPLE_PERSONS | **Standalone** | >5 consecutive frames | High | Ignore reflections (check face size ratios) |
| FACE_MISMATCH | **Standalone** | 3-zone hysteresis (>0.70 sustained 3+ checks) | Critical | + cross-terminal seat-swap on edge |
| GAZE_DEVIATION | **Standalone + Fusion** | Standalone: >30s at +30°, >3s at +35°, any at +45°; Fusion: >5s with other signal | Medium/High/Critical | Baseline calibrated per-student (10s median+MAD) |
| SUSPICIOUS_AUDIO | **Standalone + Fusion** | Standalone: >10s sustained; Fusion: >5s with gaze | High | Common-mode rejection at edge |
| SUSPICIOUS_POSTURE | **Fusion-only** | Head pitch >30° down >10s + GAZE_DEVIATION | High | NEVER standalone — too noisy from reading/thinking |

### Fusion Rules (3, on edge server)

| Combination | Result | Confidence |
|-------------|--------|------------|
| GAZE_DEVIATION + FACE_MISMATCH within ±5s | CONFIRMED_IMPERSONATION | Very High |
| SUSPICIOUS_AUDIO + GAZE_DEVIATION within ±5s | POSSIBLE_CONSULTATION | High |
| SUSPICIOUS_POSTURE + GAZE_DEVIATION within ±5s | POSSIBLE_MATERIAL_USE | High |

### Cross-Terminal Seat-Swap Detection (Edge Server)

Enrollment embeddings from all 30 terminals are stored on edge. On FACE_MISMATCH at any terminal, edge compares the mismatched embedding against ALL enrolled candidates. If a match is found at another terminal, emit CONFIRMED_SEAT_SWAP with both terminal IDs. ~2 hours implementation, potential demo-winning moment.

### Common-Mode Audio Rejection (Edge Server)

Buffer VAD events for 500ms. If ≥3 terminals trigger simultaneously, suppress all but the highest-amplitude source. Addresses the 30-microphone physics problem in a single room.

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
│  │ CCTV Live Feed (display-only + motion overlay)          ││
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
| CCTV feed | RTSP/USB ingest → display + background subtraction motion overlay, no AI |
| Degradation status | FPS mode per terminal (full / shed_gaze / shed_audio / survival) |
| Post-exam summary | Per-student risk report, all flagged episodes, confusion matrix |

---

## FPS Degradation Modes

| FPS | Mode | Active Models |
|-----|------|---------------|
| ≥12 | FULL | All models active |
| ≥7 | SHED_GAZE | Drop gaze tracking |
| ≥4 | SHED_AUDIO | Drop gaze + audio |
| <4 | SURVIVAL | Presence + multi-face only |

Dashboard shows degradation mode per terminal. Degradation is visible to the operator.

---

## Models (All INT8 ONNX, Local on Terminal)

| Model | Purpose | Size | Speed |
|-------|---------|------|-------|
| BlazeFace | Face detection | 3MB | ~15ms/frame |
| MOSSE/CSRT | Lightweight face tracking | N/A (OpenCV) | <1ms/frame |
| MobileFaceNet + ArcFace | Face verification (512D embeddings) | 4MB | ~18ms/run |
| Face Mesh | Gaze tracking + head pitch (478 landmarks) | Included with MediaPipe | ~20ms/frame |
| Silero VAD | Audio voice activity detection | 2MB | ~1ms/30ms chunk |

**NOT included:** YOLOv8-nano — cut entirely due to COCO domain mismatch (phone detection unreliable at desk angles, "cheat sheet" is not a COCO class), high CPU cost (~80ms/frame), and AGPL licensing risk.

**CLAHE preprocessing** applied to every frame before face detection (~2ms overhead). Dramatically improves detection under uneven exam hall lighting (window glare, fluorescent flicker).

---

## Deployment

### Packaging (Both Prepared, Decide On-Site)

| Option | Method | Pros | Cons |
|--------|--------|------|------|
| Plan A (primary) | WinPython portable on USB | No admin needed, no install | Larger USB payload |
| Plan B (fallback) | PyInstaller `--onedir` | Single directory copy | May hit AV false positives |

Decide after Day 0 test on target machine. Include VC++ Redistributable + DirectX runtime on USB as insurance.

### Pre-Flight Diagnostic Script

Run on every terminal before exam:
- Camera accessible and returning valid frames
- Microphone accessible
- ONNX models load successfully
- Write permissions to working directory
- LAN connectivity to edge server

### Memory Budget (4GB Terminal)

| Component | Usage |
|-----------|-------|
| Python + ONNX Runtime | ~150MB |
| Models loaded (INT8) | ~50MB |
| OpenCV + frame buffers | ~80MB |
| Browser (exam UI) | ~150MB |
| **Total** | ~430MB |
| Windows 10 base | ~1.5GB |
| **Remaining headroom** | ~2GB |

Mitigation: `OMP_NUM_THREADS=2` to limit ONNX parallelism. INT8 quantized models only. 640x480 capture resolution.

### Time Synchronization

App-level sync: RTT/2 offset estimation on every heartbeat, median of last 5 readings, monotonic clocks for interval measurement. Edge server is the time authority.

---

## Implementation Schedule (10 Days)

| Day | Focus | Exit Criterion |
|-----|-------|----------------|
| **Day 0** | **GO/NO-GO:** Pre-flight diagnostic on non-dev Windows machine. Test WinPython portable and PyInstaller `--onedir`. Confirm target OS. Include VC++ Redist + DirectX on USB. | Deployment method confirmed or pivot |
| **Day 1** | Webcam capture + 3 threads + BlazeFace + CLAHE + `--source` flag + MOSSE tracker. Local HTTP/WS API skeleton for browser UI. Create `exam_content.json` (10 MCQs). Episode event schema. 640x480 capture. | Face boxes on live feed with tracking |
| **Day 2** | Enrollment (5 embeddings, quality gates, Force Proceed after 3 failures). Face verification (3-zone hysteresis). Push enrollment embeddings to edge. Edge server start (FastAPI, SQLite WAL, async single-writer queue). Browser enrollment UI. | Enroll → verify + embeddings on edge |
| **Day 3** | Multi-face + absence + gaze (baseline calibration: 10s median+MAD). FPS degradation modes. Episode events to edge. Cross-terminal seat-swap on edge. | All core detections + seat-swap working |
| **Day 4** | **HARD DEADLINE: vertical slice with 3 terminals.** Events flow terminal → edge → dashboard. Discover LAN/SQLite/firewall issues. Silero VAD (10s threshold) — cut if unstable. App-level time sync. Store-and-forward (JSONL spool). Hash-chaining. | 3 terminals → edge → dashboard with time sync |
| **Day 5** | Dashboard: terminal grid (30 tiles via SSE), alert feed, terminal detail, degradation status. Browser exam UI (MCQs, timer). Exam start/stop broadcast. Ground truth logger. Malpractice trigger app. | **MINIMUM VIABLE DEMO works end-to-end** |
| **Day 6** | Evidence snapshot push (JPEG to edge). Common-mode audio rejection. Evidence viewer. 3 fusion rules on edge. CCTV display stub (RTSP/USB + motion overlay). | Dashboard shows evidence + fusion + CCTV |
| **Day 7** | Confusion matrix (1:1 greedy matching, ±15s window). Post-exam summary. Stress test 5-10 terminals. Threshold tuning. Scalability one-pager. | Scoring works, stable under load |
| **Day 8** | Multi-terminal stress test (10+ terminals simulated). Threshold tuning on target hardware. Detect-track-verify optimization if FPS insufficient. | Stable under load |
| **Day 9** | Calibration exam: run ALL planned malpractice scenarios. Record raw scores. Tune thresholds. Test degradation modes. **Record fallback demo video** (3-min walkthrough — NON-OPTIONAL). | Thresholds tuned, fallback video recorded |
| **Day 10** | **Demo rehearsal only.** Practice scenarios in confidence order. Verify confusion matrix output. Prepare USB backups of both packaging options. | Ready for demo |

**Milestone Gate:** If Tier 0 items (preflight, core pipeline, enrollment, edge server, dashboard, 3-terminal vertical slice) are not complete by end of Day 5, cut ALL Tier 2+ items and focus exclusively on core 4 alert types + seat-swap + confusion matrix.

---

## Demo Choreography

Show detections in confidence order (highest-reliability first):

1. **CANDIDATE_ABSENT** — student leaves seat, alert fires in 5s. Easy, reliable, dramatic.
2. **MULTIPLE_PERSONS** — second person enters frame. Visually obvious to jury.
3. **FACE_MISMATCH + CONFIRMED_SEAT_SWAP** — two students swap seats. Edge cross-references embeddings, identifies both terminals. Wow moment.
4. **GAZE + AUDIO fusion** — student looks sideways while whispering. Fusion rule fires POSSIBLE_CONSULTATION.
5. **Confusion matrix live** — show TP/FP/FN computed against ground truth in real-time.

### Demo Day Checklist

- [ ] 2-3 backup USB webcams + USB ring lights
- [ ] WinPython portable AND PyInstaller binary on USB drives, tested on target OS
- [ ] VC++ Redistributable + DirectX runtime installers on USB
- [ ] Pre-flight diagnostic script tested and ready
- [ ] Pre-recorded fallback demo video (3-min walkthrough, queued to play within 30s)
- [ ] Edge server mini-PC with SSD + Ethernet switch + LAN cables (NO WiFi dependency)
- [ ] Portable WiFi router (~₹500-1,000) for malpractice trigger app on mobile
- [ ] USB webcam (~₹1,500) as CCTV camera fallback
- [ ] 10 MCQ questions loaded in `exam_content.json`
- [ ] Ground truth logger + malpractice trigger app tested
- [ ] Dashboard on large screen/projector connected to edge server
- [ ] Windows Defender exclusion configured on all terminals
- [ ] All scenarios rehearsed in confidence order
- [ ] `config.yaml` with tuned thresholds from Day 9 calibration
- [ ] Scalability one-pager printed for jury

---

## Scenario Coverage

| Malpractice Scenario | Detection Method | Confidence |
|----------------------|-----------------|------------|
| Candidate leaves seat | CANDIDATE_ABSENT (face gone >5s) | Very High |
| Extra person enters frame | MULTIPLE_PERSONS (2+ faces >5 frames) | Very High |
| Impersonation / seat swap | FACE_MISMATCH (3-zone) + CONFIRMED_SEAT_SWAP (edge) | High |
| Talking to neighbor | SUSPICIOUS_AUDIO standalone (>10s) | High |
| Looking at neighbor's screen | GAZE_DEVIATION standalone (>30s sustained) | High |
| Phone under desk / in lap | SUSPICIOUS_POSTURE + GAZE → POSSIBLE_MATERIAL_USE (fusion) | Medium-High |
| Notes on desk / lap | SUSPICIOUS_POSTURE + GAZE → POSSIBLE_MATERIAL_USE (fusion) | Medium-High |
| Passed note from outside | GAZE_DEVIATION + possible MULTIPLE_PERSONS | Medium |

**Reliable catches: 7-8/10. Precision over recall.** It is better to detect 8/10 events with <3% FP than to detect 10/10 events with 15% FP. The jury will penalize false positives as much as missed detections.

## Expected Performance

| Metric | Target | Basis |
|--------|--------|-------|
| Face detection accuracy | >95% | BlazeFace with CLAHE preprocessing |
| Face verification accuracy | >90% | MobileFaceNet 99.55% LFW; 3-zone hysteresis reduces FP but may miss some mismatches |
| Gaze deviation (standalone) | 70-80% | 30s sustained threshold = high precision |
| Multi-face detection | 90-95% | BlazeFace multi-face with persistence filtering |
| Audio speech detection (standalone) | 60-70% | 10s sustained speech = high confidence |
| Head-pitch posture (fusion-only) | 65-75% | Pitch >30° for >10s catches phone-in-lap and notes-on-desk |
| **Overall 8/10 target** | **STRONG** | 6 alert types + 3 fusion rules cover 7-8/10 scenarios |
| False positive rate | <3% | Conservative standalone thresholds + persistence filtering |
| Processing FPS | 12-15fps (i5), 8-10fps (Celeron) | INT8 models, 640x480, CLAHE + detect-track-verify |
| Alert latency | <2 seconds | Local processing, no network dependency for detection |

---

## Multi-LLM Debate Consensus (Claude Opus + GPT-5.2 + Gemini 3 Pro)

Full debate reports: `debate-report.md` (Round 1, 3 rounds), `debate-report-r2.md` (Round 2, 5 rounds — CONVERGED), `debate-report-r3.md` (Round 3, 3 rounds).

**Consensus Level: FULL AGREEMENT** — All 3 models reached AGREE by Round 5 of Round 2 (auto-stopped). Round 3 added refinements with near-consensus.

### 49 Points of Unanimous Agreement

**Architecture:**
1. Edge-first, on-terminal inference, no cloud dependency for core functionality
2. **Headless Python agent + Browser UI (HTML/JS)** — NO Electron, NO PyQt6. Agent runs as background process with local HTTP/WS API. Exam UI served as HTML/JS in system browser. Faster to develop, more reliable to deploy on unknown government machines.
3. **3-thread architecture:** Thread 1 (webcam capture → frame queue), Thread 2 (model inference → event queue), Thread 3 (network sender)
4. **Detect → Track → Verify** as the BASELINE architecture (not optimization): BlazeFace 3-5fps → lightweight tracker (MOSSE/CSRT) → MobileFaceNet every 2s or on track loss
5. **CLAHE preprocessing** on every frame before face detection (~2ms, handles uneven lighting)

**Scope Cuts:**
6. **YOLOv8-nano CUT ENTIRELY** — COCO domain mismatch, high CPU cost, FP risk, AGPL licensing
7. **Person Re-ID, bias metrics, adaptive scheduling ALL CUT**
8. No message broker (NATS/MQTT/Redis) — in-memory queues + HTTP/SSE sufficient

**Alert Strategy (Precision Over Recall):**
9. **6 standalone alert types:**
   - CANDIDATE_ABSENT (>5s no face detected)
   - MULTIPLE_PERSONS (>5 consecutive frames with 2+ faces)
   - FACE_MISMATCH (3-zone hysteresis: match <0.45, uncertain 0.45-0.70, mismatch >0.70 sustained 3+ checks)
   - GAZE_DEVIATION (sustained >30s off-center beyond baseline + 30°) — promoted from fusion-only with conservative standalone threshold
   - SUSPICIOUS_AUDIO (sustained speech >10s via Silero VAD) — promoted from fusion-only with conservative standalone threshold
   - SUSPICIOUS_POSTURE (head pitch >30° downward sustained >10s) — detects phone-in-lap/notes-on-desk via Face Mesh pitch angle
10. **Gaze remains a fusion signal too** — standalone fires only at 30s sustained; fusion fires at 5s when combined with other signals
11. **Audio remains a fusion signal too** — standalone fires only at 10s sustained speech; fusion fires at 5s when combined with gaze
12. **3 hardcoded fusion rules:** (1) GAZE + FACE_MISMATCH within ±5s → CONFIRMED_IMPERSONATION; (2) AUDIO + GAZE within ±5s → POSSIBLE_CONSULTATION; (3) SUSPICIOUS_POSTURE + GAZE_DEVIATION within ±5s → POSSIBLE_MATERIAL_USE
13. Every false alarm during demo destroys jury credibility — tune for precision
14. **Rationale for promotion:** With only 3 standalone alerts, the system could reliably catch ~3/10 malpractice scenarios. Promoting gaze and audio to standalone (with conservative thresholds) and adding head-pitch posture detection brings reliable catches to 7-8/10.

**Event Model:**
15. **Episode-based events** (not per-frame spam): terminals emit EPISODE_START/EPISODE_END with stable episode_id
16. **1:1 greedy matching** for confusion matrix: each ground truth event matches at most one episode within ±15s
17. **Common-mode audio rejection at edge:** if ≥3 terminals trigger within 500ms → suppress all but highest-amplitude source

**Dashboard:**
18. **FastAPI + SSE + vanilla JS/HTMX** — Streamlit rejected (full-page rerenders), React rejected (build-chain complexity)
19. SSE sends **delta messages only** (episode start/end + periodic heartbeat per terminal)
20. **LAN-first**, hosted on edge server, cloud optional
21. **Ground truth live logger:** `/admin/ground-truth` page with big buttons per malpractice type, capturing `server_now()` on click (NO manual time entry)

**Infrastructure:**
22. **Mini-PC with SSD** for edge server (not Raspberry Pi + SD card)
23. **SQLite WAL mode** on edge as primary persistence, single async writer queue
24. Terminal-side **JSONL files** as store-and-forward buffer for edge-down resilience
25. **Push-based evidence:** terminal pushes encrypted JPEG to edge via HTTP POST on high/critical alerts only

**Deployment:**
26. **Day-0 packaging spike = existential go/no-go gate.** Test on non-dev target machine.
27. **Deployment decision tree:** Plan A = WinPython portable on USB (no admin needed); Plan B = PyInstaller; Plan C = bootable Linux (only if BIOS confirmed)
28. Include **VC++ Redistributable + DirectX runtime** on USB as insurance

**Reliability:**
29. **App-level time sync:** RTT/2 offset estimation every heartbeat, median of last 5, monotonic clocks for intervals
30. **FPS-based degradation modes:** ≥12fps = full; ≥7fps = shed gaze; ≥4fps = shed audio; <4fps = presence + multi-face only (SURVIVAL mode). Dashboard shows mode per terminal.
31. **Enrollment quality gates:** face confidence >0.9, face >80px, blur/brightness checks, auto-retry with guidance. After 3 failures → "Force Proceed" → terminal flagged DEGRADED
32. **Video file fallback:** `--source` CLI flag (device index or file path)
33. Prepare **10 MCQ questions** as hardcoded JSON on Day 1
34. **Exam start/stop synchronization** via edge broadcast to all terminals

### Round 3 Additions

35. **Browser UI chosen over PyQt6.** Headless Python agent exposes local HTTP/WS API; exam interface is HTML/JS served locally in system browser. Eliminates Qt dependency chain.
36. **Cross-terminal seat-swap detection** — highest-ROI feature. Enrollment embeddings pushed to edge; on FACE_MISMATCH, edge compares against all 30 enrolled candidates; emits CONFIRMED_SEAT_SWAP with both terminal IDs.
37. **CCTV display stub mandatory** — RTSP/USB ingest → display on dashboard + background subtraction motion overlay. No AI. Checks Component 1 compliance.
38. **SUSPICIOUS_POSTURE demoted to fusion-only** — never fires standalone. Only contributes via POSTURE + GAZE → POSSIBLE_MATERIAL_USE.
39. **Audio common-mode rejection at edge** — buffer VAD events 500ms; if ≥3 terminals trigger simultaneously, suppress all but highest-amplitude source.
40. **Hash-chaining re-added** — `hash_i = sha256(hash_{i-1} || canonical_json(event_i))` in JSONL. Directly addresses "tamper-proof logs" requirement.
41. **Malpractice Trigger app** — mobile web page from edge server, 6 buttons + terminal selector. Actors press when staging event → microsecond-accurate ground truth.
42. **Multi-terminal testing moved to Day 4-5** (from Day 8). LAN/SQLite/firewall issues discovered late are project-ending.
43. **640x480 capture resolution** — explicit, non-negotiable on 4GB machines.
44. **Both packaging options prepared** — WinPython portable + PyInstaller `--onedir`. Decide on-site after Day 0 test.
45. **Fallback demo video on Day 9** — 3-min flawless walkthrough, non-optional insurance.
46. **SQLite single-writer async queue** on edge — batch flush every 250ms or 50 pending events. Prevents SQLITE_BUSY under 30-terminal concurrency.
47. **Scalability one-pager** for jury — edge-server-per-hall model, ~₹25K/hall, metadata to district/state over 4G.
48. **Demo in confidence order** — absence → multi-face → seat-swap wow → fusion → confusion matrix live.
49. **Milestone gate** — if Tier 0 not complete by Day 5, cut ALL Tier 2+ items.
