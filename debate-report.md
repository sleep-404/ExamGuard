# Multi-LLM Debate Report

**Models:** Claude Opus, GPT-5.2, Gemini 3 Pro
**Rounds completed:** 3

---
## Round 1

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

This is a remarkably well-thought-out architecture that has clearly benefited from multiple rounds of refinement. The core decisions — local processing, episode-based events, detect→track→verify pipeline, SQLite WAL, no Electron, no YOLO — are all sound. The degradation ladder is clever. The scope cuts are almost all correct. But I have significant concerns about demo-day execution risk and a few architectural choices that could be the difference between winning and losing.

**What works well:**

1. **The detect→track→verify pipeline** is the right pattern. BlazeFace at 3-5fps with MOSSE tracking in between is an excellent tradeoff for 4GB machines. The 2s verification interval is sensible.

2. **Episode-based event model** is the right abstraction. Per-frame spam would kill both the network and the dashboard UX. This shows architectural maturity the judges will notice.

3. **The degradation ladder** (≥12fps full → ≥7fps shed gaze → ≥4fps shed audio → <4fps SURVIVAL) is practical and demonstrates real systems thinking. However, it needs to be *visible* on the dashboard — judges should see you gracefully degrading, not silently failing.

4. **Ground truth logger** built into the admin dashboard is a strategic choice. Making it easy for the jury to log their staged events and immediately see the confusion matrix is a competitive advantage.

5. **Store-and-forward with JSONL spool** — correct for LAN reliability.

6. **CLAHE preprocessing** at ~2ms is the right call for classroom lighting variation.

**What concerns me deeply:**

**The PyQt6 decision is the single biggest risk in this architecture.** Here's why:

- You're deploying to *unknown government desktops* that you don't control. PyQt6 needs platform-specific Qt binaries. WinPython portable with PyQt6 is a ~400-500MB distribution that may hit antivirus/group-policy blocks on locked-down government machines.
- PyQt6 has known issues with webcam integration on certain Windows builds. OpenCV's `VideoCapture` + PyQt6's event loop can deadlock if the threading model isn't exactly right.
- You're asking 30 terminals to simultaneously run PyQt6 + ONNX Runtime + OpenCV + Silero — on machines that might be Celerons with 4GB RAM. PyQt6 alone consumes 80-150MB.
- **A browser-based terminal UI (FastAPI serving lightweight HTML/JS) with the AI pipeline as a headless Python process** would be dramatically simpler to deploy, more resilient, and avoids the entire Qt dependency chain. The exam UI is just 10 MCQ questions with a timer and navigation — this is a web form, not a desktop application.

**The 10-day schedule is back-loaded with risk.** Multi-terminal testing doesn't happen until Day 8. That means you discover LAN congestion, port conflicts, firewall issues, and race conditions with only 2 days to fix them. In a hackathon, the #1 cause of demo failure is "worked on one machine, broke at scale."

**The audio strategy has a fundamental flaw.** Silero VAD with a 10-second sustained speech threshold will miss almost every real cheating scenario. Students whispering to each other do so in 2-3 second bursts, not 10-second monologues. But dropping to 3-5 seconds will generate false positives from coughs, chair scraping, and ambient noise. The "common-mode audio rejection" (subtracting signals detected across multiple terminals simultaneously) is a good idea but is non-trivial to implement correctly with clock skew across terminals.

**The CCTV omission is a strategic risk.** The challenge document *explicitly* lists "Integrate CCTV and laptop camera feeds" as requirement #1. While the architecture debate correctly identified CCTV as high demo-failure risk, completely dropping it means you're visibly not addressing a stated requirement. Even a minimal "CCTV feed viewer with motion heatmap" on the dashboard — no AI, just display + simple background subtraction — would check this box.

CONCERNS:

1. **PyQt6 deployment on unknown government machines is the most likely cause of demo-day failure.** Antivirus quarantine, missing Visual C++ redistributables, Windows Defender blocking unsigned executables, group policy restrictions on running portable Python distributions. You have *zero* control over these machines and likely limited time for setup.

2. **Multi-terminal testing at Day 8 is too late.** You need at least 3-terminal testing by Day 5. LAN issues (30 terminals pushing JPEG evidence + JSON events to a single FastAPI server) will surface bugs you can't predict: port exhaustion, asyncio backpressure, SQLite WAL write contention under concurrent inserts.

3. **The gaze baseline calibration is underspecified.** "Yaw >baseline+30°" implies a per-candidate baseline capture during enrollment. How? If the candidate looks slightly left during enrollment, their baseline is skewed. If you use a fixed threshold instead, inter-individual variation in sitting position relative to the webcam will produce systematic false positives for candidates at the edges of the room.

4. **MobileFaceNet with 5 enrollment embeddings may not be robust enough for the "seat swap" scenario.** The challenge specifically mentions seat swaps. If two candidates swap seats (and both are enrolled), the system needs to detect that Terminal A's face now matches Candidate B's enrollment — this requires cross-checking against ALL enrolled embeddings, not just the terminal's own. Is the architecture doing this? If not, you'll miss the marquee demo scenario.

5. **30 concurrent webcam captures on a shared USB bus** could be a problem. If these are USB webcams on desktops connected through shared USB controllers or hubs, bandwidth contention could cause frame drops or outright device enumeration failures. This is especially bad on older Celeron machines with limited USB bandwidth.

6. **The 3-zone hysteresis for face mismatch (0.45/0.70 thresholds) needs hardware-specific calibration.** MobileFaceNet cosine distance distributions shift significantly with webcam quality, lighting, and JPEG compression. The thresholds that work in your dev environment may produce either constant false positives or complete misses on exam hall hardware.

7. **No explicit handling of enrollment fraud.** If the wrong person enrolls at a terminal (before the exam starts), the system will happily verify them for the entire exam. The architecture assumes enrollment integrity but provides no mechanism to verify it.

8. **SQLite WAL under 30 concurrent writers** (if events are pushed to edge server) — WAL mode helps but SQLite still has a single-writer lock. Under burst conditions (30 terminals simultaneously reporting events), you could see `SQLITE_BUSY` errors. Need explicit retry logic with backoff.

9. **The SUSPICIOUS_POSTURE alert (head pitch >30° downward for >10s) will fire constantly** for students who look down to read questions or think. This is a normal exam behavior. Without a way to distinguish "reading the screen" from "looking at phone in lap," this alert type will dominate your false positives and tank your confusion matrix.

10. **No mention of webcam resolution or frame size strategy.** Are you capturing at 640x480? 1280x720? This matters enormously for both BlazeFace accuracy and CPU load. On 4GB machines, 720p frames in a Python process with OpenCV will consume significant memory in the capture buffer alone.

SUGGESTIONS:

1. **Replace PyQt6 with a browser-based terminal UI.** Run a lightweight local Flask/FastAPI server on each terminal (or serve the exam UI from the edge server). The exam interface is trivially a web page: 10 MCQ questions, a timer, navigation buttons. The AI pipeline runs as a headless Python process on the terminal. This eliminates the entire Qt dependency chain, makes deployment a matter of "run this .bat file," and the exam UI renders identically everywhere. If you insist on PyQt6, have a browser-based fallback ready as a Day 0 contingency.

2. **Move multi-terminal testing to Day 4-5.** After the vertical slice on Day 3-4, immediately test with 3 terminals. Even if the dashboard isn't pretty yet, verify that events flow correctly from multiple sources concurrently. Discover LAN issues early.

3. **Add a minimal CCTV integration.** On the edge server, accept an RTSP stream from one camera, display it on the dashboard as a live feed with a simple OpenCV background subtraction overlay (motion highlights). No AI, no detection — just visual display. This takes ~2-3 hours and checks the "CCTV integration" box without risk. Mark it as "Phase 1: visual monitoring; Phase 2: AI-powered body tracking" in the scalability roadmap.

4. **For seat swaps, implement cross-terminal face matching.** When a FACE_MISMATCH is detected on Terminal A, the edge server should compare the mismatched embedding against all enrolled embeddings across all terminals. If it matches Terminal B's enrollment, flag CONFIRMED_SEAT_SWAP with both terminal IDs. This is a ~30-minute implementation on the edge server and will be a standout moment during the demo.

5. **Drop SUSPICIOUS_POSTURE or redefine it narrowly.** Head pitch alone is too noisy. Either: (a) only fire it when pitch >30° AND the student hasn't interacted with the exam UI for >15 seconds (requires exam UI to report last-interaction timestamps), or (b) cut it entirely and focus on the 5 alerts you can do reliably. 5 high-quality alerts beat 6 alerts where one is a false-positive machine.

6. **For audio, lower the threshold to 5 seconds but add a minimum energy gate.** Silero VAD + a simple RMS energy threshold above ambient baseline (calibrated during the first 30 seconds of the exam) would let you catch 5-second whispered conversations while rejecting low-energy ambient noise. 10 seconds is too conservative for the demo.

7. **Capture at 640x480, not 720p.** Explicitly set `cap.set(cv2.CAP_PROP_FRAME_WIDTH, 640)`. This halves the pixel count, reduces memory usage, and BlazeFace is designed for this resolution range. On Celerons, this could be the difference between hitting 12fps and hitting 6fps.

8. **Add a "system health" pre-exam check** — a 30-second automated diagnostic that verifies webcam capture, microphone input, model inference speed, and network connectivity to the edge server. Display pass/fail on each terminal. Run this before every demo session. This both prevents embarrassing failures and demonstrates operational maturity to judges.

9. **Pre-bake a fallback video demo.** On Day 9, record a flawless 3-minute screen capture of the entire flow working: enrollment → exam → malpractice detection → dashboard alerts → confusion matrix. If anything fails on demo day, you can narrate over the video while fixing the issue. This is standard hackathon survival strategy.

10. **Add a one-slide "10,000 center scalability" architecture diagram.** Show how the edge-server-per-hall model scales: each hall is autonomous, metadata aggregates to a district/state dashboard via 4G. Include cost breakdown (~₹25,000 for mini-PC + SSD + CCTV camera per hall). This addresses the jury's scalability evaluation criterion directly.

KEY_TRADEOFFS:

