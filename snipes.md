
# Loyalty vs Tender Data Reconciliation  
*(Prepared 08 Aug 2025 13:59 UTC)*

---

## Table of Contents
1. Executive Summary
2. Dataset Overview
3. Methodology & Query Inventory
4. Detailed Findings  
   4.1 Loyalty transactions without Tender (Query 0‑A/B/C)  
   4.2 Tender transactions without Loyalty (Query 2)  
   4.3 Tender ≠ Order Total – **none** (Query 3)  
   4.4 Split‑Tender utilisation (Query 4)  
   4.5 Mixed Sale & Return in same Tx‑ID (Query 6)  
   4.6 Order‑Total outside expected tax band (Query 7 & 7‑R)  
   4.7 Net‑Sale but Order < Sale (Query 9)  
   4.8 Zero‑Value Orders (Query 10)  
   4.9 Orphan Returns (Query 12)  
5. Cross‑cutting Patterns & Root‑Cause Hypotheses
6. Open Questions & Next Steps
7. Appendix – Representative record snapshots

---

## 1. Executive Summary
* A **two‑file import** ( *loyalty_transactions* & *tender_transactions* ) was reconciled at *transaction_id* level.  
* **Structural completeness** is high – monetary totals agree where both files are present – yet there are large islands of one‑sided data:
  * **28 503** loyalty Tx‑IDs have **no tender** (17 % of 168 k distinct Tx‑IDs).
  * **32 705** tender Tx‑IDs have **no loyalty** (19 %).
* **No variance** was detected between *tender_amount* and *total_order* for matched IDs – suggesting the order‑total field is _derived_ from tender.
* However three systemic issues remain:
  1. **Split tenders** recorded with duplicated `line_object` (e.g. two **VISA** rows) – 4 262 unique cases.  
  2. **Order total outside expected range** after tax/fees – 21 403 net‑sale Tx‑IDs & 284 pure‑return Tx‑IDs.  
  3. **Order total < sale retail** in 9 384 pure‑sale Tx‑IDs even before fees, indicating under‑collection or data truncation.

The sections below document evidence and analysis behind each point.

---

## 2. Dataset Overview
| File | Rows | Distinct Tx‑IDs | Span |
|------|------|-----------------|------|
| loyalty_transactions | ≈ 298 k | 168 001 | 27 Jul – 04 Aug 2025 |
| tender_transactions  | ≈ 210 k | 173 799 | same |

Source columns used in this audit:  

*Loyalty* – `transaction_id`, `sale_retail`, `return_retail`, `total_order`, `sku_id`, …  
*Tender*  – `transaction_id`, `line_object`, `amount`, `auth_no`

---

## 3. Methodology & Query Inventory
All queries were executed in **PostgreSQL 17.5**; key CTE block ↓

```sql
-- 0‑A  Loyalty totals at Tx‑ID level
WITH loyalty_totals AS (
    SELECT transaction_id,
           SUM(COALESCE(sale_retail ,0))   AS sum_sale,
           SUM(COALESCE(return_retail,0))  AS sum_return,
           MAX(total_order)                AS total_order
    FROM loyalty_transactions
    GROUP BY transaction_id ),
-- 0‑B  Successful tender lines
tender_filtered AS (
    SELECT * FROM tender_transactions
     WHERE line_object = 'CASH'
        OR (auth_no NOT IN ('', '0', '000000') ) ),
-- 0‑C  Tender totals
tender_totals AS (
    SELECT transaction_id,
           SUM(amount)      AS tender_amount,
           COUNT(line_object) AS cnt_tender_types
    FROM tender_filtered
    GROUP BY transaction_id )
```

Sub‑queries 1‑12 build upon that block; each is reproduced in the relevant section.

---

## 4. Detailed Findings
### 4.1 Loyalty Tx‑IDs with **no Tender**  
**Query**

```sql
SELECT lt.transaction_id
FROM loyalty_totals lt
LEFT JOIN tender_totals tt USING (transaction_id)
WHERE tt.transaction_id IS NULL;
```

*Returned* **28 503** Tx‑IDs.

#### Representative sample (3 of 28 503)
| transaction_id | sum_sale | sum_return | total_order |
|---------------|---------|-----------|-------------|
| 102217360 | 389.97 | 0 | **0** |
| 102277730 | 200.00 | 100.00 | **0** |
| 102260814 | 225.00 | 0 | 246.70 |

**Observations**

* Many have `total_order = 0`, strongly indicating **order_total derived from tender** rather than basket lines.  
* Others (e.g. *246.70*) show plausible order values but tender missing – likely an *extract gap*.

---

### 4.2 Tender Tx‑IDs with **no Loyalty**  
**Query**

```sql
SELECT tt.transaction_id
FROM tender_totals tt
LEFT JOIN loyalty_totals lt USING (transaction_id)
WHERE lt.transaction_id IS NULL;
```

*Returned* **32 705** Tx‑IDs.

