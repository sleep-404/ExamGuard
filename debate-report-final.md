# Multi-LLM Debate Report

**Models:** Claude Opus, GPT-5.2, Gemini 3 Pro
**Rounds completed:** 3

---
## Round 1

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

This is a remarkably thorough architecture for a hackathon project. The core design decisions — headless Python agent, edge-local processing, ONNX INT8 models, episode-based events — are sound and well-suited to the constraints. But the plan's greatest risk is its own ambition. Let me go through each axis critically.

**Accuracy (8/10):**
The six alert types cover the explicitly mentioned malpractice scenarios well. The thresholds are generally reasonable, but I have specific concerns:

- The 3-zone hysteresis for face verification (0.45/0.70) is well-designed but the thresholds need empirical tuning. MobileFaceNet+ArcFace cosine similarity distributions vary significantly with lighting, camera quality, and INT8 quantization error. On government webcams with fluorescent lighting, you may see the entire distribution shift. You need a 15-minute calibration session built into Phase 2 to validate these thresholds on actual hardware.
- Gaze deviation at 30° with Face Mesh is ambitious on 640x480. MediaPipe Face Mesh accuracy degrades substantially below 720p, especially for yaw estimation beyond 25°. The calibrated baseline (median+MAD) is smart and partially compensates, but I'd widen the initial threshold to 35° and rely more on duration than angle precision.
- The 5-second absence threshold is correct for the demo but might be too aggressive for real exams (student drops pencil, bends down). For the hackathon demo specifically, it's fine since events are choreographed.
- **Blind spot:** Unauthorized material detection is listed in the challenge requirements ("unauthorized materials") but not addressed. A phone on the desk or a cheat sheet won't trigger any of these detectors. This is a potential miss in the 8/10 count if the jury includes a "student reading from phone" scenario. Even a simple object detection for rectangular objects (phone/paper) in the lower frame region using color segmentation or a tiny MobileNet-SSD would help.

**Performance:**
The ~430MB memory budget is tight but plausible. Here's the breakdown I'd expect: Python runtime (~40MB), ONNX Runtime (~80MB), OpenCV (~60MB), MediaPipe (~50MB), model weights (~13MB), frame buffers and working memory (~50MB), OS and overhead (~100MB). That's ~393MB, leaving ~37MB headroom on a 4GB system assuming ~3.5GB usable after Windows. This is razor-thin.

- The 3-thread architecture (capture, inference, network) is correct for Python given the GIL. The inference thread is CPU-bound but ONNX Runtime releases the GIL during inference calls, so you get real parallelism where it matters. Multiprocessing would add ~200MB overhead per process for the duplicated Python runtime and is the wrong call here.
- **Critical risk:** ONNX Runtime + MediaPipe in the same process can have DLL/shared library conflicts on Windows, especially with the portable Python approach. Test this in Phase 0 or you'll discover it on demo day.
- MOSSE is the right tracker choice over CSRT. CSRT is 5-10x slower and you need those cycles elsewhere. MOSSE's lower accuracy is acceptable since you re-verify with MobileFaceNet every 2 seconds anyway.
- The detect-track-verify pipeline with BlazeFace at 3-5fps and tracking in between is a well-established pattern. The 2-second re-verification interval is a good balance.

**Innovation:**
The cross-terminal seat-swap detection is genuinely impressive for a hackathon and directly addresses a jury requirement. The common-mode audio rejection is clever and defensible. The hash-chained audit trail is solid for the "tamper-proof" requirement. The confusion matrix during demo is an excellent move — it turns the demo from subjective to quantitative.

However, the "wow factor" could be stronger. The jury is government education officials who may not appreciate the elegance of common-mode audio rejection. They will appreciate things they can *see*.

**Model Choices:**
BlazeFace is fine but MediaPipe's built-in face detection (which *is* BlazeFace under the hood) would reduce integration complexity since you're already using MediaPipe for Face Mesh. Using MediaPipe's holistic pipeline could simplify the code and reduce memory by sharing the face detection result. However, this couples your detection to MediaPipe's pipeline which is less flexible. For a hackathon, the reduced code complexity might be worth it.

MobileFaceNet+ArcFace in INT8 is the right call. InsightFace's implementation is well-tested. No better lightweight alternative exists as of 2025 for this specific task.

Silero VAD is excellent — small, accurate, well-maintained. No notes.

**Architecture:**
The store-and-forward design is good but may be over-engineered for a demo with a reliable LAN. I'd implement the happy path first and add store-and-forward only if Phase 4 time permits.

SQLite WAL + JSONL dual storage is pragmatic. SQLite WAL handles concurrent reads from the dashboard while the JSONL provides the audit trail. Smart.

CONCERNS:

1. **Unauthorized material detection gap.** The challenge explicitly mentions "unauthorized materials" and the architecture has no detector for phones, cheat sheets, or earpieces. If even one of the 10 test events is "student looks at phone on desk," you drop to 7/10 or below.

2. **Memory cliff on 4GB machines.** 37MB headroom is dangerously thin. Windows background services, antivirus (government machines likely have mandatory AV), and Windows Update can easily consume that. One memory spike during ONNX inference with a complex frame and you hit swap, which on a Celeron with a slow HDD means multi-second freezes, not graceful degradation.

3. **MediaPipe Face Mesh accuracy at 640x480.** Yaw/pitch estimation from 478 landmarks at this resolution has significant noise. The 30° gaze threshold may produce both false positives and false negatives depending on camera quality. The calibration helps but doesn't eliminate the resolution limitation.

4. **ONNX Runtime + MediaPipe DLL conflicts on Windows portable Python.** This is a known issue. MediaPipe bundles its own protobuf and OpenCV, which can conflict with your standalone OpenCV and ONNX Runtime's protobuf dependency. Phase 0 must explicitly test this combination.

5. **Webcam access conflicts.** On government Windows machines, only one process can typically access a webcam. If the exam platform (browser) is also using the webcam, your agent can't access it. The architecture doesn't address this. You may need to capture from the agent and serve frames to the browser, or use a virtual camera approach.

6. **Demo fragility with 30 terminals.** If even 3-4 machines fail to run the agent (AV blocking, missing DLLs, webcam issues), the demo looks bad. The pre-flight diagnostic is mentioned but needs to be extremely thorough and run well before demo time.

7. **Scope creep risk.** The Phase 4 features (common-mode audio rejection, confusion matrix, CCTV stub, store-and-forward) are nice-to-haves. The milestone gate is smart, but the temptation to start Phase 4 features before Phase 1-2 are rock-solid is real. The architecture document should explicitly mark which features are demo-critical vs. demo-enhancing.

8. **Ground truth comparison assumes synchronized clocks.** The confusion matrix compares AI detections against ground truth timestamps. If there's even a 3-5 second clock drift between the ground truth logger and the terminal, events will appear as false positives/negatives. NTP sync on a LAN should be fine, but the edge server's time sync authority role needs to be more than a stub.

SUGGESTIONS:

1. **Add a simple unauthorized object detector.** Use color segmentation + contour analysis to detect rectangular objects (phones/papers) in the desk region of the frame. This doesn't need a neural network — simple CV techniques on the lower 40% of the frame can detect a phone screen's blue-white glow or a paper's white rectangle against a desk. Budget: 4-6 hours. This plugs a critical gap in the 8/10 target.

2. **Reduce memory footprint by using MediaPipe's unified pipeline** for face detection + mesh instead of separate BlazeFace + Face Mesh. This shares internal buffers and eliminates one model load. Saves ~20-30MB and reduces DLL conflict surface.

3. **Add a dramatic visual "timeline" to the dashboard.** Government jury members respond to visual proof. A horizontal timeline per terminal showing green (normal), yellow (warning), red (alert) segments with clickable evidence thumbnails is both visually impressive and functionally useful. This is ~6-8 hours of frontend work and massively improves demo impact.

4. **Build a "golden demo" fallback.** Pre-record a 3-terminal demo with perfect detections. If hardware fails on demo day, you can replay this as a video with the dashboard showing live reactions to pre-recorded events. This is insurance, not dishonesty — present it as "here's what it looks like at scale" while doing a live demo on whatever terminals work.

5. **For the webcam conflict issue**, capture video in the Python agent and expose a `/video_feed` MJPEG endpoint that the browser exam UI can use via `<img>` tag. This way the agent owns the camera and the browser doesn't need direct access. This also lets you overlay detection visualizations.

6. **Tighten the degradation mode thresholds.** The jump from FULL (≥12fps) to SHED_GAZE (≥7fps) is large. Consider an intermediate mode at 10fps that reduces gaze check frequency from every frame to every 3rd frame rather than eliminating it entirely.

7. **For Phase 0, create a single-script hardware compatibility test** that tests: webcam access, ONNX Runtime load, MediaPipe load, available RAM after loading, sustained FPS for 30 seconds, microphone access, and network connectivity to edge. Output a simple PASS/FAIL with specific failure reasons. This script should run in under 60 seconds and be the very first thing executed on every terminal.

8. **Consider replacing Face Mesh gaze estimation with a simpler iris-tracking approach.** MediaPipe's iris landmark model (a subset of Face Mesh) gives direct iris position relative to eye corners, which is more robust for gaze direction than full head pose estimation from 478 landmarks. It's also lighter computationally.

KEY_TRADEOFFS:

1. **Complexity vs. reliability.** The architecture tries to be comprehensive (6 alert types, 3 fusion rules, 4 degradation modes, dual storage, cross-terminal detection) at the cost of having many failure surfaces. For a hackathon, 4 rock-solid detectors beat 6 flaky ones. The milestone gate partially addresses this, but the temptation to spread effort thin is real. **This tradeoff leans slightly wrong** — I'd rather see 4 detectors at 99% reliability than 6 at 85%.

