# Multi-LLM Debate Report

**Models:** Claude Opus, GPT-5.2, Gemini 3 Pro
**Rounds completed:** 3

---
## Round 1

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

This is one of the most thoroughly debated hackathon architectures I've seen, and the prior rounds have clearly sharpened it. The core design is sound — agent-owns-camera, lightweight INT8 ONNX models, edge fusion, SQLite WAL. But there are several implementation-critical issues that could genuinely cost you the win, and some places where the architecture is still over-engineered for a hackathon while simultaneously under-investing in what the jury will actually score.

**What works well:**
- The camera ownership model via headless Python agent + MJPEG is genuinely clever and will solve the Windows webcam exclusivity problem that will trip up other teams. This is a real differentiator.
- The model stack (BlazeFace + MOSSE + MobileFaceNet + PnP + Silero VAD) at ~11MB total is well-chosen for the hardware constraints.
- Bounded queue size 1 with frame dropping is the correct backpressure strategy.
- Phase 0 pre-flight diagnostic is critical and correctly prioritized.
- Cross-terminal seat-swap is a genuine "wow moment" that directly addresses an explicit jury requirement.

**What concerns me deeply:**

**1. ACCURACY — Realistic score is 7/10, not 8/10.**

The architecture has a material detection hole that is larger than acknowledged. Here's my concrete scenario analysis against likely staged events:

| Staged Event | Detection Method | Likely Outcome |
|---|---|---|
| 1. Wrong person sits down | MobileFaceNet embedding mismatch | ✅ DETECTED (high confidence) |
| 2. Seat swap mid-exam | Cross-terminal embedding match | ✅ DETECTED (your best feature) |
| 3. Student looks at neighbor's screen | PnP head pose >30° yaw | ✅ DETECTED if sustained >3s at >35° |
| 4. Student leaves seat | CANDIDATE_ABSENT >5s | ✅ DETECTED |
| 5. Second person leans in | MULTIPLE_PERSONS >5 frames | ✅ DETECTED |
| 6. Student uses phone (held up) | Phone glow heuristic + possible gaze | ⚠️ MAYBE — depends on screen brightness, angle |
| 7. Student uses phone (under desk) | Phone glow below frame | ❌ MISSED — camera doesn't see it |
| 8. Student reads notes on lap | Pitch >30° down heuristic | ⚠️ MAYBE — 10s threshold too long for quick glances |
| 9. Whispered conversation (brief) | Silero VAD <5s | ❌ MISSED — below standalone threshold |
| 10. Student dictates/receives answers | Silero VAD >10s | ✅ DETECTED |

That's 5-6 solid detections, 2 maybes, 2 likely misses. To hit 8/10, you need BOTH maybes to work. That's tight.

The **phone-under-desk** scenario is almost certainly going to be staged — it's the most common real-world cheating method. You have NO coverage for it. The pitch-down heuristic helps with "reading notes on lap" but a 10s threshold means a quick 3-second glance at notes goes undetected.

**The gaze threshold tuning is the single most important variable in your entire system.** Your 30°/30s and 35°/3s and 45°/instant thresholds are reasonable starting points, but PnP from 5-6 BlazeFace keypoints is COARSE. I've worked with PnP head pose from sparse landmarks — you'll get ±5-8° noise on yaw and ±8-10° on pitch, especially with low-quality webcams and varying distances. This means your effective detection boundary is really ~38-43° for the "35° for 3s" rule. Subtle sideways glances at 20-25° of actual head rotation will read as noise.

**2. PERFORMANCE — It will run, but Chrome is your enemy.**

Your RAM math: 1.5GB (Windows) + 200MB (AV) + 500-1500MB (Chrome) + 260MB (Python+ONNX+OpenCV+models) = 2.46-3.46GB on 4GB. The problem is Chrome. A Chrome tab with MJPEG stream, exam UI, timer, and any JavaScript will easily consume 800MB-1.2GB, not the optimistic 500MB. On 4GB with 2GB pagefile, you'll be swapping.

**Critical mitigation the architecture doesn't address:** You should specify Chrome launch flags: `--disable-gpu --disable-extensions --disable-background-networking --js-flags="--max-old-space-size=256"`. Better yet, test with Edge (already on Windows, lower baseline memory than Chrome). Even better: can the exam UI be served as a minimal HTML page with zero JavaScript frameworks? Vanilla JS + SSE for status updates, no bundler, no polyfills.

The per-frame pipeline at ~21ms + overhead should sustain 15+ FPS on an i5 and ~10-12 FPS on Celeron. That's adequate. But the MobileFaceNet verification every 2s at 18ms will cause a visible frame stutter — on Celeron that's ~2 dropped frames every 2 seconds. Not a functional problem, but the MJPEG stream will visibly hiccup in the dashboard. The jury will notice.

**3. INNOVATION — You're under-investing in demo theatrics.**

The architecture is technically sophisticated, but hackathon juries (especially government juries) respond to VISUAL IMPACT more than architectural elegance. Hash-chained audit trails are technically impressive but invisible during a demo. Here's what will actually win:

- **The confusion matrix is EXPLICITLY REQUIRED.** You have a ground truth logger, but I don't see a prominent real-time confusion matrix widget on the dashboard. This should be the CENTERPIECE of your dashboard — a big, beautiful, auto-updating confusion matrix with TP/FP/TN/FN counts, accuracy %, and per-event-type breakdown. The jury member said it in so many words. Make it impossible to miss.

- **The risk score per student is a must-add, not a nice-to-have.** It takes ~20 lines of code (weighted sum of alert severities with time decay), and it gives the dashboard a single sortable column that screams "this terminal needs attention." Color-code it: green/yellow/orange/red. This is trivially implementable and dramatically improves dashboard UX.

- **Post-exam summary report is HIGH impact, LOW effort.** Generate a one-page HTML per student: photo, timeline of alerts with thumbnails, final risk score, exam score. Export as PDF via browser print. ~2-3 hours of work. The jury will see this as "production-ready."

CONCERNS:

1. **Material detection gap is existential for the 8/10 target.** Phone-under-desk and quick note glances are the two most likely staged scenarios that you will miss. The phone glow heuristic requires the phone screen to be visible to the webcam, which it won't be if held below desk level. You need a fallback — even a simple "candidate's face disappeared from expected region but no CANDIDATE_ABSENT (partial occlusion)" detector would help.

2. **PnP noise from BlazeFace's 6 keypoints will produce false positives OR require thresholds so high you miss subtle cheating.** You haven't budgeted time for per-webcam calibration or threshold tuning. The 30°/35°/45° numbers are theoretical. On a cheap webcam at 640x480 with compression artifacts, BlazeFace keypoint jitter will be significant. You need a calibration step in enrollment: "look straight at camera... now look left... now look right" to establish per-user baseline, or at minimum a running median filter on pose angles (window of 5-7 frames).

3. **The 3-thread architecture has a subtle race condition risk.** Thread 1 (capture) puts frames in a bounded queue of size 1. Thread 2 (inference) reads frames. If inference takes 25ms on a Celeron frame, and capture runs at 30fps (33ms), you're dropping every other frame. That's fine. But if inference occasionally spikes to 50ms+ (GC pause, Windows Defender scan, thermal throttling), you could drop 2-3 consecutive frames. A 2-second malpractice event at 12fps is ~24 frames — missing 3 consecutive is tolerable. But if the spike is longer (antivirus full-scan), you could miss a 5-frame MULTIPLE_PERSONS event entirely. **Add a "frames_dropped_consecutive" counter and alert if it exceeds a threshold.**

4. **Enrollment without liveness is a demo risk.** If a jury member holds a phone photo in front of the webcam during enrollment, BlazeFace will detect the face, MobileFaceNet will create an embedding, and you've enrolled a photo. Later, the real person sits down and gets flagged as FACE_MISMATCH. This is a demo-day embarrassment scenario. **Blink detection via eye aspect ratio from BlazeFace keypoints is ~30 lines of code and 2 hours of work. Do it.**

5. **SQLite WAL on Windows with antivirus is a known pain point.** Windows Defender real-time scanning of .db files can cause write latency spikes of 100-500ms. Pre-exclude the data directory in Windows Defender (add to Phase 0 diagnostic checklist) or use an in-memory SQLite with periodic flush to disk.

6. **No fallback if webcam quality varies wildly across 30 terminals.** Government desktops may have webcams ranging from 0.3MP to 2MP, some with auto-exposure that causes brightness oscillation. CLAHE helps but doesn't solve everything. BlazeFace's detection rate drops significantly below 320x240 effective face resolution. What's the minimum webcam resolution you've tested against?

7. **The "golden demo" fallback video (consensus point 14) needs more specificity.** What exactly is the fallback? Pre-recorded agent output fed to the edge server? A video playing in the dashboard? This needs to be a concrete implementation plan, not a vague insurance policy. If 5 of 30 terminals fail on demo day, you need to show the system working with the other 25, not switch to a video.

8. **Store-and-forward in P2 is too late.** On 4G connectivity with 30 terminals, you WILL get packet loss and connection drops. If a terminal loses connection to the edge server for 10 seconds during a staged malpractice event, the event is lost. A simple 100-line circular buffer that retains the last 60 seconds of events and replays on reconnection should be P1.

SUGGESTIONS:

1. **Add "face position anomaly" as a material detection supplement (~3 hours).** During normal exam-taking, the face bounding box center stays within a predictable region. If the face drops significantly in the frame (looking down at lap) or shifts laterally (reaching for something), flag it. This is just bounding box statistics — zero model cost. Track a 30-second rolling mean of face center Y-coordinate; if current Y deviates by >20% of face height for >3s, trigger SUSPICIOUS_POSTURE. This catches "reading notes on lap" even without accurate pitch estimation.

2. **Add enrollment liveness via blink detection (P1, ~2 hours).** Use BlazeFace's eye keypoints to compute eye aspect ratio (EAR). Require 2 blinks during enrollment. This is ~30-50 lines of code, no new model, and prevents the photo-attack demo embarrassment.

3. **Add a running median filter on PnP angles (~1 hour).** Window of 5 frames. This eliminates single-frame noise spikes that would cause false gaze alerts or mask real deviations. Critical for accuracy.

4. **Move store-and-forward to P1 (~3 hours).** Simple: append events to a local deque (maxlen=200). On successful POST to edge, pop. On failure, retain and retry on next cycle. This is trivial to implement and prevents data loss on 4G.

5. **Build the confusion matrix dashboard widget FIRST in P3 (~2 hours).** Make it the largest element on the dashboard. Auto-updating via SSE. Show per-event-type breakdown. This is what the jury explicitly asked for.

6. **Add per-student risk score (~1 hour).** `risk = sum(alert.severity * alert.confidence * time_decay(alert.age))`. Display as color-coded bar on dashboard. Sort students by risk. Trivial, high impact.

7. **Post-exam one-page report (~2-3 hours).** HTML template with: student photo, enrollment photo, risk timeline (SVG sparkline), list of alerts with thumbnail snapshots, final risk score, exam score. Print to PDF. Hand this to the jury physically after the demo.

8. **Reduce SUSPICIOUS_POSTURE pitch threshold from 10s to 5s when fused with face-position-drop.** The 10s threshold for "looking down" is too generous. Students reading notes on their lap do it in 3-5 second bursts. If face Y-position drops AND pitch is elevated, trigger at 5s.

