# Logistics Pricing & Profitability Analytics

An end-to-end Excel analytics model that links operational **Delivery Orders** with financial **Invoices** to measure shipment margins, locate revenue leakage and quantify receivables risk for a regional logistics company (FY 2025).

> **Note on data:** the dataset is synthetic and was generated with Python for portfolio purposes. All figures in this repository are illustrative.

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Business Objectives](#business-objectives)
3. [Dataset](#dataset)
4. [Workbook Architecture](#workbook-architecture)
5. [Methodology](#methodology)
6. [Key Results](#key-results)
7. [Insights](#insights)
8. [Recommendations](#recommendations)
9. [Dashboard](#dashboard)
10. [Tools & Skills](#tools--skills)
11. [Repository Structure](#repository-structure)
12. [Documented Edge Cases & Maintenance Notes](#documented-edge-cases--maintenance-notes)
13. [Author](#author)

---

## Project Overview

The company needs a pricing and profitability analysis to evaluate shipment margins, identify revenue leakage and map direct operational costs (HPP) across its regional network. The core of the work is a reconciled dataset in which every operational Delivery Order is matched to its financial Invoice, so that margin tracking and accounts receivable (AR) management rest on one consistent source.

The project covers three areas:

- **Data cleaning and standardisation** of raw operational and billing fields.
- **Revenue and cost modelling** at shipment level, from billable weight through to gross profit and margin health.
- **Profitability and leakage diagnostics**, summarised in an executive dashboard and a set of quantified recommendations.

## Business Objectives

- Calculate net revenue per shipment, including contract discounts, accessorial fees (COD and insurance) and SLA penalties.
- Model direct operational cost (linehaul, first mile, sorting, last mile and return-to-shipper overhead) to derive gross profit and gross margin %.
- Link every Delivery Order to exactly one Invoice and compute PPN, due dates and overdue aging.
- Identify the least profitable shippers and routes, with root causes.
- Present the results in a clean executive dashboard.

## Dataset

| Sheet | Rows | Columns | Role |
|---|---|---|---|
| Delivery_Orders | 12,000 | 43 | Operational shipment transactions (primary fact table) |
| Invoices (Table1) | 12,000 | 16 | Billing records, linked 1:1 to Delivery Orders via DO_Number |
| Master_Shipper | 30 | 9 | Shipper tiers, contract discounts, rebate rules, SLA exemption flags |
| Master_Route_Rate | 25 | 7 | Zone category and base tariff per KG (Standard / Express / Cargo) |
| Master_Cost_Structure | 25 | 6 | Linehaul cost per KG, fixed parcel costs, fuel surcharge |
| Master_SLA_Target | 15 | 5 | Target lead time and penalty per late day by service and zone |
| Master_Fee_Rules | 5 | - | COD rate, insurance rate, RTS rate, volumetric divisor, PPN |

Delivery dates cover January to December 2025.

## Workbook Architecture

The workbook contains 21 worksheets organised as a multi-fact, star/snowflake relational model across four layers:

| Layer | Sheets | Purpose |
|---|---|---|
| Transactional facts | 2 | Delivery_Orders, Invoices |
| Master dimensions | 5 | Rate cards, shipper contracts, cost schedules, SLA rules, fee/tax parameters |
| Analytical and pivot | 11 | Leakage matrices, margin pivots, rebate evaluation, AR breakdown, chart staging tables |
| Executive presentation | 3 | Pricing & Profitability Dashboard |

**Join keys**

| From | To | Cardinality | Retrieves |
|---|---|---|---|
| Delivery_Orders.Shipper_ID | Master_Shipper.Shipper_ID | Many-to-one | Contract discount %, SLA exemption flag |
| Delivery_Orders.Route_ID | Master_Route_Rate, Master_Cost_Structure | Many-to-one | Zone, tariffs, linehaul cost, fuel surcharge, fixed parcel costs |
| Delivery_Orders.SLA_Target_ID | Master_SLA_Target.SLA_Target_ID | Many-to-one | Target lead time, penalty per late day |
| Invoices.DO_Number | Delivery_Orders.DO_Number | One-to-one | Net revenue per shipment |
| Invoices.Shipper_ID_Clean | Master_Shipper.Shipper_ID | Many-to-one | Tier-level AR analysis |

## Methodology

### 1. Data cleaning and standardisation

Eight raw fields in Delivery_Orders and four in Invoices were cleaned and validated with auditable Excel formulas.

| Area | Raw problem | Result |
|---|---|---|
| Shipper ID | Mixed casing, whitespace, missing hyphens | Normalised keys, zero lookup mismatches against Master_Shipper |
| Service type | 9 label variants (Std, STNDRD, Exp, Crgo, ...) | 3 rate-card categories: Standard (6,341), Express (3,836), Cargo (1,823) |
| Package dimensions | Mixed separators and unit suffixes (29X33X27 CMcm, 10 * 26 * 29) | Parsed length, width and height for all 12,000 rows, feeding volumetric weight |
| Actual weight | 5,319 rows (44.3%) stored as text | Consistent numeric evaluation |
| COD / insured flags | 12 variants (Y, TRUE, 1, no, ...) | Binary Yes/No flags |
| Delivery status | 11 variants | 4 states: Delivered (5,615), Returned_To_Shipper (2,768), In_Transit (1,873), Lost_Damaged (1,744) |
| Dates | Mixed MM-DD-YYYY, ISO timestamps, blanks | Validated and standardised to DD-MM-YYYY, - sentinel for undelivered POD |
| Payment status | 9 variants | 3 buckets: Paid, Unpaid, Overdue |
| Invoice date | 2,607 blank dates (21.73%) | Null-safe Due_Date formula avoids #VALUE! errors |
| Failed deliveries | Billed fees on 1,744 Lost_Damaged orders | Net revenue forced to Rp 0 to remove phantom revenue |

A post-cleansing audit found no formula errors (#N/A, #VALUE!, #DIV/0!, #NAME?) across the 24,000 transactional records.

### 2. Revenue and billing logic


Volumetric Weight  = (L x W x H) / 6,000
Billable Weight    = MAX(Actual, Volumetric)          -- Cargo has a 10 kg minimum
Gross Shipping Fee = Base Tariff per KG x Billable Weight
Net Shipping Fee   = Gross Shipping Fee x (1 - Contract Discount %)
COD Fee            = MAX(Rp 2,500, 2.5% x Declared Value)     -- COD shipments only
Insurance Fee      = MAX(Rp 1,000, 0.2% x Declared Value)     -- insured shipments only
SLA Penalty        = Penalty per late day, if shipper is not exempt and lead time > target
Net Revenue        = Net Shipping Fee + COD Fee + Insurance Fee - SLA Penalty   (Rp 0 if Lost_Damaged)


The base tariff is retrieved with a two-way INDEX/MATCH on route and service type. The SLA target is resolved with a composite XLOOKUP on service type and route zone.

### 3. Direct cost (HPP) and margin


Linehaul Cost      = Billable Weight x Linehaul Cost per KG x (1 + Fuel Surcharge %)
Total Direct Cost  = Linehaul + First Mile + Sorting Hub + Last Mile + RTS Return Cost
RTS Return Cost    = 50% x Last-Mile Cost             -- Returned_To_Shipper only
Gross Profit       = Net Revenue - Total Direct Cost
Gross Margin %     = Gross Profit / Net Revenue       -- guarded against zero revenue


Each shipment is also classified into a margin health tier: Negative/Unprofitable (<= 0%), Low (0-15%), Healthy (15-35%) or High (> 35%).

### 4. Invoice reconciliation and AR

- Invoice_Net_Revenue is pulled from Delivery_Orders by DO_Number, so every invoice is tied to exactly one shipment (12,000 to 12,000).
- PPN is computed at 11% on invoice net revenue.
- Due_Date is invoice date plus the credit days parsed from the payment terms (Net 15 / 30 / 45).
- Overdue_Days is calculated against a reference cutoff date for unpaid and overdue invoices.
- Outstanding AR is the sum of Unpaid and Overdue invoice totals.

## Key Results

| KPI | FY 2025 |
|---|---|
| Delivery orders | 12,000 |
| Gross shipping revenue | Rp 3,046,260,544 |
| Contract discounts granted | Rp 445.24M (14.62% of gross) |
| **Net revenue** | **Rp 2,347,171,466** |
| Total direct cost | Rp 1,333,478,741 |
| **Gross profit** | **Rp 1,013,692,725** |
| **Weighted gross margin** | **43.19%** |
| Total billed (incl. PPN) | Rp 2,575,458,711 |
| **Outstanding AR** | **Rp 1,268,323,969 (49.25% of billed)** |

Other figures worth noting:

- **Fulfilment exceptions:** 37.6% of orders end as Returned-to-Shipper or Lost/Damaged.
- **Ancillary fees:** COD (Rp 181.84M) and insurance (Rp 14.32M) add Rp 196.15M, which cushions discounts and SLA penalties (Rp 34.99M).
- **Cost structure:** linehaul is about 91.1% of direct cost. Rp 199.22M of direct cost sits in Lost/Damaged orders that generate no revenue.
- **Margin polarisation:** 7,867 orders (65.6%) reach High Margin, while 2,083 orders (17.4%) are Negative/Unprofitable.

## Insights

### Shipper profitability and discount leakage

The five lowest-profit shippers carry 22-25% contract discounts, against a portfolio average of about 14.6%. Higher discounts line up with lower total gross profit.

| Rank | Shipper | Tier | Contract discount | SLA penalty (FY25) | Gross profit |
|---|---|---|---|---|---|
| Bottom 1 | SHP-018 (Electronics Mega Store) | Key Account | 25% | Rp 1,494,000 | Rp 18,533,620 |
| Bottom 2 | SHP-014 (Sport Center ID) | SME Starter | 25% | Rp 1,592,000 | Rp 23,946,143 |
| Bottom 3 | SHP-023 (Gourmet Snacks Co) | Enterprise Silver | 22% | Rp 0 (exempt) | Rp 26,276,586 |
| Bottom 4 | SHP-019 (Footwear Kingdom) | SME Growth | 22% | Rp 1,604,500 | Rp 27,518,832 |
| Bottom 5 | SHP-025 (Toyland Direct) | Enterprise Silver | 22% | Rp 1,627,500 | Rp 27,595,337 |
| Benchmark | SHP-022 (Office Depot Indo) | Enterprise Gold | 8% | Rp 1,642,500 | Rp 49,011,071 |
| Benchmark | SHP-016 (Organic Food Express) | Key Account | 5% | Rp 0 (exempt) | Rp 43,654,882 |

### Route profitability: RT-002 (Jabodetabek to Bandung)

RT-002 is the only corridor below the 10% critical margin threshold, with an average shipment margin of 7.82% (7.70% across its 448 active DOs; 32.33% on a weighted basis).

**Root cause:** the tariff barely clears direct cost, and small parcels fall below it once discounts and penalties apply.

| Item | Value |
|---|---|
| Tariff per KG (Standard / Express / Cargo) | Rp 11,000 / Rp 15,399 / Rp 8,250 |
| Linehaul cost per KG (base, with 5% fuel surcharge) | Rp 5,097 (Rp 5,352) |
| Fixed handling per parcel | Rp 8,300 (pickup 2,500 + sorting 1,800 + last mile 4,000) |

When contract discounts of up to 25% and Rp 3,000 per day Intra-Java SLA penalties are applied to light parcels (1-2 kg), net revenue falls below direct handling cost.

### Service tier unit economics

| Service | Orders | Gross profit | Gross margin |
|---|---|---|---|
| Express | 3,836 | Rp 483.44M | 46.85% |
| Standard | 6,341 | Rp 447.17M | 35.96% |
| Cargo | 1,823 | Rp 83.09M | 27.42% |

### SLA penalty leakage

Late deliveries cost Rp 34,987,500 in revenue deductions across non-exempt shippers. The largest single shipper-route concentration is SHP-019 on RT-014 (Rp 155,000). SHP-028 (Rp 1,739,000) and RT-016, Jabodetabek to Balikpapan (Rp 1,740,000), are the largest cumulative shipper and corridor drivers.

### Volume rebate liability

Nine shippers (SHP-002, 004, 006, 007, 008, 009, 016, 024, 028) exceed their annual volume threshold of 300 shipments, triggering **Rp 21,177,288** in year-end rebate payouts.

### AR and cash-flow risk

5,941 invoices (Unpaid 3,157 and Overdue 2,784) are still open, worth about Rp 1.27 billion. Paid invoices make up 50.75% of billed value, Unpaid 26.16% and Overdue 23.09%.

| Shipper tier | Overdue | Unpaid | Open AR |
|---|---|---|---|
| Key Account | Rp 156.56M | Rp 196.54M | Rp 353.11M |
| Enterprise Gold | Rp 127.97M | Rp 137.94M | Rp 265.91M |
| SME Starter | Rp 117.86M | Rp 132.16M | Rp 250.02M |
| Enterprise Silver | Rp 111.93M | Rp 121.16M | Rp 233.10M |
| SME Growth | Rp 80.25M | Rp 85.95M | Rp 166.20M |

## Recommendations

| # | Initiative | Action plan | Estimated impact |
|---|---|---|---|
| 1 | **Reprice RT-002** | Raise base tariff by 15-20% (Standard to Rp 12,650-13,200/kg, Express to Rp 17,709-18,479/kg, Cargo to Rp 9,488-9,900/kg). Introduce a Rp 12,000 minimum net shipping charge per parcel on Intra-Java routes so fixed handling cost (Rp 8,300) is always covered. | +15%: +Rp 4.13M gross profit (unweighted margin 7.82% to 26.86%).<br>+20%: +Rp 5.50M gross profit (to 29.98%). |
| 2 | **Restructure shipper discounts** | Renegotiate SME Starter/Growth rate cards (SHP-014 at 25%, SHP-019 at 22%) to tier ceilings of 8-12%. Cap combined concessions (contract discount plus volume rebate) at 20% for Key Account and Silver tiers, which removes SHP-006's 25% combined concession. | Cutting the bottom five shippers from 22-25% to a 15% tier-aligned cap recovers +Rp 41.8M to +Rp 52.0M gross profit per year. |
| 3 | **Linehaul and SLA optimisation** | Audit carrier SLAs on RT-016 (Jabodetabek to Balikpapan) and RT-014 to remove repeat late-delivery bottlenecks. Recalibrate Cargo tariff and minimum weight. | Cutting SLA breaches by 40% recovers +Rp 14.0M net revenue. Cargo recalibration adds +Rp 25.0M to +Rp 30.3M gross profit. |

Impact figures are estimates derived from the FY 2025 synthetic dataset.

## Dashboard

![Pricing & Profitability Dashboard](images/dashboard.png)

The executive dashboard contains:

- **KPI cards:** total delivery orders, net revenue, gross profit with gross margin %, and outstanding AR.
- **Monthly financial and billing trend:** monthly revenue and monthly gross margin %, with a month slicer.
- **Invoice payment status breakdown:** Paid / Unpaid / Overdue split, with a payment status slicer.
- **Top and bottom 5 shipper profitability:** by gross profit, with a Shipper ID slicer.
- **Route margin alert:** routes below 10% average margin, shown with their current tariffs and a price recommendation.

## Tools & Skills

- **Microsoft Excel:** XLOOKUP, INDEX/MATCH, VLOOKUP, SUMIFS, IFS, SWITCH, MAX, GETPIVOTDATA, structured tables, PivotTables, slicers and charts
- **Python:** synthetic dataset generation
- **Data cleansing and validation:** text normalisation, type coercion, date standardisation, null-safe formulas
- **Financial analysis:** unit economics, margin modelling, revenue leakage, AR aging, volume rebate evaluation


## Documented Edge Cases & Maintenance Notes

- **Weighted vs unweighted margin:** on low-weight parcels with high discounts and fixed SLA penalties, row-level Gross Margin % ranges from -5,718% to +2,041%. Portfolio margin is therefore reported on a weighted basis (total gross profit / total net revenue = 43.19%).
- **Pivot cache range:** some pivot caches reflect an earlier 12,049-row staging range instead of the 12,000-row baseline and need a refresh against the trimmed range.
- **Helper column reference:** Invoice Payment Status Pct has #REF! in helper cells C3:C5, to be re-pointed to =B3/$B$6.
- **Rebate formula:** in Volume Rebate Evaluation, the GETPIVOTDATA shipper argument is hardcoded and should reference each row's shipper cell.
- **Cargo billable weight:** the Cargo branch of the billable-weight formula should compare the 10 kg minimum against actual weight (column I) rather than the dimension text (column K).

## Author

**Yosafat Kevin Sirumapea**
Data Analyst / ERP Functional Consultant, Jakarta, Indonesia
GitHub: [@yoskev](https://github.com/yoskev)
