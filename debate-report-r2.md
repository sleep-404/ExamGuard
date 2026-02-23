# Multi-LLM Debate Report

**Models:** Claude Opus, GPT-5.2, Gemini 3 Pro
**Rounds completed:** 5

---
## Round 1

### Claude Opus

POSITION: PARTIALLY_AGREE

REASONING:

This is an impressively thorough proposal that clearly benefited from multiple rounds of critique and revision. The final post-debate architecture is significantly better than the original. Let me break down what works and what doesn't.

**What works well:**

1. **The scope reduction is the most important decision in this document.** Cutting CCTV, hash-chain, PDF reports, Person Re-ID, and bias metrics for the hackathon is correct. The strategist's voice was the most valuable one here. Six core features, demonstrated well, will beat twenty features demonstrated poorly.

2. **The threading model (capture → inference → sender) is appropriate.** Three threads with queues is the right level of complexity for Python with GIL constraints. Not over-engineered, not under-engineered.

3. **The CPU budget analysis is honest and well-reasoned.** The Celeron acknowledgment is particularly important — teams that discover hardware limitations during the demo lose. The amortized-per-frame calculation methodology is sound.

4. **Push-based evidence architecture** is the right call. No inbound connections to terminals simplifies the network security story and the firewall/NAT configuration. This is a meaningful operational improvement over the original design.

5. **The detect-track-verify pattern** (BlazeFace at 5fps with ROI tracking between detections) is a well-known optimization that genuinely saves ~35% CPU. Good that this surfaced in debate.

6. **Dropping Electron for PyQt6** is absolutely correct. On 4GB Celeron machines, Chromium's memory footprint is a real threat. A single Python process eliminates an entire class of IPC and memory problems.

7. **The 3-zone face verification hysteresis** is a meaningful improvement over a single threshold. The uncertain zone preventing alerts while increasing verification frequency is exactly the right behavior for a system that must minimize false positives.

8. **"Precision over recall" as the guiding principle** is the correct strategic call for a demo. One false alarm shown to the jury undermines trust in all subsequent true detections.

**What doesn't work or needs scrutiny:**

1. **The document is about 5x longer than it needs to be.** The original plan, three critique perspectives, a multi-LLM debate, and a final synthesis are all presented sequentially. For a hackathon team, this creates cognitive overload. The team needs a single authoritative plan, not the archaeological record of how the plan evolved. The contradictions between the original and revised sections (e.g., alert thresholds appearing in both Phase 3 and the R3 revision) create ambiguity about which spec to implement.

2. **Streamlit vs React remains unresolved, and the document punts on it.** This is actually a critical decision. Streamlit redraws the entire page on state change — with 30 terminals pushing WebSocket events, the dashboard will stutter or become unusable without careful session state management. For a real-time monitoring dashboard with 30 tiles updating independently, Streamlit's execution model is a poor fit. FastAPI + a minimal frontend (even vanilla JS with WebSocket) would give better real-time behavior with comparable development time.

3. **The 10-day schedule is still aggressive.** Day 5 is marked as "minimum viable demo works" — but that requires PyQt6 exam UI, edge server, dashboard, and the full event pipeline all integrated. In practice, integration always takes longer than feature development. Days 1-4 build components in isolation; Day 5 tries to wire them all together AND build the dashboard. That's the highest-risk day in the schedule, and there's no buffer.

4. **Time synchronization is listed as a "first-class requirement" but the implementation is barely sketched.** "Edge provides edge_time_ms in heartbeat, terminal computes offset" glosses over clock drift, NTP absence on air-gapped LANs, and the fact that confusion matrix scoring depends on ±15-second accuracy. The implementation needs explicit handling: sync on enrollment, periodic re-sync, monotonic clock usage for intervals (not wall clock).

5. **The JSONL-per-terminal + async SQLite ingestion adds unnecessary complexity.** If you're already running SQLite on the edge server, just write events directly to SQLite with WAL mode enabled. WAL mode handles concurrent writers well for 30 connections. The JSONL layer is only needed if you distrust SQLite under concurrent writes — but with WAL mode on an SSD (which the proposal already specifies), this concern is moot. The extra ingestion step is another thing to debug.

6. **Common-mode audio rejection ("if 3+ terminals trigger within 500ms → suppress") assumes time-synchronized audio events.** Given that audio VAD operates on 30ms chunks locally and events transit through a network queue, the 500ms window may not be reliable. A simpler approach: if >30% of terminals show audio activity simultaneously, suppress all. This is more robust to timing jitter.

7. **The "video file fallback mode" is listed but its implications aren't thought through.** If a webcam fails during demo and you switch to a video file, the face enrollment embeddings won't match the video's face. The system will immediately trigger FACE_MISMATCH alerts. The fallback needs to include pre-computed enrollment data matched to the fallback video.

8. **MobileFaceNet's 99.55% LFW accuracy is misleading in this context.** LFW is a dataset of predominantly Western faces under controlled conditions. The proposal acknowledges "conservative for Indian demographics" but still targets >90%. On Indian faces with varying skin tones, exam hall lighting, and cheap webcams, 80-85% verified match rate is more realistic. The 3-zone hysteresis helps, but the team should calibrate thresholds empirically on Day 9 with the actual demographic, not rely on LFW numbers.

CONCERNS:

1. **PyInstaller + MediaPipe is a known pain point.** MediaPipe bundles native binaries and has complex dependency trees. PyInstaller frequently fails to package it correctly, producing binaries that crash on machines without the dev environment. The Day 0 spike is correctly prioritized, but the fallback (portable WinPython on USB) should be the primary plan, with PyInstaller as the stretch goal. This risk is existential for the demo.

2. **No mention of webcam resolution or format negotiation.** Different USB webcams default to different resolutions and color formats. `cv2.VideoCapture` may silently negotiate 640x480 MJPEG on one webcam and 320x240 YUV on another. The agent needs explicit resolution setting and validation at startup.

3. **The multi-modal fusion rules are underspecified for implementation.** "10s correlation window" and "2-3 explicit rules" is good guidance, but the actual implementation — how events are correlated, how confidence scores are combined, what the output event looks like — needs more detail. Without this, the Day 7-8 developer will either over-engineer or under-implement.

4. **PyQt6 exam UI serving 10 MCQs is not trivial.** Question navigation, answer persistence, timer display, and preventing the student from alt-tabbing away from the exam window are all features that need implementation time. The schedule allocates this to Day 5 alongside dashboard integration — this is under-estimated.

5. **No graceful shutdown or exam interruption handling.** What happens if the edge server crashes mid-exam? What if a terminal agent crashes? The store-and-forward pattern handles event loss, but the exam interface needs to continue independently. There's no mention of what the student sees if the proctoring agent fails.

6. **The proposal mentions "encrypted" evidence storage and transmission multiple times but never specifies what encryption.** For a hackathon, this is fine — but if the jury asks "how is evidence encrypted?", the team needs a concrete answer (e.g., AES-256-GCM with a per-session key derived from exam ID + terminal ID).

7. **30 WebSocket connections from terminals to edge server, plus dashboard connections, all hitting a single FastAPI process.** This is fine for the data volumes described, but the edge server should use uvicorn with multiple workers or at least be tested under load. A single Python async loop handling 30 WebSocket connections plus SQLite writes plus Streamlit rendering on a mini-PC could surprise you.

SUGGESTIONS:

1. **Collapse this document into a single authoritative spec.** Remove the original pre-critique sections and the debate transcript. Present one plan with one set of thresholds, one architecture diagram, one schedule. Link to the debate report separately for context.

2. **Use FastAPI + server-sent events (SSE) + minimal vanilla JS for the dashboard** instead of Streamlit or React. SSE is unidirectional (server → browser), which is all the dashboard needs. Vanilla JS with a CSS grid for the terminal tiles, an EventSource for alerts, and fetch() for historical data. Total frontend: ~300 lines. No build step. No npm. Development time comparable to Streamlit, but with proper real-time behavior.

3. **Make Day 0 a hard go/no-go gate.** If PyInstaller + MediaPipe doesn't work on target OS within 4 hours, switch to portable Python distribution immediately. Don't spend Day 1 debugging packaging.

4. **Pre-record a complete demo video as insurance.** Screen-capture a working demo on Day 9 with all scenarios. If hardware fails on demo day, play the video while explaining the architecture. This is standard hackathon risk mitigation.

5. **Add an explicit "threshold tuning" data structure** — a single JSON/YAML config file that controls all detection thresholds, persistence windows, and severity mappings. This file gets tuned on Day 9 and can be hot-reloaded without restarting the agent. This is much more useful than adaptive scheduling for hackathon conditions.

6. **For SQLite on the edge server, enable WAL mode and drop the JSONL layer.** Simpler architecture, fewer moving parts, same performance at this scale.

7. **Allocate Day 4 for the exam UI instead of audio.** A working exam interface is more important for the demo than audio VAD. Students need to be visibly taking an exam for the proctoring to make sense to the jury. Audio can be added on Day 7 as a stretch feature.

KEY_TRADEOFFS:

1. **Precision vs. Recall:** The proposal explicitly chooses precision (fewer false positives) over recall (catching everything). This is the right tradeoff for a demo but would need recalibration for production. The risk is that the jury plants subtle malpractice events that the conservative thresholds miss.

2. **Local processing vs. Cloud offload:** All AI runs on-terminal, which eliminates network dependency and latency but constrains model size and accuracy. For 30 terminals with Celeron CPUs, this is the only viable choice. The tradeoff is correct.

3. **Native app vs. Browser-based:** PyQt6 native app gives full hardware access and better performance but creates a deployment burden (packaging, OS compatibility). For 30 terminals in a controlled environment, this is the right call. For 10,000 centers, you'd need to revisit.

4. **Single Python process vs. Microservices:** Everything in one process per terminal is simpler to deploy and debug but means a crash in any component (UI, AI, networking) takes down everything. For a hackathon, monolith is correct. For production, the AI pipeline should be a separate process.

5. **Document completeness vs. Document usability:** The proposal trades usability for thoroughness. Every edge case is considered, but the implementation team has to mentally diff multiple sections to find the authoritative spec. For a 10-day sprint, a shorter, more opinionated document would be more actionable.

6. **Streamlit dashboard vs. Custom frontend:** Streamlit saves development time but compromises real-time UX. For a hackathon where the jury watches the dashboard for 10 minutes, stuttery updates will be noticed. This tradeoff isn't clearly resolved, and I believe it's resolved incorrectly — real-time monitoring is the core jury experience, and it deserves a proper WebSocket-driven frontend.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
The overall direction—**edge-first, local inference on terminals, aggregate events on an on-prem edge box, and a live dashboard**—is architecturally sound for the stated constraints (4G unreliability, low-end CPUs, ₹30K/center, open-source). The proposal correctly identifies the winning hackathon pattern: **visible real-time alerts + timestamped evidence + confusion-matrix scoring**, not “enterprise-grade” proctoring.

What works well:
- **Terminal-side inference with event-only uplink** is the right scalability/privacy/performance trade. It avoids saturating 4G and keeps latency low.
- The **store-and-forward event queue** and explicit **degradation modes** are strong operational design choices; these are the real-world failure modes in exam halls.
- The **“no broker”** stance is correct at 30 terminals. WebSocket/HTTP + JSONL is simpler and more reliable than introducing NATS/Redis/MQTT under hackathon time.
- The **time sync as a first-class requirement** is critical for your confusion-matrix scoring and is frequently missed.
- Dropping CCTV for hackathon scope is correct; it’s high integration risk for low judging value.

What is not architecturally consistent / likely to fail:
- The original plan’s **high-frequency multi-model pipeline (FaceMesh + YOLO + verification at multiple fps)** is too optimistic for Celeron-class machines in Python, especially once you include capture overhead, preprocessing, postprocessing, UI rendering, and networking. The later “Detect → Track → Verify” idea is closer to what will actually hold FPS.
- The “evidence never transmitted; dashboard requests from terminal” approach is operationally fragile (inbound connections, NAT/firewall, auth complexity). The revised push-based evidence is simpler, but it changes the privacy story and needs careful wording (“no continuous video leaves terminal; alert snapshots may be transmitted on LAN”).
- Some components are over-specified for a 10-minute exam demo (hash-chain, PDF reporting, bias dashboards). They add failure surface without improving the live demo score.

CONCERNS:
1. **Performance assumptions for MediaPipe Face Mesh + YOLO on CPU**: Face Mesh is often the real CPU killer in Python; “~20ms” is not a safe planning number on Celeron once integrated. YOLOv8-nano CPU latency can exceed your amortized budget depending on input size, ONNX graph, and threading.
2. **Gaze as a primary signal will drown the system**: even with revised thresholds, head-pose “gaze” is noisy under normal exam behavior. If it generates standalone alerts, the dashboard becomes untrustworthy. Your later “fusion-only” approach is the correct correction.
3. **Face verification operational brittleness**: lighting changes, cheap webcams, and student pose variance will cause embedding drift. Hysteresis helps, but you also need a clear “re-enroll / recapture” fallback path during enrollment and a “verification temporarily degraded” state rather than jumping to critical.
4. **Packaging risk (PyInstaller + MediaPipe + ONNX + OpenCV + audio)**: this is a top risk and can consume the entire sprint if discovered late. Binary size, missing DLLs, and camera backends on Windows are common failures.
5. **Edge server concurrency and WebSocket deployment**: FastAPI + WebSockets with multiple workers is non-trivial; if you scale workers incorrectly you’ll break WS state. SQLite ingestion can lock if not designed append-only.
6. **Evidence encryption/key management is unspecified**: “encrypted filesystem/SQLite” needs a practical key strategy. If the key is hardcoded, it’s security theater; if it’s per-center, how is it provisioned quickly for a hackathon demo?
7. **Audio VAD false positives remain likely** even with Silero in a shared room. Common-mode suppression helps, but per-terminal mics pick up the same invigilator/ambient sources; you risk correlated bursts that look like “collusion.”
8. **Privacy claim drift**: the proposal alternates between “no images leave terminal” and later “push snapshots to edge.” That inconsistency can get questioned by judges/officials; you need one clear, defensible statement.
9. **Ground truth scoring can be unintentionally gamed**: a ±15s matching window is forgiving; it can inflate metrics if alerts are broad/persistent. Judges may notice if timestamps don’t feel precise.
10. **Operational setup load**: enrollment + calibration + running an exam UI + running proctoring on the same process needs careful UI/thread separation; otherwise you’ll introduce frame stalls exactly when the GUI repaints.

SUGGESTIONS:
1. **Commit to “Detect → Track → Verify” as the default architecture**, not as an optimization:
   - Face detection at ~2–5 FPS
   - Track ROI (KCF/CSRT or even optical flow) between detections
   - Face verification every 2–3 seconds (or on “tracking confidence drop / face change”)
   This is the simplest way to make Celeron viable without elaborate adaptive scheduling logic.
