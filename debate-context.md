# ExamGuard Debate Context

## Challenge (from challenge.json)

**Challenge Code:** 100006 — School Education - Exam Guard
**Department:** School Education Department, Government of Andhra Pradesh
**Goal:** AI-Enabled Live Proctoring for Online Examinations — detect cheating, flag suspicious behavior, ensure secure and credible assessments.

### Expected Solution Components
1. AI-Powered Multi-Modal Monitoring: Integrate CCTV and laptop camera feeds with audio input to detect malpractices in real time.
2. Anomaly Detection: Use AI/ML to identify multiple faces, absence of candidates, unauthorized materials, or gaze deviations.
3. Audio Analytics: Detect and flag background speech or suspicious sounds indicating collaboration or consultation.
4. Automated Evidence Capture: Generate tamper-proof logs including flagged snapshots, video/audio clips, and timestamps.
5. Real-Time Alerts: Provide instant notifications to human proctors for live intervention during suspicious events.
6. Integration: Enable compatibility with existing online exam platforms and CCTV systems through secure APIs.

### POC Scope
- 30 registered candidates seated in one exam hall for 10-minute sessions.
- Each test will simulate 6–10 controlled malpractice events, including wrong-person seat swaps.
- **Success Criteria:** Out of 10 malpractice events, detect at least 8 accurately. AI detection results should match timestamped ground-truth logs recorded by human monitors.

### Key Constraints (from orientation session with Anil sir, Dept. of School Education)
- **Budget:** ~₹30,000 per center
- **Terminals:** 10-40 desktops with webcams per classroom
- **CCTV:** 1 camera from back of room (privacy-preserving, no faces)
- **Connectivity:** 4G baseline — so local processing is critical
- **No invigilator present** — fully AI-monitored
- **Local processing** for facial recognition, audio, multi-face detection on terminals
- **CCTV can be cloud-processed** (it's from the back, no privacy issues)
- **Basic hardware:** Government school desktops (i5/Celeron, 4GB RAM)
- **Open-source only** — no proprietary licensing
- **Minimize false positives** — false alarms destroy credibility
- **Seat swap detection** is explicitly mentioned as required ("recognize the person, whether he is swapping")
- **Privacy:** DPDP Act compliance, faces stay local, only metadata/timestamps go over network
- **Scale vision:** 5,000-10,000 centers eventually

## Winning Criteria (what the jury evaluates)
1. **Accuracy** — 8/10 malpractice events detected with minimal false positives
2. **Performance** — Real-time processing on basic hardware with 4G connectivity
3. **Innovation** — Novel approaches that go beyond basic face detection
4. **Completeness** — Multi-modal (video + audio + CCTV), tamper-proof logs, real-time alerts
5. **Scalability** — Architecture that can scale to 5,000+ centers
6. **Demo impact** — Live working demo matters more than slides
