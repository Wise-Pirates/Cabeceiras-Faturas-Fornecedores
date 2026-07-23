# Cabeceiras — Sistema de Faturação: Intelligence Audit

> **Date**: 2026-07-23
> **Author**: Andie (via Flow OS)
> **Collaborator**: Nathi (creating shared repo on Wise Pirates account)
> **Original Developer**: Robson Advincula (on sick leave)
> **Client**: Cabeceiras.pt — Jorge Brandão Gonçalves (brandao@cabeceiras.pt)
> **Status**: READ-ONLY INTEL GATHERING — no changes made

---

## 1. Executive Summary

The "Sistema de Faturação" at Cabeceiras is a complex multi-component automation system built by Robson Advincula over ~5 months (March–July 2026). The system automates invoice processing from supplier PDFs into Business Central (BC) ERP. The client reports it's "not working" — the most likely cause is that the main n8n workflow was **DESATIVADO (deactivated)** at the end of the last working session (July 15, 2026) and was never reactivated, pending Jorge's validation of 13 test ECs.

Additionally, the n8n API key appears to be expired/invalid — the API returns 0 workflows despite the instance being healthy (200 OK on /healthz).

---

## 2. Sources Consulted

| Source | What We Got |
|--------|-------------|
| **ClickUp** (Task 869dvuvec) | Full task history, 6+ detailed comment reports from Robson (June 30 – July 8, 2026) |
| **ClickUp** (Task 869dxb5ux) | "Ajustes solicitados pelo cliente" — Tesouraria bug fixes, Reconciliação corrections |
| **GitHub: Wise-Pirates/wise-n8n-backups** | 60+ Cabeceiras workflow JSON backups (including Faturas Fornecedores Fase 1 & 6, EMMA, Reconciliação, Tesouraria) |
| **GitHub: Wise-Pirates/cabeceiras-n8n-backups** | Single full backup file from Jan 20, 2026 |
| **GitHub: robsonadvincula-svg/sistema-automacao-faturas** | **THE SHARED REPO** — Robson's handoff repo with ONBOARDING.md, ESTADO-ATUAL.md, all relatórios, workflow JSONs, project docs. Created July 16, 2026. Public. |
| **GitHub: Wise-Pirates/efatura_app** | Flask + Playwright app for collecting invoices from Portuguese tax portal (e-Fatura) |
| **GitHub: Wise-Pirates/reconciliacao-app** | Python reconciliation engine (e-Fatura × Dynamics 365 BC). Last commit by Nathielle on June 23, 2026. |
| **GitHub: Wise-Pirates/cabeceiras-tesouraria** | Tesouraria workflow + reports |
| **Live Infrastructure** | n8n instance UP, efatura app UP, reconciliacao app 404 (old URL works) |

---

## 3. System Architecture — What Was Built

### 3.1 Main System: Faturas Fornecedores (Scanner → OCR → ERP)

**Purpose**: Automate supplier invoice processing into Business Central ERP

**Flow**:
```
Scanner (Epson) → PDF → OneDrive "faturas por inserir"
    ↓ (n8n polls every 3-5 min)
n8n Workflow (62wyOKnNBy0bnJUw, 69 nodes)
    ├─ Google Document AI → extracts header (NIF, total, date, supplier)
    ├─ Claude Sonnet 4.5 → extracts invoice lines (literal transcription)
    ├─ GPT-5 → verifies signature, maps items
    ├─ Validates against BC (vendor lookup by NIF, duplicate detection)
    ├─ Creates Purchase Order (EC) in Business Central PROD
    ├─ Logs to Excel (OneDrive via Graph API)
    └─ Moves PDF to "faturas inseridas/YYYY/MM/"
```

**Key Design Decisions**:
- Document AI for header (100% reliable)
- Claude Sonnet 4.5 for lines (literal extraction, no hallucination)
- Excel as PK→BC code dictionary (Jorge fills column D)
- Match ONLY by PK (never by description — avoids false positives)
- ME003007 as placeholder item when no BC catalog match
- Flags in `postingDescription` tell Jorge what to review
- NEVER blocks — always delivers EC with flags, Jorge corrects manually

**Cost**: ~$0.02 per invoice (Document AI + Claude)