1. **PyQt6 native app vs browser-based UI:** The proposal chose PyQt6 to save RAM (no Electron), but this trades deployment simplicity for memory efficiency. On 4GB machines, the RAM savings matter — but if the app won't *launch* due to deployment issues, the RAM savings are irrelevant. A lightweight local web server + browser tab would use ~150-200MB (less than Electron's 300-500MB) while being dramatically easier to deploy. **I believe this is the wrong tradeoff for a hackathon on unknown hardware.**

2. **Cutting CCTV entirely vs. minimal integration:** The proposal correctly identified CCTV AI processing as high-risk, but went too far by cutting it entirely. A "display-only" CCTV feed on the dashboard costs almost nothing and satisfies a stated requirement. **The tradeoff of zero CCTV risk vs. zero CCTV compliance is too aggressive.**

3. **6 alert types vs. fewer, higher-quality alerts:** The proposal includes SUSPICIOUS_POSTURE which will likely be the noisiest alert. The tradeoff is coverage breadth vs. confusion matrix quality. Given that the jury *explicitly* wants minimal false positives and will compute a confusion matrix, **a false-positive-generating alert type is worse than no alert at all.** Cut or significantly constrain it.

4. **Episode-based events vs. per-frame reporting:** This is the **right** tradeoff. Episode-based reduces network load, simplifies the dashboard, and produces cleaner evidence. The cost is slightly delayed initial detection (need to accumulate frames before declaring an episode), but 3-5 seconds of delay is invisible in a proctoring context.

5. **Local-only processing vs. cloud-assisted:** This is the **right** tradeoff for the demo. No cloud dependency means no demo-day failure from bad internet. The scalability story (local processing per hall, only metadata to cloud) is actually *more* compelling to a government buyer than a cloud-dependent architecture.

6. **Conservative thresholds vs. aggressive detection:** The proposal leans conservative (10s audio, 30s gaze, 3-zone hysteresis). For a hackathon demo with *staged* malpractice events (which will be obvious and sustained), slightly more aggressive thresholds would catch more events. The risk is that if the jury's staged events are subtle or brief, conservative thresholds will miss them. **Tune thresholds for the demo scenario (staged, obvious events), not for production (subtle, adversarial events).** You can always tighten them later.

7. **Schedule back-loading:** The proposal front-loads individual component development and back-loads integration testing. This is the **wrong** tradeoff for a hackathon. Integration issues are the #1 killer. A vertical slice by Day 3-4 is good, but multi-terminal testing needs to happen by Day 5, not Day 8.

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
The core shape (terminal-side lightweight inference + edge-side aggregation/dashboard + episode-based alerts + no raw video uplink) is architecturally sound for a 7–10 day hackathon demo. It optimizes for the two things that usually kill demos: unstable networks and over-ambitious real-time video streaming.

What works and is likely to impress the jury:
- **Episode-based events + evidence-on-demand**: This directly addresses the explicit “minimize false alarms” guidance and keeps the dashboard usable.
- **Degradation modes**: Very pragmatic for unknown CPU/webcam quality; also a strong “scalability story.”
- **Ground-truth logger + confusion matrix**: This is unusually aligned with the evaluation rubric; most teams will hand-wave metrics.
- **No cloud dependency**: Strong government-friendly operational narrative (DPDP/privacy, cost, offline resilience).

Where it’s not “battle-ready” yet is not the model choices—it’s the **demo-day operational risk** and a couple of spec-expectation gaps:
- The official spec expects **CCTV integration** and **tamper-proof logs**. You can downscope these, but you must still show *something credible* or risk losing points to teams that demo even a thin “integration stub.”
- The “8/10” target is achievable only if you **control false positives**. Your current alert set includes two high-FP candidates in real classrooms: **gaze** and **posture** (especially posture). If FPs flood the proctor view, you’ll lose even if you catch 8/10 staged events.

Scope vs team/time:
- For **1 developer**: this is too much (PyQt exam app + multi-threaded AI + store/forward + time sync + edge server + dashboard + evidence viewer + fusion + metrics). You’ll end up with many “almost done” components.
- For **2 developers**: feasible if you lock the vertical slice by Day 4 and aggressively cut anything that doesn’t increase the probability of hitting 8/10 with low FP.

Biggest differentiation vs “everyone uses MediaPipe/MobileFaceNet”:
- Not the models—**the evaluation harness** is your moat: episode semantics, live ground-truth capture, confusion matrix, and “degradation transparency” (showing when/why the system sheds modalities). That looks like a deployable product rather than a model demo.

CONCERNS:
1. **Single most likely demo-day failure: deployment/device I/O (webcam+mic+ONNX) on unknown Windows machines.**  
   - OpenCV camera index issues, DirectShow quirks, mic permissions, missing VC++ runtimes, CPU lacking AVX, ONNXRuntime provider mismatch—any one of these can stall the entire hall demo.
2. **CCTV integration is fully cut, but the challenge explicitly lists it as a component.**  
   - Even if the orientation deprioritized facial privacy from CCTV, the jury may still score “integration readiness.” A hard “we don’t do CCTV” can be a points loss relative to competitors.
3. **“Tamper-proof logs” is in the spec; you cut hash-chaining.**  
   - For hackathon you don’t need blockchain—yet you need at least a minimal tamper-evident story or you look incomplete.
4. **SQLite under concurrent writes (30 terminals) can stall or wedge without careful write serialization.**  
   - WAL helps, but contention plus a naive “write on every event” can still bite. Demo-day failures here look like “dashboard lag” or missing events.
5. **Timestamp accuracy risk is underestimated.**  
   - Ground-truth matching depends on consistent clocks and deterministic buffering. Multi-threaded capture + inference + network retries can easily introduce 2–10s skew unless you standardize “event time = capture time” and carry offsets properly.
6. **GAZE_DEVIATION and SUSPICIOUS_POSTURE are likely FP generators in real exam posture/camera placements.**  
   - Low webcams, students naturally looking down to read questions, different seating heights, glasses glare—these create noisy head-pose estimates. High FP directly conflicts with “minimize false alarms.”
7. **Audio VAD without robust hall-noise handling can cause “everyone is flagged” moments.**  
   - Fan noise won’t trigger Silero VAD much, but classroom speech, announcements, and neighbor talk can. Your common-mode rejection helps, but the thresholding needs to be extremely conservative for the demo.
8. **CSRT can be too heavy; tracker failures cascade into face mismatch false alarms.**  
   - If tracking drops frequently, you’ll over-trigger re-verification and can oscillate between uncertain/mismatch.
9. **Push “encrypted JPEG” is easy to promise and hard to implement correctly in 7–10 days.**  
   - If you don’t actually encrypt in transit/at rest, don’t claim it; the jury may ask.
10. **1–2 fps gaze estimation is a red flag for “real-time.”**  
   - Even if architecturally acceptable, the dashboard experience may look laggy unless you message it correctly (episode-based, not per-frame).

SUGGESTIONS:
1. **Do a thin “CCTV integration stub” (2–4 hours, not a full product):**
   - Edge server ingests RTSP/USB camera feed (even a laptop webcam pointed backward) and runs **background subtraction + person-count blobs** or simple motion heatmap.
   - Expose `/cctv/status` + one alert type like **AISLE_ACTIVITY** or **GROUP_MOVEMENT**.  
   - The goal is scoring + narrative (“we integrate CCTV, privacy-safe because back-facing, and we only emit metadata”), not perfect detection.
2. **Add minimal tamper-evidence with hash chaining to satisfy the spec (1–2 hours):**
   - For each JSONL event: `hash_i = sha256(hash_{i-1} + canonical_json(event))`
   - Store `hash_i` in the line. This gives a credible “tamper-evident log” story without heavy infra.
3. **Cut or gate posture for the demo to protect FP rate:**
   - Make **SUSPICIOUS_POSTURE** “fusion-only” (requires POSTURE + prolonged ABSENT-from-face-center or POSTURE + GAZE), or keep it disabled unless the jury explicitly stages “phone in lap.”
4. **Make gaze calibration explicit and short:**
   - First 10 seconds: collect baseline yaw distribution while candidate looks at screen. Use robust stats (median + MAD) instead of a single baseline value.
   - Also cap how often gaze can alert (cooldown) to prevent alert storms.
5. **Prioritize the 4 alerts that most reliably hit staged scenarios:**
   - CANDIDATE_ABSENT, MULTIPLE_PERSONS, FACE_MISMATCH, (conservative) SUSPICIOUS_AUDIO.  
   - Treat gaze as “bonus” unless you can prove low FP in the actual room.
6. **SQLite: enforce single-writer queue on the edge server:**
   - One async task/thread that batches inserts every 250–500ms.
   - Everything else publishes to an in-memory queue. This avoids demo-day “database locked” surprises.
7. **Timestamping: standardize on capture-time + server offset:**
   - Terminal stamps `t_capture_monotonic` and `t_capture_wall` (if available).
   - Periodically measure `(server_time - terminal_wall_time)` with RTT compensation; store offset and include both raw and corrected times in events.
8. **Deployment strategy: pick the option with the fewest unknown dependencies**
   - For hackathon reality: **WinPython portable** (with pinned wheels, ONNXRuntime included, OpenCV included) is often safer than PyInstaller when you can’t predict DLL/runtime issues.
   - Keep a “device diagnostic” executable/script that tests: camera open, mic open, ONNX inference, write permissions, LAN reachability—this should be your Day 0 gate.
9. **Differentiate in the demo script, not just architecture:**
   - Show “Degradation Status” tile live (CPU load/FPS → which modalities are active).
   - Run a 10-minute session, then immediately show confusion matrix populated from your ground-truth logger. Many teams won’t have that closed loop.

KEY_TRADEOFFS:
- **You traded model sophistication for operational reliability (good for hackathon).** The right call—winning is about a working end-to-end system with credible metrics, not best-in-class CV.
- **You traded full compliance scope (CCTV + tamper-proof) for speed (risky).** This is the main strategic risk: even a thin stub would reduce scoring downside.
- **You traded continuous monitoring for episode semantics (good).** This reduces alert spam and makes the dashboard usable, aligning with “minimize false positives.”
- **You traded “catch everything” for “catch staged events reliably.”** That’s correct for ≥8/10, but only if you aggressively control FP-heavy modalities (gaze/posture/audio) and ensure timestamp integrity.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The architecture is technically sound for the hardware constraints (i5/4GB RAM) and correctly identifies bandwidth as a bottleneck by processing locally. The "Fusion Rules" at the Edge Server are a winning differentiator—they directly address the "Low False Positive" requirement from the orientation. Dropping Electron for PyQt6 saves crucial RAM.

