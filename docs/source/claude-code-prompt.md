# Prompt for Claude Code

Paste everything below the line into Claude Code as your first message, from inside the project folder.

---

You are building **Funding Axis**, a South African SME funding platform, from a working prototype and a set of specs in this folder. You are the lead engineer. Work carefully, in small committed steps, and stop for my review at the checkpoints marked **STOP**.

## Context: read before writing any code

Read these files in this order and do not skip any:

1. `claude-code-handoff.md`: the master brief. It overrides everything else where they conflict.
2. `build-spec.md`
3. `phase-1-backend-spec.md`
4. `phase-2-portal-notes.md`
5. `phase-3-operations-spec.md`
6. `connector-runbook.md`
7. `competitive-gap-analysis.md`
8. `Funding Axis Platform.dc.html`: the clickable prototype. The UI is the visual and copy reference. The logic class inside `<script data-dc-script>` is the reference for every business rule.

Ignore `deployment-runbook-netlify.md` for hosting; we are on Vercel. Its Google Drive section (section 3) still applies. Ignore `support.js`; it only runs the prototype.

After reading, reply with:
- a one-paragraph summary of the product in your own words,
- a list of every business rule you found in the prototype logic (Bankability metrics and weights, grade cut-offs, knockouts, fit-score formula, `FIT_COMPAT`, pre-qual estimate, role permissions, document rules, automations, retention),
- every contradiction or gap between the specs and the prototype, with your proposed resolution,
- any questions that would block step 1.

**STOP** and wait for my answers.

## Product rules you must preserve

- **Bankability Metric:** 14 sub-metrics across 5 categories, 50 points, grades A (41+), B (31+), C (21+), D (below 21). Knockouts cap the grade regardless of total (see `KNOCKOUTS`). Grade D is a referral to advisory, never a decline.
- **Fit score, not likelihood:** the matching output is a criteria-fit score out of 100. Never label it a probability or "chance of approval" anywhere: UI, emails, API field names or docs.
- **Funder identity is admin-only.** Brokers and analysts see stable aliases (Funder 01 … 31). Masking happens on the server. No real funder name, contact, ID, logo or email address may reach a non-admin session through any API response, export, email template, log line sent to the client, or error message.
- **Submission and release are admin-only**, and blocked unless the score is complete, consent is on file, and required documents are verified and in date.
- **Provisional scores block submission.** A score built on self-declared answers is labelled provisional until an analyst verifies it against documents or the bureau.
- **Fee disclosure copy is verbatim from the prototype.** Do not rewrite legal or disclosure text.
- **Grants** (SEFA, IDC, NEF, NYDA, DTIC) are a fourth instrument family.
- **SMS/WhatsApp consent is separate from email consent**, with STOP/HELP handling.
- **Offer comparison** computes true annualised cost per instrument; never hard-code a "best offer" badge.

## Stack (fixed; ask before deviating)

- Monorepo with pnpm workspaces: `apps/web` (Next.js App Router, TypeScript strict), `packages/engine` (pure TS business logic, no framework imports), `packages/db` (schema, migrations, seed).
- Postgres via Drizzle ORM. Local: Docker Compose. Production: AWS RDS `af-south-1`.
- Auth.js with database sessions. Clients: email magic link. Staff: email + password + TOTP, mandatory.
- Vercel, functions region `cpt1`.
- Google Drive (Shared Drive, service account) for documents.
- Inngest for background jobs.
- Postmark for email.
- Zod for every input boundary.
- Vitest for unit tests, Playwright for end-to-end.
- Tailwind for styling. Port the prototype's palette and type exactly: Playfair Display for headings, Poppins for body, colours taken from the prototype's inline styles. Extract them into a Tailwind theme rather than retyping hex values.

## Ground rules

