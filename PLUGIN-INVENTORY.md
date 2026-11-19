# IQU Registration plugin — system inventory for the Next.js rebuild

> **Source:** the `iqu-registration` WordPress plugin v3.7.19 (≈34,000 lines of PHP), read on 19 Nov 2026.
> **What this is:** the business rules, workflows and admin capabilities that already run IQU in production. The Next.js platform must keep every behaviour listed here, or record a deliberate decision to change it.
> **What this is not:** code to port. Never translate the PHP line by line. Rebuild the behaviour with the Next.js stack and the visual system in `DESIGN.md`.
> **Precedence:** this file describes how IQU works *today*. Where it conflicts with `README.md` (the UI spec, written Aug 2026, before tuition billing existed), this file wins for business rules and data, and the spec wins for page content and visual intent. Every known conflict is listed in §12.
> **Live status:** Stripe tuition billing has been in **Live mode since 16 Nov 2026** with real families. Treat billing as a running financial system, not a feature to redesign freely.

---

## 1. Programs and public forms

| Program | Form type(s) | Ages | Money | Shortcode / page today |
| --- | --- | --- | --- | --- |
| **Enroll for Free** (monthly classes) | `free` | all | Monthly tuition by course and days (§2), first month free for new public enrolments | `[iqu_registration_form]` |
| **Summer Program** | `summer_level1` (5–10), `summer_level2` (11–15) | 5–15 | One-time fee for the whole two-month program (§2.4) | `[iqu_summer_form]` |
| **Weekend Ilm Program** | `weekend_new`, `weekend_existing` | 5–12 | Free for everyone, no payment | `[iqu_weekend_form]` → `/weekend-program/` |

**Shared behaviour across all three forms**

- **Security layers, in order:** nonce, honeypot field, Google reCAPTCHA v3 (score threshold 0.5), server-side validation, output escaping.
- **Idempotent submit:** each enrolment carries a random `submit_id`. A repeat (second tap, network retry, return from Stripe) gets the first answer back for 30 minutes and never saves a second row or re-sends emails, Telegram or Meta events. While the first request is still running, a repeat gets "already being saved".
- **Rate limits:** a cap on accepted enrolments per IP per hour and per email, sized for a large family but too low for a flood. Keys are HMACs, so no personal data appears in key names.
- **Date of birth:** one Month | Day | Year control shared by all forms. "Today" is the site time zone. Someone born on 29 Feb has their birthday on 28 Feb in non-leap years. Older rows have an age only.
- **Phone:** international phone input with a country picker (intl-tel-input). Numbers are validated against the US/Canada NANP area-code list.
- **Re-enrolment:** a submission that matches an existing record (same program, same student name ignoring case/spaces/punctuation, same email) **updates that record** instead of creating a new one. The ID and original date are kept and `reenrolled_at` is set. If billing is still running, only contact and guardian details change; course, fee and billing are never disturbed. Different programs and siblings (same email, different name) stay separate. An archived match is restored.
- **US and Canada only (eligibility):**

  | Phone country | IP country (GeoLite2) | Country of residence | Result |
  | --- | --- | --- | --- |
  | US/CA | US/CA | US/CA or empty | allow |
  | US/CA | other | — | allow, flag `location` for review |
  | US/CA | — | not recognised | allow, flag `location` |
  | not US/CA (incl. +1 Caribbean area codes) | — | — | block |
  | — | — | clearly another country | block |

  - Modes (admin setting): `off` · `monitor` (log only, default) · `enforce`.
  - Blocked visitors see a friendly modal and can join the **waitlist** (§8).
  - Only 2-letter country codes are stored. No IP address is ever stored.
- **Meta Pixel + Conversions API:**
  - `Lead` fires at submit, browser and server, sharing one `event_id` so Meta deduplicates them.
  - `CompleteRegistration` / `Purchase` fire later, from the Stripe webhook.
  - An explicit "Deny" for marketing cookies stops all events (`marketing_consent` is stored on the row).
  - Names sent are the parent's. A minor's own name and any date of birth are never sent.
- **Emails on submit:** an admin notification and a registrant confirmation. The field list for email and Telegram comes from one shared definition, so both channels always show the same fields.

---

## 2. Pricing — the single source of truth

### 2.1 Model

`monthly fee = days per week × 4 weeks × per-class rate`. Session length does not change the price. The server **always** recalculates; an amount sent from the browser is never trusted.