2. **Edge processing vs. terminal processing.** Fusion rules and seat-swap on the edge is architecturally correct (the terminal shouldn't need to know about other terminals). But it adds a network dependency for the most impressive feature. If the LAN hiccups during demo, seat-swap detection fails silently. **Right tradeoff overall**, but needs explicit fallback behavior (terminal should show "network degraded" rather than silently missing detections).

3. **INT8 quantization vs. accuracy.** INT8 saves ~4x memory and ~2x inference time but introduces quantization error, particularly for face embeddings where small cosine distance differences matter. The 3-zone hysteresis helps absorb this, but the thresholds may need to be wider than with FP32 models. **Right tradeoff** for the hardware constraints, but demands empirical threshold tuning on quantized models specifically.

4. **Privacy-first architecture vs. demo visibility.** Keeping face images local and only sending metadata to the edge is correct for DPDP compliance and the real deployment. But for the demo, the jury wants to *see* the face match/mismatch, the gaze deviation visualization, the bounding boxes. The architecture handles this via the terminal's local browser UI, but the edge dashboard can only show thumbnails pushed as evidence. **This is the right tradeoff** — show the terminal UI on a projected screen during demo for the visual impact, and show the dashboard for the aggregate view.

5. **Portable Python (WinPython) vs. PyInstaller.** WinPython is more debuggable and flexible; PyInstaller produces a cleaner deployment. For a hackathon where you'll be debugging on-site, **WinPython as Plan A is the right call.** But test the PyInstaller build early — don't discover packaging issues on demo day.

6. **No unauthorized material detection vs. implementation time.** The current plan omits this to save complexity. But given the challenge explicitly requires it and 10 events need 8 detections, dropping this category means you're betting that none of the 10 events involve unauthorized materials. **This is the wrong tradeoff.** Even a simple heuristic detector is better than nothing.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
The plan is directionally strong for a hackathon POC: it prioritizes the three easiest-to-score, high-precision events (absence, multiple persons, seat-swap) and uses edge aggregation for the “wow” factor. The detect→track→verify pipeline with graceful degradation is also architecturally sound for weak hardware, and the episode-based event model is the right abstraction to reduce false positives.

Where it weakens is (a) reliability of gaze/audio on real exam hardware/environments, (b) runtime/memory risk from mixing MediaPipe + ONNX Runtime + OpenCV in Python on 4GB machines, and (c) some under-specified thresholds and security/operational details that will directly affect “8/10 with minimal false positives”.

Accuracy (8/10) is realistic **only if the 10 controlled malpractice events are biased toward what you detect well** (absence, multi-face, seat swap, sustained talking). If the test includes “phone use”, “looking down at notes”, “off-screen consultation”, “impersonation via photo/video replay”, or “whispering”, your current stack will miss too many or generate false positives unless you add at least one of: object/material detection, lightweight liveness, or stronger audio classification.

Performance-wise, the *intended* compute plan is reasonable, but the *implementation choice* (Python + MediaPipe Face Mesh) is the biggest landmine. Face Mesh is often the dominant CPU cost; on Celeron-class CPUs it can push you into SHED_GAZE/SURVIVAL too often, which then undermines your “innovation” detections.

Innovation-wise, cross-terminal seat swap + tamper-evident audit + degradation modes + live confusion matrix is already differentiating. The risk is that the jury values “caught real cheating modes” over cryptographic/audit novelty; without a “phone/material” story, you may look incomplete relative to the problem statement.

CONCERNS:
1. **MediaPipe Face Mesh performance on Celeron/4GB**: Face Mesh can dominate CPU and memory; combined with ORT + OpenCV you may not sustain stable FPS, causing frequent mode shedding and inconsistent detections.
2. **Threading model in Python**: 3 threads can still bottleneck due to GIL/contention and frame queue backpressure. If any stage blocks (camera read, audio read, inference), latency spikes and “>5s absent” can false-trigger.
3. **Face verification thresholds are under-specified / possibly inverted**: The “<0.45 match, 0.45–0.70 uncertain, >0.70 sustained” only makes sense if those are cosine similarities; if it’s a distance, it’s reversed. Without calibration on your actual capture conditions, seat-swap and mismatch will either miss or false-alarm.
4. **No explicit “phone/material/book” detection**: This is a major cheating mode and is explicitly implied by “unauthorized materials”. Relying on posture/gaze alone will miss “phone below camera” or “notes on desk”.
5. **No liveness/anti-spoof**: A printed photo / phone replay can defeat MobileFaceNet verification. Even a cheap blink/head-micro-motion check would materially reduce that risk.
6. **Common-mode audio rejection can suppress real cheating**: If ≥3 terminals trigger, you suppress most—this can hide coordinated collusion (which is exactly when you want stronger alerts). Also mic gains differ wildly, making “highest amplitude” unreliable.
7. **Audio capture reliability on government desktops**: Driver permissions, default device selection, exclusive-mode conflicts, and disabled microphones are common. Your “high confidence” claim for audio is optimistic operationally.
8. **Local HTTP/WS UI attack surface**: If the student can access the local UI/API, they can attempt to stop the agent, flood it, or tamper with evidence. Binding to 0.0.0.0 or predictable ports is risky.
9. **Tamper-proof hash chain is not actually tamper-proof without anchoring**: A local attacker can delete/replace the entire log and recompute hashes unless you periodically anchor to edge server (or sign with a key not accessible to the user).
10. **Store-and-forward disk growth**: Evidence clips + snapshots across 30 terminals can fill disks quickly. No quota/retention policy is specified.
11. **Time sync and ground truth alignment risk**: “Time sync authority” is stated, but if you rely on wall-clock timestamps without monotonic+offset handling, your confusion matrix and seat-swap correlation can be off by seconds.
12. **CCTV component looks token**: A “stub” may be judged as not meeting “integrate CCTV feeds” unless the dashboard shows at least some meaningful CCTV-derived signal (even simple motion/occupancy).

SUGGESTIONS:
1. **Unify runtime: pick ORT-only or MediaPipe-only for face pipeline**
   - If performance is tight, strongly consider using **MediaPipe Face Detection + lightweight landmarks/head pose** *or* go fully ORT with a small landmark model. Mixing heavy stacks increases memory and failure probability.
2. **Replace Face Mesh with a cheaper head-pose estimator**
   - For gaze/posture, you can get “looking away/down” with far fewer landmarks:
     - Use a small 68/106-landmark model (quantized ONNX) + solvePnP for yaw/pitch.
     - Or use MediaPipe’s lighter face landmark options (if available in your chosen build) and run it at low duty-cycle (e.g., 1–2 FPS) with temporal smoothing.
3. **Add a minimal “phone/material” detector (quick-win, big scoring)**
   - Even one class (“cell phone”) changes the demo narrative. Use a lightweight INT8-friendly detector (e.g., MobileNet-SSD / tiny YOLO alternative if allowed) at **very low FPS (0.5–1 FPS)** and only in the lower half of the frame to control compute.
   - If you must avoid “YOLO”, call it “MobileNet-SSD object detector” explicitly.
4. **Add lightweight liveness (≤4 hours if scoped tightly)**
   - Use Face landmarks for **blink rate / eye aspect ratio** + a simple challenge (“look left/right”) during enrollment only, or passive liveness via micro-motions. This directly hardens seat-swap/impersonation.
5. **Rework audio “common-mode rejection” into classification**
   - Instead of suppressing when ≥3 trigger, label it as **ROOM_NOISE_EVENT** vs **LOCAL_SPEECH_EVENT**:
     - Compare each mic’s VAD energy against its own rolling baseline; if many trigger simultaneously, downgrade severity but do not discard.
6. **Make thresholds data-driven (must do before coding hardens them)**
   - Run a 30–60 minute capture session on the target hardware and compute:
     - Face verify genuine/impostor similarity distributions → set threshold by desired FAR.
     - Typical head pose variance during “normal exam” → set gaze deviation thresholds using baseline+MAD but also clamp to sane minima.
7. **Process architecture: favor a single realtime loop + bounded queues**
   - Use one capture thread and one inference worker with **bounded queue size = 1** (drop frames) to avoid latency explosion.
   - If you keep 3 stages, make them **non-blocking** and measure end-to-end latency; stale frames are worse than low FPS.
8. **Harden local agent controls**
   - Bind APIs to **localhost only**, require a random per-session token, and run the agent as a background user with minimal privileges. If kiosk-mode is possible, do it.
9. **Anchor the hash chain**
   - Every N events, send a chain tip hash to edge; edge stores it immutably. This makes “delete and recompute locally” detectable.
10. **Add evidence retention/quotas**
   - E.g., keep last 10 minutes of circular buffer video locally; only export clips for confirmed alerts. Enforce max disk usage per terminal.
11. **CCTV minimal meaningful feature**
   - Implement simple **motion heatmap + aisle/desk occupancy anomalies** (no face ID) and show it on dashboard. It satisfies “CCTV integrated” without heavy AI.
12. **Demo resilience**
   - Have a “replay mode” where terminals can stream from prerecorded clips if webcam/audio fails. Label it clearly as fallback; jury cares that the system behavior is correct.

KEY_TRADEOFFS:
- **Precision over coverage**: You chose events that are easier to detect reliably (absence/multi-face/seat-swap) and avoided harder but common cheating modes (phone/material). This helps “minimal false positives” but risks failing the 8/10 if the test includes those modes.
- **Edge intelligence vs terminal simplicity**: Cross-terminal seat swap and audio correlation on the edge is a good trade for wow/scalability, but it increases dependence on LAN/time sync and introduces correlation failure modes.
- **Innovation vs operational robustness**: Hash-chaining and live confusion matrix are impressive, but the jury will likely reward “caught cheating live on weak PCs” more than audit cryptography. Shifting a bit of effort from audit sophistication to “phone/material + liveness + robust head pose” is the better trade for winning.
- **MediaPipe convenience vs predictable performance**: MediaPipe is fast to integrate but can be heavy/unpredictable on low-end CPUs in Python. A simpler, lower-landmark head pose path is more scalable and stable for your constraints.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The architectural logic regarding **Cross-Terminal Seat-Swap** and **Common-Mode Audio Rejection** is technically brilliant and constitutes a winning differentiator. The decentralized "Store-and-Forward" approach correctly addresses the LAN reliability constraint.

However, the **Performance** strategy is dangerously optimistic. Running a multi-model pipeline (Face+Landmarks+Embeddings+VAD) inside a Python threaded environment on a Celeron with 4GB RAM is a recipe for GIL (Global Interpreter Lock) contention. While ONNX Runtime releases the GIL during inference, the pre-processing (numpy operations, cv2.cvtColor) and post-processing do not. On a dual-core Celeron, the UI (served via local HTTP) will lag, and frame drops will likely cascade into system freezes.

Additionally, you have a gap in **Unauthorized Material Detection**. The fusion rule `POSSIBLE_MATERIAL_USE` exists, but you explicitly excluded object detection (YOLO). Relying solely on gaze/posture to infer material usage is a weak proxy that will lower your detection score below the 8/10 target if the student holds a phone up without looking down.

CONCERNS:
1.  **Python GIL Bottleneck:** The "3-thread" architecture is insufficient for real-time video processing on weak hardware. The capture thread and inference thread will fight for the GIL during pre/post-processing, causing high latency in the API serving thread.
2.  **Memory Starvation:** A modern browser (Chrome/Edge) running the exam UI + the dashboard can easily consume 2GB+. With 4GB total, leaving ~1.5GB for the OS, your Python agent has <500MB. If swapping occurs, 12fps drops to 0.5fps.
3.  **Overkill Gaze Model:** Using MediaPipe Face Mesh (468/478 landmarks) just to get Yaw/Pitch is computationally expensive (approx 10-15ms on i5, 40ms+ on Celeron). You only need 6 points for PnP (Perspective-n-Point) pose estimation.
4.  **CCTV "Stub" Risk:** The jury expects "Multi-modal." A CCTV feed with "no AI" might be seen as non-compliant with the "Integrate CCTV... to detect malpractices" requirement.
5.  **Lighting Sensitivity:** The "Candidate Absent" logic (Face < 5s) will trigger false positives in fluctuating lighting (auto-exposure adjustments) common in cheap webcams.

SUGGESTIONS:
1.  **Architecture Change:** Move the Inference Pipeline to a separate **Process** (using `multiprocessing` or a separate micro-service), not a Thread. Communicate via Shared Memory (minimize serialization overhead). This bypasses the GIL.
2.  **Model Replacement:** Swap MediaPipe Face Mesh for a lightweight **PnP Head Pose** estimator based on the 5-6 landmarks provided by BlazeFace/MobileFaceNet. This saves ~30ms per frame on Celeron.
3.  **Heuristic "Phone" Detection:** Since YOLO is out, implement a "Hand-Face Overlap" check. If skin-tone pixels (or a lightweight hand landmark model) appear in the lower 3rd of the face ROI, flag as `SUSPICIOUS_POSTURE` (phone/eating).
4.  **Lighting Robustness:** Add a pre-check: If `Global Brightness < Threshold` OR `Contrast < Threshold`, pause detection and alert "POOR LIGHTING" instead of "ABSENT."
5.  **CCTV Innovation:** Even if the CCTV has no heavy AI, use simple **Optical Flow** (background subtraction) on the Edge Server to detect "High Commotion" (running/fighting) in the back of the room. This is computationally cheap and satisfies the "AI on CCTV" requirement.

KEY_TRADEOFFS:
**Precision vs. Recall on Materials:** By excluding object detection (YOLO), you trade the ability to explicitly detect phones for higher FPS. This is the correct tradeoff *only if* you implement strict posture/hand heuristics to compensate.
**Edge vs. Cloud:** The heavy reliance on the Edge node for seat-swap and audio fusion creates a single point of failure (if the Mini-PC dies, the advanced features die), but this is acceptable given the 4G constraint.

---
## Round 2

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After reviewing the other participants' feedback, my position strengthens on several points and shifts on a few. There's substantial convergence across all three reviews, which itself is a signal about where the real risks lie.

**Where I've shifted:**

1. **The GIL/threading concern is more serious than I initially weighted.** GPT-5.2 and Gemini both flag this, and Gemini's point about pre/post-processing (numpy, cv2) holding the GIL is correct. I initially said "ONNX Runtime releases the GIL during inference calls, so you get real parallelism where it matters" — but that's only true for the inference call itself. The CLAHE preprocessing, frame resizing, embedding normalization, and threshold comparisons all hold the GIL. On a dual-core Celeron, this is a real bottleneck. However, I **disagree** with Gemini's suggestion to use `multiprocessing` — the ~200MB memory overhead of a second Python process on a 4GB machine is fatal. The right answer is: keep 3 threads, but use a **bounded queue of size 1** (as GPT-5.2 suggests) so stale frames are dropped rather than queued, and accept that you'll get 5-8fps on Celeron rather than 8-10fps. The degradation modes already handle this.

2. **Unauthorized material detection is now a consensus critical gap.** All three of us flagged it independently. This is no longer a "nice to have" — it's a must-have. The question is implementation approach:
   - I suggested color segmentation + contour analysis (simple CV, no model).
   - GPT-5.2 suggested MobileNet-SSD at 0.5-1fps on the lower frame half.
   - Gemini suggested skin-tone "hand-face overlap" heuristics.
   
   I now think the **best approach is a combination**: skin-tone detection in the lower face ROI (Gemini's idea — cheap, no model needed) as the primary signal, with a brightness/glow detector for phone screens. This avoids adding another model to the memory budget while still catching the two most common material cheating modes. MobileNet-SSD at 0.5fps is appealing but adds ~20-30MB of model memory we can't afford.

3. **The CCTV stub needs to be more than a stub.** Both GPT-5.2 and Gemini flag this. Gemini's optical flow / background subtraction suggestion for "high commotion" detection is exactly right — it's computationally trivial on the edge server (which has resources to spare), requires no face detection (privacy-compliant), and satisfies the "AI on CCTV" requirement. This should be promoted from Phase 4 to Phase 3. Budget: 3-4 hours.

**Where I hold firm:**

1. **Face Mesh vs. lightweight head pose.** Gemini suggests swapping Face Mesh for PnP from 5-6 BlazeFace landmarks. GPT-5.2 suggests a 68-landmark ONNX model + solvePnP. I partially agree — Face Mesh is heavy — but there's a middle ground. My original suggestion of **iris tracking** (a subset of Face Mesh that MediaPipe already provides) gives you actual gaze direction, not just head pose. Head pose from 5-6 landmarks cannot detect "eyes looking sideways while head faces forward," which is the most common subtle cheating behavior. I'd now refine my position: use MediaPipe Face Mesh but **run it at 1-2fps duty cycle** (not every frame), with temporal smoothing. This gives you iris-level gaze accuracy without the continuous CPU cost. On frames where Face Mesh doesn't run, use the cheap 5-point head pose as a coarse filter.

2. **Multiprocessing is wrong for 4GB.** Gemini advocates for it; I maintain it's the wrong call. A second Python process duplicates the interpreter, imported libraries, and ONNX Runtime — easily 150-200MB. On 4GB with Windows + browser + exam platform, you cannot afford this. Shared memory communication also adds complexity and debugging difficulty during a hackathon. The bounded-queue threading approach is the pragmatic choice.

3. **The architecture's overall ambition is still the primary risk.** All three reviews converge on "too much, too fragile." The plan needs a clearer **kill list** — features that are explicitly abandoned if time runs short, ordered by sacrifice priority.

**Where I agree with others but want to sharpen:**

- GPT-5.2's liveness/anti-spoof point is valid but I'd **scope it to enrollment only** — a simple blink detection during the 5-capture enrollment (eye aspect ratio from the landmarks you already have). Cost: 2 hours. Value: blocks the simplest photo attack on seat-swap verification. Don't do continuous liveness — it's too expensive.

- GPT-5.2's point about common-mode audio rejection suppressing coordinated collusion is excellent and I missed it. The fix is simple: don't suppress, **reclassify**. If ≥3 terminals trigger simultaneously, label it `ROOM_NOISE` with low severity but keep it in the log. If exactly 2 adjacent terminals trigger, escalate to `POSSIBLE_CONSULTATION`. This preserves the innovation while fixing the false-negative risk.

- GPT-5.2's concern about hash chain tamper-proofness is technically correct (local attacker can recompute) but practically irrelevant for a hackathon demo. Periodic anchoring to edge is a 1-hour addition if time permits.

- Gemini's lighting robustness point about cheap webcam auto-exposure causing false ABSENT triggers is a real risk I hadn't considered. The fix is simple: track the frame's mean brightness alongside face detection. If brightness drops below threshold, emit `POOR_LIGHTING` rather than `CANDIDATE_ABSENT`. Cost: 30 minutes.

CONCERNS:

1. **Unauthorized material detection remains unaddressed in the plan.** Consensus across all reviewers. Must be added to Phase 2 or early Phase 3 using skin-tone heuristics + phone screen glow detection. No new models.

2. **Memory budget has no safety margin.** All reviewers agree ~430MB is razor-thin. Browser memory consumption (which Gemini correctly notes can be 1-2GB for Chrome) was not in my original estimate. If the exam UI runs in Chrome on the same machine, the actual available memory for the Python agent may be closer to 300MB. This needs a hard measurement in Phase 0.

3. **Face Mesh CPU cost on Celeron without duty-cycling.** Running Face Mesh every frame at 640x480 will dominate CPU. All reviewers agree it's too heavy. Must be duty-cycled to 1-2fps with temporal smoothing.

4. **CCTV component is too thin for jury expectations.** Both other reviewers flagged this. Background subtraction + motion heatmap on edge is cheap and necessary.

5. **Common-mode audio rejection as currently designed can hide real collusion.** GPT-5.2's point is sharp. Reclassify rather than suppress.

6. **No liveness check makes seat-swap vulnerable to photo attacks.** GPT-5.2 raised this. Enrollment-time blink detection is a cheap mitigation.

7. **Webcam exclusivity conflict with browser exam platform.** My original concern, not addressed by other reviewers but still critical. If the exam platform opens the webcam in the browser, the Python agent can't access it. The `/video_feed` MJPEG proxy approach must be part of the core architecture, not an afterthought.

8. **The plan lacks an explicit priority-ordered kill list.** All reviewers agree scope is too large. The architecture needs a table: Feature | Priority (P0/P1/P2) | Cut if behind by X hours.

SUGGESTIONS:

1. **Create an explicit feature priority table immediately:**
   - **P0 (demo fails without):** CANDIDATE_ABSENT, MULTIPLE_PERSONS, FACE_MISMATCH, enrollment flow, edge dashboard with alert feed, pre-flight diagnostic
   - **P1 (8/10 at risk without):** GAZE_DEVIATION (duty-cycled), SUSPICIOUS_AUDIO, seat-swap cross-terminal, unauthorized material heuristic, lighting robustness check
   - **P2 (innovation/wow):** Confusion matrix, common-mode audio reclassification, hash-chained audit, CCTV motion heatmap, enrollment liveness, store-and-forward, timeline visualization
   - Rule: No P2 work begins until all P0 items pass a live 3-terminal test.

2. **Implement the "dual gaze" strategy:** Face Mesh at 1-2fps for iris-level gaze accuracy + 5-point head pose at full framerate as a coarse filter. If coarse filter shows head centered, skip Face Mesh that cycle. This gives accuracy where needed and saves CPU otherwise.

3. **Add unauthorized material detection as a P1 feature** using: (a) skin-tone blob detection in lower-face ROI for "hand near face holding object," (b) bright rectangle detection in lower frame for phone screens. No neural network. Budget: 4 hours.

4. **Phase 0 must measure actual available RAM** after launching: Windows + antivirus + Chrome with exam UI + Python interpreter with all imports loaded (before inference). If <350MB remains, the architecture needs to shed a model. Know this before writing detection code.

5. **Rework common-mode audio** from suppress → reclassify. Simultaneous triggers across ≥3 terminals become `ROOM_NOISE` (low severity, logged). Two adjacent terminals become `POSSIBLE_CONSULTATION` (elevated severity).

6. **Add CCTV background subtraction + motion heatmap to Phase 3** (not Phase 4). OpenCV's `BackgroundSubtractorMOG2` + frame accumulation on the edge server. Display as an overlay on the CCTV tile in dashboard. Budget: 3-4 hours.

7. **Build the `/video_feed` MJPEG proxy into Phase 1** as part of the core agent. The agent owns the camera; the browser exam UI consumes the feed via `<img src="http://localhost:PORT/video_feed">`. This solves webcam exclusivity and enables detection overlay visualization for the jury.

8. **Add Gemini's lighting check:** Track per-frame mean brightness. If it drops below a calibrated threshold for >2 seconds, emit `POOR_LIGHTING` instead of `CANDIDATE_ABSENT`. This prevents false positives from auto-exposure, flickering lights, or hand-over-camera scenarios. Budget: 30 minutes.

KEY_TRADEOFFS:

1. **Threads vs. multiprocessing on 4GB.** All reviewers acknowledge the GIL problem. Gemini advocates multiprocessing; GPT-5.2 and I advocate bounded-queue threading. The tradeoff is: multiprocessing gives true parallelism but costs ~200MB of irreplaceable memory. Threading limits CPU utilization but preserves memory headroom. **On 4GB machines, memory is the harder constraint.** Threading with frame-dropping is the right call. If Phase 0 reveals >500MB available, revisit.

2. **Face Mesh accuracy vs. CPU cost.** I originally suggested iris tracking; Gemini suggests 5-point PnP; GPT-5.2 suggests 68-landmark ONNX + PnP. The convergent answer is **duty-cycled Face Mesh** — you get iris-level accuracy (which catches eyes-only cheating that head-pose-only misses) but only pay the CPU cost 1-2 times per second. The tradeoff is a 0.5-1 second latency on gaze detection, which is acceptable given that gaze alerts already require 30 seconds of sustained deviation.

3. **No object detection model vs. heuristic material detection.** All reviewers want material detection; the disagreement is approach. Adding MobileNet-SSD gives higher accuracy but costs ~25MB memory and ~15ms per frame. Skin-tone + glow heuristics cost nearly zero but have higher false positive rates. **For a hackathon on 4GB machines, heuristics are the right call.** If the heuristic fires, you capture an evidence frame that the human proctor can verify on the dashboard — the system doesn't need to be certain, it needs to flag.

4. **Scope ambition vs. demo reliability.** This is the meta-tradeoff that all three reviews identify. The plan has enough features for a production system but needs to ship in hackathon time. The explicit P0/P1/P2 priority list with hard cut rules is the mechanism that makes this tradeoff manageable. **The right tradeoff is: ship P0+P1 reliably, then add P2 features only with proven remaining time.** The original architecture leans toward "build everything, hope it works." That's the wrong side of this tradeoff.

5. **CCTV stub vs. minimal AI on CCTV.** The original plan treats CCTV as display-only. All reviewers agree this underdelivers against "integrate CCTV feeds with AI." Background subtraction is the minimum viable investment. **The right tradeoff is: spend 3-4 hours on motion heatmap to check the "CCTV + AI" box, rather than spending those hours on audit hash-chain anchoring that the jury won't visibly appreciate.**

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position largely stays the same, but two points from others materially increase the “must-fix before coding” list:

- **Webcam ownership conflict** (Claude): if the exam browser also opens the camera, your agent may fail to access it on many Windows builds. That can zero out the entire system on demo day. This is not a corner case; it’s common.
- **Memory cliff realism** (Claude + Gemini): the 430MB agent budget might be “plausible on paper” but is operationally fragile once you add Windows AV + browser RAM + any swapping on HDDs.

There’s strong convergence across all feedback on three themes:
1) **Unauthorized material/phone detection is a scoring gap** that can sink 8/10 if included in the 10 events.  
2) **Face Mesh is overkill** for yaw/pitch and is the biggest performance risk on Celeron.  
3) **Phase 0 preflight + packaging/DLL conflict testing** must be treated as a first-class deliverable, not a checklist.

