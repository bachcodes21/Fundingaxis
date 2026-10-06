# Funding Axis: handoff to Claude Code

Read this first. Then read, in order: `build-spec.md`, `phase-1-backend-spec.md`, `phase-2-portal-notes.md`, `phase-3-operations-spec.md`, `connector-runbook.md`, `competitive-gap-analysis.md`. `deployment-runbook-netlify.md` is superseded by section 3 below.

## 1. What exists

`Funding Axis Platform.dc.html` is a single-file clickable prototype. Use it as the reference for **screens, copy, rules and data shapes**. It is not deployable code.

| Area | State in prototype |
| --- | --- |
| Marketing site, pre-qualification, application form, client portal, staff console (6 tabs) | Complete as UI |
| Bankability Metric (14 sub-metrics, 50 points, grades A–D, knockouts) | Logic correct, port as-is |
| Matching engine (31 funders, fit score, hard and soft gaps) | Logic correct, criteria unconfirmed |
| Login, roles, funder masking | UI only. Masking runs in the browser; real names are in the page code |
| 14 connectors (Drive, email, bureau, etc.) | UI only. "Test" and "Connect" change a label, nothing more |
| Data | Hard-coded: one case (Kubu Logistics, PO), sample pipeline, sample messages |

### Logic to extract (from the `<script data-dc-script>` block)
Port these into `packages/engine` as pure TypeScript with unit tests. No UI imports.

| Prototype symbol | Becomes |
| --- | --- |
| `BM` | `bankability/metrics.ts` (questions, options, weights, prescriptions) |
| `GRADE_OF`, `KNOCKOUTS`, `bmCompute()` | `bankability/score.ts` returns `{ total, rawGrade, grade, knockouts, answered, provisional }` |
| `GRADE_INFO`, `PROGRAMS` | `bankability/routes.ts` |
| `FUNDER_CRITERIA` | Database table `funder_criteria` (seed from the array; admin-editable) |
| `FIT_COMPAT`, `FAMILY_LABEL`, `COL_LABEL` | `matching/families.ts` |
| `matchRun()` | `matching/run.ts`, takes `(application, score, criteria[])`, returns ranked results |
| `pqVals()` estimate block | `prequal/estimate.ts` |
| `ROLES`, `PERM_MATRIX` | `auth/permissions.ts` |
| `DOC_RULES`, `AUTOMATIONS`, `RETENTION`, `WEIGHTS` | Seed data for admin settings tables |

## 2. Changes made in this round (already in the prototype)

1. **Knockouts.** Judgements, defaults or a default in the last 24 months cap the grade at D. Adverse listings, a restructured facility, or loss-making with no cash buffer or liabilities ahead of assets cap at C. The cap overrides the points total and is shown as the top flag with an explanation.
2. **"Likelihood" renamed to "fit score".** It is a criteria-fit score out of 100, not a probability. The client view now uses live match results; the hard-coded percentages are gone.
3. **Pre-qualification on the 50-point scale.** It estimates the score from age and revenue, assumes 0.6 on every unknown metric, shows a range, and removes funders whose minimum grade is above the estimate. It cannot reach Grade A, by design: A needs verified credit and financials.
4. **Product-agnostic matching.** Each application carries a product family (`fit`). `FIT_COMPAT` decides exact, compatible (soft gap) or incompatible (hard gap). Security rules apply to any family. Rejection reasons are generated from the actual product.

## 3. Target stack

| Layer | Choice |
| --- | --- |
| App | Next.js (App Router), TypeScript, one repo: `apps/web`, `packages/engine`, `packages/db` |
| Hosting | Vercel Pro, functions region `cpt1` (Cape Town) |
| Database | Postgres on AWS RDS `af-south-1`, Drizzle or Prisma, point-in-time recovery on |
| Login | Auth.js, database sessions. Clients: email magic link. Staff: password + TOTP, mandatory |
| Documents | Google Drive Shared Drive via service account (Workspace required). Folder structure and release flow are in `deployment-runbook-netlify.md` section 3; they apply unchanged |
| Uploads | Browser to Drive resumable upload session. Never through a function (Vercel body limit ~4.5 MB) |
| Jobs | Inngest: nudges, expiry sweeps, alert escalation, permission expiry, retention purge |
| Email | Postmark, separate transactional and nudge streams, inbound webhook for replies |
| SMS / WhatsApp | Provider per `connector-runbook.md`; separate consent, STOP/HELP handling |
| Credit bureau | Per `connector-runbook.md`; needs a signed contract before build |
| Monitoring | Sentry, plus an audit log table for every read of client data |

## 4. Data model (minimum)

