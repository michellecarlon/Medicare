# Medicare Claims In-Sourcing — PRD v0

**Status:** Draft, 2026-07-02
**Author:** Michelle Carlon + Claude
**Population:** Medicare members (traditional Medicare / Medicare Advantage with secondary supplemental)
**Target delivery:** Q4 2026

---

## TL;DR

Talkspace pays Advantum to run Medicare claims through Tebra/Trizetto. Advantum submits claims, receives ERAs, tracks secondary/supplemental claims, posts payments, and invoices members via Stripe. Talkspace receives ERAs but does not ingest them — there is no ERA-to-claim association in the TS system today, no visibility into secondary claim status, and no locality-based rate logic.

We will:

1. **Ingest and process Medicare ERAs** inside the TS claims system. Associate primary and secondary ERA data to the originating claim and expose it in backoffice.
2. **Manage Coordination of Benefits (COB)** — create and submit secondary/supplemental claims triggered by primary ERA outcome (autoforward detected, or secondary on file with no autoforward).
3. **Apply locality-based Medicare rates** from a master rate table, using provider home zip code to determine the correct allowed amount for each claim.

Net effect: Advantum is fully replaced for Medicare claims. Rev Ops gains real-time visibility into claim status, ERA associations, and member balances for both primary and secondary insurance. One vendor relationship is eliminated and one Rev Ops FTE currently absorbed by Advantum is redirected to managing the process in-house.

### Scope split: v1 vs. v1.5 vs. v2

**v1 (ship first):** ERA ingestion + association; COB/secondary claim lifecycle; new backoffice fields; locality rate table; Medicaid/DUALS billing block; Rev Ops manual secondary submission; backoffice secondary insurance view/edit. Manual Stripe invoicing (Rev Ops initiates). Direct clearinghouse connection via Trizetto (same path Advantum uses today).

**v1.5 (defer):** Client-facing secondary insurance add/update flow. Eligibility verification on secondary policies.

**v2 (future):** Automated Stripe invoicing triggered by claim outcome. Full A/R reporting suite (takebacks, refunds, overpayments, multi-payment tracking). SOX-compliant CMS rate update workflow.

---

## Claim lifecycle — one Medicare member, end to end

```
Primary claim path (standard):

  ┌─────────────┬───────────────┬───────────────┬─────────────────┐
  │   SESSION   │  CLAIM SENT   │  MAC RETURNS  │  BACKOFFICE     │
  │  COMPLETE   │  TO TRIZETTO  │     ERA       │  UPDATED        │
  │             │               │               │                 │
  │ Provider    │ TS submits    │ ERA received  │ Allowed Amt,    │
  │ session     │ claim at      │ 14–30 days    │ primary paid,   │
  │ ends        │ $224.46 via   │ later         │ claim status,   │
  │             │ locality rate │               │ ERA ID stored   │
  └──────●──────┴──────●────────┴──────●────────┴────────●────────┘

ERA outcome branches:

  PROCESSED_AS_PRIMARY          PROCESSED_AS_PRIMARY_FORWARDED
       │                                   │
       ▼                                   ▼
  Record primary payment         Record primary payment
  Check for secondary on file    Mark claim autoforwarded
  If yes → submit secondary      Await secondary ERA
  Member owes remainder          Update secondary fields on receipt
       │                                   │
       ▼                                   ▼
  Rev Ops reviews or             Claim fully settled
  auto-invoice member            Member balance $0 (or remainder)
```

---

## Problem

Today, the TS claims system has no Medicare-specific logic and no ERA ingestion. The following gaps exist:

- **No ERA-to-claim association.** ERAs arrive and are forwarded to Tebra; they are never stored or linked to a claim record in the TS database. Rev Ops cannot see ERA data in backoffice without going to Tebra.
- **No secondary claim visibility.** Secondary / supplemental claims are created and managed entirely in Tebra. TS has no record of secondary claim status, secondary payments, or secondary ERA outcomes.
- **No Coordination of Benefits logic.** TS cannot detect when a primary ERA indicates autoforward, cannot trigger a secondary claim, and cannot route a claim to a secondary payer on behalf of a member who has supplemental insurance.
- **No locality-based rates.** Every claim is submitted at $224.46 regardless of the MAC (Medicare Administrative Contractor) jurisdiction. The correct allowed amount varies by locality. TS has the rate table but does not apply it to claims.
- **No Medicaid/DUALS detection.** Members with Medicaid as secondary (DUALS) must never be billed out-of-pocket. Today there is no automated block; this is a manual process and a compliance risk.
- **Member balance is not updated from ERA outcomes.** When Medicare pays and Optum pays secondary, the member's balance in backoffice is not reconciled automatically. Rev Ops manually posts payments.
- **Vendor dependency.** Advantum/Tebra is a single point of failure for the entire Medicare revenue cycle. Contract, pricing, and operational control sit with the vendor.

---

## Goals (v1)

1. **ERA ingestion:** Every Medicare ERA (primary and secondary) is ingested into the TS system and associated to the originating claim in the database.
2. **COB / secondary claims:** TS creates and submits secondary/supplemental claims triggered by primary ERA outcome, without manual intervention for the auto-forward path.
3. **Locality rates:** All Medicare claims are submitted at the locality-specific allowed amount based on provider home zip code.
4. **Medicaid block:** Members identified as Medicaid or DUALS are never invoiced via Stripe. The block is applied automatically from remark codes and can be set manually at the insurance or patient level.
5. **Backoffice visibility:** Rev Ops can view and manage all Medicare claim fields — primary and secondary — in backoffice without leaving the TS system.

## Non-Goals (v1)

- Client-facing secondary insurance add/update — deferred to v1.5
- Automated Stripe invoicing on claim outcome — deferred to v2 (Rev Ops manually invoices in v1)
- Full A/R reporting suite (takeback tracking, refund reporting, overpayment detection) — deferred to v2
- SOX-process tooling for biannual CMS rate updates — deferred to v2; interim process is manual with Rev Ops ownership
- Eligibility verification on secondary policies — deferred to v1.5
- Non-Medicare payers — rate table infrastructure benefits BCBS and TRICARE but payer-specific logic is out of v1 scope

---

## Users & Stories

### Rev Ops

- **As a Rev Ops analyst**, I want to see primary and secondary ERA data associated to each claim in backoffice, so that I can verify payment without logging into Tebra.
- **As a Rev Ops analyst**, I want the system to automatically create and submit a secondary claim when a primary ERA indicates autoforward or when a secondary insurance is on file, so that I only intervene for exception cases.
- **As a Rev Ops analyst**, I want to manually submit a secondary claim from backoffice when the automatic path doesn't apply, so that I have a fallback for edge cases without going to a vendor system.
- **As a Rev Ops analyst**, I want to view and edit a member's secondary insurance in backoffice, so that I can correct data without requiring the member to update it themselves.
- **As a Rev Ops analyst**, I want the member balance to reflect both primary and secondary payments once ERAs are processed, so that I know the correct amount to invoice the member (if any).
- **As a Rev Ops lead**, I want a backoffice filter for all Medicare payers, so that I can monitor the Medicare book of business without custom SQL.
- **As a Rev Ops lead**, I want the system to alert me when an ERA returns an allowed amount higher than the net revenue (locality rate), so that I can initiate a rate review before the variance compounds.

### Member (Medicare)

- **As a Medicare member**, I want my Stripe invoice to reflect the correct amount after both Medicare and my supplemental insurance pay, so that I am not overbilled.
- **As a Medicare-Medicaid dual-eligible (DUALS) member**, I want to never receive an invoice from Talkspace, so that I am not asked to pay a copay that regulations prohibit.

### Provider

- **As a provider**, I want my claims submitted at the correct locality-specific rate, so that my Medicare reimbursement is accurate for my jurisdiction.

### Engineering / Backoffice

- **As an engineer**, I want ERA data stored in a normalized, claim-associated table, so that downstream features (A/R reporting, balance reconciliation, secondary claim triggers) have a clean data source.

---

## Current State (detailed)