### 3.2 Supporting Systems

| System | Purpose | Status |
|--------|---------|--------|
| **e-Fatura App** (efatura_app) | Flask + Playwright web app for collecting invoices from Portuguese tax authority portal | Running (200 OK) |
| **Reconciliação App** (reconciliacao-app) | Python engine matching e-Fatura × Dynamics 365 BC entries | Running (old URL); 404 on new URL |
| **Tesouraria Workflow** | Payment reconciliation (Shopify, Stripe, IFTHENPAY, SIBS TPA) | Unknown status |
| **EMMA Dashboard** | Business Central → Excel OneDrive dashboard | Unknown status |
| **Fase 6 Alertas** | Daily check for overdue purchase orders, email alerts | Unknown status |
| **Agente de Orçamentos** | Quote automation for multiple factory locations | Unknown status |

### 3.3 Infrastructure

| Component | URL/ID | Status |
|-----------|--------|--------|
| n8n Cabeceiras | `primary-production-0fe7d.up.railway.app` | ✅ UP (200 OK healthz) |
| efatura App | `efatura-production.up.railway.app` | ✅ UP (200 OK /health) |
| Reconciliação App (new) | `reconciliacao-app-production.up.railway.app` | ❌ 404 |
| Reconciliação App (old) | `reconciliacao-production.up.railway.app` | ✅ UP (200 OK /health) |
| n8n API | API key returns 0 workflows | ⚠️ Key expired or workflows removed |
| Railway Workspace | Wise Pirates - PA, Project: N8N Cabeceiras | — |

---

## 4. ClickUp Task Analysis

### Main Task: "Sistema de Automação de Faturas de Fornecedores (scanner → OCR → ERP)"
- **ID**: 869dvuvec
- **Status**: In Progress
- **Priority**: High
- **Assignee**: Robson Advincula (on sick leave)
- **Due Date**: ~August 2026 (timestamp 1790737200000)
- **Time Spent**: 75 hours logged
- **Last Updated**: Very recently (July 19-20, 2026 approx)

### Comment History (6 detailed reports):
1. **June 30, 2026** — Full system report: 5 phases in production, 1 pending
2. **July 3, 2026** — End-to-end BC PROD validation, 2 Flex2000 invoices processed, 7 bugs fixed, Centro de Custo added
3. **July 6, 2026** — Image (JPEG) support, discounts, "ATENÇÃO VALORES" flag, 9 deploys
4. **July 7, 2026** — GPT-4o→GPT-5 migration (fixed NIF hallucination), 14 deploys, refactor to OpenAI Responses API
5. **July 8, 2026** — 7 ECs in BC PROD, 7 deploys, dedup regex fix, TOPBOIS items preserved
6. **July 15, 2026** (in shared repo, not ClickUp) — 5 bugs fixed, 13 ECs delivered, Claude Sonnet 4.5 migration, env vars migration, GitHub repo published, **workflow DESATIVADO**

### Related Tasks (Cabeceiras PROJECTOS list):
| Task | Name | Status |
|------|------|--------|
| 869dxb5ux | Ajustes solicitados pelo cliente | In Progress |
| 869cayz2b | Upgrade sistema e-fatura e Dynamics | Complete |
| 869c9h1e9 | Integração e-fatura Dynamics 365 | Complete |
| 869c6p63q | Integração Conferência Faturas SHOPIFY E STRIPE | Complete |
| 869c34k4x | Integração Conferência Faturas KLARNA E IFTHENPAY | Complete |
| 869ce5feh | Sistema EMMA | Complete |
| 869d5h919 | Conversion Facebook | Complete |
| 869czk48b | Renovar Token Shopify | Complete |

---

## 5. Likely Root Causes of "Not Working"

### PRIMARY: Workflow Deactivated
The main workflow (`62wyOKnNBy0bnJUw`, 69 nodes) was **DESATIVADO** at the end of the July 15 session. Robson's report states:

> "Workflow **desativado** aguardando resposta do Jorge."

Jorge was supposed to validate 13 ECs (EC2602356-364, EC2602374-376, EC2602378). Since Robson went on sick leave, Jorge likely:
- Never validated the ECs (or did validate and asked for changes)
- The workflow was never reactivated
- New invoices placed in OneDrive "faturas por inserir" are NOT being processed