However, the proposal carries **existential operational risks** regarding deployment and audio physics that could cause a total demo failure. The plan relies on `WinPython` portable setups working flawlessly on 30 unknown machines, which is optimistic. Additionally, processing audio on 30 terminals in a single room without advanced noise cancellation/source separation will likely trigger `SUSPICIOUS_AUDIO` on *everyone* whenever *anyone* speaks, destroying your precision metrics.

CONCERNS:
1.  **Deployment "Dependency Hell" (Critical):** relying on `WinPython` requires the target machines to have specific VC++ redistributables and driver states. If 5 out of 30 machines fail to launch due to a missing DLL, you start the demo in panic mode.
2.  **Audio "Crosstalk" False Positives:** In a quiet room with 30 mics, one person talking can trigger VAD on neighbor terminals. If the "Common-mode audio reject" isn't perfectly tuned (hard to do in 2 days), you will flag innocent students.
3.  **PyQt6 UI Velocity:** You have 1-2 developers. Building a functional, decent-looking "Exam Interface" (MCQ rendering, timer, navigation) in PyQt6 takes 3x longer than HTML/CSS. You risk running out of time on the *exam* part, making the *proctoring* look useless.
4.  **CCTV Compliance Gap:** The challenge explicitly asks for CCTV integration (Component 1). Cutting it entirely ("high demo-failure risk") is a strategic gamble. If a judge insists on seeing the CCTV feed on the dashboard, you have zero answer.
5.  **Ground Truth Synchronization:** You need to match AI logs to human logs to prove 8/10 accuracy. If the human logger (you) and the system clocks are off by even 2 seconds, your "Confusion Matrix" will be invalid.

SUGGESTIONS:
1.  **Switch to PyInstaller (`--onedir` mode):** Do not rely on WinPython. Build a self-contained executable that bundles *everything*. Test this on a fresh Windows sandbox immediately.
2.  **Implement "Loudest-Mic-Wins" or Drastic Audio Gating:** For the audio fusion, the Edge Server should compare timestamps of VAD triggers. If Terminal A and Terminal B trigger within 100ms, but Terminal A's amplitude is higher, ignore Terminal B. Or, simplify: Only flag audio if it persists >10s *and* is validated by lip movement (face mesh).
3.  **Hybrid UI (Webview):** Consider using `PyQt6.QtWebEngineWidgets` to render the Exam MCQs using local HTML/JS. Keep the AI in Python, but let the UI be simple HTML. This speeds up "Exam" development significantly.
4.  **Fake the CCTV Stream:** Do not process CCTV for AI, but *do* ingest the RTSP stream and display it on the Dashboard. This checks the "Integration" box without the AI risk.
5.  **Automated Ground Truth:** Build a "Malpractice Trigger" app for the actors. When an actor commits "Gaze Deviation," *they* press a hidden button (or use a remote clicker) that logs the *exact* intended malpractice time to the Edge Server. Do not rely on a human observer to log 30 people manually.

KEY_TRADEOFFS:
**PyQt6 vs. Web:** You are trading development speed (Web is faster) for system resource control (PyQt is lighter). Given the 4GB RAM constraint, this is the **correct** tradeoff, provided you keep the UI very simple.
**Fusion vs. Raw Speed:** You are delaying alerts by 5-10 seconds to process "Fusion Rules" (e.g., waiting for Gaze + Audio). This trades "instant" alerting for "accurate" alerting. This is the **correct** tradeoff for winning; judges hate flashing red lights that mean nothing.
**Edge Server vs. Cloud:** You are trading scalability (harder to set up LAN) for reliability (immune to bad internet). This is the **winning** tradeoff for a live demo in India.

---
## Round 2

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After reviewing both GPT-5.2 and Gemini 3 Pro's feedback, I'm struck by the strong convergence across all three reviewers on the critical issues. My position hasn't fundamentally changed, but several arguments have sharpened my thinking and I want to update accordingly.

**Where all three reviewers converge (high-confidence consensus):**

1. **CCTV must have a minimal stub.** All three of us flagged this independently. GPT-5.2's framing is the most precise: a 2-4 hour RTSP ingest + background subtraction + motion heatmap on the dashboard. This isn't optional — it's a stated requirement, and having *zero* answer when a judge asks "where's the CCTV?" is an unforced error. I fully endorse this as a must-do.

2. **SUSPICIOUS_POSTURE is a false-positive bomb.** All three of us identified this. GPT-5.2's suggestion to make it fusion-only (require POSTURE + GAZE or POSTURE + absence-from-face-center) is the best mitigation I've seen. I retract my earlier suggestion to drop it entirely — making it fusion-only preserves coverage while protecting the confusion matrix. This is a better answer.

3. **Audio crosstalk in a single room is a physics problem, not a software problem.** Gemini's point about 30 mics in one room is devastating and I underweighted it in my initial review. My suggestion of lowering the threshold to 5 seconds would actually make this *worse*. I now think the audio strategy needs a more fundamental rethink: either (a) Gemini's "loudest-mic-wins" amplitude comparison at the edge server, or (b) GPT-5.2's suggestion to keep audio extremely conservative and treat it as "bonus" detection. For the demo specifically, given that the jury will stage events in a single room, audio will be the noisiest channel. I'd now recommend: keep the 10s threshold, add amplitude-based common-mode rejection at the edge, and treat SUSPICIOUS_AUDIO as a low-priority alert that contributes to fusion but rarely fires standalone. **I was wrong to suggest lowering to 5 seconds.**

4. **Deployment on unknown machines is the #1 demo-day risk.** Universal agreement. However, I now disagree with Gemini's PyInstaller suggestion.

**Where I disagree with the other reviewers:**

1. **Gemini's PyInstaller recommendation is wrong for this scenario.** PyInstaller `--onedir` bundles everything into a single directory, yes — but it also bundles the Python interpreter, and the resulting package for this stack (PyQt6 + ONNX Runtime + OpenCV + Silero + numpy + scipy) would be 800MB-1.2GB per terminal. Deploying that to 30 machines over LAN is a significant time cost. Worse, PyInstaller is notorious for antivirus false positives — the bootloader executable gets flagged by Windows Defender more often than a plain Python distribution. WinPython portable, while imperfect, at least lets you debug dependency issues interactively. **My updated recommendation: WinPython portable as primary, with a pre-built PyInstaller `--onedir` as a fallback on a USB drive. Test both on Day 0.**

2. **I'm softening on PyQt6 vs. browser-based UI, but not retracting.** Gemini's `QtWebEngineWidgets` suggestion is clever but actually makes things worse — QtWebEngine adds ~200MB to the distribution and is essentially bundled Chromium, giving you the worst of both worlds. GPT-5.2 didn't push hard on the UI question. I still believe a browser-based exam UI served from the edge server (with the AI pipeline as a headless process) is the lower-risk path for deployment. But I acknowledge that PyQt6 gives you tighter control over webcam/mic access without browser permission dialogs, which matters for the demo. **Updated position: if the team has PyQt6 experience, keep it but build the exam MCQ portion as a simple QWebEngineView-free layout (just QRadioButtons + QPushButtons). If the team is more comfortable with web, switch to browser-based. The deciding factor is team familiarity, not architecture.**

3. **GPT-5.2's hash-chaining suggestion is worth the 1-2 hours.** I initially agreed with cutting hash-chain tamper-proof logs, but GPT-5.2 is right that `hash_i = sha256(hash_{i-1} + event_json)` is trivially implementable and directly addresses a stated requirement. This is a 30-line addition. It should go back in.

**What GPT-5.2 added that I missed:**

- **Timestamp standardization** is more important than I initially recognized. The `t_capture_monotonic` + server offset approach is correct. Without this, the confusion matrix — your primary evaluation artifact — will have misaligned entries. This needs to be a Day 2-3 priority, not an afterthought.

- **SQLite write serialization via an async batching queue** is the right pattern. My concern about `SQLITE_BUSY` was correct but my suggestion (retry with backoff) was the naive solution. A single-writer queue that batches every 250-500ms is cleaner and eliminates the problem structurally.

**What Gemini added that I missed:**

- **The "Malpractice Trigger" app for actors** is brilliant and neither GPT-5.2 nor I thought of it. Having the person staging the malpractice press a button at the exact moment they begin acting creates microsecond-accurate ground truth. This eliminates the clock synchronization problem between the human observer and the system entirely. It also looks incredibly professional during the demo. This should be a simple mobile web page served from the edge server with 6 big buttons (one per malpractice type) that logs `{terminal_id, malpractice_type, timestamp}`.

**My updated assessment of the 8/10 target:**

With the audio crosstalk issue properly understood, and posture as a FP risk, the reliable detection set is:
- CANDIDATE_ABSENT: near-certain (Very High)
- MULTIPLE_PERSONS: near-certain (Very High)
- FACE_MISMATCH: high if cross-terminal matching is implemented (High)
- GAZE_DEVIATION: medium-high with proper baseline calibration (Medium-High)
- SUSPICIOUS_AUDIO: medium at best in a shared room (Medium)
- SUSPICIOUS_POSTURE (fusion-only): low standalone, medium in fusion (Medium)

For 8/10 staged events, assuming the jury stages a mix: 2 absences, 2 extra persons, 1-2 seat swaps, 2-3 gaze/talking/phone events — you'd reliably catch 5-6 from the first three categories and need 2-3 more from the noisier categories. It's tight. **The cross-terminal seat-swap detection I suggested earlier is now even more critical — it could be the difference between 7/10 and 8/10.**

CONCERNS:

1. **Audio crosstalk in a 30-terminal single-room setup** is now my top concern, elevated from my original list. Without amplitude-based common-mode rejection working correctly, audio becomes a net-negative feature that generates more false positives than true detections. The edge server MUST compare VAD trigger timestamps across terminals and suppress common-mode activations.

