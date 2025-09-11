# 📘 AI-Powered KYC System Knowledge Base

> **Note:** This knowledge base is written for ingestion into a RAG system (ChromaDB). **Important operational constraint:** the KYC pipeline accepts **image formats only** for document uploads — **JPG/JPEG** and **PNG**. **PDFs are not accepted.**

---

## Table of Contents
1. Introduction to KYC
2. High-level Objectives & Global Standards
3. AI KYC System Workflow (step-by-step)
4. Document Requirements & Upload Rules (IMAGE ONLY)
5. Text Extraction (AWS Textract) — integration notes
6. Face Recognition & Liveliness (AWS Rekognition) — integration notes
7. Video KYC — process & compliance
8. System Architecture & Components
9. Security, Privacy & Compliance Guidelines
10. Benefits of AI-Powered KYC
11. Guidance for Vectorizing this KB into ChromaDB (RAG)
12. Frequently Asked Questions (50 Q&A)
13. Common Error Codes & Fixes
14. Appendices: Logging, Retention, and Troubleshooting tips

---

## 1. Introduction to KYC
**Know Your Customer (KYC)** is the set of processes and controls organizations use to verify and monitor the identity of their customers. KYC protects businesses and users by preventing identity theft, financial crime, and regulatory breaches. Modern KYC combines regulatory controls with AI-driven automation to make verification faster, more accurate, and auditable.

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

---

## 11. Guidance for Vectorizing this KB into ChromaDB (RAG)
- **Chunking:** split the markdown into logical sections (Intro, Workflow, Document Rules, Textract, Rekognition, Video KYC, Architecture, Security, Each FAQ, Error Codes). Aim for chunks ~200–400 words for good embedding quality.
- **Metadata:** attach `section`, `tags` (e.g., `ocr`, `recon`, `video-kyc`, `faq`, `error-codes`), `version`, and `last_updated` fields to each vector document.
- **Embeddings:** use sentence or text-embedding models tuned for semantic search (OpenAI embeddings or similar).
- **Test queries:** create sample prompts that your RAG agent should answer (e.g., “How to fix ERR-603”, “What formats are allowed for upload?”).
- **Evaluation:** verify answers against canonical KB content; check retrieval precision and recall for common queries.

---

## 12. Frequently Asked Questions (50 Q&A)
> Each question below is unique and has a clear, simple, and sufficiently detailed answer to be shown directly to end users or used by a RAG assistant. No repeated questions or duplicate answers are included.

### Q1: What is KYC and why does it matter?
**Answer:** KYC (Know Your Customer) is a process used by organizations to verify the identity of their customers before offering financial or regulated services. It matters because it prevents fraud, money laundering, and helps companies comply with laws. Without KYC, providers risk losses, fines, and reputational damage.

### Q2: What documents can I use for KYC here?
**Answer:** We accept **Aadhaar, PAN, and Passport** images only. All uploads must be in JPG/JPEG or PNG format. Alternative IDs (Driving Licence, Voter ID) may be supported depending on specific service configurations — check the service-specific prompt for allowed IDs.

### Q3: Why are PDFs not accepted for document upload?
**Answer:** We enforce image-only uploads (JPG/PNG) to simplify preprocessing, ensure single-frame captures, and avoid multi-page ambiguities. Image-only flow improves OCR accuracy and reduces user confusion around which page to upload. If you have a PDF, convert the relevant page to an image and upload it as JPG/PNG.

### Q4: How do I capture a good document image?
**Answer:** Place the document on a flat surface, ensure even lighting, avoid direct glare, keep four corners visible, and take a straight-on photo (not at an angle). Use the in-app camera with auto-crop/edge detection for best results.

### Q5: My document was rejected for being blurry — how can I fix it?
**Answer:** Retake the photo in a brighter location, stabilize your device (use two hands or a stand), enable autofocus, and ensure the camera lens is clean. Confirm the entire document fits inside the frame and text is legible.

### Q6: What fields are extracted from my document?
**Answer:** Typical fields: full name, date of birth, government-issued document number (Aadhaar/PAN/Passport), address (if present), and MRZ components for passports. The system records extraction confidence for each field and flags low-confidence fields for correction.

### Q7: How accurate is OCR (text extraction)?
**Answer:** OCR accuracy depends on image quality, text contrast, and language. On clear, properly-captured images, modern OCR (AWS Textract) achieves high accuracy (often >95% on common fields). However, poor lighting, handwriting, or unusual fonts reduce accuracy and will be flagged for re-capture or manual review.

### Q8: What happens if OCR extracts the wrong value?
**Answer:** If the extracted value differs from the user-entered data or fails validation rules (e.g., PAN format), the system will prompt you to re-upload or manually correct the field. For critical mismatches, the case may be escalated to manual verification.

### Q9: Do you verify my Aadhaar via UIDAI or only read the image?
**Answer:** The system primarily extracts data from the image. Some implementations optionally perform Aadhaar OTP-based verification or UIDAI API checks if the regulator and user consent allow it. Check the flow for an explicit OTP step if Aadhaar verification is configured.