### SECONDARY: n8n API Key Expired
The n8n API key (JWT issued January 17, 2026) returns 0 workflows. This could mean:
- The key expired and needs renewal
- The workflows were modified/removed
- The n8n instance was reset/rebuilt

**Impact**: Even if someone tries to reactivate the workflow via API, they can't with this key.

### TERTIARY: BC DEV Configuration Blocker
A BC DEV environment issue (C_EC series missing "Starting No.") prevents creating Purchase Orders in DEV. This is documented in `CHECKLIST-BC-DEV-FIX.md` — a 2-minute fix in BC DEV, but requires someone with BC admin access.

### OTHER KNOWN ISSUES (from last session):
- FSM: Claude reads price `0.588` when invoice shows `0,5880` (0.87€ discrepancy)
- Emma Matratzen: Structural numbering difference between AT and Dynamics (requires manual reconciliation)
- Reconciliação app URL changed (new URL 404, old URL works) — n8n may be calling wrong URL
- Credential rotation recommended (3 Azure client secrets were exposed in workflow before env var migration)

---

## 6. The Shared Repo (Robson's Handoff)

**Repo**: `robsonadvincula-svg/sistema-automacao-faturas` (PUBLIC)
**Created**: July 16, 2026 — Robson prepared this as an onboarding/handoff package

### Contents:
| File | Purpose |
|------|---------|
| `ONBOARDING.md` | Complete onboarding guide — 30 min read, covers context, access requests, architecture, operational playbook |
| `ESTADO-ATUAL.md` | Live state snapshot (last updated July 3) |
| `relatorio_2026-06-29.md` through `relatorio_2026-07-15.md` | 7 detailed session reports |
| `workflow_fase1_ocr.json` | Main workflow export |
| `workflow_fase6_alertas.json` | Alerts workflow export |
| `product-discovery.md` | Initial requirements |
| `planeamento-projeto.md` | Project planning |
| `technical-note.md` | Technical notes |
| `transcript.txt` | Meeting transcript with Jorge (June 25) |
| `REUNIAO-JORGE-FASE3.md` | Meeting prep for Phase 3 |
| `PERGUNTAS-JORGE-SEGUNDA-2026-07-06.md` | Questions for Jorge |
| `CHECKLIST-BC-DEV-FIX.md` | BC DEV fix instructions |
| `README.md` | Repo README |

### Access Required (from ONBOARDING.md):
1. **Azure Portal** — 3 app registrations (Graph API, BC PROD, BC DEV) — need client_secrets or collaborator invite
2. **Google Cloud** — `cabeceiras-ocr` project, Document AI processor — need Service Account JSON
3. **Anthropic** — API key (credential `srXSApQJ2OBjvnxL` in n8n)
4. **OpenAI** — API key (credential `CogELPgsbfrLHzNg` in n8n)
5. **n8n Railway** — URL + API key + UI credentials
6. **Business Central** — PROD and DEV access
7. **OneDrive** — brandao@cabeceiras.pt (Graph API access)
8. **Railway** — Workspace: Wise Pirates - PA, Project: N8N Cabeceiras

---

## 7. All Cabeceiras Repos on Wise-Pirates GitHub

| Repo | Description | Last Activity |
|------|-------------|---------------|
| `cabeceiras-tesouraria` | Tesouraria workflow + reports | May 25, 2026 |
| `cabeceiras-n8n-backups` | n8n workflow backups | Jan 20, 2026 |
| `cabeceiras-bc-app` | Business Central app | May 15, 2026 |
| `cabeceiras-conversion-facebook` | Facebook conversion tracking | May 15, 2026 |
| `cabeceiras-shopify-token` | Shopify token management | May 15, 2026 |
| `cabeceiras-tesouraria-brandao` | Tesouraria (Brandão variant) | May 15, 2026 |
| `cabeceiras-relatorios-alertas` | Reports and alerts | May 15, 2026 |
| `cabeceiras-trackpod` | TrackPOD integration | May 15, 2026 |
| `cabeceiras-agente-orcamentos` | Quote agent | May 15, 2026 |
| `cabeceiras-orcamentos` | Quote management | Apr 2, 2026 |
| `efatura_app` | e-Fatura Playwright app | Apr 6, 2026 |
| `reconciliacao-app` | Reconciliation engine | Jun 23, 2026 (Nathielle) |

