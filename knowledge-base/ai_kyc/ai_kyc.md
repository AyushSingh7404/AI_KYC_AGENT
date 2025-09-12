# 📘 AI-Powered KYC System Knowledge Base

> **Note:** This knowledge base is written for ingestion into a RAG system (ChromaDB). **Important operational constraint:** the KYC pipeline accepts **image formats only** for document uploads — **JPG/JPEG** and **PNG**. **PDFs are not accepted.** If you have a PDF (e.g., e-Aadhaar), convert the relevant page to an image and upload it as JPG/PNG.

---

## 1. Introduction to KYC
**Know Your Customer (KYC)** is the set of processes and controls organizations use to verify and monitor the identity of their customers. KYC protects businesses and users by preventing identity theft, financial crime, and regulatory breaches. Modern KYC combines regulatory controls with AI-driven automation to make verification faster, more accurate, and auditable.

AI KYC is a digitally automated identity-verification process that uses OCR, face recognition, liveness detection, and optionally video to confirm who you are. Compared to traditional/manual KYC, AI KYC automates extraction and matching, provides auditable logs, and scales to large user volumes while improving speed and fraud detection. The KB highlights that AI KYC still follows the same regulatory goals (prevent fraud, AML/CFT) but adds automated heuristics and machine checks to reduce manual effort and increase accuracy.

---

## 2. High-level Objectives & Global Standards
**Primary objectives**
- Verify that a customer is who they claim to be.
- Prevent financial crimes (AML/CFT).
- Meet local and international regulatory obligations.
- Maintain records for audit and compliance reviews.

**Standards & references**
- FATF Recommendations, Basel Committee guidance, EU AML Directives
- Country-specific rules (e.g., RBI/SEBI/UIDAI guidelines in India)
- Data protection (GDPR, regional privacy laws)
- Sanctions & watchlist screening (OFAC/UN/EU/Local lists)

---

## 3. AI KYC System Workflow (step-by-step)

### 3.1 Session start
- User opens the app/website or initiates via chatbot. System displays clear instructions and obtains explicit consent.
- A secure `session_id` is created and all subsequent actions are tied to it for traceability.

### 3.2 Document selection
- User chooses one government-issued ID: **Aadhaar, PAN, or Passport**.
- The UI must explain accepted formats (JPG/JPEG/PNG) and show example images for good/bad uploads.

### 3.3 Document upload (Images only — JPG/PNG)
- The system accepts **only image formats**: `*.jpg`, `*.jpeg`, `*.png`. **PDFs are explicitly rejected.**
- Uploads are protected with TLS and temporarily stored in access-controlled buckets for processing.
- Pre-upload client-side checks: file type, file size (e.g., <5 MB), image dimensions, and basic quality heuristics (no blur, not rotated).

### 3.4 Text extraction (OCR)
- Use **AWS Textract** (or equivalent) to extract structured fields: name, DOB, document number, address, MRZ fields for passports.
- Extracted fields are passed to validation logic (regex formats, checksums).

### 3.5 Face verification & liveliness
- User takes a **live selfie** with the camera. AWS Rekognition or equivalent performs face matching and liveliness/anti-spoof checks.
- Liveness may include challenge-response (blink/smile/turn), 3D depth cues, and anti-deepfake checks.

### 3.6 Video KYC (if required)
- A secure video session (AI-assisted or human officer) is used to verify original ID and live presence. The session and critical screenshots are recorded and stored as proof.

### 3.7 Final decision and audit trail
- The system aggregates results (OCR confidence, face match score, liveness pass, video officer notes) and produces a decision: **Approved / Manual Review / Rejected**.
- All actions are logged immutably for compliance review.

---

## 4. Document Requirements & Upload Rules (IMAGE ONLY)
- **Accepted documents:** Aadhaar, PAN, Passport (others configurable as per regulation: Driving Licence, Voter ID).
- **Accepted file types:** **JPG, JPEG, PNG only.** **PDF IS NOT ACCEPTED.** (Why? To enforce a single-image capture workflow, reduce multi-page ambiguity, and simplify OCR preprocessing pipelines.)
- **Quality requirements:** sharp image, no glare, all four corners visible, full document in frame, readable text and portrait photo.
- **File size & dimensions:** recommended max 5 MB, minimum 800×600 px for clear OCR and face crop extraction.
- **Client-side guidance:** show sample good/bad images; perform auto-rotation and auto-crop suggestions before upload.
- **Privacy note:** mask or redact unrelated personal information client-side if required by policy before upload (but prefer server-side redaction after ingestion under strict controls).

---

## 5. Text Extraction (AWS Textract) — Integration Notes
- **Input:** images (JPG/PNG). Since PDFs are not accepted, ensure Textract is used in image mode (document/text detection).
- **Field extraction strategy:** run document analysis to extract lines, key-value pairs, and (where present) table fields. Use post-processing rules to map fields to canonical names (e.g., `date_of_birth`, `doc_number`).
- **Validation:** regex and checksum for PAN and Aadhaar (UIDAI checksum) and MRZ parsing for passports. If confidence < threshold (e.g., 80%), flag for re-capture or manual review.
- **Edge cases:** multi-language content (Hindi/English) — configure Textract or add language-specific OCR fallback. If a scanned image uses a non-supported script, escalate to human review.
- **Metadata:** Save `textract_confidence` per field and include coordinates of text blocks for visual debugging and copy verification.

---