9. **Add webcam resolution check to Phase 0.** If webcam is below 640x480, warn. If below 320x240, fail. BlazeFace accuracy drops precipitously on tiny faces.

10. **For the demo: prepare a one-page architectural diagram with icons (not boxes-and-arrows) that shows the data flow.** Government juries love professional-looking diagrams. Spend 30 minutes in Figma/Canva. Show: webcam → terminal AI → edge server → dashboard, with model names and latency numbers at each step. Print it poster-size if possible.

KEY_TRADEOFFS:

1. **PnP from sparse keypoints vs. Face Mesh accuracy:** You're trading gaze precision for ~35ms/frame savings. This is the RIGHT tradeoff for Celeron hardware, but it means your gaze detection will be coarser than competitors using Face Mesh on better hardware. Mitigate with the median filter and face-position heuristics.

2. **Heuristic material detection vs. YOLO object detection:** You're trading detection coverage for RAM and latency. This is a RISKY tradeoff — if even one staged scenario is "student uses phone" in a way your heuristics can't see, you lose a point. Other teams using YOLO (even at 5fps) will catch this. Consider: is there a way to run a TINY object detector (NanoDet-Plus, ~1.8MB INT8, ~20ms on i5) at 2fps specifically for the desk region? This would be a P2 item but could be the difference between 7/10 and 9/10.

3. **SQLite vs. something simpler:** SQLite WAL is correct for this use case, but you're adding complexity with the async writer queue and batch flushing. The tradeoff is robustness vs. implementation time. For a hackathon with 30 terminals generating maybe 5 events/minute each, a simple append-to-JSONL with periodic fsync would work fine and be simpler. Keep SQLite for queryability but don't over-engineer the write path.

4. **3 fusion rules vs. more rules:** You have exactly 3 fusion rules. This is the RIGHT constraint — more rules means more tuning, more false positives, and more debugging. But the 3 you have all depend on GAZE_DEVIATION as one input. If gaze detection is noisy (see concern #2), ALL fusion rules degrade together. This is a single point of failure in your accuracy model.

5. **Hackathon scope vs. production scope:** The architecture reads like a production system compressed into a hackathon. Some elements (hash-chained audit trail, DPDP compliance, adaptive FPS degradation) are impressive on paper but consume implementation time that might be better spent on demo polish, threshold tuning, and the confusion matrix display. The jury will spend ~10-15 minutes with your system. Prioritize what they SEE over what they'd AUDIT.

6. **Headless agent vs. Electron/native:** This is absolutely the right call. Electron would add 200MB+ RAM overhead. The MJPEG approach is novel and solves a real problem. The tradeoff is that MJPEG latency (100-200ms) means the webcam feed in the browser will be slightly delayed. This is invisible to the jury unless they look for it, and it's worth the simplicity.

**Final assessment:** This architecture can win, but it's closer to 7/10 accuracy as-is than the claimed 8/10. The two critical additions that could push it to 8-9/10 are: (a) face-position anomaly tracking for material detection, and (b) median filtering on PnP angles. Both are <4 hours total. The biggest demo-day risk is not technical failure but the jury staging a phone-under-desk scenario that you simply cannot detect. Have your presenter acknowledge this limitation proactively ("our system det

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
The overall architecture is *hackathon-viable* and notably more operationally realistic than the common “YOLO + MediaPipe + Streamlit” stack. The agent-owning-webcam + MJPEG pattern, bounded queues, and edge-side fusion are the right direction for Windows labs and weak PCs. That said, your current detector set has two big “jury-staged event” blind spots (materials + subtle consultation) and one operational landmine (Windows/CPU instruction compatibility for ONNX Runtime). As written, you’re more likely at **7/10–8/10** than a confident 8/10, unless you add one or two *low-risk* upgrades that improve coverage without blowing perf.

Where it works:
- **Seat-swap + face mismatch** with edge-wide embedding search is a genuine differentiator and should score well.
- **Event-driven pipeline** (episode alerts, fusion windows, hash-chaining) maps well to the “confusion matrix + evidence” requirement.
- **PnP head pose** is a smart performance tradeoff vs FaceMesh on Celerons.

Where it’s fragile:
- **Material detection via glow + skin blob** will miss a large portion of “phone under desk / notes on lap” when screen brightness is low, phone is angled, or notes are matte/no glow.
- **PnP from 5–6 keypoints** cannot catch “eyes-only glance” behavior; it mainly catches *head turns*. You’ll detect exaggerated gaze deviation, not subtle.
- **Audio thresholds** (10s standalone, 5s+gaze fusion) will miss the most common staged “quick whisper” unless the jury plays fair and makes it obvious.
- **No liveness in enrollment** invites the easiest possible spoof: photo/video held to camera during enrollment, then seat-swap later.

Net assessment: with your current list, the most likely outcome in a staged evaluation is **7/10** if they include at least 2 “materials” events and 1 “quick whisper” event. If the staged scenarios are more obvious (overt head turn + talking), you can hit **8/10**.

CONCERNS:
1. **Materials (phone/notes) detection is underpowered**: glow/skin heuristics are high-variance across lighting, camera exposure, desk color, and phone brightness. Expect to miss **30–60%** of “phone under desk” and **50–80%** of “notes on lap” unless the notes/phone are clearly visible.
2. **Gaze is effectively “head turn detection”**: with sparse landmarks, you will miss “side-eye while head forward.” A jury can easily stage that.
3. **Frame-based thresholds + queue size 1 can distort time logic**: if you use “5 consecutive frames” anywhere, FPS drops will change behavior. All persistence rules must be **time-based**, not frame-count-based, or you’ll create false negatives/positives under load.
4. **ONNX Runtime CPU instruction set risk**: some Celerons lack AVX/AVX2. If you bring the wrong ORT wheel, models will fail to load on demo day. This is a real derail risk.
5. **Windows webcam/audio device contention**: Chrome, other apps, privacy permissions, or driver quirks can break exclusive access. If your “agent owns webcam” fails on even 5/30 machines, you lose credibility fast.
6. **No store-and-forward until P2** is risky for a 4G baseline. A brief LAN hiccup could erase evidence/alerts during the exact staged malpractice window.
7. **MobileFaceNet verification every 2s may be too sparse** during fast seat-swap / brief impersonation events. If someone swaps for ~3–5 seconds, you could miss it depending on timing.
8. **CLAHE every frame is wasted cost and can introduce artifacts** that hurt face detection/verification in some lighting (over-amplifies noise, changes embedding distribution).

SUGGESTIONS:
1. **Add enrollment liveness (active, head-pose challenge) — 3–6 hours**
   - Don’t do blink EAR (you don’t have eyelid landmarks).
   - Do a scripted liveness prompt during enrollment: “Turn head LEFT → RIGHT → UP → center” within 5–8 seconds.
   - Verify the prompt via your existing **PnP yaw/pitch** trajectory + require embedding stability across frames.
   - This blocks static photo spoofing and is easy to demo (“liveness passed” badge).

2. **Upgrade materials detection without YOLO: “desk ROI change detector” — 4–8 hours**
   - Maintain a running background model (simple median / MOG2) *only for the desk ROI* (lower third).
   - Trigger POSSIBLE_MATERIAL_USE when a **new persistent object** appears in desk ROI + skin proximity (hand) + low motion afterwards (reading).
   - This catches “notes on lap/desk” even without glow and is still lightweight. It’s less sexy than YOLO, but much more reliable than glow alone.

3. **Make all persistence rules time-based — 1–2 hours**
   - Replace “5 consecutive frames” with “≥0.4s” etc, using timestamps.
   - This directly mitigates the bounded-queue/frame-drop behavior and makes performance degradation predictable.

4. **Increase face verification cadence *on suspicious triggers* — 2–3 hours**
   - Keep 2s normally, but if any of these occur: face lost/reacquired, gaze deviation starts, multiple persons event, or sudden scale change → run verification immediately (rate-limit to e.g. once per 400–600ms).
   - This improves detection of short impersonation/seat-swap stunts without constant cost.

5. **Change CLAHE policy to conditional — 1–2 hours**
   - Only apply CLAHE when measured brightness/contrast falls below threshold, and apply on downscaled grayscale used for detection (not on the full MJPEG frame).
   - This saves CPU and reduces embedding drift.

6. **Promote store-and-forward to P1 (minimal version) — 3–5 hours**
   - You don’t need full P2 reliability. Just buffer events locally in a JSONL queue (and optionally 1–2 evidence snapshots) and retry POST until ACK.
   - This prevents “network blip during staged malpractice” from destroying your score.

7. **Operational hardening for Windows/CPU — 2–4 hours**
   - Ship/test an ORT build that works on non-AVX machines (or include fallback).
   - Add Phase-0 checks: “ORT session load ok”, “webcam open ok”, “audio capture ok”, “fps baseline ok”, “disk write ok”.
   - Log a single green/red readiness page for invigilator/jury.

8. **Add “risk score + timeline” to dashboard (high impact, low risk) — 4–6 hours**
   - Weighted score with decay (e.g., exponential) + color-coded sparkline per student.
   - Jury impact: they instantly see “top 3 risky candidates” and can click to replay evidence.

9. **Evidence presentation: annotated incident card — 4–8 hours**
   - For each incident: snapshot with overlays (face bbox, yaw/pitch numbers + arrow, desk ROI box, audio VAD bar).
   - Include “confidence + rule fired + fusion explanation” (“GAZE + AUDIO within 5s → POSSIBLE_CONSULTATION”).
   - This materially improves perceived accuracy, even if the raw detector is imperfect.

KEY_TRADEOFFS:
- **You traded “broad object coverage” (YOLO) for CPU safety and reliability.** That’s the right instinct on Celeron-class machines, but the *current* heuristic-only materials detection is too weak for staged evaluation. A lightweight ROI change detector is the right middle ground.
- **You traded “true gaze” for head pose.** Correct for performance, but it means you must frame the feature honestly in the demo: call it “head orientation away from screen” and rely on fusion with audio/posture/material cues.
- **You traded simplicity on terminals for complexity on edge (fusion, cross-terminal matching).** That’s good: terminals stay light, and the edge creates the “innovation” story. But it increases dependency on network/time sync—hence store-and-forward should come earlier.
- **Bounded queue + frame dropping favors responsiveness over completeness.** Acceptable for hackathon, but only if all thresholds are time-based and you add “verify-on-suspicion” to avoid missing short critical events.

If you implement only two changes before coding deepens: **(1) active liveness at enrollment** and **(2) desk ROI change-based material detection**, you significantly increase your odds of a real **8/10** without blowing your performance budget.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The architecture is pragmatic and well-tailored to the severe hardware constraints (Celeron/4GB), but it holds two critical weaknesses that jeopardize the "8/10 accuracy" and "win" conditions:

1.  **The "Material Detection" Fallacy:** Relying on skin-tone blobs and screen glow is technically unsound for a hackathon environment. A "skin-tone blob" in the desk ROI is indistinguishable from a student resting their hands on the table or crossing their arms. This will cause high False Positives (FP), irritating the jury. Conversely, screen glow detection fails in well-lit rooms. You cannot rely on these heuristics to catch "phone under desk."
2.  **Memory Contention on 4GB:** You are underestimating the volatility of Windows 10 memory management. If Chrome takes 1.5GB and the OS takes 1.5GB, you have <1GB left. If the system swaps to disk (likely on a Celeron during a browser refresh), your Python agent's frame processing loop will stall, causing frame drops and missing the "short whisper" or "quick glance" malpractice events.
3.  **PnP Stability:** 6-point PnP is extremely jittery without temporal smoothing. Raw output will result in a gaze vector that shakes violently, triggering false GAZE_DEVIATION alerts.

However, the **Agent-as-owner MJPEG architecture** and **Cross-terminal seat swap** are winning differentiators that must be preserved.

CONCERNS:
1.  **High False Positive Rate on Material Heuristics:** The "skin-tone blob" logic will flag legitimate hand placements. This destroys trust in the system.
2.  **PnP Signal Noise:** Without a filter (e.g., OneEuroFilter), the gaze vector will be too noisy to reliably trigger the 30° threshold.
3.  **Audio Fusion Latency:** "Common-mode audio" requires time-alignment across distributed nodes. Network jitter on 4G could make 3-terminal correlation fail or lag significantly.
4.  **Visual Evidence Gap:** The dashboard lists text logs. Juries ignore text. They need to *see* the cheating.
5.  **Enrollment Spoofing:** A photo held up to the camera will pass Phase 0 enrollment, embarrassing the team if a jury member tests it.

SUGGESTIONS:
1.  **Pivot Material Detection to "Behavioral Proxy":** Drop the "skin-tone blob" heuristic immediately. It is garbage. Instead, tighten the **SUSPICIOUS_POSTURE** rule. If `Pitch > 25° down` AND `Eyes not visible` (or `Gaze down`), flag as POSSIBLE_MATERIAL_USE. You cannot detect the object, so detect the *act of using it*. This is more robust and defensible.
2.  **Add "Blink Liveness" (EAR):** Implement Eye Aspect Ratio (EAR) using the existing BlazeFace landmarks. Require 1 blink during enrollment. It costs ~0ms (math only) and prevents the "photo spoof" attack.
3.  **Implement OneEuroFilter for Gaze:** You must smooth the PnP output. Use a simple OneEuroFilter (10 lines of code) to stabilize the head pose vector.
4.  **"Live Heatmap" Dashboard:** The jury needs a visual hook. Add a grid view of the 30 seats to the dashboard. Color-code seats (Green/Orange/Red) based on a **composite Risk Score**. This answers the "Novel Visualization" question.
5.  **Aggressive Chrome Taming:** Launch Chrome with flags `--renderer-process-limit=2` and `--disable-site-isolation-trials` to reduce RAM footprint.
6.  **Risk Score Logic:** Yes, implement this. `Score = (Gaze_Count * 1) + (Audio_Count * 2) + (Posture_Count * 3) + (Face_Mismatch * 10)`. Display this prominently.

KEY_TRADEOFFS:
*   **Object Detection vs. Heuristics:** You are trading the ability to definitely say "That is a phone" (YOLO) for system stability (Heuristics). This is the **right tradeoff** for Celeron hardware, *provided* you shift to behavioral detection (posture) rather than feature detection (skin blob).
*   **Local UI vs. Electron:** Using the system browser instead of bundling Electron saves ~300MB RAM but risks browser version inconsistencies. This is the **right tradeoff** for the 4GB constraint.
*   **PnP vs. Mesh:** PnP is less accurate for gaze but 10x faster. **Right tradeoff**, provided you apply smoothing.

---
## Round 2

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After reviewing GPT-5.2 and Gemini 3 Pro's feedback, I'm struck by how much convergence there is across all three analyses. We all independently arrived at the same core diagnosis: the architecture is structurally sound but has two accuracy-threatening gaps (material detection and gaze noise) and one operational landmine (memory on 4GB). The disagreements are mostly about *remedies*, not *diagnosis*. Let me synthesize.

**Where all three of us agree (strong consensus):**

1. **Material detection via glow + skin-tone is the #1 accuracy risk.** GPT-5.2 estimates 30-60% miss rate on phone-under-desk, Gemini calls skin-tone blobs "garbage" for false-positive reasons, and I flagged it as existential. We're unanimous: the current heuristics are insufficient AND potentially harmful (false positives from hands on desk).

2. **PnP from 6 keypoints needs temporal smoothing.** Gemini specifically calls for OneEuroFilter, GPT-5.2 frames it as "head turn detection, not gaze," and I called for a median filter. The specific filter matters less than the consensus that raw PnP output will be too noisy to use reliably.

3. **Store-and-forward must move to P1.** Both GPT-5.2 and I flagged this independently. A network blip during a staged event loses you a point.

4. **Enrollment liveness is needed.** All three of us flagged this. The disagreement is on method — I'll address this below.

5. **Risk score + visual dashboard is critical for jury impact.** All three independently suggested this.

6. **Realistic accuracy is 7/10 as-is**, needing targeted improvements to reach 8/10.

**Where I've updated my position based on others' feedback:**

- **Gemini is right that skin-tone blob should be dropped entirely, not just supplemented.** My original suggestion was to ADD face-position anomaly tracking. But Gemini's point about false positives from normal hand placement is valid — the skin-tone heuristic will actively HURT your score by generating false alarms that erode jury trust. I now agree: drop skin-tone blob, keep phone glow (low FP risk since you're looking for bright rectangles specifically), and pivot to behavioral proxies (posture + face position change).

