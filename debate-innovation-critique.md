# Multi-LLM Debate Report

**Models:** Claude Opus, GPT-5.2, Gemini 3 Pro
**Rounds completed:** 3

---
## Round 1

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

The architecture is technically sound and well-scoped for the constraints. The debate rounds have produced a clean, deployable system. But the question isn't "is this good engineering?" — it's "is this enough to win?" And the honest answer is: **this is a strong top-3 system, but it has a clear vulnerability that could cost you the win.**

Let me break this down across the dimensions asked:

**1. Innovation Assessment: Competent but Not Distinctive**

The core pipeline (BlazeFace + MobileFaceNet + MediaPipe + Silero) is essentially the "standard open-source proctoring stack" that any team with decent AI engineers would assemble. The episode-based event system and 3-zone hysteresis are genuinely clever engineering, but they're invisible to a non-technical jury unless you dramatize them. The edge-first privacy story is your strongest differentiator — but only if you make it visceral in the demo (more on this below).

**2. The Biggest Accuracy Gap: No Object Detection**

You rejected YOLO for valid reasons (COCO mismatch, AGPL, CPU cost). But you left a gaping hole: **phone/cheatsheet detection is probably 2-3 of the 10 staged malpractice scenarios.** The judges will stage someone pulling out a phone, someone reading from a hidden note, someone showing a paper to another candidate. You will miss all of these. This is potentially a 20-30% hit on your 8/10 target.

The YOLO rejection rationale is partially flawed. You don't need COCO's 80 classes — you need a tiny custom detector or even a simpler approach:

- **NanoDet-Plus** (Apache 2.0, ~1MB INT8) trained on just 3 classes (phone, paper, hand-near-face) would cost ~8ms per frame on i5 and dodge the AGPL issue entirely
- Alternatively, a **hand-region heuristic**: MediaPipe Hands (you already have the MediaPipe dependency) detecting hand positions in the upper body zone that correlate with holding objects. This isn't object detection — it's posture anomaly detection. A hand hovering at ear level for 5+ seconds? That's a phone call. A hand held flat at desk level with gaze directed down? That's reading a cheatsheet.
- Even cruder: **frame differencing in the desk region** as a motion anomaly signal. Something appeared on the desk that wasn't there during enrollment.

Not having any answer to physical object cheating is the single biggest risk to losing.

**3. Temporal Analysis: The Cheap Win You're Missing**

You asked about lightweight temporal patterns without RNNs. Yes, absolutely — and this is a genuine differentiator:

Implement a **sliding window state machine** (30-60 second windows) that tracks:
- Gaze deviation frequency (intermittent glancing vs. sustained looking away)
- Head pose variance (nervous repeated checking vs. calm engagement)
- Face verification confidence trend (gradual drift = fatigue/lighting vs. sudden drop = swap)

This is just numpy operations on your existing signal buffers. No new models. Maybe 50 lines of code. The output is a **behavioral anomaly score** that compounds over time. A single glance away is nothing. Twelve glances in 60 seconds toward the same azimuth is highly suspicious. This is what separates "detect-and-alert" (what everyone builds) from "behavioral intelligence" (what wins hackathons).

Store these as time-series in your JSONL and render them as sparklines on the dashboard. The jury sees a rising risk curve, not just a binary alert. That's visually compelling and narratively powerful.

**4. Exam Interaction Analytics: The Zero-Cost Signal**

Since you control the exam interface (PyQt6), you have access to signals that require zero additional CPU:
- **Answer timing**: completing a hard question in 3 seconds = copied answer
- **Question navigation patterns**: sequential = normal, random jumping = looking for specific answers (being fed information)
- **Keystroke dynamics**: typing speed variance, burst patterns
- **Copy-paste events**: if clipboard is monitored

This is 1 day of work maximum. It adds an entire modality that doesn't touch the camera pipeline at all. And it's deeply defensible in the "scalability to 10K centers" story because it works on Celeron machines with no camera at all.

**5. Performance Left on the Table**

A few concrete optimizations:

- **ONNX Runtime's IntraOpNumThreads**: Ensure you're setting this to 2-3, not defaulting to all cores. With 3 threads competing for CPU, thread contention on a 4-thread i5 is real.
- **Frame skipping with motion gating**: If the scene hasn't changed (SSIM > 0.98 between frames), skip the full pipeline entirely. In a real exam, candidates sit still for long periods. This could reduce average CPU load by 40-50%.
- **Batch face verification**: When MULTIPLE_PERSONS fires, verify all detected faces in one MobileFaceNet pass rather than sequential calls.
- **Quantized MediaPipe**: The default MediaPipe Face Mesh ships FP32. A custom INT8 TFLite conversion cuts inference time ~40%.

**6. What the Winning Team's Architecture Looks Like**

Everything you have, plus:
- One lightweight object/posture detector (NanoDet or MediaPipe Hands heuristic)
- Behavioral anomaly scoring with temporal windowing
- Exam interaction analytics as a zero-cost signal layer
- A "Risk Timeline" visualization on the dashboard showing the narrative arc of each incident
- A live demo script that's rehearsed to the second

The winning team won't necessarily have better models. They'll have **better storytelling through their dashboard** and **coverage of all 10 scenario types**.

**7. Demo Strategy (This Is Where Hackathons Are Won)**

For a non-technical jury, the demo IS the product. Specific recommendations:

- **Open with the failure case**: Show a 5-second clip of what happens with basic webcam monitoring (nothing detected). Then show ExamGuard catching the same scenario. The contrast sells.
- **Rehearse the 10 malpractice scenarios in exact sequence**, escalating in sophistication: (1) candidate walks away, (2) someone else sits down, (3) impersonation attempt, (4) phone at ear, (5) cheatsheet on desk, (6) two people whispering, (7) someone looking at neighbor's screen, (8) coordinated gaze pattern, (9) answer-feeding via audio, (10) candidate swaps back. Each one should trigger a visible alert on the dashboard within 3-5 seconds.
- **The dashboard must tell a story in real-time**: Color-coded terminal tiles going from green → yellow → red, a scrolling event feed with thumbnails, and the confusion matrix filling in live as events are detected and matched.
- **Name your features**: "3-Zone Hysteresis" means nothing to a bureaucrat. "Graduated Confidence System" or "Triple-Check Verification" sounds like due process. "Episode Intelligence" not "episode-based events." "Privacy-First Architecture" not "edge computing."
- **The DPDP compliance slide should be one sentence**: "No video ever leaves the exam room. No biometric data is stored after the exam. Full compliance with DPDP Act 2023." That's it. Don't explain the architecture. State the outcome.
- **End with the scalability number**: "This system runs on ₹30,000 hardware. Deploying to all 10,000 centers in Andhra Pradesh costs ₹30 crore — less than ₹1 per student per exam." That's the number that makes a government official lean forward.

CONCERNS:

1. **Object detection gap is critical**: 2-3 of the 10 staged scenarios will likely involve physical objects (phone, cheatsheet, textbook). You currently detect zero of these. This alone could drop you from 8/10 to 5-6/10.
2. **Audio as fusion-only is risky**: If a scenario involves two candidates openly talking (without gaze deviation — e.g., both facing forward, whispering), you'll miss it entirely. Consider making sustained audio (>15s) a standalone low-confidence alert.
3. **Gaze as fusion-only may be too conservative**: Sustained gaze deviation toward a specific direction (>20s continuous) is inherently suspicious. Keeping it strictly fusion-only means you can't detect "reading from a phone held below desk level" even when gaze clearly points down-left for 30 seconds straight.
4. **The confusion matrix matching with ±15s window may be too tight**: If your episode-based system takes 5 seconds to confirm + network latency + judge's ground truth timestamp is approximate, you could miss valid detections on a technicality. Consider ±20-25s or provide the jury a manual override to accept near-matches.
5. **Celeron degradation is underspecified**: You mention FPS-based degradation modes but don't specify what gets cut. If BlazeFace alone takes 30ms on a Celeron, you're at ~2-3fps for the full pipeline. At that rate, a fast impersonation swap (5 seconds) might only get 10-15 frames of detection opportunity. The track-between-detections strategy helps, but verify your actual Celeron numbers early.
6. **Enrollment robustness**: 5 frames from "different angles" is vague. If the enrollment lighting differs significantly from exam lighting (e.g., enrollment in well-lit office, exam in fluorescent-lit hall), MobileFaceNet distances will shift. Consider a quick re-calibration step at exam start (capture 2 verification frames and adjust thresholds).
7. **No liveness detection at enrollment**: Someone could enroll with a photo held up to the camera. Face Mesh depth estimation (the z-coordinates) can flag flat faces with ~10 lines of code. Trivial to add.

SUGGESTIONS:

1. **Add MediaPipe Hands as a posture anomaly detector** (1 day): Detect hand-at-ear (phone call), hand-at-desk-with-gaze-down (reading), two hands raised with object (showing cheatsheet). This is your cheapest path to covering the object detection gap. MediaPipe Hands runs at ~12ms on i5 INT8 and you already have the dependency.
2. **Implement a 60-second sliding window behavioral score** (0.5 day): Aggregate gaze deviation frequency, head pose variance, and verification confidence jitter into a single 0-100 risk score per terminal. Display as a sparkline on each dashboard tile. This is your "AI intelligence" differentiator — every other team will show binary alerts.
3. **Add exam interaction signals** (1 day): Answer timing anomalies, navigation patterns, keystroke cadence. These are pure Python signals from your PyQt6 exam interface. Zero CPU cost. Fuse with camera signals for higher confidence.
4. **Promote sustained audio (>15s) to standalone low-confidence alert** (0.5 day): Don't miss the "two candidates talking" scenario just because gaze happens to not trigger.
5. **Add Face Mesh liveness check at enrollment** (2 hours): Use the z-coordinate flatness of face mesh landmarks to reject photo-based spoofing. Minimal code, impressive to judges.
6. **Motion-gated frame skipping** (0.5 day): Compute frame difference magnitude. If < threshold, skip full pipeline, just run tracker. This buys you CPU headroom for the Hands model addition.
7. **Build a "Risk Timeline" visualization** (1 day): For the evidence viewer, show a horizontal timeline per terminal with colored segments (green/yellow/red) and event markers. When a judge clicks a terminal, they see the narrative arc of that candidate's exam. This is visually stunning and no other team will have it.
8. **Rehearse the demo 5+ times with actual staged scenarios** (1 day): Script every second. Know which scenarios trigger which alerts. Have fallback explanations for any misses. The demo must feel effortless.

