# Step 0: reading review

My reply to the "Context" section of `claude-code-prompt.md`. Sources are in `docs/source/`. Line numbers refer to `Funding Axis Platform.dc.html`.

---

## 1. The product in one paragraph

Funding Axis is a South African credit introducer for SMEs. It does not lend. An owner gets a quick, no-account estimate (pre-qualification), then fills in a five-step application and keeps a reusable document vault. They answer 14 questions that produce the **Bankability Metric**: a score out of 50, graded A to D. Knockouts cap the grade when there is adverse credit or the business is distressed. Staff check those answers against documents and the credit bureau. A deterministic engine compares the file with about 36 funders' criteria and gives each funder a **fit score** (not a probability), sorted into clear, near and out, with plain-language gaps. Analysts and brokers see funders only under aliases. Only an admin can see real identities, submit a file or release documents, and only once the score is complete and verified, consent is on file and the gating documents are in date. Offers are compared on a computed annualised cost. Files that are not yet fundable are referred to paid advisory work: Route to Bankability at R3 000 for B and C, Full Turnaround at R2 000 for D. Grants and development finance are the route for D. Revenue is funder commission plus the advisory fees, and both are disclosed. POPIA consent, an audit trail and retention are built into the core, not added later.

---

## 2. Business rules found in the prototype logic

### 2.1 Bankability Metric (`BM`, l.2301)
Score per metric = `max × factor` of the chosen option. **An unanswered metric scores 0.** Total = sum of all 14 (max 50).

| Category | Metric (key) | Max | Option factors (in order) |
| --- | --- | --- | --- |
| Financial Health (10) | profitability | 2.5 | >15% margin 1 · ≤15% 0.75 · broke even 0.4 · loss 0.1 |
| | liquidity | 2.5 | ≥3 months 1 · 1–2 months 0.6 · <1 month 0.3 · nothing 0.1 |
| | solvency | 2.5 | assets well ahead 1 · balanced 0.6 · liabilities ahead 0.2 · not sure 0.1 |
| | efficiency | 2.5 | <30d 1 · 30–60d 0.7 · 60–90d 0.4 · >90d/unsure 0.15 |
| Business Stability (10) | age | 3.33 | ≥5y 1 · 2–5y 0.8 · 12–24m 0.5 · <12m 0.15 |
| | consistency | 3.33 | growing+steady 1 · seasonal 0.75 · uneven 0.4 · declining 0.15 |
| | diversification | 3.34 | <20% 1 · 20–40% 0.7 · 40–70% 0.4 · >70% 0.15 |
| Growth Potential (10) | market | 3.33 | growing 1 · stable 0.7 · crowded 0.4 · shrinking 0.15 |
| | sales | 3.33 | CRM pipeline 1 · referrals 0.65 · ad hoc 0.35 · none 0.1 |
| | innovation | 3.34 | launched new 1 · adapted 0.65 · little change 0.35 · struggled 0.1 |
| Management Quality (10) | experience | 5 | >10y 1 · 5–10y 0.75 · 2–5y 0.5 · <2y 0.2 |
| | strategy | 5 | written plan 1 · goals 0.6 · month to month 0.3 · none 0.1 |
| Credit History (10) | creditScore | 5 | strong 1 · minor settled 0.65 · adverse listings 0.3 · judgements/defaults 0.1 |
| | repayment | 5 | never missed 1 · 1–2 late 0.7 · restructured 0.4 · defaulted <24m 0.1 |

Each metric also carries a prescription string, used in the diagnostic and "gaps to close".

### 2.2 Grades, knockouts, derived outputs (l.2567–2576, `bmCompute` l.2956, `bmVals` l.2979)
- `GRADE_OF`: A ≥ 41, B ≥ 31, C ≥ 21, otherwise D. Applied to the **unrounded** float total.
- `KNOCKOUTS`: creditScore option 3 caps at D. creditScore option 2 caps at C. repayment option 3 caps at D. repayment option 2 caps at C. profitability 3 **and** liquidity 3 caps at C. profitability 3 **and** solvency 2 caps at C. The final grade is the lower of the raw grade and the lowest cap hit. A cap shows as the top flag with "This overrides the points total…".
- Category colour bands: ratio ≥ 0.8 is green, ≥ 0.55 amber, otherwise red. A category under 0.7 raises a flag naming its weakest metric.
- Gaps to close: the three metrics with the most points lost, where the loss is over 0.2 points.
- "Next grade" copy: the points needed to reach 21, 31 or 41.
- Lead quality and route wording per grade (l.3056).
- Provisional means `answered < 14` (prototype). Release and submit are disabled while provisional (l.3336–3344).