Where I *partly* disagree with Gemini is the blanket recommendation to move inference to multiprocessing. Multiprocessing can reduce GIL contention, but on 4GB machines it can also be a net loss due to duplicated memory (Python runtime, DLLs, model weights) and higher operational complexity. The better convergence path is: **minimize Python-side pre/post, keep bounded queues, and only split into a second process if measurements prove you’re GIL-bound and you can fit RAM**.

CONCERNS:
1. **Webcam contention with the exam platform**: if the browser locks the camera, the agent can’t capture. This is a top demo-day failure mode.
2. **Material/phone detection gap**: all reviewers agree this is explicitly required and currently missing; it can directly prevent 8/10.
3. **Face Mesh cost vs value**: 478 landmarks at 640×480 is both noisy (accuracy) and expensive (performance). High risk on Celeron.
4. **Memory headroom is too thin in real conditions**: AV + browser + Windows services can push you into swap; swap on slow disks destroys “real time” regardless of graceful degradation logic.
5. **Windows packaging conflicts (ORT + MediaPipe + OpenCV)**: DLL/protobuf conflicts are real; must be validated early on the *exact deployment method*.
6. **Clock sync for confusion matrix / correlation**: drift or mismatch between terminals, edge, and ground-truth logger will “manufacture” false positives/negatives in scoring.
7. **Audio “common-mode suppression” can hide coordinated cheating**: suppression is the wrong semantic; it should be classification/downgrade, not discard.

