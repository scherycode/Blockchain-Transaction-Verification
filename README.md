# BlueBlock — Blockchain Audit Verification System

A proof-of-concept terminal application demonstrating real-time, decentralized transaction auditing using SHA-256 hashing.

## Concept

Two parties to a transaction (Company A and Vendor B) independently enter standardized transaction data. The system normalizes both inputs and produces SHA-256 hashes. If both parties entered the same underlying data, their hashes will be identical — regardless of formatting differences like capitalization, currency symbols, or decimal precision. An auditor can then inspect a shared ledger and instantly flag any discrepancy.

## Quick Start

**Requirements:** Python 3.8+, `rich`, `openpyxl`

```bash
pip install -r requirements.txt
python blockchain_audit.py
```

**Demo credentials** (all passwords: `password`):

| Username     | Role      |
| ------------ | --------- |
| `companya` | Company A |
| `vendorb`  | Vendor B  |
| `auditor`  | Auditor   |

## Demo Script

1. **Log in as `companya`** — enter transaction `INV-001` with full details
2. **Log in as `vendorb`** — enter the same transaction (different formatting, e.g. lowercase, `$10,000`) — the hash will match
3. **Log in as `auditor`** — view verification status: `INV-001` shows **VERIFIED** with both hashes displayed
4. **Fraud demo** — both parties enter `INV-002` with mismatched amounts → auditor sees **MISMATCH - FLAG**; hashes visibly differ

## How It Works

The core mechanic is a two-step pipeline: **normalize → hash**. Both parties run the same pipeline independently on the same transaction. If they entered the same underlying data, they will always produce the same 64-character hash — regardless of how they formatted their input.

### Step 1 — Validation

Before normalization, each field is validated to ensure it can be meaningfully compared. Invalid input is rejected immediately; the user must correct it before submission.

| Field | Validator | What it accepts |
| -------------- | ------------------- | ---------------------------------------------------------------- |
| Transaction ID | Non-empty check | Any non-blank string |
| Date | `validate_date()` | `YYYY-MM-DD` strictly (other formats are rejected) |
| Product ID | Non-empty check | Any non-blank string |
| Amount | `validate_amount()` | Numbers with or without `$` and `,` — must be non-negative |
| Currency | Non-empty check | Any non-blank string |
| Quantity | `validate_quantity()` | Any positive number in any format |
| Payer ID | Non-empty check | Any non-blank string |
| Seller ID | Non-empty check | Any non-blank string |
| On Credit? | `validate_yes_no()` | `yes`, `y`, `true`, `1` → YES; `no`, `n`, `false`, `0` → NO |

### Step 2 — Normalization

Once validated, `normalize_transaction()` applies a deterministic transformation to every field. The rules are fixed and applied identically to every submission, regardless of who submitted it.

| Field | Rule | Example input | Canonical output |
| -------------- | ---------------------------------------- | -------------- | ---------------- |
| Transaction ID | `.upper()` + strip spaces | `inv-001` | `INV-001` |
| Date | Enforced to `YYYY-MM-DD` by validator | `2025-9-8` | `2025-09-08` |
| Product ID | `.upper()` + strip spaces | `widget 500` | `WIDGET-500` |
| Amount | Strip `$`/`,`, format to 2 decimals | `$10,000` | `10000.00` |
| Currency | `.upper().strip()` | `usd` | `USD` |
| Quantity | Format to 2 decimals | `100` | `100.00` |
| Payer ID | `.upper()` + strip spaces | `Company A` | `COMPANYA` |
| Seller ID | `.upper()` + strip spaces | `Vendor B` | `VENDORB` |
| On Credit? | Map accepted values to `YES` or `NO` | `1` / `true` | `YES` |

The normalized fields are assembled into a **canonical string** — a single pipe-delimited line in a fixed, predetermined field order:

```
{TXN_ID}|{DATE}|{PRODUCT_ID}|{AMOUNT}|{CURRENCY}|{QUANTITY}|{PAYER_ID}|{SELLER_ID}|{ON_CREDIT}
```

Example:

```
INV-001|2025-09-08|WIDGET-500|10000.00|USD|100.00|COMPANYA|VENDORB|NO
```

