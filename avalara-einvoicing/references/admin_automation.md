# Admin Automation Tasks and Workflows

## Table of Contents
1. [Daily Admin Tasks](#daily-admin-tasks)
2. [Webhook Setup Guide](#webhook-setup-guide)
3. [Mandate Activation Workflow](#mandate-activation-workflow)
4. [ERP Integration Checklist](#erp-integration-checklist)
5. [User Management](#user-management)
6. [Monitoring and Alerting](#monitoring-and-alerting)
7. [Troubleshooting Playbooks](#troubleshooting-playbooks)
8. [Automation Scripts](#automation-scripts)

## Daily Admin Tasks

### Invoice Monitoring Checklist
```
[ ] Check Avalara Cockpit dashboard for failed invoices
[ ] Review webhook delivery status
[ ] Verify no stuck documents in "Processing" > 2 hours
[ ] Check for mandate updates or new field requirements
[ ] Review trading partner lookup failures
[ ] Validate tax identifier rejections
```

### Health Checks
| Check | Command/Location | Expected Result |
|-------|-----------------|----------------|
| API connectivity | `GET /mandates` | 200 OK with mandate list |
| Authentication | Token refresh | New token within 5s |
| Webhook status | Cockpit > Integrations | Active, last delivery < 1 hour |
| Mandate validity | Cockpit > Mandates | All active mandates "Valid" |
| Document queue depth | Cockpit > Monitoring | < 100 queued documents |

## Webhook Setup Guide

### Step 1: Prepare Your Endpoint
Create an HTTPS endpoint that accepts POST requests:
```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)

@app.route('/webhook/avalara', methods=['POST'])
def handle_webhook():
    # Verify signature
    signature = request.headers.get('X-Avalara-Signature')
    payload = request.get_data()
    
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(signature, expected):
        return 'Invalid signature', 401
    
    event = request.json
    
    # Handle event types
    if event['eventType'] == 'Document.StatusChanged':
        doc_id = event['data']['documentId']
        new_status = event['data']['status']
        # Update your system
    
    return 'OK', 200
```

### Step 2: Configure in Avalara Cockpit
1. Go to **Avalara Home > Integrations**
2. Navigate to **Webhooks** section
3. Click **Add webhook**
4. Enter your endpoint URL: `https://your-api.com/webhook/avalara`
5. Generate and copy the **Signature Key** (for payload verification)
6. Select event types to subscribe to:
   - `Document.StatusChanged` - Status changes
   - `Document.ValidationFailed` - Validation errors
   - `Document.Received` - Incoming documents
   - `Mandate.Updated` - Mandate configuration changes
7. Set webhook to **Active**
8. Save and test

**Guide:** [Add webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/ruy1708406539000.html)

### Step 3: Update Webhook
1. Go to **Integrations > Webhooks**
2. Find the webhook and click **Edit**
3. Modify URL, events, or status
4. Save changes

**Guide:** [Update webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/wnx1708406594984.html)

### Step 4: Delete Webhook
1. Go to **Integrations > Webhooks**
2. Find the webhook and click **Delete**
3. Confirm deletion

**Guide:** [Delete webhook](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/rpk1708426744674.html)

### Webhook Event Types

| Event Type | Description | Payload Fields |
|------------|-------------|----------------|
| `Document.StatusChanged` | Document status updated | documentId, status, previousStatus, timestamp |
| `Document.ValidationFailed` | Document failed validation | documentId, errors[], errorCode |
| `Document.Received` | Incoming document received | documentId, sender, format, timestamp |
| `Document.Completed` | Document processing complete | documentId, authorityReference, qrCode |
| `Mandate.Updated` | Mandate configuration changed | mandateId, changes[], effectiveDate |
| `TradingPartner.Discovered` | New trading partner found | partnerId, network, identifier |

### Webhook Retry Policy
- First attempt: Immediate
- Retry 1: 30 seconds
- Retry 2: 2 minutes
- Retry 3: 10 minutes
- Max retries: 3
- After max retries: Webhook marked as failed, alert sent

## Mandate Activation Workflow

### New Country/Mandate Setup

```
Step 1: Log into Avalara Cockpit
   |
Step 2: Navigate to E-Invoicing Activations
   |
Step 3: Select country from available list
   |
Step 4: Choose mandate type (B2B, B2G, B2C)
   |
Step 5: Enter legal entity details
   - Company registration number
   - VAT/Tax ID
   - Legal address
   - Authorized signatory
   |
Step 6: Submit activation request
   |
Step 7: Avalara reviews (1-3 business days)
   |
Step 8: Receive activation confirmation
   |
Step 9: Configure ERP mapping templates
   |
Step 10: Validate test invoice
   |
Step 11: Go live
```

### Mandatory Fields Checklist
Before activating a mandate, ensure you have:
- [ ] Valid tax registration for the target country
- [ ] Legal entity details match tax authority records
- [ ] ERP data mapping configured for required fields
- [ ] Test environment access (for validation)
- [ ] Webhook endpoint configured for status notifications

### Updating Mandate Configuration
When Avalara updates a mandate:
1. Navigate to **Cockpit > Mandates**
2. Check notification for "Update Available"
3. Review changes in update notes
4. Click **Update** to synchronize
5. Review and update field mappings if new fields added
6. Re-validate test documents
7. Confirm production readiness

**Guide:** [Update mapping](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vgv1719490070082.html)

## ERP Integration Checklist

### Pre-Integration
- [ ] Confirm Avalara ELR subscription includes target countries
- [ ] Activate all required country mandates in Cockpit
- [ ] Generate API Client ID and Secret
- [ ] Identify ERP integration method (native/middleware/API)
- [ ] Map ERP invoice fields to mandate required fields

### Integration Phase
- [ ] Configure sandbox connection
- [ ] Test authentication (token generation)
- [ ] Submit test invoice to sandbox
- [ ] Verify status retrieval
- [ ] Download processed document
- [ ] Test error scenarios (invalid data, missing fields)
- [ ] Configure webhook endpoint
- [ ] Verify webhook delivery and signature verification
- [ ] Test cancellation workflow (if supported by mandate)

### Go-Live
- [ ] Switch to production credentials
- [ ] Submit first production invoice (low volume)
- [ ] Monitor status and confirm delivery
- [ ] Enable automated processing
- [ ] Set up monitoring alerts

### Post-Go-Live
- [ ] Daily health checks
- [ ] Weekly review of failed invoices
- [ ] Monthly audit trail review
- [ ] Quarterly mandate update check

### Field Mapping Template
```json
{
  "mandateId": "IT-B2B-CLEARANCE",
  "mappings": {
    "invoiceNumber": "ERP.invoice_num",
    "issueDate": "ERP.invoice_date",
    "supplierName": "ERP.company_name",
    "supplierVat": "ERP.company_vat",
    "customerName": "ERP.customer_name",
    "customerVat": "ERP.customer_tax_id",
    "totalAmount": "ERP.total_amount",
    "taxTotal": "ERP.tax_amount",
    "currency": "ERP.currency_code",
    "lineItems": {
      "description": "ERP.line_description",
      "quantity": "ERP.line_quantity",
      "unitPrice": "ERP.line_unit_price",
      "lineTotal": "ERP.line_amount",
      "taxRate": "ERP.line_tax_rate"
    }
  }
}
```

**Mapping Guides:**
- [Export mapping for inbound](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/tvh1721909167973.html)
- [Export mapping for outbound](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/zay1721909243761.html)
- [Update mapping for inbound](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vlj1719400038829.html)
- [Update mapping for outbound](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vgv1719490070082.html)

## User Management

### Creating Users
1. Go to **Cockpit > User Management**
2. Click **Add User**
3. Enter user details (name, email)
4. Assign role:
   - **Org Admin:** Full access (recommended max 2-3 users)
   - **Standard User:** View/monitor only
5. Send invitation

### Role Responsibilities
| Task | Org Admin | Standard User |
|------|-----------|---------------|
| Activate mandates | Yes | No |
| Manage webhooks | Yes | No |
| Generate API keys | Yes | No |
| Manage users | Yes | No |
| View invoices | Yes | Yes |
| Check document status | Yes | Yes |
| Export reports | Yes | Yes |
| Configure connectors | Yes | No |

**Details:** [User types and roles](https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/iba1708427593315.html)

## Monitoring and Alerting

### Key Metrics to Monitor
| Metric | Alert Threshold | Action |
|--------|----------------|--------|
| Failed invoices/hour | > 5 | Check validation errors |
| Average processing time | > 5 minutes | Check API status |
| Webhook delivery failures | > 2 consecutive | Check endpoint health |
| Queue depth | > 100 | Scale processing or contact support |
| Authentication failures | > 3/hour | Check token refresh logic |
| Mandate validation errors | > 10/day | Review field mappings |

### Setting Up Alerts in Cockpit
1. Navigate to **Monitoring > Alerts**
2. Click **Create Alert Rule**
3. Select metric and threshold
4. Configure notification channels:
   - Email
   - Webhook (to your incident management)
5. Set severity level
6. Save and enable

### Audit Trail Review
The audit trail tracks all actions on both inbound and outbound invoices:
- Document submission timestamp
- Status changes with timestamps
- Authority references
- Validation results
- Delivery confirmations
- User actions (manual interventions)

Access via: **Cockpit > Audit Trail**

## Troubleshooting Playbooks

### Issue: Invoices Stuck in "Queued"
1. Check API connectivity: `GET /mandates`
2. Verify authentication token not expired
3. Check mandate status in Cockpit (not disabled)
4. Review if maintenance window active
5. Contact Avalara support if > 30 minutes

### Issue: Validation Errors
1. Retrieve specific errors: `GET /documents/{id}/status`
2. Cross-check with mandate data-input-fields
3. Verify all required fields present in UBL
4. Validate UBL XML against schema
5. Update field mappings in ERP
6. Resubmit after fixes

### Issue: Webhook Not Received
1. Check webhook status in Cockpit (Active?)
2. Verify endpoint URL accessible from internet
3. Check webhook logs in Cockpit
4. Verify SSL certificate valid
5. Test endpoint with curl from external server
6. Review server firewall rules

### Issue: 401 Unauthorized
1. Check token expiration time
2. Verify Client ID not revoked
3. Regenerate Client Secret if needed:
   - Cockpit > Integrations > License key and client secrets
   - Generate new key (invalidates old one!)
   - Update all integrations with new secret
4. Check environment (sandbox vs production)

### Issue: Mandate Not Available
1. Confirm country activated in Cockpit
2. Check if mandate requires additional verification
3. Verify subscription includes target country
4. Contact Avalara to add country if not in list

## Automation Scripts

### Python: Batch Invoice Status Check
```python
import requests
import csv
from datetime import datetime, timedelta

BASE_URL = "https://api.avalara.com/einvoicing"
CLIENT_ID = "your_id"
CLIENT_SECRET = "your_secret"

# Token host is NOT under /einvoicing and differs per environment:
TOKEN_URL = "https://identity.avalara.com/connect/token"       # production
# TOKEN_URL = "https://ai-sbx.avlr.sh/connect/token"          # sandbox

def get_token():
    resp = requests.post(TOKEN_URL, data={
        "grant_type": "client_credentials",
        "client_id": CLIENT_ID,
        "client_secret": CLIENT_SECRET
    })
    return resp.json()["access_token"]

def check_batch_status(document_ids):
    token = get_token()
    headers = {
        "Authorization": f"Bearer {token}",
        "avalara-version": "1.6"
    }
    
    results = []
    for doc_id in document_ids:
        resp = requests.get(
            f"{BASE_URL}/documents/{doc_id}/status",
            headers=headers
        )
        data = resp.json()
        results.append({
            "documentId": doc_id,
            "status": data.get("status"),
            "timestamp": datetime.now().isoformat()
        })
    
    return results

# Export to CSV
results = check_batch_status(["doc1", "doc2", "doc3"])
with open("invoice_status.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["documentId", "status", "timestamp"])
    writer.writeheader()
    writer.writerows(results)
```

### Python: Webhook Signature Verification
```python
import hmac
import hashlib
import base64

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    """Verify Avalara webhook signature."""
    expected = base64.b64encode(
        hmac.new(
            secret.encode("utf-8"),
            payload,
            hashlib.sha256
        ).digest()
    ).decode("utf-8")
    return hmac.compare_digest(f"sha256={expected}", signature)
```

### Python: Mandate Health Check
```python
def mandate_health_check():
    token = get_token()
    headers = {"Authorization": f"Bearer {token}", "avalara-version": "1.6"}
    
    # Get all mandates
    resp = requests.get(f"{BASE_URL}/mandates", headers=headers)
    mandates = resp.json().get("value", [])
    
    # Check document status for each
    health_report = []
    for m in mandates:
        mandate_id = m["mandateId"]
        country = m["countryCode"]
        status = m.get("status", "Unknown")
        
        health_report.append({
            "mandate": mandate_id,
            "country": country,
            "status": status,
            "healthy": status == "Active"
        })
    
    return health_report
```

## Quick Reference Links

| Topic | URL |
|-------|-----|
| E-Invoicing Overview | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/cis1680268965091.html |
| Generate API Credentials | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/uev1682056823407.html |
| Webhooks | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/dfc1708406514769.html |
| Add Webhook | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/ruy1708406539000.html |
| Update Webhook | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/wnx1708406594984.html |
| Delete Webhook | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/rpk1708426744674.html |
| Connector Settings | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vrq1740135240603.html |
| User Roles | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/iba1708427593315.html |
| Export Inbound Mapping | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/tvh1721909167973.html |
| Export Outbound Mapping | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/zay1721909243761.html |
| Update Inbound Mapping | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vlj1719400038829.html |
| Update Outbound Mapping | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/vgv1719490070082.html |
| Update Execution Rules | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/dsw1719400057015.html |
| Disable Mandate | https://knowledge.avalara.com/bundle/nsz1680268923536_nsz1680268923536/page/glf1719400091828.html |
| E-Invoicing API v1.6 | https://developer.avalara.com/api-reference/e-invoicing/v1.6/ |
| E-Invoicing Demo | https://www.youtube.com/watch?v=ZL-Fpy9XajU |
| Partner API Info | https://www.avalara.com/blog/en/north-america/2023/05/e-invoicing-easier-avalara-partner-api.html |
| Product Page | https://www.avalara.com/us/en/products/e-invoicing.html |
| AvaTax API Explorer | https://rest.avatax.com/ |
| Developer Community | https://developer.avalara.com/developer-community/ |