- **GPT-5.2's point about frame-count vs. time-based thresholds is critical and I missed it.** "5 consecutive frames" means different things at 15fps vs. 8fps. Under load, frame drops change the effective duration window. ALL persistence rules must be time-based. This is a 1-2 hour fix that prevents a class of subtle bugs.

- **GPT-5.2's suggestion of adaptive verification cadence (verify-on-suspicion) is excellent.** Keep 2s baseline, but trigger immediate verification when gaze deviates, face is lost/reacquired, or multiple persons appear. This catches short seat-swap stunts without constant 18ms overhead. I hadn't considered this and it directly addresses the "3-5 second impersonation" gap.

- **GPT-5.2's concern about CLAHE artifacts affecting embeddings is valid.** If CLAHE changes pixel distribution in the face region, MobileFaceNet embeddings may drift from enrollment. The fix is simple: apply CLAHE only for detection (BlazeFace), but crop the original (non-CLAHE) frame for MobileFaceNet embedding extraction. This is a ~5-line change.

**Where I disagree with other participants:**

- **Gemini's suggestion to use EAR for blink detection with BlazeFace keypoints.** BlazeFace provides 6 keypoints: 2 eyes (centers), nose, mouth center, 2 ear tragions. These are NOT eyelid landmarks — they're eye center points. You cannot compute Eye Aspect Ratio without upper/lower eyelid landmarks. GPT-5.2 correctly identified this and proposed active head-pose liveness instead ("turn left, right, up, center"). **GPT-5.2's approach is correct.** The head-pose challenge uses your existing PnP pipeline, requires no new landmarks, and is actually MORE robust than blink detection against video replay attacks. Implementation: ~3-4 hours, not 2, because you need the UI prompt flow and PnP trajectory validation.

- **Gemini's specific risk score formula (`Gaze*1 + Audio*2 + Posture*3 + FaceMismatch*10`) is too rigid.** Hard-coded integer weights will behave unpredictably. I prefer a normalized approach: each alert type contributes a 0-1 severity score, weights are in config.yaml, and there's exponential time decay so old alerts don't permanently stigmatize a student. The formula should be tunable, not baked in.

- **GPT-5.2's desk ROI background subtraction idea is interesting but risky for a hackathon.** MOG2 on a desk ROI will trigger on: papers being shuffled, pencils being moved, hands moving, shadows from overhead lighting, other students walking past. The tuning required to distinguish "new persistent object appeared" from "student moved their answer sheet" is significant. I'd rather invest those 4-8 hours in perfecting the behavioral proxy (face position drop + pitch) which has fewer false positive modes.

**The material detection strategy — my revised recommendation:**

After considering all three perspectives, here's the synthesis:

1. **Keep phone glow detection** (bright rectangular region) — low FP, catches phones held up or at desk level with screen on
2. **Drop skin-tone blob** — too many false positives from normal hand positions (Gemini is right)
3. **Add face bounding-box position tracking** (my original suggestion) — face Y-drop as behavioral proxy for "looking at lap," zero model cost
4. **Tighten SUSPICIOUS_POSTURE to 5s** when face-Y-drop corroborates pitch estimate (reducing from 10s)
5. **Do NOT add desk ROI background subtraction** — too much tuning risk for hackathon timeline
6. **Consider NanoDet-Plus as a P3/P4 stretch goal**, not P1/P2 — only if everything else works

This combination gives you: phone-held-up (glow), phone-at-desk-screen-on (glow), reading notes on lap (face drop + pitch), looking down at phone under desk (face drop + pitch). You still miss phone-under-desk-with-head-barely-tilted, but that's a scenario most systems miss.

**The single highest-risk component:**

All three of us danced around this, but let me state it directly: **the highest risk is not any single component — it's the 30-terminal scale-up on demo day.** You'll develop and test on 1-3 terminals. Going from 3 to 30 simultaneously introduces: edge server load (30 concurrent SSE connections + event ingestion), cross-terminal embedding search (30x30 = 900 comparisons on FACE_MISMATCH), network congestion (30 MJPEG streams if dashboard shows all), and the probability that at least 2-3 terminals will have hardware quirks (bad webcams, different Windows builds, missing DLLs). **Your Phase 2 "3-terminal vertical slice" milestone is the most important gate in the entire project.** If it doesn't work cleanly at 3, it won't work at 30.

CONCERNS:

1. **Material detection (revised): Even with behavioral proxies, phone-under-desk with minimal head tilt remains a gap.** All three reviewers agree this is the hardest scenario. Mitigation: brief the presenter to proactively acknowledge this and frame your approach as "behavioral analysis" rather than "object detection" — position it as a deliberate architectural choice for privacy (no constant desk surveillance) rather than a limitation.

2. **ONNX Runtime CPU instruction set compatibility (GPT-5.2 raised this, I missed it).** This is a genuine demo-day killer. Some Celeron N-series chips lack AVX2. If you bring an ORT build that requires AVX2, models won't load. **Phase 0 must test `ort.InferenceSession` load for every model file, not just check RAM.** Use `onnxruntime-openvino` or the default CPU EP with `--x86` target as fallback. Test on the actual demo hardware during setup.

3. **Cross-terminal embedding search at 30 terminals.** On FACE_MISMATCH, you compare one 512D embedding against 30 enrolled embeddings. That's 30 cosine similarities — trivial (~0.1ms). But if multiple terminals fire FACE_MISMATCH simultaneously (e.g., during a staged seat-swap involving 4+ people), the edge server handles it fine. **However**: what if enrollment embeddings are stored only in edge server memory and the edge server restarts? You need embeddings persisted to disk (SQLite or JSONL). Verify this is in the design.