SUGGESTIONS:
1. **Resolve webcam conflict by making the agent the camera owner**
   - Agent captures frames and serves them to the browser via **local MJPEG/WebRTC** endpoint (read-only). This also makes overlays trivial and reduces “two apps fighting for camera” risk.
2. **Replace Face Mesh with lightweight head-pose / gaze proxy**
   - Use a small landmark set + `solvePnP` for yaw/pitch, run at low duty-cycle (e.g., 2 FPS) with temporal smoothing.
   - Raise initial gaze angle threshold (e.g., **35°**) and lean on duration; 30° at 640×480 is fragile.
3. **Add a “phone/material” signal with minimal compute**
   - Prefer a tiny **MobileNet-SSD INT8** phone/paper detector at 0.5–1 FPS on the lower frame region.
   - If you absolutely can’t add a detector, implement a heuristic fallback: **hand-in-desk-region + sustained downward pitch** → POSSIBLE_MATERIAL_USE (lower confidence).
4. **Treat Phase 0 as a gating deliverable**
   - One script that tests: webcam open, mic open, ORT load, MediaPipe load (if used), RAM after load, sustained FPS 30s, edge connectivity, time sync offset, disk write permission; outputs PASS/FAIL + reason.
5. **Be conservative about multiprocessing**
   - Start with a single-process design with bounded queue (size 1) and aggressively reduce Python pre/post work.
   - Only introduce a second process if profiling shows sustained GIL-bound pre/post and you still meet RAM constraints.