### 2.3 Routes and programmes (`GRADE_INFO`, `PROGRAMS`, l.2356)
A: Direct submission, "No cost to you", this week. B: Route to Bankability, 90 days, R3 000. C: Route to Bankability, 90–120 days, R3 000. D: Full Turnaround, 120–180 days, R2 000. Band names and actions per grade as written.

### 2.4 Matching (`matchRun`, l.3181)
Each funder in `FUNDER_CRITERIA` (36 rows, l.2522) is tested on the following.

**Hard gaps:**
- grade below the funder's minimum
- amount below the ticket minimum, or above the maximum
- months trading below the minimum
- turnover below 75% of the funder's turnover guide (only when the guide is above 0)
- industry in the funder's exclusions
- funder needs card turnover and the client has no card book
- funder needs property, asset, surety or cession security, the client has not offered it, **and the funder's family differs from the application's**
- product family incompatible, per `FIT_COMPAT`

**Soft gaps:**
- turnover between 75% and 100% of the guide
- funder wants equity
- compatible but not an exact family match

**Group:** any hard gap puts the funder in "out". Otherwise any soft gap makes it "near". Otherwise "clear".

**Fit score:**
```
52 + min(9, (rank(grade) − rank(minGrade)) × 6)
   + (exact ? 22 : 6)
   − 11 × softCount
   + (tv > 0 ? clamp(round((turnover/tv − 1) × 6), −6, 12) : 6)
if hardCount > 0: score = 24 − 5 × hardCount
clamp to 4..94
```
- Results are sorted by fit score, descending.
- Timeline = decision days, plus 4 when there are soft gaps.
- Wording: "inside N days" when N ≤ 3, otherwise "roughly N days".
- Bar colours: green at ≥ 65, amber at ≥ 40.

### 2.5 Families (`FIT_COMPAT`, `FAMILY_LABEL`, `COL_LABEL`, l.2561–2613)
12 families: po, trade, invoice, wc, mca, asset, property, bridge, growth, term, equity, grant. The compatibility map is directional, not symmetric: wc lists mca, but mca lists only wc; asset lists term, but term does not list asset. `grant: []`, and no other family lists grant. 7 security codes: none, cession, asset, property, surety, equity, card.

### 2.6 Pre-qualification (`pqVals`, l.3110)
Questions: amount (slider), **monthly** turnover band, business age band, need, security.
- Turnover midpoints (`REV_MID`): 25k / 100k / 300k / 1m / 3m. Age in months (`AGE_MO`): 3 / 9 / 18 / 42 / 72.
- ageR = 1 at ≥ 60 months, 0.8 at ≥ 24, 0.5 at ≥ 12, else 0.15.
- finR = 0.8 at ≥ R1m a month, 0.65 at ≥ 300k, 0.5 at ≥ 100k, else 0.35.
- `est(u) = 3.33·ageR + 10·finR + 36.67·u`. Midpoint u = 0.6, range u = 0.4 to 0.8. Band = `GRADE_OF(mid)`.
- Eligibility:
  - band at or above the funder's minimum
  - need maps to the funder's family through `PQ_NEED_FAMILY`, exact family only
  - grant funders are included only when age is under 12 months or turnover under 100k
  - amount within the ticket
  - months at or above the funder's minimum
  - turnover at or above 75% of the guide
  - property and asset security required strictly
- Output: up to 4 families ranked by desk count, each with the fastest decision days, the band copy, the range and a fixed caveat.

### 2.7 Roles (`ROLES`, `PERM_MATRIX`, l.2407)
Permission keys: score, docsRequest, docsRelease, submit, fees, billing, criteria, staff, funders.
- Analyst: score, docsRequest.
- Broker: docsRequest, **docsRelease**, fees, billing.
- Admin: everything.