4. **Common-mode audio classification across terminals (GPT-5.2 flagged time-alignment).** For "≥3 terminals with audio = ROOM_NOISE", you need event timestamps aligned to within ~1-2 seconds. Your RTT/2 time sync should handle this, but on 4G with variable latency, you could get 500ms-2s clock drift between terminals. **For the hackathon, all terminals are on the same LAN, so this is actually fine.** But document the assumption.

5. **Chrome memory on demo day is still the scariest operational risk.** GPT-5.2's Chrome flags and Gemini's `--renderer-process-limit=2` are both good, but the real mitigation is: **use Edge.** It's pre-installed on Windows 10/11, uses the same Chromium engine, and has ~100-200MB lower baseline memory. Switch the documentation and deployment scripts to target Edge. If Edge isn't available, fall back to Chrome with the flags.

6. **GPT-5.2's concern about CLAHE + embedding drift is valid and easy to fix.** Use CLAHE-processed frames for BlazeFace detection, but extract the face crop from the ORIGINAL frame for MobileFaceNet. This is ~5 lines of code but prevents a subtle accuracy degradation.

SUGGESTIONS:

1. **Implement active head-pose liveness at enrollment (P1, ~3-4 hours).** Prompt: "Please look LEFT... RIGHT... CENTER." Validate via PnP yaw trajectory. Require yaw to exceed ±20° in both directions. This uses your existing pipeline, blocks photo/video spoofing, and is more robust than blink detection (which you can't do with BlazeFace's 6 keypoints anyway). **This is the #1 consensus item across all three reviewers.**

2. **Drop skin-tone blob, keep phone glow, add face-position-drop behavioral proxy (P1, ~3 hours).** Track rolling mean of face bounding box center-Y. Trigger SUSPICIOUS_POSTURE when Y-drop exceeds 15-20% of face height for >5s (when fused with pitch) or >8s standalone. This replaces the unreliable skin-tone heuristic with a zero-cost behavioral signal.

3. **Add OneEuroFilter or exponential moving average on PnP angles (P1, ~1 hour).** OneEuroFilter is better than median for this use case — it's designed for noisy real-time signals and adapts to velocity (fast head turns pass through, slow jitter gets smoothed). It's 30 lines of Python. Gemini's specific recommendation is correct here.

4. **Convert ALL frame-count thresholds to time-based (P1, ~1-2 hours).** "5 consecutive frames" → "≥400ms". "3+ checks" → "≥6s" (at 2s check interval). This prevents FPS-dependent behavior changes under load.

5. **Add adaptive verification cadence (P1, ~2-3 hours).** On gaze deviation onset, face loss/reacquire, or MULTIPLE_PERSONS, trigger immediate MobileFaceNet verification (rate-limited to 1 per 500ms). This catches 3-5 second impersonation stunts. GPT-5.2's suggestion, which I now strongly endorse.

6. **Move store-and-forward to P1 (P1, ~2-3 hours).** Local deque of unsent events, retry on reconnection. All three reviewers agree on this.

7. **Build the dashboard around THREE visual elements (P3, ~6-8 hours total):**
   - (a) Live seat-map grid (30 cells, color-coded green/yellow/orange/red by risk score) — the jury's "at a glance" view
   - (b) Real-time confusion matrix (auto-updated against ground truth logger) — the jury's explicit requirement
   - (c) Incident cards with annotated snapshots (face bbox, pose arrow overlay, confidence %, fusion explanation) — the evidence view
   
   Skip text logs on the main screen. Everything the jury sees should be visual.

8. **Use CLAHE only for detection, extract embeddings from original frame (P1, ~15 minutes).** Trivial but important for verification accuracy consistency.

9. **Target Edge browser, not Chrome (P0, ~30 minutes).** Change deployment scripts and documentation. Test on Edge. Fall back to Chrome with `--disable-gpu --disable-extensions --renderer-process-limit=2 --disable-site-isolation-trials`.

10. **Phase 0 must include ORT model load test (P0, ~1 hour).** Don't just check RAM. Actually load each ONNX model into an InferenceSession and run a dummy inference. If it fails (AVX2 missing, DLL conflict), fail the pre-flight with a specific error message. Ship a fallback ORT build without AVX2 dependency.

KEY_TRADEOFFS:

1. **Active head-pose liveness vs. blink EAR vs. no liveness.** Head-pose liveness costs ~3-4 hours and uses existing PnP pipeline. Blink EAR is infeasible with BlazeFace's 6 keypoints (Gemini's suggestion doesn't work — this is the one genuine technical error across all reviews). No liveness risks demo-day embarrassment. **Head-pose liveness is the right answer — all cost, no new dependencies, high demo impact.**

2. **Behavioral proxy (face-position-drop) vs. desk ROI background subtraction vs. NanoDet.** Face-position-drop costs ~3 hours, has low false-positive risk, and catches the "looking at lap" posture. Desk ROI MOG2 costs 4-8 hours and has significant tuning risk. NanoDet costs 4-6 hours and adds a model + RAM. **Face-position-drop is the right hackathon choice.** Desk ROI and NanoDet are stretch goals only.

3. **OneEuroFilter vs. median filter vs. no smoothing.** OneEuroFilter is better than median for real-time pose — it's velocity-adaptive, so genuine fast head turns aren't dampened while jitter is. Implementation cost is identical (~30 lines). **OneEuroFilter wins.** Gemini's specific recommendation over my original median suggestion.

4. **Edge vs. Chrome.** Edge saves ~100-200MB RAM, is pre-installed on government Windows machines, and uses the same rendering engine. The only risk is if specific government machines have very old Edge versions (pre-Chromium, before Jan 2020). Phase 0 should check Edge version. **Edge is the right default.**