| Step | Today's actor | What happens |
|---|---|---|
| Session complete | TS | Claim created in TS, submitted to Tebra under ELCMS legal entity with state, provider state facility |
| Claim submission | Tebra/Trizetto | Claim submitted at $224.46 to MAC; Tebra auto-updates to state-level payer ID |
| Primary ERA received | MAC → TS → Tebra | TS receives ERA, forwards to Tebra. Not stored in TS DB. No association to claim in TS. |
| ERA processing | Tebra | Tebra records ERA outcome; assigns primary payment to patient record |
| Secondary claim | Tebra | Tebra tracks secondary/supplemental claims; submits to secondary payer |
| Secondary ERA / paper EOB | Advantum | If secondary payment arrives via lockbox, Advantum manually posts it |
| Secondary (Optum autoforward) | Tebra | Tebra receives autoforwarded secondary ERA from Optum; posts payment |
| Member invoicing | Tebra → Stripe | Tebra invoices member via Stripe; updates balance after Stripe payment |
| Denials / rejections | Advantum | Advantum works denials; TS has no visibility |

**Gap summary:** TS has zero visibility into ERA outcomes, secondary claim status, or member balance after Medicare pays. Every step after claim submission lives in Tebra/Advantum.

---

## Future State Requirements

### A. New Backoffice Fields

These fields are added to the claim / session record in backoffice. Marked **(new)** if not currently tracked, **(existing)** if the field exists but needs to be populated from ERA data.

| Field | Type | Source | Notes |
|---|---|---|---|
| Billed Amount | Currency | TS system | $224.46 today; will remain fixed at billed charge regardless of locality rate |
| Net Revenue | Currency (existing) | TS rate table | Locality-specific rate; used for revenue reporting. This is the expected allowed amount. |
| Allowed Amount | Currency (existing) | Primary ERA | Returned by MAC. Alert Rev Ops if Allowed Amount > Net Revenue. |
| Primary Insurance Paid | Currency (existing) | Primary ERA | Amount MAC paid |
| Secondary Insurance Paid | Currency (new) | Secondary ERA | Amount secondary/supplemental paid; supports manual post |
| Refunds / Takebacks | Currency (new) | ERA adjustment | Negative adjustments / payer recoupments |
| ERA ID — Primary | String (new) | Primary ERA | ERA transaction identifier for reconciliation |
| ERA ID — Secondary | String (new) | Secondary ERA | ERA transaction identifier for reconciliation |
| Claim Create Date — Secondary | Date (new) | TS system | When TS created the secondary claim |
| Claim Submit Date — Secondary | Date (new) | Trizetto | When secondary claim was transmitted to clearinghouse |
| Claim Status — Secondary | Enum (new) | Secondary ERA | `processing`, `completed`, `denied`, `rejected` |
| Adjustment Reason — Secondary | String (new) | Secondary ERA | CARC code from secondary ERA |
| Remark Code — Secondary | String (new) | Secondary ERA | RARC code from secondary ERA; also used for Medicaid block trigger |
| Primary Claim Status | Enum (existing) | Primary ERA | Extend to support multiple statuses per claim |

**Multiple payments per claim:** The infrastructure must support multiple payment records per claim (primary paid, secondary paid, primary recoupment, etc.). The scoping doc notes this infrastructure is partially in place.

**Multiple claim statuses per session:** A single session may have status combinations — e.g., Primary: `completed`, Secondary: `denied`. Both states must be visible and independently updatable.

---

### B. ERA Ingestion and Processing

**Primary ERA ingestion:**

1. When a Medicare ERA is received by TS, store it in a new `claim_era` table linked to the originating claim by claim ID / external claim ID.
2. Parse and store: ERA ID, payer, payment date, allowed amount, paid amount, adjustment codes (CARC), remark codes (RARC), claim status code.
3. Update the claim record with: Allowed Amount, Primary Insurance Paid, ERA ID — Primary, Primary Claim Status.
4. Detect ERA status code to determine downstream routing:

| ERA Status | Action |
|---|---|
| `PROCESSED_AS_PRIMARY` | Record payment. Check for secondary insurance on member record. If secondary exists and ERA did not autoforward → trigger secondary claim submission (see Section C). |
| `PROCESSED_AS_PRIMARY_FORWARDED` | Record payment. Mark claim as autoforwarded. Set Secondary Claim Status = `processing`. Await secondary ERA. |
| `PROCESSED_AS_SECONDARY` | Record secondary payment. Update Secondary Insurance Paid, ERA ID — Secondary, Secondary Claim Status. Reconcile member balance. |

**Secondary ERA ingestion:**

1. Match secondary ERA to the originating claim via external claim ID.
2. Store: ERA ID — Secondary, Secondary Claim Status, Adjustment Reason — Secondary, Remark Code — Secondary, Secondary Insurance Paid.
3. Trigger member balance reconciliation (see Section F).

**Remark code monitoring:**

1. On any ERA (primary or secondary), scan RARC codes for Medicaid/DUALS indicators.
2. If detected, apply billing block (see Section E).

---

### C. Coordination of Benefits — Secondary Claim Lifecycle

Three paths for secondary claim submission:

**Path 1 — Autoforward detected (ERA says `PROCESSED_AS_PRIMARY_FORWARDED`):**
- No secondary claim submission needed; MAC has forwarded to the secondary payer.
- TS marks Secondary Claim Status = `processing`.
- Await secondary ERA (Section B handles receipt).
- Rev Ops visibility: they can see the autoforward status and are not required to act.

**Path 2 — Auto-submit (secondary on file, no autoforward):**
- Trigger: ERA is `PROCESSED_AS_PRIMARY` AND the member has a secondary insurance on file in TS backoffice.
- TS system automatically creates a secondary claim using the primary ERA outcome data.
- Submits to Trizetto for transmission to the secondary payer.
- Sets Claim Create Date — Secondary, Claim Submit Date — Secondary, Secondary Claim Status = `processing`.

**Path 3 — Manual Rev Ops submission:**
- Trigger: Rev Ops initiates from backoffice when auto-submit did not fire or failed.
- Backoffice exposes a "Submit Secondary Claim" action on the claim record.
- Rev Ops can edit secondary insurance details before submitting.
- Required for any case where the secondary payer or coverage is not yet in the TS system.

**Secondary claim data requirements:**
- Secondary claim must include primary ERA data (primary payer paid amount, adjustment codes) as crossover claim information (loop 2330B in X12 837P).
- Secondary claim must use the same provider NPI, taxonomy, and service line data as the primary claim.

---

### D. Medicare Rate Table (Locality-Based Pricing)

**Objective:** Apply the correct Medicare locality rate to each claim at submission time. Today every claim is submitted at $224.46 regardless of MAC jurisdiction.

**Rate table design:**
- Master table mapping ZIP code → locality → allowed amount (by CPT code where applicable).
- Table covers all TS-active MAC jurisdictions.
- Rate used on claim = provider home ZIP code lookup.
- Net Revenue field in backoffice is populated from this table (not from ERA — ERA returns Allowed Amount, which is a separate field).

**Rate selection logic:**
1. Look up provider home ZIP code on claim.
2. Identify Medicare locality for that ZIP.
3. Apply locality-specific rate as Net Revenue and as the submitted charge amount if TS policy moves to locality-based billing.

> **Note:** The scoping doc indicates TS currently submits at $224 flat. The rate table is a prerequisite for accurate Net Revenue tracking and for future locality-accurate billing. The decision to change the submitted billed amount (Billed Amount) is a separate policy decision; v1 focuses on populating Net Revenue from the table.

**Alert logic:**
- If Allowed Amount (from ERA) > Net Revenue (from rate table), surface an alert to Rev Ops: *"ERA allowed amount exceeds expected rate for this locality — rate review recommended."*

**Rate table maintenance (interim — v1):**
- CMS updates Medicare rates biannually (January and July).
- In v1, rate table updates are a manual Rev Ops process. Owner TBD (see Open Questions).
- SOX-compliant update workflow with audit trail is deferred to v2.

---

### E. Medicaid / DUALS Detection and Billing Block