KEY_TRADEOFFS:

1. **Precision-over-recall vs. 8/10 detection target**: Your conservative thresholds minimize false positives (good for jury trust) but risk missing detections. With only 10 scenarios, each miss is 10% of your score. The fusion-only restrictions on gaze and audio mean you're relying on face-based detections for ~60-70% of scenarios. If the judges stage non-face scenarios (phone, notes, talking), you're exposed. **This tradeoff leans too far toward precision given the scoring rubric.**

2. **No object detection vs. CPU budget**: You saved ~15ms/frame by cutting YOLO, which is legitimate. But MediaPipe Hands at ~12ms gives you 80% of the coverage for object-related scenarios. The tradeoff arithmetic changed when you found the Detect→Track→Verify pattern saves 35% CPU. **You now have budget headroom you're not using.**

3. **Edge-first vs. feature richness**: The privacy-first architecture is a genuine differentiator and the right call for the government audience. But it means no cloud-based model serving, no larger models, no cross-center analytics. **Right tradeoff for this context.**

4. **Episode-based vs. frame-level events**: Clearly correct for reducing noise and improving precision. But the ±15s matching window means your episode boundaries must be tight. **Right tradeoff, but test the matching logic exhaustively.**

5. **7-10 day sprint vs. adding features**: Every suggestion I made above totals ~4-5 days. You have time if you parallelize. The risk is integration bugs. **Allocate day 7-8 purely for integration testing and demo rehearsal. Never sacrifice rehearsal time for features.**

6. **Simplicity vs. winning**: Your current architecture is simple and deployable. Adding Hands + temporal scoring + exam analytics + liveness makes it more complex but dramatically more competitive. The question is whether "clean and reliable but catches 6/10" beats "slightly more complex but catches 8-9/10." **In a hackathon with accuracy scoring, coverage wins. Add the features.**

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
Your architecture is solid for the constraints (CPU-only, 4GB RAM, unreliable 4G, 7–10 days). The “single-process terminal agent + edge event server + episode-based alerts + evidence only on high severity” is operationally sane and will outperform many teams that try cloud streaming or overbuild with browsers/electron.

But from an **innovation / competitive advantage** lens, it’s currently more “competent and safe” than “obviously ahead.” Most teams can assemble *some* combination of MediaPipe + face verify + a dashboard. Your differentiators right now are mostly **deployment pragmatism** (edge-first, privacy, low bandwidth, episode events), not **detection capability**. In a scoring scheme dominated by “detect 8/10 staged events with good timestamps and low FPs,” the biggest risk is that your detection set is narrow (no screen/OS integrity signals, no liveness/anti-spoof, no phone/book detection), so you may miss exactly the scenarios the jury will stage.

Also, a key technical point: **MediaPipe Face Mesh on CPU for gaze is expensive and fragile relative to the value it provides**, and your own plan demotes gaze to fusion-only (so you pay compute for a weak, non-decisive signal). That’s a classic “cost without leverage” choice. If you want to win, spend compute (and dev time) on signals that (a) fire as standalone alerts, (b) are explainable to non-technical judges, and (c) map to the likely staged malpractices.

CONCERNS:
1. **Likely staged scenarios not covered**: phone use, reading notes/book, second device, off-screen consultation, candidate opening other windows/apps, copy/paste, screen sharing/remote control—none are directly detected today.
2. **Face Mesh ROI/value mismatch**: high CPU + sensitivity to lighting/occlusion; yet gaze is fusion-only, so it may not materially improve the confusion-matrix score.
3. **BlazeFace robustness**: can underperform under low light, side profiles, masks, backlit rooms—exactly what exam centers look like. If detection drops, everything downstream collapses.
4. **No liveness / anti-spoof**: photo-on-phone, printed photo, replay attack on webcam feed (or even just showing a phone with a face) can pass “face present” and may confuse verification.
5. **Audio fusion-only + common-mode suppression failure modes**: common-mode reject can suppress real cheating if multiple rooms share ambient speech; also misses “one helper whispering” cases. If judges stage “candidate speaking,” you want a standalone “SPEECH” event.
6. **Threshold brittleness across centers**: face distance thresholds and “>5 frames” rules can swing wildly with camera quality and lighting; without per-device calibration you’ll see either FPs or misses.
7. **Tracker drift edge cases**: MOSSE/CSRT can drift on occlusions/fast motion; a drift can become a false mismatch or miss multiple-person entry.
8. **Evidence integrity / DPDP operational story**: “push JPEG on alerts” is good, but you need a clear retention/purge policy + encryption-at-rest on edge, otherwise DPDP questions can derail you.
9. **Time sync and scoring window**: your app-level time offset is fine, but timestamp accuracy becomes your Achilles heel if you don’t visibly demonstrate it (judges will trust what they can see).

SUGGESTIONS:
1. **Add “Screen / OS Integrity Telemetry” (1–2 days, huge differentiator, very judge-friendly)**  
   This is the fastest way to add high-accuracy signals without heavy CV:
   - Window focus loss / Alt-Tab / task switch events  
   - Detect multiple displays / projector connected  
   - Detect screen recording/remote tools/process blacklist (AnyDesk/TeamViewer/Chrome/WhatsApp)  
   - Copy/paste / PrintScreen attempts (where OS allows hooks)  
   - “Exam left fullscreen” / “App minimized”  
   These become **clean standalone alerts** with near-90–99% reliability and extremely low false positives. Most teams won’t do this well, and it maps directly to “cheating” in a way officials understand.

2. **Replace Face Mesh gaze with cheaper, more robust “Head Pose + Eye Blink” using sparse landmarks**  
   Instead of full 468-landmark mesh:
   - Use a face detector that outputs 5 keypoints (eyes, nose, mouth corners) and compute **yaw/pitch via solvePnP** (or a learned tiny headpose regressor).  
   - Add **blink/EAR** from eye keypoints if available (or a tiny landmark model).  
   This cuts CPU and increases stability. Then you can afford higher face-detect cadence or add a liveness check.

3. **Add “Active Liveness Challenge” triggered only on suspicion (1 day, very high jury impact)**  
   Don’t do continuous anti-spoof. Do *challenge-response* only when:
   - face mismatch enters “uncertain” zone, or
   - multiple persons detected, or
   - face disappears/returns.  
   Challenges: “blink twice”, “turn head left/right”, “look up”. Verify via head pose/bink signals. This is cheap, explainable, and makes you look “serious” versus generic webcam monitoring.

4. **Upgrade face detection to a more robust CPU-friendly option (low dev risk, big stability gain)**  
   Consider replacing BlazeFace with one of:
   - **YuNet** (OpenCV model, small, fast on CPU, good in-the-wild)  
   - **SCRFD (small variants)** if you can keep it light and INT8  
   The goal is not SOTA—it's *robustness under bad exam lighting*.

5. **Make audio a standalone alert with conservative thresholds (to win staged scenarios)**  
   Keep “consultation” as fusion, but add:
   - **SUSTAINED_SPEECH** standalone (e.g., >6–10s speech while face present)  
   This will catch simple staged “candidate talking” events and is easy to explain, while still avoiding FPs with long thresholds.

6. **Do “temporal logic” without RNNs: change-point / CUSUM / EWMA risk scoring (1 day)**  
   Keep discrete events, but also compute a **rolling risk score** (0–100) using:
   - EWMA of head-pose deviation  
   - count of face mismatch checks entering uncertain zone  
   - face absent durations  
   - speech durations  
   Then only emit alerts when the score crosses thresholds. This improves timestamps and reduces spam without deep learning.

7. **Performance: use compute only when it matters (you’re leaving efficiency on the table)**  
   - Run heavy steps (verification, headpose) only on: new detection, tracker confidence drop, or periodic keyframes.  
   - Dynamic FPS: idle mode 3–5 fps; suspicion mode 10–15 fps for 10–15 seconds.  
   - If Intel CPUs: seriously consider **OpenVINO backend** for ONNX models; it can be a material speedup on i5-class CPUs.

8. **Demo-proofing & DPDP** (low effort, high “government readiness” score)  
   - Edge SSD encryption-at-rest (even a simple documented approach)  
   - Evidence retention slider (e.g., auto-delete after 7 days)  
   - “No raw video leaves terminal” shown explicitly on dashboard  
   - Offline queue: if edge disconnects, terminal buffers events and syncs later

KEY_TRADEOFFS:
- **You traded coverage for precision and reliability.** That’s the right instinct (FPs destroy trust), but it becomes the wrong trade if the staged 10 scenarios include “screen switching / phone / notes,” because you’ll miss events entirely (false negatives are fatal to 8/10).
- **You traded “flashy ML” for deployability.** This is good for government adoption, but hackathon judging often rewards one or two “wow” features (like liveness challenge + OS integrity) more than a clean but generic CV stack.
- **You traded continuous multimodal fusion for hard rules.** Hard rules are debuggable and shippable in 10 days, but you need at least one “lightweight temporal scoring” layer to avoid brittle thresholds and to improve timestamp precision.

