# 🧾 COD File Format to Readable Format (CSV/JSON) – Field Extraction Guide

This guide helps you convert a `.cod` file (used for bank payment batches) into a **readable format** (e.g., CSV or JSON) by explaining **how to extract and match** fields such as:

- `statement_sequence`
- `transaction_sequence`
- `amount`
- `message`
- `transaction_date`
- `account_name`
- `account_number`
- `account_bic`

---

## Field Mapping

| Field                  | Source Line (Prefix) | Description                                                                   |
| ---------------------- | -------------------- | ----------------------------------------------------------------------------- |
| `statement_sequence`   | `1203`               | Batch sequence number (e.g., `12030` → `0`)                                   |
| `transaction_sequence` | `2100`               | Line `2100xxxxxx`, the sequence is embedded in the second set of 4 digits     |
| `amount`               | `2100`               | Characters 25–39: Amount in cents (e.g., `0000000000100000` → €100.00)        |
| `message`              | `2200` or `3100`     | Optional structured remittance info or reference (can be blank)               |
| `transaction_date`     | `2100`               | Characters 39–45: YYMMDD or DDMMYY (often `210625` → `2021-06-25`)            |
| `account_name`         | `2300`               | Characters 42–80: Name of the recipient                                       |
| `account_number`       | `2300`               | Characters 12–43: IBAN of recipient                                           |
| `account_bic`          | inferred from IBAN   | Not present in COD; must be looked up using IBAN (e.g., `REVO` → Revolut BIC) |

---

## Notes

- All fields use fixed-width formatting and require trimming whitespace.
- Amounts are expressed in cents without decimal separator and need to be divided by 100.
- The message field may span multiple lines depending on the entry type (look for `2200` or `3100` lines).
- For recurring or grouped payments, match transaction IDs from `2100` to `2300` lines using sequence IDs.

---

### 1. Understanding the Line Types

Each line in the `.cod` file starts with a **4-digit line code** that tells you what type of data is on the line:

| Code   | Purpose                                               |
| ------ | ----------------------------------------------------- |
| `0000` | File header (creator, date)                           |
| `1203` | Batch header (account, total amount)                  |
| `2100` | Payment header (amount, transaction date)             |
| `2200` | Payment message (optional message or remittance info) |
| `2300` | Recipient info (IBAN, name)                           |
| `3100` | Mandate or payment references                         |
| `8030` | Batch footer (totals)                                 |
| `9`    | File footer                                           |

---

### 2. Matching Lines into a Transaction

A **single transaction** is built from **3 or more lines** that share the same sequence number.

Example:

```
2100010000 ... <- header
2200010001 ... <- message
2300010002 ... <- recipient
```

Here, `0001` is the transaction sequence for that transaction. You must **group all `2100`, `2200`, and `2300` lines with the same sequence**.

---

### 3. Extracting Fields

#### ✅ `statement_sequence`

- From the `1203` line.
- The 5th character is the batch number:  
  `12030` → `0`

#### ✅ `transaction_sequence`

- From the `2100` line.
- Example: `2100010000` → `0001`

#### ✅ `amount`

- From the `2100` line, usually starts at position 25:

```
0000000000100000 → €100.00
```

- Divide the value by 100 to get euros.

#### ✅ `message`

- From the `2200` or `3100` line.
- These may be blank or contain remittance info or reference codes.

#### ✅ `transaction_date`

- From the `2100` line (often near column 39):

#### ✅ `account_name`

- From the `2300` line, typically starts around character 42.

#### ✅ `account_number`

- From the `2300` line, starting around position 12:

#### ✅ `account_bic`

- Not included directly in COD.
- Use the IBAN to **lookup the BIC** via an external IBAN-to-BIC API or database.
- e.g., `NL48REVO...` → `REVOLT21`

---

## 📄 Example Conversion

### Raw `.cod` snippet

```
2100010000 1000000000100000210625...
2200010001 Payment for invoice #4567
2300010002NL48REVO2468466977 ROOS ANNIEK SCHILTEN
```

### Parsed Result (CSV)

| statement_sequence | transaction_sequence | amount | message                   | transaction_date | account_name         | account_number     | account_bic |
| ------------------ | -------------------- | ------ | ------------------------- | ---------------- | -------------------- | ------------------ | ----------- |
| 0                  | 0001                 | 100.00 | Payment for invoice #4567 | 2021-06-25       | ROOS ANNIEK SCHILTEN | NL48REVO2468466977 | REVOLT21    |

---

## 🛠 Tips

- Use **fixed-width parsing** tools or regex.
- Watch for **blank message lines** – not all transactions will include one.
- Line order is consistent: `2100` → `2200` → `2300`.