2. **Cross-terminal face matching for seat swaps is unaddressed in the architecture.** All three reviewers agree seat swaps are a marquee demo scenario. The edge server needs all enrollment embeddings and must cross-check on FACE_MISMATCH events. This is ~30 minutes of implementation but architecturally significant — it means enrollment embeddings must be pushed to the edge server, not just stored locally.

3. **Deployment remains the existential risk.** All three reviewers agree. The Day 0 GO/NO-GO gate is correct but must be expanded: test on a *non-dev Windows machine* with a fresh user account, ideally one with restricted permissions similar to government desktops.

4. **Multi-terminal integration testing at Day 8 is too late.** All three reviewers converge on this. Move to Day 4-5 with at least 3 terminals.

5. **CCTV stub is mandatory.** Consensus across all reviewers. Budget 2-4 hours on Day 6-7.

6. **Hash-chaining for tamper-evident logs should be re-added.** 30 lines of code, directly addresses a stated requirement. No reason to cut it.

7. **Gaze baseline calibration needs an explicit 10-second capture protocol** during enrollment, using median + MAD as GPT-5.2 suggested, not a single snapshot.

8. **The SUSPICIOUS_POSTURE alert must be gated** — either fusion-only (GPT-5.2's suggestion) or gated by UI inactivity (my original suggestion). Standalone head-pitch alerts will destroy precision.

SUGGESTIONS:

1. **Implement Gemini's "Malpractice Trigger" app.** A mobile-friendly web page served from the edge server with 6 buttons + terminal selector. Actors press the button when they begin staging a malpractice. This creates perfect ground truth and is a 2-3 hour build that dramatically improves the confusion matrix reliability. This replaces or supplements the admin ground-truth logger.

2. **Implement cross-terminal seat-swap detection on the edge server.** Push enrollment embeddings to edge during enrollment. On any FACE_MISMATCH event, compare the mismatched embedding against all enrolled embeddings. If it matches another terminal's enrollment, emit CONFIRMED_SEAT_SWAP with both terminal IDs. This is a demo showstopper feature.

3. **Add CCTV display stub.** RTSP ingest → background subtraction → motion heatmap overlay → display on dashboard. No AI detection, just visual monitoring with highlights. 2-4 hours, checks the compliance box.

4. **Re-add hash-chaining** for tamper-evident JSONL logs. `hash_i = sha256(hash_{i-1} || canonical_json(event_i))`. Store the running hash in each log line. 30 minutes of implementation.

5. **Revise the audio strategy:** Keep 10s standalone threshold. Add amplitude-ranked common-mode rejection at the edge: if >3 terminals trigger VAD within 500ms, suppress all but the highest-amplitude source. This is the only way audio works in a shared room.

6. **Make SUSPICIOUS_POSTURE fusion-only.** Never fire it standalone. Only emit when combined with GAZE_DEVIATION (→ POSSIBLE_MATERIAL_USE) or with UI inactivity >15s.

7. **Revised schedule priorities:**
   - Day 0: GO/NO-GO on non-dev machine (unchanged, but add restricted-user test)
   - Day 1-3: Core pipeline + enrollment + verification + edge server basics
   - Day 4: Vertical slice with **3 terminals** (moved from Day 8)
   - Day 5: Dashboard + exam UI + ground-truth logger + malpractice trigger app
   - Day 6: Evidence push + CCTV stub + hash-chaining + fusion rules + seat-swap detection
   - Day 7: Confusion matrix + audio tuning + threshold calibration
   - Day 8: 10+ terminal stress test + threshold tuning on real hardware
   - Day 9: Full calibration exam + record fallback video
   - Day 10: Rehearsal only

8. **Prepare the scalability one-pager:** Architecture diagram showing edge-server-per-hall model scaling to 10,000 centers. Cost breakdown: ~₹25,000 per hall (mini-PC + SSD + CCTV). Metadata aggregation to district/state dashboard over 4G. This directly addresses jury evaluation criteria.

9. **Capture at 640x480.** Explicitly set resolution in OpenCV. Non-negotiable on 4GB Celeron machines.

10. **Pre-bake the fallback video on Day 9.** All three reviewers implicitly or explicitly support having a backup. Record a flawless 3-minute walkthrough of the entire flow.

KEY_TRADEOFFS:

1. **Audio conservative threshold (10s) + common-mode rejection vs. aggressive threshold (5s):** After Gemini's crosstalk analysis, the conservative approach is clearly correct for the demo environment. The tradeoff is missing brief whispered exchanges vs. avoiding mass false positives across 30 terminals in one room. **Updated: conservative is the right call. I was wrong earlier.**

2. **SUSPICIOUS_POSTURE as fusion-only vs. standalone:** Trades detection coverage (miss standalone phone-in-lap) for precision (eliminate the highest-FP alert type). Given the jury's explicit emphasis on minimal false positives, **fusion-only is the right call.** GPT-5.2 had the best formulation here.

3. **Hash-chaining re-added vs. keeping it cut:** Trades ~30 minutes of dev time for compliance with a stated requirement. This is trivially worth it. **All reviewers should converge on re-adding this.**

4. **Cross-terminal seat-swap detection vs. terminal-local-only matching:** Trades slight architectural complexity (enrollment embeddings on edge server, cross-matching logic) for the ability to catch the marquee demo scenario. **This is the single highest-ROI feature addition in the entire debate.** 30 minutes of implementation, potential demo-winning moment.

5. **CCTV display stub vs. no CCTV:** Trades 2-4 hours of development for compliance with an explicitly stated requirement. **Every reviewer agrees this is worth it.** The only question is timing — I suggest Day 6.

6. **Multi-terminal testing at Day 4-5 vs. Day 8:** Trades slightly less feature completeness at the time of integration testing for dramatically earlier discovery of deployment/networking issues. **All three reviewers converge: earlier is mandatory.** This is perhaps the single most important schedule change.

7. **WinPython portable vs. PyInstaller:** I disagree with Gemini here. WinPython is more debuggable on unknown machines; PyInstaller has worse antivirus false-positive rates. **Carry both as options, test both on Day 0, deploy whichever works on the target hardware.**

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position does not change on the *core* architecture: terminal-side inference + edge-side aggregation/dashboard + episode semantics + degradation + store-and-forward is the right backbone to survive a live demo and score well on “low false positives” + confusion matrix.

What *did* change after reading others: I now think **the terminal UI/deployment choice (PyQt6 + portable Python)** is a larger existential risk than I previously weighted. Claude and Gemini are directionally right: on unknown Windows desktops, the probability that *something* in Qt/VC++/AV/permissions breaks on a subset of machines is high, and any per-terminal failure reduces your 30-seat credibility quickly.

I also agree with Claude’s point that **seat swap is the marquee scenario** and the current writeup does not clearly guarantee detection if both candidates are validly enrolled (swap between terminals). Without **cross-terminal matching**, you can miss the exact scenario the jury is likely to stage.

Where I disagree with other participants:
- I *don’t* agree that “PyQt is the correct tradeoff if you keep UI simple” (Gemini). For this hackathon, **launch reliability > RAM efficiency**. A 150–250MB Chrome/Edge tab is a better failure mode than a Qt runtime that may not start at all.
- I’m not convinced **PyInstaller** is universally safer than WinPython. PyInstaller often introduces its own DLL/path/AV heuristics issues. The safer path is to **support two packaging options** and choose after Day-0 hardware reality check, not to bet everything on one.

Common ground across all feedback:
- Biggest demo risk is **operational** (deployment, multi-terminal scale, time sync), not model accuracy.
- **CCTV cannot be totally absent**; a minimal “integration stub” is worth it.
- **Audio and posture/gaze are FP traps** unless constrained/fused.
- Integration testing must move earlier.

CONCERNS:
1. **Terminal launch reliability (highest risk):** PyQt6 + portable Python on locked-down Windows + unknown drivers/AV policies can fail to start on multiple terminals.
2. **Seat-swap detection gap:** Without edge-side cross-enrollment matching, swapping two enrolled candidates between terminals can evade “FACE_MISMATCH” (or be misclassified).
3. **Audio crosstalk false positives in a 30-mic room:** One student speaking can trigger many mics; “common-mode reject” is hard under clock skew and differing mic gains.
4. **CCTV requirement optics:** Completely omitting CCTV can cost points even if you explain “risk.” Judges may score against the written spec.
5. **Ground-truth/timestamp validity:** If your time alignment is off by seconds, confusion matrix credibility collapses.
6. **Posture and gaze false positives:** Head pitch and head yaw are very sensitive to camera mounting height, screen position, and natural exam behavior (looking down to read/think).
7. **Edge persistence under burst:** SQLite can still hit `SQLITE_BUSY` unless you serialize writes and batch.
8. **Late multi-terminal testing (schedule risk):** Finding LAN/firewall/backpressure issues on Day 8 is too late.

SUGGESTIONS:
1. **Decouple terminal UI from the AI agent (primary change):**
   - Run a **headless “agent.exe”** (or `agent.bat`) that does capture/inference/spooling.
   - Serve the exam UI as **plain HTML/JS** (either from the edge server or a tiny local server), opened in the existing browser (Edge/Chrome).
   - Keep PyQt only as an optional fallback if you insist—but don’t make it the single path to starting the exam.
2. **Implement explicit seat-swap detection (high ROI differentiator):**
   - During enrollment, store an embedding per candidate on the edge (or per-terminal with periodic sync).
   - On mismatch/uncertain, edge compares the embedding against *all enrolled* and emits **CONFIRMED_SEAT_SWAP(terminal_A, terminal_B)** when it best-matches another candidate strongly.
   - This is a “wow” moment in the demo and directly targets a likely staged event.
3. **Add CCTV “checkbox compliance” with minimal risk:**
   - Edge dashboard shows CCTV feed (RTSP/USB) + optional motion heatmap/background subtraction overlay.
   - One alert type like **AISLE_ACTIVITY** is enough; don’t attempt person re-ID/body pose.
4. **Constrain audio to avoid destroying precision:**
   - Make audio **fusion-only** by default (e.g., AUDIO contributes only if paired with gaze/face absence) *or* require (a) sustained VAD + (b) above-baseline RMS energy.
   - If you keep standalone audio: keep threshold conservative for demo and add per-terminal gain normalization.
5. **Move multi-terminal integration earlier:**
   - By **Day 5**, run 3–5 terminals end-to-end (events + dashboard + DB + reconnect). Don’t wait for “pretty UI.”
6. **Time sync: make it boring and auditable:**
   - Always log **capture timestamp** at terminal; compute server offset periodically; store both raw and corrected times.
   - In the dashboard, show “clock offset” per terminal so judges see you’re controlling the variable.
7. **Packaging strategy: don’t bet on one tool**
   - Prepare **two deployables** by Day 2: (a) WinPython portable folder, (b) PyInstaller `--onedir`.
   - Day 0/1 hardware test decides which is used on demo day.

KEY_TRADEOFFS:
- **Qt native app vs browser UI:** Qt can be RAM-efficient, but the hackathon-winning metric is “starts everywhere.” Browser UI trades some RAM for dramatically higher deployment reliability and faster UI iteration.
- **Audio sensitivity vs false positives:** Lower thresholds catch more cheating but risk flagging the whole room. Given the explicit “minimize false alarms,” it’s better to be conservative and use fusion than to chase recall with noisy audio.
- **CCTV AI vs CCTV integration stub:** Full CCTV analytics is high risk; a feed + motion overlay is low risk and satisfies spec optics. This is the right compromise for demo day.
- **More alert types vs cleaner confusion matrix:** Extra modalities (posture/gaze) can improve recall but can also tank precision. For winning, precision and credible metrics likely beat breadth.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
The panel has reached a strong consensus: this architecture is **conceptually winning** (Local AI, Episode-based events, Fusion Rules) but **operationally fragile**.

I agree with **Claude Opus** that the **PyQt6 Exam UI** is a strategic error. Building a polished MCQ interface in Qt Widgets is a massive time sink for 1-2 developers compared to HTML/CSS. A "Headless Python Agent + Browser UI" approach solves the velocity problem while keeping the AI logic in Python.

I agree with **GPT-5.2** that cutting **CCTV** entirely is a non-starter. The challenge explicitly requires it. A "dumb" implementation (display-only) is infinitely better than zero implementation.

I stand by my original concern regarding **Audio Crosstalk** and **Deployment Hell**. The proposed 10-second audio threshold is a "safe" choice that renders the feature useless for detecting real whispers (which last 2-3s).

CONCERNS:
1.  **PyQt6 "Velocity Trap":** (Aligned with Claude) You will spend 50% of your time fighting Qt layouts for the Exam UI instead of tuning the AI. The UI needs to look professional; Qt makes that hard.
2.  **The "Seat Swap" Logic Gap:** (Aligned with Claude) If Terminal A detects a mismatch, the current architecture flags "Imporsonation." To *win*, it must identify that the intruder is actually the candidate from Terminal B. The current design misses this cross-terminal lookup.
3.  **CCTV Compliance:** (Aligned with GPT-5.2) Completely ignoring Component 1 (CCTV) gives judges an easy reason to disqualify you, even if the rest is perfect.
4.  **Day 8 Integration Suicide:** (Aligned with Claude) Testing multi-terminal networking on Day 8 is too late. LAN congestion and SQLite locking will appear immediately at scale.
5.  **Audio Thresholding:** 10s is too long (misses cheating). 3s is too short (false positives). Without energy gating or cross-terminal volume comparison, audio remains a liability.

SUGGESTIONS:
1.  **Architecture Pivot: Headless AI + Browser UI:**
    *   **Exam UI:** A simple HTML/JS app served locally (FastAPI/Flask) or opened as a file in the system browser (Chrome/Edge). This guarantees a clean UI and saves days of dev time.
    *   **AI Agent:** Runs as a background Python process (system tray icon). It handles the webcam, models, and WebSocket comms to the Edge Server.
    *   **Overlay:** If you need to lock the screen, use a simple full-screen browser kiosk mode.

2.  **Implement the "CCTV Stub":**
    *   On the Edge Server, ingest the RTSP stream.
    *   Display it on the Dashboard.
    *   **Do not** run AI on it. Just show it. This checks the "Integration" box.

3.  **Smart "Seat Swap" Detection:**
    *   Update the Edge Server: When `FACE_MISMATCH` events arrive, compare the "intruder" embedding against the *global* enrollment list.
    *   Return: "ALERT: Candidate B is at Terminal A." This is a "Wow" moment for the demo.

4.  **"Loudest-Mic" Audio Gate:**
    *   Modify the Edge Server to buffer audio events for 500ms.
    *   If multiple terminals report `SUSPICIOUS_AUDIO` simultaneously, only flag the one with the highest amplitude/confidence.

5.  **Day 0 "Pre-Flight" Diagnostic Tool:**
    *   Build a standalone `.exe` that checks: Camera availability, Mic levels, VC++ Runtimes, and Write permissions.
    *   Run this on every machine immediately upon entering the hall to identify "dead" terminals before deployment.

KEY_TRADEOFFS:
**Browser UI vs. PyQt6:** You are trading **RAM efficiency** (Browser uses more) for **Development Velocity** (HTML is faster). given the timeline (7 days), Velocity is the correct tradeoff. The 4GB RAM can handle a single Chrome tab + ONNX.
**CCTV Stub vs. Full AI:** You are trading **Technical Depth** for **Checklist Compliance**. In a government hackathon, checking the box is more important than a feature that crashes the demo.
**Cross-Terminal Logic vs. Edge Simplicity:** You are adding complexity to the Edge Server (global matching) to gain a massive **Competitive Differentiator** (detecting specific seat swaps). This is worth the cost.

---
## Round 3

### Claude Opus



POSITION: PARTIALLY_AGREE

REASONING:

After three rounds of debate, the panel has achieved remarkable convergence. I'll focus this final response on crystallizing where we've landed, the one remaining genuine disagreement, and the prioritized action list.

**Full consensus items (all three reviewers agree — these are now architectural requirements):**

1. **Cross-terminal seat-swap detection on the edge server.** This is the single highest-ROI addition identified across the entire debate. Every reviewer independently flagged it. Implementation: enrollment embeddings pushed to edge, cross-matching on FACE_MISMATCH, emit CONFIRMED_SEAT_SWAP with both terminal IDs. ~1-2 hours of work, potential demo-winning moment. **Non-negotiable.**

2. **CCTV stub is mandatory.** RTSP ingest → display on dashboard + optional background subtraction motion overlay. No AI detection on CCTV feed. 2-4 hours. Checks compliance box for Component 1. **Non-negotiable.**

3. **SUSPICIOUS_POSTURE must be fusion-only.** Never fires standalone. Only contributes through fusion rules (POSTURE + GAZE → POSSIBLE_MATERIAL_USE). This protects the confusion matrix, which Anil sir explicitly said is a primary evaluation artifact. **Non-negotiable.**

4. **Audio needs common-mode rejection at the edge.** The 30-mic-in-one-room physics problem is real. Buffer VAD events for 500ms at the edge; if >3 terminals trigger simultaneously, suppress all but the highest-amplitude source. Keep 10s standalone threshold. **Non-negotiable.**

5. **Multi-terminal integration testing moves to Day 4-5.** Finding LAN/firewall/SQLite contention issues on Day 8 is a project-ending discovery. 3 terminals by Day 4, full vertical slice. **Non-negotiable.**

6. **Hash-chaining re-added.** 30 lines of code. `hash_i = sha256(hash_{i-1} || canonical_json(event_i))`. Directly addresses the "tamper-proof logs" requirement stated in the challenge. **Non-negotiable.**

7. **Pre-bake a fallback demo video on Day 9.** All reviewers support this as insurance. 3-minute flawless walkthrough of enrollment → exam → detection → dashboard → confusion matrix.

**The one remaining genuine disagreement: PyQt6 vs. Browser UI**

This is the only point where the panel hasn't fully converged, and I want to be precise about the actual tradeoff because I think we're talking past each other slightly.

GPT-5.2 and Gemini advocate for: headless Python agent (background process) + browser-based exam UI (HTML/JS served locally or from edge). The argument is deployment reliability and UI development velocity.

I previously said this depends on team familiarity. After further reflection, I want to update my position with more specificity:

**The browser-based approach has a hidden integration cost that the other reviewers are underweighting.** Specifically:

- The headless agent needs to communicate with the browser UI for: enrollment flow (capture frames, show quality feedback, confirm enrollment), exam state sync (timer, question navigation, submission), and webcam status indicators (green/red light showing monitoring is active). This requires either a local WebSocket server or a shared state mechanism. That's not trivial — it's a second communication protocol alongside the terminal→edge protocol.

- Browser webcam access via `getUserMedia` requires HTTPS or localhost, and the permission dialog will appear on every machine. In a government exam setting with 30 students, a single confused student clicking "Block" on the camera permission breaks that terminal's monitoring silently. PyQt6's `cv2.VideoCapture` has no permission dialog on Windows.

- The "kiosk mode" lockdown that Gemini mentions (full-screen browser) is fragile. Alt+F4, Alt+Tab, Ctrl+W all escape it. PyQt6 can intercept keyboard events at the OS level more reliably.

**However**, GPT-5.2's core point is valid: if PyQt6 fails to launch on multiple terminals due to missing VC++ runtimes or DLL conflicts, you lose the entire demo. This is a real risk.

**My final position on this:** The architecture should support **both paths** with a decision made on Day 0-1 based on the hardware reality check. The AI pipeline is Python regardless. The exam UI is a thin layer on top. Design the agent to expose a local HTTP API (`/api/enrollment/status`, `/api/exam/state`, `/api/webcam/frame`) from Day 1. If PyQt6 works on the target hardware, use it for tighter control. If it doesn't, pivot to browser UI consuming that same API. **The local HTTP API is the insurance policy, not a commitment to either UI framework.** This adds perhaps 2 hours of work but eliminates the existential deployment risk.

**On Gemini's concern about audio thresholds being too conservative:**

Gemini argues 10s is too long and misses real cheating (whispers last 2-3s). I understand the argument but strongly disagree for this specific demo context. In a single room with 30 terminals, a 3-second threshold will flag 10+ terminals every time any student coughs, mutters while reading, or the jury members discuss among themselves. The confusion matrix will be destroyed. For the demo, audio is a **bonus** channel that adds to fusion but should almost never fire standalone. If we catch one genuine AUDIO + GAZE fusion event during the demo, that's a win. If audio generates 15 false positives, that's a loss. **Conservative is correct here.**

**On deployment packaging (WinPython vs PyInstaller):**

All three reviewers now effectively agree: prepare both, decide on Day 0. I'll add one concrete suggestion — Gemini's "pre-flight diagnostic tool" idea is excellent and should be the very first thing built. Not as a standalone .exe (that introduces its own packaging problem), but as a simple Python script that can run from the WinPython portable distribution:

```python
# preflight.py — run on each terminal before deployment
import sys, os, json
checks = {}
try:
    import cv2; cap = cv2.VideoCapture(0); checks["camera"] = cap.read()[0]; cap.release()
except: checks["camera"] = False
try:
    import sounddevice; checks["mic"] = len(sounddevice.query_devices()) > 0
except: checks["mic"] = False
try:
    import onnxruntime; checks["onnx"] = True
except: checks["onnx"] = False
checks["ram_gb"] = round(os.sysconf('SC_PAGE_SIZE') * os.sysconf('SC_PHYS_PAGES') / (1024.**3), 1) if hasattr(os, 'sysconf') else "check manually"
checks["python"] = sys.version
checks["write_test"] = os.access(".", os.W_OK)
print(json.dumps(checks, indent=2))
```

Run this on every terminal within the first 15 minutes of setup. Identify dead machines immediately.

CONCERNS:

1. **The local HTTP API abstraction layer (UI-agnostic agent) must be designed from Day 1.** If the team builds deeply coupled PyQt6 code for 4 days and then discovers Qt won't launch on exam center machines, pivoting to browser UI without this abstraction layer means rewriting the entire UI integration. This is the one concern I want to elevate above all others because it's a process risk, not a technical risk.

2. **Cross-terminal seat-swap detection introduces an enrollment data flow that isn't in the current architecture.** Terminals must push embeddings to the edge during enrollment. The edge must store and index them. On FACE_MISMATCH, the edge must do a 1:N comparison (N=30, trivial compute but the data flow must be designed). This needs to be in the Day 2 enrollment implementation, not bolted on later.

3. **The CCTV stub needs a camera.** Has anyone confirmed whether the demo venue will have an IP camera available, or does the team need to bring one? A USB webcam on the edge server is a viable fallback (~₹1,500) but the RTSP ingest code would change to `cv2.VideoCapture(1)`. Test both paths.

4. **SQLite write serialization remains important.** GPT-5.2's async batching queue is the right pattern. With 30 terminals pushing events + the ground truth logger + the CCTV stub potentially writing motion events, the edge server's SQLite needs a single-writer queue. Use `asyncio.Queue` → batch writer coroutine that flushes every 250ms or on 50 pending events, whichever comes first.

5. **The malpractice trigger app (Gemini's idea, which I endorsed) needs network access from a mobile device on the same LAN.** This means the edge server's WiFi/LAN must be accessible from phones. In a school network, this might require a portable WiFi router. Budget ₹500-1,000 and bring one.

SUGGESTIONS:

**Prioritized implementation list (final, incorporating all three reviewers' convergence):**

**Tier 0 — Existential (must work or demo fails):**
1. Day 0 preflight diagnostic on non-dev machine (camera, mic, ONNX, write permissions)
2. UI-agnostic agent with local HTTP API (insurance against PyQt6 failure)
3. Core pipeline: BlazeFace → MOSSE → MobileFaceNet with CLAHE, running at stable FPS
4. Enrollment with quality gates + embedding push to edge server
5. Edge server with FastAPI + SQLite WAL + async write queue
6. Dashboard with terminal grid + alert feed (vanilla JS + SSE)
7. Multi-terminal vertical slice by Day 4 (3+ terminals)

**Tier 1 — Demo-winning (differentiators):**
8. Cross-terminal seat-swap detection (CONFIRMED_SEAT_SWAP with both terminal IDs)
9. Episode-based event semantics (not per-frame spam — this IS a differentiator vs competitors)
10. Confusion matrix with 1:1 greedy matching, displayed on dashboard
11. Malpractice trigger app for actors (mobile web page, 6 buttons)
12. FPS-based graceful degradation with status indicator on dashboard

**Tier 2 — Compliance (required but not differentiating):**
13. CCTV display stub on dashboard (RTSP or USB fallback)
14. Hash-chaining for tamper-evident JSONL logs
15. Store-and-forward with replay on reconnect
16. Evidence snapshot push (encrypted JPEG on high/critical alerts)

**Tier 3 — Polish (if time permits after Day 7):**
17. Common-mode audio rejection at edge
18. Post-exam summary report
19. Scalability one-pager for jury
20. Gaze baseline calibration protocol (10s during enrollment)
21. Fallback demo video (Day 9, non-optional)

**One final suggestion the panel hasn't discussed: demo choreography.** On demo day, the order in which you demonstrate scenarios matters enormously. Start with the highest-confidence detections (CANDIDATE_ABSENT, MULTIPLE_PERSONS) to build jury confidence, then show FACE_MISMATCH + seat-swap cross-identification as the "wow" moment, then gaze + audio fusion. Never start with the noisiest detection channels. This is a presentation strategy, not an architecture decision, but it directly impacts the 8/10 score.

KEY_TRADEOFFS:

1. **UI-agnostic agent (local HTTP API) vs. tightly-coupled PyQt6:** Adds ~2 hours of abstraction work upfront but eliminates the single largest existential risk (deployment failure). Every reviewer agrees deployment is the #1 risk. This is the correct investment. **Right tradeoff.**

2. **Audio as bonus/fusion-only vs. audio as standalone detection:** All three reviewers now converge on conservative. The 30-mic physics problem dominates. Audio should contribute to fusion and rarely fire standalone. Missing a 3-second whisper is acceptable; generating 15 false positives is not. **Right tradeoff for this demo context**, though in production with individual rooms the threshold should drop.

3. **Cross-terminal seat-swap detection vs. simpler per-terminal-only matching:** Adds ~2 hours of work (embedding sync + edge-side 1:N matching) but creates the single most memorable demo moment. The jury will almost certainly stage a seat swap. Identifying *which* candidate is at the wrong terminal, not just "this isn't the right person," is a qualitative leap in capability. **Clearly the right tradeoff.**

4. **CCTV stub (display-only) vs. CCTV AI vs. no CCTV:** Display-only is the Pareto-optimal point for the demo. Full CCTV AI is high-risk with marginal demo benefit. No CCTV is a compliance failure. **Universal consensus, right tradeoff.**

5. **Both packaging options (WinPython + PyInstaller fallback) vs. committing to one:** Doubles the testing surface on Day 0 but halves the probability of deployment failure. For a hackathon where you get one shot, redundancy in deployment strategy is worth the prep cost. **Right tradeoff.**

6. **Overall scope question — are we doing too much?** This is the meta-tradeoff. The Tier 0-1-2-3 prioritization above explicitly addresses this. If the team executes Tier 0 + Tier 1 items 8-10 + Tier 2 items 13-14, they have a winning demo. Items 15-21 are genuine "nice to have." The risk is not scope creep per se, but rather spending too long perfecting lower-tier items at the expense of rock-solid Tier 0 execution. **The schedule must be milestone-gated: if Tier 0 isn't complete by Day 5, all Tier 2+ items are cut.**

### GPT-5.2

POSITION: PARTIALLY_AGREE

REASONING:
My position has shifted slightly: the **core topology is still correct**, but I now consider the current proposal **not “battle-ready to win” unless you apply 4–5 specific changes immediately**. The other participants’ feedback strengthened the case that your biggest threats are *demo optics + operational reliability*, not ML modeling.

1) **Have any arguments changed my position?**
- **Yes—audio and posture need stronger containment.** Claude’s updated take (audio is a “physics problem”) is correct. In a single-room, 30-mic setup, standalone audio is very likely to hurt precision unless you add edge-side suppression. Likewise, posture as standalone is too FP-prone; fusion-only is the right move.
- **Yes—ground truth needs a “trigger app”.** Gemini’s “Malpractice Trigger” app is a high-leverage idea: it makes your confusion matrix credible and reduces reliance on perfect clock sync between humans and system.
- **Yes—CCTV cannot be zero.** Consensus is right: even a dumb CCTV tile is better than an explanation. This is demo scoring/requirements compliance, not engineering purity.

2) **Which concerns from others do I agree/disagree with?**
- **Agree strongly** with Claude + Gemini:
  - CCTV minimal stub is mandatory.
  - Seat swap must be identified cross-terminal (not just “mismatch”).
  - Multi-terminal integration on Day 8 is too late.
  - Audio crosstalk will create false positives without suppression.
- **Partial disagreement / nuance:**
  - **PyQt6 vs Browser UI:** I align more with Gemini/Claude that browser UI is faster and more robust *for hackathon velocity*. But I don’t think PyQt is inherently “wrong”—it’s wrong **if it becomes the critical path** (layout polish + packaging fragility). The safer stance is: *headless agent + browser exam UI* as primary, PyQt as optional fallback only if your team is already fluent and can keep it minimal.
  - **PyInstaller as “the answer”:** I still disagree with making PyInstaller the primary bet. AV false-positives and huge bundles are real. Carry both WinPython portable and PyInstaller `--onedir`, choose after Day-0 tests.

3) **Common ground (high-confidence convergence)**
There is strong convergence across all reviewers that the winning path is:
- Keep the edge/terminal split + episode semantics.
- Add **CCTV stub**, **cross-terminal seat-swap**, **early multi-terminal testing**, **audio suppression**, **fusion-only posture**, and **better ground truth instrumentation**.