| Course key | Label | Per-class rate | Minimum days/week | Coupon prefix |
| --- | --- | --- | --- | --- |
| `qaidah` | Qa'idah / Nazirah | $5 for 1–2 days · $4 for 3–7 days | 1 | `QAD` |
| `hifz` | Quran Hifz | $5 | **3** (Hifz does not progress on fewer) | `HFZ` |
| `arabic` | Arabic Language | $5 | 1 | `ARB` |
| `tarjumatul_quran` | Tarjumatul Quran | $5 | 1 | `TRJ` |
| `islamic_studies` | Islamic Studies | $5 for 1–2 days · $4 for 3–7 days | 1 | `ISL` |

Maximum is 7 days a week. Imports also accept aliases such as "Tarjumatul Qur'an", "tarjuma", "quran translation" and "islamic study".

### 2.2 Special discount (Qa'idah / Nazirah only, opt-in plan)

| Days/week | Standard monthly | Special monthly |
| --- | --- | --- |
| 3 | $48 | **$40** |
| 4 | $64 | **$45** |
| 5 | $80 | **$50** |
| 6 | $96 | **$70** |
| 7 | $112 | **$85** |

The difference is recorded in the discount ledger under the **Special** fund (§6).

### 2.3 Derived "from" prices for course cards

- Qa'idah / Nazirah: **from $20/month** (1 day/week)
- Hifz: **from $60/month** (3 days/week minimum)
- Arabic Language: **from $20/month**
- Tarjumatul Quran: **from $20/month**
- Islamic Studies: **from $20/month**
- Weekend Ilm Program: **free**
- Summer Program: **$40 one-time** (see §2.4)

Confirm these against the live settings before publishing. Show them with a scholarship note.

### 2.4 Summer one-time fee (admin settings, not hard-coded)

| Option on the form | Amount | Route |
| --- | --- | --- |
| Standard | setting `summer_fee`, currently **$40** | Stripe Checkout |
| Supported | setting `summer_fee_supported`, currently **$30** | Stripe Checkout |
| Existing student | complimentary | free, flagged `existing_student` |
| Flexible / help | — | `pending_review`, no payment |

- Enrolment completes **only** after Stripe confirms payment.
- An unpaid family gets a signed resume link and one reminder email after 24 hours.
- Payment route is an admin switch: Zeffy (legacy) or Stripe.

### 2.5 Rules

- **No sibling discount.** Every child pays the full fee.
- **USD only.**
- Admins can choose a course and day count from the fixed list. The price for that choice is still computed by the pricing rules.
- An **agreed fee** (negotiated) is billed as agreed. A lower agreed fee is stored as a discount and must be assigned a fund (Zakat or Special). Ticking "agreed fee" with no amount is an error, never $0.

### 2.6 Coupons

- **Format:** `{SHORT}-{YYMMDD}-{RAND4}-{VALUE}`. Examples:
  - `HFZ-260826-X7K2-80` — Hifz, 80% off
  - `QAD-260826-M4P9-100` — full scholarship
  - `ARB-260826-B2T7-F25` — $25 fixed off
  - `ALL-…` — applies to every course
- `RAND4` excludes 0/O and 1/I/L, so codes are easy to read aloud and type on WhatsApp.
- **Fields:** code, occasion name, course type, discount type (percent / fixed), value, expiry date, maximum uses, used count, status, description, created by.
- **Admin screens:** Create Coupon, and Coupon List with usage, totals and discount summaries.
- **Validation messages:** valid · expired · not applicable to this course · usage limit reached (`coupon_used_up`).
- **A coupon used at enrolment routes the registration to review** (`help`) instead of straight to Stripe. Coupon discounts count under the **Zakat** fund.

---

## 3. Registration record and lifecycle

- **Table:** `iqu_registrations`, 68 columns (§10).
- **Statuses:** `pending` · `confirmed` · `contacted` · `enrolled` (the active state) · `cancelled` · `card_pending` · `pending_review` · `expired` · `paid`.
- **Origin (`created_via`):**
  - `public` — website form.
  - `admin` — Billing → Add Existing Student.
  - `import` — CSV/XLSX import.
  - Admin and import rows **never** get a free month, Meta events, card reminders, the 7-day expiry or `card_pending`.