`company`, `director`, `application` (product family, amount, term, purpose, status), `bm_answer` (metric key, option index, source: self-declared or verified, verified_by, verified_at), `bm_score` (snapshot: total, raw grade, grade, knockouts, provisional), `document` (type, drive_file_id, sha256, valid_until, verified), `funder` (real name, alias, contact; admin-only), `funder_criteria` (versioned), `match_result` (snapshot per run), `submission`, `release` (documents, funder, consent ref, expiry, revoked_at), `offer` (rate, fees, term, computed annualised cost), `message`, `consent` (channel, scope, timestamp, text shown), `staff`, `audit_event`.

Rules:
- Store every score and match as a **snapshot** with the criteria version used. Never recompute history.
- `bm_answer.source` matters: a score built only from self-declared answers is labelled provisional even when complete.
- Reference numbers come from a database sequence (`FA-2026-000142`), not random numbers.

## 5. Security requirements (non-negotiable)

1. **Funder masking on the server.** API responses to broker and analyst roles must never contain a real funder name, contact, or ID that maps to one. Alias = stable hash of funder ID with `FUNDER_ALIAS_SALT`. Test: grep every API response in the test suite for any real funder name.
2. **Role checks in every route handler and server action**, from `auth/permissions.ts`. The UI hiding a button is not a check.
3. **Submission and release are admin-only** and require: complete score, no provisional flag, client consent on file, documents verified and in date.
4. **Clients never get Drive links.** Downloads go through the app and are logged.
5. **Two admins minimum** at launch, both with TOTP.
6. Rate-limit the public pre-qualification and login endpoints.

## 6. Model work still open

| Item | What to build |
| --- | --- |
| Unanswered metrics | Currently score 0. Show "Incomplete" instead of a grade until all 14 are answered; keep the floor visible to staff only |
| Self-declared vs verified | Analyst verification step per metric against documents and bureau. Grade shown to funders only from verified answers |
| Weights | Current weights are judgement. Log every outcome (approved, declined, reason, terms). After about 50 decided applications, refit weights and grade cut-offs against approval data |
| Fit score | Constants (base 52, +22 exact, −11 per soft gap) are judgement. Same calibration once outcomes exist; until then keep the label "fit score" |
| Grant eligibility | Add ownership fields (youth %, black ownership %, women %, B-BBEE level) and eligibility rules for NYDA, NEF, SEFA, IDC, DTIC schemes |
| Industry exclusions | Most `exc` arrays are empty. Fill from funder confirmations |
| Pre-qualification | Add one credit question ("Any judgements or defaults?") so knockouts apply before signup |
| Offer comparison | Annualised cost calculation must be unit-tested against worked examples for each instrument (term loan, MCA factor rate, invoice discount, PO fee) |

## 7. Connectors

`connector-runbook.md` lists all 14 with credentials and setup steps. For each one build: a credentials form (admin), an encrypted secret store (never returned to the browser after save), a **real** test call with the provider's error shown verbatim, a health check job, and an audit entry on change. Build order: Postgres, Auth, Postmark, Google Drive, Inngest, then SMS/WhatsApp, bureau, accounting and bank-feed connectors.

## 8. Testing before launch

- Unit tests for `packages/engine`: every knockout, every grade boundary (20.9, 21, 30.9, 31, 40.9, 41), every `FIT_COMPAT` pair, pre-qual range edges.
- Golden tests: 10 fictional applications across products and grades with expected grade, cap and top-3 matches, agreed with the credit team.
- Role tests: broker and analyst sessions cannot reach a real funder name through any endpoint, export, email template or error message.
- Upload test: 25 MB PDF, interrupted upload resumes, sha256 recorded.
- Release test: release, funder accesses, revoke, access fails.
- Load test the public pre-qual.
- Accessibility pass (keyboard, contrast, form labels) on the client-facing flow.

## 9. Business decisions needed before go-live (owner: you)

1. **Regulatory position.** Get a legal opinion on whether introducing clients to credit providers and receiving commission needs FAIS (FSP) authorisation or NCA registration, and on the wording of the introducer disclosure.
2. **Commission rates to disclose.** The disclosure copy promises to tell clients what you earn. Get the actual rates from each introducer agreement.
3. **Funder criteria confirmation.** For each of the 31 funders: ticket range, minimum trading history, turnover, security, exclusions, turnaround. Until confirmed, label match results "indicative" in the client view.
4. **External advisor channel.** Build or defer.
5. **Bureau contract.** Required before credit history can be verified.
6. **POPIA.** Appoint an Information Officer and register them with the Information Regulator. Sign data processing agreements with Google, AWS, Vercel, Postmark and the SMS provider. Publish storage locations in the privacy notice.
7. **Second admin** named.

## 10. Suggested build order

1. Repo, CI, `packages/engine` ported with tests (week 1)
2. Database, auth, roles, server-side masking (weeks 2–3)
3. Application form and client portal on real data, Drive uploads (weeks 3–5)
4. Staff console: pipeline, client file, matching, messages (weeks 5–7)
5. Admin: connectors, document rules, staff, retention (weeks 7–8)
6. Jobs, email, SMS, alerts (week 8–9)
7. Security review, golden tests, pilot with 5 real clients (week 10)
