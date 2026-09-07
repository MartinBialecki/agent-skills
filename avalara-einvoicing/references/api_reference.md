# Avalara E-Invoicing API Reference

## Table of Contents
1. [Base URLs](#base-urls)
2. [Authentication](#authentication)
3. [Headers](#headers)
4. [Document Operations](#document-operations)
5. [Mandate Operations](#mandate-operations)
6. [Trading Partner Operations](#trading-partner-operations)
7. [Error Codes](#error-codes)
8. [Code Examples](#code-examples)

## Base URLs

| Environment | Base URL |
|-------------|----------|
| Sandbox | `https://api.sbx.avalara.com/einvoicing` |
| Production | `https://api.avalara.com/einvoicing` |

## Authentication

### OAuth 2.0 Client Credentials Flow

**Token Endpoint (different host per environment; NOT under `/einvoicing`):**
- Sandbox: `https://ai-sbx.avlr.sh/connect/token`
- Production: `https://identity.avalara.com/connect/token`

**Request:**
```
POST {token_uri}
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials
&client_id={CLIENT_ID}
&client_secret={CLIENT_SECRET}
```

**Response:**
```json
{
  "access_token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJSUzI1NiIs...",
  "token_type": "Bearer",
  "expires_in": 3600
}
```

**Using the Token:**
```
Authorization: Bearer {access_token}
```

### Generating Client Credentials

1. Go to **Avalara Home > Integrations > License key and client secrets**
2. Under **Client key and secrets for E-Invoicing and Live Reporting**, select **Generate client key and secret**
3. Read the notification and acknowledge: "I understand that updating the access code deactivates the existing code and breaks the current connections"
4. Select **Continue**
5. Copy the **Key** (Client ID) and **Secret** (Client Secret)
6. **Important:** The Secret is only displayed once. Copy or download it before closing the window.

## Headers

| Header | Required | Description | Example |
|--------|----------|-------------|---------|
| `Authorization` | Yes | Bearer token | `Bearer eyJ0eXAi...` |
| `avalara-version` | Yes | API version | `1.6` |
| `X-Avalara-Client` | No | Client identifier for diagnostics | `MyApp/1.0` |
| `Content-Type` | Yes (for POST/PUT) | Content type | `application/json` |
| `Accept` | No | Response format | `application/json` or `application/xml` |

## Document Operations

### SubmitDocument

Submit an electronic document (invoice, credit note, etc.) for processing.

```
POST /documents
```

**Request Body:**
```json
{
  "documentType": "invoice",
  "mandateId": "IT-B2B-CLEARANCE",
  "documentBody": "<Base64-encoded UBL 2.1 XML>",
  "documentFormat": "application/xml",
  "countryCode": "IT",
  "currencyCode": "EUR",
  "totalAmount": 120.00,
  "taxTotal": 20.00,
  "issueDate": "2025-01-15",
  "dueDate": "2025-02-15",
  "supplier": {
    "name": "Your Company SRL",
    "vatNumber": "IT12345678901",
    "countryCode": "IT"
  },
  "customer": {
    "name": "Customer SpA",
    "vatNumber": "IT98765432109",
    "countryCode": "IT"
  }
}
```

**Response (202 Accepted):**
```json
{
  "documentId": "doc_abc123def456",
  "status": "Queued",
  "submissionDate": "2025-01-15T10:30:00Z",
  "estimatedCompletionTime": "2025-01-15T10:32:00Z"
}
```

**Official Docs:** [SubmitDocument](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Documents/SubmitDocument/)

### GetDocumentStatus

Check the processing status of a submitted document.

```
GET /documents/{documentId}/status
```

**Parameters:**
| Name | In | Required | Description |
|------|-----|----------|-------------|
| documentId | path | Yes | Document ID from SubmitDocument response |
| avalara-version | header | Yes | API version |

**Response (200) - real DocumentStatusResponse shape:**
```json
{
  "id": "52f60401-44d0-4667-ad47-4afe519abb53",
  "status": "Error",
  "businessStatus": "Rejected",
  "events": [
    {
      "eventDateTime": "2025-01-15T10:30:15",
      "message": "Schematron validation failed: [BR-FR-05] Buyer identifier (SIREN) is missing.",
      "category": "Validation",
      "responseKey": "authorityCode",
      "responseValue": "00404"
    }
  ]
}
```

- Error descriptions live in `events[].message` - parse those, there are no `validationMessages`/`processingSteps` fields.
- `businessStatus` = business lifecycle state from external actors (Tax Authority, PDP, ERP), e.g. acceptance or rejection.
- `StatusEvent` fields: `eventDateTime`, `message`, `responseKey` (type of acknowledgement returned by the tax authority), `responseValue`, `category` (process stage: processing, transmission, validation...).

**Status values are mandate-specific:**
The valid `status` values are defined per mandate in the `supportedDocumentStatuses` field of the `GET /mandates` response (e.g. "Approved", "Fully Paid"). Commonly seen values include `Queued`, `Processing`, `Completed`, `Sent`, `Received`, `Error`, `ValidationError`, `ConversionError`, `SendError`, `Submitted`, `Failed`. Do not hardcode a closed list; treat any status containing "error"/"fail"/"reject" as a failure.

**Official Docs:** [GetDocumentStatus](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Documents/GetDocumentStatus/)

### DownloadDocument

Retrieve the processed document in the requested format.

```
GET /documents/{documentId}/$download
```

**Headers:**
| Header | Description |
|--------|-------------|
| `Accept` | MIME type of the returned document: `application/pdf`, `application/xml`, `application/json` (available media types per mandate are listed in `getInvoiceAvailableMediaType` of `GET /mandates`) |

**Response:**
- Content-Type: Based on Accept header
- Body: Document content in requested format

**Errors:** `404` if the document/file has not been created yet (e.g. PDF unavailable while awaiting authority approval); `406` if the Accept format is unsupported for this mandate.

**Official Docs:** [DownloadDocument](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Documents/DownloadDocument/)

### GetDocumentList

Returns a summary of documents whose **processing date** falls within a date range.

```
GET /documents?startDate={start}&endDate={end}&$top={top}&$skip={skip}&$count=true&$include=events
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `startDate` | datetime | Start of range. Defaults to previous month. Format: `YYYY-MM-DDThh:mm:ss`. Filters by PROCESSING date, not document/issue date |
| `endDate` | datetime | End of range. Defaults to current date. Same format |
| `flow` | string | Optional direction filter: `out` = issued, `in` = received |
| `$count` | string | `true` = include total count of items in the response |
| `$countOnly` | string | `true` = return only the count |
| `$filter` | string | Filter by field name and value, e.g. `id eq 52f60401-...`. **Only `eq` is supported.** No `$orderby` exists - sort client-side |
| `$include` | string | `events` = each document embeds its `events` array (error descriptions!). Any other value/omitted = no events |
| `$top` | int | Page size |
| `$skip` | int | Items to skip |

**Response (200):** `{"@recordsetCount": "12", "@nextLink": "...", "value": [DocumentSummary, ...]}` (SDK exposes the @-keys as `atRecordsetCount`/`atNextLink`).

**DocumentSummary fields:** `id`, `companyId`, `processDateTime`, `status`, `businessStatus`, `supplierName`, `customerName`, `documentType`, `documentVersion`, `documentNumber`, `documentDate`, `flow`, `countryCode`, `countryMandate`, `interface`, `receiver`, `events` (only with `$include=events`), `createdAt`, `lastUpdatedAt`. **No amounts** - totals must be parsed from the downloaded XML.

**`documentType` is format-prefixed** (observed in production responses, as of Sep 2026): values arrive as `{dataFormat}-{docType}`, e.g. `ubl-invoice`, `ubl-creditnote`, `ubl-applicationresponse`, `fa_vat-invoice` (Poland KSeF), `fa_vat-creditnote`, or `xml-xml` for generic XML. To classify, strip non-alphanumerics, lowercase, and match the segment after the last `-` against `invoice`/`creditnote`/`debitnote`/`applicationresponse`; anything else = unknown.

**Tip:** wide date ranges + `$include=events` can make the server answer 502; query in day-chunks and retry with simpler parameters on 5xx.

### FetchDocuments (inbound pull)

Fetch an **inbound** document from a tax authority. Provide key-value pairs as request parameters; supported parameters vary by tax authority and country.

```
POST /documents/$fetch
Content-Type: application/json
```

**Response (200):** contains the platform `documentId` (for status checks and downloads), the returned status (e.g. `Accepted`), and `eventDateTime` when the document was accepted.

**Official Docs:** [Documents](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Documents/)

## Mandate Operations

### GetMandates

List all supported country mandates.

```
GET /mandates?$filter={filter}&$count={count}
```

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| `$filter` | string | Filter by field (supports `eq` and `contains`) |
| `$count` | boolean | Include total count |
| `$countOnly` | boolean | Return only count |

**Response (200):**
```json
{
  "value": [
    {
      "mandateId": "IT-B2B-CLEARANCE",
      "countryCode": "IT",
      "countryName": "Italy",
      "mandateName": "FatturaPA B2B Clearance",
      "documentTypes": ["invoice", "creditNote", "debitNote"],
      "complianceModel": "Clearance",
      "status": "Active",
      "effectiveDate": "2019-01-01",
      "supportedFormats": ["FatturaPA"]
    }
  ],
  "@odata.count": 45
}
```

**Official Docs:** [GetMandates](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Mandates/GetMandates/)

### GetMandateDataInputFields

Get required and optional fields for a specific mandate.

```
GET /mandates/{mandateId}/data-input-fields
```

**Response includes:**
- Field name, type, required/optional status
- Validation rules
- Default values
- Description

**Official Docs:** [GetMandateDataInputFields](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/Mandates/GetMandateDataInputFields/)

## Trading Partner Operations

### GetTradingPartners

Look up trading partners on exchange networks.

```
GET /trading-partners?countryCode={code}&identifier={id}
```

**Parameters:**
| Name | Description |
|------|-------------|
| countryCode | ISO country code |
| identifier | Tax ID or Peppol ID |
| network | Exchange network name |

**Official Docs:** [Trading Partners](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/TradingPartners/)

### GetTaxIdentifiers

Validate tax identifiers for trading partners.

```
GET /tax-identifiers?countryCode={code}&value={vat}
```

**Official Docs:** [Tax Identifiers](https://developer.avalara.com/api-reference/e-invoicing/v1.6/methods/TaxIdentifiers/)

## Error Codes

| HTTP Status | Code | Description |
|-------------|------|-------------|
| 400 | BadRequest | Invalid request format or missing required fields |
| 401 | Unauthorized | Invalid or expired access token |
| 403 | Forbidden | Insufficient permissions for this operation |
| 404 | NotFound | Document or resource not found |
| 409 | Conflict | Document already exists or state conflict |
| 422 | UnprocessableEntity | Validation failed (XSD/Schematron) |
| 429 | TooManyRequests | Rate limit exceeded |
| 500 | InternalServerError | Server-side error |
| 503 | ServiceUnavailable | Service temporarily unavailable |

## Code Examples

### Python - Submit and Check Status

```python
import requests
import base64

# Configuration
BASE_URL = "https://api.sbx.avalara.com/einvoicing"
CLIENT_ID = "your_client_id"
CLIENT_SECRET = "your_client_secret"

# 1. Get access token (token host is NOT under /einvoicing)
TOKEN_URL = "https://ai-sbx.avlr.sh/connect/token"   # sandbox
# TOKEN_URL = "https://identity.avalara.com/connect/token"   # production

def get_token():
    response = requests.post(
        TOKEN_URL,
        data={
            "grant_type": "client_credentials",
            "client_id": CLIENT_ID,
            "client_secret": CLIENT_SECRET
        }
    )
    return response.json()["access_token"]

# 2. Submit document
def submit_document(token, ubl_xml, mandate_id):
    headers = {
        "Authorization": f"Bearer {token}",
        "avalara-version": "1.6",
        "Content-Type": "application/json"
    }
    
    body = {
        "documentType": "invoice",
        "mandateId": mandate_id,
        "documentBody": base64.b64encode(ubl_xml.encode()).decode(),
        "documentFormat": "application/xml",
        "countryCode": "IT"
    }
    
    response = requests.post(
        f"{BASE_URL}/documents",
        headers=headers,
        json=body
    )
    return response.json()

# 3. Check status
def get_status(token, document_id):
    headers = {
        "Authorization": f"Bearer {token}",
        "avalara-version": "1.6"
    }
    
    response = requests.get(
        f"{BASE_URL}/documents/{document_id}/status",
        headers=headers
    )
    return response.json()

# Usage
token = get_token()
result = submit_document(token, ubl_xml_content, "IT-B2B-CLEARANCE")
doc_id = result["documentId"]
status = get_status(token, doc_id)
```

### cURL Examples

**Get Access Token:**
```bash
curl -X POST "{token_uri}" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id={CLIENT_ID}" \
  -d "client_secret={CLIENT_SECRET}"
```

**Submit Document:**
```bash
curl -X POST "https://api.sbx.avalara.com/einvoicing/documents" \
  -H "Authorization: Bearer {token}" \
  -H "avalara-version: 1.6" \
  -H "Content-Type: application/json" \
  -d '{
    "documentType": "invoice",
    "mandateId": "IT-B2B-CLEARANCE",
    "documentBody": "{base64_ubl_xml}",
    "documentFormat": "application/xml",
    "countryCode": "IT"
  }'
```

**Check Status:**
```bash
curl -g "https://api.sbx.avalara.com/einvoicing/documents/{documentId}/status" \
  -H "Authorization: Bearer {token}" \
  -H "avalara-version: 1.6"
```

**Download Document:**
```bash
curl -g "https://api.sbx.avalara.com/einvoicing/documents/{documentId}/$download" \
  -H "Authorization: Bearer {token}" \
  -H "avalara-version: 1.6" \
  -H "Accept: application/xml" \
  --output invoice.xml
```

**List Mandates:**
```bash
curl "https://api.sbx.avalara.com/einvoicing/mandates" \
  -H "Authorization: Bearer {token}" \
  -H "avalara-version: 1.6"
```

## API Version History

| Version | Status | Notes |
|---------|--------|-------|
| 1.6 | Current | Latest stable version |
| 1.5 | Supported | Previous version |
| 1.4 | Supported | Older version |
| 1.2 | Supported | Older version |
| 1.0 | Deprecated | Legacy support |

Always specify the `avalara-version` header to ensure consistent behavior.