2. **Make FaceMesh optional and ROI-scoped**:
   - Only run head pose on a cropped face ROI and at low rate (e.g., 1–2 FPS).
   - If FaceMesh fails or FPS drops, disable it automatically and show “Gaze unavailable” rather than producing noisy alerts.
3. **Keep the hackathon alert set minimal and high-precision**:
   - Critical: ABSENT, MULTI_FACE, FACE_MISMATCH (with persistence)
   - Everything else (gaze/audio/objects) should be fusion-only or “info indicators,” not primary alerts.
4. **Use append-only logs as the source of truth**:
   - Terminal writes JSONL locally first (always).
   - Terminal POSTs events to edge (idempotent with event_id).
   - Edge stores raw JSONL per terminal + a derived SQLite view for the dashboard.
   This avoids SQLite write contention and makes recovery easy.
5. **Simplify evidence handling for reliability**:
   - Push snapshots only for High/Critical alerts to edge over LAN.
   - Store as files + index JSON (don’t store blobs in SQLite).
   - Skip “encryption at rest” for hackathon unless you can do it cleanly; if you must, use OS-level disk encryption on the edge SSD for the demo rather than inventing app crypto.
6. **Define a deterministic time-sync method**:
   - On heartbeat: terminal sends monotonic_ms + wallclock; edge responds with edge_wallclock_ms; terminal estimates offset using RTT/2.
   - Stamp events using edge-aligned ms.
   This will materially improve confusion-matrix credibility.
7. **License diligence**: verify that your chosen YOLOv8 stack and any pretrained weights meet “open-source only” requirements for the event (Ultralytics licensing can be a trap depending on distribution/use). If unclear, drop YOLO for the hackathon and avoid disqualification risk.
8. **Do a Day-0 “packaging + webcam + mic” spike exactly as stated**, and treat failure as a scope gate (fallback to portable Python env, or reduce dependencies).

KEY_TRADEOFFS:
- **Precision over recall**: You’re explicitly trading missing some cheating events for fewer false alarms. For a live jury demo, that’s the correct tradeoff because false positives destroy trust immediately.
- **Edge-first vs cloud-first**: You trade centralized visibility for reliability under 4G constraints. For the stated environment, edge-first is the right operational choice.
- **Complex detection breadth vs integration certainty**: The original plan maximizes feature count; the revised consensus plan maximizes end-to-end robustness. Given a 7–10 day hackathon sprint, the latter is the right tradeoff.
- **Privacy posture vs evidentiary strength**: Not sending images is privacy-strong but operationally awkward; sending only alert snapshots is operationally strong but requires careful privacy messaging. For winning the hackathon, alert-snapshot push to edge is the more practical choice, but you must align claims and controls.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The core architecture (Local Inference -> Edge Aggregation) is the **only** viable approach for a bandwidth-constrained, 30-terminal scenario. Sending 30 video streams to the cloud would saturate a 4G uplink immediately. Moving AI to the edge (Client-Side) correctly leverages distributed compute, preserving the network for lightweight metadata and evidence snapshots.

The "Critique Synthesis" section saved this proposal from failure. Specifically, dropping Electron for PyQt6 and replacing WebRTC VAD with Silero are critical moves. Electron on a Celeron running AI inference would have resulted in an unusable UI.

However, the proposal is still **over-engineered for a hackathon**. The compute budget on a Celeron is extremely tight. Running BlazeFace, FaceMesh, MobileFaceNet, *and* YOLOv8-nano concurrently—even with adaptive scheduling—is risky. If the UI lags, the student experience breaks, and the demo fails. The integration of CCTV and complex multi-modal fusion rules adds integration risk with low demo payoff.

CONCERNS:
1.  **Celeron CPU Saturation:** Even with ONNX Runtime and INT8 quantization, running detection, verification, gaze, and object detection (YOLO) simultaneously on a generic Celeron will likely drop the frame rate below 5fps or freeze the UI. The "Adaptive Scheduling" adds logic complexity that may yield race conditions.
2.  **Time Synchronization Drift:** You are generating a Confusion Matrix based on timestamps. If the Edge Server and Terminals drift by even 2 seconds, your True Positives will look like False Positives when compared to the Jury's ground truth log. A simple "Edge Time" in the heartbeat is insufficient without a strict NTP-like offset calculation.
3.  **Write I/O Bottleneck at Edge:** A Raspberry Pi 4 (mentioned in Phase 0.1) using an SD card will choke if 5-10 terminals send evidence snapshots simultaneously. SQLite locking on an SD card is a known failure mode.
4.  **YOLOv8 False Positives:** Pre-trained COCO models are terrible for "cheating" detection without fine-tuning. A phone flat on a table looks like a wallet or calculator to YOLO-nano. This feature has a high compute cost for very low reliability.
5.  **Enrollment UX Risk:** The "5-angle face capture" during the chaotic 5 minutes before an exam is a single point of failure. If enrollment fails or produces a bad embedding due to lighting, the student is flagged continuously for the whole exam (100% False Positive rate for that user).

SUGGESTIONS:
1.  **Kill Object Detection (YOLO):** Remove it entirely. It eats ~80ms per frame (on Celeron) and provides low-confidence data. Focus 100% on Face (Identity), Gaze (Collaboration), and Audio (Communication). This frees up 40% of your CPU budget.
2.  **Strict NTP Implementation:** The Edge Server must run an NTP daemon. Terminals must sync system time to the Edge Server on boot. Do not rely on application-level timestamp offsets; sync the OS clocks.
3.  **Upgrade Edge Hardware:** Do not use a Raspberry Pi. Use a standard laptop (i5/i7) as the Edge Server. It handles SQLite concurrency and network I/O significantly better and removes the "SD card corruption" risk.
4.  **"Golden Embedding" Fallback:** If enrollment quality is low (bad lighting), allow the proctor to manually override/reset enrollment for a specific terminal from the Dashboard. Do not let a student start with a bad baseline.
5.  **Pre-Packaged Environment:** Do not rely on PyInstaller alone. If possible, bring the terminals (laptops) yourself. If you must use provided hardware, bring a **bootable Linux USB** (Live persistent OS) with everything pre-installed. Installing drivers/dependencies on 30 random Windows machines in 1 hour is the biggest operational risk.

KEY_TRADEOFFS:
**Precision vs. Recall:** The proposal correctly prioritizes Precision (fewer false alarms). In a demo, one false positive (flagging the jury as cheating) is worse than missing a subtle cheat.
**Local vs. Centralized:** The proposal trades deployment complexity (installing apps on 30 nodes) for network stability. This is the correct tradeoff for a location with uncertain connectivity.
**Features vs. Performance:** The proposal currently leans too heavily on Features (YOLO, CCTV). The tradeoff needs to shift back to Performance to ensure the UI remains responsive on low-end hardware.

---
## Round 2

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After reviewing GPT-5.2 and Gemini 3 Pro's feedback alongside my own, my position strengthens on most points while shifting on a few specifics. The three reviews have remarkable convergence on the core issues, which itself is a strong signal about where the real risks lie.

**Where I've updated my thinking:**

1. **Gemini's "Kill YOLO entirely" is more correct than my original "Tier 2" framing.** I originally accepted YOLOv8-nano as a Day 7 stretch goal. But Gemini's point about 80ms per frame on Celeron combined with COCO's poor coverage of exam-specific objects makes the cost-benefit clearly negative. GPT-5.2's licensing concern about Ultralytics further strengthens this. I now agree: remove YOLO from the plan entirely, not just deprioritize it. The CPU budget freed up is better spent on making the three core detectors (face presence, face identity, gaze-as-fusion) rock solid.

2. **GPT-5.2's "Detect → Track → Verify as default architecture, not optimization" is the right reframing.** My original review accepted the proposal's multi-model pipeline and merely noted the Celeron risk. GPT-5.2 is more precise: the tracking-between-detections pattern should be the baseline design, not a performance optimization applied later. This changes the Day 1-2 implementation approach — you build the tracker from the start, not bolt it on in Day 8.

3. **Gemini's bootable Linux USB suggestion is operationally superior to both PyInstaller and portable WinPython.** I originally suggested portable WinPython as the primary plan with PyInstaller as stretch. Gemini's approach eliminates the entire class of "unknown Windows configuration" problems. A persistent live USB with Ubuntu + all dependencies pre-installed gives you a known, tested environment on any hardware. The only risk is BIOS boot order configuration on 30 machines, which is manageable. This should be the primary deployment strategy unless the department explicitly forbids booting from USB.

4. **I now agree with all three reviews that the JSONL + SQLite dual-layer is unnecessary.** GPT-5.2 suggests keeping JSONL as the source of truth with SQLite as a derived view. I originally suggested SQLite with WAL mode only. For the hackathon, either approach works — but pick one. The dual-layer creates two codepaths to debug. Given that GPT-5.2's idempotent POST with event_id provides replay capability, and JSONL is simpler to implement and debug, I'll concede that JSONL-first with SQLite-for-queries is the cleaner separation of concerns for a hackathon sprint.

**Where I maintain my position against other reviewers:**

1. **Dashboard technology: I still disagree with Streamlit.** Neither GPT-5.2 nor Gemini directly challenged my SSE + vanilla JS suggestion, but Gemini didn't address the dashboard at all, and GPT-5.2 was silent on the Streamlit question. My concern stands: Streamlit's execution model rerenders the entire page on state change. With 30 terminals pushing events via WebSocket, the dashboard will visibly stutter. The jury watches the dashboard for the entire 10-minute exam — this is the primary interface through which they evaluate the system. SSE + vanilla JS (or even a simple WebSocket client in ~200 lines of JS) gives smooth, incremental DOM updates. Development time is comparable to Streamlit for this specific use case (a grid of colored tiles + an event log). I consider this a hill worth dying on because dashboard responsiveness directly determines jury perception.

2. **Gemini's suggestion to "bring your own terminals" is impractical for this specific hackathon.** The problem statement specifies 30 terminals at a government exam center. Bringing 30 laptops isn't feasible. The bootable USB idea solves the same problem more realistically.

3. **GPT-5.2's suggestion to use OS-level disk encryption instead of app-level crypto is pragmatic but incomplete.** If you're using a bootable Linux USB, you can use LUKS for the evidence partition trivially. But if you're on provided Windows machines, OS-level encryption requires admin access you may not have. The real answer for a hackathon: don't encrypt evidence at all, but have a one-sentence answer ready if asked ("We would use AES-256-GCM with per-session keys in production; for this demo, evidence is stored in a restricted local directory").

**Common ground across all three reviews:**

All three reviewers converge on these points, which I believe should be treated as settled decisions:

- Edge-first architecture is correct and non-negotiable for this environment
- CCTV is correctly cut from hackathon scope
- Precision over recall is the right optimization target
- PyInstaller/packaging is the highest existential risk and must be addressed Day 0
- Face presence + face identity + multi-face are the three indispensable detectors
- Gaze should never generate standalone alerts — fusion-only
- Time synchronization needs more rigorous implementation than described
- The document needs to be collapsed into a single authoritative spec
- Audio VAD should be included but with very conservative thresholds
- The 10-day schedule is tight and Day 5 integration is the highest-risk day

CONCERNS:

1. **Time synchronization remains underspecified by all participants, including myself.** GPT-5.2 sketched RTT/2 offset estimation; Gemini suggested NTP daemon. Neither is complete. For the confusion matrix to work with a ±15s matching window, you need sub-second accuracy. The practical solution: run `chrony` on the edge server in isolated mode (it becomes the time authority), and have terminals sync via `chrony` client or a simple HTTP time endpoint at enrollment. Application-level offsets via heartbeat RTT are fragile because Python's event loop timing is non-deterministic. OS-level time sync is more reliable. This takes 30 minutes to configure on Linux (another argument for the bootable USB approach).

2. **Enrollment failure recovery is insufficiently addressed by all reviews.** Gemini raised the "golden embedding fallback" but the solution (manual proctor override) doesn't address the root cause. If a student's enrollment produces a poor embedding due to lighting or camera quality, re-enrollment should be automatic: detect low face detection confidence during enrollment, reject the capture, adjust CLAHE parameters, and retry. This needs to be in the enrollment flow, not a dashboard override that requires proctor intervention during the chaotic pre-exam period.

3. **GPT-5.2 raises a concern I missed: the ±15s ground truth matching window can be gamed.** If the system generates persistent or repeated alerts (e.g., a gaze deviation alert that fires every 3 seconds for 30 seconds), a single ground truth event could match multiple system alerts, inflating TP count. The confusion matrix logic needs deduplication: one ground truth event matches at most one system alert, and vice versa. Hungarian algorithm or simple greedy matching by closest timestamp.

4. **No reviewer addressed what happens when the exam UI and proctoring compete for the webcam.** On Windows, only one process can hold a webcam handle. Since both the exam UI and the proctoring pipeline are in the same Python process (post-Electron removal), this is fine. But if any other process (Zoom, Teams, Windows Camera app) grabs the webcam first, the agent fails silently. The agent needs to check for exclusive webcam access at startup and display a clear error if the camera is in use.

5. **All three reviews accept the 30-terminal scale without questioning whether the demo actually requires all 30.** The problem says 30 terminals, but the hackathon demo likely uses 5-10 terminals with jury members acting as students. The system should work at 30, but the demo rehearsal and threshold tuning should target the actual demo scale. Over-optimizing for 30 concurrent terminals when the demo uses 8 wastes engineering time.