### Q10: What is face verification and how does it work?
**Answer:** Face verification compares a live selfie with the face photo on your ID. It uses a face-detection model to crop faces and compute embeddings (numeric representations). Similarity between embeddings determines a match score; threshold-based logic decides pass/fail. This proves that the person submitting the document is the same person in the ID photo.

### Q11: Why do I need to take a selfie in KYC?
**Answer:** The selfie proves live presence and links the person to their ID photo. It reduces impersonation and prevents using someone else’s ID. Selfies allow face-matching algorithms plus liveness checks to validate identity.

### Q12: What is liveness detection and why is it necessary?
**Answer:** Liveness detection confirms that the selfie is captured from a living person in real-time and not from a photo, replayed video, or deepfake. It is necessary to prevent spoofing attacks and ensure robust identity verification.

### Q13: What should I do if the face match fails?
**Answer:** First, re-take the selfie in good lighting, remove sunglasses/masks, and ensure your face is centered. If failure persists after several tries, the system will route your case to manual review or trigger a Video KYC session.

### Q14: Will wearing glasses or a hat cause face match to fail?
**Answer:** Glasses (especially dark sunglasses), hats, masks, or heavy makeup can interfere with matching. Remove accessories and ensure your face is fully visible. If you must wear religious head coverings, follow the app guidance about temporarily adjusting the covering for verification while maintaining respect and privacy.

### Q15: How does Video KYC differ from automated checks?
**Answer:** Video KYC involves a live interaction (AI-assisted or human officer) where you show your original ID and answer questions while being recorded. It is used for regulatory compliance or when automated checks are inconclusive — it provides a human-in-the-loop confirmation and recorded evidence for regulators.

### Q16: Is Video KYC mandatory for everyone?
**Answer:** Not always. In many jurisdictions (e.g., certain banking use-cases), Video KYC is mandatory or optional depending on risk, product type, and local regulation. If automated checks pass with high confidence, Video KYC may not be required.

### Q17: How long is a typical Video KYC session?
**Answer:** Usually between 5–10 minutes for straightforward cases. Complex or escalated cases may take longer if additional documents or clarifications are needed.

### Q18: Will the video call be recorded and stored?
**Answer:** Yes. For regulatory compliance, the video session and key screenshots are recorded, stored securely (encrypted), and retained according to the provider’s retention policy and local regulations. You are asked for explicit consent prior to recording.

### Q19: What if my internet disconnects during Video KYC?
**Answer:** The session will terminate. Many providers require an uninterrupted recording, so you'll likely need to restart the video session. The system may allow resume in some implementations — check the on-screen guidance.

### Q20: How long do you keep my KYC data?
**Answer:** Retention depends on regulator requirements. Financial services commonly retain KYC records for **5–10 years** after relationship termination. Some jurisdictions may require longer retention for specific records. Data deletion requests are handled according to privacy laws and regulatory constraints.

### Q21: Is my personal data safe on your platform?
**Answer:** Data is protected by encryption in transit (TLS) and at rest (AES-256). Access is restricted via RBAC and logging. Third-party processors are vetted and bound by contracts that require security controls and compliance certifications.

### Q22: Who can access my KYC documents and recordings?
**Answer:** Authorized personnel (compliance officers, security teams) and automated systems that need the data to make verification decisions. Access is audited and granted on a least-privilege basis. Regulators may request access during audits under legal processes.

### Q23: Can I request deletion of my KYC data?
**Answer:** You can request deletion subject to legal and regulatory constraints. In many cases, providers must retain KYC records for a statutory period even if you request deletion. Contact support to initiate a data-deletion request and learn what data can be removed.

### Q24: Why was my PAN/Aadhaar number masked in communications?
**Answer:** For privacy, partial masking (e.g., showing only last 4 digits) is used in UIs and communications to reduce exposure of sensitive identifiers while still allowing you to identify which document was used.

### Q25: What are the reasons my KYC could be rejected?
**Answer:** Common reasons include: expired or forged documents, OCR mismatches, face mismatch, liveness failure, low-quality images, missing consent, or sanctions/watchlist hits. Specific rejection reasons are logged and provided where permissible.

### Q26: What is a manual review and when is it triggered?
**Answer:** Manual review is when human compliance officers inspect images, video, and logs to make a decision. It's triggered by low-confidence automation results, suspected fraud, or regulatory requirements. Manual reviewers follow checklists and record decisions for audit.

### Q27: Can I speed up manual review?
**Answer:** Ensure you uploaded clear images and responded fully during any Video KYC prompts. If your case is pending, contacting support with the `session_id` and providing any requested documents can help accelerate the review.

### Q28: How do you handle duplicates (same ID used multiple times)?
**Answer:** The system performs duplicate checks across active accounts using identifiers and biometrics. If a duplicate is detected, the account is flagged and routed to compliance. Depending on policy, duplicates may be blocked or consolidated after human investigation.

### Q29: What if my name changed (e.g., marriage)?
**Answer:** Provide supporting legal documents (marriage certificate, gazette notification) along with updated ID if available. The compliance team will reconcile records and update your profile following verification procedures.