## 6. Face Recognition & Liveliness (AWS Rekognition) — Integration Notes
- **Face detection:** detect faces and crop ID photo region; extract face embeddings for both ID photo and selfie.
- **Face matching:** compute similarity score; set operational thresholds (example: >= 90% = pass, 80–90% = soft-pass with manual review, < 80% = fail).
- **Liveliness checks:** use challenge-response (user prompted to blink/turn), active movement detection, and passive liveness models (texture & motion analysis). Combine multiple signals to reduce false positives.
- **Anti-spoofing:** integrate anti-spoof models to catch printed photos, replays, and deepfakes.
- **Fallbacks:** If face comparison fails repeatedly, escalate to Video KYC or manual review. Allow human override with documented reason and screenshots.

---

## 7. Video KYC — Process & Compliance
- **When used:** mandated by regulators in many jurisdictions or used as fallback when automated checks fail.
- **Process steps:** initiate secure video link → verify ID on camera → ask challenge questions → capture screenshots & short clips → record final decision and officer annotation.
- **Recording & storage:** record the entire session, store encrypted video (S3 with SSE), and retain according to retention policy.
- **Regulatory checklist:** time-stamped recording, officer identity & credentials, uninterrupted recording, geolocation capture if required by law, and secure audit trail.
- **Privacy & transparency:** inform users that session is recorded and obtain consent; display officer credentials where applicable.

---

## 8. System Architecture & Components (High Level)
- **Frontend:** web/mobile interface + chatbot for guidance and image capture.
- **Gateway & Super-Agent:** a router that decides which backend agent to call (Document Agent, Face Agent, Video Agent, RAG Agent for KB queries).
- **Processing & AI services:** AWS Textract (OCR), AWS Rekognition (face & liveness), optional third-party anti-spoof services.
- **Storage:** encrypted object storage for images and videos; ephemeral storage for raw images until redaction and long-term archive.
- **Database & RAG:** ChromaDB or vector DB for KB embeddings, Postgres/NoSQL for user/session metadata.
- **Audit & Compliance:** append-only logs, WORM-like storage for regulatory evidence.
- **Monitoring:** security alerts, anomaly detection, and performance metrics dashboards.

---

## 9. Security, Privacy & Compliance Guidelines
- **Encryption:** AES-256 at rest; TLS 1.2+ in transit.
- **Access control:** role-based access (RBAC) to production data; least privilege for human reviewers.
- **Data minimization:** store minimum required fields; mask or pseudonymize where possible.
- **Consent & transparency:** explicit, logged consent for collection & recording.
- **Retention:** follow local regulator guidance (e.g., 5–10 years for financial services) and offer data deletion if allowed by law.
- **Third-party contracts:** process and subprocessors should have SOC2/ISO27001 or equivalent certifications.
- **Incident response:** defined runbooks for data breach, regulatory notification, and customer communication.

---

## 10. Benefits of AI-Powered KYC
- Faster onboarding with fewer human steps.
- Higher fraud detection via multi-modal checks (ID + face + liveness + video).
- Scalable to large user volumes with predictable costs.
- Clear audit trails for regulators.
- Better UX via step-by-step guidance and automated heuristics.
- Reduces manual paperwork and speeds approvals, which lowers operational cost for providers and can translate to faster service activation for users.
- The KB also highlights scalability, better fraud detection, and improved user experience via step-by-step guidance and auto-cropping tools.

---

## 11. Guidance for Vectorizing this KB into ChromaDB (RAG)
- **Chunking:** split the markdown into logical sections (Intro, Workflow, Document Rules, Textract, Rekognition, Video KYC, Architecture, Security, Each FAQ, Error Codes). Aim for chunks ~200–400 words for good embedding quality.
- **Metadata:** attach `section`, `tags` (e.g., `ocr`, `recon`, `video-kyc`, `faq`, `error-codes`), `version`, and `last_updated` fields to each vector document.
- **Embeddings:** use sentence or text-embedding models tuned for semantic search (OpenAI embeddings or similar).
- **Test queries:** create sample prompts that your RAG agent should answer (e.g., “How to fix ERR-603”, “What formats are allowed for upload?”).
- **Evaluation:** verify answers against canonical KB content; check retrieval precision and recall for common queries.

---

## 12. Frequently Asked Questions (Compiled & Deduplicated Q&A)
> This section compiles all unique questions from the provided sources, with merged/deduplicated answers for consistency and completeness. No repeated questions or duplicate answers are included. Questions are numbered sequentially for ease of reference.

### Q1: What is KYC and why does it matter?
**Answer:** KYC (Know Your Customer) is a process used by organizations to verify the identity of their customers before offering financial or regulated services. It matters because it prevents fraud, money laundering, and helps companies comply with laws. Without KYC, providers risk losses, fines, and reputational damage. It ensures compliance with government and regulatory standards, reduces fraud, enhances security, and provides a seamless customer onboarding experience.

### Q2: What is AI-KYC?
**Answer:** AI-KYC is an artificial intelligence–powered Know Your Customer process that verifies identity and address using digital technologies like document scanning, face recognition, and video verification. It eliminates the need for physical presence, enabling customers to complete secure KYC online quickly and conveniently. It automates the process by using OCR, facial recognition, and video-based verification, making it faster, more accurate, and accessible remotely, anytime and anywhere.

### Q3: How is AI-KYC different from traditional KYC?
**Answer:** Traditional KYC requires physical document submission and in-person verification. AI-KYC automates the process by using OCR, facial recognition, and video-based verification, making it faster, more accurate, and accessible remotely, anytime and anywhere. AI KYC automates extraction and matching, provides auditable logs, and scales to large user volumes while improving speed and fraud detection.

### Q4: Is AI-KYC legally valid in India?
**Answer:** Yes, AI-powered KYC methods such as Aadhaar-based eKYC, PAN verification, and video KYC are recognized and approved by regulatory authorities like RBI, SEBI, and UIDAI, making them fully legally valid. Institutions must follow prescribed guidelines to ensure compliance, making AI-KYC as valid and secure as traditional KYC methods.