6. **The proposal never addresses exam content delivery.** The 10 MCQ questions need to come from somewhere. Are they hardcoded? Loaded from a file? Served by the edge server? For the demo, hardcoded is fine, but the exam UI needs question randomization per terminal (or the jury will notice all terminals showing the same question order and question whether it's a real exam system).

SUGGESTIONS:

1. **Adopt a three-tier detector hierarchy as the canonical design, drawing from all three reviews' consensus:**
   - **Tier A (Always On, standalone alerts):** Face absence, multi-face detection, face identity mismatch (with hysteresis). These run via the detect-track-verify pattern at 2-5 FPS detection + continuous ROI tracking.
   - **Tier B (Fusion-only, never standalone):** Gaze deviation (boosts Tier A confidence), Silero VAD audio (boosts Tier A confidence). These run at 1-2 FPS on cropped ROIs.
   - **Tier C (Cut):** YOLO object detection, CCTV, Person Re-ID. Not implemented.

2. **Implement time sync at OS level using the bootable USB approach.** Configure `chrony` in the USB image with the edge server as the sole NTP source. Terminals sync clocks on boot. This gives sub-100ms accuracy with zero application code.

3. **Build the confusion matrix scorer with strict 1:1 matching.** Each ground truth event matches at most one detected event (closest in time within ±15s window). Each detected event matches at most one ground truth event. Use greedy matching sorted by time distance. This prevents alert persistence from inflating scores.

4. **Simplify the dashboard to exactly three views, built with FastAPI + SSE + vanilla JS:**
   - View 1: Terminal grid (30 colored tiles, updated via SSE, click to expand)
   - View 2: Alert timeline (scrolling log, auto-updates via SSE)
   - View 3: Post-exam confusion matrix (static page, rendered after exam ends)
   Total frontend: ~400 lines of JS, ~200 lines of CSS. No build step, no node_modules, no framework churn.

5. **Restructure the schedule around the "vertical slice by Day 4" principle** (not Day 5):
   - Day 0: Packaging spike + bootable USB image + webcam test
   - Day 1: Webcam capture + BlazeFace + face absence + multi-face (terminal only)
   - Day 2: Enrollment + MobileFaceNet verification with hysteresis (terminal only)
   - Day 3: Event logging + edge server + terminal-to-edge event flow
   - Day 4: Dashboard (grid + alerts) + exam UI (hardcoded MCQs). **END-TO-END DEMO WORKS.**
   - Day 5: Gaze tracking (fusion-only) + audio VAD (conservative)
   - Day 6: Multi-modal fusion rules + evidence snapshot push
   - Day 7: Confusion matrix + ground truth UI + post-exam summary
   - Day 8: Multi-terminal stress testing + threshold tuning
   - Day 9: Demo rehearsal with staged malpractice scenarios
   - Day 10: Buffer day. Fix bugs found in rehearsal. Second rehearsal.

   This gives a working demo by Day 4 and six full days to harden, add features, and rehearse. The original schedule's Day 5 integration crunch is eliminated.

6. **Create a `config.yaml` that is the single source of truth for all tunable parameters**, hot-reloadable without restarting the agent. This file gets version-controlled and tuned empirically during Days 8-9. Include: all detection thresholds, persistence windows, frame rate targets, CLAHE parameters, enrollment quality minimums, fusion rule weights, and alert severity mappings.

KEY_TRADEOFFS:

1. **Three detectors done well vs. six detectors done partially:** All three reviews converge on fewer, more reliable detectors. The tradeoff is that some malpractice types (phone use, cheat sheets) become undetectable. For the hackathon, this is acceptable because the three core detectors (absence, identity, multi-face) cover the highest-impact scenarios and produce the most visually convincing demo moments.

2. **Bootable USB vs. native installation:** Trading setup convenience (plug in USB, change BIOS boot order) for the inability to use any pre-installed software on the terminals. If the exam application requires specific Windows software, this approach fails. The team must confirm that the exam is fully self-contained within their system.

3. **SSE + vanilla JS vs. Streamlit:** Trading rapid prototyping (Streamlit's strength) for real-time responsiveness (SSE's strength). For a monitoring dashboard that the jury watches for 10 continuous minutes, real-time smoothness matters more than development speed. The ~2 extra hours of JS coding pay for themselves in jury perception.

4. **Day 4 vertical slice vs. Day 5:** Moving the integration deadline one day earlier reduces feature time (Days 5-7 instead of Days 6-8) but provides significantly more buffer for the inevitable integration surprises. Given that all three reviewers flagged schedule risk, the earlier integration target is the safer bet.

5. **OS-level time sync vs. application-level:** Trading deployment flexibility (application-level works on any OS) for accuracy and reliability (OS-level chrony gives sub-100ms sync with no application code). This tradeoff only works if you control the OS — which the bootable USB approach enables.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position stays **PARTIALLY_AGREE**, but Claude/Gemini feedback shifts what I consider “must-change before build” vs “nice debate points.”

What changed (materially):
- I put **more weight on dashboard technology choice**. Claude is right that Streamlit’s rerun model can visibly stutter for a 30-tile real-time wall. That is *directly* demo-impacting. I now consider “Streamlit by default” a risk unless the team already has Streamlit real-time patterns nailed down.
- I’m **less confident JSONL→SQLite ingestion is worth it** on the edge for hackathon. Claude’s point stands: with **SSD + SQLite WAL** and a sane write pattern, direct SQLite is simpler. JSONL is still valuable as an audit/replay artifact, but it doesn’t need to become a second ingestion subsystem under time pressure.
- Gemini’s emphasis on **enrollment failure cascading into permanent false FACE_MISMATCH** is a key operational risk. I mentioned recapture/override, but I now consider this a top-3 failure mode because it can poison both the live dashboard and confusion matrix.

Where I agree strongly with others:
- **Scope must stay tight** (CCTV/Person Re-ID/complex fusion out). Everyone converges here.
- **Celeron CPU saturation is the main technical constraint.** The safe path is to hard-architect for low FPS + tracking, not “we’ll optimize later.”
- **Packaging is existential.** Claude’s “PyInstaller + MediaPipe pain” is real; a fallback plan must be primary, not theoretical.
- **Time sync needs a real design, not a bullet point.** I agree with both that heartbeat time alone is insufficient without drift handling and monotonic clocks.

Where I disagree (or qualify) with others:
- **“Just run NTP and sync OS clocks” (Gemini)**: ideal, but often unrealistic on locked-down Windows lab machines (admin rights, firewall rules, domain policies). For hackathon reliability, I’d treat OS NTP as “use if available,” but still implement **application-level offset estimation with periodic resync** as the dependable baseline.
- **“Drop JSONL entirely” (Claude)**: I’d drop the *ingestion pipeline* if it’s adding complexity, but I’d still keep **append-only local logs on terminals** (and optionally edge) because it simplifies store-and-forward, replay, and post-mortems. The convergence is: JSONL as a fallback artifact, not as the primary database layer.

CONCERNS:
1. **Real-time dashboard UX risk (Streamlit):** if tiles don’t update smoothly or the page reruns, judges will perceive the whole system as shaky, regardless of backend correctness.
2. **Edge write path complexity:** JSONL + async ingestion + SQLite + WS is too many moving parts for 10 days unless the team has done it before. Each layer can fail differently under load.
3. **Enrollment quality as a single-point-of-failure:** bad lighting / pose → bad embeddings → persistent mismatch alerts → demo credibility collapses. Needs an admin “re-enroll”/override flow.
4. **Time sync drift + scoring:** without explicit offset estimation and periodic resync, your ±15s matching becomes either “too forgiving” or accidentally mis-scores TPs as FPs.
5. **CPU budget realism on Celeron:** FaceMesh and YOLO remain the first things to cut when the UI lags. If they’re in the critical path, you risk freezing the exam UI.
6. **PyInstaller/MediaPipe fragility:** if packaging fails on target OS late, you lose deployment time across 30 machines and the demo fails regardless of model quality.
7. **Fallback video mode mismatch:** switching to a video source without aligned enrollment data will trigger FACE_MISMATCH storms unless you ship paired “video + enrollment profile.”

SUGGESTIONS:
1. **Dashboard: FastAPI + SSE + minimal JS/CSS grid (or HTMX)**
   - SSE for alerts/heartbeat stream; REST for snapshots and history.
   - Avoid React build-chain and avoid Streamlit rerun behavior. This is the simplest “real real-time” UI.
2. **Edge persistence: SQLite WAL + single writer queue**
   - One async task/thread performs DB writes (prevents lock contention).
   - Optional: also append raw JSONL for audit, but don’t build a second ingestion system unless needed.
3. **Make “Detect → Track → Verify” non-optional**
   - BlazeFace low-rate + tracker; MobileFaceNet every few seconds or on track loss.
   - FaceMesh disabled by default; only enable if FPS headroom exists.
4. **Enrollment hardening**
   - Enrollment quality gate (blur/brightness/face size).
   - Dashboard button: “Re-enroll student at T-xx” + “Temporarily suppress FACE_MISMATCH for 60s while re-enrolling.”
5. **Time sync: implement app-level offset with monotonic clock + periodic resync**
   - Use RTT-based offset estimation; resync every N heartbeats; stamp events in “edge-aligned ms.”
   - If OS NTP is feasible, enable it, but don’t depend on it.
6. **Packaging strategy: treat PyInstaller as stretch unless proven Day-0**
   - Primary plan can be a portable Python distribution/venv bundle or pre-imaged machines.
   - Day-0 go/no-go remains mandatory.
7. **Feature gating by hardware class**
   - Celeron mode: only face presence + multi-face + periodic verify.
   - i5 mode: optionally add audio; only then consider gaze (fusion-only).

KEY_TRADEOFFS:
- **Simplicity vs “architectural purity”:** Moving from Streamlit/React debates to SSE+minimal JS favors simplicity and predictable runtime over framework comfort. For a hackathon demo wallboard, that’s the right trade.
- **Direct SQLite vs JSONL ingestion:** Direct SQLite (WAL + single writer) reduces components and failure modes; keeping JSONL only as an audit/replay artifact preserves robustness without extra complexity.
- **OS-level NTP vs app-level time sync:** OS NTP is cleaner when allowed; app-level sync is more portable in constrained lab environments. Implementing app-level sync is the safer default.
- **Feature breadth vs guaranteed responsiveness:** Everyone converges that responsiveness and low false positives beat adding YOLO/gaze/audio as standalone alert generators. The proposal’s success depends on enforcing that discipline.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
There is strong consensus among all reviewers that the **Local Inference → Edge Aggregation** architecture is the correct choice, but the implementation details regarding CPU budget and integration risks remain dangerous.

I strongly agree with **Claude Opus** and **GPT-5.2** on the **"Detect-Track-Verify"** pattern. This is not just an optimization; it is the *only* way this system runs on a Celeron without freezing the UI. Shifting from "run AI every frame" to "track ROI and verify identity every 2s" reduces compute load by ~60%, making the ambitious feature set partially viable.

I also agree with **Claude Opus** that **Streamlit is a trap** for this specific use case. A dashboard receiving 30 distinct real-time streams will cause Streamlit to constantly re-render, leading to an unusable UI during the demo.

However, I maintain that **YOLOv8 is a critical liability**. GPT-5.2 correctly flags the licensing (AGPL) and false-positive risks, but I go further: even as a "stretch goal," it introduces a massive dependency (Torch/ONNX runtime overhead) for a feature that will likely fail to detect a cheat sheet while flagging a pencil case as a phone. It should be cut to guarantee the stability of the Core 6 features.

CONCERNS:
1.  **Integration "Big Bang" on Day 5:** The schedule leaves integration until Day 5. This is a standard hackathon failure mode. If the Threading model (Python) and Networking (FastAPI) don't play nice, you have 0 buffer. Integration of *mock* components must start on Day 2.
2.  **Dashboard Rendering Lag:** As noted by Claude, Streamlit cannot handle 30 concurrent high-frequency state updates. If the dashboard lags, the jury assumes the system is broken, even if the backend is working.
3.  **Time Sync prevents Scoring:** The "Confusion Matrix" feature (requested by the judge) relies entirely on comparing System Logs vs. Ground Truth. A 5-second clock drift means you report a "False Positive" instead of a "True Positive." The proposal acknowledges this but lacks a concrete implementation for an air-gapped LAN.
4.  **Thermal Throttling:** Celeron laptops have poor cooling. Sustained AI load (even optimized) for 10 minutes plus the heat of the battery charging can cause thermal throttling, dropping CPU speeds to 800MHz. The system needs to detect this and degrade gracefully.
5.  **Deployment Reality:** You cannot count on BIOS access to boot a Live USB (my previous suggestion), nor can you count on PyInstaller working across different Windows patch levels (Claude's concern). Deployment remains the single highest operational risk.

SUGGESTIONS:
1.  **Mandate "Detect-Track-Verify":** Do not treat this as an optimization. It is the baseline architecture. BlazeFace (5 FPS) -> OpenCV Tracker (CSRT/MOSSE) -> MobileFaceNet (Every 2s or on track loss).
2.  **Dashboard Stack Swap:** Drop Streamlit. Use **FastAPI + Vanilla JS**. Serve a static HTML page. Use `EventSource` (SSE) to listen for alerts. Update the DOM directly using IDs (e.g., `<div id="terminal-15">`). This is lighter, faster, and handles 30 updates/sec easily.
3.  **Application-Level Time Sync:** Implement the GPT-5.2 recommendation: On heartbeat, Terminal sends `local_time`, Edge replies with `server_time`. Terminal calculates `offset = server_time - local_time - (RTT/2)`. All events sent to Edge must use `local_time + offset`.
4.  **Deployment Hierarchy:** Prepare three deployment methods in order of preference:
    *   **Plan A:** Portable Python Environment (WinPython) on USB. No installation, runs from stick/folder.
    *   **Plan B:** PyInstaller One-File Binary (High risk of DLL errors).
    *   **Plan C:** Live Linux USB (If BIOS allows; most robust but restricted).
5.  **Mock Integration First:** Create "Dummy Agents" on Day 2 that send fake JSON events to the Edge Server. Build the Dashboard against these Dummies immediately. Do not wait for the AI to be finished to test the Dashboard.

KEY_TRADEOFFS:
*   **Precision vs. Recall:** Consensus remains: **Precision wins.** The system must suppress 90% of marginal alerts. A silent system that catches 3 blatant cheats wins; a noisy system that flags the jury as cheating loses.
*   **Monolith vs. Microservices:** The consensus is **Monolith (Single Process)** for the terminal. It is the only way to manage memory on 4GB RAM. The tradeoff is that if the AI crashes, the Exam UI crashes. This is an acceptable risk if "Watchdog" restarts are implemented.
*   **Visuals vs. Tech:** Moving from Streamlit to Vanilla JS increases frontend coding effort (slightly) but guarantees the **Visuals (Dashboard)** actually work. In a hackathon, if the Jury can't see it, it didn't happen.

---
## Round 3

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After three rounds of debate, the convergence is strong enough that my position is now less about critiquing the original proposal and more about synthesizing a final, actionable spec from the collective feedback. The original proposal was a solid starting point; the debate has refined it into something significantly more buildable.

**Where all three reviewers now fully agree (settled decisions):**

1. **Detect → Track → Verify is the baseline architecture, not an optimization.** GPT-5.2 originated this, Gemini mandated it, and I endorsed it. This is settled. BlazeFace at 3-5 FPS for detection, lightweight OpenCV tracker (MOSSE for Celeron, CSRT for i5) between detections, MobileFaceNet verification every 2 seconds or on track loss. This single architectural decision makes everything else feasible on Celeron hardware.

2. **Dashboard: FastAPI + SSE + vanilla JS.** All three reviewers now converge here. GPT-5.2 moved from "team's fastest option" to explicitly recommending SSE + minimal JS/HTMX. Gemini explicitly says "Drop Streamlit, use FastAPI + Vanilla JS." I've argued this from the start. This is the single most important UX decision — the jury watches the dashboard, not the terminal agents.

3. **YOLO is cut entirely.** I moved to this position in Round 2 based on Gemini's arguments. GPT-5.2 flags licensing and false-positive risks. Gemini calls it a "critical liability." The CPU budget, dependency complexity, and false-positive risk all point the same direction. Cut it.

4. **Precision over recall, enforced at the architecture level.** Not just a tuning philosophy — the alert taxonomy must structurally prevent low-confidence standalone alerts. Gaze is fusion-only. Audio is fusion-only with very conservative thresholds. Only face absence, multi-face, and identity mismatch generate standalone alerts.

5. **Single Python process for the terminal agent.** No Electron, no separate browser. PyQt6 for exam UI + proctoring pipeline in the same process. Three threads (capture, inference, network) with queues.

6. **Mock integration from Day 2.** Gemini's suggestion to build dummy agents that send fake JSON events to the edge server is the right way to decouple dashboard development from AI development. This eliminates the Day 4/5 integration crunch risk.

**Where I've shifted based on this round:**

1. **Deployment strategy: I now agree with Gemini's tiered approach over my bootable USB preference.** Gemini is right that BIOS access cannot be assumed. GPT-5.2 is right that OS-level NTP cannot be assumed on locked-down Windows machines. The correct approach is Gemini's hierarchy: Plan A (WinPython portable on USB), Plan B (PyInstaller binary), Plan C (bootable Linux USB if BIOS permits). I was over-indexing on the elegance of the bootable USB without adequately weighing the BIOS access risk. WinPython portable is the safer primary plan because it requires zero system-level access — just a folder on a USB stick.

2. **Time sync: application-level is the correct default.** I previously argued for OS-level chrony. GPT-5.2 and Gemini both correctly note that OS-level NTP requires admin access and network configuration that may not be available. GPT-5.2's RTT-based offset estimation with periodic resync is the right approach. Concrete implementation: on each heartbeat (every 5s), terminal sends `{local_ts: monotonic_ns()}`, edge responds with `{edge_ts: monotonic_ns(), terminal_local_ts: <echo>}`. Terminal computes `offset = edge_ts - terminal_local_ts - RTT/2`. Use median of last 5 offsets to smooth jitter. All event timestamps = `local_monotonic + offset`. This works on any OS without admin access.

3. **JSONL as audit artifact, SQLite WAL as primary store.** GPT-5.2's refined position is correct: direct SQLite WAL with a single-writer async queue is the primary persistence layer on the edge. JSONL files on terminals serve only as local store-and-forward buffers and audit trails. No ingestion pipeline from JSONL to SQLite — events go directly into SQLite via HTTP POST from terminals. This is simpler than what I previously suggested.

4. **Gemini's thermal throttling concern is real and unaddressed.** None of us raised this before Round 3. Celeron laptops under sustained AI load for 10 minutes will thermal-throttle. The agent needs to monitor its own FPS and degrade gracefully: if measured FPS drops below 5, disable gaze and audio, keep only face presence + multi-face + periodic verify. This is 10 lines of code and prevents the demo from visibly freezing.

**Where genuine disagreement remains:**

1. **Enrollment re-enrollment flow.** GPT-5.2 suggests a dashboard "Re-enroll" button with a 60-second FACE_MISMATCH suppression window. I think this is backwards — the enrollment flow itself should prevent bad embeddings from being accepted. Quality gates during enrollment (face detection confidence > 0.9, face size > 100px, blur score below threshold, brightness within range) are simpler and more reliable than a reactive re-enrollment mechanism. If enrollment fails the quality gate, the UI says "Please adjust your position" and retries automatically. The dashboard override is a fallback for edge cases, not the primary mechanism. This is a minor disagreement about where the logic lives, not a fundamental architectural difference.

2. **FaceMesh inclusion.** Gemini implies FaceMesh should be disabled by default on Celeron ("only enable if FPS headroom exists"). GPT-5.2 suggests gating by hardware class. I think FaceMesh should be included but gated by real-time FPS measurement, not by hardware class detection. The agent doesn't need to know it's running on a Celeron — it just needs to observe that its FPS has dropped below the threshold where gaze data is useful (~3 FPS for gaze to produce meaningful sustained-deviation signals). Hardware detection is fragile; FPS measurement is direct and reliable.

CONCERNS:

1. **Thermal throttling on Celeron is the most dangerous unaddressed risk.** Gemini raised this in Round 3 and neither GPT-5.2 nor I had previously considered it. A Celeron N4020 under sustained load at 35°C ambient (typical Indian exam hall without AC) will throttle from 2.8GHz burst to 1.1GHz base within 3-5 minutes. This halves the effective FPS. The agent must implement FPS-based degradation: measure rolling 10-second average FPS, and if it drops below thresholds, progressively disable gaze → audio → reduce verification frequency. This needs to be in the Day 1 architecture, not bolted on later.

2. **WinPython portable path has its own risks.** While I now agree it's the safest primary deployment, WinPython + MediaPipe + ONNX Runtime has known issues with DLL loading on some Windows configurations. The Day 0 packaging spike must test specifically: (a) WinPython folder copied to USB, (b) plugged into a different machine than the build machine, (c) run the agent with webcam capture + all models loaded. If this fails, the team needs to know on Day 0, not Day 9.

3. **Ground truth input interface is underspecified by everyone.** The confusion matrix is a jury-requested feature, but how does the jury input ground truth? A web form on the dashboard? A paper checklist transcribed after the exam? Real-time button presses during the exam? The simplest approach: during the exam, a "Ground Truth Logger" panel on the dashboard where an admin clicks "Terminal T-xx: [event type] at [now]" for each planted malpractice event. This is ~2 hours of development and directly enables the confusion matrix. None of us have specified this concretely enough.

4. **The "6 core features" scope is correct but the fusion rules remain vague.** All three reviewers agree on multi-modal fusion, but the specific rules are hand-wavy. For the hackathon, implement exactly two fusion rules: (a) GAZE_DEVIATION within 5s of FACE_MISMATCH → boost FACE_MISMATCH to CONFIRMED_IMPERSONATION, (b) SUSPICIOUS_AUDIO within 5s of GAZE_DEVIATION → create POSSIBLE_CONSULTATION alert. That's it. Two rules, hardcoded, tested. No rules engine, no weighted scoring system.

5. **GPT-5.2's concern about fallback video mode triggering FACE_MISMATCH storms is valid and easy to solve.** If using `--source video.mp4` for demo fallback, the agent should skip face verification entirely (no enrollment data to compare against) and only run face presence + multi-face + gaze. This needs to be documented as a known limitation of fallback mode.

SUGGESTIONS:

1. **Finalize the deployment strategy as a decision tree, not a preference list:**
   ```
   Day 0 Morning: Test WinPython portable + MediaPipe + ONNX on a non-dev Windows machine
     ├── Works → WinPython is primary deployment. Copy to 30 USBs.
     └── Fails → Test PyInstaller binary on same machine
           ├── Works → PyInstaller is primary. Build for target OS.
           └── Fails → Test bootable Ubuntu USB
                 ├── BIOS accessible → Linux USB is primary.
                 └── BIOS locked → Install Python manually on 1 terminal.
                       Test. If works, script the install for 30 machines.
   ```
   This decision tree must be resolved by Day 0 EOD. Everything else depends on it.

2. **Implement FPS-based graceful degradation as a core component, not a nice-to-have:**
   ```python
   class AdaptiveScheduler:
       def get_enabled_detectors(self, rolling_fps_10s):
           if rolling_fps_10s >= 12:
               return [FACE_DETECT, FACE_VERIFY, GAZE, AUDIO]  # Full
           elif rolling_fps_10s >= 7:
               return [FACE_DETECT, FACE_VERIFY, AUDIO]  # Drop gaze
           elif rolling_fps_10s >= 4:
               return [FACE_DETECT, FACE_VERIFY]  # Drop audio
           else:
               return [FACE_DETECT]  # Survival mode: presence only
   ```
   Terminal reports its current mode to the dashboard. Dashboard shows degraded terminals in a distinct color (orange, not red).

3. **Specify exactly two fusion rules and implement them as simple temporal correlations:**
   - Rule 1: `GAZE_DEVIATION(terminal=X, t) AND FACE_MISMATCH(terminal=X, t±5s) → CONFIRMED_IMPERSONATION(critical)`
   - Rule 2: `SUSPICIOUS_AUDIO(terminal=X, t) AND GAZE_DEVIATION(terminal=X, t±5s) → POSSIBLE_CONSULTATION(high)`
   - Implementation: edge server maintains a 10-second sliding window of events per terminal. On each new event, check if any rule matches. This is 30 lines of Python.

4. **Build the Ground Truth Logger as a dashboard panel on Day 7 (same day as confusion matrix):**
   - Simple form: select terminal from dropdown, select event type from dropdown, click "Log." Timestamp is auto-filled.
   - Ground truth events stored in a separate SQLite table.
   - Confusion matrix query joins system events with ground truth events using the 1:1 greedy matching algorithm.

5. **Add enrollment quality gates that block progression:**
   ```python
   def check_enrollment_quality(frame, face_bbox):
       face_conf = face_bbox.confidence  # Must be > 0.9
       face_width = face_bbox.width      # Must be > 80px
       blur_score = cv2.Laplacian(face_crop, cv2.CV_64F).var()  # Must be > 100
       brightness = face_crop.mean()     # Must be 40-220
       if all conditions met:
           return ACCEPT
       else:
           return REJECT, "Please move closer" / "Please face the camera" / etc.
   ```
   Re-enrollment via dashboard is the fallback, not the primary mechanism.

6. **Collapse the architecture into a single-page reference document** (not the 20-page proposal). One page with: system diagram, detector list with thresholds, event format, deployment steps, and the 10-day schedule. This becomes the team's working document. The detailed proposal is archived as rationale documentation.

KEY_TRADEOFFS:

1. **WinPython portable vs. bootable Linux USB:** WinPython works on locked-down machines without admin access but inherits all Windows-specific DLL and driver issues. Bootable Linux gives a clean, known environment but requires BIOS access. The decision tree approach (Suggestion 1) makes this a runtime decision rather than a design-time commitment, which is the right call under uncertainty.

2. **Application-level time sync vs. OS-level NTP:** Application-level sync works everywhere but introduces ~10-50ms jitter that must be absorbed by the ±15s matching window. OS-level NTP gives sub-1ms accuracy but requires admin access. Since the matching window is 15 seconds and event durations are measured in seconds, application-level accuracy is more than sufficient. The pragmatic choice wins.

3. **FPS-based degradation vs. hardware detection:** FPS measurement adds ~0 overhead and adapts to actual conditions (thermal throttling, background processes, varying ambient load). Hardware detection requires maintaining a compatibility matrix. FPS-based is strictly superior for this use case.

4. **Quality-gated enrollment vs. reactive re-enrollment:** Quality gates add 1-2 seconds to enrollment per student but prevent downstream FACE_MISMATCH false positives for the entire exam duration. Reactive re-enrollment requires proctor intervention during the exam, disrupting the demo flow. Quality gates are the better investment.

5. **Two hardcoded fusion rules vs. configurable rules engine:** Two hardcoded rules are testable, debuggable, and predictable. A rules engine is flexible but introduces a configuration surface that can break during the demo. For a hackathon with exactly 6-10 planted scenarios, hardcoded rules that you've rehearsed are strictly better.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
Other participants’ feedback doesn’t change the top-level architecture verdict (edge-first + local inference remains correct), but it *does* change what I consider “default decisions” vs “optional ideas.”

**What changed my position (materially):**
- **YOLO should be cut from the hackathon plan, not just Tier-2.** Claude + Gemini are right on cost/benefit: COCO-domain mismatch + CPU cost + false positives + additional dependency surface. Add GPT-5.2’s/others’ licensing ambiguity around Ultralytics tooling, and it becomes net-negative for demo reliability. I now treat YOLO as “post-hackathon only,” unless there’s a pre-existing, proven ONNX-only pipeline and exam-specific data.
- **Detect → Track → Verify is the baseline architecture, not an optimization.** I already suggested it, but Claude/Gemini correctly push it to Day 1 design. If you build “run models on a schedule” first, you’ll end up rewriting it under performance pressure.
- **Integration must start earlier than Day 5.** Gemini’s “mock agents Day 2” and Claude’s “vertical slice by Day 4” is the right operational stance. Hackathon failures are almost always integration + deployment, not model accuracy.

**Where I agree with others (strong convergence):**
- **No Streamlit for the live wallboard.** SSE/WebSocket + incremental DOM updates is the most predictable way to keep the grid smooth.
- **Precision over recall, and gaze is fusion-only.** Standalone gaze alerts will drown the dashboard and ruin credibility.
- **Packaging/deployment is the existential risk; Day-0 go/no-go is mandatory.**
- **Time sync is scoring-critical and needs a concrete mechanism.**
- **Confusion-matrix matching must be 1:1 (dedupe),** or persistent alerts inflate TP counts.

**Where I disagree / qualify:**
- **“Bootable Linux USB as primary deployment” (Claude/Gemini):** technically superior, but operationally uncertain in govt labs (BIOS locked, USB boot disabled, policy restrictions). I’d keep it as **Plan C** unless you’ve confirmed BIOS access on the actual machines. Portable environment / minimal install remains more likely to work.
- **“JSONL-first with SQLite derived view” (Claude):** workable, but I still think the simplest hackathon path is **SQLite WAL with a single-writer queue** as the primary store *plus* optional JSONL append as an audit/replay artifact. What I fully agree with: **do not build a dual-path ingestion subsystem** that can disagree with itself.

CONCERNS:
1. **YOLO as a destabilizer:** even “stretch” features consume integration time, CPU headroom, and add failure modes; worst-case it increases false positives and hurts the confusion matrix.
2. **Late integration risk:** if the dashboard, edge, and agents are first wired together on Day 5, you’re betting the demo on the least predictable phase.
3. **Deployment uncertainty remains #1:** PyInstaller can fail; portable Python can hit missing VC runtimes/audio drivers; Linux USB can be blocked by BIOS policy. You need rehearsed fallback paths.
4. **Time sync underspecification:** heartbeat offset estimation can be noisy; OS-level sync may be unavailable on Windows without admin. Without sub-second alignment, scoring and evidence timelines become untrustworthy.
5. **Thermal throttling on Celeron:** sustained load can silently drop CPU frequency and collapse FPS mid-demo unless you detect it and degrade.
6. **Enrollment failure recovery still under-designed:** a bad enrollment can create permanent identity noise; dashboard override helps but isn’t enough—enrollment must self-reject low-quality captures.
7. **Exam UI vs proctoring resource contention:** single-process avoids webcam contention, but CPU spikes can still freeze the UI unless inference is strictly bounded and deprioritized relative to UI responsiveness.

SUGGESTIONS:
1. **Make Detect→Track→Verify the canonical pipeline (Day 1):**
   - BlazeFace at low rate (e.g., 2–5 FPS) to reacquire.
   - Lightweight tracker (MOSSE/KCF) for ROI continuity.
   - MobileFaceNet verification on track-loss or every N seconds (not per-frame).
2. **Cut YOLO from hackathon scope entirely.**
   - If you must show “object cheating,” fake it with a scripted demo overlay is worse than doing nothing—don’t.
3. **Dashboard: FastAPI + SSE (EventSource) + static HTML/JS.**
   - Terminal tiles update by ID; alert feed appends rows; no rerender loops.
4. **Start integration with dummy agents by Day 2.**
   - Generate synthetic events at realistic rates; validate edge ingestion, dashboard rendering, storage, and time sync before AI is complete.
5. **Time sync: implement app-level offset + (optional) OS-level if available.**
   - App-level: RTT-based offset estimation, periodic resync, monotonic clock usage, stamp events in edge-time.
   - If Linux image is allowed: chrony with edge as authority is great—treat as an enhancement, not the only plan.
6. **Thermal/FPS watchdog + explicit degradation modes:**
   - If FPS < threshold for X seconds: disable gaze/audio first, then reduce verify cadence, keep only presence/multi-face/identity hysteresis.
7. **Confusion-matrix scorer: enforce 1:1 matching.**
   - Greedy closest-match within window (or Hungarian if you want), but prevent one ground-truth event matching multiple detections.

KEY_TRADEOFFS:
- **Feature breadth vs demo stability:** cutting YOLO and making gaze fusion-only sacrifices coverage (phone/notes detection) to protect credibility and smooth real-time operation. For a 10-minute judged demo, this is the correct trade.
- **Deployment robustness vs policy feasibility:** bootable Linux yields the most controlled environment, but may be blocked; portable env is more feasible but more variable. The right move is a rehearsed hierarchy (A/B/C), not a single bet.
- **SQLite-first vs JSONL-first:** SQLite-first simplifies querying; JSONL-first simplifies append-only writes and replay. The important tradeoff is not which one you pick—it’s avoiding two competing “sources of truth” under hackathon time pressure.
- **App-level time sync vs OS-level time sync:** OS-level is cleaner when you control the OS; app-level is portable. Given uncertainty on admin access, app-level sync is the safer baseline with OS sync as optional acceleration.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The consensus among all reviewers is overwhelming: the original proposal is **operationally fragile** and **computationally optimistic**. There is now a hardened agreement on the following architectural pivots, which move this from "risky" to "viable":

1.  **Architectural Baseline:** The "Detect-Track-Verify" pattern is no longer a suggestion; it is the **mandatory requirement** to function on Celeron hardware.
2.  **Feature Cuts:** YOLOv8 is universally rejected. It consumes too much CPU for too little gain (low accuracy on exam materials). It must be cut to save the Core 6 features.
3.  **Dashboard Tech:** Streamlit is effectively banned. The latency risk for 30 concurrent streams is unacceptable. **FastAPI + SSE + Vanilla JS** is the converged solution.

However, I disagree with Claude Opus regarding the **Deployment Strategy**. While a Bootable Linux USB is *technically* superior, it is **administratively naive**. Government exam centers frequently have BIOS passwords, Secure Boot enabled, and disabled USB booting to prevent exactly this kind of "tampering." Plan A must be a **Portable Windows Environment (WinPython)** that runs in user-space without admin rights.

CONCERNS:
1.  **Deployment "Access" Blindspot:** Relying on a Bootable USB (Claude's Plan A) is a potential Day 1 showstopper if the lab computers have BIOS locks. Relying on PyInstaller (Original Plan) risks DLL hell. The team lacks a reliable "Plan A" for locked-down Windows machines.
2.  **Enrollment "Poisoning":** As noted by GPT-5.2 and Gemini, a poor enrollment photo (bad lighting/blur) creates a permanent "Face Mismatch" state for that student, ruining the demo. The current plan lacks a "Quality Gate" or "Re-enroll" workflow.
3.  **Webcam Contention:** If the Exam UI and the Proctoring Agent are separate processes (or if the OS locks the camera), the demo fails immediately. The architecture must ensure the camera handle is shared or managed by a single process.
4.  **Scoring Integrity:** Without the **1:1 deduplication logic** mentioned by Claude (Hungarian algorithm or greedy matching), the "Confusion Matrix" will be statistically invalid, matching one ground-truth event to 50 persistent alerts.

SUGGESTIONS:
1.  **Deployment Hierarchy (Revised):**
    *   **Plan A (Primary):** **Portable WinPython** on USB. Pre-install all dependencies (MediaPipe, OpenCV, FastAPI) in a folder. Run via batch script. No installation, no Admin rights, no BIOS access needed.
    *   **Plan B (Fallback):** Bootable Linux USB (Only if BIOS is accessible).
2.  **Dashboard Stack Finalization:** Adopt **FastAPI + Server-Sent Events (SSE) + Vanilla JS/HTMX**. This removes the React build step (complexity) and the Streamlit re-render lag (performance).
3.  **Enrollment Quality Gate:** Implement a strict check during Phase 1. If `face_confidence < 0.8` or `brightness < threshold` or `blur_score > threshold`, **reject the enrollment** and force the student to adjust.
4.  **Time Sync Implementation:** Adopt GPT-5.2's **Application-Level Offset** approach. It is OS-agnostic. The Edge server sends its timestamp in the heartbeat response; the Terminal calculates the delta. Do not rely on configuring OS-level NTP in a restricted environment.
5.  **Integration Acceleration:** Move the "Vertical Slice" (Camera -> AI -> Event -> Dashboard) to **Day 3**. If the network/threading stack isn't working by Day 3, the project is doomed. Day 5 is too late.

KEY_TRADEOFFS:
*   **Access vs. Control (Deployment):** Using a Portable Windows App (Plan A) sacrifices the clean environment of a Bootable USB for **compatibility** with locked-down government hardware. This is the correct tradeoff for a hackathon.
*   **UX Smoothness vs. Dev Speed (Dashboard):** ditching Streamlit costs ~5 hours of extra JS coding but guarantees a **lag-free demo**. In a visual evaluation, lag is interpreted as failure.
*   **Recall vs. Stability (YOLO):** Cutting object detection means missing phones, but ensures the Celeron CPU doesn't melt, preserving the stability of the Identity and Gaze features. Stability is preferred over breadth.

---
## Round 4

### Claude Opus



POSITION: AGREE

REASONING:

After reviewing GPT-5.2 and Gemini's latest feedback, I'm moving from PARTIALLY_AGREE to AGREE. The convergence is now strong enough that the remaining disagreements are implementation details, not architectural decisions. Let me explain what shifted.

**What changed my position:**

The three reviewers have independently converged on the same architecture through different analytical paths. That's the strongest possible signal that the design is correct. The remaining deltas are about ordering and emphasis, not substance.

**Complete consensus (all three reviewers agree, no qualifications):**

1. **Detect → Track → Verify as Day 1 architecture.** No one is arguing against this anymore. GPT-5.2 originated it, I endorsed it, Gemini mandated it. This is the single most important technical decision in the entire project.

2. **YOLO cut entirely from hackathon scope.** GPT-5.2 explicitly says "cut YOLO from hackathon scope entirely" and "if you must show object cheating, doing nothing is better than a bad demo." Gemini says "universally rejected." I agreed in Round 2. This is settled with zero dissent.

3. **FastAPI + SSE + vanilla JS/HTMX for dashboard.** GPT-5.2 says "FastAPI + SSE (EventSource) + static HTML/JS." Gemini says "FastAPI + SSE + Vanilla JS/HTMX." I said the same in my previous review. Three independent paths to the same answer.

4. **Precision over recall, gaze fusion-only.** GPT-5.2: "gaze is fusion-only." Gemini agrees by omission (doesn't argue for standalone gaze). I specified this explicitly. No dissent.

5. **Application-level time sync as default.** GPT-5.2 says "app-level offset + optional OS-level." Gemini says "adopt GPT-5.2's application-level offset approach." I agreed in my previous review. Settled.

6. **Deployment as a decision tree, not a single bet.** All three agree on this pattern. The only question was ordering.

7. **Day 0 packaging spike is existential.** GPT-5.2: "packaging/deployment is the existential risk; Day-0 go/no-go is mandatory." Gemini: same. Me: same. No one disagrees.

8. **Early integration (mock agents by Day 2, vertical slice by Day 3-5).** GPT-5.2 pushes Day 2 mocks. Gemini pushes vertical slice to Day 3. I said Day 2 mocks, Day 5 working demo. The spread is Day 2-5, which is fine — the point is that no one is saying "integrate on Day 8."

9. **1:1 deduplication for confusion matrix scoring.** GPT-5.2 says "enforce 1:1 matching." Gemini says "without 1:1 deduplication logic the confusion matrix will be statistically invalid." I specified greedy matching in my previous review. Unanimous.

10. **Enrollment quality gates.** GPT-5.2 flags enrollment failure recovery as under-designed. Gemini specifies confidence/brightness/blur thresholds. I provided concrete code. All three agree quality gates are the primary mechanism; dashboard re-enrollment is the fallback.

**On the deployment ordering — I now fully agree with Gemini and GPT-5.2:**

I previously presented a decision tree with WinPython as Plan A, PyInstaller as Plan B, bootable Linux as Plan C. Both other reviewers reinforced this same ordering and explicitly flagged my earlier bootable-USB-first preference as risky. They're right, and I had already shifted to this position in my previous review. The convergence is complete: WinPython portable is Plan A, period.

Gemini's framing — "administratively naive" regarding BIOS access — is correct and I don't push back on it. GPT-5.2's point about "policy restrictions" in government labs is the same concern stated differently. I concede this point fully.

**On the remaining minor disagreements:**

The only areas where there's any spread at all are:

- **Exact day for vertical slice:** I said Day 5, Gemini says Day 3, GPT-5.2 implies Day 3-4. Gemini's Day 3 is aggressive but the right aspiration. The compromise is: mock agents by Day 2 evening (synthetic events → edge → dashboard), real vertical slice (camera → AI → edge → dashboard for 1 terminal) by Day 4. This is achievable because the mock agents decouple dashboard work from AI work.

- **JSONL vs SQLite as primary:** GPT-5.2 and I now agree on SQLite WAL as primary with JSONL as audit artifact. Gemini doesn't explicitly weigh in on this but doesn't contradict it. I consider this settled.

- **Webcam contention (Gemini's Concern #3):** This is a valid concern that I hadn't explicitly addressed. The single-process PyQt6 architecture solves it — one process owns the camera handle, inference threads read from a shared frame queue. But Gemini is right to call it out as a risk: if anyone introduces a second process that tries to open the webcam, the demo dies. The architecture document should state explicitly: "Only one process may open the webcam. The terminal agent owns the camera. The exam UI is a PyQt6 widget within the same process."

CONCERNS:

1. **GPT-5.2's Concern #7 (UI responsiveness vs inference load) is the one under-discussed risk.** In a single-process PyQt6 app, if the inference thread consumes too much CPU, the exam UI (MCQ questions, answer selection) becomes sluggish or unresponsive. The student can't answer questions, which is visible to the jury and looks like a broken product. Mitigation: inference thread should `time.sleep()` to yield CPU when it detects frame processing is taking >80% of the frame budget. PyQt6's event loop runs on the main thread; inference runs on Thread 2; these don't share the GIL during ONNX inference (which releases the GIL). But OpenCV preprocessing and Python-level postprocessing do hold the GIL. The practical fix is to keep Python-side processing minimal between GIL-releasing calls (cv2 operations and ONNX inference both release the GIL). This should be tested on Day 1 with a dummy PyQt6 window + inference loop running simultaneously.

2. **Gemini's Concern #4 (scoring integrity) deserves concrete specification.** The 1:1 greedy matching algorithm: sort all system events and ground truth events by timestamp. For each ground truth event, find the nearest unmatched system event of the same type within ±15 seconds. If found, it's a TP (mark both as matched). If not found, it's a FN. After all ground truth events are processed, remaining unmatched system events are FPs. This is O(n*m) but with n<50 events, performance is irrelevant. The important thing is that the matching is deterministic and documented so the jury can verify it.

3. **No one has addressed what happens if enrollment takes too long.** With 30 students enrolling simultaneously, each taking 5 guided photos + gaze calibration, the enrollment phase could take 3-5 minutes. If a student's webcam fails during enrollment, the proctor needs a way to skip that student (mark as "manual verification only") and proceed. The enrollment flow needs a 2-minute timeout with a "Skip — manual mode" option that flags the terminal as degraded on the dashboard.

4. **The exam content itself is unspecified.** The plan mentions "10 MCQ questions" but doesn't say where they come from. For the hackathon demo, the team needs to create 10 plausible exam questions (general knowledge or basic math) that a jury member could actually answer. This is 30 minutes of work but easy to forget until demo day.

SUGGESTIONS:

1. **Publish the final architecture as a one-page decision record.** All three reviewers have converged. The team needs a single reference document, not three rounds of debate transcript. Format:
   - System diagram (terminal → edge → dashboard)
   - 6 core features with thresholds
   - Event JSON schema
   - Deployment decision tree
   - 10-day schedule with exit criteria per day
   - Demo day checklist

2. **Add a GIL-awareness test to Day 1.** Create a minimal test: PyQt6 window with a timer updating a counter label every 100ms + a background thread running ONNX inference in a loop. If the counter freezes or stutters, the architecture has a GIL problem that must be solved before proceeding. If it runs smoothly (it should, since ONNX releases the GIL), document why and move on.

3. **Enrollment timeout with graceful skip.** If enrollment doesn't complete within 120 seconds, offer "Skip enrollment — this terminal will run in degraded mode (no face verification, presence detection only)." Dashboard shows the terminal in a distinct state. This prevents one bad webcam from delaying the entire exam start.

4. **Prepare 10 MCQ questions in advance.** General knowledge, appropriate for the stated exam context. Store as a JSON file loaded by the PyQt6 exam UI. This is trivial but must not be forgotten.

5. **The vertical slice target should be Day 4, not Day 3 or Day 5.** Day 1-2: AI pipeline working on a single terminal (camera → detections → local events). Day 2: mock agents sending synthetic events to edge. Day 3: edge server + dashboard showing mock events. Day 4: replace mock agent with real terminal agent → real vertical slice. Day 5: polish and add remaining features. This threading respects the dependency chain while maintaining early integration.

KEY_TRADEOFFS:

1. **Single-process PyQt6 vs. separate exam UI and proctoring processes:** Single process eliminates webcam contention and simplifies deployment, but creates a GIL-sharing risk where heavy Python-side processing could stall the exam UI. The tradeoff favors single-process because ONNX and OpenCV both release the GIL during their heavy operations, and the alternative (two processes fighting over a webcam) is a harder problem to solve. This is the right tradeoff.

2. **Day 4 vertical slice vs. Day 3 (Gemini) or Day 5 (original):** Day 3 requires the AI pipeline to be working on Day 2, which is tight for face enrollment + verification + gaze. Day 5 leaves only 5 days for integration debugging. Day 4 gives 3 days for AI work and 6 days for integration + polish. This splits the risk evenly, which is the right call for a hackathon.

3. **Conservative audio thresholds vs. cutting audio entirely (Gemini's position):** Gemini suggests dropping audio to focus on 3 core features. GPT-5.2 and I say keep it with conservative thresholds. The tradeoff: keeping audio adds ~1 day of work and one more potential false-positive source, but also adds a detection modality that costs almost zero CPU (Silero VAD is <1ms). Given that it's fusion-only (never generates standalone alerts), the downside risk is minimal and the upside is detecting the "whispering to neighbor" scenario that the jury will likely plant. I maintain audio should stay, with the understanding that if Day 4 arrives and audio isn't working, it gets cut without hesitation.

4. **The overall tradeoff this architecture makes — and it's the right one — is sacrificing feature breadth for demo reliability.** Six features that work flawlessly on 30 terminals for 10 minutes beats fifteen features where three crash mid-demo. Every reviewer arrived at this independently. The architecture is now optimized for the actual success criterion: a jury watching a live demo and evaluating what they see.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
Other participants’ feedback *does* change a few practical “defaults,” but it doesn’t overturn the core architectural direction (edge-first, local inference, minimal network dependency). Net effect: the proposal converges into a much more buildable hackathon spec.

**What changed my position (arguments that materially moved me):**
- **Deployment plan:** Claude/Gemini’s emphasis on *locked-down government PCs* is decisive. I previously treated “portable environment vs PyInstaller vs bootable Linux” as preference; I now agree it must be a **decision tree resolved on Day 0** with **WinPython/portable user-space** as the most likely Plan A (no admin, no BIOS).
- **Thermal throttling + FPS-based degradation:** This was underweighted in the original. Sustained inference on Celerons will throttle; you need **runtime FPS gates** and explicit detector shedding. This is not an optimization—this is a reliability requirement.
- **Dashboard tech convergence:** I already leaned to minimal front-end; the group’s convergence on **FastAPI + SSE + vanilla JS/HTMX** is the correct “jury-safe” choice vs React/Streamlit for this context.

**Where I agree with others (strong convergence / common ground):**
- **Detect → Track → Verify is mandatory** to survive Celeron performance (face detect low FPS + tracker + periodic verify).
- **Cut YOLO entirely** for hackathon: domain mismatch + CPU cost + FP risk + dependency/licensing ambiguity.
- **Precision-over-recall enforced structurally:** gaze (and often audio) should be **fusion-only** so the dashboard doesn’t flood.
- **Application-level time sync** (RTT/offset estimation) is the only portable assumption.
- **Early integration via mock agents** by Day 2/3 and a vertical slice before feature expansion.
- **Confusion-matrix scoring must be 1:1 matched** (dedupe), not “within a window” many-to-one.

**Where genuine disagreement remains (and why):**
- **Primary persistence model (Edge):** Claude proposes “SQLite primary + optional JSONL audit,” others sometimes imply JSONL-first. I still think the simplest operationally is:  
  - **Edge: SQLite WAL as primary** with a single-writer queue (predictable querying),  
  - **Terminal: JSONL spool** for store-and-forward when edge is down.  
  This avoids building an ingestion pipeline that can drift.
- **Audio scope:** There’s still tension. I’m aligned with “audio fusion-only + conservative thresholds,” but I would only ship audio if it’s stable on the demo hardware and doesn’t cause mic/driver issues. Otherwise, drop it rather than risk flakiness.

CONCERNS:
1. **Deployment remains the top existential risk** even with a plan: WinPython/portable can still hit DLL/runtime issues (MediaPipe, ONNX, audio). This must be validated on *non-dev target machines* Day 0.
2. **Thermal throttling + background processes** can crater FPS mid-exam; without degradation modes the UI will freeze or alerts will lag (which looks like failure to judges).
3. **Enrollment poisoning is real:** without strict quality gates (blur/brightness/face size/confidence), you’ll create “permanent mismatch” noise that you can’t fix without disrupting the demo.
4. **Time sync correctness details:** offset estimation must use monotonic clocks + smoothing; naïve wall-clock sync can drift or jump (Windows time adjustments).
5. **Confusion-matrix integrity:** without 1:1 event matching + persistence collapsing (one long event shouldn’t become 30 “detections”), your reported metrics will be obviously inflated.
6. **Dashboard update path overload:** even SSE can be misused (broadcasting full state every tick). You need incremental, per-terminal deltas to keep browser load stable on a projector PC.

SUGGESTIONS:
1. **Adopt the Day-0 deployment decision tree (and test it properly):**
   - Test on a fresh, locked-down Windows machine: webcam open, ONNX load, MediaPipe load, mic capture.
   - Plan A: portable WinPython folder + pinned wheels; Plan B: PyInstaller; Plan C: bootable Linux *only if BIOS confirmed*.
2. **Make FPS-based degradation a first-class component (wired to the dashboard):**
   - Rolling FPS + “mode” state (FULL / NO_GAZE / NO_AUDIO / SURVIVAL).
   - Hard shed order: gaze → audio → reduce verify cadence → presence/multi-face only.
3. **Enrollment quality gates + auto-retry UX:**
   - Block enrollment completion unless min thresholds are met (face conf, face size, blur var, brightness range).
   - Only if repeated failure: allow admin “re-enroll” override (fallback, not primary).
4. **Edge event model: normalize to “episodes,” not spam:**
   - Terminal should emit `ALERT_START`, `ALERT_UPDATE` (optional), `ALERT_END` with a stable `episode_id`.  
   This makes dashboard calmer and confusion-matrix matching sane.
5. **Confusion-matrix scorer: enforce 1:1 matching + collapse windows:**
   - Greedy nearest-neighbor within ±15s is fine; ensure one GT event matches at most one detection episode.
6. **Dashboard: SSE with delta messages only:**
   - Per-terminal last-known state + append-only alert feed; don’t retransmit full grid every heartbeat.

KEY_TRADEOFFS:
- **Access vs control (deployment):** WinPython/portable sacrifices environmental purity but is most compatible with locked-down govt PCs; bootable Linux is cleaner but often infeasible. Choosing portability first is the right hackathon trade.
- **Breadth vs stability:** cutting YOLO (and possibly audio) reduces “cheating type coverage” but increases demo reliability and precision—this is the correct scoring strategy when judges watch live behavior.
- **Precision vs recall:** making gaze/audio fusion-only reduces standalone detections (recall) but prevents alert floods and preserves credibility (precision). For a 10-minute demo, precision wins.
- **Simplicity vs flexibility (rules/fusion):** two hardcoded fusion rules are less “product-like” than a rules engine, but far more testable and less failure-prone under hackathon constraints.

### Gemini 3 Pro

POSITION: AGREE

REASONING:
We have reached a robust consensus. The debate has successfully filtered out the "hackathon vanity metrics" (YOLO, Electron, Complex Rules Engines) and hardened the system around operational reality (Celeron constraints, locked-down BIOS, network latency).

The converged architecture is now:
1.  **Compute:** Single-process Python (PyQt6) using a **Detect-Track-Verify** loop (BlazeFace → Tracker → MobileFaceNet).
2.  **Network/UI:** FastAPI + Server-Sent Events (SSE) + Vanilla JS for a lag-free, non-blocking dashboard.
3.  **Deployment:** A **Decision Tree** strategy (Plan A: WinPython Portable, Plan B: PyInstaller, Plan C: Linux USB) rather than a single fragile bet.
4.  **Scope:** **Precision over Recall**. YOLO is cut. Gaze and Audio are fusion-only signals.
5.  **Reliability:** **Application-level Time Sync** and **FPS-based Thermal Throttling** are now core requirements, not optimizations.

CONCERNS:
1.  **The "Integration Cliff":** While we agree on "Mock Agents on Day 2," there is a risk that the team delays integrating the *actual* heavy AI models until Day 6 or 7. Merging heavy inference threads with networking threads usually exposes GIL (Global Interpreter Lock) issues. The "Vertical Slice" with *real* models must happen by Day 3.
2.  **Ground Truth Logging Ergonomics:** The confusion matrix features depends entirely on the jury's ability to log events accurately. If they scribble times on paper and enter them later, the timestamps will not match the system's `server_time`, rendering the confusion matrix useless.
3.  **Enrollment "Frustration Loop":** Strict quality gates are necessary, but if a student (jury member) fails enrollment 5 times due to bad room lighting, the demo stalls. We need a "Force Accept" override.

SUGGESTIONS:
1.  **Codify the Fusion Rules (Final Spec):** Adopt Claude’s proposal exactly. Do not overengineer.
    *   *Rule 1:* `GAZE_DEVIATION` + `FACE_MISMATCH` (±5s) = **CONFIRMED_IMPERSONATION**.
    *   *Rule 2:* `SUSPICIOUS_AUDIO` + `GAZE_DEVIATION` (±5s) = **POSSIBLE_CONSULTATION**.
2.  **Ground Truth "Live" Logger:** The dashboard must have an "Admin/Jury Mode" with big buttons (e.g., "Planted Phone Now", "Planted Swap Now"). Clicking a button captures `server_now()` instantly. Do not rely on manual time entry.
3.  **Enrollment Fallback:** Implement the Quality Gate (check blur/brightness), but if it fails 3 times, show a "Force Proceed" button to the proctor. This prevents a demo deadlock while logging the "Low Quality Enrollment" risk.
4.  **Day 0 "DLL Hunt":** The Deployment Decision Tree is solid, but the WinPython plan often fails on missing `vcruntime140.dll` or `mfplat.dll` on stripped-down Windows Education editions. Include the **Microsoft Visual C++ Redistributable** installer on the USB stick as a standalone file, just in case.

KEY_TRADEOFFS:
*   **Precision vs. "Wow Factor":** Cutting YOLO and demoting Gaze/Audio to fusion-only means the system will be "quiet" most of the time. This risks looking inactive, but it prevents the fatal error of spamming false positives. We trade "activity" for **credibility**.
*   **Hardcoded vs. Dynamic:** We are accepting hardcoded logic (2 fusion rules, fixed thresholds) over a configurable engine. This makes the system brittle to changing requirements but **robust against configuration errors** during the live demo.
*   **Quality vs. Speed (Enrollment):** We sacrifice a fast "click-and-go" enrollment for a slower, gated process. This upfront time cost pays for itself by preventing 10 minutes of false "Face Mismatch" alerts.

---
## Round 5

### Claude Opus



POSITION: AGREE

REASONING:

My position remains AGREE. The convergence across all three reviewers is now essentially complete. GPT-5.2 moved from PARTIALLY_AGREE but their remaining points are refinements within the agreed architecture, not challenges to it. Gemini moved to AGREE with focused operational concerns. Let me address what's new and what's settled.

**Have any arguments changed my position?**

No fundamental change, but two points from this round sharpened my thinking:

1. **GPT-5.2's episode-based event model (ALERT_START / ALERT_UPDATE / ALERT_END with stable episode_id).** This is genuinely better than what I previously described. My earlier framing was "events with persistence filtering," which is the same concept but without the clean API contract. The episode model solves three problems simultaneously: it makes the dashboard calmer (one row per episode, not per frame), it makes confusion matrix matching trivial (match one episode to one ground truth event), and it prevents the "30 detections from one long gaze deviation" inflation problem. I adopt this fully. It's maybe 2 hours of additional design work on Day 4 when building the event logging system, and it pays for itself immediately.

2. **Gemini's Ground Truth "Live Logger" with big buttons.** This is the single most important suggestion that wasn't in the original architecture and that no one (including me) had fully specified until now. The confusion matrix is only as good as the ground truth timestamps. If the jury writes times on paper and someone transcribes them later, you get ±30-60 second errors that destroy the scoring. A dashboard page with large labeled buttons ("Phone Planted", "Seat Swap", "Whispering") that capture `server_now()` on click is essential. I called out the need for a "ground truth input interface" in my previous review but didn't specify the UX. Gemini's version is correct: instant capture, no manual time entry.

**Where is the common ground now?**

Everything architectural is settled. All three reviewers agree on:

- Single-process PyQt6 terminal agent with threaded pipeline
- Detect → Track → Verify inference pattern
- FastAPI + SSE + vanilla JS/HTMX dashboard on edge server
- WinPython portable as Plan A deployment
- YOLO cut from hackathon scope
- Gaze and audio as fusion-only signals
- Precision over recall as the scoring strategy
- Application-level time sync
- FPS-based degradation modes
- Day 0 packaging spike as go/no-go gate
- SQLite WAL on edge as primary persistence
- 1:1 greedy matching for confusion matrix scoring
- Enrollment quality gates with forced fallback
- LAN-first, no cloud dependency for core function

The remaining discussion items are implementation details where all three reviewers are within a narrow band of agreement.

**Addressing GPT-5.2's specific remaining concerns:**

On **SQLite WAL as primary + JSONL spool on terminal**: I already agreed with this in my previous review. GPT-5.2 frames it as a remaining disagreement, but I think we're actually aligned. Terminal writes JSONL locally for store-and-forward resilience, edge ingests into SQLite WAL for queryable persistence. The only nuance is whether the edge also writes JSONL as an audit trail. I say yes, it's one `file.write()` call before the SQLite insert, costs nothing, and provides a recovery path if SQLite corrupts. But this is not worth debating further.

On **audio scope**: GPT-5.2 says "only ship audio if stable on demo hardware." I agree completely and said the same thing previously: "if Day 4 arrives and audio isn't working, it gets cut without hesitation." We're aligned. The decision point is Day 4, the criterion is "does it work on target hardware without driver issues," and the default if unclear is to cut it.

On **SSE delta messages**: GPT-5.2 is right that broadcasting full state every heartbeat is wasteful. The episode model naturally solves this — you send ALERT_START when an episode begins, ALERT_END when it resolves, and periodic heartbeats are just `{terminal_id, fps, risk_level, timestamp}` which is tiny. The dashboard maintains client-side state. This is standard SSE practice.

On **monotonic clocks for time sync**: Correct and important. `time.monotonic()` for interval measurement, wall clock only for display and event stamping after offset correction. Python's `time.monotonic()` is available on all platforms. This is a 5-line fix in the time sync code.

**Addressing Gemini's specific remaining concerns:**

On **the "Integration Cliff" (Concern #1)**: Gemini wants the vertical slice with real models by Day 3. GPT-5.2 implies Day 3-4. I said Day 4 previously. I'll concede to Day 3 as the *target* with Day 4 as the *hard deadline*. The reasoning: if by end of Day 3 you can't get one terminal sending real face detection events to the edge server and seeing them on the dashboard, you have a fundamental integration problem that needs Day 4-5 to debug. If the target is Day 4 and you slip, you're debugging integration on Day 5-6, which is too late. So: Day 3 target, Day 4 hard deadline, and if Day 4 passes without a vertical slice, the team stops all feature work and focuses exclusively on integration.

On **Enrollment "Frustration Loop" (Concern #3)**: Gemini says "Force Proceed" after 3 failures. I said 120-second timeout with "Skip — manual mode." These are the same solution expressed differently. I'll adopt Gemini's framing because it's more precise: 3 failed quality gate attempts → "Force Proceed" button appears → terminal flagged as degraded on dashboard. The 120-second timeout is a secondary safety net in case the student just sits there without attempting enrollment.

On **Day 0 "DLL Hunt" (Suggestion #4)**: Including the VC++ Redistributable on the USB stick is a zero-cost precaution that prevents a common failure mode. Adopted without reservation. Also include the DirectX runtime installer for the same reason — some stripped Windows Education installs are missing media foundation DLLs that OpenCV needs for webcam capture.

CONCERNS:

1. **The episode model needs explicit specification before coding begins.** GPT-5.2 proposed it, everyone implicitly agrees, but no one has written the schema. Here it is: `{episode_id: uuid, terminal_id, student_id, alert_type, severity, started_at, ended_at (null if ongoing), peak_confidence, trigger_count, corroborating_episodes: [episode_ids]}`. The terminal emits `EPISODE_START` when persistence threshold is first met, `EPISODE_END` when the condition clears for >5 seconds, and optionally `EPISODE_UPDATE` if severity escalates mid-episode. The edge stores episodes in a dedicated SQLite table. The confusion matrix scorer matches ground truth events to episodes (1:1 greedy by nearest `started_at`). Without this spec written down before Day 4, three developers will implement three different event models.

2. **Gemini's "Integration Cliff" concern is the most important remaining risk.** I've seen hackathon teams where individual components work perfectly in isolation but the system fails catastrophically when connected. The mock agent approach mitigates this, but the real test is camera → inference → event → network → dashboard on actual hardware. If this doesn't work by Day 3-4, nothing else matters.

3. **GPT-5.2's thermal throttling concern deserves a concrete implementation.** The degradation mode spec should be: measure rolling average FPS over last 30 frames. If FPS drops below 10 → shed gaze (Face Mesh). Below 7 → shed audio. Below 5 → reduce face detection to every 3rd frame, verification to every 15th frame. Below 3 → presence-only mode (just "is there a face: yes/no"). Dashboard shows current mode per terminal. This is ~50 lines of code wrapping the inference loop and should be built on Day 3 alongside the inference pipeline, not added later as an afterthought.

4. **No one has specified what "exam start" synchronization looks like.** 30 terminals need to begin the exam simultaneously (or near-simultaneously). The edge server should broadcast an "EXAM_START" message via WebSocket/SSE that all terminals listen for. The proctor clicks "Start Exam" on the dashboard, edge broadcasts, terminals transition from enrollment-complete to exam-active state. Without this, terminals start at different times and the 10-minute exam window is inconsistent across terminals. Simple to implement but must be designed explicitly.

SUGGESTIONS:

1. **Write the episode event schema as a shared document on Day 1.** Include the JSON structure, the SQLite table schema, and the state machine (IDLE → ACTIVE → ENDED). All three pipeline stages (terminal emission, edge storage, dashboard rendering, confusion matrix scoring) derive from this single schema. This is 30 minutes of work that prevents days of integration pain.

2. **Build the ground truth live logger on Day 5 alongside the dashboard.** It's a separate page on the same FastAPI app: `/admin/ground-truth`. Large buttons per malpractice type. Each click writes `{type, terminal_id (dropdown), timestamp: server_now()}` to a dedicated SQLite table. The confusion matrix scorer reads from both the episodes table and the ground truth table. This is perhaps 2 hours of work and is required for the scoring to function.

3. **Exam synchronization via edge broadcast.** Dashboard has a "Start Exam" button. Edge sends `{event: "EXAM_START", timestamp: server_now()}` to all connected terminals. Terminals transition to exam mode, start the 10-minute timer, begin showing MCQ questions, and activate the full inference pipeline. "Stop Exam" works the same way. This is 30 minutes of work during Day 5-6 dashboard development.

4. **Pack VC++ Redistributable + DirectX runtime on the deployment USB.** Zero-cost insurance. Download both installers (total ~50MB) and include them in the deployment folder alongside the WinPython environment.

5. **I endorse Gemini's fusion rule codification.** Two rules, hardcoded, with a 5-second correlation window:
   - Rule 1: `GAZE_DEVIATION` episode + `FACE_MISMATCH` episode (overlapping or within 5s) → `CONFIRMED_IMPERSONATION`
   - Rule 2: `SUSPICIOUS_AUDIO` episode + `GAZE_DEVIATION` episode (overlapping or within 5s) → `POSSIBLE_CONSULTATION`
   
   These run on the edge server every time a new episode starts or updates. Output is a "fused alert" that appears prominently on the dashboard. No rules engine, no configuration, no abstraction. Two if-statements.

KEY_TRADEOFFS:

1. **Episode model vs. raw event stream:** The episode model adds a small amount of state management complexity on the terminal (tracking when an alert condition starts and ends) but dramatically simplifies everything downstream: dashboard rendering, confusion matrix scoring, and bandwidth. Every reviewer has converged on this being the right tradeoff. The cost is ~50 lines of state machine code per alert type on the terminal.

2. **Day 3 vertical slice target vs. Day 4 hard deadline:** Targeting Day 3 means Day 1-2 must be extremely focused (webcam + face detection + enrollment + basic edge server). This is aggressive but correct because it forces integration issues to surface early. The tradeoff is less time for polishing individual AI components before integration, but integration bugs are harder to fix than detection bugs, so they should be discovered first.

3. **Two hardcoded fusion rules vs. more comprehensive fusion:** We're leaving detection combinations on the table (e.g., CANDIDATE_ABSENT + audio silence = confirmed departure, MULTIPLE_PERSONS + audio = group cheating). But each additional rule is another thing that can false-fire during the demo. Two rules that work perfectly are worth more than six rules where two are flaky. If time remains on Day 8-9, a third rule could be added, but it should require the same level of testing as the first two.

4. **The meta-tradeoff of this entire debate process:** Three rounds of multi-model review have consumed significant design time but produced an architecture that is more robust, more deployable, and more focused than the original proposal. The original had 20 features across 3 tiers; the converged plan has 6 core features with explicit fallbacks for every failure mode. The tradeoff is design time vs. implementation time, and for a 10-day sprint, spending 1-2 days on rigorous design review is the right investment. The team now has a spec they can execute with confidence rather than a wish list they'd discover problems with on Day 7.

### GPT-5.2

POSITION: AGREE

REASONING:
Other participants’ feedback pushes the design from “good but risky in practice” to “architecturally sound for a hackathon demo under govt-lab constraints.” The core system is now consistently optimized for the real success criterion: **a stable, credible, low-noise live demo on weak hardware**.

**What changed my position:**  
The combination of (a) a **deployment decision tree** anchored on locked-down Windows realities, (b) making **FPS/thermal degradation modes** explicit, and (c) converging on a **minimal dashboard stack (FastAPI + SSE + vanilla JS/HTMX)** removes the biggest hidden failure modes. At this point, remaining differences are mostly scheduling and implementation details, not architectural direction.

**Common ground (clear convergence across reviews):**
- **Detect → Track → Verify** is the correct CPU architecture for Celeron-class terminals.
- **Cut YOLO** from hackathon scope: too much CPU + too much FP risk + too little demo value.
- **Precision-over-recall** is the correct scoring strategy; **gaze/audio should be fusion-only** (or at most “soft indicators”).
- **App-level time sync** is required for any defensible confusion-matrix scoring.
- **Day-0 packaging spike** is existential; without it the rest of the plan is irrelevant.
- **Confusion-matrix must use 1:1 matching** and alert “episodes,” not raw per-frame spam.

**Where I agree with specific concerns:**
- Claude’s **UI responsiveness under load** is a real risk in single-process PyQt6; you must constrain Python-side work and shed load aggressively.
- Gemini’s **ground truth ergonomics** concern is decisive: manual time entry will break scoring; you need “log now” buttons capturing server time.
- Both Claude and Gemini are right about **enrollment deadlock risk**; you need a proctor override (force proceed / degraded mode) to avoid demo stall.

**Where I slightly disagree (but it doesn’t change the architecture):**
- “Vertical slice by Day 3” is a good aspiration, but for most teams it’s safer stated as:
  - Day 2: mock agents → edge → dashboard,
  - Day 3: real inference loop local-only on 1 terminal,
  - Day 4: real terminal → edge → dashboard with at least face presence + multi-face + episode alerts.
  The key is *not* the calendar day—it’s preventing an “integration cliff” where real models meet networking too late.

CONCERNS:
1. **Integration cliff remains the #1 execution risk:** mock events can create false confidence. Real model inference + event streaming must be integrated early (by Day 4 at latest).
2. **Single-process PyQt6 can still stall the exam UI** if Python-level preprocessing/postprocessing is heavy or if thread scheduling is unmanaged under load.
3. **Ground truth capture must be “single-click = timestamped server event.”** Any manual timestamp workflow will produce unusable confusion-matrix results in a live demo.
4. **Enrollment quality gates vs demo flow:** strict gates prevent poisoning, but without a “force proceed / degraded mode” you can deadlock the entire room on one bad webcam/lighting setup.
5. **Windows dependency landmines (DLL/runtime/media stack):** portable Python + ONNX/MediaPipe/audio can fail on stripped-down Windows images; this must be validated on target machines Day 0.

SUGGESTIONS:
1. **Codify a minimal “Day 4 vertical slice” exit criterion (non-negotiable):**  
   1 terminal: camera → detect/track/verify (even at low FPS) → episode alerts → edge ingestion → dashboard tile + alert feed.
2. **Implement “Jury/Admin Live Logger” in the dashboard:** big buttons (“Swap now”, “Second face now”, “Absent now”) that write GT events with `edge_server_now()`—no free-text time entry during the demo.
3. **Enrollment fallback policy (to avoid demo deadlock):**
   - After N failures (e.g., 3), allow **Force Proceed** but mark terminal as **DEGRADED_ENROLLMENT** and disable face-verify (presence + multi-face only).
4. **UI responsiveness guardrails:**
   - Hard cap CPU usage via degradation modes; minimize Python-side loops; prefer GIL-releasing ops (cv2/onnx); add a watchdog that measures UI tick jitter and forces “SURVIVAL mode” if UI degrades.
5. **Deployment spike additions:**
   - Test on the *actual* OS build. If VC++ redist install is blocked (likely), be prepared with a packaging approach that doesn’t require admin (or a fully self-contained build). Don’t assume you can “just install” dependencies on demo day.

KEY_TRADEOFFS:
- **Reliability over feature breadth:** cutting YOLO and demoting gaze/audio reduces “wow factor,” but prevents alert spam and CPU collapse; for judged live demos, credibility beats coverage.
- **Single-process simplicity vs UI contention risk:** single-process avoids webcam contention and simplifies deployment; you accept the need for strict load shedding and UI watchdogs.
- **Strict enrollment gates vs demo throughput:** gates protect precision; overrides protect the demo schedule. The right compromise is “force proceed into degraded mode,” explicitly surfaced on the dashboard.

### Gemini 3 Pro

POSITION: AGREE

REASONING:
The consensus among all reviewers is now overwhelming. We have successfully moved the proposal from a "feature-rich fantasy" to a "battle-hardened operational plan." The convergence on **WinPython** as the deployment vector, **Detect-Track-Verify** as the compute model, and **Fusion-Only** for noisy signals (Gaze/Audio) eliminates the highest probability failure modes.

The architecture is now robust because it acknowledges the constraints of the hardware (Celeron thermal throttling) and the environment (locked-down BIOS, no Admin rights). The decision to treat **Time Synchronization** and **Thermal Management** as functional requirements rather than optimizations is the winning differentiator.

CONCERNS:
1.  **The "GIL Freeze" (Validated by Claude):** While `threading` works for I/O, heavy numpy/cv2 operations in the inference thread can still stutter the PyQt6 GUI on a single-core Celeron. If the UI lags while a student is typing, it looks like a crash.
2.  **Dashboard "Data Hose" (Validated by GPT-5.2):** Sending full state every frame via SSE will choke the dashboard browser when 30 terminals are active. The projector PC will likely be a standard laptop, not a gaming rig.
3.  **The "Forgot the Content" Risk:** Claude is right—we are so focused on the AI that we might forget to actually write the 10 MCQ questions. A blank exam screen is a demo failure.
4.  **Enrollment Deadlock:** As noted by all, if 30 people try to enroll simultaneously and 5 have bad lighting, the demo stalls.

SUGGESTIONS:
1.  **Implement "Episode" Telemetry (GPT-5.2's Idea):** Do not stream raw frame events. The terminal should define an "Episode" (e.g., `GAZE_DEVIATION` starts at T=10, ends at T=15). Send `ALERT_START` and `ALERT_END` events only. This reduces network traffic by 90% and solves the "flashy dashboard" problem.
2.  **Explicit GUI Yielding:** In the inference thread, insert `time.sleep(0.01)` or `QCoreApplication.processEvents()` (if strictly necessary, though threading is better) after every heavy model run to allow the GUI thread to breathe.
3.  **The "Golden JSON":** Create a `exam_content.json` file on Day 1 with 10 dummy questions. Hardcode it. Do not make this dynamic.
4.  **Enrollment "Bypass" Button:** Add a hidden hotkey (e.g., `Ctrl+Shift+B`) in the enrollment screen to force-pass a student with a dummy embedding. This is the "break glass in case of emergency" for the live demo.
5.  **Standardize the Ground Truth Input:** Use the "Big Button" admin interface I suggested previously to capture server-side timestamps. Do not rely on paper logs.

KEY_TRADEOFFS:
*   **Responsiveness vs. Stability:** We are deliberately adding sleeps/yields and reducing frame rates (Detect-Track-Verify) to keep the UI responsive. The AI might react 200ms slower, but the user experience remains smooth.
*   **Portability vs. Performance:** WinPython is slower to start and has overhead compared to a compiled binary or raw Linux, but it guarantees execution on locked-down Windows PCs.
*   **Granularity vs. Clarity:** "Episode" logging obscures the exact millisecond-by-millisecond AI confidence fluctuation, but provides a clean, readable signal for the dashboard and confusion matrix.

---
## Final Positions

| Model | Position |
|-------|----------|
| Claude Opus | AGREE |
| GPT-5.2 | AGREE |
| Gemini 3 Pro | AGREE |

---
## Synthesis

CONSENSUS_LEVEL: NEAR

COMMON_GROUND:
1. Edge-first architecture (local inference on terminals, event-only uplink to edge server, no cloud dependency for core functionality) is the correct and only viable approach for the stated constraints.
2. Single-process Python terminal agent using PyQt6 for both exam UI and proctoring pipeline — Electron is eliminated entirely to save 300-500MB RAM on 4GB machines.
3. Three-thread architecture on the terminal: Thread 1 (webcam capture → frame queue), Thread 2 (model inference → event queue), Thread 3 (network sender).
4. Detect → Track → Verify is the mandatory baseline inference architecture, not an optimization: BlazeFace at 3-5 FPS for detection, lightweight OpenCV tracker (MOSSE for Celeron, CSRT for i5) between detections, MobileFaceNet verification every 2 seconds or on track loss.
5. YOLOv8-nano is cut entirely from hackathon scope due to COCO domain mismatch, high CPU cost (~80ms/frame on Celeron), false-positive risk on exam materials, dependency complexity, and Ultralytics licensing ambiguity.
6. CCTV processing, Person Re-ID, hash-chain tamper-proof logs, PDF report generation, bias metrics dashboard, and adaptive scheduling are all cut from hackathon scope.
7. Dashboard technology: FastAPI + Server-Sent Events (SSE) + vanilla JS (or HTMX) — Streamlit is rejected due to full-page rerender model causing stutter with 30 concurrent terminal streams; React is rejected due to build-chain complexity under time pressure.
8. Precision over recall is the guiding principle, enforced structurally: fewer false positives are worth more than catching every malpractice event; one false alarm during demo destroys jury credibility in all subsequent detections.
9. Gaze deviation (Face Mesh) must never generate standalone alerts — it is fusion-only, boosting confidence of other detectors.
10. Audio VAD (Silero VAD, not WebRTC VAD) must use very conservative thresholds and is fusion-only; sustained speech >10 seconds only; if unstable on target hardware by Day 4, cut it entirely.
11. The three standalone alert generators are: face absence (0 faces >5s), multi-face detection (>1 face, >5 consecutive frames), and face identity mismatch (3-zone hysteresis with uncertain zone 0.45-0.70 that increases verification frequency without alerting, mismatch >0.70 sustained for 3+ checks).
12. Application-level time synchronization is the portable default: terminal sends local timestamp in heartbeat, edge responds with server timestamp, terminal computes offset using RTT/2, uses median of last 5 offsets, events stamped with corrected time. Monotonic clocks for interval measurement; wall clock only after offset correction.
13. Day 0 packaging spike is an existential go/no-go gate that must be resolved before the sprint begins.
14. Deployment strategy is a decision tree: Plan A = WinPython portable on USB (no admin, no BIOS access needed); Plan B = PyInstaller binary; Plan C = bootable Linux USB (only if BIOS access confirmed). Decision must be made by end of Day 0.
15. CLAHE preprocessing on every frame before face detection to handle uneven exam hall lighting (~2ms overhead).
16. Enrollment quality gates must block progression: face detection confidence >0.9, face size >80px, blur score threshold, brightness range check. Auto-retry with user guidance ("move closer", "face the camera"). After 3 failures, "Force Proceed" button appears, terminal flagged as DEGRADED on dashboard (face verification disabled, presence + multi-face only).
17. Edge server must be a mini-PC with SSD, not a Raspberry Pi with SD card (SD card I/O under 30 concurrent writers causes database locks).
18. SQLite with WAL mode on the edge server as the primary persistence layer, with a single async writer queue to prevent lock contention.
19. Terminal-side JSONL files serve as local store-and-forward buffers for resilience if edge is unreachable; events replayed on reconnect via idempotent POST with event_id.
20. Push-based evidence architecture: terminal pushes encrypted JPEG snapshots to edge via HTTP POST on high/critical alerts only. Dashboard reads evidence from edge. No inbound connections to terminals.
21. Confusion matrix scoring uses strict 1:1 greedy matching: each ground truth event matches at most one detected episode (by nearest timestamp within ±15s window), and vice versa.
22. Common-mode audio rejection at edge: if a high percentage of terminals (e.g., >30%) trigger audio simultaneously, suppress all as ambient noise.
23. LAN-first dashboard hosted on edge server; cloud/4G is optional and not required for core demo functionality.
24. No message broker (NATS, MQTT, Redis) — unnecessary complexity for 30 terminals; in-memory queues and HTTP/SSE are sufficient.
25. Video file fallback mode via `--source` CLI flag (device index or video file path), with the understanding that face verification must be skipped in video mode since enrollment data won't match.
26. Explicit degradation modes: if mic fails → audio disabled; if Face Mesh fails → gaze disabled; if FPS drops → progressive detector shedding. Dashboard shows terminal fault/degradation state.
27. All 5 enrollment face embeddings stored (not just median) and compared against closest match for better pose variation handling.
28. Include VC++ Redistributable and DirectX runtime installers on the deployment USB as zero-cost insurance against stripped Windows Education images.
29. Prepare 10 MCQ exam questions as a hardcoded JSON file on Day 1 — do not make this dynamic or forget it entirely.

CONVERGENCE:
1. Dashboard technology: Claude advocated SSE + vanilla JS from Round 1; GPT-5.2 initially said "team's fastest option" and was open to React/Streamlit; Gemini initially didn't address it, then proposed Streamlit or HTMX. By Round 3, all three converged on FastAPI + SSE + vanilla JS/HTMX as the only approach that guarantees smooth real-time updates for 30 terminals.
2. YOLO scope: The original proposal had object detection as Tier 2. Claude initially accepted it as a Day 7 stretch goal. GPT-5.2 flagged licensing and FP concerns but kept it as a possibility. Gemini argued for complete removal from Round 1. By Round 2, all three agreed YOLO should be cut entirely, not merely deprioritized.
3. Detect → Track → Verify: GPT-5.2 proposed it as an optimization in Round 1. By Round 2, Claude and Gemini endorsed it as the mandatory baseline architecture, not an optimization. By Round 3, all three explicitly agreed it must be designed from Day 1.
4. Deployment strategy: Claude initially favored bootable Linux USB as primary. GPT-5.2 initially listed options without strong ordering. Gemini initially suggested bootable USB, then in Round 2 called Claude's preference "administratively naive." By Round 3, all three converged on WinPython portable as Plan A with a tiered fallback decision tree.
5. Time synchronization approach: Claude initially argued for OS-level chrony. Gemini initially suggested NTP daemon. GPT-5.2 proposed application-level RTT-based offset. By Round 3, all three agreed that application-level sync is the correct default because OS-level approaches require admin access unavailable on locked-down government machines.
6. Event model: Initially all reviewers discussed raw per-frame events with persistence filtering. GPT-5.2 introduced the episode model (ALERT_START / ALERT_UPDATE / ALERT_END with stable episode_id) in Round 4. By Round 5, all three adopted this as the canonical event model for its benefits to dashboard clarity, bandwidth reduction, and confusion matrix scoring.
7. Integration timing: The original proposal integrated on Day 5. Gemini pushed for Day 3 in Round 2. Claude said Day 4-5 initially, then moved to Day 4. GPT-5.2 supported Day 3-4 with mock agents from Day 2. By the final round, all converged on: mock agents by Day 2, real vertical slice target Day 3, hard deadline Day 4.
8. FPS-based thermal/performance degradation: Gemini first raised thermal throttling in Round 3. By Rounds 4-5, all three agreed this is a first-class reliability requirement with explicit shed order (gaze → audio → reduce verify cadence → presence-only survival mode) driven by rolling FPS measurement.
9. Gaze as fusion-only: The original proposal had gaze generating standalone medium/high alerts. Claude suggested demotion early. By Round 2, all three agreed gaze should never generate standalone alerts and serves only to boost confidence of other detectors.
10. Enrollment recovery: Gemini first raised enrollment failure as a critical risk. GPT-5.2 proposed dashboard re-enroll button. Claude proposed quality gates with auto-retry. By Round 5, all converged on: strict quality gates as primary mechanism, "Force Proceed" after 3 failures as the fallback, with the terminal flagged as DEGRADED on the dashboard.

REMAINING_DISAGREEMENTS:
1. **Audio inclusion scope:** Claude and GPT-5.2 agree audio (Silero VAD) should be included with conservative thresholds as a fusion-only signal, with a Day 4 cut decision if unstable. Gemini consistently advocated dropping audio entirely to focus on 3 core features (face presence, identity, multi-face). By the final round, Gemini did not explicitly concede this point, though the gap narrowed as all agreed audio would be fusion-only and cut if problematic. The practical difference is whether audio development is attempted at all.
2. **Exact vertical slice day:** Claude targets Day 4 (with Day 3 aspiration), GPT-5.2 says Day 4 at latest with mock agents from Day 2, Gemini pushes for Day 3 with real models. All agree it must happen before Day 5, but the 1-day spread reflects different risk tolerances. This is a scheduling nuance rather than an architectural disagreement.
3. **Hash-chain logging:** Claude suggested it could be kept as a 2-hour narrative investment. GPT-5.2 and Gemini both called it "security theater" for a hackathon. This was raised in Round 1 and never fully resolved, though it was implicitly dropped from the final scope by all participants.
4. **JSONL on edge as audit artifact:** Claude and GPT-5.2 agree that an optional JSONL append on the edge (in addition to SQLite) is a low-cost audit/recovery measure. Gemini did not explicitly weigh in on this detail. The disagreement is minor — it's about whether to write one extra line of code.

RECOMMENDED_PLAN:

**Pre-Sprint (Day 0) — Go/No-Go Gate**
- [ ] Confirm target OS of demo terminals with the department (Windows version, architecture)
- [ ] Build WinPython portable environment on USB: Python 3.10+, MediaPipe, ONNX Runtime (INT8), OpenCV, PyQt6, Silero VAD, FastAPI, uvicorn
- [ ] Include VC++ Redistributable and DirectX runtime installers on USB
- [ ] Test the portable environment on a non-dev Windows machine: webcam opens, all 4 models load (BlazeFace, MobileFaceNet, Face Mesh, Silero VAD), mic captures audio
- [ ] If WinPython works → Plan A confirmed; if fails → test PyInstaller binary (Plan B); if fails → test bootable Linux USB (Plan C, only if BIOS access confirmed)
- [ ] Test FastAPI + SSE "hello world" serving a static HTML page with EventSource
- [ ] Decision: Go (deployment method confirmed) or No-Go (rethink approach)

**Architecture Decisions (Codified)**
- [ ] Terminal agent: single Python process, PyQt6 for UI, 3 threads (capture, inference, network) with queue.Queue
- [ ] Inference pattern: Detect → Track → Verify (BlazeFace 3-5 FPS → MOSSE/CSRT tracker → MobileFaceNet every 2s or on track loss)
- [ ] Alert taxonomy: 3 standalone alert types only: CANDIDATE_ABSENT (>5s), MULTIPLE_PERSONS (>5 consecutive frames), FACE_MISMATCH (distance >0.70 sustained 3+ checks with 3-zone hysteresis)
- [ ] Fusion-only signals: Gaze deviation (Face Mesh), Audio activity (Silero VAD) — these never generate standalone alerts
- [ ] Exactly 2 hardcoded fusion rules on edge server: (1) GAZE_DEVIATION episode + FACE_MISMATCH episode within ±5s → CONFIRMED_IMPERSONATION; (2) SUSPICIOUS_AUDIO episode + GAZE_DEVIATION episode within ±5s → POSSIBLE_CONSULTATION
- [ ] Episode-based event model: terminals emit EPISODE_START / EPISODE_END with stable episode_id; schema: {episode_id, terminal_id, student_id, alert_type, severity, started_at, ended_at, peak_confidence, trigger_count, corroborating_episodes}
- [ ] Edge server: mini-PC with SSD, SQLite WAL with single async writer queue, FastAPI + SSE + vanilla JS dashboard
- [ ] Dashboard SSE sends delta messages only (episode start/end events + periodic heartbeat per terminal: {terminal_id, fps, mode, risk_level})
- [ ] FPS-based degradation: rolling 30-frame FPS average → ≥12 FPS: full mode; ≥7: shed gaze; ≥4: shed audio; <4: presence + multi-face only (survival mode). Dashboard shows current mode per terminal
- [ ] Time sync: app-level RTT/2 offset estimation on each heartbeat (every 5s), median of last 5 offsets, monotonic clock for intervals, corrected wall clock for event timestamps
- [ ] Confusion matrix: 1:1 greedy matching of ground truth events to detected episodes by nearest started_at within ±15s window
- [ ] Common-mode audio rejection at edge: if >30% of terminals trigger audio within 500ms, suppress all as ambient
- [ ] Create config.yaml with all tunable thresholds, persistence windows, severity mappings — hot-reloadable without restart

**Sprint Schedule**
- [ ] **Day 1:** Webcam capture with threading + BlazeFace face detection + CLAHE preprocessing + `--source` CLI flag (device or video file). Create exam_content.json with 10 MCQ questions. Write episode event schema as shared document. GIL-awareness test: PyQt6 window updating a label every 100ms + background ONNX inference loop — confirm no stutter.
- [ ] **Day 2:** Face enrollment flow (5-frame capture, 5 embeddings stored, quality gate: confidence >0.9, face >80px, blur/brightness checks, auto-retry with guidance, Force Proceed after 3 failures). Face verification with 3-zone hysteresis. Build mock/dummy agent that sends synthetic episode events to edge server via HTTP POST. Start edge server (FastAPI, SQLite WAL, single writer queue).
- [ ] **Day 3:** Multi-face detection with persistence filtering. Candidate absence detection. Gaze tracking via Face Mesh (fusion-only, ROI-scoped, 1-2 FPS). Integrate FPS-based degradation into inference loop. Connect real terminal (1 unit) to edge server — TARGET: real camera → real inference → real episodes → edge ingestion → basic dashboard tile showing state. This is the vertical slice target.
- [ ] **Day 4 (HARD DEADLINE for vertical slice):** If vertical slice not working, stop all feature work and debug integration exclusively. If working: add Silero VAD with conservative thresholds (fusion-only, sustained speech >10s only). Implement app-level time sync. Finalize store-and-forward (JSONL local spool, idempotent POST with event_id, replay on reconnect).
- [ ] **Day 5:** Dashboard frontend: terminal grid (30 colored tiles updated via SSE by terminal ID), live alert feed (episode start/end events appended), terminal detail view (click to expand). PyQt6 exam UI (10 MCQs from JSON, navigation, answer persistence, timer). Exam start/stop synchronization via edge broadcast. Ground truth live logger: /admin/ground-truth page with large buttons per malpractice type + terminal dropdown, capturing server_now() on click.
- [ ] **Day 6:** Evidence snapshot push (JPEG to edge on high/critical episodes). Common-mode audio rejection at edge. Dashboard evidence viewer (load snapshots on demand). 2 fusion rules implemented on edge server.
- [ ] **Day 7:** Confusion matrix generator (1:1 greedy matching, reads from episodes + ground truth tables). Post-exam summary view. Multi-terminal testing (3-5 terminals simultaneously).
- [ ] **Day 