### Q30: Are minors allowed to complete KYC?
**Answer:** Minors may be allowed with guardian consent and additional documents (guardian ID, birth certificate). Specific rules depend on the product and jurisdiction.

### Q31: Can I do KYC for a business entity?
**Answer:** Yes — corporate KYC requires additional documents: certificate of incorporation, board resolutions, proof of authorized signatories, and director IDs. Business KYC workflows are separate and often more complex.

### Q32: Can I use a foreign passport for KYC?
**Answer:** Yes, foreign passports are accepted for certain services. Additional checks (translation, embassy verification) may be required depending on local rules and the provider’s risk appetite.

### Q33: How does the system prevent identity theft?
**Answer:** Multi-factor checks (document OCR + biometric match + liveness + video) make impersonation difficult. Additional signals (device fingerprinting, IP/geolocation checks, transaction patterns) help detect suspicious behavior.

### Q34: Do you screen against sanction lists?
**Answer:** Yes, KYC workflows typically include sanctions and watchlist screening (OFAC, UN, EU, local lists). Matches trigger alerts and compliance workflows for further investigation.

### Q35: What is CKYC and does it apply to me?
**Answer:** CKYC (Central KYC Registry) is an India-specific KYC registry. If your provider participates, a single KYC record may be used across multiple financial institutions. Check with your provider to see if they integrate with CKYC.

### Q36: My document is in a regional language — will OCR work?
**Answer:** Textract supports multiple languages, but coverage varies. If OCR confidence is low, your case will be flagged for manual review or additional documents. Where possible, upload the English-transliterated version or a bilingual copy.

### Q37: Is there an expiry for KYC completion?
**Answer:** KYC itself doesn't “expire” immediately, but regulators may require periodic re-KYC or refreshes (e.g., every 2–5 years). The provider will notify you if re-verification is needed.

### Q38: Will my KYC status be shared across services?
**Answer:** Only when allowed by law or when providers participate in a shared KYC registry. Otherwise, your KYC status is shared according to privacy policies and contractual agreements.

### Q39: Can I complete KYC on someone else’s behalf?
**Answer:** No — KYC must be completed by the person whose identity is being verified, except in limited cases of guardianship where additional documentation proves authority.

### Q40: What can I do if I suspect identity theft after KYC?
**Answer:** Report to the provider immediately, freeze your account if possible, and follow local identity theft procedures (police report, UIDAI/Income Tax grievance channels if applicable). The provider will initiate a security and forensic investigation.

### Q41: How is liveness data used and stored?
**Answer:** Liveness signals are used only for immediate verification decisions (pass/fail) and as part of an auditable record (screenshots, short clips) retained per policy. Raw biometric templates/embeddings should be protected and stored following applicable laws; some systems store only non-reversible embeddings to reduce privacy risk.

### Q42: What is the expected resolution and size for good images?
**Answer:** Recommended minimum is **800×600 px** and below 5 MB. High-resolution images improve OCR and face crop quality but balance with upload speed and storage costs.

### Q43: How do you handle documents with multiple photos or stamps?
**Answer:** Multi-photo items or stamps that obscure critical text are flagged. You will be asked to upload a clean image or provide alternative documentation. In some cases, manual cleaning or image enhancement is attempted before asking the user to re-upload.

### Q44: What languages are supported?
**Answer:** Primary support for English and major regional languages depending on OCR provider capability (e.g., Hindi). For less-common languages, fallbacks to human review are used.

### Q45: Can I see my KYC session logs?
**Answer:** You can request session summaries (status, timestamps, decision reason). Full logs and recordings are restricted for privacy and regulatory reasons but can be made available to regulators or through formal legal requests.

### Q46: What are the main reasons for repeated verification failures?
**Answer:** Poor image quality, strong facial accessories, bad lighting, inconsistent personal data, or network interruptions. Follow in-app guidance to retake photos and perform checks in a quiet, well-lit place.

### Q47: Does the system support offline capture?
**Answer:** Most flows require online capture for immediate processing and liveness checks. Some implementations allow offline image capture and upload later, but live liveness checks and Video KYC require a live connection.

### Q48: Can I update my KYC after onboarding?
**Answer:** Yes — you can request to update documents or personal details. Updates may require re-verification or supporting documents depending on the change.

### Q49: What logging & evidencing is stored for compliance?
**Answer:** Stored evidence often includes the uploaded images, Textract extractions and confidence scores, face-match scores, liveness challenge evidence, Video KYC recordings, officer annotations, and an immutable audit log with timestamps and decision reasons.

### Q50: What should I do if the KYC process asks for additional documents?
**Answer:** Provide the requested supporting documents promptly (proof of address, birth certificate, etc.). Follow the guidance in the request email or in-app message and ensure images meet the quality requirements. This helps complete manual review faster.

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

## Closing Notes
This markdown is structured for clarity and direct ingestion into a vector DB like ChromaDB. To maximize retrieval quality, split the file into logical chunks (each FAQ as one chunk, plus separate sections for workflow, architecture, and error codes). Attach metadata tags and versioning so your RAG agent can provide accurate, up-to-date answers.

**Reminder:** The system accepts image formats only. **Do not upload PDFs.**

---