The matrix text (11 rows) and the release, submit and mask checks are at l.3336–3341. Payment alerts are hidden from roles without billing.

### 2.8 Masking (l.3234–3242)
- Alias = `Funder NN` by **position in the criteria list**.
- Names are replaced by string substitution inside free text: the timeline, next steps and release log.

### 2.9 Document rules (`DOC_RULES`, l.2719)
10 types, each with prose rules for when it is required, how long it is valid, reminders and gating:
- bank statements: rolling 6 months, blocks submission
- AFS: 18 months from year end, blocks bank desks
- management accounts: 3 months, soft gap
- CoR14.3: no expiry, blocks submission
- SARS pin: 12 months, blocks submission
- B-BBEE: 12 months, soft gap
- director ID: no expiry, blocks submission
- proof of residence: 3 months, soft gap
- deal documents: valid to the delivery date, blocks that instrument
- proof of bank account: 3 months, blocks drawdown

### 2.10 Automations (`AUTOMATIONS`, l.2740)
10 triggers. All are on except "Advisory invoice unpaid → hold new submissions", which is off.

### 2.11 Retention (`RETENTION`, l.2767)
- documents: 7 years after the file closes, with a legal hold
- scores: indefinite, pseudonymised after 7 years
- threads: 7 years
- release and audit log: 10 years, immutable
- rejected files that never progressed: 24 months
- bureau reports: 12 months

### 2.12 Other
- Five-step form with required fields per step (`stepDone`, l.2927).
- SMS/WhatsApp consent copy with STOP/HELP (l.640).
- Fee disclosure copy, two versions (l.319 short, l.1378 long).
- Book-a-call with six fixed slots (l.3698).
- Testimonial slots deliberately left empty (l.3710).
- Connector setup fields (`CONN_SETUP`, 14).
- Message templates (`MSG_TEMPLATES`, 4).

---

## 3. Contradictions and gaps, with proposed resolutions

Where the master brief settles a point, I follow it and say so. Items marked **Q** need you; they are repeated in section 4.

### Engine
1. **Grade boundary vs displayed number.** The grade is taken from the raw float, but the UI shows one decimal place. A total of 40.96 displays "41.0" and grades B. Proposal: round the total to 1 dp first, then grade on that. **Q1.**
2. **Security exemption.** The handoff says "security rules apply to any family". The code skips the security hard gap whenever the funder's family equals the application's, so a PO desk that needs a cession passes even when no cession is offered. **Q2.**
3. **Grants are unreachable in full matching.** `FIT_COMPAT.grant = []` and no product maps to grant, so all 5 grant funders are always "out". The "D → grant referral" only works in pre-qual. Proposal: evaluate grant funders on a separate eligibility track that ignores product family, using the ownership fields from handoff §6, which are not yet captured. **Q3.**
4. **No product-to-family mapping.** Only `CASE.fit = 'po'` exists. Proposed mapping for the 10 `PRODUCTS`:

   | Product | Family |
   | --- | --- |
   | PO | po |
   | MCA | mca |
   | Secured Loan | term |
   | Credit Facility | wc |
   | Unsecured Loan | wc |
   | Equipment | asset |
   | Invoice | invoice |
   | Bridging | bridge |
   | Property Development | property |
   | Trade | trade |

   For "Not sure, match me", run every family and report the best. **Q4.**