- **Review reasons:**
  - `help` — Zakat coupon or a $0 amount.
  - `existing_student`
  - `sibling` — the email already has a billing account.
  - `location` — eligibility flag.
  - `card_country`
  - `duplicate_subscription`
  - an amount mismatch from an old checkout page.

  Each reason has a label and an admin note.
- **Enroll for Free with payments on:** exceptions are saved as `pending_review` and never sent to Stripe. Everyone else is saved as `card_pending`, a billing account is created, and the family goes straight to Stripe Checkout to add a card or US bank account.
  - **Only the Stripe webhook** marks them `enrolled`, then sends the welcome email, the Telegram "✅ Enrolled" message and Meta `CompleteRegistration`.
  - Returning to the success page does not count as completion.
- **Archive, never delete billed people:**
  - **Archive** is allowed only with no active billing. It hides the row from every active list, count, birthday list and export, but reports and the ledger keep counting past months. **Restore** undoes it.
  - **Hard delete** is allowed only for a student who was never on a live billing record and has no Zakat/Special ledger rows. It is confirmed by typing the student's name.
- **Add Existing Student / Import:** for students who joined before the website form existed.
  - Import accepts `.csv` or `.xlsx`, up to 1 MB and 500 rows.
  - It works in two steps: upload and preview every row, then confirm.
  - Every row goes through the same validation and pricing as the single form, and is checked again at confirm.
  - Exact duplicates (same name and email) are refused.
  - Nothing is emailed or sent to Telegram.

---

## 4. Monthly tuition billing (Stripe)

### 4.1 Model

- **One family = one billing account** = one Stripe customer + one subscription.
- **Members** link each enrolment (child) to the account.
- A sibling **joins the existing account** and is never given a second card or second charge.
- Every billing row carries a `mode` column (`test` / `live`), so test and live data never mix.
- **Account statuses:** `not_sent` · `link_sent` · `free_month` · `waiting_first_charge` · `active` · `past_due` · `unpaid` · `paused` · `canceled` · `review` · `bank_verification` (US bank, waiting for microdeposits) · `ending` (stops at the end of the paid period).

### 4.2 Billing calendar

- **Everyone is billed on the 1st of the month.**
- **New public students** get a 30-day free first month. Their first charge is the first billing day on or after the end of those 30 days.
- **Admin-added and imported students** get no free month.
- **Adding a sibling to a running subscription** adds the child's line with no proration and nothing charged that day. If that child is first billed later, an hourly job adds the line just before that billing day, never early.

### 4.3 Family payment page

- **Address:** `/my-billing/?t=TOKEN`, a private link for each family.
- **Token security:**
  - The token is an HMAC of the account ID with a link secret.
  - Only its SHA-256 hash is stored, so a database leak alone cannot produce links.
  - Lookups use a constant-time compare.
  - "Reset link" invalidates the old one.
  - The secret can be rotated: a previous secret is still accepted for verification.
- **Page headers:** no-store, noindex, Referrer-Policy same-origin, frame DENY, nosniff, strict CSP. It is never cached.
- **Actions:** POST only, same-origin checked, and redirects go only to `checkout.stripe.com` or `billing.stripe.com`.
- **"Find my link":** a family can request their link by email. Requests are rate-limited per IP and per email, and the page never reveals whether an email is on file.
- **Payment methods:** card and **US bank account (ACH)**. The family changes their method later through the Stripe customer portal. Canadian bank debit (PAD) is postponed.

### 4.4 Sending links and chasing

- **Admin "Check and send" screen:** shows the amount, siblings and first charge date, then sends the link by email.
- **"Message" screen:** a WhatsApp-ready text with a **Copy** button and an **Open WhatsApp** button, plus resend email and reset link. The private link is visible only to admins.
- **Bulk send:** up to 25 families at once.
- **Reminders:** emails to the family **3 and 7 days** after the link was last sent, once each. Sending again restarts the count. After the 7-day reminder, the team gets a Telegram message.
- **Missed webhook safety net:** before any reminder, an open checkout page with no subscription is read back from Stripe. A completed page is then finished exactly as the webhook would finish it, only once.
- **Expiry:** a public enrolment with no card after 7 days expires.

### 4.5 Arrears (an earlier month owed)

- **The next billing day charges at most one earlier month together with the current one**, in a single payment, never on any other day.
- Older owed months go to the team on Telegram.
- Causes:
  - a card added after (or within 48 hours of) a current family's first billing day;
  - a failed payment that was never recovered (Stripe leaves the next invoice in draft).