### Q5: Who can use AI-KYC?
**Answer:** AI-KYC can be used by banks, fintech companies, NBFCs, mutual funds, telecom operators, and any organization that requires customer identity verification. Customers across India can complete AI-KYC using Aadhaar, PAN, or Passport.

### Q6: How does AI-KYC work?
**Answer:** AI-KYC works by scanning customer documents, extracting data using OCR, verifying authenticity, and matching details with government databases. It then uses AI-powered face recognition and video verification to confirm the person’s identity and prevent impersonation.

### Q7: What are the steps involved in AI-KYC?
**Answer:** Steps include uploading identity documents, scanning for verification, capturing a selfie, completing face match, participating in video KYC (if required), and submitting details. Once verified, the customer’s KYC is approved instantly or after compliance officer review.

### Q8: How long does AI-KYC take?
**Answer:** AI-KYC usually takes 5–10 minutes, depending on internet speed and document clarity. While document upload and face recognition are instant, video KYC may take a few extra minutes. AI verification is often instant. However, if manual review is required (e.g., for unclear photos), it may take a few hours or up to 24 hours.

### Q9: Can I complete AI-KYC from home?
**Answer:** Yes, AI-KYC is fully online and can be completed from home using a smartphone, tablet, or computer with a working camera and internet connection.

### Q10: Can AI-KYC be done anytime?
**Answer:** Yes, AI-driven KYC is available 24/7. However, if manual officer review is needed, approvals may be restricted to business hours.

### Q11: Which documents are accepted for AI-KYC?
**Answer:** Aadhaar Card, PAN Card, and Passport are commonly accepted for AI-KYC. Some institutions may also accept Driving License, Voter ID, or other government-issued identity proofs. We accept **Aadhaar, PAN, and Passport** images only. Alternative IDs (Driving Licence, Voter ID) may be supported depending on specific service configurations — check the service-specific prompt for allowed IDs.

### Q12: Can I upload scanned copies or photos?
**Answer:** Yes, you can upload scanned copies or high-quality photos of your documents. Ensure the image is clear, well-lit, and all details are visible.

### Q13: What formats are supported for document upload?
**Answer:** Only JPEG and PNG image formats are supported for uploading identity documents. Please avoid PDF or other file types, as they are not accepted by the AI-KYC system.

### Q14: Can I upload Aadhaar in PDF format?
**Answer:** No, only JPEG/PNG images are accepted. If you have e-Aadhaar in PDF form, you should take a clear screenshot or convert it into a JPEG/PNG image before uploading.

### Q15: What if my document is unclear?
**Answer:** If the uploaded document is blurry or unreadable, the AI system may reject it. You can re-upload a clearer copy or rescan the document under better lighting. Retake the photo in a brighter location, stabilize your device (use two hands or a stand), enable autofocus, and ensure the camera lens is clean. Confirm the entire document fits inside the frame and text is legible.

### Q16: Why is face recognition required in AI-KYC?
**Answer:** Face recognition ensures that the person submitting documents is the same as the one in the ID proof. It prevents identity theft, impersonation, and fraudulent onboarding. The selfie proves live presence and links the person to their ID photo. It reduces impersonation and prevents using someone else’s ID.

### Q17: What if my face is not recognized?
**Answer:** Ensure you are in a well-lit area, look directly at the camera, and remove masks, glasses, or hats. Retry if needed. First, re-take the selfie in good lighting, remove sunglasses/masks, and ensure your face is centered. If failure persists after several tries, the system will route your case to manual review or trigger a Video KYC session.

### Q18: How does AI face verification work?
**Answer:** AI compares your live face scan with the photo in your documents using biometric algorithms. It also performs liveness checks to confirm you are not using a static image. It uses a face-detection model to crop faces and compute embeddings (numeric representations). Similarity between embeddings determines a match score; threshold-based logic decides pass/fail.

### Q19: What is liveness detection?
**Answer:** Liveness detection is an AI feature that ensures the person is real by asking them to blink, smile, move their head, or read a sentence. It confirms that the selfie is captured from a living person in real-time and not from a photo, replayed video, or deepfake. It is necessary to prevent spoofing attacks and ensure robust identity verification.

### Q20: Is face recognition safe?
**Answer:** Yes, all face data is encrypted and either stored securely for compliance or deleted after verification, depending on the provider’s policy.

### Q21: What is video KYC?
**Answer:** Video KYC is a short live video verification process where AI, sometimes assisted by a human officer, confirms your identity by analyzing your face, voice, and documents in real-time. It involves a live interaction (AI-assisted or human officer) where you show your original ID and answer questions while being recorded. It is used for regulatory compliance or when automated checks are inconclusive — it provides a human-in-the-loop confirmation and recorded evidence for regulators.

### Q22: Do I need to talk during video KYC?
**Answer:** No, you do not need to speak. The AI system verifies your identity through face recognition and liveness detection by checking movements like blinking, nodding, or turning your head.

### Q23: What happens if my internet disconnects during video KYC?
**Answer:** If the internet disconnects, you can restart the process or resume from the last step once you reconnect. The session will terminate. Many providers require an uninterrupted recording, so you'll likely need to restart the video session.

### Q24: Is video KYC available anytime?
**Answer:** AI-driven video KYC is available 24/7. If manual officer review is required, availability may depend on business hours.

### Q25: Can I complete video KYC on my mobile?
**Answer:** Yes, video KYC can be completed on smartphones, tablets, or computers, provided they have a working camera and microphone.

### Q26: How is my data protected in AI-KYC?
**Answer:** AI-KYC uses encryption standards like AES-256 and SSL/TLS to secure your documents, biometric data, and personal information during transmission and storage. Data is protected by encryption in transit (TLS) and at rest (AES-256). Access is restricted via RBAC and logging. Third-party processors are vetted and bound by contracts that require security controls and compliance certifications.