6. **Change “common-mode audio rejection” to “common-mode classification”**
   - Mark as ROOM_NOISE (downgrade severity), but don’t suppress alerts outright.
7. **CCTV stub → cheap “AI-enough”**
   - Add simple motion/optical-flow “commotion index” + timeline visualization so it looks like an actual modality, not a checkbox.

KEY_TRADEOFFS:
- **Threads vs processes**: Gemini optimizes for CPU isolation; Claude optimizes for RAM and simplicity. Common ground: *measure first*. On 4GB machines, “more processes” can be worse than “fewer Python ops.”
- **Gaze sophistication vs robustness**: All feedback converges that full Face Mesh is not the right cost/benefit. A simpler pose estimator improves both accuracy stability and FPS.
- **Innovation vs scoring coverage**: Seat-swap + audit trail is strong, but without a phone/material story you’re betting the test won’t include it. The consensus is that bet is unnecessary and risky—add at least a minimal material signal.
- **Demo wow vs operational survivability**: The best “wow” is a demo that doesn’t fail. Webcam ownership, packaging sanity, and preflight diagnostics are higher leverage than adding more fusion rules.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
There is a strong consensus among all reviewers that the **Performance** strategy (Python threads + MediaPipe Face Mesh on Celeron) is the critical failure point, and the **Missing Material Detection** is the critical functional gap.

I concede to Claude's counter-argument regarding `multiprocessing`: on Windows with 4GB RAM, the memory overhead of spawning a second Python runtime (loading DLLs/libraries twice) is too risky. Therefore, we must optimize the **single-process pipeline** by removing heavy components rather than isolating them.

The **Cross-Terminal Seat-Swap** remains the strongest architectural asset. However, GPT-5.2 correctly identifies a logical flaw in the **Common-Mode Audio Rejection**: suppressing room-wide audio might hide massive collusion (e.g., a proctor dictating answers), which is a "Wow" detection scenario we cannot afford to miss.

CONCERNS:
1.  **Unauthorized Material Gap:** The consensus is unanimous. Without explicit detection for phones/books, we risk scoring <8/10. Relying on gaze to infer material use is a weak proxy.
2.  **Audio Logic Flaw:** The current rule ("suppress if >3 terminals") treats room-wide audio as noise. In a malpractice scenario, this could be an invigilator dictating answers. This logic needs to classify, not suppress.
3.  **Memory/GIL Deadlock:** Even without multiprocessing, the GIL will choke the API thread during heavy inference. A 3-thread model in Python on a dual-core Celeron often results in UI timeouts during heavy processing.
4.  **Lighting Sensitivity:** Confirmed by all reviewers; 5-second absence triggers due to webcam auto-exposure lag will ruin the credibility of the demo.

SUGGESTIONS:
1.  **Pipeline Diet (Crucial):** **Drop MediaPipe Face Mesh.** Replace it with a simple **PnP Solver** (Perspective-n-Point) using the landmarks already provided by the Face Detection/Recognition model (MobileFaceNet/BlazeFace). This recovers ~30-40ms per frame and reduces memory pressure, solving the Celeron bottleneck.
2.  **Add "Tiny" Object Detection:** Integrate a **MobileNet-SSD (v2/v3-small)** or a quantized **NanoDet**. Run this *only* once every 30-60 frames or when `Gaze_Deviation` is detected. This plugs the material detection gap with minimal compute cost.
3.  **Refine Audio Logic:** Change "Common-Mode Rejection" to **"Common-Mode Classification"**. If >3 terminals detect speech, flag as `SUSPICIOUS_ENVIRONMENT` (medium alert) rather than suppressing it. This preserves evidence of collusion while differentiating it from individual whispering.
4.  **Visual Timeline (Demo):** Adopt Claude's suggestion for a **timeline visualization** on the dashboard. It effectively visualizes the "store-and-forward" data and gives the jury an immediate "God Mode" view of the exam hall.

KEY_TRADEOFFS:
**Heuristics vs. Deep Learning (Gaze):** We are trading the high-precision of a dense Face Mesh for the raw speed of PnP algorithms. On a Celeron, this is the mandatory tradeoff to free up CPU cycles for the Material Detector.
**Recall vs. Noise (Audio):** Changing audio rejection to classification increases alert volume (noise) but ensures we don't suppress massive collusion (recall). For a government exam system, High Recall is preferred over Low False Positives for environmental anomalies.

---
## Round 3

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After this second round, we've reached strong convergence on nearly every major issue. The remaining disagreements are narrow and technical rather than architectural. Let me map the consensus and the few genuine remaining disputes.

**Full Consensus (all three reviewers agree, no further debate needed):**

1. **Unauthorized material detection must be added.** Unanimous across all rounds. The only remaining question is implementation approach, addressed below.
2. **Common-mode audio must reclassify, not suppress.** GPT-5.2 identified the flaw, Claude and Gemini both adopted it. The semantic shift from "rejection" to "classification" is settled.
3. **Webcam ownership must be resolved in Phase 0/1.** The agent owns the camera, serves MJPEG to the browser. This is now a P0 architectural requirement, not an integration detail.
4. **Phase 0 preflight is a first-class deliverable.** All reviewers agree it must gate everything else. RAM measurement under realistic conditions (Windows + AV + Chrome + exam UI) is mandatory.
5. **CCTV needs minimal AI on edge.** Background subtraction + motion heatmap, budget 3-4 hours, promoted to Phase 3.
6. **Scope needs an explicit P0/P1/P2 kill list with hard cut rules.** No P2 work until P0+P1 pass a live 3-terminal test.
7. **Lighting robustness check.** Frame brightness tracking to prevent false CANDIDATE_ABSENT. 30-minute fix.
8. **Threading over multiprocessing on 4GB.** Gemini explicitly conceded this point in Round 2. Bounded queue of size 1, accept lower FPS on Celeron.

**Near-Consensus (agreement on the problem, minor variance on solution):**

**A. Face Mesh vs. alternatives for gaze.**
- Gemini says: Drop Face Mesh entirely, use PnP from existing BlazeFace/MobileFaceNet landmarks.
- GPT-5.2 says: Replace with lightweight landmark model + solvePnP, raise threshold to 35°.
- Claude (me) says: Duty-cycle Face Mesh at 1-2fps, use cheap head-pose as coarse filter between cycles.

I'm now willing to **shift further toward the consensus position**, but with an important caveat. Here's my updated stance: **Start with the cheap PnP approach as the default (P0).** If Phase 0 RAM measurements show we have headroom AND the PnP-only gaze proves too coarse during Phase 2 testing (missing eyes-only cheating), THEN add duty-cycled Face Mesh as a P1 upgrade. This is the "measure first" philosophy that all three of us advocate applied to this specific decision. The practical reality is that during a controlled demo, the actors performing malpractice will likely turn their heads, not just their eyes. Iris-level precision is more valuable in production than in a hackathon demo. I was over-indexing on theoretical accuracy over demo pragmatism.

**B. Material detection approach.**
- Gemini says: MobileNet-SSD/NanoDet, triggered conditionally (every 30-60 frames or on gaze deviation).
- GPT-5.2 says: MobileNet-SSD INT8 at 0.5-1fps on lower frame, with heuristic fallback.
- Claude (me) says: Heuristics only (skin-tone + phone glow), no new model.

Gemini's Round 2 refinement — **trigger the object detector conditionally, not continuously** — changes the calculus. If you only run MobileNet-SSD when gaze deviation is already flagged, the amortized cost is negligible (maybe 2-3 inferences per minute). The memory cost (~15-25MB for a quantized INT8 model) is the real concern. Here's my updated position: **implement the heuristic approach first (P1, 4 hours), then if Phase 0 shows sufficient RAM headroom, add a conditional MobileNet-SSD as a P2 accuracy booster.** The heuristic alone is sufficient for the demo — when it fires, the evidence snapshot goes to the dashboard where a human proctor confirms. The model is insurance for production, not a hackathon requirement.

**One Genuine Remaining Disagreement:**

**Gaze threshold: 30° vs 35°.**

GPT-5.2 argues for raising the initial threshold to 35° because 30° at 640x480 with PnP is "fragile." I partially disagree. The architecture already has per-student baseline calibration (median + MAD from enrollment). A student whose natural resting gaze is 5° off-center gets a personalized threshold. With calibration, 30° *relative to their baseline* is robust. Without calibration, GPT-5.2 is right that 30° absolute is fragile. So the answer depends on implementation quality: **if calibration works reliably, keep 30° relative. If calibration proves flaky during testing, fall back to 35° absolute.** This should be a tunable parameter, not a hardcoded constant. Make it configurable in a settings file and tune during Phase 5 validation.

**What I want to sharpen from the other reviews:**

GPT-5.2 raises **clock sync** for confusion matrix correlation. This is a sleeper issue. If the ground truth logger runs on a phone and the terminals have drifted even 5 seconds, your confusion matrix will show false negatives for events that were actually detected. The edge server is already the time sync authority — the ground truth logger MUST also sync to it. A simple NTP-style offset measurement at logger startup (HTTP request to edge, compare timestamps, store offset) solves this. Budget: 1 hour. Priority: P1, because the confusion matrix is your quantitative proof and it's worthless with time skew.