CONCERNS:
1. **CCTV “missing feature” penalty:** No CCTV at all is an easy judge objection/disqualifier.
2. **Seat swap marquee scenario:** Without edge-side global embedding lookup, you may miss or under-explain the most likely staged event.
3. **Audio crosstalk mass false positives:** One loud speaker can light up many terminals; without edge suppression it will wreck precision and the confusion matrix.
4. **UI + deployment fragility:** PyQt6 + Python stack on unknown Windows desktops can fail to launch or behave inconsistently (drivers, permissions, AV, missing runtimes).
5. **Ground-truth timing credibility:** If human logs and system timestamps don’t align tightly, your confusion matrix will be challenged even if detections are good.
6. **Standalone posture/gaze thresholds:** Risk of flagging normal test-taking behavior (reading down, thinking, glancing) → FP flood.
7. **Late scale testing:** Discovering LAN/backpressure/DB contention at Day 8 risks unrecoverable instability.

SUGGESTIONS:
1. **Add a minimal CCTV stub (must-do):**
   - Edge server: RTSP/USB ingest → display tile on dashboard.
   - Optional: background subtraction / motion heatmap overlay (no heavy AI).
   - Goal: never answer “we didn’t do CCTV”; you show “integrated feed + activity highlighting.”

2. **Implement cross-terminal seat-swap detection (must-do differentiator):**
   - During enrollment, push/store embeddings centrally on edge.
   - On FACE_MISMATCH/UNCERTAIN, edge compares probe embedding vs all enrolled and emits:
     - `CONFIRMED_SEAT_SWAP {from_terminal, to_terminal, matched_candidate_id, score}`
   - This is both a compliance win (impersonation) and a demo “wow”.