### Q27: Who can access my personal information?
**Answer:** Only authorized compliance teams and regulators, when required by law, have access to your data. Unauthorized third parties cannot access it. Authorized personnel (compliance officers, security teams) and automated systems that need the data to make verification decisions. Access is audited and granted on a least-privilege basis. Regulators may request access during audits under legal processes.

### Q28: Does AI-KYC store my biometric data?
**Answer:** Some providers store biometrics temporarily for compliance, while others delete them after verification. Storage policies depend on regulatory guidelines.

### Q29: Is my Aadhaar number safe in AI-KYC?
**Answer:** Yes, Aadhaar numbers are masked or tokenized as per UIDAI guidelines to ensure data privacy and prevent misuse. For privacy, partial masking (e.g., showing only last 4 digits) is used in UIs and communications to reduce exposure of sensitive identifiers while still allowing you to identify which document was used.

### Q30: Can my data be shared with third parties?
**Answer:** No, your personal data will not be shared with unauthorized third parties. It is used strictly for verification and regulatory compliance.

### Q31: What if I cannot upload my documents?
**Answer:** Check internet connectivity, file format (JPG, PNG, PDF), and size limits. Ensure the document is not password-protected. If the issue persists, try clearing cache, switching devices, or contacting customer support for help.

### Q32: Why is my face not detected?
**Answer:** Make sure your camera is enabled, you are facing directly, and the lighting is sufficient. Avoid hats, masks, or glasses that obstruct visibility. If detection still fails, restart the app or use another device with a better camera. Ensure your face is centered, remove hats/sunglasses, and increase lighting.

### Q33: Can I use AI-KYC on any device?
**Answer:** Yes, AI-KYC works on Android, iOS, and desktops. Your device must have a functional camera and microphone. Ensure your browser or app is updated to the latest version for best results. Practical support depends on the provider’s implementation, but modern Android and iOS devices and modern browsers are expected to work; very old devices may not support liveness or high-quality capture.

### Q34: What if my camera or microphone doesn’t work?
**Answer:** Enable permissions in your device settings, close other apps using the camera, and test hardware with another app. If it still doesn’t work, use a different device or external webcam/microphone.

### Q35: What should I do if I see an error code?
**Answer:** Error codes indicate technical issues like connectivity, format mismatch, or server downtime. Restart the app, recheck your inputs, and retry. If the error persists, share the code with support for faster troubleshooting.

### Q36: Is AI-KYC approved by regulators?
**Answer:** Yes, AI-based eKYC and video KYC are legally recognized in India by RBI, SEBI, IRDAI, and UIDAI. Institutions must follow prescribed guidelines to ensure compliance, making AI-KYC as valid and secure as traditional KYC methods.

### Q37: Is Aadhaar mandatory for AI-KYC?
**Answer:** Aadhaar is widely used due to easy authentication, but it is not mandatory. You can also complete AI-KYC using PAN, Passport, or other accepted identity proofs, depending on the institution’s policy and regulatory requirements.

### Q38: What happens if I don’t complete KYC?
**Answer:** Without completing KYC, you may face restrictions like limited transactions, inability to open new accounts, or blocked access to financial services. Regulators mandate KYC for customer safety and fraud prevention, so completing it is essential. If KYC is not completed you may be unable to open accounts, transact above certain limits, or access regulated services.

### Q39: How often do I need to update my KYC?
**Answer:** KYC updates are generally required only if your personal details (like address, mobile number, or ID document) change. Some financial institutions may request periodic re-KYC, usually every 2–5 years, to comply with updated regulatory norms. KYC itself doesn't “expire” immediately, but regulators may require periodic re-KYC or refreshes (e.g., every 2–5 years). The provider will notify you if re-verification is needed.

### Q40: Is online KYC accepted worldwide?
**Answer:** Online KYC is increasingly accepted globally, though rules vary by country. Many regions, including Europe, the U.S., and Asia, allow digital verification, but document types and processes differ. Always check your provider’s policy based on jurisdiction.

### Q41: What if my KYC fails?
**Answer:** If KYC fails, you’ll be notified with the reason (unclear document, mismatch, or technical error). You can re-upload documents, retry face verification, or contact customer support to resolve the issue and complete KYC.

### Q42: How will I know if my KYC is successful?
**Answer:** Once verified, you’ll receive instant confirmation in the app, along with an email or SMS. Some institutions may also notify you via account dashboards. This ensures you are informed immediately about your KYC status.

### Q43: Can I restart KYC if I made a mistake?
**Answer:** Yes, most platforms allow restarting if wrong documents were uploaded or details mismatched. Simply choose “Redo KYC” in the app or website and repeat the process with correct inputs.

### Q44: How long does approval take?
**Answer:** AI verification is often instant. However, if manual review is required (e.g., for unclear photos), it may take a few hours or up to 24 hours. You’ll be notified once approval is complete.

### Q45: How do I contact support?
**Answer:** You can contact support via in-app chat, email, or toll-free helpline. Many providers also offer WhatsApp or chatbot assistance for faster resolution. Always keep your registered ID handy when contacting support.

### Q46: Can AI detect forged or fake documents?
**Answer:** Yes, AI uses advanced fraud detection, watermark checks, and government database cross-verification to spot tampered or fake documents. It can detect alterations like photoshopped images, mismatched fonts, or missing security features, reducing the risk of fraudulent account openings and identity theft.

### Q47: Can AI-KYC work in rural areas?
**Answer:** Absolutely. AI-KYC requires only basic internet access and a smartphone. Aadhaar authentication and offline document upload options enable users in remote villages to complete KYC without visiting branches, making financial inclusion easier and ensuring access to digital banking and other regulated services.

