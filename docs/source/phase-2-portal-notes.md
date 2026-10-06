# Funding Axis - Phase 2 client portal

Design covers: sign-in, overview, business profile, document vault, applications, offers. Notes for the build.

## Accounts
- Sign-in screen has four states: sign in, create account, forgot password, and a "check your inbox" confirmation. Validation is per-field and only shown after a submit attempt; the register form adds a password strength meter and a POPIA consent gate.
- An account is created automatically when a Phase 1 application is submitted; the applicant gets a set-password link. The create-account form exists for clients who arrive before applying.
- Support passwordless sign-in ("email me a sign-in link") as the primary path - SMME users forget passwords and this removes a support burden.
- One company file can have several users (directors, bookkeeper, accountant) with roles: owner, contributor, view-only. Design shows a single user; the data model should not assume it.

## Data model, roughly
- `company` - registered details, compliance, financial position. This is the system of record; the Phase 1 form writes into it rather than into a Drive folder alone.
- `document` - type, file, uploaded_at, valid_until, status derived from valid_until (Current / Expiring within 30 days / Expired) or absence (Outstanding).
- `application` - reference, product, amount, stage, financiers approached.
- `offer` - one per financier per application, with the fields shown on the offers screen.
- `release_log` - which documents were released to which financier, when, and on whose approval. Required for POPIA and shown to the client.

## Bankability Metric™ (now wired to your framework)
The Insights tab implements the framework document: 14 sub-metrics in five categories, scored out of 50, graded A to D.

- Weights as per the framework: Financial Health 4 x 2.5; Business Stability 3.33/3.33/3.34; Growth Potential 3.33/3.33/3.34; Management Quality 2 x 5.0; Credit History 2 x 5.0.
- Each sub-metric is a plain-language client question with four answers carrying a factor (1 / ~0.7 / ~0.4 / ~0.1). Score = factor x max points. Unanswered metrics score zero and the panel shows "x of 14 answered" so the number is never presented as final when it is provisional.
- Grades: A 41-50, B 31-40, C 21-30, D under 21, with the framework's readiness band and recommended action.
- Funder matching: top five per grade with a likelihood percentage. **The funder names and percentages in the design are illustrative** - replace them with your real Funder Mapping table (grade, funder, likelihood, funding type). The query logic in the framework maps one to one onto a lookup by grade ordered by likelihood.
- Advisory upsell is grade-driven: Route to Bankability (90 days, R3 000) for B and C, Full Turnaround (120-180 days, R2 000) for D, direct submission for A.
- Two lenses on the same data: **The client** sees the grade, funding likelihood and the advisory offer. **Funding Axis** sees diagnostic flags, a per-metric table of points scored, points lost and the prescribed intervention, plus lead quality, where to submit and data confidence. Same engine, no second source of truth.
- The overview card and its "what is holding your score back" list are derived from the same computation - the three metrics with the most points lost, each showing the points recoverable.

Still to confirm: whether answers are self-declared only, or cross-checked against vault documents and bureau data before the grade is released to a financier. The design says we cross-check and call the client where they disagree.

## Documents
- Vault is permanent and reused across applications. Uploading once should satisfy every future application.
- Expiry-driven nudges: email at 30 and 7 days before a tax pin, BEE affidavit or statement window lapses.
- Bank statement windows should be tracked as a date range, not a single file, so "six unbroken months" can be validated automatically.
- Consider Stitch or a similar open-banking feed as an alternative to uploads - it solves the "verified digital statements, not scans" requirement at the source.

## Drive
Phase 1's Drive folder stays as the broker-side working copy. In Phase 2 the portal is authoritative and syncs into Drive, not the other way around.

## Not yet designed
Notifications/settings, team invitations, the broker-side admin view of all clients, and in-portal messaging with your team. Say the word if you want any of these.