5. **The fundamental accuracy vs. scope tradeoff remains:** You can either have 6 detectors that each work at 85% reliability (current approach), or 4 detectors that each work at 95% reliability (Consensus point #11: "4 rock-solid detectors > 6 flaky ones"). After this three-way review, I believe the right set is: **CANDIDATE_ABSENT (rock solid), FACE_MISMATCH + SEAT_SWAP (rock solid), GAZE_DEVIATION (solid with smoothing), SUSPICIOUS_AUDIO (solid for sustained speech), MULTIPLE_PERSONS (rock solid)**. That's 5 rock-solid detectors covering scenarios 1-5 and 10 from my table. SUSPICIOUS_POSTURE (face-drop + pitch) and POSSIBLE_MATERIAL_USE (phone glow) are bonus coverage for scenarios 6 and 8 — helpful but not the foundation. This framing means you're confident on 6-7/10 with a realistic shot at 8-9/10 depending on what's staged.

**Convergence summary across all three reviewers:**
- Drop skin-tone blob: unanimous
- Add temporal smoothing to PnP: unanimous  
- Add enrollment liveness: unanimous (method differs — head-pose wins on technical grounds)
- Store-and-forward to P1: unanimous
- Time-based thresholds: GPT-5.2 raised, I endorse, Gemini didn't address but wouldn't disagree
- Risk score + visual dashboard: unanimous
- Realistic accuracy 7/10 without changes

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position is mostly unchanged: the *platform architecture* (agent owns camera, bounded queues, lightweight ONNX, edge fusion, evidence logs) is strong and should outperform typical hackathon stacks on constrained Windows desktops. What changed after reading Claude/Gemini is that I’m **less willing to accept the current “8/10 with <5% FP” expectation as-is**. The consensus across all feedback is that **materials + subtle consultation** are the dominant miss cases, and **PnP jitter + weak heuristics** can either create false positives or force thresholds so high you miss events.

Where I converge strongly with others:
- **Materials detection is the existential gap** for hitting 8/10 if the jury stages phone-under-desk / notes-on-lap (they likely will).
- **PnP head pose must be temporally smoothed** (median/OneEuro). Raw PnP will jitter and destabilize gaze + all fusion rules.
- **Enrollment needs liveness** to avoid an embarrassing “photo enroll” demo failure.
- **Chrome/Windows memory volatility is real**; you need a “tamed browser” plan and Phase-0 checks.
- **Demo-visible UX matters**: risk score grid, big confusion matrix, annotated evidence cards and post-exam report will score better than invisible crypto/audit features.

Where I disagree (or at least would reframe):
- **“Blink EAR from BlazeFace keypoints” is not guaranteed feasible/reliable.** BlazeFace typically provides sparse landmarks (often eye centers, nose, mouth corners). EAR needs eyelid landmarks. If you don’t truly have eyelid points, “blink detection” becomes noisy theater. I’d prioritize **active head-pose challenge liveness** using your existing PnP, which you *know* you can measure.
- **Common-mode audio time alignment** is less scary than Gemini implies *if terminals → edge are on LAN* (as your design says). Still, implement correlation with a generous window and rely on VAD episodes, not precise phase alignment.

Net: architecture is good, but **accuracy target is not safe without 2–3 specific upgrades**. With those upgrades, 8/10 becomes realistic.

CONCERNS:
1. **Materials detection will miss staged “phone under desk”** (camera can’t see it) and will be inconsistent for “notes on lap.” This is your #1 accuracy risk.
2. **Skin-tone blob heuristic likely drives false positives** (hands on desk, face rubbing, crossing arms). I agree with Gemini: as stated, it’s not defensible unless heavily gated.
3. **PnP head pose jitter** from sparse keypoints will cause either false GAZE_DEVIATION or require thresholds so high you miss subtle events; it also cascades into all fusion rules because gaze is a shared dependency.
4. **Time logic coupled to FPS**: any “N consecutive frames” thresholds will break under frame drops. Persistence must be time-based everywhere.
5. **ONNX Runtime CPU instruction-set mismatch (AVX/AVX2)** remains a demo-derailer on unknown Celeron variants.
6. **Windows Defender + SQLite WAL write spikes** can create intermittent latency (and “it worked yesterday” debugging pain).
7. **No store-and-forward until P2** risks losing the exact staged event window due to transient network/edge restarts.
8. **Browser RAM footprint variance** can trigger swapping and stall your inference thread long enough to miss short events; “Chrome is your enemy” is accurate.
9. **Webcam variability across 30 seats** (resolution/exposure/driver) can crater BlazeFace stability unless you hard-check and degrade gracefully.

SUGGESTIONS:
1. **Add temporal smoothing to PnP immediately (P1, 1 hour)**
   - Median filter (window 5–7) or OneEuroFilter on yaw/pitch.
   - Then base gaze deviation on the filtered signal only.

2. **Replace/retire “skin-tone blob” unless gated (P1, 1–2 hours)**
   - If kept: require *persistence + ROI change + hand-motion pattern*, not “skin present.”
   - Better: shift materials detection to **behavioral proxy** + ROI-change (below).

3. **Add “desk ROI persistent-change detector” (P1, 4–8 hours)**
   - Background model (median/MOG2) only for desk/lap region.
   - Trigger when a new object persists with low motion (reading) and/or coincides with downward posture.

4. **Add “face position anomaly” / “face drop” heuristic (P1, 2–3 hours)**
   - Track rolling baseline of face bbox center Y and scale.
   - If face center drops significantly for >3–5s (tunable) → SUSPICIOUS_POSTURE / POSSIBLE_MATERIAL_USE (fusion with gaze-down/pitch).

5. **Enrollment liveness: active head-pose challenge (P1, 3–6 hours)**
   - Prompt “look left/right/up/center” and verify via filtered PnP trajectory.
   - This is robust with your current sensors and prevents the simplest spoof.
   - Only do “blink detection” if you confirm you have eyelid landmarks (don’t assume).

6. **Make *all* thresholds time-based (P0/P1, 1–2 hours)**
   - Replace “5 frames” with “≥400ms,” etc. This directly addresses bounded-queue drops and CPU stalls.

7. **Promote minimal store-and-forward to P1 (3–5 hours)**
   - Local JSONL queue + retry until ACK; optionally keep last 1–2 thumbnails per incident.

8. **Demo-impact features to prioritize (P3 but start early)**
   - **Risk score + 30-seat heatmap grid** (4–6 hours): sortable “who to watch now.”
   - **Big real-time confusion matrix widget** (2–4 hours): make it the dashboard centerpiece.
   - **Annotated incident cards + post-exam one-page report** (6–12 hours combined): “evidence the jury can see.”

9. **Windows hardening tasks (P0, 2–4 hours)**
   - Phase-0 checks: ORT session load, webcam open, audio open, webcam resolution, sustained FPS, disk write latency.
   - Browser launch policy: prefer Edge or launch Chrome with memory-saving flags; disable extensions; kiosk mode.

KEY_TRADEOFFS:
- **PnP vs FaceMesh**: you correctly trade precision for speed, but that forces you to (a) smooth aggressively and (b) frame it as “head orientation,” not eye gaze. This is the right tradeoff for Celeron-class constraints.
- **Heuristics vs object detector**: skipping YOLO is sane for perf/RAM, but you must replace naive heuristics with **behavioral + ROI-change** signals to avoid both misses and false alarms. That middle path is the best hackathon/CPU compromise.
- **Invisible robustness vs visible scoring**: hash-chaining and WAL are good engineering, but the jury scores what they *see*—confusion matrix, risk heatmap, annotated evidence, and a printable report. The tradeoff should tilt toward visible outputs once basic stability is proven.
- **Bounded queue responsiveness vs event completeness**: keep queue size 1, but only if you make persistence time-based and add “verification-on-suspicion” so brief critical events aren’t missed due to timing luck.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
There is now **unanimous consensus** among all three reviewers (myself, Claude, GPT-5.2) on the architecture's two fatal flaws: **Material Detection** and **Memory Safety**. We all agree that the current "skin-tone/glow" heuristic will fail the "phone under desk" test, which is the most likely staged malpractice event. We also agree that Chrome on 4GB RAM is a ticking time bomb.

However, the path to fixing these is clear. We have converged on replacing "object detection" with "behavioral detection" and fortifying the UI/Network layer.

I explicitly **disagree** with GPT-5.2's suggestion for a "Desk ROI change detector" (background subtraction). On a laptop, typing causes screen shake, which will render background subtraction uselessly noisy. Claude's suggestion of "Face Y-axis drop" combined with my "Pitch > 25°" is the superior, vibration-resistant approach.

CONCERNS:
1.  **The "Phone Under Desk" Gap (Consensus):** You currently have 0% coverage for this. If you do not implement the behavioral proxy (looking down), you will score 7/10 at best.
2.  **Chrome Memory Hog (Consensus):** If the browser crashes or swaps, the agent dies. You must aggressively constrain Chrome.
3.  **No Visual Proof (Consensus):** Text logs won't win. The jury needs a Risk Score heatmap and a real-time Confusion Matrix.
4.  **Network Resilience:** Pushing Store-and-Forward to P2 is a mistake. 4G is flaky; losing evidence during a staged hack is fatal.
5.  **Hardware Compatibility:** GPT-5.2's point about AVX instructions on old Celerons is a hidden killer.

SUGGESTIONS:
1.  **Final Material Detection Logic (Behavioral Fusion):**
    *   Discard "Skin-tone blob".
    *   Implement **Combined Posture Heuristic**: Trigger `POSSIBLE_MATERIAL_USE` if `(Pitch > 20° Down)` OR `(Face_Centroid_Y drops > 15% from baseline)`. This captures "slumping to read" and "looking down" without needing to see the phone.
2.  **Liveness via Blink (EAR):** Implement Eye Aspect Ratio from BlazeFace landmarks. Require 1 distinct blink during enrollment. This is faster to implement (~1 hour) than GPT's "Head Turn" challenge and sufficient to block photo spoofs.
3.  **Mandatory Signal Smoothing:** Add **OneEuroFilter** (or at least a median filter) to the PnP output. Raw BlazeFace jitter will destroy your gaze accuracy.
4.  **Visual "Wow" Factors (P1 Priority):**
    *   **Risk Score:** `(Alert_Count * Weight)`. Color-code the dashboard rows.
    *   **Confusion Matrix:** A live, updating table on the dashboard comparing "Detected" vs "Ground Truth".
5.  **Operational Safety:**
    *   **Chrome Flags:** `--renderer-process-limit=2 --disable-site-isolation-trials`.
    *   **AVX Check:** In Phase 0, verify the CPU supports the instruction set required by your ONNX Runtime build (usually AVX2). If not, have a non-AVX wheel ready or fail gracefully.
6.  **Store-and-Forward:** Move the "local JSONL buffer + retry" logic to **Phase 1**.

KEY_TRADEOFFS:
*   **Behavioral vs. Object Detection:** We are trading the specificity of "Phone Detected" for the reliability of "Suspicious Looking Down". This is the **correct tradeoff** for the hardware, as it works even if the phone is hidden.
*   **Smoothing vs. Latency:** Adding a filter adds ~50-100ms lag to gaze detection. This is **acceptable** to prevent false positives from jitter.
*   **Accuracy vs. Stability:** By relying on heuristics for materials, we risk False Positives (looking at keyboard). We mitigate this by labeling the alert `POSSIBLE_MATERIAL_USE` (Medium Confidence) rather than `CONFIRMED`. This manages jury expectations correctly.

---
## Round 3

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After this third round, I'm confident we've reached a stable equilibrium. The three reviewers have converged on virtually every substantive point. My position hasn't materially changed — it's been refined and strengthened by the cross-pollination. Let me formalize what's settled, what's still contested, and what the team should actually do.

**What is now fully settled (no remaining disagreement):**

1. **Drop skin-tone blob.** Unanimous across all three rounds. Zero dissent remaining.
2. **PnP needs OneEuroFilter.** Gemini proposed it, I endorsed it in Round 2, GPT-5.2 accepts median or OneEuro. OneEuroFilter wins on technical merit (velocity-adaptive). Settled.
3. **Store-and-forward moves to P1.** Unanimous. A local deque/JSONL buffer with retry is ~2-3 hours and prevents catastrophic evidence loss.
4. **All thresholds must be time-based, not frame-count-based.** GPT-5.2 originated this, I endorsed it, Gemini didn't contest. Settled.
5. **Phase 0 must include ORT model load + dummy inference test.** AVX2 compatibility is a real risk on Celeron hardware. Settled.
6. **Dashboard needs: seat-map risk heatmap, live confusion matrix, annotated incident cards.** All three reviewers independently converged on these three visual elements. Settled.
7. **Behavioral proxy (face Y-drop + pitch) is the right material detection approach for hackathon constraints.** Gemini proposed pitch threshold, I proposed face Y-drop, GPT-5.2 proposed face position anomaly. These are the same idea from different angles. Combine them. Settled.
8. **Realistic accuracy is 7/10 without changes, 8/10 achievable with the agreed upgrades.** Settled.

**The one genuine remaining disagreement: Enrollment liveness method.**

This is the only point where the three reviewers haven't fully converged, and it matters because someone needs to code it.

- **Gemini (Round 3): Still advocates EAR blink detection from BlazeFace keypoints.** Gemini says "~1 hour, sufficient to block photo spoofs."
- **GPT-5.2 and I: Active head-pose challenge using existing PnP pipeline.** GPT-5.2 says "3-6 hours," I estimated 3-4 hours.

I want to be precise about why I maintain my position here, because Gemini's suggestion keeps recurring and could waste implementation time if someone tries it:

**BlazeFace's 6 keypoints are: left eye center, right eye center, nose tip, mouth center, left ear tragion, right ear tragion.** These are single points per feature, not contours. Eye Aspect Ratio requires at minimum 4 points per eye (upper lid, lower lid, inner corner, outer corner) — typically the 6-point-per-eye landmarks from dlib's 68-point model or MediaPipe's Face Mesh. You physically cannot compute a meaningful EAR from a single eye-center point. There is no vertical extent to measure. This isn't a tuning issue or a reliability concern — it's a geometric impossibility.

If Gemini is referring to a different BlazeFace variant that outputs more landmarks, that model is effectively Face Mesh (MediaPipe's 468-point model), which we've explicitly excluded from P0/P1 due to its ~40ms/frame cost on Celeron.

**Head-pose challenge liveness** uses PnP yaw, which we already have and have agreed to smooth with OneEuroFilter. The implementation is: display "Look LEFT" → verify yaw < -20° → display "Look RIGHT" → verify yaw > +20° → display "Look CENTER" → verify |yaw| < 10°. Each step has a 3-second timeout. This blocks printed photos (which can't turn), blocks video replays (unless the attacker knows the exact sequence, which is randomized), and adds zero new dependencies. The cost is ~3-4 hours including the UI prompt flow, but most of that is UI work you'd be doing anyway for enrollment.

**My final recommendation: head-pose challenge liveness. This is not negotiable on technical grounds — EAR from BlazeFace's 6 keypoints is not implementable.**

**On the desk ROI background subtraction disagreement:**

GPT-5.2 suggested it (Round 1-2), I opposed it (Round 2), Gemini opposed it (Round 3 — citing screen shake from typing). GPT-5.2 in Round 3 doesn't re-advocate for it strongly. I consider this settled: don't do it. The face Y-drop + pitch behavioral proxy covers the same scenarios with far less tuning risk.

**On the "what wins the hackathon" question:**

After three rounds, I want to sharpen the competitive analysis. The 10 other teams will likely build: MediaPipe Face Mesh for gaze (expensive but accurate), YOLO for object detection (phones, books), Flask/Streamlit dashboard, basic face matching, simple per-frame alerts. Their weaknesses will be: performance on Celeron (Face Mesh + YOLO will struggle), no cross-terminal intelligence, no temporal fusion, ugly dashboards, and alert spam without severity ranking.

Our genuine differentiators, ranked by jury impact:

1. **Cross-terminal seat-swap detection with live demo** — no other team will have this. When two students swap seats and the system catches it within seconds, showing both faces and the match, that's the "wow moment." This should be the centerpiece of the demo script.
2. **Live confusion matrix updating in real-time** — the jury explicitly asked for this. Most teams will generate it post-hoc. Ours updates live. High jury alignment.
3. **Risk score heatmap showing 30 students at a glance** — visually impressive, operationally useful, easy for a government jury to understand.
4. **Common-mode audio classification** — "The system knows the difference between room noise and two students whispering" is a compelling narrative.
5. **Hash-chained audit trail** — less visually impressive but addresses the "tamper-proof" requirement directly. Worth 30 seconds in the demo, not 3 minutes.

Things that are technically impressive but won't register with a government jury: MJPEG architecture, bounded queues, MOSSE tracking, adaptive FPS degradation. Don't spend demo time on these. They're infrastructure that enables everything else.

CONCERNS:

1. **Enrollment liveness method must be resolved NOW before coding starts.** If someone implements EAR with BlazeFace's 6 keypoints, they'll waste 2-3 hours discovering it doesn't work, then need to pivot to head-pose challenge anyway. Assign head-pose challenge liveness from the start. This is not a difference of opinion — it's a technical constraint.

2. **The 3-terminal to 30-terminal scale-up remains the highest operational risk.** All three reviewers have focused heavily on per-terminal concerns (memory, FPS, detection accuracy). The failure mode we haven't stress-tested is: what happens when 30 terminals simultaneously send events, SSE connections, and MJPEG streams to one edge server? The mini-PC running FastAPI needs to handle ~30 SSE connections, ~30 event streams (batch-flushed every 250ms), and dashboard rendering for a proctor viewing the seat map. This should work fine with async Python, but it needs to be tested at Phase 2 with simulated load if you only have 3 physical terminals. A simple script that generates fake events from 27 phantom terminals would take ~2 hours and could prevent a demo-day surprise.

3. **Demo script timing is under-discussed.** The jury will stage 6-10 malpractice events in a 10-minute session. That's one event every 60-100 seconds. Your detection latencies range from near-instant (CANDIDATE_ABSENT at 5s, MULTIPLE_PERSONS at ~0.3s) to slow (SUSPICIOUS_AUDIO at 10s standalone, GAZE_DEVIATION at 30s for the +30° threshold). If the jury stages events in rapid succession with <60s gaps, your sustained-duration thresholds may not resolve before the next event starts. Consider: should the ground truth logger's "skip to next" button also reset terminal-side accumulators? Otherwise event N's residual state could interfere with event N+1's detection.

4. **The exam UI quality question hasn't been answered.** All three reviewers focused on detection and dashboard. But the candidates interact with the 10-MCQ exam UI. If it looks amateurish (unstyled HTML, no timer, no question navigation), it undermines the jury's confidence in the whole system. Allocate 3-4 hours for a clean, minimal exam UI: question counter, timer bar, radio buttons, "Next"/"Submit" buttons, progress indicator. Use a CSS framework like Pico CSS (~3KB, no build step) for instant polish.

5. **Embedding persistence (from my Round 2, not addressed by others).** If the edge server process restarts mid-demo, enrollment embeddings stored only in memory are lost. Cross-terminal seat-swap matching breaks. Embeddings must be persisted — either in SQLite (as a BLOB column in an enrollments table) or as a .npy file on disk. This is ~30 minutes of implementation but prevents a catastrophic failure mode.

SUGGESTIONS:

1. **Resolve the liveness method: implement head-pose challenge (P1, 3-4 hours).** Sequence: LEFT → RIGHT → CENTER, 3s timeout per step, randomize order, validate via smoothed PnP yaw. UI: simple full-screen text prompt with webcam feed. Blocks photos, blocks pre-recorded video. Done.

2. **Implement 30-terminal load simulation (P2, 2 hours).** A Python script that connects 27 fake terminals to the edge server, each sending realistic event streams (periodic heartbeats, occasional alerts). Run this during your 3-terminal physical test. If the edge server chokes, you find out at Phase 2, not demo day.

3. **Add event accumulator reset mechanism (P2, 1-2 hours).** When the ground truth logger signals "next event," broadcast a soft-reset to all terminals that clears running counters (gaze duration, audio duration, posture duration) but NOT identity state (enrollment, face verification). This prevents cross-contamination between staged events and ensures each event gets a clean detection window.

4. **Invest in exam UI polish (P2, 3-4 hours).** Use Pico CSS or Simple.css. Implement: question text, 4 radio options, "Next" button, question counter ("3 of 10"), countdown timer bar, "Submit" button on last question. The content doesn't matter (use placeholder questions) — the visual quality does. The exam UI is the candidate's entire experience; the jury will see it on every terminal.

5. **Persist enrollment embeddings to SQLite (P1, 30 minutes).** Add an `enrollments` table with columns `terminal_id`, `candidate_name`, `embedding_blob`, `enrolled_at`. Load into memory on edge server startup. Write on enrollment. This is trivial but prevents a class of restart-related failures.

6. **Create a 90-second demo script with designated "hero moments" (P5, 2 hours).** Script the exact sequence: (a) Show seat-map, all green. (b) Trigger seat swap — watch two cells turn red, show the cross-match. (c) Trigger candidate absent — cell goes gray. (d) Trigger phone use — show behavioral detection with annotated snapshot. (e) Show confusion matrix — "8 of 10 detected, 0 false positives." (f) Show hash-chained audit log — "tamper-proof evidence." Practice this sequence 5+ times. The demo script is as important as the code.

7. **Final priority stack for implementation (revised from all three rounds):**

   - **P0 (gate):** Pre-flight: RAM check, ORT model load + dummy inference, webcam open + FPS test, audio device test, Edge/Chrome version check, disk write latency. ~4-6 hours.
   - **P1 (core):** Agent + webcam + MJPEG + enrollment with head-pose liveness + BlazeFace + MOSSE + MobileFaceNet verification (adaptive cadence) + PnP gaze with OneEuroFilter + Silero VAD + store-and-forward buffer + time-based thresholds + CLAHE for detection only. ~30-40 hours.
   - **P2 (detection + vertical slice):** All 7 alert types (with face Y-drop behavioral proxy replacing skin-tone blob) + 3-terminal test + 30-terminal load simulation + event accumulator reset + exam UI + embedding persistence. ~20-25 hours.
   - **P3 (dashboard + fusion + demo):** Seat-map heatmap + confusion matrix + incident cards + fusion rules + CCTV MOG2 heatmap + risk score + post-exam summary. ~15-20 hours.
   - **P4 (conditional stretch):** NanoDet phone detection, desk ROI background subtraction, advanced visualizations. Only if P0-P3 pass.
   - **P5 (validation):** Threshold tuning, demo script practice, golden demo fallback video, 30-terminal full rehearsal. ~8-10 hours.

KEY_TRADEOFFS:

1. **Head-pose liveness vs. no liveness (settled).** 3-4 hours of implementation prevents a trivial spoofing attack that would embarrass you in front of the jury. EAR blink detection is geometrically impossible with BlazeFace's 6 keypoints and should not be attempted. This is the only remaining disagreement across three reviewers, and it's resolved by the technical constraint.

2. **Behavioral material detection vs. object detection (settled).** Face Y-drop + pitch behavioral proxy costs ~3 hours, works when the object is invisible to the camera, and has manageable false-positive risk. Object detection (YOLO/NanoDet) costs 4-6 hours, adds ~15-30MB RAM, and still can't see objects hidden under desks. The behavioral approach is strictly superior for this specific deployment scenario. Other teams using YOLO for phone detection will fail on "phone under desk" just as badly, AND burn more CPU doing it.

3. **Demo script investment vs. more features (critical).** A polished 90-second demo with 5 rehearsed "hero moments" will outperform a feature-rich system with a fumbled live demo every single time. The jury evaluates what they see in real-time, not what's in your architecture document. Allocate the last 10% of your time budget to demo rehearsal, not feature additions.

4. **30-terminal simulation vs. trusting the architecture (new).** Spending 2 hours on a load simulation script seems wasteful when you could build another feature. But discovering at demo time that your edge server can't handle 30 concurrent SSE connections is catastrophic and unfixable on the spot. This is high-value insurance.

5. **Exam UI polish vs. detection work (under-appreciated).** The exam UI is the only component every candidate and every jury observer interacts with directly. An ugly exam UI creates a first impression that undermines confidence in the sophisticated detection happening behind the scenes. 3-4 hours with a classless CSS framework transforms the perceived quality of the entire system.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
Other participants’ feedback didn’t change my diagnosis, but it **did change my recommended “must-do before coding” list**. There is now clear convergence across all reviews that the architecture is *structurally* right for low-end Windows PCs, but **8/10 detection is not safe unless you patch three specific gaps**:

1) **Material/use-on-lap is undercovered** (current heuristics are either blind or high-FP).  
2) **Raw PnP is too noisy** and will destabilize gaze + every fusion rule that depends on gaze.  
3) **Demo-day reliability issues (RAM + network + ORT CPU compatibility)** can zero-out your score even if the algorithms are decent.