3. **Adopt the “Malpractice Trigger” app (high ROI):**
   - Mobile web page served by edge with 6 big buttons + terminal selector.
   - Logs ground truth timestamps directly to the same DB → cleaner matching and better narrative.

4. **Contain audio to protect precision:**
   - Keep 10s threshold conservative **or make audio fusion-only** by default.
   - Add edge-side suppression: if ≥N terminals trigger within 300–800ms, only keep the **highest-amplitude** (or highest-confidence) terminal; suppress the rest as common-mode.

5. **Make SUSPICIOUS_POSTURE fusion-only:**
   - Never emit standalone posture alerts.
   - Only contribute to `POSSIBLE_MATERIAL_USE` when paired with gaze and/or UI inactivity.

6. **Earlier integration testing (schedule correction):**
   - Move “3–5 terminals end-to-end” to Day 4/5.
   - Stress to 10+ as soon as you have basic events, not after UI polish.

7. **Deployment hardening:**
   - Build a **pre-flight diagnostic** script/tool: camera open, mic levels, write permissions, model load, ONNX runtime present.
   - Ship both WinPython portable and PyInstaller `--onedir`; decide on-site.

8. **Re-add tamper-evident hash chaining (cheap compliance):**
   - JSONL: `hash_i = sha256(hash_{i-1} + canonical_json(event_i))`
   - Minimal code, directly addresses “tamper-proof logs” expectation.

