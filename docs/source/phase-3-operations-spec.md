# Funding Axis - Phase 3 operations console

Internal staff view. Design covers: applications pipeline with a book-wide timeline, client file, chat-to-email thread, financier matching engine, alerts, and a setup tab holding funder criteria and the permission matrix. Open the console from the grey bar at the top of the design ("P3 · Operations console") and switch role in the sidebar.

## Staff roles

Three roles, set per staff member. The console re-renders on the role, it does not just hide buttons.

| Action | Analyst | Broker | Admin |
| --- | --- | --- | --- |
| Open any client file, score and diagnostics | Yes | Yes | Yes |
| Score or re-score the Bankability Metric | Yes | Read only | Yes |
| Request documents from a client | Yes | Yes | Yes |
| See funder names and contact details | No | No | Yes |
| See match results, likelihood and gaps | Masked names | Masked names | Yes |
| Release vault documents to a financier | No | No | Yes, with client consent |
| Submit a file to a funder | No | No | Yes |
| See and record fees and commission | No | Yes | Yes |
| Advisory invoices and payment status | No | Yes | Yes |
| Edit funder criteria and the metric weighting | No | No | Yes |
| Add staff, set roles, read the POPIA audit log | No | No | Yes |

Notes:
- Funder identities are admin-only. A broker who knows the desk can take the client there directly, so names, contacts and the criteria table are withheld and the matching engine serves aliases (Funder 01 … Funder 31) with the likelihood, timeline and gaps intact. Submission and release follow the same restriction.
- This makes the admin a bottleneck on every submission. Two admins, not one.
- Scoring sits with the analyst and the admin so the broker cannot move a grade on a file they are selling into. Worth confirming - if brokers must be able to re-score, the audit log needs to record who changed what and why.
- Payment alerts are filtered out of the analyst's alert list entirely, not greyed.
- Every release of a document is written to `release_log` with the acting staff member and the consent record it relied on.

## Data schema for matching

Two tables drive the engine. The Bankability Metric supplies the grade; the funder table supplies the gate.

`funder`
| Field | Type | Notes |
| --- | --- | --- |
| `name` | text | From your list of financiers |
| `instrument` | enum | Term loan, revolving facility, working capital, invoice discounting, merchant cash advance, bridging, purchase order, asset finance, trade finance, growth or mezzanine, property backed |
| `fit_family` | enum | `po`, `bridge`, `invoice`, `trade`, `wc`, `mca`, `asset`, `property`, `term`, `growth`. Used to test instrument fit against what the client is asking for |
| `ticket_min` / `ticket_max` | integer, ZAR | Hard gate |
| `min_grade` | A / B / C / D | Hard gate, compared to the Bankability grade |
| `min_months_trading` | integer | Hard gate |
| `min_monthly_turnover` | integer, ZAR | Hard below 75% of the guide, soft between 75% and 100%. Zero means deal led, no turnover test |
| `security_required` | enum | `none`, `cession`, `asset`, `property`, `surety`, `equity`, `card` |
| `industry_exclusions` | text array | Matched against the client's industry, hard gate |
| `decision_days` | integer | Working days to a decision, drives the timeline shown to staff |
| `contact_email` | text | From your list |
| `active` | boolean | Keeps a funder in the table but out of the engine |

`match_result` (computed per application per funder, not stored as truth)
| Field | Notes |
| --- | --- |
| `group` | `clear`, `near`, `out` |
| `hard_gaps` | Array of plain-language reasons, each naming the criterion and the client's value |
| `soft_gaps` | Fixable gaps. A file with only soft gaps is a `near` and is what advisory work is sold against |
| `likelihood` | Percentage. Base 58, plus 9 per grade of headroom above the funder's minimum, plus 12 for a natural instrument fit, plus up to 12 for turnover headroom, less 11 per soft gap. Excluded funders are capped in the low twenties so staff can still see how far off they are |
| `decision_days` | Funder's decision time plus four days where soft gaps exist |

The likelihood formula is a placeholder that behaves sensibly. Replace the constants with observed hit rates once there is enough placement history - the shape of the calculation does not need to change.

### What sits outside the five categories

The Bankability Metric grades the business. It does not answer instrument fit, ticket size, security on offer, industry exclusion or card turnover. Those are funder criteria, tested separately, and they are the reason a Grade A file can still be excluded by eleven funders. Keep them in the funder table rather than trying to fold them into the metric.

### Criteria values

The list of financiers you supplied carries funder names and email addresses only. Every criteria value in the design is our working assumption, marked as such on the Setup tab. Before go live each funder needs a short confirmation of: ticket range, minimum trading history, turnover guide, security, industries they will not touch, and turnaround. That is a one-page form per funder and it is the single highest-value data-collection task in Phase 3.

## Applications pipeline

- Stages: Submitted, Under review, Matched, Funded, Rejected. The stage is a field on `application`, changed by staff, with every change written to an audit trail.
- Stage counts and value in flight across the top; filter chips per stage.
- Each row carries the grade and score, the owner, time in stage, and its flags. Flags are the same computation as the alerts, scoped to the file.
- Fee and commission column is replaced with "Hidden at analyst level" rather than blank so the analyst knows something is there.
- Timeline below the table is the book-wide activity feed: submissions, client replies, funder requests, re-scores, drawdowns.

## Client file

One screen per application, holding: reference header with stage and owner, the Bankability score with data confidence, the diagnostic ranked by points lost with the prescribed intervention, every document with status and expiry, next steps each linked to the thing that generated them (a vault slot, a sub-metric, a release), and the POPIA release log.

Data confidence is explicit: an incomplete metric shows "Provisional, n unanswered, treat the score as a floor". A provisional score should not be releasable to a funder - suggest we block submission until all 14 are answered.

## Chat to email

- Outbound messages send from `applications@fundingaxis.co.za` with the reference in the subject line.
- Inbound replies to that address are matched on the reference and appended to the same thread, so a client who never signs in still appears in the conversation.
- Portal replies land in the same thread. Channel is recorded per message (Email, Portal, Internal note) and shown.
- Internal notes never leave the console and are visibly separated.
- Templates are staff-editable snippets, not automated sends: documents outstanding, offer walkthrough, score improvement, funder update.
- Threads are visible to every staff member on the file, not only the owner.

To confirm: the mailbox and the reply parsing. This needs a real inbound address with a webhook (Postmark, SendGrid inbound, or Google Workspace routing rule). It is the only piece of Phase 3 that depends on infrastructure outside the app.

## Alerts

Rules as designed:
- Document expiring within 30 days, or already lapsed.
- Application section incomplete for more than 24 hours.
- Bankability score below the minimum grade of any funder the file is matched to.
- No client activity for seven days on a live file.
- Advisory invoice unpaid past due.

Each alert has a severity, a client, a reference, an owner and one action. Alerts escalate to the admin after 48 hours. Payment alerts are role-filtered.

## Still open

- Confirm the criteria values per funder (see above). Until then the engine's output is directional.
- Whether the broker may re-score.
- Whether a provisional score can be released to a funder at all.
- Notifications and staff invitations are not designed yet.
