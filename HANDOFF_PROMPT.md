# ZIP ONE → Odoo Migration — Handoff Briefing

Paste everything below as the first message in a fresh Claude Code conversation
on the second laptop.

---

I'm migrating my business accounting into Odoo and need you to pick up work that was
started on another machine. Here's the full context.

## The business

Al-Tawakkal Impex / ZIP Motorcycle Oil & Spare Parts, Karachi, Pakistan. Motorcycle oil
and spare parts distribution. I'm the owner, not a developer — keep explanations plain
and tell me what to click rather than assuming I know the tooling.

## Three systems

1. **Manager** (manager.io) — the older accounting software. Being retired.
2. **ZIP ONE**, also called **"Naveed"** — Oracle 10g app running on the *other* laptop
   (Karachi). This is the live operational system: invoices and cash receipts are keyed
   in here daily. I do NOT have this machine with me.
3. **Odoo 17 Community**, self-hosted — `https://vohru-odoo.duckdns.org`, database `zip1`.
   Runs on Oracle Cloud behind a Cloudflare Tunnel.

**Important:** Odoo was built from **Manager's** data, not from ZIP ONE's. Manager and
ZIP ONE ran in parallel for a long time and drifted apart. So Odoo's ledgers inherit
Manager's version of history, which disagrees with ZIP ONE in both directions.

**Goal:** Odoo becomes the single system of record. ZIP ONE (and maybe Manager) stay
running as secondary for a while. Nothing gets posted to Odoo without my approval first.

## Where the data is

A private GitHub repo, refreshed automatically every night at 9 PM Karachi time by a
scheduled job on the other laptop:

    https://github.com/ahmedvohrra/zip1_naveed_daily_EXPORT

    git clone https://github.com/ahmedvohrra/zip1_naveed_daily_EXPORT.git

Contents — fresh Oracle/ZIP ONE extract at the top level:

| File | What it is |
|---|---|
| `invoices.csv` | 1,008 invoice headers with computed totals |
| `invoice_lines.csv` | 3,144 lines with product, qty, unit price, discount |
| `receipts.csv` | 2,193 cash receipts |
| `customers.csv` | 356 customers with balances and last-activity dates |
| `products.csv` | product master |
| `ledger_all.csv` | 3,201 unified transactions (invoices + receipts) |
| `manifest.txt` | row counts + latest dates — **check this first to confirm data is fresh** |

And `analysis/` holds the reconciliation already completed:

| File | What it is |
|---|---|
| `CUSTOMER_MAP.csv` | every customer, both balances, difference, classification, my own notes |
| `MISSING_invoices.csv` | 134 invoices in Oracle but not Odoo, PKR 5,407,829 |
| `BALANCE_COMPARISON.csv` | per-customer balance diff |
| `REVIEW_PAIRS.csv` | possible duplicate customers needing human review |
| `name_map.csv` | 214 Manager↔ZIP ONE name pairs with my hand-written notes |

If `manifest.txt` shows a stale date, the Karachi laptop is off or offline — tell me.

## Facts already verified (don't re-derive these)

**Oracle / ZIP ONE:**
- Invoice line amount = `QTY * UPRICE`. Confirmed against the ledger for several invoices.
- `TOTAL_VALUE` in `INV1S` is unreliable — null or wrong on many rows. Ignore it.
- **`DISC` is already deducted from `UPRICE`.** Never subtract it again. Verified:
  UPRICE 352.75 = list 415 − 15%. Total discount exposure is only 0.016% of sales.
- `QTYTYPE` is uniformly `'L'` — no pack/loose conversion needed.
- Oracle tables: `CUST`, `INV0S` (headers), `INV1S` (lines), `CASHR1` (receipts),
  `PROD` (products), `INVR0`/`INVR1` (returns).

**Odoo (from a backup taken 2026-08-02):**
- Odoo 17.0 **Community**, Postgres 14. No `sale` module — Invoicing only, no Sales Orders.
- Custom modules installed: `zip1_ledger_report`, `zip1_auto_reports`, `aging_report`,
  `financial_report`.
