---
name: avalara-einvoicing
description: Comprehensive guide for Avalara E-Invoicing and Live Reporting (ELR) platform administration, API integration, mandate management, and automation. Use when working with Avalara e-invoicing tasks including admin portal configuration, mandate activation, webhook setup, API operations (SubmitDocument, GetDocumentStatus, DownloadDocument, CancelDocument), ERP integrations (SAP, Oracle, NetSuite, Workday), troubleshooting invoice lifecycle issues, managing trading partners, country mandate compliance, UBL/XML transformations, or any Avalara E-Invoicing automation. Covers sandbox and production environments, authentication, user roles, audit trails, and real-time reporting mandates.
---

# Avalara E-Invoicing and Live Reporting (ELR)

## Overview

Avalara E-Invoicing and Live Reporting (ELR) is a global platform for electronic invoicing compliance. It converts UBL 2.1 data to country-specific formats (FatturaPA, FA-VAT, Peppol BIS 3.0, PINT, etc.), validates invoices via XSD/Schematron checks, routes them to tax authorities or exchange networks (Peppol, DBNA, local networks), and provides real-time monitoring with full audit trails.

**Key Capabilities:**
- Automated creation and transmission of e-invoices in mandate-specific formats
- Pre-validation of invoice content (XSD/Schematron) before authority submission
- Digital signatures, QR codes, and PDF generation
- Receipt and conversion of incoming e-invoices into unified XML
- Self-service mandate activation without development
- Real-time invoice monitoring portal with audit trail
- ISO 27001 and SOC 2 Type II certified multi-tenant cloud infrastructure

## Getting Started

### Environments

| Environment | Base URL | Purpose |
|-------------|----------|---------|
| Sandbox | `https://api.sbx.avalara.com/einvoicing` | Development and testing |
| Production | `https://api.avalara.com/einvoicing` | Live operations |

### Authentication

OAuth 2.0 Client Credentials flow. Generate credentials in the Avalara Cockpit:

1. Navigate to **Avalara Home > Integrations > License key and client secrets**
2. Under **Client key and secrets for E-Invoicing and Live Reporting**, select **Generate client key and secret**
3. Acknowledge that updating the access code deactivates existing connections
4. Copy the **Key** (Client ID) and **Secret** (Client Secret) immediately - the Secret is only shown once

**Token endpoints (per environment - different hosts, NOT under /einvoicing):**

| Environment | Token URL |
|-------------|-----------|
| Sandbox | `https://ai-sbx.avlr.sh/connect/token` |
| Production | `https://identity.avalara.com/connect/token` |

**Token Request:**
```
POST {token_uri}
Content-Type: application/x-www-form-urlencoded

grant_type=client_credentials&client_id={CLIENT_ID}&client_secret={CLIENT_SECRET}
```

Use the returned access token as `Authorization: Bearer {token}` in all API calls. The `avalara-version` header (e.g., "1.6") is required on all requests.

**Official Docs:** [Generate client key and secret](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/uev1682056823407.html)

### Required Headers

| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token from OAuth |
| `avalara-version` | Yes | API version (e.g., "1.6") |
| `X-Avalara-Client` | No | Client identifier for diagnostics |
| `Content-Type` | Yes | `application/json` |

## API Operations

### Core Document Workflow

1. **SubmitDocument** - Submit an invoice/credit note/document for processing
2. **GetDocumentStatus** - Check processing status using the returned documentId
3. **DownloadDocument** - Retrieve the processed document as XML or PDF
4. **CancelDocument** - Submit a cancellation request (where supported by mandate)

