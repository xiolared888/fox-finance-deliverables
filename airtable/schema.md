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
| --- | --- | --- | --- |
| Client ID | Record ID | Airtable record identifier for the client. | Yes (system) |
| First Name | Single line text | Applicant first name. | No |
| Last Name | Single line text | Applicant last name. | No |
| Full Name | Formula/Text | Combined display name for the client. | No |
| Email Address | Email | Client email used for intake and notifications. | No |
| Phone Number | Phone number | Client phone number. | No |
| Date of Birth | Date | Client date of birth. | No |
| Address | Long text | Client mailing/residential address. | No |
| Loan Applications | Linked records (Applications) | Related application record IDs for this client. | No |
| Created At | Created time | Timestamp when client record was created. | No |
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
| --- | --- | --- | --- |
| Application ID | Record ID | Airtable record identifier for the application. | Yes (system) |
| Client | Linked record (Clients) | Linked client record ID for this application. | Yes |
| Full Name | Single line text | Applicant name snapshot at time of submission. | No |
| Email | Email | Applicant email captured on application. | No |
| Application Date | Date/time | Submission timestamp. | No |
| Loan Amount Requested | Currency | Requested loan amount. | No |
| Loan Purpose | Long text | Freeform reason/purpose for requested loan. | No |
| Status | Single select | Application state (for example: Submitted, Under Review, Documents Requested, Approved, Rejected). | No |
| Denial Reason | Long text | Reason provided when application is denied. | No |
| Upload Links | Linked record (Upload Links) | Upload link record ID tied to this application. | No |


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
| --- | --- | --- | --- |
| Upload Link ID | Record ID | Airtable record identifier for the upload link. | Yes (system) |
| Loan Application | Linked record (Applications) | Application record ID this upload link belongs to. | No |
| clientId | Linked record (Clients) | Client record ID associated with the upload link. | No |
| Email | Email | Recipient email for the upload request. | No |
| token | Long text | Auth/access token for the upload link. | No |
| fullName | Single line text | Applicant name associated with the link. | No |
| expiresAt | Date/time | Link expiration date/time. | No |
| isActive | Checkbox | Indicates whether the upload link is active. | No |
| Documents | Linked records (Document Requests) | Document request record IDs associated with this link. | No |
| Created At | Created time | Timestamp when upload link record was created. | No |
| uploads | Linked records (Uploads) | Uploaded file names associated with this link. | No |


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
| --- | --- | --- | --- |
| Document ID | Record ID | Airtable record identifier for the document request row. | Yes (system) |
| uploadLinkId | Linked record (Upload Links) | Upload link record ID this request belongs to. | Yes |
| fullName | Single line text | Applicant name snapshot stored with request. | No |
| name | Single line text | Requested document name/type. | No |
| description | Long text | Guidance/details for the requested document. | No |
| Date Requested | Date/time | Timestamp when the document was requested. | No |
| status | Single select | Document status (for example: Requested, Uploaded, Approved, Rejected). | No |
| uploads | Linked record (Uploads) | Uploaded file names tied to the request. | No |


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
| --- | --- | --- | --- |
| fileName | Attachment/File name | Uploaded file name. | No |
| uploadId | Text/ID reference | Upload identifier stored with the file record. | Yes (System) |
| fullName | Single line text | Applicant name associated with the file. | No |
| Documents (from uploadLinkId) | Lookup/Text | Document labels derived from the related upload link/document requests. | No |
| uploadLinkId | Linked record (Upload Links) | Upload link record ID associated with the file. | No |
| documentRequestId | Linked record (Document Requests) | Document request record ID this file satisfies. | No |
| fileSize | Number | File size metadata (bytes). | No |
| fileType | Single line text | MIME type or extension metadata. | No |
| s3Key | Single line text | Object key/path in S3 storage. | No |
| s3Bucket | Single line text | S3 bucket name where file is stored. | No |
| uploadedAt | Date/time | Timestamp when the file upload completed. | No |
| download_url | URL | Signed/public URL for download access. | No |


### Use Cases

- Created by: Document Request Flow (when S3 upload completes)
- Used by: Admin to verify documents, audit upload history
- Archived: Periodically moved to cold storage

---

## Relationships & Linking


**Key Relationships:**
- `Clients.Loan Applications` links to `Applications` (`Application ID` values).
- `Applications.Client` links to `Clients` (`Client ID` values).
- `Applications.Upload Links` links to `Upload Links` (`Upload Link ID` values).
- `Upload Links.Loan Application` links to `Applications` (`Application ID` values).
- `Upload Links.clientId` links to `Clients` (`Client ID` values).
- `Upload Links.Documents` links to `Document Requests` (`Document ID` values).
- `Document Requests.uploadLinkId` links to `Upload Links` (`Upload Link ID` values).
- `Uploads.uploadLinkId` links to `Upload Links` (`Upload Link ID` values).
- `Uploads.documentRequestId` links to `Document Requests` (`Document ID` values).


**Last Updated:** April 2026
