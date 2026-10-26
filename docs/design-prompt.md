# Claude Design prompt — Dayline

Copy everything inside the block below into Claude Design.

```
Design a personal finance app called "Dayline" for iOS (iPhone) and macOS,
built in SwiftUI. Single user. Follow Apple Human Interface Guidelines, native
SF Pro / SF Symbols. Dark mode first (see VISUAL DIRECTION), light mode secondary. Primary currency Georgian Lari
(₾), secondary US Dollar ($). Use realistic sample data below. Tone: calm,
focused, encouraging — not gamified. Money numbers are the hero: large,
tabular figures.

CORE CONCEPT
The budget runs per "pay period" (from the day salary arrives to the day before
the next salary), NOT per calendar month. Salary minus fixed bills minus savings
= free money, split into a DAILY LIMIT. Under-spending a day first repays any
earlier over-spending, then goes to savings. Over-spending is spread evenly over
the remaining days, so the limit for the rest of the period shrinks.

SAMPLE DATA
- Salary 6,000 ₾ · Fixed bills 3,000 ₾ · Savings goal 2,200 ₾
- Free money 800 ₾ · Period Oct 1 – Oct 31 (31 days) · Daily limit 25.81 ₾
- Today: day 27 of 31, spent 12.50 ₾ today, saved so far this period 412.30 ₾
- Bills: Rent 1,200 · Bank loan 900 · Pet care 300 · Cleaner 120 ·
  Internet & phone 80 · Course fee 200 · Debts to friends: Friend A 100,
  Friend B 50, Friend C 50
- Categories: Transport/Taxi, Entertainment, Clothing, Health/Pharmacy, Home,
  Gifts, Other (each with an icon + color)
- Stash boxes (custom savings goals, each typed TRAVEL or SAVING):
  · "Japan trip" — TRAVEL, 1,000 $/month, balance 3,000 $, needs 4,000 $ by 15 Mar 2027
  · "Emergency fund" — SAVING (protected), 500 ₾/month, balance 2,400 ₾, target 10,000 ₾
- Bank loan: Example Bank, 12,000 ₾ remaining, 18% annual, 900 ₾/month
- Inbox: 3 auto-captured payments waiting for review (from Apple Pay and
  bank SMS), e.g. "Wolt — 24.90 ₾ · suggested: Other", "Bolt — 8.40 ₾ ·
  suggested: Transport/Taxi", "Transfer 1,200.00 ₾ · matches bill: Rent"

SCREENS (design iPhone versions first, then macOS)

1. TODAY (home)
   - Big ring/gauge: "Left today: 13.31 ₾ of 25.81 ₾"
   - Status line: ✅ "On track — 412.30 ₾ added to savings this period" or the
     warning variant: ⚠️ "Over by 9 ₾ — pay back over: [1 day] [3 days]
     [rest of period]"
   - Recovery state: ring shows reduced limit "23.00 ₾ today (normally
     26.00)", caption "Recovering from 9 ₾ overspend — back to 26 ₾ on Thu",
     small payback progress bar (6 of 9 ₾ repaid)
   - Period strip: "Day 27 of 31 · Salary expected Fri 31 Oct (estimate)" —
     the expected payday looks dashed/outlined because it's not confirmed
   - Card: "Did your salary arrive?" with buttons [Yes, today] [Another day…]
     [Not yet] (shown only near expected payday)
   - Bills summary card: "6 of 9 paid · 1,300 ₾ left to pay" with progress bar
   - Inbox card: "3 payments to review" (auto-captured, one tap to confirm)
   - Stash mini-card: "Japan trip 3,000 $ / 4,000 $ · on track"
   - Unconfirmed days chip: "2 days without entries — confirm?" (days with no
     logged expenses must not silently count as savings)
   - Prominent "+" button to add expense

2. ADD EXPENSE (sheet) — must take under 5 seconds
   - Amount first with large numeric keypad, ₾/$ toggle
   - Horizontal category chips, optional note, date defaults to Today
   - Secondary action "No spending today"

3. SALARY / PAYDAY CONFIRMATION FLOW
   - Date picker calendar highlighting predicted payday (dashed), weekends
     dimmed, with a short explanation "Usually the 1st; if weekend, last Friday"
   - Amount received field pre-filled with 6,000 ₾
   - Summary of the closing period (saved, over-spend, unpaid bills) and choice
     to carry over leftover over-spend

4. PERIOD (daily breakdown)
   - Bar chart: daily spending vs limit line, bars colored under/over
   - List rows per day: date · limit · spent · +saved / −debt · adjusted limit
   - Unconfirmed past days visually distinct
   - Category totals for the period (donut or horizontal bars)

5. BILLS CHECKLIST
   - Grouped: Due / Paid / Overdue. Each row: icon, name, amount, due date,
     linked loan/debt balance underneath
   - Swipe right = mark paid; tap = edit actual amount/date
   - Header total "Paid 1,700 ₾ · Remaining 1,300 ₾"

6. LOANS & DEBTS
   - Two sections: Bank loans / Personal debts
   - Loan card: lender, remaining balance, rate, monthly payment, payoff date,
     progress bar
   - Loan detail: amortization schedule, payment history, documents
   - Upload loan PDF flow: drop zone / Files picker → "Reading document with
     AI…" progress → REVIEW FORM showing extracted fields (bank, contract #,
     principal, rate, term, monthly payment, schedule) each editable, with
     low-confidence fields highlighted → Save
   - DUPLICATE DETECTED dialog: "This looks like your existing Example Bank loan
     (contract #…)" [Update existing] [Keep as separate] [Cancel]

7. STASH BOXES (custom savings goals)
   - Overview: total stashed, split bar Travel vs Saving
   - Box cards with type badge (TRAVEL = plane icon, SAVING = shield/lock icon
     with "Protected" label), balance / target, monthly contribution, target
     date, "On track" or "Behind by 300 $"
   - Box detail: projection chart (balance over months vs target line and
     target date marker), movement history (contributions, withdrawals)
   - Create/edit box form: name, type segmented control (Travel / Saving),
     currency, monthly amount from salary, target amount, target date
     (required for Travel), "Protected from loan advisor" toggle (Saving)

8. INBOX (auto-captured payments)
   - List of pending payments captured from Apple Pay, bank SMS and statement
     uploads; source icon on each row
   - Each row: merchant, amount, date, suggested category chip, optional
     "Matches bill: Rent → mark paid?" pill
   - Swipe right = confirm, swipe left = ignore; tap to edit; "Confirm all"
   - Upload statement (PDF/CSV) button with "12 new · 8 already captured"
     dedup result
   - Setup screen "Watch Apple Pay payments": 3 illustrated steps to create
     the Shortcuts automation (When I tap any card → Log Payment in
     Dayline → Run Immediately), a "Test it" state that turns green after
     the first captured payment, and a second section for bank SMS forwarding
     with a copyable token
   - Inbox row variant for an Apple Pay capture with missing amount:
     "Starbucks · amount needed" with inline amount field
   - Status badges on rows: Captured (grey), Verified (green),
     Declined (red, struck-through amount), Refunded (blue)

8b. PAYMENT CONFIRMATION (from notification)
   - iOS lock-screen notification mock: "Starbucks — 12.50 ₾ · Visa
     ••1234 · Was this payment successful?" with expanded actions
     [✅ Paid] [❌ Declined]
   - Confirm Payment screen opened by tapping the notification: large amount
     (editable), merchant, card, time, suggested category chips, note field,
     primary button "Paid", secondary "Declined", text button "Not an expense"
   - Missing-amount variant: amount field empty and focused with keypad up
   - Resolved toast: "Saved · 13.31 ₾ left today"

9. AI ADVISOR
   - Recommendation card: "You can put 2,000 $ toward your Example Bank loan" with
     reasoning: "Japan trip box will reach 6,000 $ by March; the trip needs
     4,000 $. Emergency fund is protected and untouched. Paying 2,000 $ saves
     ~X ₾ interest and ends the loan 4 months earlier."
   - Clear breakdown table of numbers (computed by the app, not guessed):
     per box — projected, needed, surplus
   - Variant when nothing is recommended: "All of your Japan trip money is
     needed — no extra payment suggested."
   - Buttons [Plan this payment] [Dismiss]

10. HISTORY
   - Past periods list with saved / overspent / total spent, month comparison
     chart, category trends

11. SETTINGS
   - Payday prediction rule, default salary, categories editor, bills editor,
     currencies & exchange rate, capture devices/tokens, import from Excel

PLATFORM SPECIFICS
- iPhone: tab bar — Today, Inbox, Bills, Money (Loans + Stash boxes), More.
- macOS: NavigationSplitView with sidebar (same sections), Today as a
  multi-column dashboard, toolbar "+" and ⌘N for new expense, drag-and-drop
  PDF onto Loans.

LOGO & APP ICON
- Design a distinctive, premium logo for "Dayline": a monogram or symbol
  that merges a diary/journal (bookmark ribbon, page fold, or bound spine)
  with money flow (coin, ascending line, or ₾ hint) — simple enough to read at
  29 pt, no text inside the icon.
- Deliver: iOS app icon (1024×1024, dark, default + tinted variants for iOS
  18), macOS app icon (squircle with depth), monochrome glyph for menu bar /
  notifications, and a horizontal wordmark lockup ("Dayline" in a refined
  sans or serif-sans pairing) for the splash/onboarding screen.
- Show it in context: home screen, notification, splash screen.

VISUAL DIRECTION — DARK FIRST, PRETTY
- Dark mode is the primary design; light mode is secondary (provide it but
  dark gets the polish).
- Base: deep near-black with a cool tint (e.g. #0B0D12 background, #14171F
  cards, #1C2029 elevated), subtle 1px hairline borders (#FFFFFF at 6–8%),
  soft inner glow on hero cards — never flat pure black.
- Accent: one signature color — a rich emerald-to-teal gradient
  (#10B981 → #14B8A6) used for the daily-limit ring, primary buttons and the
  logo. Use it sparingly so it feels luxurious.
- Semantic colors tuned for dark backgrounds: green = saved / within limit,
  amber (#F5A524) = over but recoverable, red (#F43F5E) = budget exhausted /
  overdue, blue (#60A5FA) = refunded / info. Text: #F5F7FA primary, #9AA3B2
  secondary.
- Subtle depth: glassy translucent tab bar / sidebar (material blur), gentle
  gradients on the hero ring, soft shadows, smooth spring animations for the
  ring filling and status changes.
- Rounded cards (20 pt radius), generous spacing, large tabular numerals
  (SF Pro Rounded or SF Pro Display for the big numbers), minimal decoration.
- Show empty states, loading state for AI parsing, and error state for an
  unreadable PDF.
- All UI text in English.
```
