# Funding Axis · full build specification

Everything an engineering team needs to implement the platform, plus an objective assessment of where the inputs are not yet good enough to build from. Written against the three designed phases: public marketing and application (Phase 1), client portal (Phase 2), operations console (Phase 3).

Read section 20 first if you only read one section. It lists the decisions that block coding.

---

## 1. What exists, what is designed, what is assumed

| Layer | State |
| --- | --- |
| Marketing site and application form | Designed, content approved, two hero variants |
| Client portal (sign-in, overview, insights, profile, vault, applications, offers) | Designed |
| Operations console (pipeline, client file, messages, matching, alerts, setup, admin) | Designed |
| Bankability Metric™ | Framework supplied: 14 sub-metrics, 5 categories, 50 points, A–D bands. Implemented in the design |
| Funder list | 31 names and contact emails only. All criteria in the build are our assumptions |
| Funder commercial terms | Not supplied. Introducer agreements, commission rates, exclusivity unknown |
| Advisory products | Route to Bankability (90 days, R3 000), Full Turnaround (120–180 days, R2 000) |
| Backend | Does not exist. Currently forms plus Google Drive folders plus email |
| Brand | Brand guide supplied and applied |

Nothing in this build depends on the Drive-and-email workflow continuing, but section 17 keeps a migration path.

---

## 2. Architecture

Recommended stack, chosen for a two-to-four person team and South African data residency options:

- **Runtime**: TypeScript end to end. Next.js (App Router) for the marketing site, portal and console in one deployment, with server actions and route handlers for the API. One codebase, one auth layer, three surfaces.
- **Database**: PostgreSQL 16, managed (Neon, Supabase, or RDS af-south-1). Prisma or Drizzle for schema and migrations.
- **Jobs and schedules**: a durable queue, not cron-in-the-web-process. BullMQ on Redis, or Inngest if you would rather not run Redis. Expiry sweeps, nudges, Drive mirroring, bureau pulls, webhook retries.
- **Files**: S3-compatible object storage, private, server-side encrypted, versioned. Cloudflare R2 if egress cost matters, S3 af-south-1 if residency does.
- **Email**: Postmark, separate streams for transactional and nudges, inbound webhook for replies.
- **Auth**: Auth.js (NextAuth) with database sessions. Clients: magic link primary, password optional. Staff: password plus TOTP, mandatory.
- **Hosting**: Vercel for the app, managed Postgres, managed Redis. Alternative if residency is a hard requirement: containers on AWS af-south-1 (Cape Town) behind CloudFront. That is roughly two extra weeks of infrastructure work; decide before starting (Decision D-14).

Why not a no-code or CRM-first build: the matching engine, the scoring model with versioning, the release audit trail and the email threading are all bespoke logic with real correctness requirements. A CRM would carry the pipeline and the thread and nothing else, and you would end up integrating against it for every screen in the console. If speed matters more than fit, the honest alternative is HubSpot or Zoho for pipeline and messaging plus a small custom service for scoring and matching — but it splits the system of record, which is exactly what Phase 2 set out to fix.

### 2.1 Environments

| Environment | Purpose | Data |
| --- | --- | --- |
| local | Development | Seeded fixtures, no real client data, mailpit for email |
| preview (per PR) | Review of each change | Fixtures only, email sink, payments in test mode |
| staging | UAT, integration testing, training | Anonymised copy of production, all third parties in sandbox |
| production | Live | Real. Access limited to two named engineers plus the admin |

CI on every PR: typecheck, lint, unit tests, migration dry-run against a scratch database, Playwright smoke suite. Deploy on merge to main, migrations run before the new version takes traffic, one-command rollback. No manual SQL in production without a reviewed migration.

---

## 3. Data model

Field lists below are the minimum. Every table gets `id` (uuid), `created_at`, `updated_at`, and where relevant `created_by_staff_id`.

### company
`registered_name`, `trading_name`, `registration_number` (unique), `date_registered`, `industry` (enum, SIC-aligned), `vat_registered`, `vat_number`, `physical_address`, `postal_address`, `website`, `employees_permanent`, `employees_contract`, `primary_bank`, `bank_account_verified_at`, `annual_revenue_band`, `average_monthly_turnover`, `gross_margin_pct`, `existing_debt_total`, `bbbee_level`, `cipc_return_filed_at`, `judgements_declared`, `judgements_verified`, `owner_user_id`, `source` (web, referral, broker), `status` (prospect, active, dormant, closed).

### company_user
Join between `user` and `company` with `role` (owner, contributor, view_only), `invited_by`, `accepted_at`. One company, several people — required by the Phase 2 notes.

### director
`company_id`, `first_name`, `last_name`, `id_type`, `id_number` (encrypted at rest, never rendered in full), `citizenship`, `shareholding_pct`, `is_signatory`, `surety_offered`, `industry_years`, `email`, `cell`.

### user
`email` (unique, lowercased), `password_hash` (nullable — magic-link-only accounts exist), `name`, `cell`, `last_login_at`, `failed_attempts`, `locked_until`, `popia_consent_at`, `popia_consent_version`, `marketing_opt_in`.