- Receivable account = **id 13, code 1121001** ("Receivable from Customers").
- Journals: INV (sale), BILL (purchase), MISC, BNK1, CSH1 (cash), plus 3 bank journals.
- 1,016 posted customer invoices + 25 credit notes, 295 partners, 2024-12-19 → 2026-07-12.
- **712 Odoo invoices carry `ref` = the Oracle invoice number, zero-padded** (`INV-0009`
  = Oracle INV-9). This is the join key between systems. July entries were keyed in by
  hand and have no ref.
- **Receipts have NO reference link at all.** Odoo has 1,295 customer payments vs Oracle's
  2,193 receipts. Transaction-level receipt matching is unreliable — work at customer
  totals instead. An earlier naive attempt claimed PKR 15.7M missing; that number is wrong.

## What the reconciliation found

- 256 of 295 Oracle customers matched into Odoo (101 exact name, 133 via the name map,
  22 fuzzy).
- **73 balances agree exactly.**
- **16 differ by exactly the amount Manager and ZIP ONE already differed** — pre-existing,
  already explained by my notes in `name_map.csv`.
- **167 are new discrepancies**, PKR 1,715,978 net.
- **134 invoices missing from Odoo**, PKR 5,407,829 — 112 of them dated July 2026.
- Customer names differ in spelling across all three systems. Some mappings in the
  original file are wrong (e.g. "Ali Break Shoe Qayyumabad" ↔ "Ali Auto Mehmodabad" are
  not the same shop).

## Business rules I've confirmed

- **`W/RIM` accounts are deliberately separate ledgers.** Separate route, separate
  salesman, lower margin — split out so payments come in faster. **Never merge them**
  with the parent shop, even though the names look like duplicates.
- The **50000-number invoice series** is a manual/miscellaneous book, used for
  `RETAIL SALE` and some engineering/supplier accounts. Not normal customer sales.
- Each salesman has his own customers/routes. This is set up in Odoo but may be
  incomplete.

## Open questions to resolve

1. **`ZIP` account (ccode 14)** — Oracle shows PKR 55,905, Odoo shows PKR 3,417,290.
   A 3.36M gap, the single largest discrepancy in the books. Unexplained. Nothing should
   be posted until this is understood.
2. **`RETAIL SALE` (ccode 1)** — I don't think we have retail sales. Oracle holds 12
   invoices totalling PKR 145,825 with zero payments ever. `INV-50017` (2026-07-15) is
   124,800 of that and breaks the pattern of the other 11 (33–4,020).
3. **7 invoices flagged `INV0S.RETURN2='Y'`** on engineering/supplier accounts (KAIZEN,
   ZHOULI ENGINEERING, HAFEEZ ENGINEERING WORKS LHR). Unclear whether these are sales
   returns, purchases, or job-work. Probably shouldn't be customer invoices at all.
4. **8 receipts carrying remarks are not really cash** — "DISCOUNT EXP", "jenretor
   reparing exp", "brake shoe rate difrens", "RCV ZIP1 PAYBLE". Total PKR 108,914. If
   posted as cash receipts the bank balance inflates; they likely belong as credit notes
   or journal entries.

## What I want done next

1. Redo the **receipt reconciliation at customer-total level**, since transaction-level
   matching doesn't work without a reference.
2. Resolve the remaining **customer identity questions** — which Odoo-only partners are
   genuinely new versus spelling variants.
3. Produce a **line-by-line approval sheet**: exactly what would be created in Odoo, split
   into "safe to post" and "needs my review".
4. I approve it. Only then does anything get posted.

## One constraint to know upfront

Claude cannot log into Odoo or write credentials into files — that restriction holds
regardless of my authorising it, so don't spend time trying to work around it. All the
analysis can be done from exported files and database backups. For the posting step,
either generate import files I drag into Odoo myself, or write a script that reads an
API key I paste into a local config file. Tell me which you'd prefer when we get there.

To read Odoo's current state, the cleanest route is a fresh backup:
`https://vohru-odoo.duckdns.org/web/database/manager` → Backup → format **zip**. The
`dump.sql` inside is a plain Postgres dump and can be read directly — that's how the
analysis above was done, no login involved.

Start by cloning the repo, checking `manifest.txt` for freshness, and reading
`analysis/CUSTOMER_MAP.csv`. Then tell me your plan before doing anything.
