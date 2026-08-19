# M365 Copilot Forensics — Pilot Program Overview

**Status**: Approved for controlled pilot  
**Data Classification**: Use-case descriptions only; no customer content  
**Platform Support**: macOS, Windows (roadmap), Azure cloud  

---

## What This Is

This is a forensic evidence capture system for Microsoft Copilot interactions. When a Copilot-assisted task fails, this system:

1. **Captures** what happened (prompts, tool calls, results)
2. **Preserves** evidence with hash chains (tamper-evident)
3. **Redacts** customer data automatically
4. **Reconstructs** a timeline from evidence
5. **Attributes** the root cause with explicit gaps

**Important**: This is not a general logging system. It's a bounded black box that retains evidence only when invited, then is opened only during incident investigation.

---

## Data We Capture

✅ **Approved**:
- Use-case descriptions ("Generate Q3 sales summary")
- Tool names ("SharePoint", "Exchange", "Teams")
- Action status ("succeeded", "failed", "timeout")
- Error messages (redacted)
- Audit log references

❌ **Not Approved**:
- Customer document content
- Personal identifiable information (PII)
- Email body text
- Credentials, API keys, tokens
- Full file paths (reference only, with hash)

### Redaction Example

```
What Copilot did:  "Email sent to alice@acme.com with sales data"
What we capture:   "Email sent to <REDACTED-EMAIL> [hash: 5e1a3d8c...] with sales data"
If investigator needs full email: Requires audit approval to retrieve
```

---

## Architecture Overview

```
┌─ Your macOS Workstation ─────────────────┐
│ Copilot SDK + AIBlackBox Gateway          │
│ (local capture, no data leaves your Mac)  │
└────────────────────────────────────────── ┘
                     ↓ (on approval)
┌─ Azure Cloud (Your Subscription) ────────┐
│ PostgreSQL: Evidence + investigations     │
│ Blob Storage: Encrypted redacted payloads │
│ Key Vault: Secrets (passwords, M365 creds)│
│ ACI: Recorder & correlation worker        │
└────────────────────────────────────────── ┘
                     ↓
┌─ Web Dashboard (Browser) ─────────────────┐
│ Search investigations                     │
│ View timelines (evidence-backed)          │
│ Export reports (audit-safe)               │
│ Manage retention & access                 │
└────────────────────────────────────────── ┘
```

**Local capture** → **Cloud storage** → **Authenticated dashboard**

---

## Setup (12 Steps, ~30 minutes)

### Prerequisites
- macOS workstation
- Azure subscription ($150/month pilot budget)
- M365 tenant admin access
- Git + Node.js 18+

### Step 1: Azure Resource Group

```bash
az login
az group create --name claimtrace-pilot --location eastus
```

### Step 2: PostgreSQL Database

```bash
az postgres flexible-server create \
  --resource-group claimtrace-pilot \
  --name claimtrace-db \
  --admin-user claimtrace \
  --admin-password <PASSWORD> \
  --version 15 \
  --sku-name Standard_B1ms
```

### Step 3: Blob Storage

```bash
az storage account create \
  --resource-group claimtrace-pilot \
  --name claimtracestorage \
  --location eastus

az storage container create \
  --account-name claimtracestorage \
  --name evidence
```

### Step 4: Key Vault

```bash
az keyvault create \
  --resource-group claimtrace-pilot \
  --name claimtrace-kv

# Store secrets (DB password, storage key, M365 SP credentials)
az keyvault secret set --vault-name claimtrace-kv \
  --name DatabasePassword \
  --value <PASSWORD>
```

### Step 5: M365 Service Principal

1. **Azure Portal** → **Azure AD** → **App registrations** → **New registration**
2. Name: `ClaimTrace Evidence Collector`
3. **Certificates & secrets** → **New client secret** → Copy value
4. **API permissions** → **Office 365 Management APIs** → `ActivityFeed.Read` → **Grant admin consent**
5. Copy **Application (Client) ID** and **Directory (Tenant) ID**

### Step 6: Store M365 Credentials

```bash
az keyvault secret set --vault-name claimtrace-kv \
  --name M365TenantId \
  --value <TENANT_ID>

az keyvault secret set --vault-name claimtrace-kv \
  --name M365ClientId \
  --value <CLIENT_ID>

az keyvault secret set --vault-name claimtrace-kv \
  --name M365ClientSecret \
  --value <SECRET_VALUE>
```

### Step 7: Configuration File

Create `.blackbox.m365.json` on your Mac:

```json
{
  "captureDepth": "excerpt",
  "tool": "m365-copilot-gateway",
  "upstream": "https://api.openai.com/v1/chat/completions",
  "port": 8443,
  "maxBodyBytes": 50000,
  "m365": {
    "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "clientId": "yyyyyyyy-yyyy-yyyy-yyyy-yyyyyyyyyyyy",
    "clientSecret": "${M365_CLIENT_SECRET}"
  }
}
```

### Step 8: Environment Setup