- **TypeScript strict, no `any`.** Every server action and route handler validates input with Zod and checks permissions with a single `requirePermission(session, action)` helper from `packages/engine/auth/permissions.ts`.
- **All business logic in `packages/engine`**, pure functions, 100% branch coverage on scoring, knockouts, matching and pre-qual.
- **Snapshots:** every score and match run is stored with the criteria version used. Never recompute historical results.
- **Secrets** live only in `.env.local` (git-ignored) and Vercel environment variables. Create `.env.example` with every key and no values. Never print a secret, commit one, or return one to the browser after it is saved.
- **Audit log:** every read or write of client data, every release, every role change and every connector change writes an `audit_event` row.
- **No mock success.** If a connector is not configured, the UI says so. Never fake a "Connected" or "Test passed" state.
- **POPIA:** collect only fields the specs require; implement the retention rules from the prototype's `RETENTION` as scheduled jobs.
- Commit after each completed sub-task with a clear message. Run lint, typecheck and tests before every commit.
- When unsure about a business rule, ask. Do not invent one.

## Build plan

Do these in order. Finish each step's acceptance checks before moving on.

### Step 1: Foundation and engine
- Scaffold the monorepo, CI (GitHub Actions: lint, typecheck, test), Docker Compose for Postgres, `.env.example`.
- Port into `packages/engine`:
  - `bankability/metrics.ts` from `BM`
  - `bankability/score.ts` from `bmCompute` + `GRADE_OF` + `KNOCKOUTS`, returning `{ total, rawGrade, grade, knockouts, answered, count, provisional, rows }`
  - `bankability/routes.ts` from `GRADE_INFO` + `PROGRAMS`
  - `matching/families.ts` from `FIT_COMPAT`, `FAMILY_LABEL`, `COL_LABEL`
  - `matching/run.ts` from `matchRun`, taking `(application, score, criteria[])` instead of the hard-coded `CASE`
  - `prequal/estimate.ts` from the pre-qual block in `pqVals`
  - `auth/permissions.ts` from `ROLES` + `PERM_MATRIX`
  - `offers/annualised-cost.ts` for term loan, MCA factor rate, invoice discount and PO fee structures
- Tests: every grade boundary (20.9, 21, 30.9, 31, 40.9, 41), every knockout alone and combined, every `FIT_COMPAT` pair, pre-qual range edges, every role × action, annualised cost against hand-worked examples (write the examples in the test file with the working shown).
- Golden tests: create 10 fictional applications spanning all product families and grades A–D, with expected grade, cap and top 3 matches. Put them in `packages/engine/__fixtures__/golden.ts`.

**STOP.** Show me the test output and the golden fixtures table so I can check the expected results with the credit team.

### Step 2: Database, auth and masking
- Schema per section 4 of the handoff: `company`, `director`, `application`, `bm_answer` (with `source` and verification fields), `bm_score`, `document`, `funder`, `funder_criteria` (versioned), `match_result`, `submission`, `release`, `offer`, `message`, `consent`, `staff`, `audit_event`.
- Reference numbers from a Postgres sequence: `FA-YYYY-000001`.
- Seed `funder_criteria` from `FUNDER_CRITERIA` with `confirmed = false` on every row.
- Auth.js: magic link for clients, password + TOTP for staff, roles `admin | analyst | broker`.
- Server-side masking: a single `presentFunder(funder, session)` function used by every response that touches a funder. Alias = deterministic from funder ID + `FUNDER_ALIAS_SALT`, zero-padded, stable forever.
- Test: a Playwright suite logs in as broker and as analyst, crawls every page and API route, and fails if any response body contains any real funder name from the seed.

**STOP.** Show me the masking test passing.

### Step 3: Public site and pre-qualification
- Port the marketing page from the prototype (testimonials removed; International Trade Finance tagged "2 weeks").
- Pre-qualification: five questions, no account, runs `prequal/estimate.ts` on the server, shows indicative grade, points range, routes and fastest decision times. Rate-limit by IP.
- Add one question: "Are there any judgements or defaults against the business or its directors in the last 24 months?" so knockouts apply before signup.
- "Book a call" scheduler as in the prototype, writing to the database and emailing staff.