5. **The form does not collect what matching needs.** Matching uses months trading and monthly turnover. The form collects 3 coarse trading bands and **annual** revenue bands. Proposal: add `date_registered`, or derive it from CIPC later, plus an "average monthly turnover" number field. Until then, use band midpoints ÷ 12 and say so in the results.
6. **The security form has no code mapping.** The 7 `SECURITY_OPTS` need codes. Proposal: both "cede contract/PO" and "cede invoices" map to `cession`, and "Card turnover" sets `card = true`.
7. **Industry exclusions never fire.** 'Property speculation', 'Primary agriculture' and 'On-lending' are not values in `INDUSTRIES`. Proposal: make exclusions a sub-list of the industry enum, and flag these 3 for re-entry when criteria are confirmed.
8. **Formula drift.** Phase-3 notes give base 58, +9 per grade, +12 fit. The handoff and the prototype give 52, capped +9, +22/+6. The prototype wins; I'll treat the phase-3 text as stale.
9. **Pre-qual can only ever return B or C.** The midpoint estimate always falls between 26.0 and 33.3 points. The A copy is unreachable, which the brief intends, but D is unreachable too. The D copy ("development finance and grants are the honest starting point") is therefore dead until the new judgements question is added. Proposal: "Yes" to judgements/defaults applies the D knockout cap to the pre-qual band. **Q5.**
10. **Pre-qual and full matching differ.** Pre-qual uses exact families only, ignores cession, surety, card and equity, and maps "Growth" to an `equity` family that has no funders. Proposal: keep pre-qual deliberately looser, as it is labelled indicative, but exclude card desks unless the client says they have card turnover.
11. **Weights are both code and admin-editable.** The handoff says port `BM` as `metrics.ts`. The build spec and runbook say weights are admin-edited and versioned. Proposal: the engine takes the model as a parameter, `metrics.ts` becomes model **v1** (the seed), and the DB stores immutable versions. The fit-score constants get the same treatment.
12. **"Likelihood" survives in names.** The build spec has `likelihood_pct` and the prototype has `bmFunders.likelihood`. Everything becomes `fitScore` / `fit_score`.
13. **Copy contradicts criteria.** Grade C is described as "Conditional PO and bridging only", and the age prescription says "under 24 months limits you to PO and bridging". The criteria, however, accept C or 12 months at about 17 working-capital, MCA and invoice desks. Either the copy or the criteria are wrong. I'm leaving both verbatim and listing this in TODO-BUSINESS.

### Permissions and masking
14. **Broker `docsRelease: 1`** in `ROLES` contradicts `PERM_MATRIX`, both specs and the brief. It only works because release also needs `funders`. Resolution: broker release is off.
15. **Alias design.**
    - The brief asks for "stable hash of funder ID with salt" *and* sequential `Funder 01…31`. A hash cut down to two digits will collide.
    - The prototype's alias follows list order. Anyone holding the supplied funder list can decode it.

    Proposal: each funder gets an alias number once, at insert, from a salt-seeded shuffle. It is stored on the admin-only funder row and never reused.
16. **The alias alone does not hide identity.**
    - Non-admins see each funder's bespoke type text ("Contract backed crowd funding", "Card turnover advance", "Property backed bridging"), plus ticket and decision days. That identifies several desks outright.
    - The prototype also shows the criteria table to every role, which the build spec (§8) forbids.

    Proposal: non-admins see the family label, not the type text, and the criteria table is admin-only. Gap text keeps the numbers, since brokers need them.
17. **Free-text masking by string substitution** cannot be done safely on the server. Proposal: never store funder names in free text such as timeline entries or tasks. Store a `funder_id` reference and render it through `presentFunder`. Staff-written notes and messages get a server-side check that rejects any real funder name typed by a non-admin.
18. **Funders visible to clients.** The client portal lists named best-fit financiers before submission, which is the same disintermediation risk as with brokers. Proposal: before submission clients see families and aliases; real names appear once a submission or offer exists. **Q6.**
19. **Shared Finance login.** `STAFF_ACCOUNTS` has "Finance (shared)" as a Broker, which gives it file access. The build spec says shared mailboxes get billing scope only. **Q7.**
20. **The permission catalogue is incomplete.** The prototype has 9 keys, while build spec §8 has around 20 actions including client roles, verify/reject document, offers and audit read. Proposal: `permissions.ts` implements the build-spec §8 table, mapped onto the prototype's matrix text.
21. **36 funders, not 31.** There are 31 commercial funders plus SEFA, IDC, NEF, NYDA and DTIC. Aliases run 01–36. The funder contact emails ("from your list") are not in the pack; I need them for seeding, but not for step 1.