Gemini raises the concern about **API thread timeouts during heavy inference** even in a single-process model. This is valid. The mitigation is straightforward: the local HTTP server (serving the browser UI and MJPEG feed) should run in its own thread with its own event loop, reading from a shared `latest_frame` variable (atomically updated by the capture thread) and a shared `latest_alerts` list. It never blocks on inference. The inference thread writes results; the API thread reads them. No locks needed if you use simple atomic reference swaps (`self.latest_frame = new_frame`). Python's GIL actually helps here — reference assignment is atomic.

CONCERNS:

1. **The plan still lacks the explicit P0/P1/P2 table as a written artifact.** All three reviewers demand it; it doesn't exist yet. This must be the FIRST deliverable before any code is written. Without it, the team will build bottom-up and run out of time with a half-working system.
2. **Clock sync between ground truth logger and edge server is unaddressed.** Even 3-5 seconds of drift will corrupt the confusion matrix scoring, undermining the most impactful demo moment.
3. **Windows DLL/protobuf conflicts between ONNX Runtime, MediaPipe, and OpenCV** remain the most likely "can't even start" failure mode. Phase 0 must validate the exact package versions together on a real government machine (or close approximation), not just on the developers' laptops.
4. **The conditional object detector (MobileNet-SSD triggered on gaze deviation) creates a coupling dependency** — if gaze detection is in SHED_GAZE degradation mode, material detection also goes dark. The heuristic fallback (skin-tone + glow) must remain active even in degraded modes as an independent signal.
5. **The 10-minute demo window is extremely tight for showing all detection types.** If the demo choreography takes longer than expected on even one scenario, later scenarios get cut. The team needs a rehearsed script with hard time limits per scenario (90 seconds each) and a "skip to next" mechanism.

SUGGESTIONS:

1. **Write the P0/P1/P2 priority table as a physical artifact TODAY**, before any code. Pin it where the whole team can see it. Include time budgets and cut triggers. This is the single highest-leverage action from this entire debate.

2. **Adopt the "cheap first, upgrade if headroom" pattern across all disputed components:**
   - Gaze: PnP from BlazeFace landmarks first → duty-cycled Face Mesh only if RAM allows and accuracy demands it.
   - Material: Skin-tone + glow heuristics first → conditional MobileNet-SSD only if RAM allows.
   - Audio: Silero VAD as planned → common-mode classification on edge.
   - This pattern reduces Phase 1-2 risk while preserving Phase 4 upgrade paths.

3. **Add clock sync to the ground truth logger** via a single HTTP timestamp exchange with the edge server at startup. Store the offset and apply it to all ground truth timestamps. 1 hour, P1 priority.

4. **Build a 90-second demo script per detection type** with a "skip" button on the ground truth logger app. If absence demo takes too long, the operator taps "skip" and the system moves to multi-face. This ensures the jury sees all capabilities within the 10-minute window.

5. **Make gaze thresholds a config file parameter**, not a hardcoded constant. Default to 30° relative (with calibration) but allow quick adjustment to 35° during validation if false positive rates are too high. Same for all duration thresholds.

6. **Ensure material detection heuristics are independent of gaze detection state** so they remain active even in SHED_GAZE degradation mode. The glow/skin-tone check adds negligible CPU cost and should run unconditionally.

7. **Package validation script for Phase 0** that imports every dependency, loads every model, opens the camera, opens the mic, measures RAM, runs 30 seconds of inference, and reports pass/fail with specific failure reasons. This script IS the Phase 0 deliverable. It should be runnable from a USB stick on a cold government machine.

KEY_TRADEOFFS:

1. **PnP gaze (fast, coarse) vs. Face Mesh gaze (slow, precise).** We've converged: start with PnP, upgrade conditionally. The tradeoff is accepting that eyes-only cheating may be missed in the hackathon demo, in exchange for reliable FPS on Celeron. This is the right tradeoff because demo actors will use obvious head movements, and the 30-second duration threshold provides a safety net against brief false positives from either approach.

2. **Heuristic material detection (cheap, noisy) vs. model-based (accurate, costly).** We've converged: heuristics as P1, model as P2 conditional upgrade. The tradeoff is accepting a higher false-flag rate on material detection in exchange for zero additional memory footprint. This is acceptable because material alerts go to a human proctor dashboard with evidence snapshots — the human closes the loop.

3. **Common-mode classification (noisier alerts) vs. suppression (cleaner feed, risk of missed collusion).** Fully converged: classification wins. The tradeoff is more alerts in the dashboard feed during noisy moments, but the alternative — silently suppressing what might be coordinated cheating — is unacceptable for an exam integrity system. The severity downgrade (`ROOM_NOISE` vs. `SUSPICIOUS_AUDIO`) preserves signal-to-noise ratio without discarding evidence.

4. **Comprehensive Phase 0 validation (costs 4-8 hours of hackathon time) vs. jumping into feature development.** All reviewers agree Phase 0 is mandatory. The tradeoff is real: those hours could build features. But a Phase 0 failure discovered on demo day is catastrophic, while a Phase 0 failure discovered early lets you pivot (e.g., drop a model, change the packaging strategy, adjust thresholds). This is the single most important tradeoff in the entire plan, and the original architecture underweights it.

5. **Rehearsed, time-boxed demo vs. freeform demonstration.** A scripted demo with skip buttons sacrifices spontaneity but guarantees coverage. Given a 10-minute window with 5-6 detection types to show plus a confusion matrix, there is no room for improvisation. The tradeoff is clearly correct: script everything, rehearse twice, have skip buttons.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position stays **PARTIALLY_AGREE**, but the center of gravity shifts further toward “simplify or you’ll miss demo-day reliability.” The other participants’ feedback doesn’t just add minor tweaks; it highlights two structural risks that can invalidate the whole system in practice:

- **Runtime co-existence risk (webcam + browser + AV + RAM):** The proposal assumes the agent can reliably capture camera/mic and keep enough RAM headroom. Claude/Gemini are right that **real Windows lab machines + Chrome + AV** turn “430MB budget” into a cliff. This pushes me to treat **Phase 0 measurement** and **camera ownership** as P0—not engineering hygiene.
- **Compute budget mismatch (Face Mesh):** Consensus is that **full Face Mesh is the biggest performance landmine** on Celeron-class CPUs. Where I diverge is the remedy: Gemini says “drop it,” Claude says “duty-cycle it.” I agree with the *principle* (reduce cost), but the best fix for hackathon robustness is: **avoid MediaPipe entirely if possible** (packaging/DLL risk) and implement **coarse head pose via Face Detection keypoints + solvePnP**, then only add a landmark model if profiling shows you still have CPU/RAM.

What did change from others’ arguments:
- I weight the **GIL/UI starvation** concern higher now. Even if ORT releases the GIL during inference, the system can still “feel frozen” if you don’t strictly bound queues and keep the UI/event thread isolated from heavy work.
- I’m more convinced that **CCTV can’t be a stub**. A cheap motion heatmap is a high-leverage, low-risk add that satisfies the “multi-modal including CCTV” requirement.

Common ground across all feedback:
1) **Material/phone detection is non-optional** for 8/10.  
2) **Audio logic must classify, not suppress** (room-wide speech may be the event).  
3) **Bounded queues + dropping frames** beats trying to process everything.  
4) **Phase 0 preflight + on-target packaging test** is a gating milestone.

CONCERNS:
1. **Webcam exclusivity conflict**: if the exam UI/browser grabs the camera, the agent may fail outright. This is a “system dead” failure, not degraded accuracy.
2. **Unauthorized material/phone detection missing**: likely included in the 10 scripted events; without it you’re betting the test case composition.
3. **Face Mesh/MediaPipe risk (performance + packaging)**: CPU-heavy on Celeron, and historically fragile to package alongside ORT/OpenCV on Windows (DLL/protobuf issues).
4. **RAM headroom is unrealistic in the real stack**: Chrome + AV + Windows services can leave too little free RAM; swap kills real-time regardless of FPS degradation logic.
5. **UI responsiveness under load (GIL + queues)**: without strict backpressure, you’ll get laggy dashboards/event push, making the demo look broken even if inference is “running.”
6. **Common-mode audio suppression creates false negatives**: room-wide dictation/collusion becomes “noise” and is discarded—exactly the kind of “wow” you should catch.
7. **CCTV “stub” under-delivers on stated requirement**: jury expects some analytic signal from CCTV, not just display.
8. **Lighting/auto-exposure edge cases**: absence/multi-face can spike false alerts when exposure hunts or lighting flickers unless you gate on brightness/quality.

SUGGESTIONS:
1. **Make the agent the camera owner (P0)**
   - Agent captures frames and serves **local MJPEG** (simplest) to the browser UI; the exam page views the feed but does not open the device.
   - This also enables overlay rendering without any GUI framework.
2. **Replace Face Mesh with a cheaper, more robust pose signal (P0/P1)**
   - Use Face Detection keypoints + `solvePnP` for **coarse yaw/pitch** at low compute.
   - If you need better gaze later, prefer a **small ONNX landmark model** over MediaPipe for packaging stability, or duty-cycle any heavier landmark step (1–2 FPS).
3. **Add material/phone detection with minimal footprint (P1)**
   - I disagree with “must add another detector model” as the default on 4GB. Start with **heuristics** that are cheap and demo-effective:
     - “Phone glow” (bright rectangular region) in lower half of frame
     - Hand/object presence proxy near desk region (skin/contour + motion persistence)
   - If Phase 0 shows you have RAM headroom, then add a tiny INT8 detector at **0.5–1 FPS** as an upgrade path.