### staff
`user_id`, `role` (analyst, broker, admin), `is_active`, `totp_secret` (encrypted), `totp_confirmed_at`. Roles are a column, not a table, until there is a fourth role.

### application
`company_id`, `reference` (unique, FA-nnnn), `product` (enum), `amount_requested`, `term_requested`, `purpose` (array), `need_by`, `stage` (submitted, under_review, matched, funded, rejected, withdrawn), `stage_changed_at`, `owner_staff_id`, `assist_requested`, `submitted_at`, `closed_at`, `close_reason`, `drive_folder_id`, `score_id` (current), `advisory_product`, `advisory_status`.

`withdrawn` is added to the five designed stages: clients do abandon, and folding that into `rejected` corrupts the decline-rate metric.

### document_type (admin input)
`code`, `name`, `category` (banking, financials, statutory, directors, deal, other), `required_when` (expression or enum), `validity_months` (nullable = no expiry), `window_months` (nullable, e.g. 6 for statements), `reminder_days` (array, e.g. [30,7]), `gates_submission` (boolean), `gates_instrument` (array of fit families), `active`.

### document
`company_id`, `application_id` (nullable — the vault is company-level and reused), `document_type_id`, `storage_key`, `original_filename`, `mime`, `size_bytes`, `sha256`, `uploaded_by_user_id` / `_staff_id`, `uploaded_at`, `period_start`, `period_end` (statement windows), `valid_from`, `valid_until`, `status` (derived: current, expiring, expired, outstanding, rejected), `verified_by_staff_id`, `verified_at`, `rejection_reason`, `virus_scan_status`, `drive_file_id`, `superseded_by_document_id`.

Status is computed, never stored as the truth. Store `valid_until` and derive.

### score, score_answer, score_model_version
- `score_model_version`: `version`, `effective_from`, `categories` and `sub_metrics` as JSON (weights, questions, option factors, prescriptions), `grade_bands`, `created_by_staff_id`. Immutable once used.
- `score`: `company_id`, `application_id`, `model_version_id`, `total`, `grade`, `answered_count`, `is_provisional`, `computed_at`, `scored_by_staff_id`, `data_confidence` (self_declared, part_verified, verified), `released_to_funder_at`.
- `score_answer`: `score_id`, `sub_metric_key`, `option_index`, `factor`, `points`, `source` (client, analyst, bureau, statements), `evidence_document_id`, `note`.

Scores are versioned rows, not a mutable field. The client-facing history graph and any future model calibration both need the series.

### funder, funder_criteria, funder_contact
- `funder`: `name`, `legal_name`, `active`, `instrument`, `fit_family`, `introducer_agreement_signed_at`, `commission_basis`, `commission_rate`, `notes`.
- `funder_criteria`: `funder_id`, `ticket_min`, `ticket_max`, `min_grade`, `min_months_trading`, `min_monthly_turnover`, `security_required`, `industry_exclusions` (array), `decision_days`, `additional_requirements` (JSON, free-form for anything that does not fit), `confirmed_by` (funder contact name), `confirmed_at`, `source` (funder_confirmed, market_assumption), `version`.
- `funder_contact`: `funder_id`, `name`, `email`, `phone`, `role`, `preferred_channel`.

`source` and `confirmed_at` are not optional. The console shows staff whether a match is based on a confirmed rule or a guess, and today every row is a guess.

### match_run, match_result
Persist each run so a submission decision can be justified after the fact. `match_run`: `application_id`, `score_id`, `criteria_snapshot_hash`, `engine_version`, `run_at`, `run_by_staff_id`. `match_result`: `match_run_id`, `funder_id`, `group` (clear, near, out), `likelihood_pct`, `decision_days`, `hard_gaps` (array), `soft_gaps` (array).

### submission, offer
- `submission`: `application_id`, `funder_id`, `submitted_at`, `submitted_by_staff_id`, `channel` (email, portal, api), `pack_document_ids`, `status` (sent, acknowledged, in_review, offer, declined, lapsed), `status_changed_at`, `decline_reason`, `follow_up_at`.
- `offer`: `submission_id`, `amount`, `total_repayable`, `rate_type`, `rate_value`, `term`, `repayment_structure`, `initiation_fee`, `monthly_fee`, `security_required`, `conditions`, `expires_at`, `status` (open, accepted, declined, lapsed), `client_decision_at`, `client_decision_by_user_id`, `apr_effective` (computed).

`apr_effective` matters: the portal currently shows "lowest cost" as a badge. Comparing 2.2% per month over 90 days against prime plus 4.5% over 120 days needs one comparable number, computed by the system, not judged by a broker.

### message, message_delivery
`application_id`, `direction` (out, in, note), `channel` (email, portal, sms, whatsapp), `author_staff_id` or `author_user_id`, `body` (text), `body_html`, `subject`, `provider_message_id`, `in_reply_to`, `thread_key`, `is_internal`, `sent_at`, `attachments`. `message_delivery`: `message_id`, `state` (queued, sent, delivered, opened, bounced, complained, failed), `provider_event_id`, `occurred_at`, `detail`.