### Score status and gates
22. **Definition of provisional.** The prototype treats a score as provisional while fewer than 14 metrics are answered. The brief adds "until every metric is verified".
    - Growth metrics are "analyst judgement" and have no document to verify against.
    - The credit metrics can only be verified by the bureau.

    So **no file can be submitted until a bureau contract exists**, and that includes the week-10 pilot. **Q8.**
23. **Stage gates.** `under_review → matched` requires a non-provisional score. With the stricter definition, a file cannot even reach Matched before verification. Proposal: Matched needs all 14 answered; Submission needs all 14 verified.
24. **Stage vocabularies.** The client portal uses Submitted / Graded / With financiers / Offers in / Funded / Declined. Staff use Submitted / Under review / Matched / Funded / Rejected, plus draft and withdrawn. Proposal: one stored stage, with the client label derived from it, and "Rejected" never shown to a D-grade client as a decline, since D is always a referral. **Q9.**

### Documents and storage
25. **Primary store.** The brief makes Drive primary. The build spec, `CONNECTORS`, the runbook env vars and the object-created webhook all assume S3/R2 primary with Drive as a mirror. Following the brief, I'll drop the S3 connector and its webhook. Consequences:
    - Virus scanning becomes a job that downloads the file from Drive and scans it.
    - "15-minute signed URLs" become streamed downloads through the app.
    - Large client downloads need streaming responses to stay under Vercel's body limit.
26. **Drive auth model.** The runbook uses domain-wide delegation with an impersonated user. The brief and Netlify §3 use a Shared Drive with the service account as Content manager. Proposal: Shared Drive with no delegation, a smaller blast radius. External sharing must be allowed on that Shared Drive for `_Releases`. I won't rely on Drive's own permission expiry; the hourly job revokes. I'll confirm the Drive API details when I build it.
27. **Folder per company vs reference per application.** The folder is named `{Registered name} - {FA reference}`, but the vault belongs to the company and is reused across applications. Proposal: name the folder after the first application's reference, put later applications in the same folder, and store the Drive ID.
28. **Upload size and types.** The form copy says 15 MB (phase 1), the build spec and tests say 25 MB. Types: phase 1 allows PDF, JPG, PNG, XLSX and DOCX, the build spec adds HEIC. Proposal: 25 MB and the union of both lists. The 15 MB copy gets updated.
29. **`DOC_RULES` are prose.** Proposal: encode them as structured rules. That needs:
    - a financial year-end date for AFS
    - a delivery date on deal documents
    - "above R1m" read as *amount requested*
    - "bank desk" as a funder category flag (Nedbank and Investec)

    The form's `DOCS` list includes Business Plan, Pitch Deck and Proof of Business Address, which `DOC_RULES` lacks. These become optional, non-gating types.
30. **Retention clash.** Rejected files that never progressed are kept 24 months, but client documents are kept 7 years after closing. Proposal: 24 months applies unless FICA identity documents were collected. This needs legal sign-off (TODO-BUSINESS).

### Auth and stack
31. **Auth.js credentials provider vs database sessions.** Auth.js's Credentials provider only supports JWT sessions, so staff password + TOTP cannot use database sessions directly. Auth.js is also now maintained by the Better Auth team. Options:
    - **(a)** Auth.js for the client magic link, plus a custom staff sign-in that verifies password + TOTP and writes the session row itself, so both share one sessions table.
    - **(b)** Better Auth, which has magic link, TOTP and Drizzle-backed database sessions built in.

    **Q10.**