```bash
export DATABASE_URL="postgresql://claimtrace:PASSWORD@claimtrace-db.postgres.database.azure.com:5432/investigations"
export STORAGE_KEY="<blob-storage-key>"
export M365_CLIENT_SECRET="<secret-value>"
```

### Step 9: Database Migration

```bash
git clone https://github.com/Shimicohen1/ClaimTraceAI-AIBlackBox.git
cd ClaimTraceAI-AIBlackBox
npm install

# Initialize schema (PostgreSQL)
npx tsx scripts/migrate-postgresql.ts
```

### Step 10: Start Gateway

```bash
npm run blackbox:gateway -- --config .blackbox.m365.json
```

Expected: `✅ AIBlackBox Gateway listening on https://localhost:8443`

### Step 11: Dashboard

```bash
npm run dev
# Visit: http://localhost:3000/dashboard
# Login with Azure AD
```

### Step 12: Test

Send a test Copilot request:

```bash
curl -X POST https://localhost:8443/v1/chat/completions \
  --header "Content-Type: application/json" \
  --header "x-aiblackbox-correlation-id: test-001" \
  --data '{"model":"gpt-4","messages":[{"role":"user","content":"Test"}]}'
```

Verify in dashboard: **Evidence** → Search by correlation ID `test-001`

---

## Using the Dashboard

### Create an Investigation

1. **+ New Investigation**
2. **Title**: "Q3 Sales Summary Failed"
3. **Detected at**: When you noticed the problem
4. **Correlation ID**: From gateway logs (optional)
5. **Create**

### View Timeline

Evidence-backed sequence of events:
- Copilot invocation (what was asked)
- Tool calls (SharePoint search, Exchange send, etc.)
- Tool results (what the system returned)
- Failure signal (if captured)

### Analyze

Deterministic attribution:
- ✅ **AI_CAUSED**: Copilot action directly caused failure
- 🟡 **AI_CONTRIBUTED**: Copilot was a contributing factor
- ❌ **NON_AI_ROOT_CAUSE**: Root cause is elsewhere
- ❓ **INSUFFICIENT_EVIDENCE**: Missing proof

### Export Report

Generate audit-safe reports (HTML, PDF, JSON):
- Investigation summary
- Evidence-backed timeline
- Attribution verdict with evidence references
- Redaction summary
- Access audit log

---

## Cost Estimate

| Service | Capacity | Cost/month |
|---------|----------|-----------|
| PostgreSQL Flexible Server | 1 vCore, 32 GB | $30–40 |
| Blob Storage (Hot) | 100 GB | $20–25 |
| Container Instances | On-demand, business hours | $40–50 |
| Key Vault + VNet | Minimal pilot | $10–15 |
| **Total** | | **~$110–145/month** |

Fits within $150/month Azure free credits for new subscriptions.

---

## Data Retention

| Type | Retention | Rationale |
|------|-----------|-----------|
| Raw payloads (Blob) | 90 days | Auto-expire |
| Evidence records (DB) | 1 year | Allow historical investigation |
| Investigation cases | Indefinite | Compliance + audit trail |
| Case reports (frozen) | Indefinite | Legal requirement |

---

## Onboarding Checklist

- [ ] Azure subscription created and funded
- [ ] Resource group, PostgreSQL, Blob Storage, Key Vault provisioned
- [ ] M365 service principal registered and consented
- [ ] All secrets stored in Key Vault
- [ ] `.blackbox.m365.json` configuration created
- [ ] Environment variables set on macOS
- [ ] Database migration completed (`npx tsx scripts/migrate-postgresql.ts`)
- [ ] Gateway running locally (`npm run blackbox:gateway ...`)
- [ ] Dashboard accessible (`npm run dev` → localhost:3000/dashboard)
- [ ] Test Copilot capture verified
- [ ] First investigation created and timeline reviewed
- [ ] Report export tested

---

## FAQ

**Q: Where does my data go?**  
A: Locally on your Mac during capture (gateway). On approval, encrypted and sent to Azure. Dashboard accesses via authenticated API.

**Q: Can you see my customer data?**  
A: No. Automatic redaction removes PII, customer identifiers, and sensitive content. Hashes are retained for correlation.

**Q: How long is data kept?**  
A: Raw payloads auto-expire at 90 days. Investigation cases are retained indefinitely for compliance.

**Q: Can I delete an investigation?**  
A: Yes, with audit logging. All access to sensitive records is logged.

**Q: What if you find a bug?**  
A: The hash chain proves if records were tampered with. Evidence is immutable; notes and conclusions are separate.

**Q: Can I use this on Windows?**  
A: Roadmap, not pilot. Current implementation is macOS only.

---

## Support

- **Setup help**: See [M365_ONBOARDING_GUIDE.md](./docs/M365_ONBOARDING_GUIDE.md) (detailed steps)
- **Architecture**: See this document (architecture overview)
- **API reference**: Available at `/api/v1` once dashboard is running
- **Questions**: Open an issue or contact the team

---

**Pilot Status**: ✅ Ready for testing  
**Last Updated**: 2026-08-19  
**License**: Private (implementation) + Public (documentation only)