**Rule:** Members with Medicaid as primary or secondary insurance (including DUALS — Medicare + Medicaid) must never receive a Stripe invoice. Federal and state regulations prohibit balance-billing Medicaid beneficiaries.

**Detection methods (AND — apply any that trigger):**

1. **Remark code on ERA:** If any ERA for a claim includes a RARC code indicating Medicaid/DUALS (e.g., CO-96, CO-45 with Medicaid payer ID context), automatically apply billing block.
2. **Payer ID match:** Maintain a list of known Medicaid payer IDs. If member's secondary insurance payer ID is on this list, block billing.
3. **Manual flag:** Rev Ops can manually set a billing block on a member record (insurance level or patient level) in backoffice.

**Billing block behavior:**
- When block is active: Stripe invoice is suppressed. Member balance is set to $0.
- Block applies to the current claim and all future claims for that member until explicitly removed.
- Rev Ops can view and clear billing blocks in backoffice.

**Block audit:** Every block application and removal is logged with timestamp, actor (system or Rev Ops user), and trigger reason (remark code, payer ID match, or manual).

---

### F. Member Balance Reconciliation

**Objective:** After each ERA (primary and secondary) is processed, the member's balance in backoffice reflects the correct patient responsibility — and Stripe invoicing (manual in v1) is based on that balance.

**Balance calculation:**

```
Member balance = Billed Amount − Primary Insurance Paid − Secondary Insurance Paid − Refunds/Takebacks
```

**Update triggers:**
- Primary ERA processed → update Primary Insurance Paid → recalculate balance.
- Secondary ERA processed → update Secondary Insurance Paid → recalculate balance.
- Manual secondary payment posted by Rev Ops → update Secondary Insurance Paid → recalculate balance.
- Takeback/recoupment ERA received → update Refunds → recalculate balance.
- Stripe payment received → reduce balance by payment amount.

**Manual secondary payment posting:**
- Rev Ops can post a secondary payment manually from backoffice (for paper EOBs / lockbox payments that arrive outside the EDI path).
- Required fields: payment amount, payment date, payer, check/EFT reference.

**Stripe sync:**
- When a member pays via Stripe from a Medicare invoice, the payment is reflected in the patient balance in backoffice.
- When Rev Ops manually invoices a member, the resulting Stripe payment updates the claim-level patient balance.
- Medicaid billing block (Section E) prevents Stripe invoice creation.

---

### G. Backoffice UI — Rev Ops Views

**Claim detail view:**
- All fields from Section A visible on the claim record.
- Primary and secondary ERA data visible side by side.
- Claim status timeline: Primary Status, Secondary Status.
- Actions: Submit Secondary Claim (Path 3), Post Manual Payment, Apply/Remove Billing Block.

**Secondary insurance view/edit:**
- Rev Ops can view secondary insurance on a member record: payer name, payer ID, member ID, group number, effective dates.
- Rev Ops can add, edit, or remove secondary insurance from backoffice.
- Source of truth for secondary insurance used in COB auto-submit (Path 2).