- **Waive:** only named refund admins, and a reason is required.
- **Display:** an "Unpaid earlier months" card on the family page.

### 4.6 Stopping, refunds, receipts

- **Stop billing:** always at the **end of the period already paid**. There is no immediate cancellation anywhere, and it is never prorated.
  - **Resume** is possible until that date.
  - **Cancel enrolment** applies before a subscription exists: the open checkout page is expired.
  - **End scholarship** ends a full scholarship at the end of the month.
  - A reason is required for all of these.
- **Cancellation policy:** families must notify at least 7 days before the next charge. With under 7 days' notice the next month may still be charged, and classes continue that month. A **full refund within 48 hours** of a charge is allowed if no class has taken place that month.
- **Refunds:**
  - Only for named admin users (an env allow-list). Nobody can refund when the list is empty.
  - Manual only, with a reason.
  - Idempotent, so a double click is one refund.
  - Logged in the actions table and announced on Telegram.
- **Receipts:** an admin setting decides who sends them, Stripe (default) or IQU's own branded email.
  - IQU sends one receipt per paid invoice, guarded by an atomic claim so it is never sent twice.
  - A failed send is queued, retried once and reported on Telegram.

### 4.7 Stripe integration rules

- **Webhook signature:** HMAC-SHA256 over the raw body, constant-time compare. Requests older than 5 minutes are refused (replay protection).
- **Mode filter:** events from the other mode (test vs live) are ignored.
- **Deduplication:** each event ID is processed once. If processing fails, the record is removed so Stripe retries.
- **The webhook body only identifies the family.** Status, amounts and dates are always **read back from the Stripe API**, and the object must belong to that family's customer.
- **Events handled:**
  - `checkout.session.completed`, `checkout.session.async_payment_succeeded`, `checkout.session.async_payment_failed`
  - `customer.subscription.created`
  - `invoice.paid`, `invoice.payment_failed`, `invoice.payment_action_required`
  - `charge.refunded`, `charge.dispute.created`
  - `setup_intent.succeeded`, `setup_intent.setup_failed`
  - `payment_intent.processing`
  - payout events
- **"Sync from Stripe"** refreshes one family on demand.
- **Payment history:** one row per Stripe invoice. Card and bank details are stored as brand plus last four digits only. Receipt and invoice links are kept only when they point at Stripe domains.
- **Secrets** live only in server config (env), never in the database or the browser. A key whose mode does not match the site mode is refused.
- **Site mode switch (Test ↔ Live):**
  - Only named refund admins can use it.
  - Switching to Live requires a live restricted key, a webhook secret and a passing connection test. The admin types `LIVE`; switching back means typing `TEST`.
  - Page caches are cleared after a switch.
- **Payouts** are deliberately **monthly**. Do not change this.

### 4.8 Admin billing screens today

The **Billing** menu has these tabs:

- **Student Billing:** every monthly enrolment with its fee, billing status and actions.
- **Payments:** KPIs and charts for one month and earlier months. Every chart has a "View as table" option and an aria summary.
- **Payouts:** each bank deposit and the family payments inside it, checked as Gross − Fees − Refunds/Disputes = Net = payout.
- **Reports:** datasets `students`, `payments`, `reconciliation` (expected vs paid per family) and `not_set_up`, on screen and as CSV/XLSX.
- **Add Existing Student**
- **Import Students**
- **Settings:** keys, mode and test connection; enrolment switches; summer fees; who sends receipts.
- **Send Links:** bulk sending.

Each family page shows the full **month-by-month payment history**, unpaid earlier months, stop/resume, refund and sync.

---

## 5. Enrolment switches (admin settings)

| Setting | Values | Default |
| --- | --- | --- |
| Country check | `off` · `monitor` · `enforce` | monitor |
| Enroll for Free payment step | `off` · `on` | off |
| Summer payment | `zeffy` · `stripe` | zeffy |
| Summer fees | Standard / Supported amounts | $40 / $30 |

While the Stripe mode is **test**, payment steps run only for logged-in administrators, and every other visitor gets the form as before.

---

## 6. Zakat and discount ledger

