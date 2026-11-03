# ClaimCare · Patient Locker & Auto Query Response

ClaimCare is a local full-stack claims operations portal for **Assignment 1**. It seeds ten synthetic patient profiles and supports profile management, patient document storage, query-letter ingestion, point-by-point matching, automatic templated follow-up, delivery attempts, and message history.

## Run locally

Requirements: Node.js 20.16 or newer and npm.

1. The folder includes a local `.env` with a demo login. Change `ADMIN_PASSWORD` before use. Keep `SESSION_SECRET` and `DATA_ENCRYPTION_KEY` stable after data is created; changing the encryption key makes existing case files unreadable.
2. Run `npm install`, then `npm start`.
4. Open `http://127.0.0.1:4173` and sign in with the password in `.env`.

## Deploy from GitHub to Render

`render.yaml` configures a Node web service, health check, and a persistent data disk mounted at `/var/data`. The app reads `DATA_DIR`, so patient records and encrypted uploads survive deploys. Render's filesystem is otherwise ephemeral, and its persistent disks are available on paid services; confirm the service and disk cost in Render before creating it. The deployment requires these secrets in Render's Environment page: `ADMIN_PASSWORD`, `SESSION_SECRET`, and `DATA_ENCRYPTION_KEY` (exactly 64 hex characters). Set a strong, unique password and retain the encryption key securely; losing that key makes stored records and files unreadable. Provider variables can be added there as well. Never commit `.env`, patient data, or API tokens.

To deploy, create a GitHub repository containing this product folder, push the code, then create a Render Blueprint from that repository and review the service plan before deploying. Render will supply the public `onrender.com` URL. Use the deployment dashboard to enter secrets; do not put secret values in `render.yaml`.

The server binds to localhost only. Case records and uploads are encrypted at rest with AES-256-GCM using `DATA_ENCRYPTION_KEY`; records are stored under `data/`. File downloads and patient routes require workspace authentication. Use synthetic data for evaluation on this local demo; a real deployment still needs managed identity, audit/retention policy, backups and production access controls.

## WhatsApp + email

Query upload automatically attempts both channels and records each provider response; no send button is required. Provider secrets stay in the server `.env`:

- WhatsApp Cloud API: `WA_PHONE_ID`, `WA_ACCESS_TOKEN`, `WA_API_VERSION` (the current version enabled for your Meta app), `WA_TEMPLATE_NAME`, `WA_TEMPLATE_LANGUAGE`. Business-initiated messages require a Meta-approved template. Configure the template body with four text parameters in this order: patient name, insurer, patient ID, requested items. The test number must be allowed by the configured WhatsApp test sender.
- Email: `RESEND_API_KEY` and `EMAIL_FROM`. The sender domain/address must be verified with Resend.

When credentials are absent, the system records `not_configured` for that channel and still records its message. Network/provider failures are captured per channel; one failing provider does not prevent the other send attempt. The retry endpoint only retries channels that have not already succeeded, avoiding duplicate patient messages. The included seed phone numbers and `example.com` email addresses are synthetic and cannot receive messages; replace them with your evaluation contacts before sending.

## Processing workflow

- Add documents from a patient profile. PDF text is extracted; scanned PDFs and image files use English OCR. The app categorizes from the file name and extracted text. If OCR identifies another sample patient's name, the document is flagged for review and excluded from query matching.
- Upload a query letter. The original file and extracted text are attached to that patient's query. The parser separates numbered/bulleted lines and request sentences, expands a point that names multiple document categories, matches each request to a document category, and excludes documents flagged for review. Duplicate query text is rejected so a repeated upload cannot send a duplicate request.
- A simple-language message is produced from the patient-friendly template and immediately dispatched to both configured channels. Each requested item includes a short reason. Unmatched clarification points are included so the patient can reply with context. The portal records statuses and provider errors.

## Files

- `server.mjs` — authenticated API, persistent data, file processing, and delivery adapters
- `public/index.html` — responsive portal interface
- `.env.example` — provider setup reference
- `index.html` — setup landing page; the server hosts the portal at the localhost URL above

The rules-based query/document matcher is intentionally explainable and shows confidence/status for every point. It is a workflow automation aid; ambiguous requests should be reviewed by claims staff. The demo is not yet ready for a live external evaluation until you configure a real WhatsApp sender, approved template, verified email sender, and the evaluator's valid patient phone numbers and email addresses. The current `trycloudflare.com` URL is temporary and only works while the host computer and tunnel are running. A permanent evaluation URL needs a hosted deployment with persistent storage and protected secrets; do not deploy this JSON/file store on an ephemeral filesystem.
