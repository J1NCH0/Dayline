# Dayline — Requirements & Analysis

> Personal expense manager for one user (me). Replaces the spreadsheet
> (Excel). Status: **draft v0.2** — 2026-10-27.

---

## 1. Stack

| Layer | Choice | Notes |
|---|---|---|
| Client | **SwiftUI multiplatform** (iOS 17+ / macOS 14+) | One codebase, two targets. SwiftData as local cache for offline entry. |
| Backend | **NestJS** (TypeScript) | REST API, JWT auth (single user, but still authenticated — it holds financial data). |
| ORM | **Prisma** | Schema + migrations. |
| DB | **PostgreSQL** | Money stored as `Decimal(14,2)` — never floats. |
| AI | **MiniMax API** | `MinimaxService` in NestJS; base URL, API key and model id from env (`MINIMAX_API_KEY`, `MINIMAX_MODEL`). |
| Files | Local disk volume on the server for uploaded PDFs | |
| Hosting | **Hetzner** server running **k3s** (Kubernetes) | `api` Deployment (NestJS) + Service + Ingress (Traefik, k3s default) with cert-manager TLS; PostgreSQL as StatefulSet (or CloudNativePG operator) on a PVC; uploaded PDFs on a PVC; secrets (`DATABASE_URL`, `MINIMAX_API_KEY`, `JWT_SECRET`) as k8s Secrets; Prisma migrations run as a Job / init container on deploy; nightly `pg_dump` CronJob to Hetzner Storage Box. |
| Language | **English only** | No localization layer needed. |

**Feasibility:** yes, all of this is standard. The only parts that need
care are (a) offline-first sync between iPhone and Mac, and (b) getting
reliable structured data out of bank PDFs via the AI.

---

## 2. What the current spreadsheet does (baseline to keep)

The spreadsheet is a **pay-period daily-limit budget**:

1. **Inputs:** net salary (6,000 ₾), extra income, payday day-of-month,
   monthly savings goal (2,200 ₾), list of fixed monthly costs (3,000 ₾:
   rent, bank loan, pet care, cleaner, internet, course fee, 3 personal debts).
2. **Pay period** = last payday → day before next payday.
3. **Free money** = income − fixed − savings goal (800 ₾).
4. **Daily limit** = free money ÷ days in period (≈25.81 ₾/day).
5. **Expense log:** date, category (7 fixed categories), description, amount.
6. **Daily roll-forward (core rule — must be preserved exactly):**
   ```
   balance_today = debt_yesterday (≤ 0) + (limit − spent_today)
   if balance_today ≥ 0 → saved_today = balance_today, debt = 0
   else                 → saved_today = 0,             debt = balance_today
   adjusted_limit = limit + debt / remaining_days
   ```
   Under-spending first repays earlier over-spending, then counts as savings.
   Over-spending is spread evenly across the remaining days.
   *(App keeps the repay-first rule but changes the spreading — see F6.1:
   overspend is repaid over a short recovery window, then the limit returns
   to normal. "Rest of period" remains available as an option.)*
7. Status message: ✅ in limit / ⚠️ over, here's your new daily limit / ⛔ budget exhausted.

### Weaknesses found (the app must fix these)

| # | Problem | Fix in app |
|---|---|---|
| W1 | A day with no entries counts as "spent 0" → fake savings (sheet currently shows 671 ₾ "saved" with zero expenses logged). | Each past day is **unconfirmed** until I log something or tap "No spending today". Unconfirmed days are excluded from savings and highlighted. |
| W2 | Payday is a fixed day-of-month. | Payday is an **entered date per period** (see F1). |
| W3 | Over-spend debt silently vanishes at period end. | Period close shows the result; leftover debt can be carried into the next period (choice at close). |
| W4 | Personal debts treated as permanent fixed costs; no remaining balance. | Debts are entities with balance, payments, and an end (F3). |
| W5 | No history — only the current period. | All periods kept; history & comparisons. |
| W6 | "Savings" is theoretical, never reconciled. | Savings goals with actual recorded transfers (F4). |
| W7 | Payday 29–31 breaks date math in short months. | Disappears with explicit payday dates. |

