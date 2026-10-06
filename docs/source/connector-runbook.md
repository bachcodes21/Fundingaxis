# Connector runbook

Where the settings live, and the order to connect things in. The in-product version of this is the operations console → **Admin** tab (visible when your role is Admin). Each connector there shows its credential fields, the webhook URL to paste into the provider, and its setup steps, with **Save and connect**, **Test connection** and **Disconnect**.

## Where settings live, by type

| Setting | Where | Who changes it |
| --- | --- | --- |
| Third-party credentials and webhooks | Admin → Connectors | Admin, in-product; values are written to the secret store |
| Document types, validity, reminders, gating | Admin → Document rules | Admin |
| Metric weights, questions, grade bands, release rule | Admin → Metric model | Admin, versioned |
| Funder criteria and confirmation status | Setup tab → Funder criteria (Edit criteria → Mark confirmed) | Admin |
| Automations on/off | Admin → Automations | Admin, audit-logged |
| Staff accounts and roles | Admin → Staff accounts → Invite staff | Admin |
| Retention periods | Admin → Retention | Admin, with legal sign-off |
| Anything structural (schema, engine constants) | Code and migrations | Engineering, via PR |

Rule of thumb: if a number changes how a client is treated, it belongs in Admin. If it changes what the system is, it belongs in code.

## Environment variables

Secrets never live in the database or the repo. The console writes them to the managed secret store; the app reads them as env vars.

```
DATABASE_URL=                  # pooled
DATABASE_URL_DIRECT=           # migrations
S3_BUCKET= S3_REGION= S3_ACCESS_KEY_ID= S3_SECRET_ACCESS_KEY=
GOOGLE_SA_JSON=                # base64 of the service account key
DRIVE_ROOT_FOLDER_ID= DRIVE_IMPERSONATE=ops@fundingaxis.co.za
POSTMARK_SERVER_TOKEN= POSTMARK_STREAM_TX=outbound POSTMARK_STREAM_NUDGE=broadcast
POSTMARK_INBOUND_SECRET= MAIL_FROM=applications@fundingaxis.co.za
STITCH_CLIENT_ID= STITCH_CLIENT_SECRET=
BUREAU_SUBSCRIBER= BUREAU_API_KEY= BUREAU_PRODUCT=
CIPC_API_KEY= CIPC_CUSTOMER_CODE=
PAY_PUBLIC_KEY= PAY_SECRET_KEY= PAY_WEBHOOK_SECRET=
ESIGN_API_KEY= ESIGN_TEMPLATE_CONSENT= ESIGN_TEMPLATE_MANDATE=
SMS_API_KEY= SMS_SENDER_ID=
SENTRY_DSN= REDIS_URL= APP_URL=https://app.fundingaxis.co.za
ENCRYPTION_KEY_ID=             # KMS key for ID numbers and bank details
```

## Webhook endpoints the app exposes

All verify a provider signature and are idempotent on the provider's event id.

```
POST /api/hooks/postmark/inbound      client and funder replies
POST /api/hooks/postmark/delivery     delivery, bounce, complaint
POST /api/hooks/storage/object-created virus scan and document row
POST /api/hooks/payments              invoice paid, refunded, failed
POST /api/hooks/esign                 consent envelope completed
POST /api/hooks/stitch/refresh        bank feed refreshed or consent revoked
POST /api/hooks/sms/status            delivery and opt-out
GET  /api/health                      uptime plus queue lag
```

## DNS records needed

On `fundingaxis.co.za`: Postmark DKIM (TXT) and return-path (CNAME); SPF including Postmark; DMARC `p=quarantine` once DKIM has passed for a week; MX or a Workspace routing rule sending `applications@` to the Postmark inbound address. Nothing else in this build touches DNS.

## Connect in this order

1. **Database** and **document storage** — nothing else works without them. Enable PITR and bucket versioning at creation, not later.
2. **Outbound email** — verify the domain first; DKIM propagation is the slowest step in the whole setup, so start it on day one.
3. **Inbound email** — after MX or routing is live. Test by replying to a real outbound message and confirming it lands in the right thread.
4. **Error and uptime** — before any client traffic, so failures during setup are visible.
5. **Google Drive** — service account, delegation, then share the root folder.
6. **Payments** — provider KYC takes days; start it in parallel with the build.
7. **Backups** — enable, then rehearse a restore before go-live.
8. **CIPC**, then **e-signature** — both improve data quality and consent evidence but block nothing.
9. **Credit bureau** — the long pole: contract, compliance review of the consent wording, then credentials. Until it is live, scores stay flagged self-declared and are not released.
10. **Open banking** and **SMS/WhatsApp** — second release. Do not switch the SMS automations on until the templates are approved by the provider.

## Testing a connector

**Test connection** in the console performs a real, harmless round trip: a probe file uploaded and deleted for storage, a folder created and removed for Drive, a probe email to the admin address, a replayed sample inbound message showing which thread it matched, a R1 charge and refund for payments, a permitted test subject for the bureau. It never touches client data. Result and actor are written to the audit log.

If a test fails, the console keeps the previous status and shows the provider's error verbatim. The three failures worth expecting: DKIM not yet propagated (wait), Drive delegation scope missing (Workspace admin, not Cloud console), and a webhook signature mismatch (secret copied from the wrong environment).

## Failure handling in production

Webhook handlers retry with backoff and record attempts on `webhook_event`. Two failures within an hour raise an admin alert. Inbound mail that cannot be matched to a reference goes to a triage queue with an alert after two hours — this is the one failure mode that silently loses client replies, so it is monitored on its own dashboard tile. Drive and SMS failures never block a client action; storage and database failures do, and surface as a plain-language error with a reference the client can quote.