### n8n Backup Workflows (wise-n8n-backups/cabeceiras/):
60+ workflow JSON files including:
- **Faturas Fornecedores**: Fase 1 OCR, Fase 6 Alertas POs
- **e-Fatura × Dynamics**: Reconciliação Completa
- **EMMA**: Business Central → Excel, Dashboard Web
- **Tesouraria**: Reconciliação Pagamentos (Klarna, IFTHENPAY, Shopify, Stripe)
- **TrackPOD**: Route optimization, daily planning
- **Agente de Orçamentos**: Multiple factory locations
- **Offline Conversions**: Facebook, Google Ads
- **Error Handlers**: For each major workflow
- **Backup**: Daily Google Drive + GitHub backup workflow

---

## 8. Key Credentials & Endpoints (from public docs — may need verification)

> ⚠️ These are from Robson's public repo and ClickUp comments. Some may be rotated/expired.

### Azure
- **Tenant ID**: `f41c8222-df66-449c-93b5-c1879e641cb2`
- **Graph App Client ID**: `b8e798a0-6988-47aa-a4a5-0d4d1fe125eb`
- **BC PROD App Client ID**: `fabec729-bb7e-48f8-a3cc-4a649ed4ab45`
- **BC DEV App Client ID**: `55e26621-e247-4026-9f47-bf17728e77bb`
- **Secrets**: Migrated to Railway env vars (`GRAPH_CLIENT_SECRET`, `BC_PROD_CLIENT_SECRET`, `BC_DEV_CLIENT_SECRET`)

### n8n
- **URL**: `https://primary-production-0fe7d.up.railway.app`
- **Workflow Fase 1**: `62wyOKnNBy0bnJUw` (69 nodes, DESATIVADO)
- **Workflow Fase 6**: `M2I1eekcsJTi9Av7` (8 nodes, alertas)
- **API Key**: In vault as `N8N_CABECEIRAS_API_KEY` (may be expired)

### Business Central
- **PROD OData**: `https://api.businesscentral.dynamics.com/v2.0/f41c8222-df66-449c-93b5-c1879e641cb2/PROD/ODataV4/Company('Jorge%20Brand%C3%A3o%20Gon%C3%A7alves')`
- **DEV**: Same tenant, environment `DEV`

### OneDrive
- **Drive ID**: `b!fxQjRtmydUqtNaD2rLbsDpw9vi50zupCl2O41a8_kHwqy-IfbKGnQa-AvUPzD_nm`
- **Pasta faturas por inserir**: `01JSSR6ZOWI4JQ7ADEZFHZT24VNIWIU42R`
- **Pasta faturas inseridas**: `01JSSR6ZLDQWWQPDGYP5CZJAQFOZUTJGI2`
- **Pasta faturas_com_erro**: `01JSSR6ZMVOBS354FCNVDZG2CAXFQFJUR6`
- **Excel Log_Faturas**: `01JSSR6ZOQEYD67CHDWZDYCTBQAOQPLZKA`
- **Excel Codigos Fornecedores**: `01JSSR6ZOQ4BV4R2JA4JEJJD4BPSVMLY2K`

### Google Cloud
- **Project**: `cabeceiras-ocr` (ID: 164548594521)
- **Document AI Processor**: `d9230429c35852ce` (region EU)

### Railway
- **Workspace**: Wise Pirates - PA
- **Project**: N8N Cabeceiras
- **Service**: Primary (n8n)

---

## 9. Recommendations for Andie & Nathi

### Immediate Actions (Information Gathering — NO CHANGES)
1. **Access the n8n UI directly** — login to `primary-production-0fe7d.up.railway.app` with UI credentials (need to get from Robson or Railway) to check:
   - Is the workflow `62wyOKnNBy0bnJUw` still there?
   - Is it activated or deactivated?
   - Are there recent execution logs?
   - What errors are showing?

2. **Check OneDrive "faturas por inserir"** — if there are unprocessed PDFs piling up, that confirms the workflow is off