**Medicare payer filter:**
- All backoffice claim views support filtering by Medicare payers (traditional Medicare + Medicare Advantage).
- Enables Rev Ops to isolate and monitor the Medicare book of business.
- Reference: [DS Report — Primary and Secondary Claims](https://looker.talkspace.com/looks/2777?toggle=fil)

---

## Happy Paths

### Happy Path 1 — Secondary with Autoforward

A Medicare member with supplemental insurance (e.g., Optum) has a session. MAC forwards the claim to Optum automatically.

| Step | Actor | Event |
|---|---|---|
| 1 | Member | Signs up with Medicare member ID; first session completed |
| 2 | TS | Submits claim 1234 at locality rate; Billed Amount: $224 |
| 3 | MAC | Returns ERA: `PROCESSED_AS_PRIMARY_FORWARDED`; paid $80; Allowed Amount: $100 |
| 4 | TS system | Ingests ERA; sets: Primary Paid = $80, Allowed Amount = $100, ERA ID — Primary, Primary Claim Status = `completed`, Secondary Claim Status = `processing` |
| 5 | TS system | Detects Allowed Amount ($100) = Net Revenue ($100) — no rate alert triggered |
| 6 | Optum | Returns secondary ERA: `PROCESSED_AS_SECONDARY`; paid $20 |
| 7 | TS system | Ingests secondary ERA; sets: Secondary Paid = $20, ERA ID — Secondary, Secondary Claim Status = `completed` |
| 8 | TS system | Recalculates member balance: $224 − $80 − $20 = $0 (if $100 allowed covers remainder via COB) |
| 9 | Rev Ops | Reviews — no member invoice needed |

**Backoffice state after Step 4:**

| Field | Value |
|---|---|
| Billed Amount | $224 |
| Net Revenue | $100 |
| Allowed Amount | $100 |
| Primary Insurance Paid | $80 |
| Secondary Insurance Paid | $0 |
| Primary Claim Status | completed |
| Secondary Claim Status | processing |

**Backoffice state after Step 7:**

| Field | Value |
|---|---|
| Primary Insurance Paid | $80 |
| Secondary Insurance Paid | $20 |
| Primary Claim Status | completed |
| Secondary Claim Status | completed |
| Member Balance | $0 |

> **Rate alert note:** If MAC returns Allowed Amount > Net Revenue (locality rate), system alerts Rev Ops and triggers rate review before the pattern recurs on future claims.

---

### Happy Path 2 — Primary Only

A Medicare member with no supplemental insurance. MAC pays primary; member owes remainder.

| Step | Actor | Event |
|---|---|---|
| 1 | Member | Signs up with Medicare only; session completed |
| 2 | TS | Submits claim 12345 at locality rate; Billed Amount: $224 |
| 3 | MAC | Returns ERA: `PROCESSED_AS_PRIMARY`; paid $80; Allowed Amount: $100 |
| 4 | TS system | Ingests ERA. No secondary on file. No autoforward. Sets: Primary Paid = $80, Member Balance = $20 (per COB: member owes coinsurance) |
| 5 | Rev Ops | Reviews member balance; manually invoices member $20 via Stripe |
| 6 | Member | Pays $20 via Stripe |
| 7 | TS system | Stripe payment received; patient balance updated to $0 |

**Backoffice state after Step 4:**

| Field | Value |
|---|---|
| Primary Insurance Paid | $80 |
| Secondary Insurance Paid | n/a |
| Secondary Claim Status | null |
| Member Balance | $20 |

---

### Happy Path 3 — Secondary Does Not Autoforward

MAC returns `PROCESSED_AS_PRIMARY` (no autoforward), but member has a secondary insurance on file in TS.

| Step | Actor | Event |
|---|---|---|
| 1 | Member | Has Medicare + secondary insurance on file in TS backoffice |
| 2 | TS | Submits primary claim 12345; Billed Amount: $224 |
| 3 | MAC | Returns ERA: `PROCESSED_AS_PRIMARY`; paid $80 |
| 4 | TS system | Ingests ERA. Detects secondary insurance on file. No autoforward in ERA. Triggers auto-submit (COB Path 2). Creates secondary claim; sets Claim Create Date — Secondary, Claim Submit Date — Secondary, Secondary Claim Status = `processing` |
| 5 | Secondary payer | Returns ERA: paid $15 |
| 6 | TS system | Ingests secondary ERA; updates Secondary Insurance Paid = $15, Secondary Claim Status = `completed`; recalculates member balance |
| 7 | Rev Ops | Reviews balance; invoices or adjusts off remainder |

**Backoffice state after Step 4:**

| Field | Value |
|---|---|
| Primary Insurance Paid | $80 |
| Secondary Insurance Paid | $0 |
| Secondary Claim Status | processing |
| Claim Submit Date — Secondary | [submitted date] |

---

## Configuration

Admin-configurable values that govern Medicare claims behavior without requiring a code deploy.

| Config key | Type | Default | Purpose |
|---|---|---|---|
| `MEDICARE_COB_AUTO_SUBMIT` | Boolean | `true` | When true, system auto-submits secondary claim (Path 2) when secondary on file and ERA is `PROCESSED_AS_PRIMARY`. Kill switch for auto-submit if needed. |
| `MEDICARE_STRIPE_BLOCK_MEDICAID_PAYER_IDS` | Array of strings | See note | List of Medicaid payer IDs that trigger automatic billing block. Maintained by Rev Ops. Updated outside SOX process in v1. |
| `MEDICARE_RATE_EFFECTIVE_DATE` | ISO date | Latest CMS update | Determines which rate table version is active. Updated biannually by Rev Ops. |
| `MEDICARE_ALLOWED_AMOUNT_ALERT_THRESHOLD` | Boolean | `true` | When true, alert fires if ERA Allowed Amount exceeds Net Revenue (locality rate). Can be toggled off during rate table updates. |
| `MEDICARE_BILLING_FILTER_PAYER_IDS` | Array of strings | All Medicare payer IDs | Payer IDs treated as Medicare for backoffice filtering and claim routing. |

---

## Success Metrics

### Primary

1. **ERA association rate:** % of Medicare ERAs ingested and linked to a claim record in TS within 48 hours of receipt. **Target: ≥ 95%** (baseline today: 0%).
2. **Secondary claim submission rate:** % of eligible secondary claims (autoforward or secondary on file) submitted within 24 hours of primary ERA processing. **Target: ≥ 90%**.

### Secondary

- Rev Ops time per Medicare claim resolution — directional reduction vs. pre-launch baseline (proxy: claims worked per analyst per day)
- Backoffice accuracy: member balance matches ERA outcome for ≥ 98% of processed claims
- Medicaid block compliance: 0 Stripe invoices sent to DUALS members post-launch

### Guardrails (any breach triggers pause + investigation)

- **Claim rejection rate increases ≥ 10% relative** from baseline after clearinghouse transition — may indicate submission format issues
- **Unmatched ERA rate ≥ 5%** — ERAs received but not associated to a claim; indicates matching logic failure
- **Missing secondary submission for eligible claims ≥ 5%** — auto-submit logic not firing as expected

---

## Phasing

**v1 (target: Q4 2026 — ship to replace Advantum):**

- ERA ingestion and primary ERA claim association
- Secondary ERA processing
- COB logic: Paths 1 (detect autoforward), 2 (auto-submit), and 3 (manual Rev Ops)
- New backoffice fields (all fields in Section A)
- Locality-based rate table (populate Net Revenue; alert on Allowed Amount variance)
- Medicaid/DUALS billing block
- Member balance reconciliation (primary + secondary)
- Rev Ops secondary insurance view/edit in backoffice
- Medicare payer filter in backoffice
- Manual Stripe invoicing by Rev Ops
- Clearinghouse connection: Trizetto (same path as Advantum today)

**v1.5 (post Q4 2026):**

- Client-facing secondary insurance add/update flow
- Eligibility verification on secondary policies
- "Billing block applied" notification to Rev Ops dashboard

**v2 (future):**

- Automated Stripe invoicing triggered by ERA outcome (zero-touch for fully-paid claims)
- Full A/R reporting: takeback tracking, refund ledger, overpayment detection, multi-payment history
- SOX-compliant CMS rate update workflow with change log, approvals, and audit trail
- Account management model for Medicare members (open question — see below)

---

## Risks

| Risk | Mitigation |
|---|---|
| Clearinghouse transition issues | Stay on Trizetto (Advantum's current path) — no clearinghouse change in v1; minimizes transition risk |
| ERA matching failures (ERA does not contain enough claim identifiers to match in TS) | Validate matching logic in QA against a sample of real Medicare ERAs before launch; add unmatched ERA queue in backoffice |
| Secondary claim format errors (invalid COB/crossover data) | Test with Trizetto in staging; validate loop 2330B crossover data against real primary ERA samples |
| Medicaid DUALS member billed accidentally | Defense in depth: payer ID list + remark code detection + manual block; log all block decisions for audit |
| CMS rate table not updated biannually → wrong Net Revenue | Establish calendar reminders for January and July updates; v2 SOX workflow formalizes this |
| SOX compliance gap on rate table updates | v1 interim: manual process with Rev Ops ownership documented in runbook; v2 adds formal controls |
| 1 FTE coverage gap (Advantum FTE no longer managing Medicare) | Rev Ops to hire or reassign 1 FTE to absorb Advantum's manual posting / denial management work pre-launch |
| Optum overpayment / takeback not captured | Refunds field captures ERA-based recoupments; paper/lockbox takebacks require manual Rev Ops post in v1 |
| HIPAA — secondary insurance data exposure | Secondary insurance stored in TS under same PHI controls as primary; no new data categories introduced |
| Advantum contract wind-down overlap | Coordinate go-live date with Advantum contract end; plan for parallel run period (claims in flight at cutover) |

---

## Open Questions

### Resolved in scoping

1. **Clearinghouse:** ✅ Stay on Trizetto — no clearinghouse change in v1.
2. **Rate basis:** ✅ Provider home ZIP code used to determine locality rate.
3. **Medicaid block scope:** ✅ Block at insurance and patient level; also triggered by remark code.
4. **Billed Amount:** ✅ Remains $224.46 flat in v1; Net Revenue populated from locality rate table separately.

### Still open

1. **Is Advantum manually submitting secondary claims today?** Scoping doc notes this as an open question. Answer determines the true scope of what TS needs to replicate in v1 (manual secondary path scope).
2. **SOX process for CMS rate updates — who owns it?** Needs an owner assigned before launch. Rev Ops seems the natural home, but requires sign-off. Biannual updates (January, July) must be tracked.
3. **Account management model for Medicare members.** Who owns member-facing communication about claims, COB status, and secondary insurance? Not defined.
4. **Advantum contract wind-down timeline.** What is the notice period / contract end date? This is the external deadline constraint on v1 delivery.
5. **Secondary insurance data source for existing members.** Members already in the TS system may not have secondary insurance entered in backoffice. What is the migration plan to populate this data for the active Medicare population before launch?
6. **Paper ERA / lockbox secondary payments — volume.** Advantum posts paper secondary payments today. What is the monthly volume? This determines how much manual Rev Ops posting burden transfers in v1 vs. how much pressure there is to automate.
7. **Secondary claim for autoforward path — does TS need to track the secondary submission separately from the Optum-handled forward?** The MAC forwards automatically to Optum; TS does not submit. The secondary claim tracked in TS is a record, not a submission. Confirm this is the correct interpretation.

---

## Decisions Log

- **Clearinghouse (v1):** Stay on Trizetto. Rationale: Advantum already submits through Trizetto; reusing the same path reduces EDI format risk and clearinghouse onboarding delay.
- **Billed Amount:** Keep at $224.46 flat in v1. Net Revenue is the locality-specific field for reporting. Changing the submitted billed amount is a separate pricing policy decision deferred to post-launch.
- **Stripe automation:** Deferred to v2. Rev Ops manually invoices members in v1. Rationale: auto-invoicing requires confident balance calculation across multiple ERA outcomes; validated in v1, automated in v2.
- **COB Path 2 (auto-submit):** System auto-submits secondary when secondary is on file AND ERA is `PROCESSED_AS_PRIMARY`. Rev Ops can override. Rationale: reduces manual work on the highest-volume case.
- **Medicaid block — defense in depth:** Block applied via payer ID list AND remark code detection AND manual flag. Any one trigger is sufficient. Rationale: compliance risk justifies redundancy.
- **Rate table alerts:** Alert fires when Allowed Amount > Net Revenue (locality rate). Rev Ops reviews, not auto-adjusted. Rationale: rate discrepancies may indicate a table error, a provider assignment error, or a legitimate CMS change; human review appropriate before any action.
- **v1.5 — client-facing secondary insurance:** Deferred. Rev Ops manages secondary insurance data in backoffice for v1. Rationale: backoffice coverage is sufficient for launch; client UX adds complexity without blocking v1 go-live.
- **Multiple payments per claim:** Infrastructure must support from launch. Rationale: Medicare + secondary + potential recoupments are a known multi-payment pattern; building this constraint in later would require schema migration.
- **DUALS member balance:** Set to $0 once billing block applied. Member is never invoiced. Rationale: regulatory requirement.