If you implement only two additions to shift from “competent” to “top-3 likely”:  
1) **OS/screen integrity telemetry as standalone alerts**, and  
2) **Active liveness challenge on suspicion** (with headpose replacing full Face Mesh).  

Those are high-impact, low-effort, highly demoable, and meaningfully harder for other teams to replicate quickly.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
Your current architecture is **architecturally superior** for deployment but **competitively dangerous** for a hackathon.

**What works:**
1.  **The "Edge-First" Stability:** While other teams struggle with WebRTC latency and cloud costs on 4G, your PyQt6 + ONNX stack will actually run. This is your baseline "survival" advantage.
2.  **Precision Strategy:** Avoiding "alert spam" via episodes and hysteresis is the correct engineering decision. It prevents the "boy who cried wolf" scenario that fails most demos.
3.  **Resource Budgeting:** The FPS-based degradation and threaded pipeline proves you understand the hardware constraints better than teams who will just `pip install ultralytics` and crash the CPU.

**What doesn't work (The "Innovation Gap"):**
1.  **Generic AI Stack:** MediaPipe + MobileFaceNet is the "Hello World" of proctoring. 80% of competitors will use this. You lack a "Magic Trick"—a differentiator that makes non-technical judges gasp.
2.  **The "Phone/Book" Blindspot:** By cutting YOLO, you have no direct way to detect the most common malpractice: holding a phone or book. Relying solely on Gaze fusion to catch this is risky; a student holding a phone just below the webcam looks identical to a student looking at the keyboard.
3.  **Visualization vs. Logic:** Your logic is deep (fusion, hysteresis), but your dashboard description (Vanilla JS grid) sounds dry. In a hackathon, the UI is 50% of the score. If the judge doesn't *see* the fusion logic (e.g., a visualization of the audio rejection), it didn't happen.

CONCERNS:
1.  **The "Head Down" Ambiguity:** Without object detection, you cannot distinguish between "looking at keyboard to type" and "looking at phone in lap." High false positives here will kill your "Precision" narrative.
2.  **Enrollment as a Single Point of Failure:** If the enrollment photo is bad (bad lighting/blur), your robust MobileFaceNet verification becomes a false-positive generator for the entire exam.
3.  **Audio "Fusion-Only" Risk:** If a student whispers answers to a neighbor without looking at them (no gaze trigger), your system misses it entirely.
4.  **Boring Demo Syndrome:** A grid of faces is standard. If you don't visualize the *invisible* logic (like common-mode audio rejection), the judges won't know you built it.

SUGGESTIONS:

**1. The "Magic Trick" (High Impact, Low Effort): Screen-Flash Liveness**
*   **Why:** You need a wow factor.
*   **How:** During random verification checks, flash the screen user interface (white/black) for 200ms. Measure the intensity change on the user's face (skin reflection) using the webcam.
*   **Result:** Simple CV math (no heavy model). Detects photo/video spoofing instantly. Judges love "active" security.

**2. Solve the "Phone" Problem without YOLO: "Geometric Proxy"**
*   **Why:** You can't run YOLO, but you need to detect phones/books.
*   **How:** Use Face Mesh to calculate **Head Pitch** (up/down angle).
*   **Logic:** If `Head Pitch > 25 degrees` (looking down) AND `Face Size` implies distance < 50cm AND `Duration > 3s`, trigger **SUSPICIOUS_POSTURE**.
*   **Sell:** "We don't need heavy object detection; we analyze biomechanics."

**3. "Visualized" Common-Mode Audio (The Demo Winner)**
*   **Why:** The jury is non-technical. They won't read code.
*   **How:** On the dashboard, show a live audio waveform for the room. When a loud noise happens (e.g., a bell), draw a **GREEN** box around all terminals saying "Environment Noise - IGNORED." When one student speaks, draw a **RED** box "Localized Speech - ALERT."
*   **Impact:** This visually proves your distributed system intelligence.

**4. The "Cheat-o-Meter" (Continuous Risk Score)**
*   **Why:** Binary alerts are harsh.
*   **How:** Implement the "Anomaly Scoring" you investigated. Start at 0. Add points for gaze deviation, looking down, audio spikes.
*   **UX:** Show a "Heat Bar" next to every student. Green -> Yellow -> Red. It feels more "AI-driven" than simple toggles.

**5. Demo Scripting (Theatricality)**
*   **Scenario 1:** The "Ghost" (Candidate leaves). System alerts in 5s.
*   **Scenario 2:** The "Imposter" (Swap seats). System alerts immediately.
*   **Scenario 3 (The Climax):** The "Synchronized Cheat." Have the proctor clap loudly (Audio). Show the system *ignoring* it across all screens. Then have *one* person whisper. Show the system *catching* it.

KEY_TRADEOFFS:
*   **Active Liveness vs. UX:** Flashing the screen is intrusive to the student, but acceptable for a "High Security" government exam demo.
*   **Geometric Proxy vs. True Object Detection:** You will miss a phone held *up* to the face, but you capture the 90% use case (phone in lap) with 0% extra CPU cost.
*   **Score vs. Alert:** A risk score is harder to explain in 10 seconds than "Red Alert," but it prevents the "False Positive" critique by allowing the proctor to decide the threshold.

**Verdict:** Your architecture is safe but needs "drama." Implement the **Screen Flash Liveness** and the **Visualized Audio Rejection** to win.

---
## Round 2

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After reviewing GPT-5.2 and Gemini's feedback alongside my own, I'm more convinced the architecture is a strong foundation but now see clearer convergence on where the critical gaps are. Let me synthesize honestly.

**Where all three of us converge (this is now settled):**

1. **The object/phone detection gap is existential.** All three reviewers independently flagged this as the #1 risk. This isn't a nice-to-have — it's the difference between 6/10 and 8/10 detections. The team must solve this somehow within CPU constraints.

2. **Audio and gaze as fusion-only is too conservative for the scoring rubric.** All three of us agree that sustained audio needs a standalone promotion path, and prolonged gaze deviation should be actionable on its own.

3. **The dashboard needs to tell a visual story.** Non-technical judges evaluate what they can see. A grid of faces with red borders isn't enough.

4. **The core edge-first architecture is correct.** Nobody is arguing against the deployment model, PyQt6 choice, or episode-based event design. This is genuinely good engineering.

**Where I've updated my thinking based on others' feedback:**

- **GPT-5.2's OS/screen integrity telemetry suggestion is better than my MediaPipe Hands suggestion.** I was focused on the camera pipeline for phone detection, but OS-level signals (Alt-Tab, window focus loss, process blacklist, clipboard monitoring) are simpler, more reliable, have near-zero false positive rates, and cost zero CPU on the vision pipeline. This is the single highest-ROI addition possible. Alt-Tab detection is ~20 lines of Python. Process scanning is another ~30. These map directly to staged scenarios (candidate tries to open a browser, use a second app, look something up). I'd now prioritize this over MediaPipe Hands.

- **GPT-5.2's suggestion to replace Face Mesh with sparse landmark head pose (solvePnP on 5 keypoints)** is architecturally sound. Face Mesh at 10ms/frame for a fusion-only signal is expensive. If BlazeFace already outputs 5 keypoints, you get head pose estimation essentially for free. This saves ~10ms/frame — enough headroom for one additional capability.

- **Gemini's "Screen-Flash Liveness" is clever but I have reservations.** It's theatrically impressive and technically simple. However, it creates a UX disruption that could annoy the jury if it fires during the demo. It also doesn't work against the threat model that matters for the POC (the staged scenarios won't include sophisticated spoofing attacks — they'll be physical malpractice). I'd implement it as an enrollment-only check rather than a random interruption.

- **Gemini's "Geometric Proxy" for phone detection (head pitch + face size + duration)** is a smart complement. It won't catch everything, but combined with OS telemetry, it covers both physical-phone-in-lap and digital-phone-on-screen scenarios. The two approaches together are better than either alone.

**Where I disagree with the other reviewers:**

- **Gemini overweights theatricality relative to reliability.** The "visualized common-mode audio rejection" demo where you clap and show it being ignored is genuinely compelling, but it requires the audio pipeline to work flawlessly in a noisy demo environment. If it misfires during the demo (suppresses real cheating or fails to suppress noise), it's catastrophic. I'd show this as a pre-recorded replay rather than live, or only if extensively tested in the actual demo room.

- **GPT-5.2's suggestion to replace BlazeFace with YuNet/SCRFD** is premature optimization for a 7-10 day sprint. BlazeFace works. Switching face detectors cascades into recalibrating every downstream threshold. This is a "week 3" improvement, not a "day 4" pivot.

- **Both reviewers underweight the exam interaction analytics signal.** Since the team controls the PyQt6 exam interface, answer timing, navigation patterns, and clipboard events are essentially free signals that require no CV at all. Neither GPT-5.2 nor Gemini emphasized this strongly enough. A student who answers a hard question in 2 seconds after a face-swap event is a slam-dunk confirmed impersonation. This cross-modal fusion is genuinely novel and differentiating.

**Revised "winning additions" priority stack (in order):**