32. **Client passwords.** The prototype's sign-in has passwords, a strength meter and a "forgot password" state. The brief says clients use magic links. Proposal: magic link only in v1, with the screens adapted. Less to secure.
33. **Reference format.** `FA-####`, `FA-nnnn`, `FA-2026-000142` and `FA-YYYY-000001` all appear. Following the brief: one global Postgres sequence that never resets, formatted `FA-{year}-{6 digits}`. The inbound-mail pattern `\[FA-(\d{4})\]` becomes `\[FA-(\d{4}-\d{6})\]`.
34. **Table names.** The brief (bm_answer, bm_score, release, audit_event) differs from the build spec (score_answer, score, release_log, audit_log, match_run). Brief names win. I'll also add the build-spec tables the brief omits: `match_run`, `score_model_version`, `user`, `company_user`, `webhook_event`, `alert`, `task`, `invoice`, `connector_config`, `call_booking`, `prequal_lead`. `bm_answer.source` becomes `self_declared | analyst | bureau_soft | bureau_full`, with `verified_by` and `verified_at` kept separate.
35. **Job times.** The expiry sweep is 07:00 in the automations and build spec, but 02:00 in the runbook and prompt. Proposal: compute at 02:00, send nudges at 09:00. All sends stay within quiet hours (08:00–18:00) and are limited to 3 per client per 7 days.
36. **What STOP turns off.** The connector runbook says STOP sets `marketing_opt_in = false`. But the SMS consent is for *application updates*, which are transactional. Resolution: STOP revokes the SMS/WhatsApp consent row entirely.
37. **Overseas processors (POPIA).** Inngest, Postmark and Sentry process data outside South Africa. Proposal: send IDs only in job payloads, never personal data, and record each processor in the cross-border register.

### Offers and copy
38. **The sample offers prove the badge wrong.**
    - Retail Capital is badged "Lowest cost": R93 500 on R850k over 90 days, about 44.6% a year (simple).
    - Nedbank: R118 000 over 120 days, about 42.2% a year. On annualised cost, Nedbank is cheaper.
    - Neither stated total matches its stated rate and fee: 2.2%/month × 3 + 1% gives R64 600, not R93 500.

    Annualised cost needs:
    - a convention, **Q11**
    - a prime rate as a dated admin setting
    - for MCAs, an assumed repayment period
39. **Two disclosure texts** exist (l.319 and l.1378). I'll port both verbatim, each where it appears. The l.1378 text promises "We will tell you what we earn on any offer if you ask". Once commission rates exist, should commission be shown on each offer automatically? This is not blocking (TODO-BUSINESS).
40. **Hero images and logo.** Your two identical café photos match the prototype's variant-A hero image. The blue-blazer portrait has no slot in the prototype. `assets/logo-wordmark.png` and the reversed logo were not supplied. **Q12.** None of these photos will be presented as a testimonial; the prototype itself warns against invented ones.

---

## 4. Questions

### Blocking step 1 (engine and tests)
- **Q1. Grade on the rounded total?** I propose grading on the total rounded to 1 dp, so the number shown and the grade always agree. Boundary tests are written against whichever you choose.
- **Q2. Security exemption for same-family funders.** Keep the prototype, where a PO desk needing a cession passes when no cession is offered? Or apply the brief, where security is tested for every family?
- **Q3. Grants in full matching.** Separate eligibility track, as proposed, and grant funders shown only to C/D files or always? Which ownership fields gate each scheme (NYDA youth %, NEF black ownership, etc.)? If unknown, I'll build the track with placeholder rules flagged unconfirmed.
- **Q4. Product → family mapping** as in item 4, especially Unsecured Loan (wc or term?) and "Not sure".
- **Q5. Pre-qual judgements answer.** Does "Yes" cap the indicative band at D, as the knockout does? Should the application form's existing judgements question also pre-fill `creditScore` as self-declared option 3?
- **Q11. Annualised cost convention.** I recommend an effective annual rate from the full cash-flow schedule (IRR, compounded), applied identically to every instrument, with fees included. Alternatives: simple annualised cost, or the NCA-style method.

### Needed before step 2
- **Q6.** Should clients see real funder names before submission?
- **Q7.** Should the shared Finance login be dropped, or should a fourth billing-only `finance` role be added?
- **Q8.** Credit metrics can only be verified by the bureau, so nothing can be submitted until the bureau contract lands. Accept that, including for the pilot? Or let analysts verify credit against a bureau report the client supplies?
- **Q9.** What should a client see when a file is rejected or referred to advisory? Proposed: "Not placed yet: advisory route".
- **Q10.** Auth.js with a custom staff sign-in, or switch to Better Auth? I lean towards Better Auth, but it is a stack change, so it's your call.

### Needed by step 3
- **Q12.** Which hero ships: A (Editorial), B (Guided), or an A/B test? Where does the blazer portrait go? Please send the logo files (normal and reversed) and confirm you have a licence for the photos.