What changed after reading Claude/Gemini:
- I’m **more convinced you should drop the skin-tone heuristic entirely**, not “gate it.” Claude/Gemini are right: it’s a false-positive factory (hands, arms, face-touching), and false positives will destroy jury trust.
- I’m **less confident in my own earlier “desk ROI background subtraction” idea as a P1 deliverable**. Claude/Gemini’s point about tuning risk is valid: “desk ROI change” becomes noisy fast (shadows, paper movement, posture shifts). I’d now treat it as optional, not core.

Where I disagree with others (genuine technical disagreement):
- **Blink liveness via EAR from BlazeFace landmarks is very likely infeasible** given BlazeFace’s typical 6 keypoints (eye *centers*, nose, mouth, tragions). EAR requires eyelid landmarks. Unless you *confirm* you truly have upper/lower eyelid points, Gemini’s blink-EAR suggestion will waste time and fail in practice. Head-pose challenge liveness (PnP) is the correct low-risk option.

Common ground across all reviewers is strong and actionable:
- Drop skin-tone; keep glow (or replace with behavioral proxy).
- Add OneEuro/EMA smoothing to PnP.
- Move store-and-forward to P1.
- Make thresholds time-based (not frame-count-based).
- Add risk score + confusion matrix + evidence cards for “jury-visible” impact.
- As-is accuracy is realistically ~7/10; with targeted fixes you can reach 8/10.

CONCERNS:
1. **Material detection gap remains the most likely reason you miss 8/10**, especially “phone under desk” or “notes on lap.” Glow-only will miss many; skin-tone will false-alarm.
2. **PnP jitter without smoothing will either inflate false positives or force thresholds so high you miss real glances**, and it cascades into your fusion logic.
3. **Blink-based liveness is likely a trap** unless you have eyelid landmarks (most BlazeFace variants don’t).
4. **Phase thresholds expressed in frames (“5 consecutive frames”) will behave differently under load** with frame dropping; this creates unpredictable detection.
5. **ORT CPU instruction-set mismatch (AVX/AVX2) is a silent demo killer** on unknown Celerons unless Phase-0 proves sessions load and inference runs.
6. **Store-and-forward in P2 is too late for a hackathon**; one network/edge hiccup during staged events loses points.
7. **CLAHE may drift face embeddings if applied to verification crops** (Claude’s point is good): could increase FACE_MISMATCH false alarms.
8. **30-terminal scale-up failure mode** is operational, not algorithmic: too many MJPEG streams, edge overload, “one weird webcam” issues.

SUGGESTIONS:
1. **Material detection: commit to “behavioral proxy” as the core (P1, low-risk)**
   - Implement **face centroid Y-drop baseline** + **pitch-down** fusion (Claude/Gemini convergence).
   - Trigger `POSSIBLE_MATERIAL_USE` when:  
     - `pitch_down_filtered > 20–25°` for ≥5s **OR** `face_center_y > baseline + 0.15*face_height` for ≥5s  
     - Stronger confidence when both are true.
   - Keep phone glow only as a *bonus* signal (low FP), not the foundation.
   - Explicitly **remove skin-tone blob**.