### Q48: Does AI-KYC support multiple languages?
**Answer:** Yes, many AI-KYC platforms provide support for regional languages. This ensures users can follow instructions, upload documents, and complete verification in their preferred language, reducing errors, improving comfort, and making the process inclusive for people across different states in India. Textract supports multiple languages, but coverage varies. If OCR confidence is low, your case will be flagged for manual review or additional documents.

### Q49: Can AI prevent deepfake fraud?
**Answer:** Yes, AI uses liveness detection, voice matching, and movement tracking to detect deepfakes. It analyzes subtle features like skin texture, eye movements, and real-time audio to confirm authenticity, ensuring fraudsters cannot use pre-recorded videos or manipulated content to bypass verification.

### Q50: Is AI-KYC faster than manual KYC?
**Answer:** Definitely. AI-KYC completes within minutes by automating document scanning, face matching, and live verification. Manual KYC may take days due to human checks, paperwork, and branch visits. AI reduces waiting time, improves accuracy, and enhances customer onboarding efficiency significantly.

### Q51: Why do I need to complete KYC online? Can I still do it offline?
**Answer:** KYC is required by regulated services to prevent fraud and meet regulatory obligations (AML/CFT, FATF, local rules). Completing KYC online speeds onboarding, avoids paper, and uses multi-modal checks (ID + selfie + liveness) to reduce impersonation risk. Offline KYC can still be available for some providers, but many modern flows favor online image + video capture; offline/manual review is used as a fallback or where regulation requires.

### Q52: Why are Aadhaar, PAN and Passport accepted — do I have to provide all three?
**Answer:** The KB specifies Aadhaar, PAN and Passport as the primary supported government IDs; acceptance depends on the service configuration. You usually need to provide one acceptable ID (not all three) as the flow asks you to choose one government-issued ID for verification. Other IDs (Driving Licence, Voter ID) may also be supported depending on the provider’s configuration and regulatory obligations.

### Q53: Is online AI KYC as trustworthy as going to a branch or submitting paper documents?
**Answer:** Yes — when implemented per the KB’s security and audit guidelines, AI KYC produces immutable logs, encrypted evidence, and recorded video sessions that meet regulatory audit requirements. The multi-modal checks (OCR + face match + liveness + video) often provide stronger anti-spoofing than simple paper checks, and all actions are logged for compliance. Trust is contingent on following the KB’s controls (encryption, RBAC, retention, vendor certs) and presenting consent and clear audit trails.

### Q54: What are the benefits of AI KYC for me (speed, convenience, costs)?
**Answer:** AI KYC provides faster onboarding (minutes for automated checks; video KYC 5–10 minutes if needed), fewer trips to branches, and clear in-app guidance to reduce errors. It reduces manual paperwork and speeds approvals, which lowers operational cost for providers and can translate to faster service activation for you. It provides faster onboarding (minutes for automated checks; video KYC 5–10 minutes if needed), fewer trips to branches, and clear in-app guidance to reduce errors.

### Q55: What are the risks of using AI for KYC?
**Answer:** Risks include false rejections (legitimate users flagged), potential bias in models if not audited, and data-security exposures if an operator fails to follow RBAC and encryption practices. There’s also the risk of interrupted sessions due to network or device problems, and dependence on image quality which can cause repeated manual review. The KB advocates safeguards: human review fallbacks, anti-spoofing measures, audits, and strict retention/consent rules to mitigate these risks.

### Q56: Does AI KYC mean a machine is making the final decision about my identity?
**Answer:** Not always. The KB defines final outcomes as aggregated decisions (Approved / Manual Review / Rejected) based on scores from OCR, face match and liveness. Automated thresholds will decide many straightforward cases, but low-confidence situations or suspicious signals are routed to manual review or Video KYC for human oversight. So AI makes the decision when confidence is high; human officers review and override when automation flags uncertainty or compliance needs demand it.

### Q57: If so, is there ever a human reviewing the decision?
**Answer:** Yes — manual review is explicitly part of the workflow. Human compliance officers inspect low-confidence cases, record annotations, and can override decisions with documented reasons. Manual review is used for edge cases such as OCR failures, face mismatch, low liveness confidence or sanctions/watchlist alerts. All human actions are audited and logged to maintain traceability for regulators.

### Q58: How does this app fit into the overall onboarding for the service I’m using (bank account, telecom, etc.)?
**Answer:** The app is the identity verification module of a larger onboarding flow; it provides extracted identity fields, face match evidence and optional video recordings as regulatory proof. Results (approved/pending/rejected) feed back to the service’s onboarding logic so account activation or product access can be granted or blocked accordingly. The KB recommends clear UX that explains what the KYC module does, which ID to use, and how results affect the broader onboarding process.

### Q59: Is the KYC process mandatory for the service I want, or optional?
**Answer:** This depends on the service and regulator: many financial and regulated services require KYC by law, while some low-risk services might offer limited access without full KYC. Regulatory obligations drive mandatory KYC in many cases (e.g., banking, investments), so providers will indicate whether it’s required for the product. If KYC is mandatory, incomplete KYC will limit or block access to certain features.

### Q60: If I don’t complete KYC, what restrictions will I face?
**Answer:** If KYC is not completed you may be unable to open accounts, transact above certain limits, or access regulated services. Providers typically enforce restrictions dictated by compliance policies (e.g., transaction caps, denial of certain features). The KB suggests clear UI messaging to indicate the consequences of not completing KYC so users understand the limitations.

