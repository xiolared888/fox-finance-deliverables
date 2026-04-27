# Fox Finance Project Deliverables

This repository contains all automation workflows, configuration files, and documentation for the Fox Finance loan processing system.

## Project Overview

Fox Finance is an AI-powered loan automation system that streamlines application intake, document handling, customer support, and admin workflows. The system uses n8n for workflow orchestration, Airtable as the central database, and integrates with multiple services including Google Docs, Gmail, OpenAI, and Google Gemini.

**Repository Structure:**
```
fox-finance-deliverables/
  README.md                    # This file
  n8n-workflows/              # All automation workflows (JSON exports)
    contact-form-auto-reply.json
    loan-application-intake.json
    document-request-flow.json
    admin-dashboard.json
  airtable/                   # Airtable schema and sample data
    schema.md                 # Database structure documentation
    clients.csv
    applications.csv
    upload_links.csv
    document_requests.csv
    uploads.csv
```

---

## N8N Workflows

### 1. Contact Form Auto-Reply
**File:** `contact-form-auto-reply.json`

**What it does:**
Handles frequently asked questions (FAQ) submitted through the Fox Finance website's contact form. Uses AI to answer questions by referencing a knowledge base stored in Google Docs. If the AI cannot find an answer, it routes the inquiry to the admin team for manual follow-up.

**Trigger type:** Webhook (POST to `/contact-form`)

**Workflow path:**
1. Contact form submission comes through webhook
2. AI agent (Gemini or OpenAI) analyzes the question
3. AI fetches relevant information from Google Docs knowledge base
4. If answer found: sends personalized email reply to customer
5. If answer not found: sends escalation email to admin team with customer details

**Required credentials/services:**
- Gmail OAuth2 (for sending emails)
- Google Docs OAuth2 (for accessing knowledge base)
- OpenAI API (optional LLM model)
- Google Gemini API (primary LLM model)

**Required environment variables:**
None—all credentials are configured in n8n.

**Setup/import instructions:**
1. In n8n, click **Workflows** → **Import from file**
2. Select `contact-form-auto-reply.json`
3. Update credentials:
   - **Gmail account:** Select your Gmail OAuth2 credentials (must have "Send as" permissions)
   - **Google Docs account:** Select your Google Docs OAuth2 credentials
   - **OpenAI API / Gemini API:** Select or add your API credentials
4. Test the webhook by posting sample JSON:
   ```json
   {
     "name": "John Doe",
     "email": "john@example.com",
     "phone": "555-123-4567",
     "message": "What is the minimum loan amount?"
   }
   ```
5. Update the Google Doc URL if your knowledge base document changes

**Known limitations:**
- Webhook timeout: 30 seconds (responses must complete within this window)
- Airtable rate limits: 5 API requests per second
- If the knowledge base document is very long, response time may exceed webhook timeout
- AI responses are limited to the information in the Google Doc; external questions will be flagged for manual review

**Webhook URL (after activation):**
`https://[your-n8n-instance].n8n.cloud/webhook/contact-form`

---

### 2. Loan Application Intake
**File:** `loan-application-intake.json`

**What it does:**
Main workflow for capturing loan applications from the Lovable frontend. Validates applicant data, stores it in Airtable, triggers the AI agent to analyze the application for approval/denial/document request, generates secure upload links for required documents, and sends confirmation emails to applicants and admins.

**Trigger type:** Webhook (POST to `/loan-application`)

**Workflow path:**
1. Loan application submitted from Lovable frontend
2. Check if applicant exists in Airtable (by email)
3. If new applicant: create new client record
4. Create application record in Airtable with status "Submitted"
5. Trigger AI agent (OpenAI) to analyze application eligibility
6. Based on AI decision:
   - **Approve:** Generate AWS S3 upload link, send to applicant
   - **Request Documents:** Email applicant with list of required documents
   - **Deny:** Send denial reason email
7. Update application status in Airtable
8. Notify admin team of new application

**Required credentials/services:**
- Airtable API token (read/write to clients and applications tables)
- OpenAI API (for AI agent analysis)
- Gmail OAuth2 (for email notifications)
- AWS S3 (if using presigned URLs for document upload)
- FastAPI backend (for secure upload link generation)

**Required environment variables:**
- `AIRTABLE_API_KEY` (if not using OAuth)
- `OPENAI_API_KEY`
- `LOAN_BASE_ID` (Airtable base ID: `appqAO2An9337EWXu`)
- `AWS_S3_BUCKET_NAME` (if using S3)
- `FASTAPI_BACKEND_URL` (FastAPI server endpoint for upload links)