4. **Change common-mode audio from suppression → classification (P1)**
   - ≥3 terminals speaking: `SUSPICIOUS_ENVIRONMENT` / `ROOM_NOISE` (logged, lower severity)
   - 2 nearby terminals: elevate `POSSIBLE_CONSULTATION`
5. **Hard backpressure everywhere (P0)**
   - Bounded queues of size 1 between capture → inference → event push; drop frames, never accumulate.
   - Keep UI/event push work constant-time; do evidence serialization off the hot path.
6. **Promote CCTV motion heatmap to Phase 3 (P1/P2, but low risk/high value)**
   - OpenCV background subtraction + motion score timeline + overlay = “AI on CCTV” without faces.
7. **Phase 0 must be an empirical go/no-go**
   - Measure: RAM free after launching Chrome + agent imports, sustained FPS, device open success, packaging conflicts on the exact deployment method (WinPython/PyInstaller), and network/clock offsets.

KEY_TRADEOFFS:
- **Accuracy vs. operational robustness:** Dropping heavy gaze (Face Mesh) may reduce sensitivity to subtle eye-only cheating, but dramatically improves the chance the system runs on Celeron without freezing. For hackathon scoring, a reliable coarse pose + long-duration rules is the right trade.
- **Heuristics vs. adding another model (material detection):** A detector model can improve recall/precision but costs RAM, packaging complexity, and CPU. Heuristics are noisier but cheap and often sufficient to *flag* events with human-verifiable evidence in the dashboard.
- **Threads vs. multiprocessing:** Consensus shifts away from multiprocessing on 4GB Windows due to memory duplication. The right trade is **single-process + bounded queues + frame dropping** unless Phase 0 proves there is ample headroom.
- **Suppressing noise vs. detecting collusion (audio):** Classification increases alert volume but prevents catastrophic false negatives. Given the domain, logging and downgrading is better than discarding.

Common ground summary: keep the architecture, but (1) **agent owns camera**, (2) **material detection added**, (3) **MediaPipe/Face Mesh de-risked or removed**, (4) **audio reclassified not suppressed**, (5) **Phase 0 becomes a hard gate**. These changes improve your probability of hitting **8/10 on real hardware** more than any additional fusion rule would.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
I am aligning with the strong convergence between Claude Opus, GPT-5.2, and myself on the critical risks, while adjusting my technical recommendations based on their valid counter-arguments regarding memory constraints.

1.  **Webcam Exclusivity (Critical Consensus):** I missed this, but Claude and GPT-5.2 are absolutely correct. If the browser exam platform tries to access the webcam while our Python agent is reading it, the system will fail. The architecture **must** be inverted: The Python Agent effectively becomes the "Virtual Camera" driver, serving frames via a local MJPEG stream to the browser.
2.  **Performance Strategy Convergence:** We have reached a consensus that `multiprocessing` is too RAM-expensive for a 4GB environment. The winning strategy is **Single Process + 3 Threads + Bounded Queue (Size 1)**. This drops frames rather than lagging, keeping the UI responsive without the memory overhead of a second Python interpreter.
3.  **Material Detection Approach:** While we all agree "Unauthorized Material" is a critical gap, I am shifting my support to Claude's **Heuristic Approach (Skin/Glow)** as the P0 implementation. Adding a MobileNet-SSD (my previous suggestion) risks pushing RAM usage over the edge when Chrome is running. We should only load a detector model if Phase 0 profiling confirms available memory.

CONCERNS:
1.  **Browser + Agent RAM Contention:** We are underestimating the memory footprint of a modern browser (Chrome/Edge) running a heavy exam UI. It can easily consume 1.5GB+, leaving <2GB for the OS and our Agent. If the system hits the page file, 15FPS drops to 0.5FPS instantly.
2.  **Heuristic Noise for Materials:** While necessary for memory reasons, skin-tone/glow heuristics generate significant false positives (e.g., touching face, reflection on glasses). This requires the "Confusion Matrix" to strictly differentiate between "High Confidence" (AI) and "Low Confidence" (Heuristic) alerts to avoid discrediting the system.
3.  **MJPEG Latency:** Routing video through Python -> HTTP -> Browser adds 50-100ms latency. This might feel "sluggish" to the user compared to a direct feed.

SUGGESTIONS:
1.  **Architectural Inversion (Agent-First Video):** Hard requirement. The Python script captures `cv2.VideoCapture(0)`, processes the frame, draws overlays (optional), and serves it at `http://localhost:8000/video_feed`. The Browser UI uses `<img src="...">`. This solves the exclusivity conflict and allows us to visualize detections on the user's screen (Innovation).
2.  **"Duty-Cycled" Gaze Strategy:** I support the hybrid model proposed by the group: Run cheap **PnP (Head Pose)** every frame. Run heavy **Face Mesh (Gaze)** only once every 10-15 frames (or 2 FPS) to calibrate the vector. This balances CPU load with precision.
3.  **Revised Audio Logic:** Rename "Common-Mode Rejection" to **"Environmental Classification"**.
    - If >3 terminals trigger: Flag `ENVIRONMENT_NOISE` (Low Severity).
    - If 2 adjacent terminals trigger: Flag `SUSPICIOUS_COLLUSION` (High Severity).
4.  **Phase 0 "Kill Switch":** Create a startup script that measures available RAM *after* launching Chrome. If free RAM < 400MB, the system must automatically disable the "Duty-Cycled Face Mesh" and fall back to PnP-only mode.

KEY_TRADEOFFS:
**Memory vs. Model Capability:** We are sacrificing the robust detection of an Object Detection Model (MobileNet-SSD) for the memory safety of Heuristics (Skin/Glow). This is the correct operational tradeoff for a 4GB constraint, even if it lowers theoretical accuracy.
**Latency vs. Reliability:** We are accepting the latency of an MJPEG loopback to guarantee Hardware Access. A slightly laggy video feed is acceptable; a "Camera Not Found" error is a demo-killer.

---
## Final Positions

| Model | Position |
|-------|----------|
| Claude Opus | PARTIALLY_AGREE |
| GPT-5.2 | PARTIALLY_AGREE |
| Gemini 3 Pro | PARTIALLY_AGREE |

---
## Synthesis

CONSENSUS_LEVEL: NEAR

COMMON_GROUND:
1. Unauthorized material/phone detection is a critical gap that must be addressed before coding begins; without it, the 8/10 detection target is at serious risk if any of the 10 test events involve phones, notes, or other physical materials.
2. The Python agent must own the webcam exclusively and serve frames to the browser exam UI via a local MJPEG endpoint (`/video_feed`). This is a P0 architectural requirement, not optional.
3. Common-mode audio rejection must be changed to common-mode audio classification — simultaneous VAD triggers across ≥3 terminals should be downgraded to `ROOM_NOISE`/`SUSPICIOUS_ENVIRONMENT` (low severity, logged), not suppressed. Two adjacent terminals triggering should escalate to `POSSIBLE_CONSULTATION`.
4. Threading (not multiprocessing) is the correct concurrency model on 4GB Windows machines. Multiprocessing duplicates ~150-200MB of Python runtime and libraries, which is unaffordable. The design should use bounded queues of size 1 with frame dropping to manage backpressure.
5. Phase 0 preflight is a first-class, gating deliverable — a single script that measures actual available RAM (after Chrome + AV + Windows services), tests webcam/mic access, validates ONNX Runtime + MediaPipe/OpenCV package coexistence, runs sustained inference for 30 seconds, checks network/clock sync, and outputs PASS/FAIL with specific reasons. No feature coding begins until Phase 0 passes.
6. The ~430MB memory budget is dangerously thin under real conditions (Windows + AV + Chrome exam UI). Phase 0 must empirically measure available RAM on representative hardware; architectural decisions (which models to load) must be conditional on that measurement.
7. Full MediaPipe Face Mesh running every frame is too expensive on Celeron-class CPUs and is the single biggest performance risk. It must be either replaced or heavily duty-cycled.
8. The CCTV component cannot remain a display-only stub. Minimal AI (background subtraction via `BackgroundSubtractorMOG2` + motion heatmap/commotion index) must be added on the edge server to satisfy the "integrate CCTV with AI" requirement. Budget: 3-4 hours, promoted to Phase 3.
9. A lighting/brightness check must be added: track per-frame mean brightness and emit `POOR_LIGHTING` instead of `CANDIDATE_ABSENT` when brightness drops below threshold for >2 seconds. This prevents false absence alerts from auto-exposure hunting and flickering lights. Budget: 30 minutes.
10. The architecture needs an explicit, written P0/P1/P2 feature priority table with hard cut rules, created before any code is written. No P2 work begins until all P0 items pass a live 3-terminal test.
11. The scope is too ambitious for a hackathon; the primary risk is spreading effort thin. Four rock-solid detectors beat six flaky ones.
12. Gaze thresholds, duration thresholds, and face verification thresholds should be configurable parameters (settings file), not hardcoded constants, to allow rapid tuning during validation.
13. Clock synchronization between the ground truth logger and the edge server is necessary for the confusion matrix to be meaningful. The ground truth logger must sync its time offset against the edge server at startup.
14. A "golden demo" fallback or replay mode should be prepared as insurance against hardware failures on demo day.
15. The demo must be scripted and rehearsed with hard time limits per scenario (~90 seconds each) and a "skip to next" mechanism to guarantee all detection types are shown within the demo window.