3. **Check Business Central PROD** — query recent ECs with `yourReference = 'AUTOMAÇÃO'` to see last processed invoice date

4. **Contact Robson** (if possible) for:
   - n8n UI credentials (username/password)
   - Current n8n API key (if the one in vault is expired)
   - Any known issues he didn't document
   - Confirmation of system state when he left

5. **Talk to Jorge** (client) to understand:
   - What exactly "not working" means (no new ECs? wrong data? errors?)
   - Did he validate the 13 ECs from July 15?
   - Are there new invoices in OneDrive that aren't being processed?
   - Any changes to BC, Azure, or OneDrive since Robson left?

### For the Shared Repo (Nathi's task)
1. Create repo on Wise Pirates account
2. Clone/fork `robsonadvincula-svg/sistema-automacao-faturas` as starting point
3. Add this intelligence report
4. Document findings from n8n UI investigation
5. Track all credentials/access requests in a secure way (NOT in git)

### Do NOT Do (Yet)
- ❌ Do NOT reactivate the n8n workflow without understanding why it was deactivated
- ❌ Do NOT modify any BC data
- ❌ Do NOT change any Azure app registrations or secrets
- ❌ Do NOT push any code to production
- ❌ Do NOT delete or modify any OneDrive files
- ❌ Do NOT touch the Tesouraria or Reconciliação systems

---

## 10. Open Questions

1. **Why is the n8n API key returning 0 workflows?** Is it expired? Were workflows deleted? Was the n8n instance rebuilt?
2. **Did Jorge validate the 13 ECs?** If yes, what was his feedback? If no, are they still in BC?
3. **Are there invoices piling up in OneDrive?** How many? From which suppliers?
4. **Is the Reconciliação app URL issue affecting production?** The n8n workflow may be calling the wrong URL
5. **What happened to the Tesouraria system?** It had 8 bug fixes on June 29 — are those still working?
6. **Does the client need the full system or just specific parts?** The system has many components — which ones are critical?
7. **What access does Andie/Nathi currently have?** Railway? n8n UI? Azure? BC? OneDrive?
8. **Is the Google Document AI processor still active?** Billing/quotas may have changed

---

## 11. Timeline Summary

| Date | Event |
|------|-------|
| Mar 3, 2026 | Robson starts building e-Fatura × Dynamics reconciliation |
| Mar 10, 2026 | Meeting with APR.pt for Dynamics API access |
| Mar 20, 2026 | First reconciliation deploy, 7 divergences fixed |
| Mar 24, 2026 | efatura_app deployed to Railway with Cabeceiras branding |
| Apr 2, 2026 | BC API integration, Railway deploy of reconciliacao-app |
| Apr 6, 2026 | Last commit on efatura_app (favicon, paste fix) |
| May 12-25, 2026 | Tesouraria system built, credentials migrated to env vars |
| Jun 23, 2026 | Nathielle commits to reconciliacao-app (NIF matching fix) |
| Jun 29, 2026 | Tesouraria: 8 bug fixes, Reconciliação: 3 normalization fixes |
| Jun 30, 2026 | Faturas Fornecedores: Full system report, 5 phases in production |
| Jul 1, 2026 | (ClinicaTear work — unrelated) |
| Jul 3, 2026 | Faturas: BC PROD validation, 7 bugs fixed, Centro de Custo |
| Jul 6, 2026 | Faturas: Image support, discounts, ATENÇÃO VALORES flag |
| Jul 7, 2026 | Faturas: GPT-4o→GPT-5 migration (NIF hallucination fix) |
| Jul 8, 2026 | Faturas: 7 ECs in BC PROD, dedup regex fix |
| Jul 14, 2026 | Faturas: Claude Sonnet 4.5 migration, 10 ECs delivered |
| Jul 15, 2026 | Faturas: 5 bugs fixed, 13 ECs, env vars migration, GitHub published, **WORKFLOW DESATIVADO** |
| Jul 16, 2026 | Robson creates handoff repo (sistema-automacao-faturas) |
| ~Jul 17+ | Robson goes on sick leave |
| Jul 23, 2026 | Andie & Nathi asked to help — this audit |

---

*This document is READ-ONLY intelligence. No systems were modified during this investigation.*