| Priority | Addition | Effort | Impact | Risk |
|----------|----------|--------|--------|------|
| 1 | OS/screen integrity telemetry | 1 day | Critical — catches 2-3 scenarios | Very low |
| 2 | Promote sustained audio/gaze to standalone | 0.5 day | High — prevents missed detections | Low |
| 3 | Head pose from sparse landmarks (replace Face Mesh) | 1 day | Medium — saves 10ms, more robust | Medium |
| 4 | Exam interaction analytics | 1 day | High — unique differentiator, zero CPU | Low |
| 5 | Geometric proxy for phone posture | 0.5 day | Medium — covers physical phone use | Low |
| 6 | Temporal risk score + sparkline visualization | 1 day | High for demo impact | Low |
| 7 | Liveness check at enrollment (Face Mesh z-coords) | 2 hours | Low-medium, but impressive | Very low |
| 8 | Risk Timeline visualization | 1 day | High for demo | Medium |

Total: ~6 days of parallel work. Leaves 2-4 days for integration, testing, and demo rehearsal.

CONCERNS:

1. **Scope creep is now the primary risk.** Three reviewers have collectively suggested ~12-15 additions. The team must ruthlessly prioritize. Adding 8 features in 7 days and having 3 of them buggy during the demo is worse than having 4 solid features that work perfectly. My priority table above reflects what I'd actually ship.
2. **OS telemetry is platform-dependent.** The terminals might run Linux, Windows, or a locked-down custom OS. Alt-Tab hooks, process scanning, and clipboard monitoring all differ by platform. The team needs to confirm the OS environment immediately and build for that specific target. If it's a locked-down Linux kiosk, some OS signals won't apply.
3. **The Face Mesh replacement (sparse landmark head pose) carries integration risk.** If BlazeFace's 5 keypoints aren't accurate enough for solvePnP, you fall back to Face Mesh anyway and have wasted a day. Prototype this in 2 hours before committing.
4. **Demo environment unknowns.** We don't know the room layout, lighting, camera quality, or network conditions for the actual POC. All three reviewers assume controlled conditions. Budget half a day for on-site calibration and threshold adjustment.
5. **Common-mode audio rejection in the demo room.** If the demo is in a conference room with echo, the 500ms cross-terminal correlation window might trigger false suppressions. Test with the actual room acoustics or disable common-mode rejection for the demo and re-enable for deployment pitch.
6. **The ±15s matching window concern remains.** All three reviewers agree this is tight. Widen to ±20s and provide a "near-match review" UI element on the dashboard so judges can manually accept borderline matches.

SUGGESTIONS:

1. **Day 1-2: OS integrity telemetry + promote audio/gaze standalone thresholds.** These two changes alone could shift detection from 6/10 to 8/10 by covering digital cheating scenarios and standalone verbal/visual cheating. This is the "must ship" tier.
2. **Day 2-3: Exam interaction analytics + geometric proxy for phone posture.** These add two novel signal modalities (behavioral + postural) with minimal CPU cost and minimal integration risk. This is the "should ship" tier.
3. **Day 3-4: Temporal risk score with dashboard sparklines.** This transforms the dashboard from "alert log" to "behavioral intelligence platform" and is the single biggest demo differentiator. Display the risk score prominently on each terminal tile.
4. **Day 4-5: Head pose from sparse landmarks (only if prototype validates in 2 hours).** If BlazeFace keypoints give stable yaw/pitch via solvePnP, swap out Face Mesh and reclaim 10ms/frame. If not, keep Face Mesh and skip this.
5. **Day 5-6: Polish dashboard visualization + liveness at enrollment.** Add the audio visualization Gemini suggested (but as a pre-recorded comparison clip in the presentation, not a risky live demo). Add z-coordinate liveness check. Polish terminal tile colors, episode feed, evidence viewer.
6. **Day 7-8: Integration testing + demo rehearsal.** Non-negotiable. Run the full 10-scenario sequence at least 5 times. Time every alert. Identify which scenarios you reliably catch and which you don't. Script the demo around your strengths. Prepare explanations for any gaps.
7. **For the presentation:** Lead with the problem (exam fraud costs AP ₹X crore annually), show the "before" (no proctoring), show ExamGuard catching 8+ scenarios in real-time, show the confusion matrix filling in live, end with the ₹1-per-student-per-exam scalability number. Name every feature in bureaucrat-friendly language. Keep it under 10 minutes.

KEY_TRADEOFFS:

1. **OS telemetry vs. camera-only purity.** Adding OS-level signals means the system is no longer a "pure AI vision" solution — it's a hybrid. Some may see this as less innovative. But it's dramatically more effective for the actual scoring criteria, and government officials care about results, not architectural purity. **Right tradeoff: ship what catches cheating.**

2. **Feature count vs. reliability.** The converged recommendation adds 6-8 features to a previously clean 3-alert system. Each addition is individually low-risk, but the cumulative integration complexity is real. The mitigation is the strict day-by-day schedule above with a hard cutoff at day 6 for new features. **The tradeoff is correct if and only if you protect the last 2 days for testing.**

3. **Replacing Face Mesh vs. keeping it.** GPT-5.2's suggestion saves CPU but carries recalibration risk. My recommendation: prototype in 2 hours, decide based on data. If the sparse approach works, you get 10ms headroom. If not, Face Mesh is already validated and you lose only 2 hours. **This is a low-cost option, not a commitment.**

4. **Live demo theatrics vs. pre-recorded safety.** Gemini's audio visualization demo is powerful but risky live. GPT-5.2's liveness challenge is impressive but disruptive. My position: **do one live "wow moment" (the common-mode audio rejection, if tested) and keep everything else as reliable automated detection.** Don't make the demo depend on multiple theatrical moments all working perfectly.

5. **All three reviewers agree on the fundamental tension:** the architecture optimizes for deployment (edge-first, privacy, low bandwidth) but underoptimizes for hackathon scoring (detection coverage, visual drama, "wow" features). The additions above bridge that gap without sacrificing the deployment story. **The winning strategy is: deploy-grade reliability + hackathon-grade detection coverage + one memorable demo moment.**

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position is unchanged (still partially agree), but **Claude/Gemini’s feedback makes one point more urgent**: the current stack is likely to be *scored against staged scenarios you don’t cover*, especially **phone/notes + “leaving the exam app”**. Those misses are not “a bit lower recall”; they are **hard zeros in a 10-scenario confusion-matrix game**. So while the architecture is deployable and robust, it is **competitively under-armed** unless you add 2–3 high-signal modalities fast.

Where we converge strongly:
- **Base architecture is sound** (single native agent, edge events, episode aggregation, evidence-on-alert). This will run when others crash.
- **Innovation gap is real**: “MediaPipe + MobileFaceNet + dashboard” is table stakes.
- **Coverage gap is the real risk**, not micro-optimizing thresholds: object/phone/notes and OS/app integrity are likely to be staged.
- **Temporal aggregation/risk scoring without RNNs** is a cheap, high-leverage differentiator.
- **Demo storytelling/UI is a decisive factor** for non-technical judges; if they can’t “see” the intelligence, it won’t count.

Where I disagree / would qualify others’ suggestions:
- **Training a custom NanoDet in 1–2 days** is risky unless you already have data + training pipeline + evaluation loop. For a hackathon, this is a classic “looks easy, eats the sprint” trap.
- **MediaPipe Face Mesh “z-flatness” as liveness**: Face Mesh “z” is not true depth; it’s a model-relative coordinate and can be fooled by replay/video. It can be a weak heuristic, not a security claim. Active liveness is more defensible.
- **MediaPipe Hands runtime claims** (e.g., “12ms INT8”) are optimistic; MediaPipe on desktop Python is often bottlenecked by graph overhead and may not be trivially INT8. Still feasible, but you must measure on target CPUs before committing.

CONCERNS:
1. **(Agree with Claude/Gemini)** No robust response to phone/notes/physical aids = likely 2–3 scenario misses → threatens 8/10.
2. **(Agree)** No OS/app integrity telemetry = easy staged “Alt-Tab / open browser / remote tool” miss that other teams may catch with simple hooks.
3. **(Agree)** Gaze/FaceMesh is compute-heavy relative to its current fusion-only role; risk of spending CPU without scoring benefit.
4. **(Agree)** Audio fusion-only can miss an explicitly staged “candidate talking” scenario; a conservative standalone speech episode is safer for scoring.
5. **(Agree)** Demo risk: if the dashboard doesn’t visually explain “why” (episodes, fusion, common-mode suppression), you lose perceived innovation.
6. **(Disagree/qualify)** Custom detector training in sprint: high integration risk, dataset risk, and evaluation risk under time pressure.
7. **(Qualify)** Passive liveness heuristics (mesh z/flatness) are easy to demo but easy to attack; don’t oversell as “anti-spoof.”

SUGGESTIONS:
1. **Add OS / exam-app integrity telemetry (highest ROI, 1 day, near-certain scoring wins)**  
   Standalone alerts: `APP_FOCUS_LOST`, `FULLSCREEN_EXIT`, `ALT_TAB`, `MULTI_MONITOR`, `BLACKLIST_PROCESS` (AnyDesk/TeamViewer/Chrome/etc.), `CLIPBOARD_PASTE` (if applicable).  
   This is both differentiating and highly explainable to officials.

2. **Cover “phone/notes” without full YOLO: posture + hands-zone heuristics (1–2 days, lower risk than training a detector)**  
   Options in increasing complexity:
   - **Head-pose/down-look episode** (cheap): sustained pitch down + low face motion + repeated pattern → `SUSPICIOUS_READING`.  
   - **Hand-to-ear / hand-near-face dwell** (if you add a lightweight hand landmark model or even skin-color/motion ROI heuristics): `POSSIBLE_PHONE_CALL`.  
   Importantly: present these as **“behavioral proxies”** (low-FP episodes), not definitive object detection.