---

## 3. Feature requirements

### F1 — Pay periods with a variable payday

**Problem:** salary normally comes on the 1st, but if the 1st is Sat/Sun it
comes the last Friday before — and in practice it's unpredictable
(sometimes last Friday, sometimes first Monday). Cannot be hard-coded.

**Model:** a `PayPeriod` has an explicit `startDate` (the day salary
arrived) and `endDate` (the day before the next salary). The period's
length drives the daily limit.

**UX proposal ("predict, then confirm"):**
1. App **predicts** the next payday with a default rule: the 1st; if
   weekend → previous Friday. Rule is editable in settings.
2. The prediction is shown as *"Expected salary: Fri 31 Oct (estimate)"*
   with a dashed/outlined style — clearly not confirmed.
3. From 3 days before the predicted date, home screen shows a card:
   **"Did your salary arrive?"** → `Yes, today` / `Yes, on another day…` / `Not yet`.
4. Confirming creates the new period starting that day, auto-closes the
   previous one (end = day before), and asks for the **actual amount
   received** (pre-filled with last salary; bonuses/deductions vary).
5. If the period is still open past the predicted date, the daily limit
   is computed against the predicted end; when the real date is
   confirmed, everything is recalculated retroactively. Days beyond the
   planned end get a "stretch" warning: *"Salary is late — limit for
   extra days: X ₾"*.
6. A quick "Change payday" action in the period header to correct the
   date at any time.

Acceptance:
- Period length can be anything 25–40 days; limit recalculates.
- Changing a payday date recalculates all daily rows of both affected periods.

### F2 — Tracking payments (bank has no API / export integration)

**Problem:** the bank gives no API, so the app can't see payments by
itself. Manual typing alone gets skipped (that's why the spreadsheet has
zero entries). Solution = **three capture channels feeding one
"Inbox"**, from most automatic to least:

| # | Channel | How it works | Covers |
|---|---|---|---|
| A | **Apple Pay watcher** (iOS Shortcuts "Transaction" automation → app's App Intent) | When I tap a card in Wallet, the automation fires (set to *Run Immediately*) and passes amount + merchant + card to the app's `LogPaymentIntent`. The intent saves the item locally as `pending` and syncs it to the backend. No HTTP call or token in the Shortcut. | Contactless Apple Pay payments with iPhone |
| B | **Bank SMS trigger** (iOS Shortcuts "Message" automation, sender = bank) | Bank's transaction SMS is forwarded to `POST /capture/sms`; backend parses amount/merchant/date (regex per bank, MiniMax as fallback). | Physical card, online payments, transfers the bank notifies about |
| C | **Statement upload** (PDF/CSV, monthly) | Backend extracts rows (MiniMax), dedups against what A/B already captured (date + amount + merchant), adds the rest. | Anything A/B missed — the safety net |
| D | Manual entry | Quick add in the app. | Cash |

**Inbox (review queue):**
- Every captured item lands as `pending` with a **suggested category**
  (rule: merchant → last used category; fallback MiniMax).
- Suggested **bill match**: if amount ≈ a due bill (rent, loan, cleaner,
  debt to a friend), propose "Mark *Rent* as paid?".
- I confirm with one tap / swipe; edit or ignore otherwise.
- Apple Pay captures count in the daily-limit engine right away
  (provisionally, see status table below); SMS / statement items count
  once I confirm them.
- Dedup key: `source + externalId` if available, else
  `date + amount + normalized merchant` within ±1 day.

**Bills checklist** (recurring obligations):
- Every recurring obligation is a `Bill` with amount, due date rule, and
  optional link to a Loan / Debt / Goal.
- Each period auto-generates `BillInstance`s: `Due` / `Paid` / `Skipped`.
- Paid either by Inbox match or by manual tap.
- Home: *"Bills: 6 of 9 paid · 1,300 ₾ still to pay"*; local
  notification on due date if unpaid.

Acceptance:
- An Apple Pay purchase appears in Inbox within seconds without opening the app.
- A bank SMS appears in Inbox parsed correctly for my bank's format.
- Uploading a statement that overlaps captured items creates no duplicates.
- Unpaid bills from a closed period are flagged overdue in the next one.
- Paid amounts for loan/debt bills reduce that loan/debt's balance.

**Apple Pay — what is and isn't possible:**
- There is **no API** for a third-party app to read Wallet transactions of
  bank cards. Apple's `FinanceKit` only exposes Apple-issued accounts
  (Apple Card / Cash / Savings) in supported countries — not Georgian
  bank cards.
- The supported route is the **Shortcuts "Transaction" automation**
  (iOS 17+). The app ships an **App Intent** (`LogPaymentIntent`:
  amount, merchant, card) so the automation is: *When I tap any card →
  Log Payment in Dayline*. One-time setup, then fully automatic.
- Limits to verify in M4 with my real card: fires only for in-store
  contactless taps on the iPhone (online Apple Pay and Apple Watch
  payments may not trigger it); some cards may not pass the amount →
  then the Inbox item asks for the amount. Channels B (SMS) and C
  (statement) cover whatever A misses.

**Transaction status (the trigger does not report it):**
The Shortcuts trigger only passes card, merchant and amount at the moment
of the tap. It does **not** say approved / declined / pending, and it never
fires for later refunds or reversals. So the app tracks status itself:

| Status | Meaning | How it gets there |
|---|---|---|
| `captured` | Tap seen, not yet proven | Apple Pay trigger |
| `verified` | Bank confirmed the charge | Matching bank SMS (same amount, merchant ≈, ±10 min) or statement row |
| `declined` | Payment failed | Bank "declined" SMS, or I mark it, or no match on statement → asked to review |
| `refunded` | Money came back | Refund SMS / statement credit matched to the original |
| `ignored` | Not an expense | Me |

Rules:
- `captured` items **already count** toward today's limit (safer to
  assume the money is gone); `declined` / `refunded` are removed and the
  daily numbers recalculate.