### Q61: What do I need to start (documents, stable internet, camera, ID numbers)?
**Answer:** You need an acceptable government ID image (Aadhaar, PAN, Passport), a working camera for selfie/liveness, and a stable internet connection for real-time processing and video KYC. The KB also recommends device readiness (microphone for video KYC), and the ability to receive OTPs or e-KYC flows if the provider enables UIDAI/Aadhaar verification. A session is created and all actions are tied to it for traceability, so have your session/device ready throughout the flow.

### Q62: Which devices and operating systems are supported (Android, iOS, web browsers)?
**Answer:** The KB implies web and mobile frontends are supported (web/mobile interface + in-app camera). Practical support depends on the provider’s implementation, but modern Android and iOS devices and modern browsers are expected to work; very old devices may not support liveness or high-quality capture. If your device lacks required capabilities, the KB recommends fallback to manual or video KYC on a supported device.

### Q63: Do I need to download a mobile app or can I do KYC through a web link?
**Answer:** Both are possible: the KB describes web/mobile interfaces and chatbot guides for image capture. Some providers deliver KYC via embedded web links (no app install), while others offer native apps for a more controlled capture experience. Check the service prompt to see which channel they provide; the KB supports either flow as long as the capture requirements are met.

### Q64: What permissions will the app ask for (camera, microphone, storage)? Why do you need each?
**Answer:** The app will ask for camera (for document scan, selfie, and video KYC), microphone (for video KYC audio if needed), and storage (to temporarily save images for upload). These are required for real-time capture and verification. Enable permissions in your device settings to proceed.

### Q65: How do I capture a good document image?
**Answer:** Place the document on a flat surface, ensure even lighting, avoid direct glare, keep four corners visible, and take a straight-on photo (not at an angle). Use the in-app camera with auto-crop/edge detection for best results.

### Q66: What fields are extracted from my document?
**Answer:** Typical fields: full name, date of birth, government-issued document number (Aadhaar/PAN/Passport), address (if present), and MRZ components for passports. The system records extraction confidence for each field and flags low-confidence fields for correction.

### Q67: How accurate is OCR (text extraction)?
**Answer:** OCR accuracy depends on image quality, text contrast, and language. On clear, properly-captured images, modern OCR (AWS Textract) achieves high accuracy (often >95% on common fields). However, poor lighting, handwriting, or unusual fonts reduce accuracy and will be flagged for re-capture or manual review.

### Q68: What happens if OCR extracts the wrong value?
**Answer:** If the extracted value differs from the user-entered data or fails validation rules (e.g., PAN format), the system will prompt you to re-upload or manually correct the field. For critical mismatches, the case may be escalated to manual verification.

### Q69: Do you verify my Aadhaar via UIDAI or only read the image?
**Answer:** The system primarily extracts data from the image. Some implementations optionally perform Aadhaar OTP-based verification or UIDAI API checks if the regulator and user consent allow it. Check the flow for an explicit OTP step if Aadhaar verification is configured.

### Q70: What is face verification and how does it work?
**Answer:** Face verification compares a live selfie with the face photo on your ID. It uses a face-detection model to crop faces and compute embeddings (numeric representations). Similarity between embeddings determines a match score; threshold-based logic decides pass/fail. This proves that the person submitting the document is the same person in the ID photo.

### Q71: Why do I need to take a selfie in KYC?
**Answer:** The selfie proves live presence and links the person to their ID photo. It reduces impersonation and prevents using someone else’s ID. Selfies allow face-matching algorithms plus liveness checks to validate identity.

### Q72: What is liveness detection and why is it necessary?
**Answer:** Liveness detection confirms that the selfie is captured from a living person in real-time and not from a photo, replayed video, or deepfake. It is necessary to prevent spoofing attacks and ensure robust identity verification.

### Q73: What should I do if the face match fails?
**Answer:** First, re-take the selfie in good lighting, remove sunglasses/masks, and ensure your face is centered. If failure persists after several tries, the system will route your case to manual review or trigger a Video KYC session.

### Q74: Will wearing glasses or a hat cause face match to fail?
**Answer:** Glasses (especially dark sunglasses), hats, masks, or heavy makeup can interfere with matching. Remove accessories and ensure your face is fully visible. If you must wear religious head coverings, follow the app guidance about temporarily adjusting the covering for verification while maintaining respect and privacy.

### Q75: How does Video KYC differ from automated checks?
**Answer:** Video KYC involves a live interaction (AI-assisted or human officer) where you show your original ID and answer questions while being recorded. It is used for regulatory compliance or when automated checks are inconclusive — it provides a human-in-the-loop confirmation and recorded evidence for regulators.

### Q76: Is Video KYC mandatory for everyone?
**Answer:** Not always. In many jurisdictions (e.g., certain banking use-cases), Video KYC is mandatory or optional depending on risk, product type, and local regulation. If automated checks pass with high confidence, Video KYC may not be required.

### Q77: How long is a typical Video KYC session?
**Answer:** Usually between 5–10 minutes for straightforward cases. Complex or escalated cases may take longer if additional documents or clarifications are needed.

### Q78: Will the video call be recorded and stored?
**Answer:** Yes. For regulatory compliance, the video session and key screenshots are recorded, stored securely (encrypted), and retained according to the provider’s retention policy and local regulations. You are asked for explicit consent prior to recording.

### Q79: How long do you keep my KYC data?
**Answer:** Retention depends on regulator requirements. Financial services commonly retain KYC records for **5–10 years** after relationship termination. Some jurisdictions may require longer retention for specific records. Data deletion requests are handled according to privacy laws and regulatory constraints.

### Q80: Can I request deletion of my KYC data?
**Answer:** You can request deletion subject to legal and regulatory constraints. In many cases, providers must retain KYC records for a statutory period even if you request deletion. Contact support to initiate a data-deletion request and learn what data can be removed.