KEY_TRADEOFFS:
- **Browser UI vs PyQt6:** Browser UI trades some RAM for *much higher velocity and launch reliability* on unknown Windows machines. For a 7–10 day hackathon, that trade is usually correct. PyQt is acceptable only if it’s not the critical path and packaging is proven early.
- **Audio recall vs precision:** In a single-room 30-terminal demo, optimizing for recall on audio is self-defeating. Conservative thresholds + suppression (or fusion-only audio) is the right trade to satisfy “minimize false alarms.”
- **CCTV AI depth vs compliance:** A shallow CCTV integration beats a deep but risky CCTV model. The right trade for demo day is “show the feed + simple activity overlay.”
- **Edge simplicity vs seat-swap clarity:** Cross-terminal matching adds complexity but buys a decisive differentiator and directly targets likely staged events. This is the right complexity to add.

### Gemini 3 Pro

POSITION: PARTIALLY_AGREE

REASONING:
There is now **unanimous consensus** across the panel on the critical path to winning: the AI architecture is sound, but the **Engineering Strategy** (UI choice, Deployment, Integration Schedule) contains fatal risks.

1.  **The "Headless" Consensus:** All reviewers identify the PyQt6 UI as a velocity and stability risk. GPT-5.2's proposal to decouple (Headless Python Agent + Browser UI) is the superior architectural compromise. It eliminates the time-sink of Qt layouts while preserving Python for the AI pipeline.
2.  **The "Seat Swap" Gap:** All reviewers agree that the current design fails to detect a specific "Candidate A moves to Terminal B" scenario. This is a high-probability demo event. The Edge Server *must* perform cross-terminal embedding matching.
3.  **CCTV & Audio Realism:** We all agree that a "CCTV Stub" is mandatory for compliance, and that "Audio Crosstalk" in a single room requires physics-aware filtering (common-mode rejection), not just threshold tuning.

CONCERNS:
1.  **The "Seat Swap" Blind Spot:** (Aligned with all) If you only check face mismatch locally, you miss the "Impersonation by another valid candidate" scenario. This is a specific, high-value demo case you will currently fail.
2.  **Deployment "Works on My Machine":** (Aligned with GPT-5.2 & Claude) Relying solely on WinPython or solely on PyInstaller is reckless. Windows permissions/AV on exam terminals are unpredictable.
3.  **Audio Physics:** (Aligned with Claude) In a quiet exam hall, a cough or whisper triggers 5-10 nearby microphones. Without amplitude comparison at the Edge, the dashboard will light up like a Christmas tree (False Positive disaster).
4.  **Late Integration:** (Aligned with all) Testing multi-terminal networking on Day 8 is a death sentence. Network congestion and database locking need to be solved on Day 4.

SUGGESTIONS:
1.  **Adopt the "Headless Agent + Browser UI" Pattern:**
    *   **Architecture:** `agent.exe` (Python) runs in the tray. It spins up a minimal `localhost` WebSocket server and handles camera/AI.
    *   **UI:** The exam interface is a standard HTML/JS file opened in the system's default browser (Chrome/Edge), communicating with the agent via local WebSocket.
    *   **Benefit:** Zero Qt layout code. Zero browser permission issues (agent handles hardware). fast UI iteration.

2.  **Implement Edge-Side "Seat Swap" Logic:**
    *   **Change:** Push all enrollment embeddings to the Edge Server (in RAM).
    *   **Logic:** When Terminal A sends a `FACE_MISMATCH` alert with the intruder's embedding, the Edge Server searches the global list.
    *   **Result:** Instead of "Unknown Person", the dashboard alerts: "**CRITICAL: Candidate B (from Terminal 5) detected at Terminal 1.**" This is a winning differentiator.

3.  **Implement the "Malpractice Trigger" App (Endorsing Claude):**
    *   Build the mobile web app for the actors/stagers.
    *   **Why:** It creates perfect, microsecond-aligned ground truth logs that you control, removing human observer latency from the confusion matrix.

4.  **Amplitude-Based Audio Voting:**
    *   **Logic:** When the Edge Server receives `SUSPICIOUS_AUDIO` from multiple terminals within a 500ms window, it compares the reported RMS amplitude.
    *   **Action:** Only display the alert for the terminal with the *loudest* signal. Suppress the neighbors.

5.  **The "CCTV Stub" (Day 6):**
    *   Do not over-engineer. Ingest RTSP -> Resize -> Display on Dashboard. Add a simple "Motion Detected" text overlay if pixel change > threshold. Done.

KEY_TRADEOFFS:
**Headless+Browser vs. PyQt:** You accept a slightly complex local WebSocket handshake to gain massive **Development Velocity** and **UI Polish**. This is the correct trade-off for a 7-day timeline.
**Edge-Side Matching vs. Local-Only:** You accept higher RAM usage on the Edge Server (storing all embeddings) to gain the ability to detect **Internal Impersonation**. This directly targets the "Seat Swap" scenario.
**Amplitude Voting vs. Simple Thresholds:** You trade "detecting every whisper" (Recall) for "not flooding the dashboard" (Precision). In a demo with 30 terminals in one room, **Precision is the only metric that matters.**

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
1. The core architecture (terminal-side lightweight inference + edge-side aggregation/dashboard + episode-based events + no raw video uplink + no cloud dependency) is sound and should not change.
2. Cross-terminal seat-swap detection on the edge server is the single highest-ROI feature addition — enrollment embeddings must be pushed to edge, and on FACE_MISMATCH the edge must compare against all enrolled candidates and emit CONFIRMED_SEAT_SWAP with both terminal IDs.
3. A minimal CCTV stub is mandatory — RTSP/USB ingest displayed on the dashboard with optional background subtraction/motion overlay, no AI detection. Completely omitting CCTV is an unforced compliance failure against a stated requirement.
4. SUSPICIOUS_POSTURE must be fusion-only — never fire standalone, only contribute through fusion rules (POSTURE + GAZE → POSSIBLE_MATERIAL_USE). Standalone head-pitch alerts will generate excessive false positives from normal reading/thinking behavior and destroy the confusion matrix.
5. Audio crosstalk in a 30-mic single room is a physics problem. Common-mode rejection at the edge server is required: buffer VAD events for 300-800ms, if multiple terminals trigger simultaneously suppress all but the highest-amplitude source. The 10s standalone threshold should remain conservative.
6. Multi-terminal integration testing must move to Day 4-5 (3+ terminals), not Day 8. Late discovery of LAN congestion, SQLite contention, and firewall issues is a project-ending risk.
7. Hash-chaining for tamper-evident JSONL logs should be re-added — trivial implementation (~30 lines), directly addresses a stated requirement in the challenge spec.
8. Deployment on unknown government Windows machines is the single most likely cause of demo-day failure. A pre-flight diagnostic script/tool (camera, mic, ONNX runtime, write permissions, network) must be built and run on every terminal before deployment.
9. Both WinPython portable and PyInstaller --onedir should be prepared; the decision of which to use should be made after Day 0 hardware testing, not committed to in advance.
10. A fallback demo video must be recorded on Day 9 — a flawless 3-minute screen capture of the entire flow as insurance against demo-day failures.
11. The ground-truth "Malpractice Trigger" app (mobile web page served from edge with 6 buttons + terminal selector for actors to log the exact moment they stage malpractice) is high-ROI and should be built — it creates microsecond-accurate ground truth and eliminates human observer timing errors from the confusion matrix.
12. Capture resolution should be explicitly set to 640x480 to reduce CPU/memory load on 4GB machines.
13. The FPS-based degradation ladder is a strong differentiator and should be made visible on the dashboard (not silent).
14. The confusion matrix with 1:1 greedy matching, populated from ground-truth logs, is a key competitive differentiator most teams will lack.
15. Demo choreography matters: start with highest-confidence detections (CANDIDATE_ABSENT, MULTIPLE_PERSONS) to build jury confidence, then show seat-swap cross-identification as the "wow" moment, then fusion alerts.
16. SQLite write serialization on the edge server should use a single-writer async batching queue (flush every 250-500ms or on N pending events) to prevent SQLITE_BUSY errors under 30-terminal concurrency.
17. A scalability one-pager (edge-server-per-hall model, cost breakdown ~₹25,000 per hall, metadata aggregation to district/state over 4G) should be prepared for the jury.

