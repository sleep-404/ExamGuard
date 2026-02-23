# ExamGuard Debate Context

## Challenge (from challenge.json)

**Challenge Code:** 100006 — School Education - Exam Guard
**Department:** School Education Department, Government of Andhra Pradesh
**Goal:** AI-Enabled Live Proctoring for Online Examinations — detect cheating, flag suspicious behavior, ensure secure and credible assessments.

### Expected Solution Components (What the Jury Evaluates)
1. **AI-Powered Multi-Modal Monitoring:** Integrate CCTV and laptop camera feeds with audio input to detect malpractices in real time.
2. **Anomaly Detection:** Use AI/ML to identify multiple faces, absence of candidates, unauthorized materials, or gaze deviations.
3. **Audio Analytics:** Detect and flag background speech or suspicious sounds indicating collaboration or consultation.
4. **Automated Evidence Capture:** Generate tamper-proof logs including flagged snapshots, video/audio clips, and timestamps.
5. **Real-Time Alerts:** Provide instant notifications to human proctors for live intervention during suspicious events.
6. **Integration:** Enable compatibility with existing online exam platforms and CCTV systems through secure APIs.

### POC Scope
- 30 registered candidates seated in one exam hall for 10-minute sessions.
- Each test will simulate 6–10 controlled malpractice events, including wrong-person seat swaps.
- **Success Criteria:** Out of 10 malpractice events, detect at least 8 accurately. AI detection results should match timestamped ground-truth logs recorded by human monitors.
- **Data Privacy:** Solutions must comply with privacy, consent, and data retention policies. Processing within secure, isolated networks with explicit consent.

### Key Constraints (from orientation session with Anil Arora, Sr. Consultant, Dept. of School Education)
- **Budget:** ~₹30,000 per center at scale (5,000-10,000 centers)
- **Terminals:** 10-40 desktops with webcams per classroom — existing government hardware, we cannot bring our own
- **CCTV:** 1 camera from back of room (privacy-preserving, no faces visible)
- **Connectivity:** 4G baseline
- **No invigilator present** — AI is the sole invigilator. "Even invigilators themselves will be part of the malpractice" — Anil sir
- **Local processing** for webcam data (face recognition, audio, multi-face) on terminals
- **CCTV can be cloud-processed** (back-facing, no privacy concern)
- **Lightweight:** "Don't try to put a lot of LLMs or something on it" — Anil sir
- **Confusion matrix explicitly required:** "You have to make that matrix — false positive, false negative, true positive, true negative — minimize the false alarms" — Anil sir
- **Seat swap detection explicitly required**
- **Privacy:** DPDP Act compliance, faces processed locally, only metadata leaves terminals
- **11 startups competing** — top 3 get a full pilot with the department

### What Makes a Winning Solution
1. **Accuracy** — 8/10 malpractice events detected with minimal false positives
2. **Performance** — Real-time on basic hardware (i5/Celeron, 4GB RAM, 4G)
3. **Innovation** — Beyond basic face detection; multi-modal fusion, tamper-proof logs
4. **Completeness** — Multi-modal (video + audio + CCTV), evidence capture, real-time alerts
5. **Scalability story** — Architecture that scales to 5,000+ centers
6. **Demo impact** — Live working demo > slides; the jury will stage malpractice events and watch

### Competition Analysis
Most competing teams will likely use:
- MediaPipe for face detection/gaze (standard approach)
- YOLO for object detection (phones, books)
- Flask/Streamlit dashboards
- Basic face verification
- Simple alert thresholds

Our architecture differs by:
- Agent-as-camera-owner (no other team will solve Windows webcam exclusivity this way)
- Episode-based event model (not per-frame spam)
- Cross-terminal seat-swap matching at edge (requires centralized embedding store)
- Common-mode audio classification (not just "mute all when noisy")
- Hash-chained tamper-proof logs (addresses "automated evidence capture" requirement)
- Adaptive FPS degradation (graceful under load instead of crash/freeze)
- PnP head pose instead of Face Mesh (saves 30-40ms/frame on Celeron)

### Prior Debate Consensus (15 Common Ground Points)
All 3 models (Claude Opus, GPT-5.2, Gemini 3 Pro) agreed on:
1. Material/phone detection is a critical gap — addressed via heuristics
2. Agent must exclusively own webcam and serve MJPEG
3. Common-mode audio must classify, not suppress
4. Threading over multiprocessing on 4GB
5. Phase 0 pre-flight is a first-class gating deliverable
6. ~430MB memory estimate is dangerously thin — must measure empirically
7. Face Mesh every frame too expensive on Celeron — replaced with PnP head pose
8. CCTV cannot be display-only — BackgroundSubtractorMOG2 motion heatmap
9. Lighting check prevents false CANDIDATE_ABSENT
10. Explicit P0/P1/P2 priority table with hard cut rules
11. Scope is too ambitious — 4 rock-solid detectors > 6 flaky ones
12. All thresholds must be configurable
13. Ground truth logger must clock-sync to edge server
14. "Golden demo" fallback video as insurance
15. Demo must be scripted with 90-second limits and "skip to next" buttons
