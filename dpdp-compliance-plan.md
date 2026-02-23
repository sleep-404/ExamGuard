# ExamGuard: DPDP Act 2023 Compliance Plan

**Date:** 2026-02-23
**Context:** AI-enabled exam proctoring for Dept. of School Education, Govt. of Andhra Pradesh

---

## 1. DPDP Act — What It Says (Key Points for ExamGuard)

### 1.1 Core Definitions

| Term | DPDP Definition | ExamGuard Mapping |
|------|----------------|-------------------|
| **Data Principal** | The individual whose personal data is processed. For children (<18), their parent/guardian. | The student taking the exam (most are minors — school exams). |
| **Data Fiduciary** | Entity that determines purpose and means of processing. | Dept. of School Education, Govt. of AP (not us — we're a technology vendor/data processor). |
| **Data Processor** | Entity processing data on behalf of fiduciary. | Us (ExamGuard). We process on the department's instructions. |
| **Personal Data** | Any data about an identifiable individual in digital form. | Face images, voice recordings, gaze data, behavioral patterns, exam responses. |
| **Processing** | Any operation on personal data — collection, storage, use, analysis, deletion. | Everything our system does: capturing webcam, running face recognition, analyzing audio, logging events. |

### 1.2 Consent Requirements

- **Standard consent:** Free, specific, informed, unconditional, unambiguous, with clear affirmative action.
- **Children's data (Section 9):** Requires **verifiable parental consent** — heightened standard. Must confirm the consenting parent/guardian is authorized (via documents like birth certificates or school records).
- **Withdrawal:** Data principal can withdraw consent at any time. Processing must stop immediately.

### 1.3 The Education Exemption (Rule 11, Schedule IV)

**This is the most important provision for ExamGuard.**

> Educational institutions are exempt from the verifiable parental consent requirement and the prohibition on tracking/behavioral monitoring of children — **BUT ONLY** when processing is limited to:
> - Tracking and behavioral monitoring **for educational activities**, OR
> - **Child safety** purposes

**What this means:** The Dept. of School Education can deploy AI proctoring (which is behavioral monitoring during an educational activity) **without** needing individual verifiable parental consent for each student, as long as:
1. The monitoring serves an educational purpose (exam integrity) or safety purpose
2. Data is used only for that purpose — not for profiling, commercial analytics, or unrelated surveillance
3. All other DPDP obligations still apply (security, retention limits, transparency, breach notification)

### 1.4 State Instrumentality Exemption (Section 7(b)-(c))

The Dept. of School Education is a state instrumentality. Under Section 7(b)-(c), state entities can process personal data **without explicit consent** when:
- Providing services, certificates, licenses, or permits to the data principal
- Performing functions under any law in force in India

Conducting examinations under state education law is a statutory function. This provides a **second legal basis** for processing without individual consent.

### 1.5 What Still Applies Even With Exemptions

Even with the education and state exemptions, the department MUST:

| Obligation | Section | What We Must Do |
|-----------|---------|-----------------|
| **Security safeguards** | 8(5) | Encrypt data in transit and at rest. Access controls. Incident detection. |
| **Data quality** | 8(3) | Ensure personal data used for decisions is accurate and complete. |
| **Purpose limitation** | 6 | Use proctoring data ONLY for exam integrity — never for anything else. |
| **Data retention limits** | 8(7) | Delete personal data once the exam purpose is served (results finalized, disputes resolved). |
| **Breach notification** | 8(6) | Notify Data Protection Board within 72 hours of any breach. Notify affected individuals ASAP. |
| **Transparency / Notice** | 5 | Inform students/parents about what data is collected, how it's processed, and their rights. |
| **Grievance redressal** | 8(10) | Mechanism for students/parents to raise complaints. Respond within 90 days. |
| **No secondary use** | 6 | Cannot share proctoring data with police, other departments, or commercial entities. |

### 1.6 Penalties

| Violation | Maximum Penalty |
|-----------|----------------|
| Security breach / inadequate safeguards | ₹250 crore (~$30M) |
| Failure to notify breach | ₹200 crore (~$24M) |
| Violations involving children's data | ₹200 crore (~$24M) |
| Consent/notice violations | ₹50 crore (~$6M) |

---

## 2. What We CAN Do — The Compliant Architecture

### 2.1 The Two-Tier Model: Open-Source + Government Infrastructure

The key insight: **if we use open-source models running entirely within government-controlled infrastructure, the data never leaves government boundaries.** This is the strongest possible compliance posture because:

1. **No cross-border transfer issues** (Rule 14) — data stays in India, on government servers
2. **No third-party data processor risk** — the government controls the compute
3. **No vendor lock-in** — open-source means the department can audit every line of code
4. **No cloud provider access to data** — no AWS/Google/Azure accessing student biometric data
5. **Full sovereignty** — the government retains complete control over all personal data

### 2.2 Architecture: Laptop Software (Piece 1)

```
┌─────────────────────────────────────────────────────┐
│  EXAM TERMINAL (Student's Desktop)                  │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────┐ │
│  │ Webcam Feed  │  │ Microphone   │  │ Exam App  │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬─────┘ │
│         │                 │                │        │
│  ┌──────▼─────────────────▼────────────────▼─────┐  │
│  │         LOCAL AI PROCESSING ENGINE            │  │
│  │                                               │  │
│  │  • Face Detection (MediaPipe BlazeFace, 3MB)  │  │
│  │  • Face Verification (MobileFaceNet, 4MB)     │  │
│  │  • Gaze Tracking (MediaPipe Face Mesh)        │  │
│  │  • Multi-face Detection                       │  │
│  │  • Audio VAD (WebRTC/energy threshold)        │  │
│  │  • Basic Object Detection (YOLOv5-nano, 2MB)  │  │
│  │                                               │  │
│  │  ALL MODELS: Open-source, <15MB total         │  │
│  │  ALL PROCESSING: On the terminal CPU          │  │
│  │  NO RAW DATA LEAVES THIS BOX                  │  │
│  └──────────────────┬────────────────────────────┘  │
│                     │                               │
│            ┌────────▼────────┐                      │
│            │ Event Metadata  │                      │
│            │ (JSON, no PII)  │                      │
│            └────────┬────────┘                      │
│                     │                               │
└─────────────────────┼───────────────────────────────┘
                      │ LAN only — JSON events
                      ▼
              [Edge Server / Dashboard]
```

**What stays on the terminal (never transmitted):**
- Raw webcam frames
- Face images / embeddings
- Audio waveforms
- Any biometric data

**What leaves the terminal (metadata only):**
```json
{
  "terminal_id": "T-15",
  "timestamp": "2026-03-15T10:05:32Z",
  "event_type": "GAZE_DEVIATION",
  "confidence": 0.87,
  "duration_seconds": 4.2,
  "severity": "MEDIUM"
}
```

**DPDP compliance achieved because:** No personal data is transmitted. The AI runs locally, processes locally, and only sends anonymous event metadata. The face embeddings are held in memory during the exam and discarded after.

### 2.3 GPU-Accelerated Processing Server (The Add-On)

Not every laptop has a capable GPU. But we can set up a **dedicated processing server** within the government network for advanced features:

```
┌─────────────────────────────────────────────────┐
│  GPU PROCESSING SERVER                          │
│  (On-premise at exam center OR state data center)│
│                                                 │
│  Hardware: Single machine with GPU              │
│  - Option A: On-premise mini-server at center   │
│    (NVIDIA Jetson Orin / desktop with RTX 3060) │
│  - Option B: State data center GPU VM           │
│    (stays within government network)            │
│                                                 │
│  What it handles:                               │
│  ┌────────────────────────────────────────────┐ │
│  │ CCTV Feed Analysis (back-facing, no faces) │ │
│  │ • Person detection (YOLOv8-medium)         │ │
│  │ • Person Re-ID (appearance-based tracking) │ │
│  │ • Seat swap detection                      │ │
│  │ • Movement anomaly detection               │ │
│  │ • Crowd counting per zone                  │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │ Advanced Audio Analysis (if needed)        │ │
│  │ • Speaker diarization (multi-voice detect) │ │
│  │ • Whisper-based speech transcription        │ │
│  │ • Audio event classification               │ │
│  └────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────┐ │
│  │ Post-Exam Deep Analysis                    │ │
│  │ • Re-analyze flagged events with bigger    │ │
│  │   models for confirmation/rejection        │ │
│  │ • Generate comprehensive confusion matrix  │ │
│  │ • Produce evidence packages for disputes   │ │
│  └────────────────────────────────────────────┘ │
│                                                 │
│  ALL MODELS: Open-source                        │
│  ALL DATA: Within government network            │
└─────────────────────────────────────────────────┘
```

**Why this is DPDP-compliant:**
- The CCTV feed is **back-facing** — no facial data captured. This is not biometric data.
- The server sits within the government's own network (on-premise or state data center).
- No data leaves government infrastructure.
- Open-source models mean full auditability — no black-box vendor processing.
- The government is both the data fiduciary AND controls the infrastructure.

**Open-Source Models for GPU Server:**

| Task | Model | Size | Why Open-Source Works |
|------|-------|------|---------------------|
| Person Detection | YOLOv8-medium (Ultralytics, AGPL) | ~50MB | State-of-art, runs great on GPU |
| Person Re-ID | OSNet / TransReID (academic, MIT/Apache) | ~20MB | Cross-camera person matching without faces |
| Speech-to-Text | Whisper-small (OpenAI, MIT License) | ~500MB | Best open-source STT, runs on GPU |
| Speaker Diarization | pyannote-audio (MIT License) | ~100MB | Multi-speaker detection |
| Audio Classification | YAMNet (Google, Apache 2.0) | ~15MB | Environmental sound classification |
| Scene Understanding | CLIP-ViT-B/32 (OpenAI, MIT) | ~350MB | Zero-shot scene analysis |

**Total GPU VRAM needed:** ~4-6 GB — a mid-range GPU (RTX 3060, Jetson Orin) handles this comfortably.

### 2.4 Dashboard (Piece 2)

```
┌──────────────────────────────────────────────────────┐
│  MONITORING DASHBOARD                                │
│  (Web app, hosted on government server)              │
│                                                      │
│  What it shows:         What it NEVER shows:         │
│  ✓ Terminal status      ✗ Student faces              │
│  ✓ Alert feed           ✗ Raw webcam feeds           │
│  ✓ Event timeline       ✗ Audio recordings           │
│  ✓ Severity indicators  ✗ Personal identifiers       │
│  ✓ Room-level CCTV view ✗ Biometric data             │
│    (back-facing only)                                │
│  ✓ Confusion matrix                                  │
│  ✓ Evidence snapshots                                │
│    (auto-captured at                                 │
│     alert moments)                                   │
│                                                      │
│  Hosted: Government intranet or state cloud          │
│  Access: Authorized exam administrators only         │
│  Authentication: Government SSO / role-based access  │
└──────────────────────────────────────────────────────┘
```

**Dashboard DPDP considerations:**
- The dashboard receives only **metadata and CCTV feeds** (back-facing, no faces)
- If evidence snapshots are captured from webcams during alerts, they should be:
  - Stored encrypted on the local edge server
  - Accessible only through authenticated dashboard with audit logs
  - Auto-deleted after the dispute resolution period
- Dashboard access must be role-based — only authorized exam administrators

---

## 3. Detailed Compliance Plan — Ensuring DPDP Is Not Violated

### 3.1 Before the Exam: Consent & Notice

Even though the education exemption (Rule 11) likely covers us, **best practice is to still provide notice and obtain consent where practical.** This creates defense-in-depth.

**Action Items:**

| # | Action | Details |
|---|--------|---------|
| 1 | **Display privacy notice at registration** | Before self-registration, show a screen explaining: what data is collected (face image for verification, audio for monitoring), why (exam integrity), how long it's kept (deleted after X days), who can access it (exam administrators only), and how to raise grievances. |
| 2 | **Obtain affirmative consent** | Student clicks "I understand and agree" before registering. For minors, this can be covered by institutional consent (school enrollment terms) + the Rule 11 exemption. |
| 3 | **Provide notice in Telugu and English** | AP's primary language is Telugu. DPDP requires notice in a language the data principal understands. |
| 4 | **Document the legal basis** | Maintain a written record stating: legal basis = Rule 11 education exemption + Section 7(b)-(c) state instrumentality + affirmative consent. Triple-layered defense. |

### 3.2 During the Exam: Data Minimization

| Principle | Implementation |
|-----------|---------------|
| **Collect only what's needed** | Face embedding (not face image) for verification. Audio energy levels (not recordings) for VAD. Gaze angles (not raw eye images). |
| **Process locally** | All biometric processing on terminal CPU. No raw biometric data transmitted anywhere. |
| **Transmit only metadata** | Event type + timestamp + confidence score + terminal ID. No PII in transit. |
| **CCTV is inherently privacy-preserving** | Back-facing camera captures no facial data. Body silhouettes and positions only. |
| **Evidence capture is selective** | Only capture screenshots at alert moments (not continuous recording). Store locally, encrypted. |

### 3.3 Data Lifecycle & Retention

```
Exam Start ──────────── Exam End ──────────── Results Final ──── Dispute Window ──── DELETION
    │                      │                       │                    │                │
    │ Face embedding       │ Face embedding         │                    │                │
    │ created in RAM       │ DELETED from RAM       │                    │                │
    │                      │                        │                    │                │
    │ Audio processing     │ Audio processing       │                    │                │
    │ in real-time         │ STOPPED                │                    │                │
    │                      │                        │                    │                │
    │ Event logs created   │ Event logs sealed      │ Logs available     │ Logs available │ LOGS
    │                      │ (hash chain finalized) │ for review         │ for disputes   │ DELETED
    │                      │                        │                    │                │
    │                      │ Evidence snapshots      │ Snapshots avail.   │ Snapshots      │ SNAPSHOTS
    │                      │ encrypted & stored     │ for review         │ for disputes   │ DELETED
```

**Retention policy:**
- **Face embeddings:** In-memory only. Deleted when exam session ends. Never persisted to disk.
- **Raw audio:** Never stored. Processed in real-time, only results (voice detected / not detected) logged.
- **Event metadata:** Retained for dispute resolution period (suggest 90 days — matches DPDP grievance timeline). Then auto-deleted.
- **Evidence snapshots:** Encrypted at rest. Retained for dispute resolution period. Then auto-deleted.
- **CCTV recordings:** Back-facing only. Retained per existing government CCTV retention policies. No facial data present.

### 3.4 Security Safeguards

| Layer | Safeguard |
|-------|-----------|
| **Data at rest** | AES-256 encryption for all stored evidence. Encrypted SQLite or PostgreSQL with disk encryption. |
| **Data in transit** | TLS 1.3 for all network communication (terminal → edge server, edge → dashboard). |
| **Access control** | Role-based access: Admin (full), Proctor (view alerts), Auditor (read-only logs). No student access to other students' data. |
| **Authentication** | Government SSO integration or strong password policy + MFA for dashboard access. |
| **Audit logging** | Every dashboard access, every data view, every export is logged with who/when/what. |
| **Tamper-proof logs** | SHA-256 hash chain on event logs. Any modification breaks the chain and is detectable. |
| **Network isolation** | Terminal ↔ edge server communication on isolated VLAN. No direct internet access from terminals during exam. |

### 3.5 Open-Source Model Compliance Benefits

| Benefit | Why It Matters for DPDP |
|---------|------------------------|
| **Full auditability** | Government can inspect every line of code. No black-box vendor claims. Supports DPDP's transparency requirements. |
| **No vendor data access** | Unlike commercial proctoring (Proctorio, ProctorU), no third-party company ever sees student data. |
| **No cross-border transfer** | Data never leaves India. Models run on government infrastructure. Zero Rule 14 risk. |
| **Customizable retention** | Government controls deletion schedules. No vendor holding data hostage. |
| **Bias auditability** | Open-source models can be tested for bias across skin tones and demographics. Results are reproducible. |
| **Cost-effective at scale** | No per-exam licensing fees. Supports 10,000 centers at ₹30,000/center without ongoing software costs. |

### 3.6 GPU Strategy: On-Premise vs. Cloud

We have two compliant options for GPU compute:

**Option A: On-Premise GPU Server at Exam Center**

| Aspect | Details |
|--------|---------|
| Hardware | NVIDIA Jetson Orin Nano (~₹40,000) or desktop with RTX 3060 (~₹25,000 used) |
| Placement | At the exam center, connected to LAN |
| Data flow | CCTV → GPU server via LAN. Never leaves the building. |
| Pros | Zero internet dependency for CCTV processing. Maximum privacy. Works offline. |
| Cons | Hardware cost per center. Maintenance burden. |
| Best for | High-stakes exams (board exams, certification). |

**Option B: State Government Cloud / Data Center**

| Aspect | Details |
|--------|---------|
| Infrastructure | AP State Data Center or NIC Cloud (MeghRaj) — government-controlled |
| Data flow | CCTV frames → 4G → state cloud. Processed by open-source models on government GPUs. |
| Pros | Centralized management. No per-center GPU cost. Easier updates. |
| Cons | Depends on 4G connectivity. Slight latency. |
| DPDP compliance | Data stays within government infrastructure. NIC/State DC is a government instrumentality. |
| Best for | Routine assessments where slight latency is acceptable. |

**Option C: Hybrid (Recommended)**

- Lightweight processing on terminal CPU (face, gaze, audio) — works offline
- CCTV processing on state cloud via 4G — only back-facing video, no PII
- Advanced analysis (post-exam deep review) on state cloud with GPU
- Fallback: if 4G is down, CCTV frames are queued locally and uploaded when connection resumes

**Why not commercial cloud (AWS/GCP/Azure)?**
- Data would be processed on servers controlled by a foreign corporation
- Even if servers are in India (Mumbai region), the cloud provider has theoretical access
- DPDP's state instrumentality exemptions are strongest when the government controls the entire stack
- Commercial cloud adds a data processor relationship that requires contractual safeguards under Section 8
- Open-source on government cloud avoids all of this

### 3.7 What About Masking / Anonymization?

**We don't need to mask anything.** Here's why:

| Data Type | Why Masking Isn't Needed |
|-----------|------------------------|
| Webcam data | Never transmitted. Processed locally, discarded after exam. There's nothing to mask because the data never goes anywhere. |
| CCTV data | Already inherently anonymized — back-facing camera shows no faces. Only body silhouettes and positions. |
| Event metadata | Contains no personal data. Just terminal IDs, timestamps, event types, confidence scores. |
| Dashboard | Shows alerts and CCTV (back-facing). No facial data displayed. |

**The architecture IS the privacy protection.** By design, personal data never leaves the terminal where it's collected. There's no data in transit or at rest that needs masking because biometric data exists only in RAM during the exam session.

The only scenario where masking would apply: if we wanted to show webcam snapshots on the dashboard during alerts. In that case, we could:
- Blur/anonymize faces in evidence snapshots before transmitting to dashboard
- Or keep snapshots on the terminal only, viewable via direct authenticated access to the terminal

**Recommendation:** Don't transmit webcam snapshots to the dashboard at all. Keep them local. If an administrator needs to review, they access the specific terminal's evidence store through an authenticated, audited connection.

---

## 4. Summary: The Compliance Argument

### Legal Basis (Three-Layered)

1. **Rule 11 Education Exemption:** Educational institutions may track and monitor children for educational activities. Exam proctoring = educational activity.
2. **Section 7(b)-(c) State Function:** The dept. conducts exams under statutory authority. This is a lawful state function.
3. **Affirmative Consent:** Students acknowledge privacy notice and consent at registration — belt-and-suspenders.

### Architecture Is the Compliance

| DPDP Requirement | How Our Architecture Satisfies It |
|-----------------|----------------------------------|
| Consent / legitimate use | Triple legal basis (education exemption + state function + consent) |
| Data minimization | Only metadata transmitted. Biometrics processed locally, never stored. |
| Purpose limitation | Data used only for exam integrity. No secondary use. |
| Security | Encryption, access controls, hash-chain logs, network isolation. |
| Retention limits | Face data = RAM only (deleted at exam end). Logs = 90-day auto-delete. |
| Breach notification | Minimal attack surface (no centralized biometric database to breach). |
| Cross-border transfer | Zero. All processing on government infrastructure within India. |
| Children's data | Education exemption applies. Additional safeguards in place. |
| Transparency | Privacy notice in Telugu + English at registration. |
| Grievance redressal | Complaint mechanism built into the system. 90-day response. |
| No vendor data access | Open-source models. Government controls all infrastructure. |

### What Makes This Stronger Than Commercial Alternatives

| Commercial Proctoring | ExamGuard |
|----------------------|-----------|
| Video uploaded to vendor cloud | Video never leaves terminal |
| Vendor has access to biometric data | No vendor access — government controls everything |
| Cross-border data transfer risk | Zero — all processing in India on government infra |
| Black-box AI, can't audit for bias | Open-source, fully auditable |
| Per-exam licensing cost | Zero software cost |
| Data retention at vendor's discretion | Government controls retention & deletion |
| ProctorU had 500K student data breach | No centralized biometric database to breach |

---

## 5. Risks & Remaining Considerations

| Risk | Mitigation |
|------|-----------|
| **DPDP Rules not yet fully notified** | Our architecture is conservative — compliant even under strictest interpretation. |
| **Data Protection Board hasn't set precedents** | We follow the text of the law, the rules, and best practices. |
| **Government exemptions could be challenged** | We don't rely solely on exemptions — we also obtain consent and minimize data. |
| **Bias in AI models** | Open-source allows testing across demographics. We include bias metrics in our confusion matrix. |
| **Evidence snapshots contain PII** | Keep on terminal, encrypted, auto-deleted. Never centralized. |
| **Edge server compromise** | Hash-chain logs detect tampering. Encrypt evidence store. Physical security at exam center. |
| **Student/parent complaints** | Built-in grievance mechanism. Transparent privacy notice. Right to access their own event log. |

---

## 6. Implementation Checklist

- [ ] Privacy notice screen (Telugu + English) in exam client
- [ ] Consent capture UI with affirmative action (checkbox + button)
- [ ] Local-only face processing (no network calls for webcam data)
- [ ] Event metadata schema (no PII in transmitted data)
- [ ] TLS 1.3 for all network communication
- [ ] AES-256 encryption for evidence store
- [ ] Hash-chain implementation for tamper-proof logs
- [ ] Role-based access control on dashboard
- [ ] Audit logging for all dashboard actions
- [ ] Auto-deletion scheduler (90-day retention for logs, immediate for biometrics)
- [ ] Data processing agreement template (for department to sign with us)
- [ ] Privacy Impact Assessment document
- [ ] Grievance redressal workflow in dashboard
- [ ] Bias testing across skin tones with documented results
- [ ] DPDP compliance documentation for department records
