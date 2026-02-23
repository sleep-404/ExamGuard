# AI-Enabled Exam Proctoring: Comprehensive Research Report

**Date:** 2026-02-23
**Purpose:** Product research for ExamGuard hackathon project

---

## Table of Contents
1. [Existing Competitors & Products](#1-existing-competitors--products)
2. [Technical Approaches Used](#2-technical-approaches-used)
3. [Cutting Edge / State of the Art (2024-2026)](#3-cutting-edge--state-of-the-art-2024-2026)
4. [Feasibility for Our Constraints](#4-feasibility-for-our-constraints)
5. [Common Pitfalls & Criticisms](#5-common-pitfalls--criticisms)

---

## 1. Existing Competitors & Products

### 1.1 Major Commercial Players

#### Proctorio
- **Market Position:** One of the largest globally; 80M+ exams proctored since 2013; ~4M weekly active users
- **Features:** Chrome extension-based; facial recognition, real-time behavior analysis (eye movement, keyboard anomalies, head movements), multiple face detection, pixel-level screen analysis, typing pattern biometrics, audio analysis
- **Pricing:** ~$5/exam attempt for basic automated monitoring; 99.991% uptime
- **Strengths:** Fully customizable monitoring levels; deep Chrome integration
- **Weaknesses:** Privacy controversies; sued researchers who criticized the product (DMCA abuse); relied on OpenCV facial detection which showed racial bias
- **Source:** https://www.eklavvya.com/blog/best-proctoring-solution/

#### ProctorU (Meazure Learning)
- **Market Position:** Dominant in US higher education; hybrid AI + live human proctoring
- **Features:** AI continuous monitoring with human escalation; LMS integration (Canvas, Blackboard, Moodle); detailed analytics/reporting
- **Pricing:** Live proctoring ~$20/hour; hybrid $15-$35/exam
- **Notable:** Announced in May 2021 it would stop selling fully-automated proctoring without human review, after finding only ~10% of faculty reviewed flagged videos
- **Data Breach:** July 2020 breach exposed ~500,000 student records
- **Source:** https://www.proctoru.com

#### Honorlock
- **Market Position:** Growing mid-market; AI + "Live Pop-In" human proctoring
- **Features:** AI flags suspicious activity, human proctors intervene in real-time; mobile device detection; LMS integration; rapid deployment (days not weeks)
- **Pricing:** Flat-rate $4-$12/exam depending on institution size
- **Source:** https://blog.autoproctor.co/top-10-proctoring-tools-ranked-compare-features-and-pricing/

#### Respondus (LockDown Browser + Monitor)
- **Market Position:** "Gold standard" for browser-based exam security; 2,000+ institutions
- **Features:**
  - LockDown Browser: blocks other apps, disables copy/paste, screen sharing, virtual machines, printing, keyboard shortcuts
  - Respondus Monitor: adds webcam AI monitoring on top of LockDown Browser
- **Pricing:** Annual licensing $2,995-$11,995 depending on institution size; annual LockDown Browser license includes 200 free Respondus Monitor seats
- **Limitation:** Only secures the primary computer; secondary devices (phones, tablets) remain unmonitored
- **Source:** https://slashdot.org/software/comparison/Honorlock-vs-Proctorio-vs-Respondus-Monitor/

#### ExamSoft (ExamMonitor)
- **Market Position:** Enterprise-focused; dominant in professional licensing (medical, legal bar exams)
- **Features:** AI-powered video/audio surveillance; optional live human proctoring; web-based and on-premises deployment; detailed flagging with timestamps
- **Pricing:** Custom enterprise pricing
- **Notable:** California Bar exam 2020 -- 3,190 of 8,920 test-takers (36%) were flagged, mostly for technical issues
- **Source:** https://www.wecreateproblems.com/blog/proctoru-alternatives

### 1.2 India-Specific Solutions

#### Eklavvya (India's leading platform)
- **Scale:** 32M+ exams completed; 52+ exam types; 50,000+ simultaneous test-takers capacity
- **Features:** AI proctoring, 360-degree monitoring, secure browser, human proctoring, Moodle LMS integration, government ID verification
- **Technical:** Proprietary CNNs for facial recognition, RNNs for behavioral pattern analysis, NLP for audio analysis; trained on 10M+ exam sessions; claims 95% detection accuracy, <5% false positives
- **Pricing:** Credit-based system (e.g., 1000 credits = 1000 candidates for 1 exam); tiered editions (Standard, Professional, Enterprise)
- **Source:** https://www.eklavvya.com/pricing/

#### Mercer Mettl
- **Scale:** 32M+ proctored interactions; 200,000 concurrent assessments/day; 600+ customers, 700+ academic institutions
- **Features:** Fully automated, AI-assisted post-review, live dual-camera, hybrid modes; 3-point authentication (email + OTP + ID verification); credibility index scoring
- **Recognition:** Gartner leader in remote proctoring; G2 global leader
- **Pricing:** Custom enterprise pricing
- **Source:** https://mettl.com/en/online-remote-proctoring/

#### TCS iON Digital Assessment
- **Market Position:** Comprehensive platform by Tata Consultancy Services
- **Features:** Online + offline hybrid assessment; machine-based proctoring (multiple faces, gaze, unauthorized materials); supports national/zonal/regional exam administration; multiple proctoring levels (none, basic lockdown, remote human, machine-based)
- **Strengths:** Scale infrastructure for government exams across India

#### Talview
- **Features:** 7 layers of security including facial recognition, room scanning, secure browser, AI behavioral analysis; "Alvy" AI agent claims 8x more effective detection; deepfake identity spoofing detection
- **Pricing:** Custom; integrity scores within 24 hours
- **Source:** https://www.capterra.in/directory/33255/proctoring/software

#### AutoProctor
- **Features:** Integrates with Google Forms and Microsoft Forms; facial detection, audio analysis, screen tracking; violation reports and trust scores
- **Pricing:** Pay-as-you-go starting at $15/month
- **Unique:** Downloads detection algorithms locally before exams -- continues working even if internet drops
- **Source:** https://blog.autoproctor.co/top-10-proctoring-tools-ranked-compare-features-and-pricing/

#### Meivy
- **Status:** Limited public information available; appears to be a niche/emerging provider
- **Features:** Likely standard AI proctoring features (face tracking, audio, browser lockdown) based on industry patterns

#### Other India Players
- **Unstop (formerly D2C):** 360-degree next-gen proctoring for hiring assessments
- **Proctor360:** India-based with AI proctoring capabilities
- **Invigilate.in:** Remote proctoring focused on Indian market
- **CertifyAssessment:** Remote proctoring software

### 1.3 Open-Source Tools

#### Safe Exam Browser (SEB)
- **Type:** Open-source (Mozilla Public License); freeware
- **Platforms:** Windows 8.1/10/11, macOS 10.6+, iOS 9.3.5+
- **Features:** Converts any PC into secure workstation; prevents other app access; disables keyboard shortcuts; blocks screen sharing/virtual machines/printing; URL filtering; virtual machine detection
- **Integrations:** Moodle, ILIAS, OpenOlat, Inspera Assessment; optional video conferencing (Jitsi Meet, Zoom)
- **Key Limitation:** No webcam/microphone required (no identity verification); only locks down the primary device
- **Strength:** Privacy-preserving; no data collection; fully auditable code
- **Source:** https://docs.openolat.org/manual_how-to/SEB/SEB/

#### OpenOLAT
- **Type:** Open-source LMS with integrated assessment and SEB configuration
- **Features:** Assessment mode creation, SEB activation, config file generation within LMS

#### Notable Open-Source GitHub Projects

| Project | Stars | Tech Stack | Features |
|---------|-------|-----------|----------|
| [vardanagarwal/Proctoring-AI](https://github.com/vardanagarwal/Proctoring-AI) | Popular | OpenCV, TensorFlow, dlib | Eye tracking (7.1 FPS on i5), mouth detection (7.2 FPS), person+phone detection (1.3 FPS via YOLOv3), head pose (8.5 FPS), face spoofing (6.9 FPS), audio via speech recognition |
| [The-Online-Exam-Proctor](https://github.com/aungkhantmyat/The-Online-Exam-Proctor) | 21 stars | YOLOv8, MediaPipe, OpenCV, dlib, Flask, MySQL | Face verification, liveness, distracted movement, multiple faces, prohibited keys, tab switch, noise detection |
| [OpenProctor](https://github.com/Codseg/openproctor) | New (2024) | TensorFlow.js BlazeFace, WebRTC | Browser SDK; camera/mic/screen capture; real-time face detection; stream uploading |
| [exam-cheating-detection](https://github.com/aarambhtech/exam-cheating-detection) | Recent | OpenCV, MediaPipe, FaceNet-PyTorch, MTCNN, Flask | Eye movement tracking, gaze analysis, mouth movement, multi-face detection |
| [serengil/deepface](https://github.com/serengil/deepface) | 55.6K stars | VGG-Face, FaceNet, OpenFace, ArcFace | Lightweight face recognition + attribute analysis; real-time webcam support |
| [ageitgey/face_recognition](https://github.com/ageitgey/face_recognition) | 55.6K stars | dlib (deep learning) | 99.38% accuracy on LFW; simple API; Raspberry Pi support |

---

## 2. Technical Approaches Used

### 2.1 Face Detection / Recognition

| Model | Size | CPU FPS (i5) | Accuracy | Notes |
|-------|------|-------------|----------|-------|
| **MediaPipe BlazeFace** | 2-4 MB | 70-80 FPS | Good | Best for real-time on CPU; 478 3D landmarks |
| **MobileFaceNet** | 4 MB | Real-time (18ms inference) | 99.55% LFW | Purpose-built for mobile; 1.25M params; ArcFace loss |
| **OpenCV DNN (Caffe)** | ~5 MB | 17.5-19.5 FPS (quantized) | Good for frontal | Mature; quantized uint8 version available |
| **dlib HOG + SVM** | ~100 MB | 7-9 FPS | Good frontal only | Deterministic; 68 landmarks; struggles with angles |
| **InsightFace/ArcFace** | 4-100+ MB | Varies | 99.55%+ LFW | State-of-art recognition; OpenVINO can give 5x speedup |
| **DeepFace** | Varies | Real-time capable | 97.4%+ | Wrapper for VGG-Face, FaceNet, ArcFace; easy API |
| **YuNet** | ~0.3 MB (75,856 params) | 1.6ms/frame on i7 | 81.1% WIDER FACE hard | Ultra-lightweight; best efficiency for face-only detection |

**Approach Summary:**
- Registration: Capture reference face embedding during enrollment
- Continuous verification: Compare live face embedding against stored reference
- Liveness detection: Micro-expression analysis, blink patterns, 3D facial mapping to prevent photo/video spoofing

### 2.2 Gaze Tracking / Eye Tracking

- **MediaPipe Face Mesh:** 478 3D facial landmarks including iris positioning; gaze direction from iris displacement; normalized iris displacement >0.4 flagged as suspicious
- **Head Pose Estimation:** Pitch/yaw/roll angles from facial landmarks; detects head turns away from screen
- **Approach:** Process small eye-region patches (~256x256) after face detection; <10ms inference per frame on CPU
- **Calibration needed:** Baseline gaze patterns during exam initialization; flag anomalies vs. normal reading behavior
- **False positive concerns:** ADHD students look away more; neurodivergent individuals have different attentional patterns

### 2.3 Audio Monitoring

- **Voice Activity Detection (VAD):** Detect speech vs. silence
- **Speaker Diarization:** Determine who is speaking; detect multiple voices (Diarization Error Rate <5% in optimal conditions)
- **MFCC Analysis:** Mel-frequency cepstral coefficients for compact audio representation
- **Speech Recognition:** Detect voice assistant commands ("Hey Siri", "OK Google")
- **Lightweight approaches:**
  - Spectrogram-based analysis + lightweight convolutional autoencoders
  - LightESD framework: statistical learning-based anomaly detection for edge devices
  - SpeechCompass: real-time audio localization on low-power microcontrollers (<20ms)
  - HILCodec: streaming, lightweight audio processing using only MAC operations

### 2.4 Object Detection

| Model | Params | Size | CPU FPS | mAP | Best For |
|-------|--------|------|---------|-----|----------|
| **YOLOv8-nano** | ~3M | ~6 MB | 10-25 FPS | ~37% COCO | General object detection |
| **YOLOv5-nano** | ~1.9M | ~1.9 MB | 10-25 FPS | ~28% COCO | Lightest YOLO variant |
| **Tiny-YOLO** | - | <50 MB | ~4 FPS (RPi) | 23.7% COCO | Embedded devices |
| **MobileNet-SSD** | ~5.9M | ~5.9 MB | 20-25 FPS | Competitive | Phone detection |
| **LMS-DN (MobileNetV2-SSDLite)** | - | Small | 12ms avg | High for phones | Phone detection specifically; robust to occlusion |
| **EfficientNetB0** | ~5.3M | <10 MB | Good | 95.49% CIFAR-10 | Transfer learning for custom objects |

**Detected objects:** Mobile phones, textbooks, notes, earpieces, additional persons
**Custom training:** Roboflow provides annotation + training infrastructure for domain-specific items

### 2.5 Multiple Person Detection

- **YOLOv8 for person detection:** ~94% precision, ~92% recall in exam rooms; 120ms/frame latency
- **Frame differencing:** Simple baseline detection for additional persons entering the room
- **Person counting:** Track count per frame; alert when >1 person at a terminal

### 2.6 Browser Lockdown Techniques

- Block other tabs/applications
- Disable keyboard shortcuts (Alt+Tab, Ctrl+C, Ctrl+Alt+Del)
- Prevent screen sharing and virtual machine execution
- Block printing and screen capture
- Remove browser navigation elements
- URL filtering for approved resources only
- Disable task bar access
- **Limitation:** Only secures the primary device

### 2.7 CCTV + Webcam Hybrid Approaches

- **Dual-camera proctoring (Mercer Mettl):** Desktop webcam + mobile device as secondary camera
- **360-degree monitoring:** Specialized devices for complete room views
- **Back-facing CCTV approach:**
  - Fixed camera captures entire exam hall
  - Person re-identification techniques to track individuals across camera views
  - Seat assignment verification against registered positions
  - Temporal analysis: track movement patterns over exam duration
- **Person Re-Identification (Re-ID):** Cross-camera retrieval to match same person across different views; active research area (CVPR/ECCV 2024 workshops)

### 2.8 Edge vs. Cloud Processing

| Aspect | Edge (Local) | Cloud |
|--------|-------------|-------|
| **Latency** | Immediate (<100ms) | Network-dependent (200ms-2s) |
| **Privacy** | Data stays local | Raw video transmitted remotely |
| **Bandwidth** | Minimal network use | 3-10 Mbps per stream |
| **Compute** | Limited by local hardware | Virtually unlimited |
| **Models** | Lightweight only (MediaPipe, MobileNet) | Large models (ResNet, ViT) |
| **Cost** | One-time hardware | Ongoing per-minute charges |
| **Best for** | Face detection, gaze, audio VAD | Scene understanding, activity recognition, post-exam review |

**Optimal Hybrid:** Edge for real-time detection + alerts; cloud for CCTV analytics + post-exam deep analysis

---

## 3. Cutting Edge / State of the Art (2024-2026)

### 3.1 Multimodal AI for Proctoring

- **Weighted Fusion Models:** Integrate signals from video, audio, and interaction logs; significantly outperform unimodal systems in accuracy and fairness
- **CNN + RNN architectures:** Process 3-frame sequences to capture temporal patterns; reduce false positives
- **Experimental:** Gemini used to analyze screen recordings + webcam captures during exam sessions (https://www.kaggle.com/code/orvalg/using-gemini-as-a-proctor-for-online-exams)
- **GPT-4V / Gemini Vision:** Not yet deployed in production for real-time proctoring; used experimentally for post-exam analysis of recorded sessions
- **Vision-Language Model Performance:** GPT-4 achieved 51-72% on medical licensing exams; Gemini-pro-vision 41-59% -- approaching but not matching human performance

### 3.2 LLM-Based Cheating Detection

- **Response text analysis:** Vocabulary sophistication, sentence structure, thematic coherence to detect AI-generated answers
- **Keystroke + text analysis:** Detect copy-paste from generative AI tools vs. original typing
- **Arms race:** LLMs improving faster than detection systems; Turnitin's 4% sentence-level false positive rate is 4x their initial claim
- **Source:** https://www.ijraset.com/research-paper/ai-driven-proctoring-system-for-fair-online-examinations

### 3.3 Privacy-Preserving Approaches

- **Federated Learning:**
  - Train models on edge devices; share only model parameters, not raw data
  - Demonstrated comparable accuracy to centralized models (no statistically significant difference: t(8)=2.29, p=0.051)
  - Secure aggregation protocols merge updates without revealing individual contributions
  - Source: https://arxiv.org/html/2503.11711v1

- **Differential Privacy:** Add calibrated noise to model updates; prevents individual data reconstruction from aggregates
- **Edge Computing:** Process biometrics locally; transmit only anomaly scores/flags (not raw video)
- **Data Minimization:** Collect only minimum data needed for monitoring
- **Entropy-Adaptive Differential Privacy:** Dynamically adjust noise levels based on output information content

### 3.4 Advanced Behavioral Analytics

- **Micro-expression detection:** Stress, concentration, distress patterns
- **Pulse rate monitoring:** Subtle skin color variations in facial regions
- **Speech emotion recognition:** Voice stress analysis
- **Keystroke dynamics:** Biometric signature from typing rhythm and pressure
- **Deepfake detection:** Talview's 7-layer security specifically targets deepfake identity spoofing

### 3.5 Blockchain for Immutable Records

- Every exam action recorded in distributed ledger with cryptographic timestamps
- Instant credential verification without centralized repositories
- Supports rapid dispute resolution with definitive timestamped evidence
- **Practical approach:** Private institutional blockchains (not fully decentralized)
- Tradeoff: immutability makes error correction difficult; high computational overhead

### 3.6 Low-Bandwidth Proctoring Innovation

- Adaptive video streaming: reduces bandwidth from 3-5 Mbps to 15-20 MB per entire exam
- Upload-after-completion models rather than continuous streaming
- Mobile-first proctoring via smartphones
- Critical for India's connectivity challenges

---

## 4. Feasibility for Our Constraints

### 4.1 Constraint Summary

| Constraint | Value |
|-----------|-------|
| Terminals per hall | 10-40 |
| CCTV cameras | 1 per room (back-facing) |
| Internet | 4G connectivity |
| Processing | Local on terminals (webcam); Cloud (CCTV) |
| Human invigilator | None |
| Budget | ~30,000 INR per center |
| POC | 30 students, 10-min sessions, detect 8/10 malpractice events |

### 4.2 Recommended Architecture

```
+------------------+      LAN        +------------------+
| Terminal 1       |<--------------->|                  |
| - Webcam         |                 |  Local Edge      |
| - Mic            |                 |  Server          |
| - Exam Client    |                 |  (aggregation,   |
+------------------+                 |   alerting,      |
        ...                          |   logging)       |
+------------------+                 |                  |
| Terminal N       |<--------------->|                  |
| - Webcam         |                 +--------+---------+
| - Mic            |                          |
+------------------+                     4G Upload
                                              |
+------------------+                 +--------v---------+
| CCTV Camera      |---local-LAN--->| Cloud Analytics   |
| (back of room)   |   or 4G        | (CCTV processing, |
+------------------+                 |  dashboard,       |
                                     |  evidence store)  |
                                     +-------------------+
```

**Key Design Decisions:**
- **Per-terminal local processing** for webcam (face, gaze, audio) -- privacy-preserving, low latency
- **Edge server** for aggregation, alert correlation, tamper-proof logging
- **Cloud processing** for CCTV feed -- more compute-intensive person tracking/re-ID
- **Screenshot-based approach** for CCTV (1 frame/sec) rather than continuous streaming to save bandwidth

### 4.3 Detection Tasks: Model Recommendations

#### Face Detection + Verification (Local, per terminal)
- **Detection:** MediaPipe BlazeFace (2-4 MB, 70-80 FPS on i5)
- **Verification:** MobileFaceNet with ArcFace loss (4 MB, 99.55% LFW accuracy)
- **Approach:** Register face at enrollment; continuous comparison during exam
- **Detects:** Candidate absence, seat swaps (face mismatch), impersonation

#### Gaze Deviation (Local, per terminal)
- **Model:** MediaPipe Face Mesh (30-60 FPS on i5 CPU)
- **Approach:** 478 3D landmarks; iris displacement calculation; head pose estimation (pitch/yaw/roll)
- **Threshold:** Flag sustained gaze away from screen (>3 seconds); iris displacement >0.4

#### Multiple Faces (Local, per terminal)
- **Model:** MediaPipe or OpenCV DNN face detector
- **Approach:** Count faces per frame; alert if >1 face detected for sustained period (>5 frames)

#### Audio Anomaly Detection (Local, per terminal)
- **Approach:** Minimal compute
  1. Voice Activity Detection (VAD) using WebRTC VAD or simple energy thresholding
  2. MFCC feature extraction at 8kHz sampling rate
  3. Simple 1D-CNN on spectrograms for background voice detection
  4. Keyword detection for voice assistant commands
- **Expected accuracy:** 80-85% (limited by ambient noise)

#### Unauthorized Materials / Phone Detection (Local, per terminal)
- **Model:** YOLOv5-nano (1.9 MB) or YOLOv8-nano (6 MB)
- **Alternative:** MobileNetV2-SSDLite for phone-specific detection (12ms avg)
- **Training:** Custom fine-tuning on prohibited items dataset (~500 labeled images per class minimum)
- **Expected FPS:** 10-25 on i5 CPU
- **Expected accuracy:** 87-92%

#### Seat Swap Detection via CCTV (Cloud)
- **Approach:**
  1. Capture CCTV frames at 1 fps (saves bandwidth: ~50KB x 1fps = ~0.4 Mbps)
  2. Person detection using YOLOv8 in cloud
  3. Person Re-Identification: track individuals across frames using appearance features
  4. Seat assignment mapping: each person assigned a seat position at exam start
  5. Flag position changes over time
- **Cloud service:** AWS Rekognition Video ($0.10/min first 1000 min; $0.00817/min for streaming)
- **Alternative:** Google Cloud Video Intelligence ($0.10/min; 1000 min free/month)

### 4.4 Budget Breakdown (30,000 INR per center)

| Item | Cost (INR) | Notes |
|------|-----------|-------|
| Edge server (Raspberry Pi 4, 8GB or mini-PC) | 6,000-8,000 | Aggregation + logging |
| Network switch | 3,000-4,000 | Connect terminals + CCTV |
| CCTV camera (1080p, PoE) | 2,500-3,500 | Back of room, wide angle |
| Webcams (if not built-in) | 0-10,000 | Assume terminals have webcams |
| Cabling + misc hardware | 2,000-3,000 | Ethernet, mounts, etc. |
| Cloud compute (POC period) | 3,000-5,000 | AWS/GCP for CCTV analytics |
| Software/licensing | 0 | All open-source models |
| **Total** | **16,500-34,500** | Within budget if terminals have webcams |

### 4.5 Bandwidth Requirements (4G feasibility)

| Stream | Bandwidth | Notes |
|--------|-----------|-------|
| CCTV to cloud (1 fps, 720p JPEG) | ~0.4 Mbps | 50KB/frame x 1fps |
| CCTV to cloud (continuous 15fps 720p H.264) | 2-3 Mbps | Compressed |
| Terminal alerts to edge server | Negligible | JSON events only |
| Edge server to cloud (alerts + metadata) | <0.5 Mbps | No raw video |
| **Total** | **3-4 Mbps** | Feasible on 4G |

All webcam processing happens locally -- no video leaves the terminal, preserving privacy and saving bandwidth.

### 4.6 POC Implementation Plan (30 students, 10-minute sessions)

**Phase 1: Core Detection (Week 1-2)**
1. Deploy lightweight exam client on 30 terminals (Python + OpenCV + MediaPipe)
2. Implement face detection + verification using MediaPipe + MobileFaceNet
3. Implement gaze tracking using MediaPipe Face Mesh
4. Basic audio VAD for multi-voice detection
5. Set up hash-chain event logging on edge server

**Phase 2: Object Detection + CCTV (Week 3-4)**
1. Train YOLOv5-nano on phone/book detection dataset
2. Deploy CCTV feed processing (cloud or edge server)
3. Implement seat position tracking
4. Test with synthetic malpractice scenarios

**Phase 3: Integration + Testing (Week 5-6)**
1. Integrate all detection modules
2. Dashboard for real-time alerts
3. Run 10-minute test sessions with planted malpractice events
4. Target: detect 8/10 events

**Expected Detection Rates:**

| Malpractice Event | Expected Detection | Approach |
|-------------------|-------------------|----------|
| Seat swap | 85-90% | Face verification mismatch + CCTV position tracking |
| Multiple faces | 90-95% | MediaPipe/OpenCV multi-face detection |
| Candidate absence | 95%+ | No face detected for sustained period |
| Gaze deviation | 80-85% | MediaPipe iris tracking + head pose |
| Suspicious audio | 75-85% | VAD + speaker count |
| Unauthorized phone | 85-90% | YOLOv5-nano object detection |
| Unauthorized books/notes | 80-85% | YOLOv5-nano with custom training |
| Impersonation | 90-95% | Face verification against enrollment photo |

### 4.7 Tamper-Proof Evidence Logs

**Recommended: Hash Chain Architecture**
- Each event includes hash of previous event, creating mathematical dependency chain
- Canonical JSON formatting with sorted keys for deterministic hashing
- Event structure: `{event_id, trace_id, timestamp, event_type, payload, previous_hash, event_hash}`
- First event references "GENESIS" as previous hash
- Any tampering breaks the chain at the modified point

**Digital Signatures** for high-severity events:
- PKI-based signatures on flagged events
- Provides non-repudiation (system cannot deny generating a flag)

**Implementation:** Use SHA-256 hashing; store logs on edge server with periodic sync to cloud; blockchain integration can be deferred to later phase.

---

## 5. Common Pitfalls & Criticisms

### 5.1 Privacy Concerns & Legal Issues

#### India - DPDP Act 2023
- Requires **explicit consent** before collecting biometric data
- Special protections for children (verifiable parental consent required)
- Mandatory data protection officers for large-scale data handling
- Strict data retention limits; notification required for breaches
- Penalties up to **500 crore INR (~$60M)** for violations
- Source: https://www.ey.com/en_in/insights/cybersecurity/decoding-the-digital-personal-data-protection-act-2023

#### EU - GDPR + AI Act
- AI Act classifies education monitoring as **high-risk AI** (effective 2026)
- Requires: documented risk management, meaningful human oversight, explainable outputs
- Penalties up to 35M EUR or 7% of global revenue
- Source: https://integrityadvocate.com/resource-center/blog/the-eu-ai-act-gdpr-and-what-they-mean-for-online-proctoring-and-assessment-security/

#### US - BIPA (Illinois Biometric Information Privacy Act)
- Multiple class-action lawsuits against ProctorU, Respondus, Examity
- Damages up to $5,000 per willful violation per class member
- ProctorU allegedly retained biometric data since 2012 without justification

#### Key Principle
- **Surveillance must be proportionate** to legitimate aim
- **Least restrictive means** must be employed
- **Ogletree v. Cleveland State University:** Court found room scans violated 4th Amendment

### 5.2 Racial and Skin-Tone Bias

**The evidence is severe:**
- **University of Louisville study (357 students):** Students with darker skin tones flagged at **6x the rate** of lighter-skinned peers
  - Darkest skin: flagged 7.64% of exam time
  - Lightest skin: flagged 1.56% of exam time
  - Face detection rate: 78% for darker skin vs. 92% for lighter skin
  - Black students 2.5x more likely to receive "low facial detection" flag
- **MIT/Stanford study (3 commercial systems):** Gender classification error rates:
  - Light-skinned men: never >0.8%
  - Darker-skinned women: 20.8%, 34.5%, 34.7%
- **Root cause:** Training datasets skewed (one system: 77% male, 83% white)
- **Proctorio specifically** used OpenCV facial detection; Maryland Test Facility found clear bias with failure rates increasing from lightest to darkest skin
- Source: https://www.frontiersin.org/journals/education/articles/10.3389/feduc.2022.881449/full

### 5.3 False Positive Rates

- **California Bar (ExamSoft, 2020):** 36% of test-takers flagged; most for technical issues (Lenovo laptop mic incompatibility, intermittent eye positioning)
- **Turnitin AI detection:** Claimed <1% false positive; revised to **4% sentence-level false positive** after real-world deployment
- **Non-native English speakers:** **61% false positive rate** for AI content detection vs. <10% for native speakers
- **GPTZero:** Advertised 99% accuracy; independent test showed **80% accuracy with 10% false positive rate**
- **University of Iowa audit:** Only 14% of faculty reviewed Proctorio results they received
- **Maryland researchers' benchmark:** Acceptable false positive for high-stakes systems should be **0.01% (1 in 10,000)** -- currently "impossible" with existing methods
- Source: https://lawlibguides.sandiego.edu/c.php?g=1443311&p=10721367

### 5.4 Student Pushback & Accessibility

**Anxiety and Performance Impact:**
- Students taking lockdown-browser exams or under AI webcam surveillance reported **elevated anxiety and worse academic performance**
- Students with high trait anxiety: discomfort directly correlated with lower exam scores
- Hypervigilance about own behavior becomes itself a barrier to performance

**Disability Barriers:**
- ADHD students: naturally look away more; flagged as suspicious
- Students with tics/involuntary motor movements: false cheating accusations
- Diabetic students: food consumption flagged as violation
- Blind/low-vision: difficulties with face-to-camera positioning
- Deaf/hard of hearing: audio-dependent systems cannot accommodate
- Screen reader users: cannot navigate proctoring interfaces
- Religious head coverings (hijabs): systems struggle with partial face coverage

**Equity Issues:**
- Students in shared housing: family members flagged as multiple persons
- Low-income families: shared computers, unreliable internet
- No private testing space: background activity flagged

**Documented extreme cases:**
- Students unable to use bathrooms during multi-hour exams, resulting in urination at desks
- Female students uncomfortable with male proctors monitoring them at home
- Students forced to remove religious head coverings for facial recognition

### 5.5 Legal Challenges and Lawsuits

| Company | Lawsuit | Allegation |
|---------|---------|-----------|
| **ProctorU** | BIPA class action | Collected facial geometry, eye movements, keystroke biometrics without consent; 500K student data breach |
| **Respondus** | BIPA class action (Northwestern, DePaul, Lewis U students) | Facial recognition data collected without informed consent |
| **Examity** | BIPA class action (Western Governors U) | Biometric data retained beyond original purpose |
| **Proctorio** | EFF lawsuit (behalf of researcher Erik Johnson) | DMCA abuse to suppress criticism of the software |
| **Proctorio** | Lawsuit against Ian Linkletter (UBC) | 5-year copyright battle over sharing training videos; withdrawn in 2024 |
| **ExamSoft** | California Bar scrutiny | State auditor found contract awarded without proper value evaluation |

Source: https://www.eff.org/deeplinks/2021/06/long-overdue-reckoning-online-proctoring-companies-may-finally-be-here

### 5.6 How Companies Have Responded

- **ProctorU:** Stopped selling fully-automated proctoring (May 2021); requires human review of AI flags
- **Proctorio:** Partnered with BABL AI for bias audit; claims to have implemented new face-detection algorithm with "no measurable bias" (disputed by independent researchers)
- **Industry-wide framing:** "We don't make cheating determinations, we only flag anomalies" -- but algorithmic flags fundamentally shape what humans review
- **Context-aware AI:** Research shows reducing false positives by 25-30% through contextual behavior analysis
- **Human-in-the-loop:** Emerging best practice: AI flags + trained human review before any consequences
- **Institutions abandoning proctoring:** University of Michigan-Dearborn, McGill University dropped proctoring software entirely

### 5.7 Best Practices for Ethical Implementation

1. **Algorithmic impact assessments** before deployment
2. **Human review** for all high-stakes decisions (not just faculty, but trained reviewers with anti-bias training)
3. **Data minimization:** Collect only what's needed; clear retention limits with automatic deletion
4. **Transparency:** Clear explanation to students of what's collected, how it's used, how long retained
5. **Meaningful alternatives:** Allow students to opt for human proctoring or alternative assessments
6. **Clear appeal processes:** Access to evidence, explanations of flags, fair hearings
7. **Regular bias audits:** Test across demographic groups; follow 80/20 rule for fairness assessment
8. **Disability accommodations:** Design with accessibility from the start, not as afterthought
9. **Proportionality:** Match monitoring intensity to exam stakes (low-stakes = minimal monitoring)
10. **Edge processing by default:** Keep biometric data local; transmit only aggregate flags

---

## Key Sources

### Competitor & Market Research
- https://www.eklavvya.com/blog/best-proctoring-solution/
- https://blog.autoproctor.co/top-10-proctoring-tools-ranked-compare-features-and-pricing/
- https://www.proctor365.ai/affordable-pricing-guide-for-online-proctored-exam-software/
- https://mettl.com/en/online-remote-proctoring/
- https://www.eklavvya.com/pricing/
- https://www.proctoru.com

### Technical Research
- MobileFaceNet paper: https://arxiv.org/pdf/1804.07573.pdf
- MediaPipe face detection evaluation: https://www.diva-portal.org/smash/get/diva2:1574943/FULLTEXT01.pdf
- OpenCV Zoo benchmarks: https://github.com/opencv/opencv_zoo
- DeepFace library: https://github.com/serengil/deepface
- Proctoring-AI benchmarks: https://github.com/vardanagarwal/Proctoring-AI
- CNN+RNN proctoring model: https://educationaldatamining.org/EDM2025/proceedings/2025.EDM.short-papers.20/
- Federated learning for assessment: https://arxiv.org/html/2503.11711v1
- Gemini as proctor experiment: https://www.kaggle.com/code/orvalg/using-gemini-as-a-proctor-for-online-exams

### Bias & Privacy Research
- Skin-tone bias study: https://www.frontiersin.org/journals/education/articles/10.3389/feduc.2022.881449/full
- AI detection false positives: https://lawlibguides.sandiego.edu/c.php?g=1443311&p=10721367
- EFF proctoring reckoning: https://www.eff.org/deeplinks/2021/06/long-overdue-reckoning-online-proctoring-companies-may-finally-be-here
- EU AI Act implications: https://integrityadvocate.com/resource-center/blog/the-eu-ai-act-gdpr-and-what-they-mean-for-online-proctoring-and-assessment-security/
- India DPDP Act: https://www.ey.com/en_in/insights/cybersecurity/decoding-the-digital-personal-data-protection-act-2023
- Privacy concerns in proctoring: https://www.e-assessment.com/news/viewpoint/privacy-concerns-with-online-proctoring-and-how-to-address-them
- Student harm documentation: https://sparcopen.org/news/2021/higher-education-reckons-with-concerns-over-online-proctoring-and-harm-to-students/

### Cloud Analytics Pricing
- AWS Rekognition: https://aws.amazon.com/rekognition/pricing/ ($0.10/min first 1K; $0.00817/min streaming)
- Google Cloud Video Intelligence: https://cloud.google.com/video-intelligence/pricing ($0.10/min; 1000 min free/month)
- Azure Video Indexer: https://azure.microsoft.com/pricing/details/video-indexer/

### Person Re-Identification
- Survey: https://ietresearch.onlinelibrary.wiley.com/doi/full/10.1049/cvi2.12316
- Re-ID challenges: https://link.springer.com/article/10.1007/s42979-024-03271-9
- CCTV indoor detection model: https://universe.roboflow.com/cctvindoorperson/cctv-indoor-person-detection
