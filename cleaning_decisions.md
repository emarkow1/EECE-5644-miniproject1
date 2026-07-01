# Cleaning Decisions Log

## Overall Philosphy
- The raw `df` was loaded once and never mutated.
- All cleaning was performed on a separate `df_cleaned` object.
- Questions about removed data are answered from the
  raw`df`, since those rows no longer exist in the clean set.

## Row-level decisions

| Decision | What | Why |
|---|---|---|
| **Cancellations removed** | Rows where `invoice_no` starts with `C` | A cancelled order is not a completed sale. |
| **Non-positive quantity removed** | `quantity <= 0` | Zero/negative units indicate returns or adjustments, not sales. |
| **Non-positive price removed** | `unit_price <= 0` | Free/negative prices are not valid revenue lines. |
| **Non-product codes removed** | `stock_code` in `[POST, DOT, M, m, C2, D, S, BANK CHARGES, AMAZONFEE, CRUK, B]` | These are not merchandise. |
| **Gift vouchers removed** | `stock_code` starting `gift_` | Vouchers are store credit and therefore do not generate revenue on their own|
| **Unrecoverable blank descriptions removed** | ~112 rows still blank after recovery | A line with no product name and (typically) zero price / no customer is not a sellable item. |
| **Exact duplicates removed** | `drop_duplicates` on the business columns | Repeated identical transaction lines would double-count revenue. *(Count was ~0 after the filters above, which remove the rows that carried most duplicates.)* |

## Repair / transformation decisions

| Decision | What | Why |
|---|---|---|
| **Country labels standardised** | `EIRE → Ireland`, `RSA → South Africa`, `Unspecified → NaN` | Fix inconsistent labels and ensures that `Unspecified` will insread be captured by na functions|
| **Blank descriptions recovered** | Filled from the most common description per `stock_code` | One-off data-entry noise shouldn't outweigh the real product name. |
| **Description text normalised** | `.strip().title()` | Removes inconsistent spacing/case so the same product doesn't fragment across variants in grouping. |
| **`invoice_date` parsed** | text → `datetime`, used as time index | Required for monthly/weekly resampling and per-customer active-month metrics. |

## Missing `customer_id`
- **Kept** these rows in the cleaned dataset so that revenue totals, seasonality,
  best-sellers and market analyses stay complete.
- **Dropped** them only *within* customer-level analyses via `dropna(subset=['customer_id'])`, since spend cannot
  be attributed to an unknown buyer.
- Rationale: a missing ID does not mean the sale didn't happen

## Column decisions
- **Kept** both `stock_code` and `description` since stock_code is still useful as a
  stable product key.
- **Created** `line_revenue = quantity × unit_price`, and `region`
- **Dropped** the synthetic `unique_id` index before export since it was only needed to
  demonstrate label-based selection; it carries no analytical value.

## Assumptions
1. Negative quantity indicates a return or cancellation, not a valid sale.
2. Zero or negative unit price indicates an invalid sale.
3. One `stock_code` maps to one product (basis for mode-based description repair).
4. A missing `customer_id` is still a real sale, not an error to drop.
5. Gift vouchers are excluded from product-sales revenue.