**Setup/import instructions:**
1. In n8n, click **Workflows** → **Import from file**
2. Select `loan-application-intake.json`
3. Update credentials:
   - **Airtable:** Select your Personal Access Token
   - **OpenAI API:** Add your API key
   - **Gmail:** Select your Gmail OAuth2 credentials
4. Update the Airtable base/table IDs if they differ from defaults
5. Configure FastAPI backend URL if using external API server
6. Test the webhook by posting sample JSON:
   ```json
   {
     "firstName": "Jane",
     "lastName": "Doe",
     "email": "jane@example.com",
     "loanAmount": 50000,
     "purpose": "Home Improvement"
   }
   ```

**Known limitations:**
- Webhook timeout: 30 seconds
- Airtable rate limits: 5 requests/second
- AWS S3 presigned URLs expire after 1 hour (configurable)
- AI analysis depends on prompt quality and model capability
- Email delivery depends on Gmail rate limits (may be delayed)

**Webhook URL (after activation):**
`https://[your-n8n-instance].n8n.cloud/webhook/loan-application`

---

### 3. Document Request Flow
**File:** `document-request-flow.json`

**What it does:**
Handles document uploads and status updates. When an applicant uploads required documents, this workflow validates the submission, stores the documents in AWS S3, updates the application record in Airtable with document metadata, and sends confirmation emails to both the applicant and admin team.

**Trigger type:** Airtable record update (when "Uploaded Documents" field is populated)

**Workflow path:**
1. Airtable detects new document in "Uploaded Documents" field
2. Validate file type and size
3. Move file to AWS S3 secure bucket
4. Update Airtable record with S3 URL and timestamp
5. Change application status to "Documents Received"
6. Send confirmation email to applicant
7. Notify admin team of document submission

**Required credentials/services:**
- Airtable API token (read/write to applications table)
- AWS S3 (for document storage)
- Gmail OAuth2 (for notifications)

**Required environment variables:**
- `AIRTABLE_API_KEY`
- `LOAN_BASE_ID`
- `AWS_S3_BUCKET_NAME`
- `AWS_ACCESS_KEY_ID`
- `AWS_SECRET_ACCESS_KEY`

**Setup/import instructions:**
1. In n8n, click **Workflows** → **Import from file**
2. Select `document-request-flow.json`
3. Update credentials (Airtable, AWS S3, Gmail)
4. Configure AWS S3 bucket settings
5. Activate the Airtable trigger to start monitoring document uploads
6. Test by uploading a document through the application form

**Known limitations:**
- File size limit depends on AWS S3 configuration (typically 5GB)
- S3 upload timeout: 300 seconds
- Document validation rules must be configured in the workflow
- Airtable automation triggers have a 5-second minimum delay

**Airtable Trigger Path:**
Workflow listens for changes to the "Uploaded Documents" field in the applications table

---

### 4. Admin Dashboard
**File:** `admin-dashboard.json`

**What it does:**
Provides a read-only API endpoint that returns all loan applications sorted by application date (newest first). Used by the Lovable frontend admin dashboard to display application status, borrower details, and upload links.

**Trigger type:** Webhook (GET/POST to `/admin-ui-test`)

**Workflow path:**
1. Admin dashboard requests application data
2. Fetch all records from Airtable applications table
3. Sort by "Application Date" (descending)
4. Parse and format records for frontend display
5. Return JSON response with count and records

**Required credentials/services:**
- Airtable API token (read-only to applications table)

**Required environment variables:**
- `AIRTABLE_API_KEY`
- `LOAN_BASE_ID`

**Setup/import instructions:**
1. In n8n, click **Workflows** → **Import from file**
2. Select `admin-dashboard.json`
3. Update Airtable credentials
4. Update base/table IDs if different
5. Test the webhook:
   ```bash
   curl -X POST https://[your-n8n-instance].n8n.cloud/webhook/admin-ui-test
   ```
6. Expected response:
   ```json
   {
     "count": 5,
     "records": [
       {
         "id": "rec123...",
         "Application ID": "APP-001",
         "Client": "Jane Doe",
         "Loan Amount Requested": 50000,
         "Status": "Approved",
         "Email": "jane@example.com"
       }
     ]
   }
   ```

**Known limitations:**
- Returns all records (no pagination)—consider adding pagination for large datasets
- Webhook timeout: 30 seconds (may timeout if Airtable has 1000+ records)
- Read-only endpoint—does not accept data updates
- Sorting is only by "Application Date"

**Webhook URL (after activation):**
`https://[your-n8n-instance].n8n.cloud/webhook/admin-ui-test`

