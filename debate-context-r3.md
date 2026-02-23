# ExamGuard Hackathon Context — Binding Constraints & Requirements

This is the binding context for the architecture debate. Every recommendation MUST be feasible within these constraints.

---

## 1. Official Challenge Requirements (from challenge.json)

**Challenge Code:** 100006
**Department:** School Education Department, Government of Andhra Pradesh
**Organizer:** RTGS (Real Time Governance Society) with EY + Gates Foundation

### 6 Expected Solution Components

1. **AI-Powered Multi-Modal Monitoring:** Integrate CCTV and laptop camera feeds with audio input to detect malpractices in real time.
2. **Anomaly Detection:** Use AI/ML to identify multiple faces, absence of candidates, unauthorized materials, or gaze deviations.
3. **Audio Analytics:** Detect and flag background speech or suspicious sounds indicating collaboration or consultation.
4. **Automated Evidence Capture:** Generate tamper-proof logs including flagged snapshots, video/audio clips, and timestamps.
5. **Real-Time Alerts:** Provide instant notifications to human proctors for live intervention during suspicious events.
6. **Integration:** Enable compatibility with existing online exam platforms and CCTV systems through secure APIs.

### POC Scope

- 30 registered candidates seated in one exam hall
- 10-minute test sessions
- 6-10 controlled malpractice events staged (including seat swaps)
- **Success: detect at least 8 out of 10 malpractice events accurately**
- AI detection results must match timestamped ground-truth logs from human monitors

### Data Privacy

Solutions must comply with privacy, consent, and data retention policies for student video and audio data. Processing should be conducted within secure, isolated networks with explicit consent from all participants.

---

## 2. Orientation Session Key Decisions (Binding — from Anil Arora, Senior Consultant)

These were stated verbally during the orientation call on 24 Dec 2025 and are NOT in the formal challenge document but are binding:

### No Invigilator in the Room
> "We totally remove invigilator in this thing. Only students will be there."

Simplification: every person at a terminal is the registered candidate. Every additional person is suspicious.

### Facial Recognition via Self-Registration
> "Candidates themselves put register at front end, then they will go to examination."

Students self-register face at terminal before exam. No external database integration needed.

### CCTV is Back-Facing Only
> "It will be from back side. Whenever we are connecting it to cloud, we make sure that his privacy is protected."

No facial data from CCTV. Only body/position tracking possible from CCTV.

### Confusion Matrix is Mandatory
> "You have to make that matrix — false positive, false negative, true positive, true negative — and minimize the false alarms."

Anil sir explicitly wants very few false positives, zero false negatives, and a formal confusion matrix as part of evaluation.

### Hardware Terminals are Fixed
> "Those terminals would remain — these 40 desktops will remain."

Cannot bring own hardware for terminals. Existing desktops must be used. For CCTV/cloud, own components are fine.

### Lightweight is Key
> "Don't try to put a lot of LLMs or something on it... spending around 30,000."

~30,000 INR per center budget. Must scale to 5,000-10,000 centers. No heavy cloud dependency on terminal side.

### Internet Baseline
> "Just assume we have a 4G connection in all our schools."

4G (~10 Mbps peak) is the connectivity baseline.

### Processing Model (Hybrid)
- **Local on terminal:** Webcam feeds (face, gaze, audio) — privacy-sensitive
- **Cloud:** CCTV feed (back-facing, no privacy concern) — heavier compute OK
- **Only metadata leaves terminal:** timestamps, event types, confidence scores — NO raw video

---

## 3. Competition Context

- **11 startups shortlisted** — we are one of them
- **Top 3 winners** get full pilot contracts with the department
- **7-10 days** to build a working solution from orientation to demo
- Demo will be in a **real classroom** provided by the department
- The department will stage specific cheating scenarios and compare system flags against their own notes
- Both detection AND timestamp accuracy matter

---

## 4. What the Jury Will Evaluate (Inferred)

Based on the challenge spec, orientation, and typical government hackathon scoring:

1. **Detection accuracy** — Can it catch 8/10 malpractice events? (Hard requirement)
2. **Low false positive rate** — Does it flood the screen with false alarms? (Explicitly emphasized by Anil sir)
3. **Real-time responsiveness** — Does the dashboard show alerts as they happen?
4. **Evidence quality** — Are flagged events timestamped and matched to ground truth?
5. **Scalability story** — Can this work at 10,000 centers? Cost-effective?
6. **Privacy compliance** — Does face data stay local? DPDP Act awareness?
7. **Working demo** — Does the system actually work in a live classroom setting?
8. **Confusion matrix** — Formal accuracy reporting

---

## 5. Technical Constraints Summary

| Constraint | Value |
|-----------|-------|
| Terminals per hall | 10-40 (30 for demo) |
| Terminal hardware | Basic desktops (i5 or Celeron), 4GB+ RAM, webcam |
| CCTV | 1 camera, back-facing, 1080p |
| Internet | 4G baseline (~10 Mbps) |
| Webcam processing | Must be local on terminal (privacy) |
| Budget | ~30,000 INR per center |
| Timeline | 7-10 days |
| Team size | 1-2 developers |
| Demo duration | 10-minute exam sessions |
| Malpractice events | 6-10 staged events per session |
| Success threshold | ≥8/10 detection rate |
| OS | Unknown (likely Windows 10/11) — must confirm |