CONVERGENCE:
1. **Audio threshold direction**: Claude initially suggested lowering the audio threshold to 5 seconds, then explicitly retracted after Gemini raised the 30-mic crosstalk problem. All three converged on keeping 10s conservative threshold + edge-side amplitude-based common-mode rejection.
2. **CCTV inclusion**: The original proposal cut CCTV entirely. All three reviewers independently flagged this as a strategic error and converged on a "display-only stub" as the Pareto-optimal solution.
3. **Posture alert handling**: Reviewers initially ranged from "drop it entirely" (Claude R1) to various gating mechanisms. All converged on GPT-5.2's "fusion-only" formulation by Round 2.
4. **Integration testing timing**: The original schedule had multi-terminal testing at Day 8. All three reviewers converged independently on moving this to Day 4-5.
5. **Hash-chaining**: Originally cut from the proposal. GPT-5.2 raised re-adding it in Round 1; by Round 2-3 all reviewers agreed the minimal implementation cost justified compliance.
6. **Seat-swap detection**: Not in the original architecture. Claude raised it in Round 1; by Round 2 all three reviewers elevated it to the highest-ROI addition.

REMAINING_DISAGREEMENTS:
1. **PyQt6 vs. Browser-based exam UI as the primary path:**
   - **GPT-5.2**: Advocates headless Python agent + browser-based exam UI as primary. Browser UI offers dramatically higher deployment reliability and faster UI development velocity. PyQt6 acceptable only as optional fallback.
   - **Gemini 3 Pro**: Strongly advocates the headless agent + browser UI pattern. Argues PyQt6 is a "velocity trap" where 50% of time is spent fighting Qt layouts. Proposes local WebSocket communication between agent and browser.
   - **Claude Opus**: Acknowledges deployment risk of PyQt6 but identifies hidden costs in the browser approach: (a) requires a second communication protocol (local WebSocket) between headless agent and browser, (b) browser webcam permission dialogs can be silently blocked by confused students, (c) kiosk lockdown is fragile in a browser. Proposes a UI-agnostic local HTTP API as an abstraction layer so both paths remain viable, with the choice made on Day 0-1 based on hardware reality. Does not endorse browser-first as strongly as the other two.

2. **PyInstaller reliability assessment:**
   - **Gemini 3 Pro**: Initially recommended PyInstaller --onedir as primary (Round 1), though softened by Round 3 to agreeing both should be prepared.
   - **Claude Opus**: Explicitly argued against PyInstaller as primary due to antivirus false positives and large bundle sizes (~800MB-1.2GB). Recommends WinPython as primary with PyInstaller as fallback.
   - **GPT-5.2**: Agreed neither should be the sole bet; carry both and decide after Day 0 testing.

RECOMMENDED_PLAN:

**Architecture Decisions (lock immediately):**
- [ ] Keep terminal-side inference + edge-side aggregation + episode-based events + no cloud dependency as the core topology — no changes needed.
- [ ] Design the terminal agent from Day 1 to expose a local HTTP API (`/api/enrollment/status`, `/api/exam/state`, `/api/webcam/frame`) so the exam UI can be served either via PyQt6 or browser. This is the insurance policy against deployment failure — adds ~2 hours upfront, eliminates the existential UI/deployment risk.
- [ ] Choose PyQt6 vs. browser-based UI on Day 0-1 based on hardware test results and team familiarity. Do not commit before testing on a non-dev machine.
- [ ] Prepare both WinPython portable and PyInstaller --onedir packages. Test both on Day 0. Deploy whichever works on target hardware.
- [ ] Set capture resolution explicitly to 640x480 on all terminals.

**Must-Add Features (all reviewers agree these are non-negotiable):**
- [ ] Cross-terminal seat-swap detection: push enrollment embeddings to edge during enrollment; on FACE_MISMATCH, edge compares probe embedding against all 30 enrolled candidates; emit CONFIRMED_SEAT_SWAP with both terminal IDs and candidate identifiers. Implement during Day 2-3 enrollment work.
- [ ] CCTV display stub: edge server ingests RTSP stream (or USB camera fallback); display on dashboard tile; optional background subtraction motion overlay; no AI detection. Budget 2-4 hours on Day 6.
- [ ] Hash-chaining for tamper-evident logs: `hash_i = sha256(hash_{i-1} || canonical_json(event_i))`, stored in each JSONL line. ~30 minutes, implement on Day 4-5.
- [ ] Malpractice Trigger app: mobile-friendly web page served from edge server, 6 buttons (one per malpractice type) + terminal selector, logs `{terminal_id, malpractice_type, timestamp}` to edge DB. Build on Day 5.
- [ ] Pre-flight diagnostic script: test camera open, mic open, ONNX runtime load, write permissions, LAN connectivity to edge. Run on every terminal within first 15 minutes of setup.

**Alert Strategy Modifications:**
- [ ] Make SUSPICIOUS_POSTURE fusion-only. Remove standalone firing. Only contributes via POSTURE + GAZE → POSSIBLE_MATERIAL_USE (or POSTURE + UI inactivity >15s if exam UI reports last-interaction timestamps).
- [ ] Keep SUSPICIOUS_AUDIO at 10s standalone threshold (conservative for demo).
- [ ] Implement amplitude-based common-mode audio rejection at edge: buffer VAD events for 500ms; if ≥3 terminals trigger simultaneously, suppress all but the highest-amplitude source.
- [ ] Prioritize the 4 highest-confidence alerts for standalone detection: CANDIDATE_ABSENT, MULTIPLE_PERSONS, FACE_MISMATCH, GAZE_DEVIATION. Treat AUDIO and POSTURE as fusion contributors.

**Edge Server Hardening:**
- [ ] SQLite write serialization: single async writer task consuming from an in-memory queue, batch-flushing every 250ms or on 50 pending events.
- [ ] Store all enrollment embeddings in memory for cross-terminal matching.
- [ ] Implement time sync: terminals stamp `t_capture_monotonic` and `t_capture_wall`; edge periodically measures offset with RTT compensation; store both raw and corrected times in all events.

**Revised Schedule:**
- [ ] Day 0: GO/NO-GO — pre-flight diagnostic on non-dev machine with restricted user account. Test both WinPython and PyInstaller. Decide UI approach (PyQt6 vs browser). Decide packaging tool.
- [ ] Day 1: Webcam + 3 threads + BlazeFace + CLAHE + MOSSE tracker. Local HTTP API skeleton for UI-agnostic agent.
- [ ] Day 2: Enrollment (5 embeddings, quality gates) + embedding push to edge. Face verification with 3-zone hysteresis. Edge server start with FastAPI + SQLite WAL + async write queue.
- [ ] Day 3: Multi-face + absence + gaze (baseline calibration: 10s median+MAD during enrollment). FPS degradation modes. Episode-based event flow to edge.
- [ ] Day 4: **HARD DEADLINE vertical slice with 3 terminals.** Events flow from 3 terminals → edge → dashboard tile. Discover LAN/SQLite/firewall issues here. Silero VAD integration. Time sync.
- [ ] Day 5: Dashboard (terminal grid, alert feed, detail view, degradation status). Exam UI (MCQ + timer + navigation). Ground truth logger. Malpractice trigger app. **Minimum viable demo works end-to-end.**
- [ ] Day 6: Evidence snapshot push. Common-mode audio rejection. Evidence viewer. Fusion rules. CCTV display stub. Hash-chaining. Cross-terminal seat-swap detection.
- [ ] Day 7: Confusion matrix (1:1 greedy matching). Post-exam summary. Stress test with 5-10 terminals. Threshold tuning.
- [ ] Day 8: Multi-terminal stress test (10+ terminals). Threshold tuning on real hardware. Scalability one-pager.
- [ ] Day 9: Full calibration exam with ALL planned malpractice scenarios. Tune thresholds. **Record fallback demo video (3-minute flawless walkthrough).** This is non-optional.
- [ ] Day 10: Demo rehearsal only. Practice 6-10 scenarios in presentation order. Prepare USB backups of both packaging options.

**Milestone Gate:**
- [ ] If Tier 0 items (preflight, core pipeline, enrollment, edge server, dashboard, 3-terminal vertical slice) are not complete by end of Day 5, cut ALL Tier 2+ items (CCTV stub, hash-chaining, store-and-forward, evidence encryption) and focus exclusively on making the core 4 alert types + seat-swap detection + confusion matrix work flawlessly.

**Demo Day Strategy:**
- [ ] Demonstrate scenarios in confidence order: (1) CANDIDATE_ABSENT, (2) MULTIPLE_PERSONS, (3) FACE_MISMATCH + CONFIRMED_SEAT_SWAP as the "wow" moment, (4) GAZE + AUDIO fusion, (5) show confusion matrix populated live.
- [ ] Show degradation status tile live (FPS → active modalities) as a scalability/robustness narrative.
- [ ] Have fallback video queued and ready to play within 30 seconds if live demo fails.
- [ ] Bring a portable WiFi router (~₹500-1,000) for the malpractice trigger app to work from mobile devices on the same LAN.
- [ ] Bring a USB webcam (~₹1,500) as CCTV camera fallback if venue lacks an IP camera.