---

## Airtable Setup

The system uses a single Airtable base called **Loan Application Automation** with the following tables:

### Tables Overview

**Base ID:** `appqAO2An9337EWXu`

1. **clients** (`tbljb8QrynopeYs5x`)
   - Stores borrower information
   - Fields: Email, Full Name, Phone, Address, Employment Status, etc.

2. **applications** (`tbl6QMr05RqJSOPBw`)
   - Stores loan applications
   - Fields: Application ID, Client (link), Loan Amount, Status, Application Date, etc.

3. **upload_links** (`tblUnicWkL9YYH79T`)
   - Stores secure S3 upload URLs
   - Fields: Application ID, Document Type, Upload Link, Expiration Date, etc.

4. **document_requests** (`tblkPad3w2UQZM6Qr`)
   - Tracks requested documents per application
   - Fields: Application ID, Document Type, Required, Received Date, etc.

5. **uploads** (`tbls4BjW2Z8hZQ0Rd`)
   - Stores metadata about uploaded files
   - Fields: Application ID, Filename, S3 URL, Upload Date, File Size, etc.

### Accessing Airtable

All workflows use the **Airtable Personal Access Token** for authentication. To set this up:

1. Log in to Airtable
2. Go to **Account** → **Personal Access Tokens**
3. Create a new token with scopes: `data.records:read`, `data.records:write`
4. Copy the token to n8n credentials

See `airtable/schema.md` for detailed field definitions.

---

## Credentials & API Keys

All workflows require the following external credentials to be configured in n8n:

| Service | Type | Where to Get | Used By |
|---------|------|-------------|---------|
| Airtable | Personal Access Token | airtable.com → Account → Personal Access Tokens | All workflows |
| Gmail | OAuth2 | Google Cloud Console → OAuth 2.0 Credentials | Contact Form, Loan Intake, Document Flow |
| OpenAI | API Key | platform.openai.com → API Keys | Loan Intake, Contact Form (optional) |
| Google Gemini | API Key | makersuite.google.com → API Keys | Contact Form (primary) |
| Google Docs | OAuth2 | Google Cloud Console → OAuth 2.0 Credentials | Contact Form |
| AWS S3 | Access Key + Secret | AWS Console → IAM → Access Keys | Document Flow |

---

## Deployment

### Quick Start (n8n Cloud)

1. Import each workflow JSON file into your n8n instance
2. Configure all required credentials (see above)
3. Activate each workflow
4. Copy webhook URLs to your Lovable frontend configuration
5. Test each workflow with sample data

### Self-Hosted n8n

1. Follow n8n self-hosted setup guide
2. Import workflows (same process as Cloud)
3. Configure Docker environment variables for credentials
4. Use ngrok or Cloudflare tunnel for external webhook access
5. See FastAPI README for backend setup

---

## Testing

Each workflow includes sample data in the JSON files (under `pinData`). To test:

1. Open the workflow in n8n
2. Find the **Webhook** node
3. Click **Copy** to get the webhook URL
4. Use curl, Postman, or similar to send test data:

```bash
# Contact Form
curl -X POST https://[webhook-url] \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Test User",
    "email": "test@example.com",
    "phone": "555-0000",
    "message": "How do I apply?"
  }'

# Loan Application
curl -X POST https://[webhook-url] \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "John",
    "lastName": "Applicant",
    "email": "john@example.com",
    "loanAmount": 75000,
    "purpose": "Auto Purchase"
  }'
```

---

## Troubleshooting

**Webhook timeout errors:**
- Check that all API calls complete within 30 seconds
- Consider breaking long workflows into multiple steps
- Enable async processing if available

**Airtable errors (rate limits):**
- Add delays between API calls (1-2 second minimum)
- Batch updates where possible
- Contact Airtable support if you exceed plan limits

**AI agent not responding:**
- Verify OpenAI/Gemini API keys are active
- Check that the knowledge base Google Doc is accessible
- Ensure the prompt is clear and specific

**Email not sending:**
- Verify Gmail OAuth2 credentials have "Send as" permission
- Check Gmail spam/trash folders
- Ensure sender email address matches OAuth2 account

---

## Support & Questions

For questions about specific workflows or setup, refer to:
- n8n docs: https://docs.n8n.io/
- Airtable API: https://airtable.com/api/
- OpenAI docs: https://platform.openai.com/docs/
- Google Gemini: https://ai.google.dev/

---

**Last Updated:** April 2026  
**Project:** Fox Finance AI Loan Automation System  
**Team:** Annette Partida, Greg Huynh, François Yengue