3. **Add active liveness on suspicion (1 day, high jury impact, more defensible than passive depth)**  
   Trigger only when face mismatch/uncertain or re-entry occurs:
   - “Blink twice” or “turn head left-right” verified via sparse landmarks/head-pose.  
   Gemini’s **screen-flash reflectance** can be a great demo trick, but test camera auto-exposure; if it’s unstable, keep it as an optional secondary check, not your only liveness.

4. **Temporal risk score (0.5–1 day): make the system look “intelligent,” not rule-based**  
   Implement EWMA/CUSUM-style scoring over 30–60s of: face absent, mismatch jitter, down-look/headpose variance, speech duration, app-focus losses.  
   Render as a **Cheat-o-meter** per tile + per-candidate timeline. This is common ground across all feedback and is genuinely differentiating.

5. **Promote sustained speech to a standalone low-severity episode (0.5 day)**  
   Keep your long thresholds to protect precision, but don’t require gaze to fire. This protects against a simple staged “talking” event.

6. **Dashboard “show the intelligence” (1 day, likely worth more than another model)**  
   - Risk timeline per candidate (green/yellow/red segments)  
   - Visualize common-mode audio suppression (“environment noise ignored” banner)  
   - Evidence thumbnails embedded in the event feed  
   This turns your invisible engineering (episodes/fusion) into judge-visible innovation.

7. **Scoring/window robustness**  
   Consider widening matching tolerance (±20–25s) or show an explicit “episode start/end” timestamping method to avoid losing points to annotation jitter. At minimum, test the ground-truth logging workflow end-to-end.

KEY_TRADEOFFS:
- **Precision vs coverage (hackathon reality):** Your current design optimizes trust (low FP) but risks missing entire scenario classes. For an 8/10 target, you must shift slightly toward coverage via additional modalities (OS telemetry + posture/speech).
- **Model complexity vs integration risk:** Adding a trained object detector could improve accuracy, but it’s the highest schedule risk. Behavioral proxies + OS signals give most of the scoring benefit with less fragility.
- **Security rigor vs demo impact:** Active liveness and screen-flash checks are “wow” features, but can affect UX and reliability. Use them as *on-suspicion* checks and avoid overclaiming.
- **Engineering excellence vs judge-visible value:** Episodes/fusion are great, but unless the UI narrates them, they won’t be credited. Investing in visualization is an architectural decision for winning, not just “UI polish.”

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The consensus across all three critiques is clear: **Your architecture is an operational tank, but a competitive sedan.** You will likely have the most stable system in the room, but you are at high risk of losing because you fail to detect specific, common cheating scenarios (phones, notes, OS abuse) that other teams *will* catch.