- Two captures with the same card + amount + merchant within 2 minutes
  (re-tap after a failed attempt) → second one flagged *"Possible retry"*,
  first one suggested as declined.
- On statement upload, `captured` items with no matching row after the
  statement covers their date are listed as *"Not on statement — declined?"*.
- Inbox row shows a status badge (captured / verified / declined / refunded).

To verify in M4: whether the trigger fires on a declined tap at all.

**Confirm-by-notification flow (primary way to resolve `captured`):**
1. `LogPaymentIntent` saves the capture, then schedules a **local
   notification** right away (no APNs/server needed — it's created on
   the iPhone itself):
   > **Starbucks — 12.50 ₾** · Visa ••1234
   > Was this payment successful?
2. Notification has quick actions (long-press / expand), so it can be
   answered without opening the app:
   `✅ Paid` → status `verified (by me)` · `❌ Declined` → `declined`.
3. **Tapping** the notification deep-links (`dayline://capture/<id>`)
   to a **Confirm Payment** screen: amount (editable — fill in if the
   trigger didn't pass it), merchant, card, time, suggested category
   chips, optional note, big buttons **Paid** / **Declined** /
   **Not an expense**. Paid saves it as an expense in one tap.
4. Not answered → one reminder after 1 hour; still open items stay
   in the Inbox with a badge count on the app icon.
5. If a bank SMS verifies (or declines) the capture first, the
   notification is removed automatically and the item resolves itself.
6. Captures made on the iPhone also appear on the Mac's Inbox after sync
   and can be resolved there; resolving on either device clears the
   notification on the iPhone.

Status table addition: `verified` can come from the bank (SMS/statement)
**or from me** (`verifiedBy: bank | user`) — a statement match later can
still upgrade/overrule a user answer and flag a mismatch.

Security: the SMS channel posts to `POST /capture/sms` with a per-device
token stored in the Shortcut; rate-limited; HTTPS only. Channel A needs
no token (runs inside the app).

### F3 — Loans & debts

Two kinds:

| Kind | Source | Example |
|---|---|---|
| **Bank loan** | Uploaded loan PDF, parsed by AI | consumer loan / installment, 900 ₾/month |
| **Personal debt** | Entered manually | Friend A 100/mo, Friend B 50/mo… |

Both have: lender, principal, current balance, monthly payment,
interest rate (bank only), start/end date, payment history. Balance goes
down as F2 payments are recorded. A debt auto-archives when balance = 0
and its monthly bill disappears (→ more free money, shown as a win).

#### F3.1 Loan PDF upload + AI extraction
1. Upload PDF (iOS share sheet / Files, macOS drag-and-drop).
2. Backend extracts text (`pdf-parse`; OCR fallback if scanned) and
   sends it to MiniMax with a strict JSON schema:
   `bank, contractNumber, currency, principal, annualRate, effectiveRate,
   termMonths, startDate, monthlyPayment, schedule[] (date, principal,
   interest, total, remaining), earlyRepaymentFee`.
3. Result is shown as a **review form** — I confirm/edit before it's saved.
   AI output is never trusted blindly.

#### F3.2 Deduplication
A loan uploaded twice must be detected:
1. **Exact file:** SHA-256 of the PDF bytes → same file = instant
   "already uploaded".
2. **Same loan, different file** (e.g. new statement for same contract):
   match on `bank + contractNumber`; if no contract number, fuzzy match
   on `bank + principal + startDate (+ rate)`.
3. On match: *"This looks like your existing loan '…'. Update it with
   this document / Keep as separate loan / Cancel."* Updating replaces
   the schedule and keeps history (document versions).

DB: unique `(userId, fileSha256)` on `LoanDocument`; unique
`(userId, bank, contractNumber)` on `Loan` where contractNumber not null.

### F4 — Stash boxes (custom savings goals)

Nothing is hard-coded — I create as many stash boxes as I want. Each one:

| Field | Example |
|---|---|
| Name | "Japan trip", "Emergency fund" |
| **Type** | `TRAVEL` or `SAVING` |
| Currency | $ / ₾ |
| Monthly contribution (taken from salary each period) | 1,000 $ |
| Target amount (when / how much is needed) | 4,000 $ |
| Target date | 2027-03-15 (required for TRAVEL, optional for SAVING) |
| Current balance | 3,000 $ |

**Type difference:**
- **TRAVEL** — money is planned to be *spent* on the target date. After
  the trip, the box is closed (or rolled to the next trip) and any money
  left over becomes free.
- **SAVING** — money is *kept* (emergency fund, long-term savings). Has no
  spend date; balance just grows. Marked **protected** by default.

**Budget effect:** the sum of all active boxes' monthly contributions
replaces the spreadsheet's fixed "2,200 ₾ savings goal" in the formula:
`free money = income − fixed bills − Σ monthly contributions`.

Each period every box gets a contribution item in the Bills checklist →
mark as transferred (actual amount). Balance updates; I can also record
withdrawals (e.g. paying for flights).

Views: per box — balance vs. target, projected balance at target date,
"on track / behind by X" indicator. Overview — split of all stash money
into Travel vs. Saving.

Multi-currency: store original currency + exchange rate at entry
(manual rate, optional NBG rate fetch later); totals shown in ₾.

### F5 — AI loan-repayment advisor

**Rules that must be respected:**
- Money **needed** for a TRAVEL box is never touched.
- **SAVING** boxes are protected by default — the advisor ignores them
  unless I turn off "protected" on that box (then only the amount above
  its target counts as surplus).

**Logic (computed deterministically in NestJS — AI only explains):**
```
for each TRAVEL box:
  projected_at_date = balance + monthly_contribution × months_until_target_date
  needed            = target_amount × (1 + safety_buffer)   // buffer configurable, e.g. 10%
  surplus           = max(0, projected_at_date − needed)
  // only money already in the box can be moved now:
  available_now     = max(0, min(surplus, balance − max(0, needed − monthly_contribution × months_until_target_date)))
for each unprotected SAVING box:
  available_now     = max(0, balance − target_amount)
total_available = Σ available_now (converted to loan currency)
```
Example: 6 months × 1,000 $ = 6,000 $, trip needs 4,000 $ → surplus
2,000 $ → may suggest putting 2,000 $ toward a loan (as soon as that
money is actually in the box). If the trip needs ≥ 6,000 $ → suggest
nothing.

**Which loan to pay first:** highest effective interest rate first
(avalanche), adjusted for early-repayment fees; option to switch to
smallest-balance-first (snowball) for personal debts.

**Output:** recommendation card with amount, target loan, interest saved,
months shortened, and a short AI-written explanation. Numbers come from
the backend, never invented by the model. Always a suggestion — I
accept/dismiss; accepting creates a planned extra payment.

### F6 — Daily expense tracking (from spreadsheet, improved)
- Fast entry: amount first (numeric keypad), category chips, optional
  note, date defaults to today. Goal: under 5 seconds.
- Editable categories (icon + color); seed with the 7 from the spreadsheet.

#### F6.1 — Overspend recovery (limit drops, then returns to normal)

When I spend more than the daily limit, the extra is paid back by
**temporarily lowering** the limit on the next days. Once the overspend is
repaid, the limit goes **back to the normal value**.

```
base_limit        = free_money / days_in_period            // e.g. 26.00 ₾
overspend         = spent_today − today_limit               // only if > 0
recovery_days     = min(setting, days_left_in_period)      // setting default: 3
recovery_limit    = base_limit − overspend / recovery_days
after recovery    → limit = base_limit again
```

**Worked example** (base limit 26 ₾, 31-day period, recovery = 3 days):

| Day | Limit | Spent | Over / under | Overspend left | Note |
|---|---|---|---|---|---|
| 10 | 26.00 | **35.00** | −9.00 | 9.00 | ⚠️ Over by 9 ₾ |
| 11 | **23.00** | 23.00 | — | 6.00 | recovering (26 − 9/3) |
| 12 | **23.00** | 23.00 | — | 3.00 | recovering |
| 13 | **23.00** | 23.00 | — | 0.00 | ✅ recovered |
| 14 | **26.00** | … | | 0.00 | back to normal |

Same overspend with recovery = **rest of period** (spreadsheet
behaviour): 9 ₾ / 21 days left → **25.57 ₾/day** until the period ends.

Rules:
- **Recovery window** is a setting: `1–7 days` (default **3**) or
  `Rest of period`. Can be changed per overspend from the warning card
  ("Pay back over: 1 day · 3 days · rest of period").
- If I spend **less** than the recovery limit, the difference repays the
  overspend faster (recovery ends earlier). If I spend **more**, the new
  extra is added to the overspend and the recovery limit is recomputed
  from that day.
- A new overspend during recovery merges into the same open overspend;
  recovery limit recomputed over the window from the new day.
- Savings: a day only adds to savings when there's **no overspend left**
  (same as spreadsheet — underspending repays debt first).
- If the recovery limit would fall below a **minimum daily limit**
  (setting, default 30% of base, e.g. 7.80 ₾), the window is extended
  automatically until it fits; if even the rest of the period can't
  absorb it → ⛔ "Budget exhausted — X ₾ over for this period".
- Recovery never crosses into the next pay period; leftover overspend at
  period close is handled by the close-period choice (W3).
- Today screen while recovering: *"Recovering from 9 ₾ overspend — 23 ₾
  today, back to 26 ₾ on Thu"* with a small progress bar of the payback.
- Daily-limit engine exactly as in §2.6, plus W1 fix ("No spending today").
- Today screen: today's limit remaining, adjusted limit, saved this
  period, overspend debt, status message.
- Period view: daily table/chart (same columns as the დღიური sheet).
- History: past periods, per-category totals, trends.

### F7 — Platform & data
- iOS + macOS from one SwiftUI codebase; macOS gets a sidebar layout
  and keyboard shortcuts (⌘N new expense).
- Offline entry on phone; syncs to NestJS when online (client-generated
  UUIDs, `updatedAt` last-write-wins — single user, conflicts rare).
- Import initial settings from the xlsx (salary, fixed costs, categories).
- UI in English only; currency ₾ primary, $ secondary.

---

## 4. Draft data model (Prisma, outline)

```
User
PayPeriod      id, startDate, endDate?, plannedEndDate, salaryAmount, extraIncome, status(open|closed), carriedDebt
Category       id, name, icon, color, archived
Expense        id, periodId, date, categoryId, amount, currency, note
DayConfirmation date (marks "no spending today")
Bill           id, name, amount, dueRule, kind(fixed|loan|debt|savings), loanId?, debtId?, goalId?, active
BillInstance   id, billId, periodId, dueDate, status(due|paid|skipped), paidAmount, paidDate
Loan           id, kind(bank|personal), lender, contractNumber?, currency, principal, balance, annualRate?, monthlyPayment, startDate, endDate?, earlyRepaymentFee?
LoanDocument   id, loanId, fileSha256 (unique per user), storagePath, extractedJson, uploadedAt
LoanScheduleRow loanId, dueDate, principal, interest, total, remaining
StashBox       id, name, type(TRAVEL|SAVING), currency, monthlyContribution, targetAmount?, targetDate?, balance, protected, status(active|closed)
StashMovement  id, boxId, periodId?, kind(contribution|withdrawal), amount, currency, fxRate, date
CapturedTxn    id, source(wallet|sms|statement|manual), externalId?, rawText?, date, amount, currency, merchant, suggestedCategoryId?, suggestedBillInstanceId?, status(pending|confirmed|ignored), expenseId?
StatementUpload id, fileSha256 (unique per user), bank, periodFrom, periodTo, uploadedAt
DeviceToken    id, name, tokenHash, lastUsedAt     (for Shortcut capture endpoints)
MerchantRule   merchantPattern, categoryId         (learned from confirmations)
AiRecommendation id, createdAt, inputsSnapshot, suggestedAmount, loanId, explanation, status(new|accepted|dismissed)
```

---

## 5. Decisions log

| Date | Decision |
|---|---|
| 2026-10-27 | Stack: SwiftUI (iOS + macOS) + NestJS + Prisma + PostgreSQL. |
| 2026-10-27 | Hosting on existing Hetzner server running k3s (Kubernetes). |
| 2026-10-27 | Apple Pay capture via Shortcuts "Transaction" automation calling an in-app App Intent (no public Wallet API exists for bank cards). |
| 2026-10-27 | English-only UI. |
| 2026-10-27 | AI = MiniMax API, integrated directly (PDF text extracted on backend, then sent to MiniMax). |
| 2026-10-27 | Payment tracking = Wallet/SMS Shortcut capture + statement upload + manual, all through an Inbox (F2). |
| 2026-10-27 | Savings = custom stash boxes, type TRAVEL or SAVING; no fixed seasons (F4). |
| 2026-10-27 | Overspend recovery: limit lowered for a recovery window (default 3 days, configurable / rest of period), then returns to base limit (F6.1). |
| 2026-10-27 | Apple Pay captures trigger a local notification to confirm Paid / Declined; tap opens Confirm Payment screen (F2). |

## 6. Open questions

1. Which bank(s)? Needed to write the SMS parser and statement parser
   (sample SMS + a sample statement file).
2. Exchange rate for $ boxes: manual entry is the default — fetch NBG
   official rate automatically later?
3. Safety buffer for travel boxes — default 10%?

---

## 7. Milestones (suggested)

1. **M1 — Core budget:** periods with variable payday, expenses, daily
   engine, today screen, deploy to k3s on Hetzner. (Replaces the spreadsheet.)
2. **M2 — Bills checklist** + personal debts with balances.
3. **M3 — Stash boxes** (travel / saving).
4. **M4 — Capture Inbox:** Wallet + SMS Shortcuts, merchant rules.
5. **M5 — MiniMax:** loan PDF extraction + dedup, statement import.
6. **M6 — AI repayment advisor.**
7. **M7 — macOS polish, history & charts.**