2. **PnP stabilization: OneEuroFilter (P1, must-do)**
   - Filter yaw/pitch/roll (or at least yaw/pitch). This is non-negotiable if gaze drives fusion rules.

3. **Enrollment liveness: head-pose challenge (P1, must-do)**
   - Randomized prompt sequence: LEFT → RIGHT → CENTER (optionally UP) with timing constraints.
   - Validate trajectory crosses thresholds (e.g., yaw < -20°, yaw > +20°) within N seconds.
   - This blocks printed photo / static screen spoof and is implementable with your existing PnP.

4. **Convert all persistence logic to time-based (P1, must-do)**
   - Replace “N frames” with “≥X ms” everywhere (MULTIPLE_PERSONS, etc.).
   - This is the correct fix for bounded queue + frame drops.

5. **Adaptive verification cadence (P1, high leverage)**
   - Baseline verify every 2s, but **verify immediately on suspicion** (gaze deviation onset, face reacquire, multiple persons).
   - This directly improves catching short staged “seat swap” events without constant compute.

6. **Store-and-forward promoted to P1 (must-do)**
   - Local JSONL queue + retry until ACK. Keep at least last 1–2 thumbnails per incident locally until confirmed delivered.

7. **CLAHE scoping fix (P1, 15 minutes, do it)**
   - Use CLAHE only for BlazeFace detection.
   - Crop verification faces from the **original frame** for MobileFaceNet to avoid embedding drift.

8. **Demo impact (P3, but design now)**
   - A single main dashboard view should show:
     - **30-seat risk heatmap grid**
     - **live confusion matrix** (jury explicitly asked)
     - **incident cards with annotated snapshots** (pose arrow, face box, “fusion explanation” text)
   - Avoid showing 30 live MJPEG streams; show “click to open stream” to prevent edge/network overload.

KEY_TRADEOFFS:
- **Behavioral proxy vs object detection**: You trade “I see a phone” specificity for “I see down-looking / lap-reading behavior” reliability on weak CPUs. Given constraints, this is the right tradeoff—and it’s what will actually push you from ~7/10 toward 8/10.
- **PnP smoothing vs latency**: You add ~50–150ms lag but massively reduce jitter-driven false alarms. Correct tradeoff for proctoring (seconds-scale events).
- **Head-pose liveness vs blink liveness**: Head-pose uses sensors you already trust (PnP), blink likely fails with sparse BlazeFace landmarks. Correct tradeoff is head-pose.
- **Scope vs robustness**: Removing skin-tone and deprioritizing desk-ROI subtraction reduces feature count, but increases credibility and reduces tuning risk. For a hackathon judged live, that’s the right prioritization.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The debate has produced a highly coherent execution plan. I accept the correction from Claude Opus and GPT-5.2 regarding **Liveness Detection**: my previous suggestion of Eye Aspect Ratio (EAR) was technically flawed because standard BlazeFace (6-keypoint) lacks the necessary eyelid landmarks. The **Head-Pose Challenge** (Look Left/Right/Center) is the superior, zero-dependency alternative that utilizes the existing PnP pipeline.

We have reached a "Grand Consensus" on the Critical Path:
1.  **Material Detection is Behavioral:** We cannot reliably detect objects (phones). We must detect the *act* of using them (looking down, face dropping).
2.  **Stability over Raw Performance:** Time-based thresholds, signal smoothing (OneEuroFilter), and memory safety (Edge/Restricted Chrome) are now P0 requirements, not optimizations.
3.  **Visual Proof is King:** The dashboard must visualize "Risk," not just log events.

I disagree with GPT-5.2 regarding the "Desk ROI Change" detector. As Claude noted, keyboard vibration and lighting shifts make background subtraction too brittle for a hackathon. The "Face Drop" heuristic is cleaner and sufficient.

CONCERNS:
1.  **The "Time-Based" Refactor Risk:** Converting all logic from `frames` to `milliseconds` is a pervasive change. If done hastily, it introduces bugs (e.g., `time.time()` drift vs monotonic clocks).
2.  **AVX Instruction Set (Critical):** I strongly echo the concern about Celeron CPUs. Standard `onnxruntime` builds often require AVX2. If the venue's desktops are old Celerons (Atom-based), the app will crash on import.
3.  **Calibration Dependency:** The "Face Drop" heuristic relies on a stable baseline. If a student adjusts their laptop screen angle mid-exam, the baseline is invalid, potentially triggering false positives for the rest of the session.
4.  **Network Saturation:** 30 terminals sending MJPEG + Events + Embedding searches simultaneously could choke a cheap 4G router/switch, causing "Candidate Absent" false positives due to dropped frames.

SUGGESTIONS:
1.  **Adopt "Head-Pose Challenge" for Liveness (P0):** Replace the missing blink detection. During enrollment, prompt: "Turn head LEFT... hold... RIGHT... hold." Verify using the PnP yaw values you already calculate.
2.  **Finalize Material Heuristic (The "Face Drop" Logic):**
    *   *Calibration:* Compute average Face Center Y over first 10s of exam.
    *   *Trigger:* `(Current_Y > Baseline_Y + 15%)` AND `(Pitch > 15° Down)`.
    *   *Recalibration:* If `Face_Confidence` is high but position is "Wrong" for >60s, slowly drift the baseline to the new position (exponential moving average) to handle screen adjustments.
3.  **Implement Adaptive Verification (P1):**
    *   Default: Verify MobileFaceNet every 5s (save CPU).
    *   Trigger: On `GAZE_DEVIATION` start, `FACE_MISMATCH`, or `POSSIBLE_MATERIAL_USE`, force an immediate verification (debounce 500ms). This catches the "quick seat swap."
4.  **Browser & Memory Strategy:**
    *   **Primary:** Target **Microsoft Edge**. It is pre-installed, lighter than Chrome, and harder for users to mess up (fewer extensions installed).
    *   **Safety:** Use Phase 0 to check for AVX support. If missing, have a fallback `onnxruntime` wheel (non-AVX/OpenVINO) ready in the repo.
5.  **Refine Dashboard Visuals (P2 -> P1):**
    *   **Grid View:** 30 squares. Green (OK), Yellow (Warning), Red (Critical).
    *   **Confusion Matrix:** Live update comparing `Alerts` vs `GroundTruth_Log` (if available/mocked).
6.  **Network Safety:** Implement a "Heartbeat" packet separate from the MJPEG stream. If MJPEG lags but Heartbeat arrives, do *not* trigger CANDIDATE_ABSENT.

KEY_TRADEOFFS:
*   **Head-Pose Liveness vs. UX:** This is slightly more annoying for the user than a passive blink check, but it is **technically guaranteed** to work with your model set, whereas blink is not.
*   **Behavioral Detection vs. Specificity:** "Face Drop" cannot distinguish between "reading a phone" and "slumping tiredly." We accept this False Positive risk because catching the malpractice is worth the proctor checking a tired student.
*   **Edge vs. Chrome:** We trade the developer familiarity of Chrome for the operational stability and memory footprint of Edge. Given the 4GB constraint, this is the correct move.

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
1. The overall architecture (agent-owns-webcam via MJPEG, bounded queue size 1 with frame dropping, lightweight INT8 ONNX models, edge-side fusion, SQLite WAL) is structurally sound and well-suited to the Celeron/4GB constraint.
2. The current architecture, without modifications, will realistically achieve ~7/10 accuracy, not the claimed 8/10.
3. The skin-tone blob heuristic must be dropped entirely — it produces unacceptable false positives (hands on desk, arm positions, face touching) and will erode jury trust.
4. Phone glow detection (bright rectangular region) should be kept as a low-FP bonus signal, but not relied upon as a primary detector.
5. Material detection must pivot to a behavioral proxy: face bounding-box centroid Y-drop from baseline combined with pitch-down from PnP, triggering POSSIBLE_MATERIAL_USE.
6. Raw PnP head pose from BlazeFace's 6 keypoints is too noisy without temporal smoothing and will destabilize gaze detection and all three fusion rules that depend on it.
7. OneEuroFilter is the preferred smoothing method for PnP output (velocity-adaptive, suppresses jitter while passing genuine fast head turns).
8. All persistence/duration thresholds must be time-based (milliseconds), not frame-count-based, to prevent unpredictable behavior under frame dropping and variable FPS.
9. Store-and-forward must be promoted from P2 to P1 — a local JSONL queue with retry-until-ACK is ~2-3 hours and prevents catastrophic evidence loss during network blips.
10. Phase 0 pre-flight must include actual ONNX Runtime session load and dummy inference on every model file, not just RAM checks, to catch AVX/AVX2 instruction set incompatibility on older Celerons.
11. Enrollment requires liveness detection to prevent the trivial photo-spoof attack that would embarrass the team during a demo.
12. The dashboard must be built around three visual elements: (a) a 30-seat risk heatmap grid color-coded green/yellow/orange/red, (b) a live auto-updating confusion matrix comparing detections vs. ground truth, and (c) annotated incident cards with face bbox overlays, pose arrows, confidence scores, and fusion explanations.
13. A per-student risk score (weighted sum of alert severities with configurable weights and time decay) is a must-add, not a nice-to-have, and should be prominently displayed and sortable.
14. Microsoft Edge should be the primary target browser over Chrome — it is pre-installed on Windows 10/11, uses the same Chromium engine, and has ~100-200MB lower baseline memory. Chrome with restrictive flags is the fallback.
15. CLAHE should be applied only for BlazeFace detection; face crops for MobileFaceNet embedding extraction must come from the original (non-CLAHE) frame to prevent embedding drift.
16. Cross-terminal seat-swap detection is the single strongest differentiator and demo "wow moment" — it should be the centerpiece of the demo script.
17. The 3-terminal vertical slice (Phase 2) is the most critical gate in the entire project; if it doesn't work at 3 terminals, it won't work at 30.
18. PnP head pose should be honestly framed as "head orientation detection," not "gaze tracking," since sparse keypoints cannot capture eyes-only glances.
19. Hash-chained audit trails and other invisible infrastructure features deserve brief mention in the demo but should not consume demo time — prioritize visually demonstrable features.
20. A post-exam one-page HTML summary per student (photo, risk timeline, alert list with thumbnails, exam score) is high-impact and low-effort (~2-3 hours).
21. Enrollment embeddings must be persisted to disk (SQLite or .npy file), not held only in edge server memory, to survive process restarts.