- **Table:** `iqu_discount_ledger`, one row per **student × month × fund**.
- **Funds:**
  - `zakat` = full scholarship + percentage coupon + fixed-amount coupon.
  - `special` = the special-discount plans.
  - An agreed reduced fee and an admin discount count under whichever fund the admin chose (a "not set" fund counts nowhere).
- **The free first month counts as neither fund.**
- **Rows are written once a month and never recalculated**, so a later fee change never alters past months. A "Calculate past discounts" action previews first, then confirms.
- **Reporting:**
  - Total Zakat given and total Special discount for a range of months, plus CSV.
  - The **Zakat & Discount students** list includes everyone who ever had a ledger row, even archived or ended students. Per student: guardian, age/DOB, WhatsApp, program, kind, fund, monthly amount, first and last month, months counted, total given, status. It has filters, search and CSV.
- **This is an accounting record for Zakat.** It must stay exact and auditable.

---

## 7. Donations (Zeffy)

- **Table:** `iqu_zeffy_payments`, 37 columns. **Money is stored in cents**, never as a float.
- **Real time:** the Zeffy webhook `payment.completed`, verified by HMAC signature (`t=…,v1=…` over the raw bytes) with a tolerance window. It is idempotent on `payment_id`.
- **Backfill** walks all history once. **Reconcile** runs twice daily and re-polls the last 30 days to catch refunds, disputes and edits, because Zeffy has no webhook for those.
- **Admin dashboard:** 6 metric cards, 7 charts, filters and a CSV export.
- **This is the only record of giving.** Never create a second one.

---

## 8. Waitlist

- **Who:** families outside the US/Canada leave an email and optionally a name to hear when classes open in their country.
- **Endpoint:** `POST /wp-json/iqu/v1/waitlist`, protected by nonce and reCAPTCHA, rate-limited to 5 an hour per visitor (keyed by an IP HMAC, IP never stored).
- **Storage:** one row per email + program. Only the country code is stored.
- **Admin:** metric cards, filters (search, country, program, date range) and CSV export.

---

## 9. Notifications

### 9.1 Telegram (team)

- **Delivery:** non-blocking. Values are escaped for HTML mode. Credentials live in server config.
- **Events:**
  - New registration (real time)
  - New Zeffy donation (real time)
  - Refund or dispute found (twice daily)
- **Billing alerts:**
  - payment method added
  - link not used
  - site mode switched
  - arrears to team
  - month waived
  - sibling added / sibling line late / sibling line failed
  - amount mismatch
  - duplicate subscription
  - payment received / payment failed
  - receipt failed
  - bank not verified
  - payment disputed
  - payout paid / payout failed
  - billing ended
  - refund issued
  - needs contact
- **Daily summary at 21:00 Asia/Dhaka** (late morning in the US): registrations, donations and tuition. **Counts and totals only, never names.**
- Each billing alert identifies the family (guardian, students, IQU number, course, days, contact, card brand + last four). **Payout alerts carry totals only.**

### 9.2 Email (families and admin)

- Registration: admin notification and registrant confirmation.
- Billing and enrolment emails use **one formal layout**:
  - logo header, title, greeting, details table;
  - "What you need to do" numbered steps;
  - one primary button;
  - a "Good to know" list and a "Need help?" box;
  - sign-off and footer.
  - 600 px, table-based, with a hidden preheader and a matching plain-text part.
- Replies go to the team inbox. Site email is sent via SMTP.

### 9.3 WhatsApp (manual only)

- "Copy message" and "Open WhatsApp" buttons for payment links.
- **Birthday wishes:** version A goes to the student if 18 or older, version B to the parent/guardian if under 18. It opens WhatsApp with the text.
- **There is no WhatsApp API.**

---

## 10. Data model (current MySQL tables)

| Table | Columns | Holds |
| --- | --- | --- |
| `iqu_registrations` | 68 | Every enrolment. Key columns: form type, student and guardian, DOB, contact, schedule answers, course type, days, per-class rate, calculated amount, coupon, discount, special discount, agreed fee and its fund, Zakat declaration, status, review reason, country (IP), billing account, created_via, existing student, consent, archive and re-enrol dates, refunded amount |
| `iqu_waitlist` | 7 | Waitlist entries |
| `iqu_coupons` | 14 | Coupons |
| `iqu_discount_ledger` | 8 | Monthly Zakat/Special rows |
| `iqu_zeffy_payments` | 37 | Donations (cents) |
| `iqu_billing_accounts` | 36 | Families / Stripe subscriptions |
| `iqu_billing_members` | 8 | Enrolment ↔ account |
| `iqu_billing_events` | 9 | Stripe events seen (dedupe and audit, no payloads) |
| `iqu_billing_payments` | 22 | One row per invoice |
| `iqu_billing_actions` | 10 | Admin action log (refund, stop, waive …) |
| `iqu_billing_arrears` | 15 | Earlier months owed |

