# SaaS Cohort Retention & Scenario Model

An Excel model that turns a monthly subscription ledger into cohort retention, net revenue retention, unit economics, and a Monte Carlo forecast. No macros, so it runs on a locked-down work laptop.

## What it shows

The headline is a gap that one blended number would hide. By month 12 the business has lost about 30% of its customers, but it keeps about 95% of the revenue, because the ones who stay upgrade. So the problem to fix is early churn, not pricing.

## The numbers

- 2,000 customers, 24 months, 17,585 rows
- Customer retention about 70% at month 12; net revenue retention about 95%
- ARPA about $204 a month, average lifetime about 34 months, gross LTV about $5,555
- Monte Carlo, 1,000 runs and reproducible: next-year revenue retention lands near 95%, with a range of about 83% to 110%

## How it is built

- **Power Query** loads every monthly CSV from a folder, promotes headers, combines them, types the columns, and derives each row's tenure in months from real signup and activity dates. The folder is a parameter (`FolderPath`), so pointing it at new data and refreshing is the only step to update everything downstream.
- **Power Pivot (DAX)** defines each metric once — cohort size, active customers, total and starting MRR, logo retention, net revenue retention, ARPA — and the one PivotTable reads straight from that model.
- The **Cohort, NRR and LTV sheets** reproduce the same logic with classic `COUNTIFS` / `SUMIFS` against named ranges, so the shipped file shows correct numbers and opens with zero errors on any modern Excel, even with the data connection switched off.
- A separate sheet shows the same simulation written with **dynamic-array formulas** (spill), for readers on Excel 365.
- The **Monte Carlo** runs 1,000 scenarios. Each scenario draws a monthly churn rate and a monthly net-expansion rate from a normal distribution (the means and standard deviations live on the Assumptions sheet), compounds them across 12 months, and the sheet reports the mean, P5, P95, and the share of scenarios that fully retain revenue.
- A small **reconciliation block** on the dashboard cross-checks the cohort counts two different ways and flags `OK` only if they agree.

## The metrics, in DAX

Every metric is defined once in the data model and reused. The anchor for both cohort measures is the first month of a customer's life (`tenure_months = 0`), held with `ALLEXCEPT` on `signup_month`. `DIVIDE` is used throughout so an empty cohort never throws a divide-by-zero.

```DAX
Active Customers := DISTINCTCOUNT ( Ledger[customer_id] )

Total MRR := SUM ( Ledger[mrr] )

Cohort Size :=
CALCULATE (
    DISTINCTCOUNT ( Ledger[customer_id] ),
    FILTER ( ALLEXCEPT ( Ledger, Ledger[signup_month] ), Ledger[tenure_months] = 0 )
)

Starting MRR :=
CALCULATE (
    [Total MRR],
    FILTER ( ALLEXCEPT ( Ledger, Ledger[signup_month] ), Ledger[tenure_months] = 0 )
)

Logo Retention % := DIVIDE ( [Active Customers], [Cohort Size] )

Net Revenue Retention % := DIVIDE ( [Total MRR], [Starting MRR] )

ARPA := DIVIDE ( [Total MRR], [Active Customers] )
```

## The data load, in Power Query (M)

One query reads the whole `ledger` folder, so dropping in a new month of CSVs and refreshing is all it takes to update the model. Tenure is computed here, in months, from parsed dates — not with text math.

```m
section Section1;

shared Ledger = let
    // FolderPath is a text parameter pointing at the folder of monthly CSVs
    Source   = Folder.Files(FolderPath),
    OnlyCsv  = Table.SelectRows(Source, each Text.Lower([Extension]) = ".csv"),
    // open and header-promote each file, so every file's own header row is consumed
    Parsed   = Table.AddColumn(OnlyCsv, "Data", each
        Table.PromoteHeaders(
            Csv.Document([Content], [Delimiter = ",", Encoding = 65001, QuoteStyle = QuoteStyle.Csv]),
            [PromoteAllScalars = true]
        )
    ),
    Combined = Table.Combine(Parsed[Data]),
    Typed    = Table.TransformColumnTypes(Combined, {
        {"customer_id", type text}, {"signup_month", type text},
        {"activity_month", type text}, {"mrr", type number},
        {"plan_tier", type text}, {"region", type text}
    }),
    AddSignup   = Table.AddColumn(Typed,      "signup_date",   each Date.FromText([signup_month]   & "-01"), type date),
    AddActivity = Table.AddColumn(AddSignup,  "activity_date", each Date.FromText([activity_month] & "-01"), type date),
    AddTenure   = Table.AddColumn(AddActivity,"tenure_months",
        each (Date.Year([activity_date]) - Date.Year([signup_date])) * 12
           + (Date.Month([activity_date]) - Date.Month([signup_date])), Int64.Type)
in
    AddTenure;

// Set this to the /ledger folder on your own machine before refreshing.
shared FolderPath = "<path-to-the-ledger-folder-in-this-repo>"
    meta [IsParameterQuery = true, Type = "Text", IsParameterQueryRequired = true];
```

## How to open and refresh

1. Download or clone the repo. Keep `saas_mrr_ledger.csv` inside the `ledger` folder.
2. Open `cohort_model.xlsx`. It shows correct numbers immediately — nothing needs to run.
3. To refresh from data: **Data → Queries & Connections**, set the `FolderPath` parameter to your local `ledger` folder, then **Refresh All**.
4. No macros are used, so it works on a restricted machine.

## Files

<<<<<<< HEAD
- `cohort_model.xlsx` — the working model: Power Query load, Power Pivot model, dashboard, and the per-metric worksheets
- `ledger/saas_mrr_ledger.csv` — the data, one row per customer per active month
- `SaaS_Cohort_Model_Documentation.pdf` — the full write-up in plain language, with the charts drawn in
- `README.md` — this file: the numbers, the DAX, the M, and the build steps
=======
- `cohort_model.xlsx`: the working model
- `saas_mrr_ledger.csv`: the data, one row per customer per active month
- `Project_E_Documentation_Report.pdf`: the full write-up in plain language, with the charts drawn in
>>>>>>> 14dced6ac3645467ea47931717c6a110724dc37a

## A note on the data

The ledger is synthetic, generated from a documented behavioural model, because no public SaaS dataset is both a proper monthly ledger and small enough for Excel. The behaviour is realistic; the figures are illustrative, not from a real company.

## If this were real data

Because the ledger is synthetic, the cohort shape was set by the assumptions, not discovered in the wild. A real customer export would need a cleaning layer this file does not have to exercise. The model and the DAX would not change — the extra work lands entirely in the Power Query step:

- **De-duplicate** customer-month rows. A billing correction can write the same customer-month twice.
- **Handle gaps.** A customer who churns and later reactivates leaves a missing month that breaks naive tenure counting.
- **Decide on refunds, credits, and partial months** before counting — either as negative MRR or excluded, but consistently.
- **Reconcile mid-month plan changes** so MRR is the month-end figure, not the sum of two part-months.
- **Confirm the cohort anchor** is the first *paid* month, not the signup or trial month.

That cleaning is the part this project shows the tooling for, but — on synthetic data — does not have to prove.
