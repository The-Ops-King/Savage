# Savage Zapier Rebuild — Session Context

> Picking up from a web Claude Code session. Read this top-to-bottom, then continue with whatever the user asks next.

---

## Who / What

User: **Tyler** (Mac mini, local Claude Code). Owner of Savage / The-Ops-King.
Goal: rebuild a sprawling Zapier account that fires off Whop payment webhooks. Lots of tech debt, lots of duplication across products.

## Product hierarchy

- **Freedom Accelerator** — lower high-ticket entry / downsell
- **SSA** — flagship high-ticket
- **Empire** — high-ticket upsell

Customer journey variations:
- Freedom → SSA → Empire (full ladder)
- SSA → Empire
- Freedom → SSA only
- SSA only
- Freedom only

Other context:
- Customers may have bought a low-ticket before any high-ticket.
- They may or may not have any prior payment.
- They may or may not have logged into Heartbeat (community platform) before.
- Whop sometimes returns 2 emails for one buyer — first one wins.

**Current scope:** Freedom Accelerator first. Then SSA. Then Empire. Shared logic should eventually live in a sub-zap.

---

## Zapier MCP capabilities (FYI)

The connected Zapier MCP (29 apps) can:
- Run individual actions (Whop, Close, Airtable, Slack, Gmail, Google Docs/Sheets/Drive, Heartbeat, Stripe, Zoom, ActiveCampaign, Pipedrive, etc.)
- List / discover / enable actions
- Create + run "Zapier Skills" (saved multi-step workflows callable from chat)

It **cannot** read, edit, or trigger Zaps in the user's account. It cannot fetch a Zap by editor URL. So all rebuilds happen by giving Tyler step-by-step build instructions to apply in the Zapier UI.

---

## Original Freedom Accelerator Zap (what's deployed today)

Reconstructed from screenshots:

1. Whop — Payment Succeeded
2. Filter — Only Freedom Accelerator
3. Code (JS) — Normalize name
4. Slack — Alert Tyler privately
5. Formatter — Date → MMMM YYYY
6. Airtable — Create or Update Record (Leads table) · sets stage = "Won - New Client", tags = "Freedom Accelerator, SFA"
7. Sub-Zap — Find Client (fuzzy match by name/email variants, returns Airtable Client ID)
8. Airtable — Find Client (returned record)

Branches off step 8:

**SUCCESS (client exists):**
- 12. Airtable — Find Previous SSA Payment *(named "SSA" but in Freedom zap — likely leftover)*
  - **SUCCESS (prior payment):**
    - 18. Formatter — Utilities
    - 19. Formatter — Add 1 to previous payment count
    - 20. Formatter — Utilities
    - 21. Airtable — Create New Payment
    - 22. Slack — Private channel msg ("UPGRADE")
  - **ERROR (no prior payment):**
    - 13. Airtable — Create New Payment
    - 14. Airtable — Update Client
    - 15. Formatter — Utilities (pick first email from Whop's two)
    - 16. Heartbeat — Invite Users
    - 17. Email by Zapier — Send welcome email

**ERROR (no client):**
- 9. Airtable — Create Client
- 10. Airtable — Create Payment Records
- 11. Slack — Private channel msg

---

## Critique already delivered (8 issues)

1. **Step 6 runs before find-client.** Creating/updating a Lead by Whop email *before* the fuzzy find means a buyer who exists under a different email gets a duplicate Lead row, then step 8 finds the old client. **Find first, write second.**

2. **Step 12 is named "Find Previous SSA Payment" inside the Freedom zap.** Either copy-paste leftover or wrong logic. Freedom is bottom of the ladder; "previous payment" here should mean any prior purchase (low-ticket), and to detect rebills you'd look for prior *Freedom* payments, not SSA.

3. **Heartbeat invite + welcome email only fire in ONE branch** — the "client exists, no previous payment" path. Brand-new clients (path 9–11) get *no* Heartbeat invite, *no* welcome email. Three of the four cases are missing onboarding.

4. **"Pick first email" Formatter (step 15) is buried in one branch.** Should run at the top, before step 6's email-keyed write.

5. **No idempotency.** Whop retries / replays will create duplicate Payment rows and double-Slack. Need to guard on Whop transaction ID.

6. **"Add 1 to previous payment" (step 19) is fragile.** Race conditions with concurrent payments. Let Airtable derive count via `COUNT()` rollup of linked Payment records.

7. **Three separate Slack messages from three branches.** Step 4 (always), step 11, step 22. Consolidate to one summary at the end.

8. **Leads vs Clients table ambiguity.** Step 6 writes "Leads"; steps 8/9 read/write "Client." If they're the same table with a stage field, naming is misleading. If different tables, two sources of truth.

---

## Rebuilt Freedom Accelerator Zap (the design)

```
1. Whop — Payment Succeeded                      (trigger)
2. Filter — Only Freedom Accelerator
3. Code (JS) — Normalize all inputs              (firstName, lastName, email, txnKey, monthYear)
4. Airtable — Find Payment by Txn ID             (idempotency guard)
5. Filter — Stop if duplicate
6. Sub-Zap — Find Client (fuzzy)                 (returns clientId or empty)
7. Paths
   ├── PATH A: clientId Exists
   │   A1. Airtable — Update Client (stage, tags, current product, last payment month)
   │   A2. Airtable — Create Payment (linked to clientId)
   │   A3. Airtable — Find Prior Freedom Payments (other than this one)
   │   A4. Paths
   │       ├── A-i. First Freedom (A3 empty): A5 + A6 + A7-new
   │       │   A5. Heartbeat — Invite User
   │       │   A6. Email — Welcome
   │       │   A7. Slack — "🎉 NEW Freedom (upgrade from low-ticket)"
   │       └── A-ii. Rebill (A3 has record): A7-rebill only
   │           A7. Slack — "🔁 Freedom rebill"
   └── PATH B: clientId Does Not Exist
       B1. Airtable — Create Client (stage, tags, current product, source)
       B2. Airtable — Create Payment (linked to B1.id)
       B3. Heartbeat — Invite User
       B4. Email — Welcome
       B5. Slack — "🎉 NEW Freedom client"
```

### Key node settings

**3. Code by Zapier — Normalize (JavaScript)**

Inputs:
- `rawName` = `{{1.user_name}}`
- `rawEmails` = `{{1.email}}`
- `txnId` = `{{1.transaction_id}}`
- `paymentDate` = `{{1.created_at}}`

```javascript
const titleCase = s => s.toLowerCase().replace(/\b\w/g, c => c.toUpperCase());
const raw = (inputData.rawName || '').trim();
const parts = raw.split(/\s+/);
const firstName = parts[0] ? titleCase(parts[0]) : '';
const lastName  = parts.slice(1).length ? titleCase(parts.slice(1).join(' ')) : '';

const email = (inputData.rawEmails || '')
  .split(/[,;\s]+/)[0]
  .toLowerCase()
  .trim();

const txnKey = `freedom_${inputData.txnId}`;

const d = new Date(inputData.paymentDate);
const monthYear = d.toLocaleDateString('en-US', { month: 'long', year: 'numeric' });

output = { firstName, lastName, email, txnKey, monthYear };
```

**4. Airtable — Find Payment by Txn ID**
- Base: Sales Savage · Table: Payments
- Search field: `Whop Transaction ID` · value: `{{3.txnKey}}` · limit 1

**5. Filter** — continue only if `{{4.id}}` Does Not Exist

**6. Sub-Zap** — existing fuzzy-find sub-zap; ensure it returns empty string (not null) when no match

**7. Paths** — A: `{{6.clientId}}` Exists / B: `{{6.clientId}}` Does Not Exist

**A1. Update Client** — Record ID `{{6.clientId}}`, Stage `Won - New Client`, Tags `Freedom Accelerator, SFA` (append), Current Product `Freedom Accelerator`, Last Payment Month `{{3.monthYear}}`

**A2. Create Payment** — Whop Txn ID `{{3.txnKey}}`, Client `{{6.clientId}}`, Product `Freedom Accelerator`, Amount `{{1.amount}}`, Payment Month `{{3.monthYear}}`

**A3. Find Prior Freedom Payments** — formula:
```
AND(
  {Client} = "{{6.clientId}}",
  {Product} = "Freedom Accelerator",
  {Whop Transaction ID} != "{{3.txnKey}}"
)
```

**A4. Paths** — A-i: `{{A3.id}}` Does Not Exist (first Freedom) / A-ii: `{{A3.id}}` Exists (rebill)

**B1. Create Client** — First/Last/Email from step 3, Stage `Won - New Client`, Tags `Freedom Accelerator, SFA`, Current Product `Freedom Accelerator`, Source `Whop`

**B2. Create Payment** — Whop Txn ID `{{3.txnKey}}`, Client `{{B1.id}}`, Product `Freedom Accelerator`, Amount, Payment Month

### Convergence note

Zapier Paths don't reconverge. Heartbeat / Welcome / Slack are duplicated in A-i and B by design. To DRY this up, wrap them in a `Process Customer` sub-zap and call from both paths.

---

## Open questions (not yet answered)

1. Is **"Leads"** the same Airtable table as **"Clients"** with a stage field, or two separate tables?
2. What field on the Whop trigger holds the **unique transaction ID**?
3. **One Slack channel** for all Freedom events, or separate channels per outcome (new / upgrade / rebill)?
4. Should **rebills** also send a "thanks for renewing" email, or stay silent?

These change a couple of node mappings — answer before final build.

---

## Master-engineer principles (apply across all three zaps)

1. **One thin entry zap per product** (filter + product config), all calling a shared "Process Whop Payment" sub-zap. Fix bugs once.
2. **Parameterize product config** — pass `product_name`, `tags`, `prospect_stage`, `welcome_template`, `heartbeat_group` into the sub-zap.
3. **Idempotency** via Airtable lookup on Whop transaction ID, or Storage by Zapier.
4. **Derive, don't store** — payment count = Airtable rollup; "is upgrade?" = formula field.
5. **Run log table** — every zap run writes a row (timestamp, txn ID, product, email, outcomes). Debugging becomes "open Airtable."
6. **Real failure paths** — Heartbeat fail → Slack alert with retry. No silent half-onboarded customers.
7. **Naming hygiene** — no "Find Previous SSA Payment" inside the Freedom zap.

---

## Roadmap

1. ✅ Critique current Freedom flow
2. ✅ Design rebuilt Freedom flow + node settings
3. ✅ Visual artifact (`freedom-accelerator-zap.html` in this branch)
4. ⏳ **NEXT:** Tyler answers the 4 open questions, then builds Freedom in Zapier
5. ⏳ Repeat for SSA (will overlap heavily — extract shared sub-zap once SSA design exists)
6. ⏳ Repeat for Empire (upsell logic — likely simpler, no fuzzy find since Empire buyer already exists as SSA client)
7. ⏳ Refactor: pull common steps into `Process Whop Payment` sub-zap, parameterize per product
8. ⏳ Add run-log table + real failure paths

---

## Files in this branch

- `freedom-accelerator-zap.html` — visual diagram + per-node settings, open in any browser
- `CONTEXT.md` — this file

## How to resume in local Claude Code

```bash
cd ~/path/to/Savage
git fetch origin claude/check-zapier-mcp-access-pzNRP
git checkout claude/check-zapier-mcp-access-pzNRP
open freedom-accelerator-zap.html       # visual reference
claude                                  # then: "read CONTEXT.md, continue from roadmap step 4"
```