Delivery state is a separate table because one message produces several provider events and staff need to see "bounced" rather than assuming the client is ignoring them.

### release_log, consent
- `release_log`: `application_id`, `funder_id`, `document_ids`, `released_at`, `released_by_staff_id`, `consent_id`, `delivery_method`, `recipient_email`.
- `consent`: `company_id`, `type` (popia_processing, release_to_funder, bureau_enquiry, marketing), `scope` (funder ids or global), `granted_at`, `granted_by_user_id`, `evidence` (signature id, IP, user agent, wording version), `revoked_at`.

A checkbox is thin evidence for a bureau enquiry or a document release. Section 13 argues for e-signature on both.

### alert, task
- `alert`: `type`, `severity`, `company_id`, `application_id`, `owner_staff_id`, `title`, `payload` (JSON), `raised_at`, `due_at`, `escalated_at`, `resolved_at`, `resolved_by_staff_id`, `resolution` (enum).
- `task`: the next-step items shown on the client file. `application_id`, `title`, `linked_entity` (type plus id: document slot, sub-metric key, release), `owner_staff_id`, `due_at`, `status`, `blocking` (boolean), `note`.

Alerts are generated and self-resolving. Tasks are created by people. Do not merge them.

### invoice, payment
`company_id`, `application_id`, `product` (route_to_bankability, full_turnaround, other), `amount`, `status` (draft, sent, part_paid, paid, written_off), `due_at`, `provider_reference`, `paid_at`, `method`. Payment status is written by the provider webhook only.

### audit_log
`actor_staff_id` or `actor_user_id` or `system`, `action`, `entity_type`, `entity_id`, `before` (JSON), `after` (JSON), `ip`, `user_agent`, `at`. Append-only, no update or delete grant on the table for the application role. Everything that changes a score, a stage, a criteria row, a permission, or releases a document writes here.

### webhook_event
`provider`, `provider_event_id` (unique), `payload`, `received_at`, `processed_at`, `attempts`, `last_error`. Idempotency by unique provider event id — the single most common source of duplicate emails and double-counted payments.

### Indexes worth naming up front
`application(stage, owner_staff_id)`, `document(company_id, document_type_id, valid_until)`, `message(application_id, sent_at)`, `alert(resolved_at, severity, owner_staff_id)`, `score(application_id, computed_at desc)`, `funder_criteria(min_grade, ticket_min, ticket_max)`.

---

## 4. Storage and file handling

Upload path, client or staff:

1. Client requests an upload slot. Server validates document type, mime and size (25 MB cap, PDF, JPG, PNG, HEIC, XLSX for statements), issues a presigned PUT to a private bucket under `company/{id}/{document_type}/{uuid}`.
2. Client PUTs directly to storage. The app never proxies file bytes.
3. Object-created event enqueues a job: virus scan (ClamAV in a worker, or the provider's scan), compute sha256, extract page count, reject encrypted PDFs with a plain-language message, detect duplicates by hash.
4. Job writes the `document` row, derives status from `validity_months` or the supplied `period_start`/`period_end`, recomputes pack completeness for every open application on that company, resolves any outstanding-document alert, and mirrors to the Drive folder.
5. Download is always a fresh 15-minute signed URL, logged with the actor. Documents are never public and never embedded by URL in email.

Bank statement windows: statements are tracked as ranges, not files, exactly as the Phase 2 notes require. A `statement_coverage` view derives whether the last six months are unbroken from the union of `period_start`–`period_end` on current banking documents. "Six unbroken months" then becomes a computed boolean the matching engine and the submission gate can both read, rather than a broker's judgement.

Drive mirror: one-way, service account, folder per client named `{Registered name} - {reference}`, subfolders by document category, file names stamped with validity. Mirroring failures raise an admin alert and retry with backoff; they never block the client. Nothing is ever read back from Drive into the database.

---

## 5. Integrations, in detail

| Integration | Auth | What it writes | Failure mode and handling |
| --- | --- | --- | --- |
| Postmark outbound | Server token per stream | `message`, `message_delivery` | Queue with retry, 3 attempts, then alert the owner. Bounce marks the client email as suspect and raises an alert rather than silently retrying |
| Postmark inbound | Signed webhook, IP allowlist | `message` (direction in) | Match order: reference in subject → In-Reply-To against `provider_message_id` → sender email against a company user. No match goes to a triage queue with an admin alert after 2 hours. Strip quoted history and signatures before storing the body |
| Google Drive | Service account, domain-wide delegation | `document.drive_file_id` | Retry with backoff. Quota errors back off for an hour. Alert admin on repeated failure |
| Stitch (open banking) | OAuth per client, revocable | Transaction data, derived turnover and coverage | Consent expiry is expected: nudge the client to re-link. Never treat missing feed as a decline |
| Credit bureau | Contracted API key, per-enquiry cost | `score_answer` (credit sub-metrics), `company.judgements_verified` | Requires explicit consent record. Cache 90 days. Log every enquiry with cost for reconciliation |
| CIPC | Reseller API | Verifies `registration_number`, directors, return status | Advisory only. Mismatch raises a review alert, never blocks the client at the form |
| Payments | Provider keys, signed webhook | `invoice`, `payment` | Webhook is the only writer of paid status. Reconcile daily against the provider's settlement report |
| E-signature | API key, webhook | `consent.evidence` | Envelope id and completion certificate stored with the consent row |
| SMS / WhatsApp | Provider key | `message` (channel sms) | Opt-out handling is mandatory; store `marketing_opt_in` and honour STOP |
| Sentry / uptime | DSN | — | Alert on error rate, queue depth over threshold, any webhook handler failing twice in an hour |

**No funder APIs.** None of the 31 funders is assumed to expose an API. Submissions go out as email with a generated credit pack PDF plus signed document links, tracked as a `submission` row, with the reply handled by the same inbound pipeline as client mail. That is a deliberate design choice, not a gap: building per-funder integrations before the criteria are even confirmed would be premature. Revisit for the two or three funders that carry the most volume once there is volume.

---

## 6. Processes and state machines

### 6.1 Application lifecycle

```
draft → submitted → under_review → matched → funded
                              ↘ rejected
                              ↘ withdrawn
```

Transition rules, enforced server-side:
- `submitted` requires the mandatory form sections and POPIA consent. Creates company, application, Drive folder, user account with a set-password link, and an unassigned-owner alert.
- `submitted → under_review` requires an owner. Assignment is manual, or round-robin among active analysts if you want it automatic (Decision D-9).
- `under_review → matched` requires a non-provisional score (all 14 answered) and at least one `clear` or `near` match result. This is the gate that stops guesswork reaching funders.
- `matched → funded` requires an accepted offer and a drawdown confirmation.
- Any stage → `rejected` requires a reason from a fixed list plus a free note. Rejection triggers the advisory offer appropriate to the grade.
- Stage changes are audit-logged with actor and timestamp. Time in stage feeds the pipeline view and the funnel report.

### 6.2 Document lifecycle

`outstanding → uploaded → verified → current → expiring (30 days) → expired`, with `rejected` as a branch from uploaded. Verification is a staff action (analyst or above) and is what turns a client upload into something releasable. Expiry sweeps run daily at 07:00 SAST and drive both client nudges and console alerts.

### 6.3 Scoring

1. Client answers the 14 questions in the portal, or an analyst answers them on the client's behalf during a call.
2. Cross-check: profitability, liquidity, solvency and efficiency against AFS and statements; business age against CIPC; consistency against the statement feed; credit sub-metrics against the bureau. Each `score_answer` records its `source` and evidence document.
3. Where the client's answer and the evidence disagree, the analyst records the evidenced answer and the console shows both. The Phase 2 notes say we call the client — the system should therefore create a task, not silently overwrite.
4. Score is recomputed as a new `score` row on every answer change. Nothing is edited in place.
5. Provisional until 14 of 14 are answered. A provisional score may be shown to the client with the "treat as a floor" wording, and may not be released to a funder.

### 6.4 Matching

Deterministic, re-runnable, and logged. Inputs: current score and grade, application amount and instrument, company months trading, monthly turnover, industry, security on offer, pack completeness. For each active funder with a criteria row:

- **Hard gates** (any one excludes): grade below `min_grade`; amount outside ticket range; months trading below minimum; monthly turnover below 75% of guide; industry in exclusions; security required not on offer (property, asset, card turnover); instrument family incompatible (asset and property desks cannot fund a purchase order).
- **Soft gaps** (reachable): turnover between 75% and 100% of guide; equity or profit share wanted; not the funder's primary instrument for this deal.
- **Group**: no hard and no soft → `clear`. No hard, some soft → `near`. Any hard → `out`.
- **Likelihood**: base 52, plus grade headroom above the funder's minimum capped at +9, plus 22 for an exact instrument match or 6 for an adjacent one, plus up to 12 for turnover headroom, less 11 per soft gap; excluded funders capped in the low twenties so staff can see how far off they are. Clamped 4–94.
- **Timeline**: funder decision days, plus four where soft gaps exist.
- Pack completeness is reported once at file level, not repeated per funder.

The likelihood weights are a considered placeholder. They must be replaced by observed conversion once there are at least 30 submissions per instrument family; until then the number is directional and should not be shown to clients as a precision claim (see 20.3).

### 6.5 Submission and release

1. Broker or admin selects funders from the `clear` group.
2. System checks: score not provisional, all `gates_submission` documents current, client consent to release to those specific funders present and unrevoked, no unpaid advisory invoice blocking (if that rule is on).
3. Credit pack PDF is generated: company profile, financial position, deal summary, Bankability grade with category breakdown, document index. The diagnostic and the prescribed interventions are internal and are excluded from the funder pack (Decision D-6 confirms whether the grade itself goes out at all).
4. Email to the funder contact with the pack and signed document links, `submission` row created, `release_log` written, client notified that their file went out and to whom.
5. Funder replies land in the thread through the inbound pipeline and are classified by staff as acknowledgement, request, offer or decline.

### 6.6 Messaging

Outbound from `applications@fundingaxis.co.za`, subject `[FA-4821] …`, reply-to the same address. Client-visible messages appear in the portal immediately and are emailed; internal notes are never sent. Inbound matching as per section 5. Rate limits: no more than three automated nudges to one client in seven days, and never before 08:00 or after 18:00 SAST. Every automated message respects an unsubscribe for marketing but not for transactional mail about their own application.

### 6.7 Alerts

Generated by the daily sweep and by domain events, owned by the file owner, escalated to admin after 48 hours unresolved. Types and thresholds are the five designed rules. Each alert has exactly one primary action, resolves itself when the underlying condition clears, and records who resolved it and how.

### 6.8 Billing

Advisory product is offered on grade (A direct submission, B and C Route to Bankability, D Full Turnaround). Invoice raised on acceptance, paid through the gateway, webhook sets status. Unpaid past due raises an alert and, if the rule is enabled, blocks new submissions but never blocks the client from uploading documents or seeing their score.

---

## 7. Admin inputs inventory

Every value below is configuration, not code. All are audit-logged, all are admin-only.

| Input | Default | Validation |
| --- | --- | --- |
| Funder list, active flag, contacts | 31 from the supplied list | Unique name, valid email |
| Funder criteria (11 fields per funder) | Current assumptions | Ticket min < max, grade in A–D, decision days 1–60, `source` required |
| Instrument families and their compatibility with each product | As designed | Every product maps to at least one family |
| Document types, validity, windows, reminders, gating | 10 types as designed | Validity months 1–120 or null; reminder days descending |
| Metric weights, questions, option factors, prescriptions | The supplied framework | Category weights sum to 50; factors between 0 and 1 |
| Grade bands and readiness wording | A 41–50, B 31–40, C 21–30, D under 21 | Contiguous, no gaps or overlaps |
| Provisional release rule | Blocked | Boolean |
| Likelihood formula constants | As in 6.4 | Numeric, versioned |
| Alert rules and thresholds | Five rules as designed | Days 1–90 |
| Escalation window | 48 hours | Hours 1–168 |
| Automation on/off switches | As on the Admin tab | Audit entry on change |
| Email templates and sender identities | Four staff templates plus system mails | Merge fields validated against a whitelist |
| Nudge cadence and quiet hours | 3 per 7 days, 08:00–18:00 SAST | — |
| Advisory products and prices | R3 000 / R2 000 | Currency, VAT treatment stated |
| Commission basis per funder | Unknown | Required before any revenue reporting |
| Staff accounts and roles | Three named staff | Email domain restricted, TOTP enforced |
| Retention periods | Section on the Admin tab | Minimums per regulation |
| Consent wording versions | v1 | Immutable once used |

Two admin inputs are missing from the design and should be added when the answers exist: **VAT treatment of advisory fees**, and **commission per funder**, because both change what the finance reports mean.

---

## 8. Permissions, technically

Enforce at the data-access layer, not in the UI. Every query takes the actor; every mutation checks a policy function. The console hiding a button is a convenience, not the control.

**Funder identity is restricted to admin.** Brokers and analysts are the disintermediation risk: a broker who knows which desk funds which deal can take the client there directly and cut Funding Axis out. So funder names, contacts and the criteria table are admin-only, and the matching engine serves everyone else the same results under stable aliases (Funder 01 … Funder 31). Group, likelihood, timeline and the gaps to close are all visible, because that is what a broker needs to do their job; only the identity is withheld. Submission and document release therefore also sit with admin — you cannot send a file to a party you are not allowed to see.

Technically this must be enforced server-side, not by masking in the response renderer: the API returns `funder_alias` and never `funder.name` for a non-admin actor, and `funder_contact` is not queryable at all outside the admin role. A masked field that still ships the real value in the JSON payload is not a control.

Two consequences to accept: an admin becomes a bottleneck on every submission, so a second admin is operationally necessary rather than optional; and the console's search must not match on funder name for non-admin actors, or the alias leaks by probing.

| Resource · action | Client (owner) | Client (view_only) | Analyst | Broker | Admin |
| --- | --- | --- | --- | --- | --- |
| Own company, read | Yes | Yes | — | — | — |
| Own documents, upload | Yes | No | — | — | — |
| Own score, read | Grade and client lens | Grade and client lens | — | — | — |
| Any client file, read | — | — | Yes | Yes | Yes |
| Diagnostic lens | — | — | Yes | Yes | Yes |
| Create or edit score | — | — | Yes | Read only | Yes |
| Verify or reject a document | — | — | Yes | Yes | Yes |
| Request documents | — | — | Yes | Yes | Yes |
| See funder names and contact details | — | — | No | No | Yes |
| See match results with masked funder names | — | — | Yes | Yes | Yes |
| Release documents to a funder | Consent only | No | No | No | Yes |
| Create submission | — | — | No | No | Yes |
| Record offer | — | — | No | No | Yes |
| Accept or decline an offer | Yes | No | No | No | No |
| Fees and commission, read | Own invoices | No | No | Yes | Yes |
| Invoices, raise | — | — | No | Yes | Yes |
| Funder criteria, write | — | — | No | No | Yes |
| Metric model, write | — | — | No | No | Yes |
| Staff and roles | — | — | No | No | Yes |
| Audit log, read | Own release log | Own release log | Own actions | Own actions | All |

Client-side: magic link primary, 15-minute single-use tokens, optional password with 8-character minimum and breach-list check. Staff: password plus TOTP, 12-hour sessions, IP logged, lockout after five failures. No shared logins with file access — a shared finance mailbox gets billing scope only.

---

## 9. Compliance

**POPIA.** Funding Axis is the responsible party; funders are separate responsible parties once documents are released. That makes the release the moment of legal significance, which is why it is gated, logged and consent-scoped per funder rather than blanket. Required artefacts: a privacy notice at the form (present in the design), consent for processing, separate consent for bureau enquiry, separate scoped consent per funder release, an information officer registered with the Regulator, a PAIA manual (the footer already links one — it must exist), a data subject request process with a 30-day clock, and a breach notification runbook. Director ID numbers and bank details are the sensitive fields: encrypt at rest with a KMS key, mask in the UI, exclude from logs and from any analytics export.

**Financial services regulation.** Whether Funding Axis needs an FSP licence or acts as a juristic representative depends on whether it advises on financial products or merely introduces. The site says "we compare offers and recommend"; that language sits close to advice. This is a legal question, not an engineering one, and it should be answered before launch (Decision D-1). It affects record-keeping obligations, the wording of every recommendation in the portal, and whether the likelihood percentage can be shown to clients at all.

**FICA.** Funders do their own onboarding, but if Funding Axis holds identity documents as part of the introduction it should keep them per FICA retention rules. Hence seven years in the retention table.

**Consumer credit.** Marketing says "R0 cost to you, always" while the advisory products cost R2 000–R3 000. Those two statements coexist in the same product and will be read as contradictory. Recommend the marketing line becomes "no cost to apply, no broker fee on funding" with the advisory product priced transparently. This is a copy decision with a compliance edge (Decision D-2).

**Cross-border.** Postmark, Sentry and most gateways process outside South Africa. POPIA section 72 permits this with adequate safeguards; record the transfer basis per processor in a register and include it in the privacy notice.

---

## 10. Security

Threat model, in priority order: credential theft on a staff account (leads to the whole client book), document exfiltration through leaked signed URLs, inbound email spoofing to inject a message into a thread, webhook forgery to mark invoices paid, and injection through uploaded files.

Controls: TOTP mandatory for staff; short-lived signed URLs with the actor logged; SPF, DKIM and DMARC on the sending domain and signature verification plus IP allowlist on inbound; signature verification and event-id idempotency on every webhook; virus scan and content-type verification on every upload with rendering only through a sandboxed viewer; parameterised queries only; secrets in a managed store with rotation; least-privilege database roles with no delete grant on audit tables; dependency and container scanning in CI; an external penetration test before go-live and annually.

---

## 11. Observability and operations

SLOs: portal and console availability 99.5% in business hours; application submission success 99.9%; inbound email appears in the thread within 2 minutes at p95; expiry sweep completes by 07:30 SAST daily.

Dashboards: queue depth and failure rate by job type, webhook success by provider, email delivery and bounce rate by stream, matching engine run time, unresolved alerts by age and owner, submissions awaiting funder response past their decision days.

Runbooks needed on day one: inbound webhook down (clients' replies are being lost), Drive quota exhausted, payment webhook replay, bureau enquiry failing, a wrongly released document (containment, notification, log), and restore-from-backup with a quarterly rehearsal.

---

## 12. Reporting

The business questions the data model must answer without bespoke SQL each time:

- Funnel by month: applications, scored, matched, submitted, offered, funded, with conversion at each step and median days in each stage.
- Funder performance: submissions, offers, funded, decline reasons, median decision days, average rate offered. This is what turns the assumed criteria into confirmed ones.
- Model calibration: predicted likelihood against actual outcome, bucketed. Directly feeds section 6.4.
- Grade distribution of the book, and grade movement for clients who bought advisory work. This is the evidence that Route to Bankability works, and it is also the sales asset.
- Document friction: which document type stalls files longest. Expect bank statements to dominate, which is the business case for the Stitch feed.
- Revenue: advisory sold and collected, commission earned per funder, per broker.

---

## 13. Migration from the current state

1. Export the existing client set from the Drive folders and whatever spreadsheet holds pipeline today. Expect inconsistent naming; reconcile on registration number, not company name.
2. Create `company` and `application` rows, then upload documents into storage with `uploaded_at` preserved and `valid_until` reconstructed where the document states it. Anything undated is marked outstanding rather than guessed at.
3. Invite existing clients with a set-password link and a short explanation. Expect roughly half not to activate; the file works without them.
4. Run the metric against existing files as an analyst-completed score so the pipeline is not empty on day one.
5. Freeze the old process on a stated date. Two systems in parallel for more than two weeks will produce two versions of the truth.

---

## 14. Testing

- **Unit, exhaustively, on the two engines.** Scoring: every factor, unanswered handling, band boundaries at 20/21, 30/31, 40/41, weight sums, model versioning. Matching: each hard gate independently, each soft gap, group assignment, likelihood clamps, ranking stability, and a fixture per funder shape (micro-lender, bank, PO desk, property desk, card-turnover desk).
- **Integration**: inbound email matching by each of the three strategies plus the unmatched path; webhook idempotency by replaying the same event; expiry sweep across a fixture set spanning the boundaries; Drive mirror failure and retry.
- **End to end (Playwright)**: apply → account created → upload → score → match → submit → offer → accept; and the console equivalents per role, asserting that a role without permission gets a 403 from the API, not just a hidden button.
- **UAT scripts** for the three staff roles, run by the actual staff on staging, with the funder criteria confirmation exercise built in as the first real task.
- **Load**: modest. 100 concurrent portal users and 10 000 documents is the realistic first-year ceiling; test to five times that and stop.

---

## 15. Build sequence

Dependency-ordered, not padded. Assumes two engineers, one designer, one product owner from Funding Axis available for decisions.

| Stage | Content | Weeks |
| --- | --- | --- |
| 0 | Decisions in section 20 answered, accounts and contracts opened, environments and CI stood up | 1–2 |
| 1 | Data model, auth, marketing site, application form, submission, account creation, Drive mirror | 3–4 |
| 2 | Document storage pipeline, vault, expiry engine, client portal shell, profile | 3 |
| 3 | Scoring engine, model versioning, insights (client lens and diagnostic lens) | 2–3 |
| 4 | Operations console: pipeline, client file, tasks, alerts | 3 |
| 5 | Messaging: outbound, inbound webhook, threading, templates, internal notes | 2 |
| 6 | Matching engine, funder criteria admin, submission and release with consent gating | 2–3 |
| 7 | Offers, client comparison and acceptance, payments and invoicing | 2 |
| 8 | Reporting, retention jobs, audit log surfacing, penetration test, UAT, migration | 3 |

Roughly 21–25 weeks to a complete build. Stages 1–2 alone give a working intake and vault, which is the fastest route to retiring the current Drive workflow.

---

## 16. Costs, monthly, order of magnitude

Hosting and app 40–90 USD; Postgres 25–70; Redis 10–20; object storage 5–25; Postmark 15–50; Sentry and uptime 30; e-signature 25–50; SMS and WhatsApp usage-based, roughly R0.30–R0.80 per message. Bureau enquiries are the variable to watch: R30–R120 per business enquiry depending on the contract, which at 100 applications a month with both business and consumer pulls is a real line item. Total fixed infrastructure is small — under 300 USD a month at expected volume. The cost that matters is the bureau contract and the engineering time.

---

## 17. Critical analysis of the inputs

This is the objective assessment requested. Each item states what is missing, why it blocks or degrades the build, and what would resolve it.

### 17.1 Blocking — cannot be coded correctly without an answer

**B-1. Funder criteria do not exist.** The supplied list has 31 names and emails. Every ticket range, grade minimum, trading requirement, turnover guide, security requirement, exclusion and turnaround time in the design is our assumption. The matching engine is the centrepiece of Phase 3 and it currently runs on invented data. The engine's structure is sound and testable; its output is not trustworthy. Resolution: a one-page confirmation per funder, six questions, sent to the contacts already on file. Two weeks of someone's time. Nothing else in this list matters as much.

**B-2. Commission and introducer agreements are unknown.** Whether Funding Axis is paid by the funder, and how much, determines the revenue model, whether "no cost to you" is accurate, how brokers should be incentivised, and what the finance reports contain. There is no field to populate and no report to build until this is known.

**B-3. Credit sub-metrics rely on self-declaration.** Credit Score and Repayment History carry 10 of 50 points and are currently answered by the applicant about themselves. A file can be graded B on the strength of a client's optimistic self-assessment, matched to funders on that grade, and declined immediately on the funder's own bureau pull. This is the largest correctness problem in the metric. Resolution: a bureau contract, plus a rule that a score containing unverified credit answers is marked `self_declared` and cannot be released.

**B-4. Nobody has decided whether the grade goes to funders.** The Phase 2 notes leave this open. It changes the credit pack, the release consent wording, and arguably the regulatory position, because a graded recommendation looks more like advice than an introduction.

**B-5. Regulatory position is unresolved.** FSP licence, juristic representative, or pure introducer. Affects portal copy, the likelihood display, record-keeping, and whether recommendations can be automated at all.

### 17.2 Degrading — buildable, but the result is weaker than it looks

**D-1. The likelihood percentage is manufactured.** It behaves sensibly and it is honest about its inputs internally, but shown to a client as "74%" it implies evidence that does not exist. Until there is outcome data, show clients a band (Strong, Possible, Unlikely) and keep the number internal.

**D-2. Self-declared answers throughout.** Beyond credit, ten more sub-metrics are self-reported. The design says we cross-check against the vault and call the client on disagreement. That is the right answer and it is also a staffing commitment: roughly 20–30 minutes of analyst time per file. If that time is not available, the grade is a self-assessment with a serif font.

**D-3. Industry taxonomy is undefined.** Exclusions are matched against `company.industry`, but the form currently collects a free-ish industry string. Without a fixed enum, exclusion matching silently fails. Small fix, real consequence.

**D-4. "Security on offer" is not collected.** The matching engine tests it, but the application form never asks what the client can pledge (property, the asset, a cession, sureties, nothing). It has to be a form field or an analyst input, otherwise every property and asset desk fails on a null.

**D-5. Offer comparability.** Offers arrive in incompatible shapes: percent per month, prime plus, flat fees, settled on invoice. The portal shows a "lowest cost" badge. Without a computed effective cost, that badge is a guess, and it is the single highest-stakes number a client acts on.

**D-6. No funder response SLA.** The console shows a decision timeline derived from assumed decision days. Nothing tracks whether a funder actually replied, so "24–48 hours to feedback" on the marketing site is unenforceable. The `submission.follow_up_at` field and a stale-submission alert fix this, but the promise should be checked against reality before it stays on the homepage.

**D-7. Advisory delivery is undesigned.** Route to Bankability and Full Turnaround are sold in the portal, invoiced by the platform, and then happen entirely outside it. If a 90-day programme has milestones, they belong in the file as tasks, otherwise the client's only evidence of progress is an invoice.

**D-8. Multiple users per company is in the data model but not in any screen.** Directors, bookkeepers and accountants all touch documents. Invitations, role assignment and an activity attribution per user are not designed. This will be requested within a month of launch.

**D-9. Client-side mobile is unproven.** SMME owners will do this on a phone. The designs are responsive but were composed at desktop width, and the vault, offer comparison and 14-question metric are the three screens most likely to break down on a 390px viewport. Budget a mobile pass.

### 17.3 Missing entirely from the current design set

Notifications and preferences for clients; team invitations; staff onboarding and training material; a client-facing document rejection flow (staff can reject, the client is never told why in the design); withdrawal and re-application; a re-scoring history view for the client; funder-side portal (deliberately out of scope, but worth stating); export of a client's own data for a POPIA request; a "no funders match" empty state for a Grade D file, which is a real and common case; and anything at all for a client whose email bounces permanently.

---

## 18. Risk register

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Funder criteria never get confirmed and the engine stays speculative | High | Severe | Treat as a project deliverable with a named owner and a deadline, not a background task. Show `source` on every criteria row so staff can see what is a guess |
| Bureau contract delayed | Medium | High | Ship with the self-declared flag and the release block. Do not quietly release unverified grades |
| Inbound email threading fails silently and client replies are lost | Medium | High | Monitoring on the webhook, triage queue, admin alert after 2 hours |
| Clients do not activate their portal account | High | Medium | Email-only path works end to end through the thread; never make the portal the sole channel |
| Staff continue working in Drive and email | Medium | High | Hard cutover date, one-way mirror only, no read-back |
| POPIA complaint about a document release | Low | Severe | Per-funder scoped consent, e-signature evidence, immutable release log |
| Grade inflation pressure from brokers who need a placement | Medium | Medium | Scoring stays with analysts, model versioned, every change audited |
| Scope drift into a funder-side portal | Medium | Medium | Out of scope in writing until 50 placements a month |

---

## 19. Decisions register

Answer these and the build can start. Numbered so they can be replied to directly.

1. Is Funding Axis an FSP, a juristic representative, or an introducer only?
2. Does the marketing "R0 cost to you, always" stand alongside paid advisory products, or does the wording change?
3. Does the Bankability grade go to funders, or stay internal?
4. Who owns funder criteria confirmation, and by what date?
5. Commission basis and rate per funder — and is it disclosed to clients?
6. Bureau: which provider, business only or business and consumer, and who signs?
7. May a broker re-score a file?
8. May a provisional score ever be released?
9. Owner assignment: manual, round-robin, or by product?
10. Does an unpaid advisory invoice block new submissions?
11. VAT treatment of the R2 000 and R3 000 products.
12. Industry taxonomy: adopt SIC codes, or a short fixed list?
13. Is open banking (Stitch) in the first release or the second?
14. Data residency: is South African hosting a requirement or a preference?
15. Retention: confirm seven years for documents and ten for the audit trail with your legal adviser.
16. Client likelihood display: percentage, or band?
17. Who is the registered POPIA information officer?
18. Cutover date for retiring the Drive-and-email process.