### Step 4: Application form and client portal
- Five-step application as in the prototype, saved per step, resumable.
- Bankability questions captured as `bm_answer` with `source = 'self_declared'`.
- Document upload: browser → Google Drive resumable upload session created by the server; completion handler verifies size and sha256, writes `document`, applies `DOC_RULES` validity.
- Client portal: score, gaps to close, best-fit financiers (fit score, labelled "indicative" while criteria are unconfirmed), document vault, offers with annualised cost, messages.
- Show "Incomplete" instead of a grade to clients until all 14 metrics are answered.

### Step 5: Staff console
- Six tabs as in the prototype: Pipeline, Client file, Messages, Matching, Alerts, Admin.
- Analyst verification: per metric, mark verified against a document or bureau result; the score is provisional until every metric is verified.
- Matching runs on demand and on every score change; results stored as snapshots.
- Submission and release (admin only): select funder and documents, check every precondition, copy files into `_Releases/{date} {funder}/` in Drive, share with the funder's email with an expiry, write `release` and `audit_event`. Revoke removes the permission.

**STOP.** Demo the full flow end to end with a seeded client.

### Step 6: Connectors
Build in this order: Postmark, Google Drive, Inngest, credit bureau, then the rest from `connector-runbook.md`.

For every connector:
- an admin settings form whose fields come from `CONN_SETUP` in the prototype,
- encrypted storage (AES-256-GCM, key from env); after saving, show only the last 4 characters,
- a **real** test call that shows the provider's actual response or error,
- a health-check job every 15 minutes that raises an Alert on failure,
- an audit entry on every change.

**Google Drive specifics:** a Shared Drive and a service account with Content manager access. Folder per company: `{Registered name} - {FA reference}/01 Company, 02 Financials, 03 Bank statements, 04 Directors, 05 Offers and contracts, _Releases`. Store Drive IDs, never look up by name. Staff downloads above 5 MB use temporary reader permission expiring after 1 hour; clients never receive Drive links.

**Credit bureau specifics:**
- Write a provider interface `BureauProvider { softEnquiry(subject, consent), fullEnquiry(subject, consent), commercialReport(company, consent) }` with a sandbox implementation first. I will supply the real provider (TransUnion, Experian or XDS) and test credentials later.
- **Soft enquiry** runs after pre-qual or signup, only with explicit recorded consent (consent text, timestamp, IP, subject ID). It pre-fills `creditScore` and `repayment` as `source = 'bureau_soft'`.
- **Full enquiry** runs at verification, before submission, with a second explicit consent. It sets `source = 'bureau_full'`, marks those metrics verified, and re-runs knockouts.
- Store the raw bureau response encrypted, with a retention period from `RETENTION`. Show staff a summary only.
- Never run an enquiry without a matching consent row. Write a test that proves it.
- Log cost per call so I can see bureau spend per application.

### Step 7: Jobs, messaging and alerts
- Inngest functions: document expiry sweep (daily 02:00 SAST), client nudges (daily 09:00), alert escalation (every 15 minutes), Drive permission expiry (hourly), retention purge (weekly), connector health (every 15 minutes).
- Messages: email via Postmark with inbound replies threaded; SMS/WhatsApp behind separate consent with STOP/HELP.
- Templates from `MSG_TEMPLATES`. All funder references in templates pass through `presentFunder`.

### Step 8: Hardening
- Security headers, CSP, rate limits on auth and pre-qual, CSRF on mutations.
- Accessibility pass on all client-facing pages (keyboard, labels, 4.5:1 contrast).
- Playwright suites: full client journey, full staff journey, masking crawl, upload (25 MB, interrupted, resumed), release and revoke, consent-gated bureau calls.
- `README.md` with setup, env vars, deployment to Vercel, and a runbook for rotating every secret.

**STOP.** Hand over with a list of anything not done, every assumption you made, and every place a business decision is still needed.

## Things I still owe you (do not invent these)
- The credit bureau provider and its sandbox credentials
- Confirmed criteria for each of the 31 funders
- The actual commission rates for the disclosure page
- Legal confirmation of the regulatory wording
- The names and emails of the two admins

Where these are needed, build the mechanism, use clearly marked placeholders, and add the item to a `TODO-BUSINESS.md` file.

Begin by reading the files and replying as described under "Context".