**Converged Analysis:**
1.  **The "Blind Spot" Consensus:** We all agree that rejecting YOLO created a vacuum. You cannot detect a student holding a phone, reading a note, or Alt-Tabbing to ChatGPT. Relying on "Gaze Fusion" to catch these is gambling that the student will look away long enough to trigger your conservative thresholds.
2.  **The "Boring Dashboard" Problem:** Claude and I both emphasize that a grid of video feeds is not a winning demo. Judges need to see *thinking*, not just *seeing*. A "Risk Score" or "Timeline" (Claude) or "Waveform" (Me) is mandatory.
3.  **The OS Opportunity (GPT-5.2's Critical Insight):** I missed this in round 1, but GPT-5.2 is absolutely right. Since you are building a desktop app (PyQt6), ignoring OS-level signals (Alt-Tab, clipboard, VM detection) is a massive missed opportunity. It costs 0% CPU and catches 30% of cheating methods.

CONCERNS:
1.  **Zero Defense Against OS Cheating:** If a student opens a browser window overlay or uses a screen sharing tool, your camera-based system sees nothing wrong. This is an instant "fail" from a judge.
2.  **MediaPipe Hands vs. Head Pitch:** I previously suggested using Head Pitch as a proxy for phone usage. Claude argues for **MediaPipe Hands**. I am changing my stance to agree with Claude. Head pitch is too ambiguous; detecting a hand near the ear or holding an object is definitive.
3.  **Audio Conservatism:** We all worry your "Fusion-Only" audio rule is too strict. If two students whisper without looking at each other, you miss it.
4.  **Face Mesh Efficiency:** GPT-5.2 notes correctly that running full Face Mesh just for fusion-only gaze is a poor ROI on CPU cycles.

SUGGESTIONS:

**1. The "Zero-Cost" Win: OS Telemetry (Agreed with GPT-5.2)**
*   **Implementation:** In PyQt6, hook `windowStateChanged`, check for external displays, and monitor clipboard changes.
*   **Demo:** Have a student try to Alt-Tab or plug in a second monitor. Show an immediate "SYSTEM INTEGRITY VIOLATION" alert.

**2. The "Object" Fix: MediaPipe Hands (Agreed with Claude)**
*   **Why:** My previous suggestion (Head Pitch) triggers false positives when tired students look down. Hands detection is specific.
*   **Logic:** Run MediaPipe Hands on the cropped body region.
    *   Hand touching face = `SUSPICIOUS_POSTURE` (Audio cheating?)
    *   Hand stationary in "holding" pose = `POSSIBLE_PHONE`
*   **Cost:** It fits your budget if you run it only when Gaze is effectively "down" or "away."

**3. The "Visualized Brain": Behavioral Risk Score (Synthesis)**
*   **Concept:** Instead of just binary alerts, implement the **Sliding Window Score** (Claude) and visualize it as a **Heat Bar** (Me/GPT).
*   **Math:** `Score = (Gaze_Dev_Duration * 1.5) + (Hand_Pose_Conf * 2.0) + (Audio_Energy * 0.5)`.
*   **Demo:** Show the bar rising slowly as a student fidgets, then turning RED when they pull out a phone.

**4. The "Flash" Liveness (Refined)**
*   I maintain that **Screen Flash** is superior to GPT-5.2's "Active Challenge" (asking students to move heads) because it doesn't disrupt the exam flow. However, add **Claude's Depth Check** (z-axis flatness) during enrollment to prevent the initial spoof.

KEY_TRADEOFFS:
*   **Complexity vs. Coverage:** Adding MediaPipe Hands and OS Telemetry adds integration complexity (Day 7-8 risk), but without them, you are capped at catching ~6/10 scenarios. You must take this risk to reach 8/10.
*   **CPU Budget:** Running Hands + Face Mesh + BlazeFace is tight on i5. **Mitigation:** Do not run Hands every frame. Run it only when `Gaze_Pitch > 15 deg` (looking down). This "conditional execution" saves the CPU.

**Final Verdict:**
Your operational backbone is winning quality. Your detection logic is losing quality.
**Adopt the "OS Telemetry" and "MediaPipe Hands" suggestions immediately.** These close your biggest gaps without breaking your architecture.

---
## Round 3

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

We have strong three-way convergence. I'll focus on what's settled, what's changed, and the one remaining disagreement that matters.

**What is now definitively settled (all three agree):**

1. **OS telemetry is the #1 addition.** Zero CPU cost, near-zero false positives, catches 2-3 staged scenarios that are otherwise invisible. This is no longer debatable — it's a must-ship item. The team controls the PyQt6 process; hooking `focusChanged`, scanning process lists, and monitoring clipboard are trivial and platform-testable in hours.

2. **Sustained audio and prolonged gaze must have standalone promotion paths.** The current fusion-only policy is too conservative for a scored hackathon. A student talking for 15+ seconds with no one else detected is cheating. A student staring away for 20+ seconds is cheating. These need to fire alerts independently, with conservative thresholds to protect precision.

3. **The dashboard must visualize intelligence, not just stream alerts.** Whether it's sparklines, heat bars, or risk timelines, the judges need to see the system *thinking*. This is unanimously the highest-impact demo investment.

4. **The core architecture is correct and likely more stable than competitors.** Edge-first, episode-based, precision-oriented. This foundation doesn't change.

5. **Temporal risk scoring is a cheap, high-impact differentiator.** EWMA or sliding-window aggregation over 30-60s windows, rendered visually. All three of us recommended this independently.

**Where my position has evolved:**

- **MediaPipe Hands: I'm now less confident this is the right call.** Gemini switched to support my earlier suggestion of MediaPipe Hands, but GPT-5.2's concern about runtime overhead on desktop Python is valid. MediaPipe's Python graph overhead is real — the "12ms INT8" figure I assumed comes from mobile/native benchmarks, not Python desktop. On a Celeron with 4GB RAM, adding another MediaPipe model graph could push total memory past safe limits and introduce unpredictable latency spikes. **I'm now converging toward GPT-5.2's "behavioral proxy" approach**: head-pitch-down episode + hand-region motion heuristics (skin-color blob in lower frame quadrant) as a lighter alternative. This is less precise but far less risky for the sprint timeline.

- **Active liveness vs. screen-flash vs. passive depth:** GPT-5.2's "blink twice / turn head" active challenge on suspicion is more defensible than my passive z-depth suggestion or Gemini's screen-flash. The reason: it works with any webcam, doesn't depend on auto-exposure behavior, and is immediately understandable to non-technical judges ("the system asked the student to prove they're real"). I'd now implement active liveness as an on-suspicion check triggered by face mismatch entering the uncertain zone. Keep it simple — "please turn your head left" verified via head pose change > 30°.

- **Face Mesh replacement priority: downgraded.** All three of us see the CPU waste, but the integration risk in a 7-10 day sprint is real. I previously listed this as Priority 3. I'm dropping it to Priority 7 (nice-to-have). If BlazeFace's keypoints give stable solvePnP in a 2-hour prototype, great. If not, ship with Face Mesh. The 10ms savings matters less now that OS telemetry and exam interaction analytics consume zero vision-pipeline CPU.

**The one remaining genuine disagreement:**

**Exam interaction analytics.** I maintain that GPT-5.2 and Gemini are both underweighting this. Since the team controls the exam application, behavioral signals from the exam itself — answer timing, question navigation patterns, rapid correct answers after a face-swap event, copy-paste attempts — are:
- Zero CPU cost on the vision pipeline
- Zero false positive risk (copy-paste is binary; answer timing is statistical)
- A genuinely novel fusion signal that no competitor building a camera-only solution will have
- Immediately explainable to judges: "The system noticed the student answered 5 questions correctly in 10 seconds right after someone else sat down"

GPT-5.2 doesn't mention it. Gemini doesn't mention it. But this is the kind of cross-modal intelligence that makes a jury say "nobody else did this." It's the difference between a proctoring camera and a proctoring *system*. I'd put this at Priority 3, after OS telemetry and audio/gaze standalone promotion.

**Revised final priority stack (reflecting convergence):**

| Priority | Addition | Effort | Consensus |
|----------|----------|--------|-----------|
| 1 | OS/screen integrity telemetry | 1 day | All 3 agree |
| 2 | Promote audio + gaze to standalone (conservative thresholds) | 0.5 day | All 3 agree |
| 3 | Exam interaction analytics (answer timing, navigation, clipboard) | 1 day | Claude advocates; others silent |
| 4 | Temporal risk score + visual heat bar per terminal | 1 day | All 3 agree |
| 5 | Head-pitch-down + motion-blob proxy for phone/notes | 0.5 day | Converged (lighter than Hands) |
| 6 | Active liveness on suspicion (head-turn challenge) | 0.5 day | GPT-5.2 proposed; others agree |
| 7 | Dashboard polish (timeline, evidence thumbnails, fusion viz) | 1 day | All 3 agree |
| **HARD CUTOFF** | | **5.5 days** | |
| 8 | Face Mesh → sparse landmark swap (only if 2hr prototype works) | 1 day | All 3 agree it's optional |
| 9 | Liveness depth heuristic at enrollment | 2 hours | Nice-to-have |

Remaining 2-4 days: integration testing, demo rehearsal, on-site calibration.

CONCERNS:

1. **Scope creep remains the #1 execution risk.** We've converged on 7 additions above the hard cutoff. Even with conservative effort estimates, that's 5.5 days of development. If the team has 3 developers working in parallel, this is feasible. If it's 1-2 people, cut items 5-7 and ship 1-4 bulletproof. A flawless 5-feature system beats a buggy 8-feature system every time.
2. **OS telemetry platform dependency is still unresolved.** The team needs to confirm the terminal OS within 24 hours. If it's Windows, `pywin32` gives you everything. If it's Ubuntu, `xdotool` and `/proc` scanning work. If it's a locked-down kiosk OS, some hooks may be unavailable. Build the abstraction layer first, platform-specific implementations second.
3. **The "behavioral proxy" for phone detection is weaker than object detection.** All three of us acknowledge this. Head-pitch-down episodes will generate some false positives (tired students, reading the question carefully). The mitigation is the same as the rest of the system: conservative thresholds, episode-based aggregation, and framing it as "SUSPICIOUS_POSTURE" rather than "PHONE_DETECTED." Judges will understand a behavioral indicator flagged for human review.
4. **Demo environment remains the wild card.** Lighting, camera quality, room acoustics, network stability — none of these are known. The team must build a "demo mode" toggle that can adjust all thresholds by ±20% on the fly. A single slider labeled "Sensitivity" that scales all thresholds simultaneously would let them calibrate in the room in 60 seconds.
5. **The ±15s matching window needs to be tested end-to-end with the ground truth logging workflow.** If the ground truth logger adds 3-5 seconds of human reaction time to the timestamp, you effectively have ±10s of real tolerance. Widen to ±20s and test the full logging workflow 5+ times before demo day.

SUGGESTIONS:

1. **Implement Priority 1-4 in parallel across team members.** OS telemetry and exam interaction analytics are completely independent of the vision pipeline. One developer handles those while another implements audio/gaze standalone promotion and temporal risk scoring within the existing event system. These four items are architecturally orthogonal — no merge conflicts, no dependency chains.
2. **Build a "scenario coverage matrix" immediately.** List all 10 likely staged scenarios (candidate absent, impersonation, phone use, note reading, talking, looking at neighbor's screen, Alt-Tab, second person enters, coordinated cheating, identity swap mid-exam). Map each to which detection modality covers it. Identify any scenario with zero coverage and prioritize closing that gap. This is the single most important planning artifact for the remaining sprint.
3. **For the demo, script the scenario order to build narrative tension.** Start with the easy wins (candidate leaves seat → ABSENT detected in 5 seconds). Build to medium difficulty (someone talks → standalone audio fires). Climax with the multi-modal catch (face swap + rapid correct answers + gaze deviation → CONFIRMED_IMPERSONATION with exam analytics correlation). End with the OS integrity demo (Alt-Tab → instant alert). This narrative arc is more persuasive than random scenario ordering.
4. **Prepare a 60-second "architecture differentiator" slide.** Show a comparison: "Cloud-based solutions: 200ms latency, privacy risk, bandwidth dependency" vs. "ExamGuard: <50ms local detection, zero data leaves the center, works offline." This reframes the edge-first architecture from a technical choice into a policy advantage for government deployment. Government officials will immediately understand "student photos never leave the exam center."
5. **Build the confusion matrix visualization as a live-updating element during the demo.** As each scenario plays out, show the matrix cell filling in real-time. This makes the scoring methodology transparent and builds confidence. If you catch 8/10, the matrix tells that story better than any slide.

KEY_TRADEOFFS:

1. **Behavioral proxies vs. object detection accuracy.** We're collectively recommending head-pitch + motion-blob instead of YOLO or a trained object detector. This will produce lower precision on phone/notes detection than a dedicated model. But it ships in 0.5 days vs. 3-5 days, has no licensing issues, and combined with OS telemetry, covers most of the gap. For a hackathon sprint, this is the right tradeoff. For production, you'd invest in a proper object detector.

2. **Exam interaction analytics: novel but unproven.** No other reviewer has validated this suggestion. It's genuinely differentiating if it works, but the team needs to define the specific signals and thresholds during implementation. The risk is that answer timing patterns are too noisy in a 10-minute exam with only ~20 questions to establish a statistical baseline. Mitigation: use it only as a fusion signal that *confirms* other alerts, not as a standalone detector. A fast correct answer after a face-swap is damning; a fast correct answer alone means nothing.

3. **7 features vs. 4 bulletproof features.** The priority stack above is ambitious. The honest assessment: if the team is 3+ strong developers, ship all 7 above the cutoff. If it's 1-2 people, ship only Priority 1-4 and invest the saved time in testing and demo polish. **A perfectly executed 6/10 detection with beautiful visualization and clear explanations will outscore a buggy 8/10 detection with a confusing demo.** But a clean 8/10 with good visualization wins outright.

4. **The fundamental competitive bet:** We're betting that "edge-first reliability + multi-modal intelligence (vision + OS + exam behavior) + visual storytelling" beats "cloud-based + more models + more features but less stable." This is the right bet for a government hackathon where deployability and trust matter. But it only works if the detection coverage actually reaches 8/10. The additions above are designed to close that gap.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position does not change: the architecture is **deployable and robust**, but **competitively incomplete** for a confusion-matrix, staged-scenario hackathon unless you add 2–3 “coverage” signals quickly.

**What changed after reading Claude/Gemini:** I’m more confident that the winning delta is *not* “better models,” it’s **closing scenario-class blind spots with non-CV signals** (OS/app integrity + exam-app telemetry) and adding **one physical-cheating proxy** (hands/posture) without blowing CPU.

### Common ground (clear convergence across all 3 reviews)
- **OS/app integrity telemetry is the highest-ROI add.** Near-zero CPU, low FP, maps directly to common staged events (“Alt-Tab”, browser, remote tools, second display). This is the most “unfair advantage” because many teams stay camera-only.
- **Fusion-only audio/gaze is too conservative for scoring.** At least one of these needs a standalone episode path (conservative thresholds).
- **Dashboard must visualize “thinking.”** Risk timeline/score, episode spans, evidence thumbnails. Otherwise your best engineering is invisible.
- **Object/phone/notes gap is existential.** If the jury stages “phone in lap / reading notes,” you need *some* detection path—even if it’s a proxy.

### Where I genuinely disagree / qualify
- **MediaPipe Hands as “the” object fix:** It can help, but it’s not a guaranteed win. Hands models can be fragile under occlusion/low light/cheap webcams, and “holding phone” classification is non-trivial without an object model. I’d treat Hands as a **conditional, supportive signal** (triggered sparsely) rather than betting the sprint on it.
- **Screen-flash liveness as superior to active challenge:** I don’t buy “non-disruptive” in practice. Flashing the screen *is* disruptive and can misbehave with auto-exposure, different monitors, and ambient lighting. Active challenge (blink/head turn) is more controllable and easier to message, but should be **on-suspicion only**.
- **FaceMesh replacement with sparse head-pose:** I agree with Claude that it’s a good trade *if* keypoints are stable, but it’s a **prototype-first** decision; otherwise you risk burning time recalibrating.

CONCERNS:
1. **Feature creep / integration risk is now #1 operational threat.** The combined suggestions easily exceed sprint capacity; shipping 8 half-working features loses to 4 rock-solid ones.
2. **Platform dependency of OS telemetry.** Windows vs Linux kiosk changes everything (Alt-Tab hooks, process enumeration permissions, clipboard APIs, multi-monitor detection). You must confirm target OS immediately.
3. **Hands detection reliability on low-end webcams.** Occlusions (desk edge), partial hands, motion blur, and camera placement can tank recall and increase FP.
4. **Audio demo fragility.** Common-mode suppression and waveform theatrics can backfire in an echoey room; if shown, it needs rehearsal in similar acoustics or be demoed as replay.
5. **Compute ROI of Face Mesh remains questionable** if it stays fusion-only; it can crowd out higher-scoring features.
6. **Scoring sensitivity (±15s window) and ground-truth tooling.** If annotation is manual, you can lose points due to timestamp jitter more than model error.

SUGGESTIONS:
**A ruthlessly prioritized “win stack” (minimize risk, maximize scenario coverage):**

1. **Must-ship (Day 1–2): OS/app integrity telemetry**
   - Alerts: `APP_FOCUS_LOST`, `FULLSCREEN_EXIT`, `ALT_TAB/TASK_SWITCH` (or Linux equivalent), `SECOND_MONITOR`, `BLACKLIST_PROCESS` (AnyDesk/TeamViewer/Chrome), `NETWORK_DROP`.
   - Evidence: screenshot of your app losing focus + active window title/process list (where permitted).
   - This is both *differentiating* and *judge-legible*.

2. **Must-ship (Day 2): Exam interaction analytics (Claude is right—this is underweighted)**
   - Signals you control in PyQt: copy/paste attempts, abnormal navigation bursts, answer-time anomalies, “idle then sudden perfect streak,” window blur/focus events correlated with answer changes.
   - Fusion: “Integrity violation + sudden answer change burst” → high-confidence episode.
   - Zero CV cost, high innovation optics (“multimodal: screen + behavior + vision”).

3. **Should-ship (Day 2–3): Promote sustained speech to standalone episode**
   - Keep conservative thresholds (e.g., sustained voiced segments >8–10s, repeated within 60s).
   - Don’t require gaze. This covers an entire staged class cheaply.

4. **Should-ship (Day 3–4): Risk score + timeline UI**
   - Implement simple EWMA/CUSUM over episodes + integrity events.
   - Show per-terminal “risk bar” + 10-minute timeline with colored segments and clickable evidence.
   - This is your “nobody else did this” moment without new ML.

5. **Optional (only if time + measured on target): Hands OR posture proxy**
   - If you add **Hands**, run it **conditionally** (e.g., only when head pitch/down-look or face bbox drops low in frame for >N seconds).
   - If you skip Hands, implement **head-pose/down-look + dwell** as a conservative proxy (`SUSPICIOUS_READING`)—not as “phone detected.”
   - Either way: message it as “behavioral suspicion,” not definitive object detection.

6. **CPU reallocation move (prototype in 2 hours): replace Face Mesh**
   - Try head pose from sparse landmarks / solvePnP; if stable, you reclaim budget for Hands or higher FPS.
   - If not stable, keep Face Mesh and accept the cost; don’t force a pivot.

7. **Liveness: keep it minimal and defensible**
   - Do **on-suspicion active challenge** (blink/head turn) rather than relying on screen flash.
   - If you want flash for “wow,” gate it to enrollment only and keep it optional.

KEY_TRADEOFFS:
- **Coverage vs purity:** Adding OS + exam telemetry makes this a hybrid proctor (not “pure AI vision”), but it maximizes detection points and lowers FP. For this hackathon rubric, that’s the correct trade.
- **Innovation vs reliability:** “Hands/flash/liveness theatrics” look innovative, but OS telemetry + interaction analytics are *quietly* more reliable and score-relevant. Innovation that fails live is net-negative.
- **Compute allocation:** Spending 10ms/frame on Face Mesh for a fusion-only signal is hard to justify if it prevents you from adding a scenario-covering feature (Hands or higher FPS detection). Prototype-driven decision is the right compromise.
- **More detectors vs better explanation:** Given non-technical judges, a risk timeline + evidence viewer can outperform an extra model in perceived intelligence and trust.

Overall convergence: keep your backbone, but shift the “winning layer” to **integrity + interaction analytics + risk visualization**, then add **one physical-cheating proxy** only if it doesn’t endanger stability.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
**The Consensus:** There is a rare and powerful unanimity across all three critiques. We all agree that your current architecture is **technically robust but competitively naive.** You have built a system that will not crash, but it will also not win, because it ignores the two highest-ROI signals available to you: **OS Telemetry** and **Visualized "Thinking."**

**The Pivot Point:**
You are treating this like a Computer Vision problem. It is not. It is a **Proctoring** problem.
1.  **The "OS Gap" is Fatal:** All three of us agree that ignoring OS-level signals (Alt-Tab, clipboard, external monitor) in a desktop app is a massive strategic error. It costs near-zero CPU and catches the most common cheating methods (Googling answers, ChatGPT) that your camera blindly misses.
2.  **The "Invisible Brain" Problem:** We all agree that complex fusion logic (Audio+Gaze) is useless if the judge doesn't *see* it happen. A grid of video feeds is 2018 tech. A "Risk Timeline" or "Cheat-o-Meter" that visibly spikes when multiple signals correlate is 2025 tech.
3.  **The CPU/Feature Trade-off:** We differ slightly on *how* to catch phones (Hands vs. Geometry), but we agree you cannot leave the "phone in lap" scenario uncovered.

CONCERNS:
1.  **The "Alt-Tab" Humiliation:** If a judge asks, "What if I just open Chrome?" and your answer is "We don't check that," you lose immediately to the team that wrote 10 lines of code to hook window focus events.
2.  **The "Static" Demo:** A 10-minute exam with episode-based alerts might result in a boring dashboard that sits idle for 9 minutes. Judges need to see the "Risk Score" fluctuating even when no alert is sent, proving the system is vigilant.
3.  **Audio Conservatism:** We collectively fear your "Fusion-Only" audio rule will result in the "Whispering Student" scenario being missed entirely.
4.  **The i5 Bottleneck:** Adding MediaPipe Hands or complex logic threatens your 15fps target on Celeron/i5 hardware.

SUGGESTIONS:

**1. The "Must-Have" Integration: OS Telemetry (Consensus Priority #1)**
*   **Action:** Since you are using PyQt6, use `QApplication.focusWindowChanged` and `QClipboard`.
*   **Detect:** Alt-Tab (Focus Loss), VM presence (simple MAC/Bios check), External Display connection, Clipboard copy/paste.
*   **Cost:** <1ms CPU. **Impact:** Catches 30% of cheating scenarios instantly.

**2. The "Conditional" Vision Upgrade: MediaPipe Hands (Resolution)**
*   **The Conflict:** I wanted Hands; Claude wanted Geometry/Head Pose to save CPU.
*   **The Fix:** **Conditional Execution.** Do not run MediaPipe Hands every frame.
*   **Logic:**
    1.  Run BlazeFace + Head Pose (Via 5-point solvePnP or FaceMesh).
    2.  **IF** `Head_Pitch > 15 deg` (looking down) **OR** `Face_Size < Threshold` (leaning back):
    3.  **THEN** Run MediaPipe Hands (checking for phone/notes).
*   **Why:** This satisfies the CPU constraints (runs rarely) while closing the "phone in lap" gap.

**3. The "Judge-Friendly" Dashboard: Visualized Risk Score (Consensus Priority #2)**
*   **Action:** Implement the **"Risk Thermometer"** or Sparkline.
*   **Formula:** `Risk = (Gaze_Dev * 0.5) + (Audio_Vol * 0.3) + (OS_Events * 1.0)`.
*   **Demo Strategy:** Even if a student isn't cheating, show the bar rising slightly when they fidget. This proves the AI is "watching" and "thinking," not just waiting for a threshold.

**4. The "Smart" Liveness: Active Challenge on Suspicion**
*   **The Conflict:** I liked Flash; Claude/GPT preferred Active Challenge or Passive Depth.
*   **The Fix:** **Active Challenge on Suspicion.**
*   **Logic:** Don't flash the screen (disruptive). Don't ask everyone to blink (annoying). **Only** when the `Risk Score > 0.7` (High), pause the exam and prompt: *"Please turn your head left and right to verify presence."*
*   **Why:** This turns a security feature into a UX feature—it only bothers potential cheaters.

KEY_TRADEOFFS:
*   **Purity vs. Pragmatism:** Adding OS hooks makes this a "hybrid" system rather than "pure AI." **Take the trade.** The judges want to stop cheating, not admire your neural network purity.
*   **CPU Volatility:** "Conditional" models (running Hands only sometimes) means CPU load will spike during suspicious events. You must ensure this spike doesn't freeze the UI. Threading (which you already have) is critical here.
*   **Recall vs. Precision (Audio):** Relaxing the audio rules to catch "whispering" will increase False Positives. **Mitigation:** Visualize the audio suppression (Common Mode Rejection) in the dashboard so judges see *why* an alert wasn't sent for background noise, preserving trust.

**Final Verdict:**
Your foundation is solid. To win, you must:
1.  **Add OS Telemetry** (Day 1).
2.  **Visualize the Risk Score** (Day 2-3).
3.  **Add Conditional Hands Detection** (Day 4).
Do these three things, and you move from "Competent" to "Winner."

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
1. The core architecture (PyQt6 single process, edge-first, episode-based events, ONNX INT8 models, push-based evidence) is sound and likely more stable than what most competitors will build.
2. OS/app integrity telemetry (Alt-Tab, focus loss, clipboard monitoring, process blacklist, multi-monitor detection) is the single highest-ROI addition — near-zero CPU cost, near-zero false positives, catches 2-3 staged scenarios that are otherwise invisible. All three models independently ranked this as Priority #1.
3. The current fusion-only policy for audio and gaze is too conservative for a scored hackathon. Sustained speech (>8-15s) and prolonged gaze deviation (>20s) must have standalone alert promotion paths with conservative thresholds.
4. The dashboard must visualize system "intelligence" — a grid of video feeds with binary alerts is insufficient for non-technical judges. A temporal risk score (EWMA/CUSUM-style), rendered as a heat bar or sparkline per terminal, is needed.
5. The phone/notes/physical-object detection gap is existential and threatens the 8/10 detection target. Some form of detection proxy is required.
6. No new heavy object detection model (YOLO) should be trained or integrated — the risk/time tradeoff is wrong for a 7-10 day sprint.
7. Feature creep and integration risk are now the #1 operational threats. Testing and demo rehearsal time must be protected (minimum 2 days).
8. The ±15s confusion matrix matching window is likely too tight; should be widened to ±20-25s.
9. The demo must be scripted and rehearsed 5+ times with escalating scenario difficulty and narrative arc.
10. Features must be named in bureaucrat-friendly language (e.g., "Triple-Check Verification" not "3-Zone Hysteresis") and the DPDP compliance story should be stated as outcomes, not architecture.
11. Face Mesh replacement with sparse-landmark head pose (solvePnP) is a good idea in theory but should only be attempted if a 2-hour prototype validates it; otherwise keep Face Mesh.
12. The system should be messaged as a multi-modal proctoring platform (vision + OS + behavioral analytics), not a pure CV solution.

CONVERGENCE:
1. **Phone/object detection approach:** Claude initially proposed MediaPipe Hands as the primary solution. Gemini initially proposed head-pitch geometric proxy. GPT-5.2 was cautious about both. By Round 3, all three converged on: use head-pitch/posture as a lightweight proxy, with MediaPipe Hands as a conditional/optional addition run only when posture triggers suggest it (e.g., sustained downward gaze). All agree to frame outputs as "SUSPICIOUS_POSTURE" behavioral indicators, not definitive object detection.
2. **Liveness detection:** Claude initially proposed passive Face Mesh z-depth. Gemini proposed screen-flash reflectance. GPT-5.2 proposed active challenge (blink/head turn). By Round 3, all converged on active liveness challenge on suspicion only (triggered by face mismatch or high risk score), with screen-flash demoted due to auto-exposure concerns and passive depth acknowledged as a weak heuristic.
3. **Face Mesh value:** All three initially noted Face Mesh was expensive for a fusion-only signal. By Round 3, all agreed to keep it unless sparse-landmark prototype succeeds in ≤2 hours, treating it as a low-priority optimization rather than a blocking issue.
4. **Exam interaction analytics:** Claude advocated strongly across all rounds. GPT-5.2 explicitly endorsed it in Round 3 ("Claude is right — this is underweighted"). Gemini remained silent on it but did not object. By Round 3, two of three explicitly support it as a high-priority, zero-CPU-cost differentiator.
5. **Audio demo risk:** Claude and GPT-5.2 converged on showing common-mode audio rejection as a pre-recorded replay or only after extensive on-site acoustic testing, rather than as a risky live demo. Gemini softened from insisting on live visualization to accepting that it needs rehearsal in similar acoustics.

REMAINING_DISAGREEMENTS:
1. **Exam interaction analytics priority:** Claude ranks this as Priority #3 (must-ship tier) and considers it a unique differentiator no competitor will have. GPT-5.2 endorsed it explicitly in Round 3 as Priority #2. Gemini never addressed it in any round — neither endorsing nor rejecting it. This is a two-agree-one-silent situation rather than a true disagreement, but the omission means there is no full three-way validation of its value or feasibility.
2. **MediaPipe Hands: conditional inclusion vs. skip entirely.** Gemini advocates conditional Hands detection as one of three "must-do" items (Day 4). Claude downgraded Hands in Round 3 due to Python overhead concerns and prefers head-pitch + motion-blob proxy as sufficient. GPT-5.2 treats Hands as optional — only if time permits and measured on target hardware. The disagreement is whether Hands is worth the integration risk given the sprint timeline.
3. **Screen-flash liveness as a viable technique.** Gemini still considers screen-flash superior in principle (non-disruptive to exam flow) though accepted active challenge as the group consensus. Claude and GPT-5.2 both rejected screen-flash due to auto-exposure instability and UX disruption concerns. This is a resolved-but-not-unanimous point.
4. **How many features to ship vs. reliability.** Claude explicitly states "a perfectly executed 6/10 detection with beautiful visualization outscores a buggy 8/10" and caps at 7 items with a hard cutoff. GPT-5.2 similarly warns about feature creep being the #1 threat. Gemini gives a more aggressive 3-item "must-do" list that includes conditional Hands detection on Day 4, implying tighter timelines are acceptable. The models differ on where exactly the ambition/reliability line should be drawn.

RECOMMENDED_PLAN:

**Phase 1: Must-Ship (Days 1-2) — Non-negotiable for competitive viability**
- [ ] Confirm target OS (Windows/Linux) within first 4 hours; this gates OS telemetry implementation
- [ ] Implement OS/app integrity telemetry: `APP_FOCUS_LOST`, `FULLSCREEN_EXIT`, `ALT_TAB`, `SECOND_MONITOR_DETECTED`, `BLACKLIST_PROCESS` (AnyDesk, TeamViewer, Chrome, WhatsApp), `CLIPBOARD_PASTE`. Use `QApplication.focusWindowChanged`, `QClipboard`, and platform-specific process enumeration
- [ ] Promote sustained audio to standalone alert: voiced speech >10s with face present → `SUSTAINED_SPEECH` (low-severity standalone episode). Remove requirement for gaze co-occurrence
- [ ] Promote prolonged gaze deviation to standalone alert: continuous off-screen gaze >20s → `PROLONGED_GAZE_DEVIATION` (low-severity standalone episode)
- [ ] Widen confusion matrix matching window from ±15s to ±20s

**Phase 2: Should-Ship (Days 2-4) — Strong differentiators, low integration risk**
- [ ] Implement exam interaction analytics in PyQt6: log answer timestamps, question navigation sequence, copy-paste attempts, idle-then-burst patterns. Generate `ANOMALOUS_ANSWER_PATTERN` and `CLIPBOARD_ATTEMPT` events. Fuse with vision events (e.g., face-swap + rapid correct answers = `CONFIRMED_IMPERSONATION`)
- [ ] Implement temporal risk score: 60-second sliding window EWMA over face-absent duration, gaze deviation frequency, speech duration, OS integrity events. Output 0-100 score per terminal
- [ ] Build dashboard risk visualization: per-terminal heat bar (green→yellow→red) showing live risk score, plus a 10-minute timeline with colored segments and clickable event markers with evidence thumbnails
- [ ] Implement head-pitch-down posture proxy: sustained pitch >25° for >3s with low face motion → `SUSPICIOUS_POSTURE` episode. Frame as behavioral indicator, not object detection

**Phase 3: Nice-to-Have (Day 4-5) — Only if Phase 1-2 are solid**
- [ ] Prototype sparse-landmark head pose (solvePnP on BlazeFace 5 keypoints) in ≤2 hours. If stable yaw/pitch, replace Face Mesh and reclaim ~10ms/frame. If not, keep Face Mesh
- [ ] Implement active liveness challenge on suspicion: when face mismatch enters uncertain zone OR risk score >70, prompt "Please turn your head left then right." Verify via head-pose change >30°. On-suspicion only, not continuous
- [ ] (Only if CPU headroom confirmed on target hardware) Add conditional MediaPipe Hands: trigger only when head pitch >15° sustained. Detect hand-at-ear (phone) or hand-in-holding-pose. Run at reduced cadence (every 3rd frame)

**Phase 4: HARD CUTOFF — No new features after Day 5-6**

**Phase 5: Integration, Testing & Demo Prep (Days 6-8+)**
- [ ] Build scenario coverage matrix: list all 10 likely staged scenarios and map each to detection modality. Identify and address any scenario with zero coverage
- [ ] Run full 10-scenario sequence at least 5 times end-to-end. Time every alert. Record detection rates
- [ ] Build "demo mode" sensitivity slider that scales all thresholds ±20% for on-site calibration
- [ ] Test audio pipeline in a room with similar acoustics to demo venue (or disable common-mode rejection for demo if untested)
- [ ] Test ground-truth logging workflow end-to-end to verify timestamp accuracy within ±20s window
- [ ] Script demo scenario order for narrative arc: (1) candidate leaves → ABSENT, (2) impersonation → FACE_MISMATCH, (3) talking → SUSTAINED_SPEECH, (4) phone in lap → SUSPICIOUS_POSTURE, (5) Alt-Tab → APP_FOCUS_LOST, (6) coordinated noise → show common-mode rejection (pre-recorded if risky), (7) face swap + rapid answers → CONFIRMED_IMPERSONATION with exam analytics, (8) climactic multi-signal fusion with rising risk timeline
- [ ] Build live-updating confusion matrix display for the demo
- [ ] Rename all features in judge-friendly language: "Behavioral Intelligence Engine," "Multi-Modal Threat Scoring," "Privacy-First Architecture," "Zero-Upload Security"
- [ ] Prepare one-sentence DPDP slide: "No video ever leaves the exam room. No biometric data stored after the exam."
- [ ] Prepare scalability closing line: "₹30,000 per center. 10,000 centers. Less than ₹1 per student per exam."
- [ ] Rehearse full presentation + live demo minimum 3 times with timing