**Official API Reference:** [E-Invoicing API v1.6](https://developer.avalara.com/api-reference/e-invoicing/v1.6/)

### API Endpoints

| Operation | Method | Endpoint |
|-----------|--------|----------|
| SubmitDocument | POST | `/documents` |
| GetDocumentStatus | GET | `/documents/{documentId}/status` |
| DownloadDocument | GET | `/documents/{documentId}/$download` |
| GetDocumentList | GET | `/documents` |
| FetchDocuments (inbound pull from tax authority) | POST | `/documents/$fetch` |
| GetMandates | GET | `/mandates` |
| GetMandateDataInputFields | GET | `/mandates/{mandateId}/data-input-fields` |
| GetTradingPartners | GET | `/trading-partners` |
| GetTaxIdentifiers | GET | `/tax-identifiers` |

### MCP Server

Avalara provides an MCP server for AI-assisted access:
```json
{
  "mcpServers": {
    "avalara-elr": {
      "type": "http",
      "url": "https://mcp.avalara.com/elr"
    }
  }
}
```

Tools: `get_document_status`, `get_document`

**Details:** [E-Invoicing MCP Server](https://developer.avalara.com/mcp-servers/E-Invoicing/)

## Admin Portal (Cockpit) Configuration

### Company Setup
- Navigate to **Avalara Home > E-Invoicing Activations**
- Create and configure your primary company
- Enter required legal entity and registration details per country
- Activate country mandates

### Mandate Management
1. **Activate countries** in the Avalara Cockpit for your target markets
2. **Select mandates** for each country (CLEARANCE, PRE-CLEARANCE, REAL-TIME REPORTING, etc.)
3. **Self-service activation** - no development required to add new mandates
4. **Validate** mandates before use to ensure field mappings are complete

**Official Guide:** [Mandates in ELR](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/cis1680268965091.html)

### Webhook Configuration

Webhooks notify your servers when transaction status changes occur.

1. Go to **Avalara Home > Integrations**
2. Add webhook URL in the Integration page (Org Admin only)
3. Generate a **Signature Key** for verifying webhook payloads
4. Set webhook to active or inactive as needed

**Webhook Management:**
- **Add webhook:** [Guide](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/ruy1708406539000.html)
- **Update webhook:** [Guide](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/wnx1708406594984.html)
- **Delete webhook:** [Guide](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/rpk1708426744674.html)
- **Webhooks overview:** [Docs](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/dfc1708406514769.html)

### User Roles

| Role | Permissions |
|------|-------------|
| Organization Administrator (Org Admin) | Full access: user management, mandate activation, webhook configuration, integration settings, company setup |
| Standard User | View and monitor invoices, check document status, limited configuration access |

**Details:** [User types and roles](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/iba1708427593315.html)

### Connector Settings
- Manage ERP connector configurations
- Update endpoint URLs and credentials
- Test connectivity to business systems

**Guide:** [Manage connector settings](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vrq1740135240603.html)

## Invoice Lifecycle & Status Tracking

### Status Flow
```
Submitted → Queued → Processing → [Completed / Error]
                                    ↓
                              [Sent / Received]
                                    ↓
                               [Archived]
```

### Key Statuses

| Status | Description |
|--------|-------------|
| Queued | Document waiting to be processed |
| Processing | Document being validated and transformed |
| Completed | Successfully processed, document available for download |
| Sent | Successfully delivered to recipient/tax authority |
| Received | Incoming document received from trading partner |
| Error | Processing failed - check error details |
| Validation Error | Invoice content failed XSD/Schematron validation |
| Conversion Error | Format transformation failed |
| Send Error | Delivery to recipient failed |

**Note:** The exact status vocabulary is mandate-specific — read the `supportedDocumentStatuses` field in the `GET /mandates` response for the definitive list per mandate, and treat any status containing "error"/"fail"/"reject" as a failure. `DocumentStatusResponse` and `DocumentSummary` also carry a `businessStatus` field (tax authority / PDP acceptance or rejection).

### Monitoring
- Use the **Avalara Cockpit** portal for real-time monitoring
- Check document status via API: `GET /documents/{documentId}/status`
- Set up webhooks for automated status notifications
- Review audit trail for all inbound and outbound invoice actions

## Mandate Types by Country

### European Mandates (2026-2027)
| Country | Mandate | Format | Model | Live Date |
|---------|---------|--------|-------|-----------|
| Belgium | B2B e-invoicing | UBL 2.1 / Peppol BIS 3.0 | Peppol (Decentralized) | Jan 2026 |
| France | B2B e-facture | UBL, CII, Factur-X | PDP / Chorus Pro | Sep 2026 |
| Germany | B2B XRechnung | UBL, CII | Peppol / Direct | Jan 2027 (Wave 1) |
| Italy | FatturaPA | XML FatturaPA | SDI Clearance | Live |
| Poland | KSeF | FA(3) XML | Centralized (KSeF) | Feb 2026 |
| Spain | VERI*FACTU | UBL, Facturae, EDIFACT | AEAT / Private platforms | Jan 2027 |
| Hungary | RTIR | XML | Real-time Reporting | Live |

### Asia-Pacific Mandates
| Country | Format | Model | Status |
|---------|--------|-------|--------|
| India | GST JSON | Real-time (IRP/IRN) | Live |
| Malaysia | UBL 2.1 XML / JSON | MyInvois Clearance | Live (Phase 4) |
| Singapore | Peppol BIS / PINT-SG | Peppol (InvoiceNow) | Phased rollout |
| Australia | Peppol PINT A-NZ | Peppol | Voluntary (growing) |
| New Zealand | Peppol PINT A-NZ | Peppol | Growing adoption |

### Americas
| Country | Format | Model | Status |
|---------|--------|-------|--------|
| Mexico | CFDI 4.0 | PAC Clearance (SAT) | Live |
| Brazil | NF-e / NFS-e XML | SEFAZ Clearance | Live |
| Colombia | UBL 2.1 | DIAN Clearance | Live |
| Chile | XML | Real-time Reporting | Live |

### Middle East & Africa
| Country | Format | Model | Status |
|---------|--------|-------|--------|
| UAE | XML/UBL 2.1 (PINT-AE) | DCTCE 5-Corner (Peppol) | Jan 2027 (Wave 1) |
| Saudi Arabia | XML ZATCA | ZATCA Fatoora Clearance | Live |
| Egypt | XML | ETA Platform | Live |

**Full mandate tracker:** [EY E-invoicing developments tracker](https://www.ey.com/content/dam/ey-unified-site/ey-com/en-gl/technical/tax-guides/documents/en-gl-einvoicing-developments-tracker.pdf)

## ERP Integration Patterns

### Integration Approaches

1. **Native ERP Module** - Built-in e-invoicing capability (SAP DRC, Oracle CMK, Dynamics Globalization Studio, NetSuite SuiteApp)
2. **Middleware/Integration Layer** - Separate service between ERP and Avalara (for legacy/multi-ERP setups)
3. **Certified Access Point** - Direct regulated connection (Peppol Access Point, PDP, ASP)

### Supported ERP Systems

| ERP | Integration Method | Notes |
|-----|-------------------|-------|
| SAP S/4HANA & ECC | IDOC/BAPI or Service Layer | SAP Document and Reporting Compliance (DRC) |
| Oracle Cloud/Fusion | REST API / CMK | Collaboration Messaging Framework |
| NetSuite | SuiteApp / SuiteTalk | Electronic Invoicing SuiteApp via OBN |
| Microsoft Dynamics 365 | Power Platform / API | Electronic Invoicing service via Globalization Studio |
| Workday | Studio / REST API | Workday Financials integration |
| QuickBooks | Middleware / API | Lightweight connector |
| Sage | API connector | Format conversion |

**Product Page:** [Avalara E-Invoicing](https://www.avalara.com/us/en/products/e-invoicing.html)

### UBL 2.1 Standard Format

Avalara ELR uses **UBL 2.1** (Universal Business Language) as the intermediate format:
- Submit documents in UBL 2.1 XML format
- Avalara converts to country-specific formats automatically
- Receive incoming invoices converted back to unified UBL XML

**Structure:** Header (BG-1 to BG-4) > Parties (BG-5 to BG-11) > Lines (BG-25+)

## SDKs and Developer Tools

| Language | SDK Repository | Package |
|----------|---------------|---------|
| Python | [GitHub](https://github.com/avadev/AvaTax-REST-V2-Python-SDK) | `pip install Avalara` |
| C# / .NET | [GitHub](https://github.com/avadev/AvaTax-REST-V2-DotNet-SDK) | NuGet |
| Java | [GitHub](https://github.com/avadev/AvaTax-REST-V2-JRE-SDK) | Maven/Gradle |
| JavaScript/TypeScript | [GitHub](https://github.com/avadev/AvaTax-REST-V2-JS-SDK) | npm |
| PHP | [GitHub](https://github.com/avadev/AvaTax-SDK-PHP) | Composer |
| Ruby | [GitHub](https://github.com/avadev/AvaTax-SDK-Ruby) | Gem |

**Developer Portal:** [developer.avalara.com](https://developer.avalara.com)
**AvaTax API Reference:** [API Explorer](https://rest.avatax.com/)
**Community Forums:** [Avalara Developer Community](https://developer.avalara.com/developer-community/)

## Troubleshooting

### Common Issues

| Issue | Cause | Resolution |
|-------|-------|------------|
| 401 Unauthorized | Expired or invalid token | Regenerate client key and secret in Cockpit |
| 403 Forbidden | Insufficient permissions | Check user role; requires Org Admin for some operations |
| Validation Error | Missing required fields | Check mandate data-input-fields; ensure all required fields mapped |
| Conversion Error | Invalid UBL structure | Validate UBL 2.1 XML against schema before submission |
| Send Error | Network/authority issue | Check webhook notifications; retry after resolving connectivity |
| Mandate not available | Country not activated | Activate country in Avalara Cockpit first |
| Webhook not firing | URL or signature mismatch | Verify Notification URL and Signature Key match between systems |

### Diagnostic Steps
1. **Test connectivity:** Ping the API with `GET /mandates` using valid credentials
2. **Check mandate status:** Verify the country mandate is active in Cockpit
3. **Validate document:** Ensure all required fields for the target mandate are populated
4. **Review webhook logs:** Check for notification delivery failures
5. **Check audit trail:** Use Cockpit portal to trace full invoice lifecycle

### Key Knowledge Center Links

| Topic | Link |
|-------|------|
| E-Invoicing Overview | [knowledge.avalara.com](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/cis1680268965091.html) |
| Generate API Credentials | [Generate client key](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/uev1682056823407.html) |
| Webhooks | [Webhooks guide](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/dfc1708406514769.html) |
| Add Webhook | [Add webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/ruy1708406539000.html) |
| Update Webhook | [Update webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/wnx1708406594984.html) |
| Delete Webhook | [Delete webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/rpk1708426744674.html) |
| Connector Settings | [Manage connectors](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vrq1740135240603.html) |
| User Roles | [User types and roles](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/iba1708427593315.html) |
| Export Mapping (Inbound) | [Export mapping](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/tvh1721909167973.html) |
| Export Mapping (Outbound) | [Export mapping](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/zay1721909243761.html) |
| Update Mapping (Inbound) | [Update mapping](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vlj1719400038829.html) |
| Update Mapping (Outbound) | [Update mapping](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vgv1719490070082.html) |
| Update Execution Rules | [Execution rules](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/dsw1719400057015.html) |
| Disable Mandate | [Disable mandate](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/glf1719400091828.html) |
| E-Invoicing API Ref | [developer.avalara.com](https://developer.avalara.com/api-reference/e-invoicing/v1.6/) |
| E-Invoicing Demo | [YouTube Demo](https://www.youtube.com/watch?v=ZL-Fpy9XajU) |
| Partner API Blog | [Partner API article](https://www.avalara.com/blog/en/north-america/2023/05/e-invoicing-easier-avalara-partner-api.html) |

## Reference Files

- **Detailed API reference:** See [references/api_reference.md](references/api_reference.md) for full endpoint details, request/response schemas, and code examples
- **Country mandate guide:** See [references/country_mandates.md](references/country_mandates.md) for mandate-specific configuration details by country - including France's Chorus Pro annuaire bulk lookup (mass consultation, 5000 SIREN/SIRET per file) for checking which customers are registered
- **Admin automation scripts:** See [references/admin_automation.md](references/admin_automation.md) for common admin tasks and automation workflows