### Q81: Why was my PAN/Aadhaar number masked in communications?
**Answer:** For privacy, partial masking (e.g., showing only last 4 digits) is used in UIs and communications to reduce exposure of sensitive identifiers while still allowing you to identify which document was used.

### Q82: What are the reasons my KYC could be rejected?
**Answer:** Common reasons include: expired or forged documents, OCR mismatches, face mismatch, liveness failure, low-quality images, missing consent, or sanctions/watchlist hits. Specific rejection reasons are logged and provided where permissible.

### Q83: What is a manual review and when is it triggered?
**Answer:** Manual review is when human compliance officers inspect images, video, and logs to make a decision. It's triggered by low-confidence automation results, suspected fraud, or regulatory requirements. Manual reviewers follow checklists and record decisions for audit.

### Q84: Can I speed up manual review?
**Answer:** Ensure you uploaded clear images and responded fully during any Video KYC prompts. If your case is pending, contacting support with the `session_id` and providing any requested documents can help accelerate the review.

### Q85: How do you handle duplicates (same ID used multiple times)?
**Answer:** The system performs duplicate checks across active accounts using identifiers and biometrics. If a duplicate is detected, the account is flagged and routed to compliance. Depending on policy, duplicates may be blocked or consolidated after human investigation.

### Q86: What if my name changed (e.g., marriage)?
**Answer:** Provide supporting legal documents (marriage certificate, gazette notification) along with updated ID if available. The compliance team will reconcile records and update your profile following verification procedures.

### Q87: Are minors allowed to complete KYC?
**Answer:** Minors may be allowed with guardian consent and additional documents (guardian ID, birth certificate). Specific rules depend on the product and jurisdiction.

### Q88: Can I do KYC for a business entity?
**Answer:** Yes — corporate KYC requires additional documents: certificate of incorporation, board resolutions, proof of authorized signatories, and director IDs. Business KYC workflows are separate and often more complex.

### Q89: Can I use a foreign passport for KYC?
**Answer:** Yes, foreign passports are accepted for certain services. Additional checks (translation, embassy verification) may be required depending on local rules and the provider’s risk appetite.

### Q90: How does the system prevent identity theft?
**Answer:** Multi-factor checks (document OCR + biometric match + liveness + video) make impersonation difficult. Additional signals (device fingerprinting, IP/geolocation checks, transaction patterns) help detect suspicious behavior.

### Q91: Do you screen against sanction lists?
**Answer:** Yes, KYC workflows typically include sanctions and watchlist screening (OFAC, UN, EU, local lists). Matches trigger alerts and compliance workflows for further investigation.

### Q92: What is CKYC and does it apply to me?
**Answer:** CKYC (Central KYC Registry) is an India-specific KYC registry. If your provider participates, a single KYC record may be used across multiple financial institutions. Check with your provider to see if they integrate with CKYC.

### Q93: My document is in a regional language — will OCR work?
**Answer:** Textract supports multiple languages, but coverage varies. If OCR confidence is low, your case will be flagged for manual review or additional documents. Where possible, upload the English-transliterated version or a bilingual copy.

### Q94: Will my KYC status be shared across services?
**Answer:** Only when allowed by law or when providers participate in a shared KYC registry. Otherwise, your KYC status is shared according to privacy policies and contractual agreements.

### Q95: Can I complete KYC on someone else’s behalf?
**Answer:** No — KYC must be completed by the person whose identity is being verified, except in limited cases of guardianship where additional documentation proves authority.

### Q96: What can I do if I suspect identity theft after KYC?
**Answer:** Report to the provider immediately, freeze your account if possible, and follow local identity theft procedures (police report, UIDAI/Income Tax grievance channels if applicable). The provider will initiate a security and forensic investigation.

### Q97: How is liveness data used and stored?
**Answer:** Liveness signals are used only for immediate verification decisions (pass/fail) and as part of an auditable record (screenshots, short clips) retained per policy. Raw biometric templates/embeddings should be protected and stored following applicable laws; some systems store only non-reversible embeddings to reduce privacy risk.

### Q98: What is the expected resolution and size for good images?
**Answer:** Recommended minimum is **800×600 px** and below 5 MB. High-resolution images improve OCR and face crop quality but balance with upload speed and storage costs.

### Q99: How do you handle documents with multiple photos or stamps?
**Answer:** Multi-photo items or stamps that obscure critical text are flagged. You will be asked to upload a clean image or provide alternative documentation. In some cases, manual cleaning or image enhancement is attempted before asking the user to re-upload.

### Q100: What languages are supported?
**Answer:** Primary support for English and major regional languages depending on OCR provider capability (e.g., Hindi). For less-common languages, fallbacks to human review are used.

### Q101: Can I see my KYC session logs?
**Answer:** You can request session summaries (status, timestamps, decision reason). Full logs and recordings are restricted for privacy and regulatory reasons but can be made available to regulators or through formal legal requests.

### Q102: What are the main reasons for repeated verification failures?
**Answer:** Poor image quality, strong facial accessories, bad lighting, inconsistent personal data, or network interruptions. Follow in-app guidance to retake photos and perform checks in a quiet, well-lit place.

### Q103: Does the system support offline capture?
**Answer:** Most flows require online capture for immediate processing and liveness checks. Some implementations allow offline image capture and upload later, but live liveness checks and Video KYC require a live connection.

### Q104: Can I update my KYC after onboarding?
**Answer:** Yes — you can request to update documents or personal details. Updates may require re-verification or supporting documents depending on the change.

### Q105: What logging & evidencing is stored for compliance?
**Answer:** Stored evidence often includes the uploaded images, Textract extractions and confidence scores, face-match scores, liveness challenge evidence, Video KYC recordings, officer annotations, and an immutable audit log with timestamps and decision reasons.