---

## 11. Cross-cutting rules the new platform must keep

**Money and secrets**
- Money is always recalculated on the server.
- Secrets live only in server env.
- Every write uses prepared queries and whitelisted fields.

**Exports**
- Every list exports as **CSV and Excel (.xlsx)** from one "Export" button.
- CSV is UTF-8 with BOM.
- **Formula injection guard:** cells starting with `= + - @ TAB CR` get a leading apostrophe, except plain numbers. The guard also applies after `;` for European Excel.
- Phone numbers stay as text, never shown as `1.47E+10`.

**Tables**
- One pager pattern everywhere: a "Showing 1–10 of 40" chip, a 10/20/30 page-size control, and numbered pages.
- Filters and search state live in the URL.

**Navigation and display**
- **"← Back" returns to the exact screen the admin came from:** the list, tab, filter, search and page, including after a save.
- Charts always offer "View as table" and an aria summary.
- One icon set: **Lucide** outline icons.
- Dates use the site time zone. The team works in **Asia/Dhaka**.

**Privacy**
- No IP addresses are stored. Only 2-letter country codes are kept.
- Telegram summaries carry no names. Payout alerts carry totals only.

**Approvals**
- Destructive or financial actions need a reason and a confirmation dialog. The highest-risk ones (refund, waive, mode switch) are limited to named admins.

---

## 12. Gaps and conflicts with the UI spec (`README.md`)

### 12.1 The spec has no screen for these (add them to the admin portal)

- **Billing area:**
  - Student Billing, family billing page, Payments, Payouts, Reports
  - Send link / Message (WhatsApp copy) / bulk send
  - Add Existing Student, Import Students
  - arrears and waive, stop/resume, refund, sync
  - site mode and billing settings
- **Waitlist**
- **Zakat & Discount ledger** and the Zakat & Discount students list
- **Archive / restore / hard-delete rules**
- **Enrolment switches** (country check, payment steps, summer fees)
- **Birthday list and wishes**
- **The family's private billing page** (`/my-billing/?t=…`). In the portal it becomes part of `/p/billing`, but the signed-link entry without login must remain.

### 12.2 The spec conflicts with how IQU works

- **Course catalogue:**
  - The spec lists 6 courses. The priced courses are Qa'idah/Nazirah (one course), Hifz, Arabic Language, **Tarjumatul Quran** and **Islamic Studies** (the last two are not in the spec).
  - Weekend is free. Summer is a one-time fee.
  - The marketing catalogue and the priced list must be reconciled.
- **Sibling discount:** the spec has one (§7.3, §7.11, §7.17, §7D.4). IQU has **none**.
- **Payment during enrolment:** the spec says "No payment is taken in this flow". IQU takes a card at Stripe Checkout after Enroll for Free, and the first month is free.
- **Enrolment states:** the spec's admin status flow (new → reviewing → teacher assigned → trial scheduled → enrolled → declined) does not match the real statuses (§3). Merge them deliberately.
- **Currency:** the spec mentions CAD. Tuition is USD only.
- **Fees step:** the spec's three fee choices (standard / reduced / Zakat) do not match the live model (course + days → price; coupon and agreed fee → review).
- **Messaging:** the spec lists WhatsApp as an automated channel. IQU uses WhatsApp manually only.

---

## 13. Rebuild constraints

1. **Billing is live.** Do not move Stripe billing, the webhook or the ledger off WordPress until the new implementation has run alongside the old one and matched it month for month. A Stripe webhook endpoint can have only one owner per event, so the cut-over needs its own plan.
2. **Build order that keeps risk low:**
   - public pages and the enrolment UI first, with WordPress staying the system of record behind them;
   - then the read-only portal views;
   - billing and admin money screens last.
3. **The behaviour in §1–§11 is the contract.** A new screen may look and flow better than the plugin's, but it must not lose a rule, a state, a safeguard or a report listed here.
