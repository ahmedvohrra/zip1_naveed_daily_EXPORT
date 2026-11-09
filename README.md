# ZIP ONE — daily export of the Naveed (Oracle) database

Refreshed automatically every day (see `manifest.txt` for the time of the last export and the latest invoice/receipt dates).
Source: Oracle 10g, SID `new`. Everything here is a read-only copy; nothing is ever written back.

**Purpose:** a complete copy of the accounting data so it can be migrated to Odoo from another machine.

## Start here — which files to trust

| File / folder | What it is | Trust it for |
|---|---|---|
| **`ledger_naveed.csv`** | Naveed's *own* ledger (its view `VCUSTHIST1`): one row per debit/credit on every account | **the ground truth** — every balance, aging and statement in Naveed is built from this |
| **`customer_balances.csv`** | Closing balance per account using Naveed's own `BAL()` rule | opening balances / reconciliation |
| **`raw/`** | **Every business table, every column**, exactly as stored (`raw/<TABLE>.csv`; big tables are split into `_partNNN.csv`) | anything not in the ledger: line items, products, routes, headers, tax/discount columns |
| `raw/_tables.csv` | per table: rows in Oracle vs rows written, status | verifying the export is complete |
| `raw/_checksums.csv` | per numeric column: SUM | verifying the files were not damaged in transit |
| `invoices.csv`, `invoice_lines.csv`, `receipts.csv`, `customers.csv`, … | convenient *summaries* with names joined in (older format) | browsing only — **not** for balances (see warning below) |

> **Warning — do not rebuild balances from `invoices.csv` − `receipts.csv`.**
> `customers.csv` (`total_invoiced`, `total_received`) and `ledger_all.csv` are simplified and leave out discounts,
> sale returns, invoice-level receipts, cash-sale credits, supplier purchases, expenses and opening balances.
> Checked on 2026-11-09 against Naveed's own balances, that shortcut was wrong for 147 of 513 accounts (ZIP ONE)
> and 2,847 of 3,105 accounts (ZIP MAIN). Use `ledger_naveed.csv` / `customer_balances.csv`.

## How Naveed computes a balance

`balance = SUM(debits, AORS='A') − ABS(SUM(credits, AORS='S'))` over `ledger_naveed.csv`
(positive = the account owes us / receivable; negative = we owe it / payable).
Opening balances (`CUST.COBAL`) are inside the ledger as TTYPE 5 (debit) and 6 (credit).

### `ttype` legend (column `ttype` in `ledger_naveed.csv`)

| ttype | aors | Meaning | Comes from |
|---|---|---|---|
| 1 | A | Sales invoice (credit or cash) = `SUM(QTY*UPRICE)` where `RETURN2='N'` and `QTY>0` | `INV0S`+`INV1S` |
| 2 | S | Cash received against a **cash** sale (`CORD='C'`), net of `INVCASHD..`, + `BILTICHG`, − `BAMT`, + `FEE`, + withholding tax | `INV0S` header columns |
| 3 | S | Purchase invoice (supplier credit) | `INV0P`+`INV1P`, `RETURN2='N'` |
| 4 | A | Cash paid against a cash purchase | `INV0P` |
| 5 / 6 | A / S | Opening balance (`COBAL` ≥ 0 / < 0) | `CUST` |
| 7 | A | Cash payment made | `CASHP1` (+ `INV0P.INVCASHP`) |
| 8 | S | Cash receipt from customer | `CASHR1` (+ `INV0S.INVCASHR` keyed on credit invoices, + recovery sheets `CASH_REC0/1`) |
| 9 | S | **Sales discount** | `CASHD1` **+ `INV0S.INVCASHD`** (this is most of it) |
| 10 | A | Purchase discount | `CASHZ1`, `INV0P.INVCASHZ`, `CASHP1.CASHPCASHZ` |
| 11 / 12 / 13 | S / A / S | Cash investment / withdrawal / other income | `CASHI`, `CASHW1`, `CASHS` |
| 14 | A | Expenses | `CASHE1`, `INV0P.BILTIEXP` |
| 15 | A | **Purchase return** (goods sent back to a supplier) | `INV0S` where `RETURN2='Y'` |
| 17 | S | **Sale return** (goods returned by a customer) | `INV0P` where `RETURN2='Y'` |
| 16 / 18 | S / A | Cash side of a purchase return / sale return | `INV0S` / `INV0P` |
| 19 | S | Manufacturing/labour liability | `MAN*` + `PROD` (`PTYPE='L'`) |

## Things that are easy to get wrong

* **`CUST` is the whole chart of accounts, not just customers.** `CODE`: `C` customer/supplier-type party, `G`/`E`/`W` general,
  expense and withdrawal accounts, `B` both. Names such as `EXPENSE ACCOUNT`, `CAPITAL ...`, `... HOME EXP` are *accounts*, not customers.
  Classify before creating Odoo partners. Negative balances are payables.
* **`INV0S.RETURN2='Y'` is a *purchase* return** (debit to a supplier — e.g. RECCON, KAIZEN, ZHOULI ENGINEERING), **not** a sale return.
* **Sale returns are stored in the *purchase* table `INV0P`** (`RETURN2='Y'`); genuine supplier purchases are `INV0P` with `RETURN2='N'`.
  (`purchases.csv` carries both — filter on `is_return`.)
* **`INVR0` / `INVR1` are not returns** (they hold retail-style bills, `INVTYPE='B'`); `invr0_header.csv`/`invr1_lines.csv` are plain copies.
  Earlier files named `sales_returns.csv` / `sales_return_lines.csv` were mis-named and have been removed.
* **Invoice amount = `SUM(QTY*UPRICE)`.** `DISC`/`DISC_PCENT` on a line are already inside `UPRICE`; never subtract them again.
  `INV1S.TOTAL_VALUE` is unreliable. Header-level discount/receipt/freight columns (`INVCASHD..INVCASHD4`, `INVCASHR`, `BILTICHG`, `BAMT`, `FEE`) *do* matter — they are in `raw/INV0S.csv`.
* Invoice numbers: `INV0S.INVNO`, `INV0P.INVNO`, receipt numbers `CASHR1.CASHNO` — Odoo may carry them as `INV-0009`-style refs.
* The two databases differ slightly: ZIP MAIN has `PRODINV`, `DEPT`, `TAR0/1` and no `CUST.COBAL_DATED`; ZIP ONE has no `PRODINV`.
* Not exported on purpose (not accounting): audit trail, SMS queues, menu/role tables, product photos.

## Verifying a download

1. `raw/_tables.csv`: every row should say `status = OK` and `rows_written = rows_in_oracle`.
2. `raw/_checksums.csv`: the SUM of each numeric column recomputed from the CSVs must equal the stored value.
3. `ledger_naveed.csv` grouped by `ccode` with the rule above must equal `customer_balances.csv`.

Verified on 2026-11-09 against Oracle's own `BAL()` function: all 513 accounts match exactly (0 mismatches).