### Q106: What should I do if the KYC process asks for additional documents?
**Answer:** Provide the requested supporting documents promptly (proof of address, birth certificate, etc.). Follow the guidance in the request email or in-app message and ensure images meet the quality requirements. This helps complete manual review faster.

### Q107: What if I don’t complete KYC, what restrictions will I face?
**Answer:** The KB notes that regulatory obligations drive mandatory KYC in many cases (e.g., banking, investments), so providers will indicate whether it’s required for the product. If KYC is mandatory, incomplete KYC will limit or block access to certain features. Providers typically enforce restrictions dictated by compliance policies (e.g., transaction caps, denial of certain features).

### Q108: Why do I need to complete KYC online? Can I still do it offline?
**Answer:** Completing KYC online speeds onboarding, avoids paper, and uses multi-modal checks (ID + selfie + liveness) to reduce impersonation risk. Offline KYC can still be available for some providers, but the KB notes many modern flows favor online image + video capture; offline/manual review is used as a fallback or where regulation requires.

---

## 13. Common Error Codes & Fixes

> Use these codes in your support UI or automated replies to help users resolve issues quickly.

### Document Upload Errors
- **ERR-401: Invalid Document Format**  
  **Cause:** Uploaded file is not JPG/PNG.  
  **Fix:** Convert your document to JPG or PNG and re-upload. PDFs are not accepted.

- **ERR-402: File Too Large**  
  **Cause:** File exceeds allowed size (recommend <5MB).  
  **Fix:** Compress or resize the image using in-app tools or your phone settings and retry.

- **ERR-403: Document Not Clear**  
  **Cause:** Blurry, glare, or cropped edges.  
  **Fix:** Re-take image in good natural lighting, clean lens, hold steady, and ensure all corners are visible.

- **ERR-404: Document Expired**  
  **Cause:** ID is no longer valid.  
  **Fix:** Upload an alternate valid ID or renew the document before retrying.

- **ERR-405: Multiple Documents in Single Upload**  
  **Cause:** Two or more IDs in one photo.  
  **Fix:** Upload only the requested single document image.

### OCR (Text Extraction) Errors
- **ERR-501: OCR Failed**  
  **Cause:** Low-confidence OCR due to quality or script.  
  **Fix:** Provide a clearer image, or choose a document in a supported language.

- **ERR-502: Data Mismatch**  
  **Cause:** Extracted fields don’t match user input.  
  **Fix:** Confirm and correct the entered values or re-upload the document.

- **ERR-503: Unsupported Language**  
  **Cause:** Text in a language/script not supported by OCR.  
  **Fix:** Provide an English or supported language copy, or request manual review.

### Face Recognition & Liveness Errors
- **ERR-601: Face Not Detected**  
  **Cause:** Face out of frame or too dark.  
  **Fix:** Ensure your face is centered, remove hats/sunglasses, and increase lighting.

- **ERR-602: Face Mismatch**  
  **Cause:** Selfie does not match ID photo.  
  **Fix:** Reattempt with better lighting and no obstructions. If still failing, prepare for Video KYC/manual review.

- **ERR-603: Liveliness Failed**  
  **Cause:** Spoof suspected or challenge not performed.  
  **Fix:** Follow on-screen prompts (blink/turn head) and disable face filters.

- **ERR-604: Multiple Faces Detected**  
  **Cause:** More than one person in the frame.  
  **Fix:** Retake selfie alone in a clear background.

### Video KYC Errors
- **ERR-701: Video Session Disconnected**  
  **Cause:** Network interruption.  
  **Fix:** Use a stable connection and retry. Prefer Wi-Fi or 4G/5G with good signal.

- **ERR-702: Poor Audio Quality**  
  **Cause:** Background noise or mic issue.  
  **Fix:** Move to a quiet place, allow microphone access, and speak clearly.

- **ERR-703: ID Document Not Visible**  
  **Cause:** Document not held clearly to the camera.  
  **Fix:** Hold original ID steady, zoom in if needed, and ensure text/photo are readable.

- **ERR-704: Background Tampering Detected**  
  **Cause:** Virtual background or heavy blur flagged.  
  **Fix:** Use a natural, uncluttered background with visible surroundings.

### System & Compliance Errors
- **ERR-801: Session Timeout**  
  **Cause:** Inactivity or long processing time.  
  **Fix:** Restart the KYC flow; keep your session active while completing steps.

- **ERR-802: Duplicate KYC Detected**  
  **Cause:** Same ID used in another account.  
  **Fix:** Contact support with details; the case will be reviewed by compliance.

- **ERR-803: Geolocation Mismatch**  
  **Cause:** Location doesn't match expected region.  
  **Fix:** Enable location services or explain travel status to support.

- **ERR-804: Consent Not Provided**  
  **Cause:** User skipped or declined consent.  
  **Fix:** Provide explicit consent to proceed with KYC.

- **ERR-805: Regulatory Blacklist Match**  
  **Cause:** Name/ID matched watchlist or sanctions list.  
  **Fix:** Compliance team will handle; provide clarifying documents if requested.

---

## 14. Appendices: Logging, Retention, and Troubleshooting tips
- **Logging:** Keep granular logs per `session_id` with timestamps, decisions, confidence scores, and operator annotations. Store logs in append-only storage for compliance.
- **Retention policy:** Follow regulator rules (commonly 5–10 years). Redact or pseudonymize personal data when permissible to reduce exposure.
- **Support tips:** expose helpful error messages and `session_id` to users so they can reference the case when contacting support. Provide visual guides and quick retry options in the UI to reduce support load.

---

**Reminder:** The system accepts image formats only. **Do not upload PDFs.**