The field order is hardcoded as `[transaction_id, date, product_id, amount, currency, quantity, payer_id, seller_id, on_credit]` and never changes. This is essential: if the same fields appeared in different orders, the resulting strings — and therefore the hashes — would differ even for identical data.

The canonical string is stored in the ledger alongside the hash, making every submission independently auditable.

### Step 3 — Hashing

The canonical string is passed to `hash_transaction()`, which runs a single line:

```python
hashlib.sha256(canonical_string.encode("utf-8")).hexdigest()
```

Three things happen in sequence:

1. **Encoding** — `canonical_string.encode("utf-8")` converts the string to a UTF-8 byte sequence. This guarantees that characters with values above ASCII 127 are always encoded the same way, regardless of platform or locale.

2. **Hashing** — Python's `hashlib.sha256()` runs the SHA-256 algorithm on those bytes and produces a 256-bit (32-byte) digest.

3. **Hex encoding** — `.hexdigest()` converts the binary digest to a 64-character lowercase hexadecimal string — the form stored in the ledger and shown to users.

The result looks like this:

```
3b4c1a9f7d2e0854a6b3c9d1e5f2087a4c6e8b0d2f4a7c9e1b3d5f7a9c2e4b6
```

(64 hex characters = 256 bits)

### Why SHA-256?

SHA-256 (Secure Hash Algorithm 256-bit) was standardized by NIST in 2001 and is the same algorithm used to hash transactions in Bitcoin and most major blockchains. While the application of it here is straightforward, four cryptographic properties make it the right choice for this system:

**1. Determinism** — The same input always produces the same output, every time, on every machine. This is the foundational guarantee: if Company A and Vendor B entered the same data, they will always get the same 64-character hash, without coordinating directly.

**2. Collision resistance** — It is computationally infeasible for two different inputs to produce the same hash. The probability of a collision is approximately 1 in 2²⁵⁶ — a number larger than the estimated number of atoms in the observable universe. If two hashes match, you can be effectively certain the underlying data matched.

**3. One-way (pre-image resistance)** — Given a hash, you cannot reverse-engineer the original string. This means the auditor can verify that two entries match without ever seeing the transaction details — which is precisely how real blockchain validators confirm transactions without reading their contents. The auditor interface enforces this: only hashes are displayed, never the underlying fields.

**4. Avalanche effect** — A single-character difference in the input produces a completely different, unpredictable hash. If one party records `10000.00` and the other records `10001.00`, the two 64-character hashes will look nothing alike. There is no way to tell "how different" the inputs were just by comparing the hashes — a mismatch of one cent looks exactly as wrong as a mismatch of a million dollars. This makes discrepancies immediately visible and impossible to obscure.

### Why Normalization Is Essential

Without normalization, any formatting difference between the two parties would produce a hash mismatch — even when both recorded the exact same transaction. The hash function is byte-perfect: a lowercase `a` and an uppercase `A` produce different hashes.

| Input pair | Without normalization | With normalization |
| -------------------------------- | --------------------- | ------------------ |
| `$10,000` vs `10000.00` | Different hashes | Same hash |
| `inv-001` vs `INV-001` | Different hashes | Same hash |
| `yes` vs `1` vs `true` | Different hashes | Same hash |
| `100` vs `100.00` | Different hashes | Same hash |
| `Company A` vs `COMPANYA` | Different hashes | Same hash |

Normalization is what makes the hash comparison meaningful. The hash is only trustworthy because both parties feed the same canonical form into it. This is the core claim of the system: **format-agnostic audit verification** — two parties can independently record the same transaction in their own house style, and the system can still definitively verify whether they agree.

### End-to-End Example

Company A and Vendor B each enter the same transaction in their own style:

| Field | Company A input | Vendor B input |
| -------------- | ------------- | -------------- |
| Transaction ID | `inv-001` | `INV-001` |
| Date | `2025-09-08` | `2025-09-08` |
| Product ID | `Widget 500` | `WIDGET-500` |
| Amount | `$10,000.00` | `10000` |
| Currency | `usd` | `USD` |
| Quantity | `100` | `100.00` |
| Payer ID | `companya` | `COMPANYA` |
| Seller ID | `Vendor B` | `VENDORB` |
| On Credit? | `no` | `0` |

Both produce the same canonical string:

```
INV-001|2025-09-08|WIDGET-500|10000.00|USD|100.00|COMPANYA|VENDORB|NO
```

And therefore the same SHA-256 hash. The auditor sees: **VERIFIED**.

## Transaction Fields

| Field          | Example        | Canonical Format     |
| -------------- | -------------- | -------------------- |
| Transaction ID | `inv-001`    | Uppercase, no spaces |
| Date (UTC)     | `9/8/2025`   | `YYYY-MM-DD`       |
| Product ID     | `widget 500` | Uppercase, no spaces |
| Total Amount   | `$10,000`    | `10000.00`         |
| Currency       | `usd`        | Uppercase            |
| Quantity       | `100`        | `100.00`           |
| Payer ID       | `companya`   | Uppercase, no spaces |
| Seller ID      | `vendorb`    | Uppercase, no spaces |
| On Credit?     | `n`          | `YES` or `NO`    |

## Ledger

Entries are stored in `ledger.json` (auto-created at runtime). Each entry records the submitter, timestamp, and hash. Delete `ledger.json` to reset for a fresh demo.

## Auditor — Block Explorer

The auditor interface is styled as a private blockchain block explorer. All transactions are stored in a single block. The auditor menu provides:

| Option                       | Description                                                                      |
| ---------------------------- | -------------------------------------------------------------------------------- |
| [1] View Full Ledger         | Block #1 header + transaction table (Entry ID, Submitter, Timestamp, Hash)       |
| [2] View Verification Status | One color-coded panel per transaction group showing both hashes and match result |
| [3] Search by Hash           | Paste a full SHA-256 hash to find the entry and check counterparty verification  |
| [4] Search by Entity         | Filter transactions by submitter (COMPANYA or VENDORB)                           |

A stats bar at the top of each view shows: `1 Block | N Transactions | ✔ Verified | ○ Pending | ✖ Flagged`.

## Test Data Generator

`generate_test_data.py` creates two pre-formatted Excel files for demo/testing purposes:

```bash
python generate_test_data.py
```

Outputs to `Test Data/`:

- `company_a_transactions.xlsx` — 100 transactions in Company A formatting style (uppercase, `$X,XXX.XX` amounts, integer quantities, `yes`/`no` credit flag)
- `vendor_b_transactions.xlsx` — same 100 transactions in Vendor B formatting style (lowercase, plain decimal amounts, `X.XX` quantities, `1`/`0` credit flag)

Both files normalize to identical hashes when imported, demonstrating the normalization engine. They can be bulk-imported via the "Upload Transactions from Excel" option in the Company menu.

## File Structure

```
blockchain_audit.py          # All application code
generate_test_data.py        # Test data Excel generator
ledger.json                  # Auto-generated data store
ledger_export.xlsx           # Auto-generated Excel export (auditor)
CLAUDE.md                    # Project architecture notes
requirements.txt             # Python dependencies
Test Data/
    company_a_transactions.xlsx   # 100 test transactions (Company A style)
    vendor_b_transactions.xlsx    # 100 test transactions (Vendor B style)
```

## Excel Bulk Upload (Companies)

Company users can upload transactions in bulk from an `.xlsx` file via the "Upload Transactions from Excel" menu option. The file must have a header row and 9 columns:

| Column | Field              |
| ------ | ------------------ |
| A      | Transaction ID     |
| B      | Date (YYYY-MM-DD)  |
| C      | Product ID         |
| D      | Amount             |
| E      | Currency           |
| F      | Quantity           |
| G      | Payer ID           |
| H      | Seller ID          |
| I      | On Credit (yes/no) |

Each row is validated individually — invalid rows are skipped with clear error messages while valid rows proceed. All valid transactions are submitted to the ledger in one batch.

## Excel Export (Auditor)

After viewing the full ledger, the auditor is prompted to export to Excel. Choosing "y" saves `ledger_export.xlsx` in the project directory with full 64-character hashes.

## Auditor Verification States

| Status                | Meaning                                                 |
| --------------------- | ------------------------------------------------------- |
| VERIFIED (green)      | Two entries from different parties with matching hashes |
| PENDING (yellow)      | Only one party has submitted so far                     |
| MISMATCH - FLAG (red) | Same Transaction ID, different hashes — possible fraud |