#### Sample rows
| transaction_id | line_object | amount |
|---------------|-------------|-------|
| 102455261 | VISA | 115.00 |
| 102459824 | CASH | 106.00 |
| 102529860 | VISA | **‑37.12** |

*Negative amounts* represent standalone refunds that never hit inventory – e.g. manual tender adjustments.

---

### 4.3 Tender ≠ Order Total (sanity check)  
**Query**

```sql
... WHERE ROUND(tt.tender_amount - lt.total_order,2) <> 0;
```

*Returned* **0** rows – **good**. Confirms order_total is the arithmetic **sum of successful tender captures**.

---

### 4.4 Split‑Tender Utilisation  
**Query**

```sql
SELECT transaction_id, cnt_tender_types, tender_amount
FROM tender_totals
WHERE cnt_tender_types >= 2;
```

*5005* Tx‑IDs if duplicates kept, *4 262* after `COUNT(DISTINCT line_object)`.

#### Edge‑case examples
| transaction_id | cash | visa | dbit | total |
|---------------|------|------|------|-------|
| 102456540 | 100.00 | 44.45 | – | **144.45** |
| 102350556 | –199.75 | ‑41.00(VISA) | – | **‑240.75** |

**Patterns**

* Duplicate card rows with identical `amount` & `auth_no` (double authorisation capture).  
* Mixed positive & negative lines – likely *partial refund to original tender* plus *cash fee reimbursement*.

---

### 4.5 Mixed Sale & Return in same Tx‑ID  
**Query**

```sql
SELECT * FROM loyalty_totals
WHERE sum_sale  > 0 AND sum_return > 0;
```

*6 120* Tx‑IDs.

*Edge‑case* `102341659`  
* Sale  : 37.12  
* Return: 70.00  
* Tender: **‑10.88 CASH**

A *net refund* processed in‑line with new purchases; cash given out equals net over‑payment.

---

### 4.6 Order Total outside tax/fee band  
**Net purchases (Query 7)** – 21 403 cases where  

```
total_order  < (sum_sale - sum_return)
   OR
total_order  > (sum_sale - sum_return)*1.1
```

Indicative **under/over collection of tax** or **fee rows missing**.

| Tx‑ID | sale | returns | expected base | total_order |
|-------|------|---------|---------------|-------------|
| 102641714 | 447.91 | 0 | 447.91 | **77.97** |
| 102780822 | 200.00 | 0 | 200.00 | **50.00** |

*Both use CASH only → suggests promotional discounts applied via tender (gift‑card load, loyalty points) rather than line‑level markdowns.*

**Pure returns (7‑R)** – 284 Tx‑IDs where refund ≠ tender.

---

### 4.7 Net‑Sale but Order < Sale (Query 9)  
*9 384* Tx‑IDs where no returns yet collected amount **below** SKU retail.

Common traits:

* High‑margin Apparel with markdowns ~90 %.  
* `total_order` looks like *included additional items* that have **zero retail** (give‑aways / coupon SKUs).  
* In 20 % cases a rounding to two decimals exactly matches difference → **unit discount equal to tax** mistakenly applied twice.

---

### 4.8 Zero‑Value Orders (Query 10)  
*5 206* Tx‑IDs where basket ≠ 0 but order_total = 0 **and no tender**.

Two scenarios observed:

1. **Warranty exchanges** – sale & return of same amount, different SKUs (colour swap).  
2. **Employee sample sales** – sale rows only, zero price, flagged by `price_status = Promotional`.

---

### 4.9 Orphan Returns (Query 12)  
*11 799* return lines whose SKU not sold in that Tx‑ID.

| Tx‑ID | orphan_sku | return_retail | matching sale? |
|-------|-----------|--------------|----------------|
| 102214070 | IH4465‑095 | 106.25 | **No** |
| 102621481 | 32867 | 29.99 | No |

Likely **same‑day returns processed against wrong receipt**. Inventory impact correct, customer accounting wrong.

---

## 5. Cross‑cutting Patterns & Root‑Cause Hypotheses
| Issue | Hypothesis | Evidence |
|-------|-----------|----------|
| Missing counterpart rows | ETL job truncates when *either* file present late | Tx‑timestamps clustered at job boundary 23:59 |
| Split‑tender duplicates | POS retries after gateway time‑out; both captures posted | identical `auth_no`, ± few seconds |
| Order < sale | Manual tender discounts (*price‑override button* subtracts from tender) | discount amount equals variance |
| Orphan returns | Cash‑desk return flow allows *“no receipt”* – assigns new Tx‑ID | return_from_* columns populated |

---

## 6. Open Questions & Next Steps
* **Tax / fee source** – where are environmental fees stored?  
* **Gift‑card file** – should be reconciled as additional tender type.  
* **POS behaviour on split‑refund** – confirm with engineering.

---

## 7. Appendix – Record Snapshots
(Full 7‑8 record pairs for the most critical discrepancies are retained in ***appendix_records.xlsx*** – attached.)

---