CONVERGENCE:
1. **Material detection strategy**: GPT-5.2 initially proposed desk ROI background subtraction (MOG2), Claude proposed face-position anomaly tracking, Gemini proposed tightened pitch thresholds. By Round 3, all converged on the combined behavioral proxy (face centroid Y-drop + pitch-down fusion) as the primary approach, with GPT-5.2 explicitly deprioritizing their own desk ROI suggestion due to tuning risk.
2. **Desk ROI background subtraction**: GPT-5.2 advocated for it in Rounds 1-2; Claude and Gemini opposed it citing tuning noise (shadows, paper movement, typing vibration). By Round 3, GPT-5.2 conceded it should be optional/stretch only, not core.
3. **CLAHE policy**: Initially specified as applied to every frame before all processing. Claude raised the embedding drift concern in Round 2 based on GPT-5.2's initial flag. All converged on applying CLAHE only for detection, not for verification crops.
4. **Store-and-forward timing**: Originally slated for P2 in the proposal. Claude and GPT-5.2 independently flagged this in Round 1. By Round 2, all three agreed it belongs in P1.
5. **Adaptive verification cadence**: GPT-5.2 proposed verify-on-suspicion in Round 1. Claude endorsed it in Round 2. Gemini adopted it in Round 3 (with a suggestion to relax the baseline from 2s to 5s). All agree on the principle of event-triggered immediate verification.
6. **Smoothing method**: Claude initially proposed median filter, Gemini proposed OneEuroFilter, GPT-5.2 accepted either. By Round 2, all converged on OneEuroFilter as technically superior (velocity-adaptive).
7. **Realistic accuracy assessment**: The proposal claimed 8/10 with <5% FP. All three reviewers independently assessed ~7/10 as-is in Round 1 and converged on 8/10 being achievable only with the agreed targeted upgrades.
8. **Browser choice**: Not initially specified in the proposal. Claude raised Edge in Round 2, Gemini endorsed it in Round 3, GPT-5.2 agreed. All converged on Edge as default with Chrome as fallback.

REMAINING_DISAGREEMENTS:
1. **Enrollment liveness method**: Gemini advocated Eye Aspect Ratio (EAR) blink detection from BlazeFace keypoints through all three rounds, claiming ~1 hour implementation. Claude and GPT-5.2 both argued this is geometrically impossible with BlazeFace's standard 6 keypoints (eye centers, nose, mouth, ear tragions), which lack the eyelid landmarks required for EAR computation. Claude and GPT-5.2 advocated active head-pose challenge liveness using the existing PnP pipeline (~3-4 hours). In Round 3, Gemini formally accepted the correction and endorsed head-pose challenge. **Status: Effectively resolved in favor of head-pose challenge, though Gemini's concession was late and implementation time estimates differ (Claude: 3-4 hours, GPT-5.2: 3-6 hours).**

2. **Baseline verification cadence**: Gemini suggested relaxing the default MobileFaceNet verification interval from every 2s to every 5s to save CPU, with event-triggered immediate verification compensating. Claude and GPT-5.2 kept the 2s baseline with adaptive acceleration on suspicion. **Status: Minor disagreement — all agree on verify-on-suspicion; the baseline cadence (2s vs 5s) is unresolved but low-stakes since it's a tunable parameter.**

3. **Face-drop baseline recalibration**: Gemini uniquely raised the concern that the face centroid Y baseline could be invalidated by a student adjusting their screen mid-exam, and proposed exponential moving average recalibration if the position persists >60s. Claude and GPT-5.2 did not address this specific concern. **Status: Gemini's recalibration idea is uncontested but also unendorsed — it should be evaluated during implementation.**

4. **Heartbeat vs. MJPEG for presence detection**: Gemini uniquely proposed a separate lightweight heartbeat packet so that MJPEG stream lag doesn't trigger false CANDIDATE_ABSENT alerts. Claude and GPT-5.2 did not address this specific mechanism. **Status: Uncontested but unendorsed — a reasonable idea that needs evaluation.**

RECOMMENDED_PLAN:

**Pre-Coding Decisions (Resolve Now)**
- [ ] Enrollment liveness method: **Head-pose challenge** (Look LEFT → RIGHT → CENTER via PnP yaw). Do NOT attempt EAR blink detection with BlazeFace's 6 keypoints.
- [ ] Material detection strategy: **Drop skin-tone blob entirely.** Keep phone glow as bonus signal. Primary detection is face centroid Y-drop + pitch-down behavioral proxy.
- [ ] Browser target: **Microsoft Edge** as default. Chrome with `--disable-gpu --disable-extensions --renderer-process-limit=2 --disable-site-isolation-trials` as fallback. Phase 0 checks browser version.
- [ ] All persistence thresholds: **Time-based (milliseconds), not frame-count.** Enforce this as a coding standard from day one.
- [ ] Risk score formula: Configurable weights in `config.yaml`, normalized 0-1 severity per alert type, exponential time decay. NOT hard-coded integer weights.

**Phase 0 — Pre-Flight Diagnostic (~4-6 hours)**
- [ ] RAM availability check (warn if <800MB free after OS + browser estimate)
- [ ] ONNX Runtime session load + dummy inference for ALL model files (catches AVX/AVX2 incompatibility)
- [ ] Ship a fallback non-AVX ORT wheel (or `onnxruntime-openvino`) in the repo
- [ ] Webcam open + resolution check (warn below 640×480, fail below 320×240)
- [ ] Sustained FPS baseline test (~5 seconds of capture + inference)
- [ ] Audio device access test
- [ ] Disk write latency test
- [ ] Edge/Chrome version check
- [ ] Windows Defender exclusion reminder for the data directory
- [ ] Single green/red readiness dashboard page for invigilator

**Phase 1 — Core Pipeline (~30-40 hours)**
- [ ] Headless Python agent with 3-thread architecture (capture → inference → event push)
- [ ] `cv2.VideoCapture(0)` exclusive webcam ownership + MJPEG serving to browser
- [ ] CLAHE preprocessing for BlazeFace detection only; original frame crops for MobileFaceNet
- [ ] BlazeFace face detection (INT8 ONNX)
- [ ] MOSSE tracker between detections
- [ ] MobileFaceNet + ArcFace verification with **adaptive cadence**: 2s baseline, immediate on suspicion (gaze deviation onset, face loss/reacquire, multiple persons), rate-limited to 1 per 500ms
- [ ] PnP head pose from BlazeFace keypoints with **OneEuroFilter** on yaw and pitch
- [ ] Silero VAD for audio
- [ ] Phone glow heuristic (bright rectangular region in lower frame)
- [ ] **Face centroid Y-drop behavioral proxy**: compute baseline from first 10s, trigger when Y-drop >15% of face height for ≥5s; consider Gemini's slow-drift recalibration for screen adjustments
- [ ] Head-pose challenge enrollment liveness: randomized LEFT/RIGHT/CENTER prompts, yaw must cross ±20°, 3s timeout per step
- [ ] Per-frame brightness tracking (brightness gate for CANDIDATE_ABSENT)
- [ ] **Store-and-forward**: local JSONL deque (maxlen ~200), retry POST until ACK, retain last 1-2 evidence thumbnails per incident
- [ ] Enrollment embedding persistence to SQLite (BLOB column) or .npy on disk
- [ ] App-level time sync (RTT/2 offset, median of last 5)
- [ ] Browser-served enrollment flow + exam UI stub

**Phase 2 — All Detections + 3-Terminal Vertical Slice (~20-25 hours)**
- [ ] All 7 alert types implemented with time-based thresholds
- [ ] SUSPICIOUS_POSTURE: face Y-drop + pitch fusion, trigger at 5s when both signals agree
- [ ] MULTIPLE_PERSONS: ≥400ms (time-based) with 2+ faces
- [ ] CANDIDATE_ABSENT: >5s no face with brightness gate
- [ ] FACE_MISMATCH: 3-zone hysteresis, cosine >0.70 sustained across 3+ checks (~6s)
- [ ] GAZE_DEVIATION: filtered PnP, >30s at +30°, >3s at +35°, instant at +45°
- [ ] SUSPICIOUS_AUDIO: >10s standalone, >5s with gaze fusion
- [ ] POSSIBLE_MATERIAL_USE: face Y-drop + pitch behavioral proxy + phone glow bonus
- [ ] 3-terminal physical test on actual or similar hardware
- [ ] **30-terminal load simulation script** (~2 hours): 27 phantom terminals generating realistic event streams + SSE connections to stress-test edge server
- [ ] **Event accumulator reset mechanism**: ground truth logger "next event" broadcasts soft-reset of running counters (gaze duration, audio duration, posture duration) to terminals — does NOT reset identity state
- [ ] Exam UI with Pico CSS or Simple.css: 10 MCQs, question counter, countdown timer, radio buttons, Next/Submit, progress indicator (~3-4 hours)
- [ ] Implement separate heartbeat from MJPEG stream (evaluate Gemini's suggestion to prevent false CANDIDATE_ABSENT from stream lag)

**Phase 3 — Dashboard + Fusion + CCTV + Demo Infrastructure (~15-20 hours)**
- [ ] FastAPI + SSE + vanilla JS/HTMX dashboard
- [ ] **30-seat risk heatmap grid** (color-coded green/yellow/orange/red by risk score) — largest element
- [ ] **Live confusion matrix** widget auto-updating against ground truth logger — second largest element
- [ ] **Annotated incident cards**: snapshot with face bbox, pose arrow overlay, confidence %, fusion rule explanation text
- [ ] Per-student sortable risk score column
- [ ] 3 fusion rules: GAZE+FACE_MISMATCH→CONFIRMED_IMPERSONATION, AUDIO+GAZE→POSSIBLE_CONSULTATION, POSTURE+GAZE→POSSIBLE_MATERIAL_USE
- [ ] Cross-terminal seat-swap: on FACE_MISMATCH, compare embedding against all 30 enrolled
- [ ] Common-mode audio classification: ≥3 terminals→ROOM_NOISE, 2 adjacent→POSSIBLE_CONSULTATION, 1→SUSPICIOUS_AUDIO
- [ ] CCTV: BackgroundSubtractorMOG2 + motion heatmap overlay
- [ ] Post-exam one-page HTML summary per student (photo, timeline, alerts with thumbnails, risk score, exam score) — printable via browser
- [ ] Ground truth logger + malpractice trigger app with "skip to next" button
- [ ] Do NOT show 30 live MJPEG streams simultaneously — use "click to open stream" to prevent network/edge overload

**Phase 4 — Conditional Stretch (Only if P0-P3 pass 3-terminal test)**
- [ ] NanoDet-Plus phone detection (~1.8MB INT8, ~20ms on i5, 2fps on desk ROI)
- [ ] Desk ROI background subtraction
- [ ] Advanced visualizations
- [ ] Any P2 upgrades from original plan

**Phase 5 — Validation + Demo Prep (~8-10 hours)**
- [ ] Threshold tuning on actual hardware with staged events
- [ ] **90-second demo script** with designated hero moments: (1) all-green seat map, (2) seat-swap detection with cross-match display, (3) candidate absent, (4) behavioral material-use detection with annotated snapshot, (5) confusion matrix showing "8/10 detected, 0 false positives," (6) hash-chained audit log (brief mention)
- [ ] Practice demo sequence 5+ times
- [ ] Golden demo fallback: concrete plan (pre-recorded agent output fed to edge server for terminals that fail on demo day)
- [ ] One-page architectural diagram with icons (Figma/Canva, ~30 minutes) showing data flow with model names and latency numbers at each step — print poster-size if possible
- [ ] 30-terminal full rehearsal if possible
- [ ] Brief presenter on framing: "head orientation detection" not "gaze tracking"; behavioral analysis as a deliberate privacy-preserving choice, not a limitation; proactively acknowledge phone-under-desk-with-minimal-tilt as a known boundary

**Key Risk Mitigations**
- [ ] Highest risk: 3→30