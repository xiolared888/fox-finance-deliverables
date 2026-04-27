# Airtable Schema Documentation

## Base Information

**Base Name:** Loan Application Automation  
**Base ID:** `appqAO2An9337EWXu`  
**Purpose:** Central database for Fox Finance loan processing system

---

## Table 1: Clients

**Table ID:** `tbljb8QrynopeYs5x`

Stores borrower/applicant information. One record per unique applicant.

### Fields

| Field Name | Type | Description | Required |
|-----------|------|-------------|----------|
| Email Address | Email | Primary identifier for clients | Yes |
| Full Name | Single line text | Applicant's full name | Yes |
| Phone | Phone Number | Contact phone number | Yes |
| Address | Long text | Current residential address | Yes |
| Employment Status | Single select | Employed / Self-Employed / Unemployed / Retired | Yes |
| Employment Details | Long text | Current employer, position, years employed | No |
| Annual Income | Number | Gross annual income (USD) | Yes |
| Credit Score | Number | Estimated or provided credit score (300-850) | No |
| Existing Loans | Number | Number of outstanding loans | No |
| Date Created | Created time | Auto-populated system field | Auto |
| Notes | Long text | Internal notes about applicant | No |

### Use Cases

- Populated by: Loan Application Intake workflow (creates new if email doesn't exist)
- Referenced by: All workflows to link applicant data
- Used to: Avoid duplicate applications, personalize communications

---

## Table 2: Applications

**Table ID:** `tbl6QMr05RqJSOPBw`

Stores loan application requests. One record per loan application (applicant can have multiple applications).

### Fields

| Field Name | Type | Description | Required |
|-----------|------|-------------|----------|
| Application ID | Formula | Auto-generated unique ID (e.g., APP-001) | Auto |
| Client | Link to Clients table | Reference to applicant | Yes |
| Loan Amount Requested | Number | Requested loan amount (USD) | Yes |
| Purpose | Single select | Home / Auto / Business / Personal / Other | Yes |
| Status | Single select | Submitted / Under Review / Approved / Denied / Documents Requested / Complete | Yes |
| Application Date | Created time | When application was submitted | Auto |
| AI Decision | Single line text | AI agent's decision/analysis | No |
| Documents Requested | Multiple select | List of required documents | No |
| Uploaded Documents | Attachment | Documents submitted by applicant | No |
| Upload Links | Link to Upload Links table | S3 presigned URLs for document upload | No |
| Approval Reason | Long text | Reason for approval (if approved) | No |
| Denial Reason | Long text | Reason for denial (if denied) | No |
| Follow-up Notes | Long text | Admin notes on application | No |
| Last Updated | Last modified time | Timestamp of last change | Auto |

### Status Flow

```
Submitted → Under Review → (AI Analysis)
                          ├→ Approved → Complete
                          ├→ Denied → Complete
                          └→ Documents Requested → (Wait for Upload) → Under Review
```

### Use Cases

- Created by: Loan Application Intake workflow
- Updated by: Document Request Flow, Admin Dashboard
- Used to: Track application lifecycle, generate reports, display admin dashboard

---

## Table 3: Upload Links

**Table ID:** `tblUnicWkL9YYH79T`

Stores temporary, secure S3 upload links generated for document submission.

### Fields

| Field Name | Type | Description | Required |
|-----------|------|-------------|----------|
| Upload Link ID | Formula | Auto-generated unique ID | Auto |
| Application ID | Link to Applications | Reference to loan application | Yes |
| Document Type | Single select | ID / Proof of Income / Tax Returns / Bank Statement / Paystub / Other | Yes |
| Upload URL | Long text | AWS S3 presigned URL (example: https://s3.aws.../documents/app-001?signature=...) | Yes |
| Expiration Date | Date | When the presigned URL expires (typically 1 hour after generation) | Yes |
| Created At | Created time | When URL was generated | Auto |
| Status | Single select | Active / Expired / Used | Yes |
| Notes | Long text | Instructions or notes for applicant | No |

### Use Cases

- Created by: Loan Application Intake (FastAPI backend generates S3 URLs)
- Sent to: Applicant via email
- Used by: Applicant to upload documents to S3
- Tracked by: Document Request Flow to confirm upload

---

## Table 4: Document Requests

**Table ID:** `tblkPad3w2UQZM6Qr`

Tracks which documents are required for each application and whether they've been received.

### Fields

| Field Name | Type | Description | Required |
|-----------|------|-------------|----------|
| Request ID | Formula | Auto-generated unique ID | Auto |
| Application ID | Link to Applications | Reference to loan application | Yes |
| Document Type | Single select | ID / Proof of Income / Tax Returns / Bank Statement / Paystub / Other | Yes |
| Required | Checkbox | Is this document required for approval? | Yes |
| Received | Checkbox | Has the document been received? | Yes |
| Received Date | Date | When the document was uploaded | No |
| File Size | Number | Size in bytes (auto-filled from S3) | No |
| Notes | Long text | Reviewer notes or feedback on document | No |
| Requested Date | Created time | When the request was sent to applicant | Auto |

### Use Cases

- Created by: Loan Application Intake (based on AI analysis)
- Updated by: Document Request Flow (when documents upload)
- Used by: Applicant to see what's needed, admin to track completeness

---

## Table 5: Uploads

**Table ID:** `tbls4BjW2Z8hZQ0Rd`

Stores metadata about each file uploaded by applicants.

### Fields

| Field Name | Type | Description | Required |
|-----------|------|-------------|----------|
| Upload ID | Formula | Auto-generated unique ID | Auto |
| Application ID | Link to Applications | Reference to loan application | Yes |
| Filename | Single line text | Original filename (e.g., "tax-return-2023.pdf") | Yes |
| Document Type | Single select | ID / Proof of Income / Tax Returns / Bank Statement / Paystub / Other | Yes |
| S3 URL | Long text | Full S3 path (https://bucket.s3.aws.com/path/to/file.pdf) | Yes |
| File Size | Number | Size in bytes | Yes |
| Upload Date | Created time | When file was uploaded to S3 | Auto |
| Upload IP | Single line text | IP address of uploader | No |
| Virus Scan Status | Single select | Pending / Clean / Infected / N/A | No |
| Extracted Text | Long text | OCR-extracted text from document (if applicable) | No |
| Notes | Long text | Admin notes about the upload | No |

### Use Cases

- Created by: Document Request Flow (when S3 upload completes)
- Used by: Admin to verify documents, audit upload history
- Archived: Periodically moved to cold storage

---

## Relationships & Linking

```
Clients (1) ──────→ (many) Applications
              │
              └───→ (many) Document Requests
                    │
                    └─→ (many) Uploads
                    │
                    └─→ (many) Upload Links
```

**Key Relationships:**

1. **Clients → Applications:** One applicant can have multiple loan applications
2. **Applications → Upload Links:** One application generates multiple upload URLs for different document types
3. **Applications → Document Requests:** One application can request multiple documents
4. **Document Requests → Uploads:** Each document request can receive one or more uploaded files

---

## Data Entry & Automation

### Manual Entry (Admin Only)
- Clients: Full Name, Phone, Address, Employment Status, Annual Income (if not auto-imported)
- Applications: Follow-up Notes, Approval/Denial Reason

### Auto-Populated (Workflows)
- Clients: Email, Date Created (first application submission)
- Applications: Application ID, Application Date, Status, AI Decision
- Upload Links: Upload Link ID, Upload URL, Expiration Date, Created At
- Document Requests: Request ID, Requested Date, Received Date (when documents upload)
- Uploads: Upload ID, S3 URL, Upload Date, File Size, Created At

---

## Sample Data

### Example Client Record
```
Email Address: jane.doe@example.com
Full Name: Jane Doe
Phone: (555) 123-4567
Address: 123 Oak Street, Springfield, IL 62701
Employment Status: Employed
Employment Details: Senior Manager at Tech Corp, 8 years
Annual Income: 120000
Credit Score: 720
Existing Loans: 1
```

### Example Application Record
```
Application ID: APP-001
Client: Jane Doe [linked record]
Loan Amount Requested: 75000
Purpose: Home
Status: Documents Requested
Application Date: 2026-04-20 14:30 UTC
AI Decision: "Eligible for approval pending income verification"
Documents Requested: [Proof of Income, Tax Returns, Bank Statement]
```

---

## API Endpoints (Airtable)

To interact with these tables programmatically:

```bash
# List all applications
GET https://api.airtable.com/v0/appqAO2An9337EWXu/tbl6QMr05RqJSOPBw

# Create new client
POST https://api.airtable.com/v0/appqAO2An9337EWXu/tbljb8QrynopeYs5x
Content-Type: application/json
Authorization: Bearer [PAT]

{
  "records": [
    {
      "fields": {
        "Email Address": "new@example.com",
        "Full Name": "John Smith",
        "Phone": "(555) 987-6543",
        ...
      }
    }
  ]
}
```

---

## Performance Notes

- **Airtable Rate Limit:** 5 requests per second
- **Record Limit:** No theoretical limit, but UX slows with 10,000+ records
- **Field Limit:** 500 fields per table (currently using ~15)
- **Attachment Limit:** 20 MB per file (enforced by Airtable)

For large datasets, consider:
- Archiving old records to a separate table
- Adding filtering views for admin dashboard (e.g., "Pending Documents")
- Implementing pagination in API calls

---

**Last Updated:** April 2026
