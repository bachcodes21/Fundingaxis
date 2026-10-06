# Funding Axis - Phase 1 submission pipeline

What the design assumes happens after an applicant presses **Submit application**. Hand this to your developer with the design file.

## 1. Submit
- Form posts JSON (all typed answers) + uploaded files to a single endpoint.
- Server generates a reference: `FA-####` (shown to the applicant and used everywhere below).
- Validate required fields server-side as well as in the UI.

## 2. Google Drive
- Service account with domain-wide delegation on the Funding Axis Workspace, scoped to `drive.file` on one parent folder (e.g. `Applications`).
- Create child folder named: `<Registered company name> - <reference>`.
  Fall back to trading name, then applicant name, if the registered name is blank.
- Upload every applicant file into that folder, renamed by document type:
  `<reference> - Company Registration Certificate.pdf`, `<reference> - Bank Statements 1.pdf`, etc.

## 3. Generated PDF
- Render the "Generated PDF" layout in the design (server-side HTML → PDF, e.g. Puppeteer/Chromium).
- Filename: `<reference> Application Summary.pdf`.
- Upload into the same Drive folder.

## 4. Email
- Send to `applications@fundingaxis.co.za` (configurable) via a transactional provider (Postmark/SES) - not SMTP from the web host.
- Subject: `New application - <company name> (<reference>)`.
- Body: applicant name, contact details, amount, term, purpose, revenue band, trading period, judgements flag, list of documents received and documents still outstanding.
- Attach the summary PDF; link to the Drive folder rather than attaching client documents.
- Send a confirmation email to the applicant with the same summary PDF.

## 5. Reliability and compliance
- Queue steps 2-4 so a Drive or mail outage does not lose a submission; retry with backoff.
- Store the submission record in your own database first - Drive is storage, not the system of record. This is also the foundation for Phase 2 logins.
- POPIA: explicit consent checkbox is captured with the submission (timestamp + IP). Documents are shared with financiers only after that consent, and the Drive folder should not be publicly linkable.
- Max upload 15MB per file, as on the current form. Accept PDF, JPG, PNG, XLSX, DOCX.

## Open items for you
- Confirm the destination email address and Drive parent folder.
- Confirm whether the applicant should receive a copy of the summary PDF.
- Reversed (white) logo asset is needed for dark surfaces.