CONVERGENCE:
1. **Multiprocessing vs. threading**: Gemini initially advocated moving inference to a separate process to bypass the GIL. Claude and GPT-5.2 argued the ~200MB memory overhead was fatal on 4GB machines. By Round 2, Gemini explicitly conceded and all converged on single-process + 3 threads + bounded queues with frame dropping.
2. **Material detection approach**: Gemini initially suggested MobileNet-SSD/NanoDet as a model-based detector. GPT-5.2 initially leaned toward MobileNet-SSD at low FPS. Claude argued for heuristics only (skin-tone + phone glow) to avoid memory cost. By Round 3, all converged on: heuristics first (P1), with a conditional model-based upgrade (P2) only if Phase 0 shows sufficient RAM headroom.
3. **Face Mesh handling**: Claude initially suggested iris tracking via Face Mesh subset; GPT-5.2 suggested a 68-landmark ONNX model + solvePnP; Gemini suggested dropping Face Mesh entirely for PnP from BlazeFace landmarks. By Round 3, all converged on: start with cheap PnP head pose from existing face detection keypoints as default, with optional duty-cycled Face Mesh (1-2 FPS) as a conditional upgrade if RAM/CPU allows. Gemini specifically endorsed the "duty-cycled" hybrid in Round 3.
4. **CCTV treatment**: The original plan had it as a display-only stub. All three reviewers independently flagged this as insufficient, and by Round 2-3 all agreed on background subtraction + motion heatmap on the edge server, promoted to Phase 3.
5. **Common-mode audio logic**: GPT-5.2 first identified the flaw (suppression can hide coordinated collusion). Claude and Gemini both adopted the reclassification approach in subsequent rounds. Full convergence by Round 2.
6. **Webcam ownership**: Claude raised this in Round 1. GPT-5.2 endorsed it in Round 2. Gemini acknowledged missing it and fully adopted it in Round 3. All converged on agent-as-camera-owner with MJPEG serving.

REMAINING_DISAGREEMENTS:
1. **Gaze angle threshold (30° vs. 35°)**: GPT-5.2 argues the initial threshold should be raised to 35° because 30° at 640x480 with PnP-only is fragile. Claude argues 30° relative to the per-student calibrated baseline is robust (since baseline calibration absorbs individual variance), but concedes 35° absolute is appropriate if calibration proves unreliable. Gemini did not take a strong position on the specific number. This is a narrow disagreement that all agree should be resolved empirically — make it a configurable parameter and tune during validation.
2. **Whether to avoid MediaPipe entirely vs. keep it as an optional upgrade**: GPT-5.2 in Round 3 explicitly suggests avoiding MediaPipe altogether if possible due to packaging/DLL risks, preferring a small standalone ONNX landmark model if more than PnP keypoints are needed. Claude and Gemini are willing to keep duty-cycled Face Mesh as an upgrade path. The disagreement is about packaging risk tolerance — GPT-5.2 weights DLL/protobuf conflict risk higher than the others.
3. **Whether the conditional object detector (MobileNet-SSD) should be triggered by gaze deviation or run on a fixed low-FPS schedule**: Gemini suggested triggering on gaze deviation (every 30-60 frames or when gaze flags). Claude noted this creates a coupling dependency — if gaze is in SHED_GAZE mode, material detection also goes dark. GPT-5.2 leaned toward fixed low FPS (0.5-1 FPS) if used at all. This is moot unless Phase 0 shows sufficient RAM for the model, and all agree heuristics are the primary path regardless.

RECOMMENDED_PLAN:

**Pre-Coding (Day 0 / Before Sprint):**
- [ ] Write and publish the P0/P1/P2 feature priority table as a physical/visible artifact:
  - **P0 (demo fails without):** Agent camera ownership + MJPEG feed, CANDIDATE_ABSENT (with lighting check), MULTIPLE_PERSONS, FACE_MISMATCH, enrollment flow, edge dashboard with alert feed, preflight diagnostic script
  - **P1 (8/10 at risk without):** GAZE_DEVIATION (PnP-based, configurable 30°-35° threshold relative to calibrated baseline), SUSPICIOUS_AUDIO (Silero VAD), cross-terminal SEAT_SWAP, unauthorized material heuristics (skin-tone + phone glow), common-mode audio classification, clock sync for ground truth logger
  - **P2 (innovation/wow, only if P0+P1 pass 3-terminal test):** Live confusion matrix, CCTV motion heatmap, duty-cycled Face Mesh gaze upgrade, conditional MobileNet-SSD material detector, hash-chained audit trail, store-and-forward, timeline visualization, enrollment liveness (blink detection)
- [ ] Hard rule: No P2 work until all P0+P1 items pass a live 3-terminal integration test

**Phase 0 — Preflight (Gating Milestone):**
- [ ] Build a single-script diagnostic that tests on target hardware (or closest approximation):
  - Webcam open/close, microphone access
  - ONNX Runtime import + model load
  - OpenCV import + basic frame processing
  - If using MediaPipe at all: import + load test (watch for DLL/protobuf conflicts)
  - Available RAM measurement after launching Chrome with representative exam UI page + Windows AV active
  - Sustained 30-second inference loop reporting FPS
  - Network connectivity to edge server + clock offset measurement
  - Disk write permission test
- [ ] **Go/No-Go decisions based on Phase 0 results:**
  - If available RAM < 350MB after Chrome + AV: drop any plan for MobileNet-SSD; use heuristics only for material detection; use PnP-only for gaze (no Face Mesh even duty-cycled)
  - If MediaPipe causes DLL/protobuf conflicts with ONNX Runtime: eliminate MediaPipe entirely; use ONNX-based face detection + landmark model or BlazeFace keypoints + solvePnP only
  - If sustained FPS < 5 on target hardware: reconsider frame resolution (reduce to 480x360) or eliminate audio processing from terminal

**Phase 1 — Core Pipeline:**
- [ ] Agent owns the camera via `cv2.VideoCapture(0)` and serves MJPEG at `http://localhost:{PORT}/video_feed`; browser exam UI consumes this feed via `<img>` tag
- [ ] 3-thread architecture: capture thread, inference thread, event/API thread
  - Bounded queue of size 1 between capture → inference (drop stale frames)
  - API/event thread reads shared state atomically (latest_frame, latest_alerts), never blocks on inference
- [ ] Implement gaze via PnP head pose from face detection keypoints (5-6 points + `solvePnP`) — no Face Mesh in Phase 1
- [ ] Gaze threshold as a configurable parameter (default: 30° relative to calibrated baseline; fallback: 35° absolute)
- [ ] Per-frame brightness tracking: if mean brightness < threshold for >2s, emit `POOR_LIGHTING` not `CANDIDATE_ABSENT`
- [ ] Face verification with 3-zone hysteresis (thresholds as configurable parameters, explicitly validated as cosine similarity not distance)
- [ ] FPS degradation modes as designed (FULL → SHED_GAZE → SHED_AUDIO → SURVIVAL)

**Phase 2 — Detection + Vertical Slice:**
- [ ] All P0 alert types working: CANDIDATE_ABSENT, MULTIPLE_PERSONS, FACE_MISMATCH
- [ ] P1 alert types: GAZE_DEVIATION (PnP-based), SUSPICIOUS_AUDIO (Silero VAD)
- [ ] Cross-terminal SEAT_SWAP on edge server
- [ ] Unauthorized material heuristics (independent of gaze state, runs even in SHED_GAZE mode):
  - Bright rectangular region detection in lower frame half (phone screen glow)
  - Skin-tone blob detection in lower-face / desk ROI (hand holding object)
  - Triggers POSSIBLE_MATERIAL_USE with evidence snapshot to dashboard
- [ ] Common-mode audio classification on edge: ≥3 terminals → `ROOM_NOISE` (low severity, logged); 2 adjacent terminals → `POSSIBLE_CONSULTATION` (elevated)
- [ ] Ground truth logger time-synced to edge server via HTTP timestamp offset at startup
- [ ] **Milestone gate:** 3-terminal live test with all P0+P1 detections working reliably

**Phase 3 — Dashboard + CCTV + Exam UI:**
- [ ] Edge dashboard: 30-tile grid, alert feed, evidence viewer
- [ ] CCTV tile with OpenCV `BackgroundSubtractorMOG2` + motion heatmap overlay on edge server (satisfies "AI on CCTV" — no face processing)
- [ ] Ground truth logger app (mobile-friendly) with "skip to next scenario" button
- [ ] Demo choreography script with 90-second hard limits per detection type

**Phase 4 — Conditional Upgrades (only if time + RAM permit):**
- [ ] If Phase 0 showed RAM headroom: add duty-cycled Face Mesh at 1-2 FPS for iris-level gaze (prefer small ONNX landmark model over MediaPipe if packaging was problematic)
- [ ] If Phase 0 showed RAM headroom: add conditional MobileNet-SSD INT8 for phone/paper detection at 0.5-1 FPS (triggered independently, not gated by gaze state)
- [ ] Live confusion matrix overlay on dashboard
- [ ] Hash-chained audit trail with periodic anchoring to edge
- [ ] Timeline visualization per terminal (green/yellow/red segments with evidence thumbnails)
- [ ] Enrollment-time liveness check (blink detection via eye aspect ratio from existing landmarks)
- [ ] Store-and-forward for LAN failures

**Phase 5 — Validation + Demo Prep:**
- [ ] Tune all configurable thresholds (gaze angle, face verification zones, duration windows) on actual hardware using a 30-60 minute capture session
- [ ] Rehearse full demo choreography at least twice with timing
- [ ] Prepare "golden demo" fallback: pre-recorded 3-terminal video that can replay through the system if hardware fails
- [ ] Package validation: test final deployment artifact (WinPython USB or PyInstaller) on a clean machine
- [ ] Verify confusion matrix accuracy by checking clock alignment between ground truth logger and edge