# Deployment runbook: Netlify + Google Drive

Replaces the Vercel hosting line in build-spec.md section 2. Everything else in the spec stays the same.

## 1. Stack on Netlify

| Layer | Choice | Notes |
| --- | --- | --- |
| App | Next.js (App Router), one deployment | Netlify runs Next.js natively. Marketing site, portal and console in one codebase |
| Login | Auth.js with database sessions | Already in the spec. Does not depend on Netlify Identity, which Netlify has deprecated |
| Database | Postgres (Neon or Supabase) | Holds applications, scores, stages, messages, consent and audit. Drive cannot do this |
| Documents | Google Drive Shared Drive | See section 3 |
| Jobs | Netlify Scheduled Functions + Background Functions, or Inngest | Background Functions run up to 15 minutes. Use Inngest if you want retries and queues without Redis |
| Email | Postmark | Inbound webhook points at `/api/webhooks/postmark` |

## 2. Login

- Clients: email magic link through Postmark. Password optional.
- Staff: password plus TOTP, mandatory. Optionally restrict staff sign-in to your Google Workspace domain.
- Roles on the `staff` table: `admin`, `analyst`, `broker`. Check them in every route handler and server action, not only in the UI.
- Funder masking: the API returns `Funder 01 … 31` to non-admins. The real name never leaves the server for those roles.
- Sessions in Postgres, cookie `httpOnly`, `secure`, `sameSite=lax`.

## 3. Google Drive as the document store

### One-time setup
1. Google Workspace account (required: service accounts have no storage of their own, so uploads to a personal My Drive fail).
2. Create a Shared Drive, e.g. `Funding Axis - Client Files`.
3. Google Cloud project → enable Drive API → create a service account → download the JSON key.
4. Add the service account email to the Shared Drive as **Content manager**.
5. Store the key in Netlify as `GOOGLE_SERVICE_ACCOUNT_JSON` (base64 encoded).

### Folder structure
```
/{Registered name} - {FA-reference}/
  01 Company/
  02 Financials/
  03 Bank statements/
  04 Directors/
  05 Offers and contracts/
  _Releases/            (one subfolder per funder release, admin only)
```
Store `drive_folder_id` on `company` and `drive_file_id` on `document`. Never look files up by name.

### Uploads
Netlify Functions cap request bodies at about 6 MB, so files must not pass through them.
1. Client asks `/api/documents/upload-url`. The function checks the session, creates a Drive **resumable upload session** in the right folder and returns the session URL.
2. The browser uploads straight to Google.
3. The browser calls `/api/documents/complete` with the file ID. A background job verifies size and sha256, writes the `document` row, sets validity and recomputes pack completeness.

### Downloads
- Small files (under ~5 MB): stream through a function after a role check, and log the access.
- Larger files: grant the staff member's Workspace account temporary reader access to that file, expiring after 1 hour, and log it.
- Clients never get Drive links. They download through the app.

### Release to funders (admin only)
1. Admin selects documents and a funder.
2. A background function copies them into `_Releases/{date} {Funder}/`, writes the release log and consent reference.
3. Share that folder with the funder's email, with an expiry. Never share the client's main folder.
4. Revoking a release removes the permission and logs it.

### Trade-off to decide
Drive has no signed URLs, no versioning guarantees and API quotas. The spec's default is private S3 or R2 as the primary store with Drive as a one-way mirror for staff. Use Drive as primary if staff must keep working in Drive. Switch to R2 with a Drive mirror if file volume or release speed becomes a problem. The data model is the same either way.

## 4. Netlify configuration

`netlify.toml`
```toml
[build]
  command = "npm run build"
  publish = ".next"

[functions]
  node_bundler = "esbuild"

[[headers]]
  for = "/*"
  [headers.values]
    Strict-Transport-Security = "max-age=63072000; includeSubDomains; preload"
    X-Frame-Options = "DENY"
    X-Content-Type-Options = "nosniff"
    Referrer-Policy = "strict-origin-when-cross-origin"
```

### Environment variables
| Key | Purpose |
| --- | --- |
| `DATABASE_URL` | Postgres, pooled connection |
| `AUTH_SECRET` | Auth.js session signing |
| `AUTH_URL` | Production URL |
| `POSTMARK_SERVER_TOKEN` | Outbound transactional |
| `POSTMARK_NUDGE_TOKEN` | Outbound nudges stream |
| `POSTMARK_INBOUND_SECRET` | Webhook verification |
| `GOOGLE_SERVICE_ACCOUNT_JSON` | Drive access, base64 |
| `GOOGLE_SHARED_DRIVE_ID` | Root Shared Drive |
| `FUNDER_ALIAS_SALT` | Stable alias mapping |

Set separate values for the `production` and `deploy-preview` contexts. Deploy previews must point at a staging database and a staging Shared Drive, never production.

### Scheduled jobs
| Job | Schedule |
| --- | --- |
| Document expiry sweep | Daily 02:00 SAST |
| Client nudges | Daily 09:00 SAST |
| Alert escalation | Every 15 minutes |
| Drive permission expiry cleanup | Hourly |
| Retention purge | Weekly |

## 5. Go-live checklist
- [ ] Custom domain and HTTPS on Netlify
- [ ] Postmark sender domain verified (SPF, DKIM, DMARC)
- [ ] Two admin accounts with TOTP
- [ ] Database backups with point-in-time recovery enabled
- [ ] Scheduled export of the Shared Drive (Drive is not a backup)
- [ ] Data processing agreements with Google, the database host and Postmark
- [ ] Storage regions recorded in the privacy notice (POPIA section 72)
- [ ] Test: a broker account cannot see a funder name in any API response
- [ ] Test: a 25 MB bank statement uploads and downloads
- [ ] Test: revoking a release removes the funder's access
