# Retail Demand Forecasting in Power BI

A transaction-based retail demand forecasting project built in **Power
BI, DAX, and Power Query (M)**. The project converts transaction-level
retail data into a daily SKU-level forecasting dataset, develops three
forecasting approaches, backtests them, and compares accuracy and bias.

## Objectives

-   Convert transaction-level retail data into a clean **StockCode ×
    Date** forecasting dataset.
-   Build a continuous DateTable and complete SKU × Date grid.
-   Create lag and 7-day rolling-demand calculations.
-   Develop and compare:
    -   7-day Rolling Average
    -   WMA3
    -   WMA5
-   Perform a backtest without allowing future actual demand to leak
    into the recursive WMA forecasts.
-   Evaluate models using **MAE** and **ME**.
-   Document practical Power BI/DAX/Power Query troubleshooting.

## Data and Forecasting Period

The final modelling calendar covers **25 June 2011 -- 25 July 2011**.

For the final recursive WMA backtest:

-   Last actual available to the model: **13 July 2011**
-   Forecast period: **14 July -- 25 July 2011**
-   Actual Qty remains visible during the test period only for
    evaluation.

## Workflow

``` text
Transaction Data
      ↓
Cleaning / Validation
      ↓
Daily SKU Aggregation
      ↓
Complete SKU × Date Grid
      ↓
Continuous DateTable
      ↓
forecast_ready
      ↓
 ┌──────────────────────────┐
 │ Rolling Average          │
 │ Recursive WMA3 / WMA5    │
 └──────────────────────────┘
      ↓
Backtest
      ↓
Forecast vs Actual
      ↓
MAE + ME
      ↓
Model Comparison
```

## 1. DateTable

A continuous DateTable was created for exactly 25 June through 25 July
2011.

It contains:

-   Date
-   Year
-   Month Number
-   Month Name
-   Day Name
-   Day of Week
-   Is Weekend
-   Year-Month

The continuous calendar is required so time-based calculations do not
lose dates where a SKU had no transaction.

## 2. Daily Aggregation and Complete SKU Grid

Transactions were aggregated to **StockCode × Date** so multiple
transactions for the same SKU/day became one daily demand observation.

A complete SKU × Date grid was then created. This ensures every SKU has
a row for every calendar date.

Formula/code for this step is intentionally omitted from this README.

Historical no-sale days are represented as zero demand. Future unknown
demand is not treated as historical zero demand.

## 3. Time-Series Calculations

A one-day lag was created with DAX:

``` dax
Qty_Lag1 =
CALCULATE(
    SUM(forecast_ready[Qty]),
    FILTER(
        ALL(forecast_ready),
        forecast_ready[StockCode_Clean] =
            MAX(forecast_ready[StockCode_Clean])
            &&
        forecast_ready[Date] =
            MAX(forecast_ready[Date]) - 1
    )
)
```

A seven-day rolling quantity was created using:

``` dax
Qty_Roll7 =
CALCULATE(
    SUM(forecast_ready[Qty]),
    DATESINPERIOD(
        DateTable[Date],
        MAX(DateTable[Date]),
        -7,
        DAY
    ),
    FILTER(
        ALL(forecast_ready),
        forecast_ready[StockCode_Clean] =
            MAX(forecast_ready[StockCode_Clean])
    )
)
```

`ALL()` allows the calculation to search the full table, while the SKU
condition puts the calculation back onto the current SKU.
`DATESINPERIOD()` creates the seven-day window.

## 4. 7-Day Rolling Average

The baseline forecast was:

``` dax
Forecast_Roll7 =
CALCULATE(
    DIVIDE([Qty_Roll7], 7, 0),
    forecast_ready[Test/Train] = "Test"
)
```

This averages the previous seven days of demand.

The initial Train/Test flag was:

``` dax
Test/Train =
IF(
    forecast_ready[Date] < DATE(2011,7,15),
    "Train",
    "Test"
)
```

## 5. Recursive WMA3 and WMA5

The final WMA process uses actual data only through 13 July and
forecasts 14--25 July.

### WMA3

Weights:

``` text
3, 2, 1
```

Formula:

``` text
WMA3 = (Latest × 3 + Previous × 2 + Previous × 1) / 6
```

### WMA5

Weights:

``` text
5, 4, 3, 2, 1
```

Formula:

``` text
WMA5 =
(Latest × 5 + Previous × 4 + Previous × 3
 + Previous × 2 + Previous × 1) / 15
```

### Recursive M logic

The final Power Query script uses `List.Accumulate` to carry the
forecast state forward:

``` text
Actuals through 13 July
        ↓
Forecast 14 July
        ↓
Use 14 July forecast as next input
        ↓
Forecast 15 July
        ↓
Repeat through 25 July
```

The critical point is that actual quantities from the test period are
**not** fed back into the WMA calculation. This prevents data leakage.

The full M script is included in the project's detailed Word
documentation.

## 6. Forecast Evaluation

The project uses:

### Signed Error

``` text
Error = Forecast - Actual
```

Therefore:

-   Positive = over-forecast
-   Negative = under-forecast
-   Zero = exact forecast

### MAE

``` text
MAE = Average(ABS(Forecast - Actual))
```

MAE measures forecast accuracy.

**Lower MAE = better.**

### ME

``` text
ME = Average(Forecast - Actual)
```

ME measures directional bias.

A value close to zero means less overall bias, but it does not
necessarily mean the forecasts are accurate because positive and
negative errors can cancel.

## 7. Rolling Average Total Troubleshooting

The Rolling Average forecast values were correct at row level, but the
Power BI grand total initially returned an incorrect value.

The issue was caused by the forecast measure being re-evaluated in the
grand-total filter context rather than simply summing the visible
SKU-date forecasts.

A total-specific measure was created:

``` dax
Forecast_Roll7_Total =
SUMX(
    SUMMARIZE(
        'forecast_ready',
        'forecast_ready'[Date],
        'forecast_ready'[StockCode_Clean]
    ),
    [Forecast_Roll7]
)
```

`SUMX` forces Power BI to evaluate the forecast at the Date × SKU grain
and then add those row-level results.

## 8. Troubleshooting

Key issues resolved during development:

-   Corrected/checked the DateTable relationship and compatible Date
    columns for time intelligence.
-   Used **Date + StockCode_Clean** as the merge key when bringing WMA
    forecasts back to the complete grid.
-   Changed the DateTable to an explicit 25 June--25 July range.
-   Changed the WMA setup from a future forecast after the last actual
    to a historical backtest with a 13 July cutoff.
-   Removed historical WMA forecasts so WMA3/WMA5 appear only from the
    intended forecast period.
-   Prevented future actual quantities from entering the recursive WMA
    calculation.
-   Fixed the Rolling Average grand-total calculation with `SUMX`.
-   Distinguished signed error from absolute error so bias and accuracy
    are evaluated separately.

## 9. Results

  --------------------------------------------------------------------------
  Model                            MAE                   ME Interpretation
  --------------- -------------------- -------------------- ----------------
  WMA3                           10.21                -8.52 Highest error;
                                                            under-forecast
                                                            bias

  WMA5                            9.11                -6.22 Better than
                                                            WMA3;
                                                            under-forecast
                                                            bias

  Rolling Average             **7.94**            **+1.61** Lowest error;
                                                            slight
                                                            over-forecast
                                                            bias
  --------------------------------------------------------------------------

### Outcome

The **7-day Rolling Average performed best on this backtest**.

-   Rolling Average MAE: **7.94**
-   WMA5 MAE: **9.11**
-   WMA3 MAE: **10.21**

The Rolling Average also had the smallest absolute bias among the three:

-   Rolling Average ME: **+1.61**
-   WMA5 ME: **-6.22**
-   WMA3 ME: **-8.52**

Under the project's `Forecast - Actual` convention, the negative WMA ME
values indicate under-forecasting.

## 10. Key Takeaways

1.  Forecasting requires a consistent **SKU × Date** grain.
2.  A complete SKU-date grid prevents missing observations from
    distorting time-series calculations.
3.  A moving average calculated with known actuals is not the same as a
    genuine multi-step forecast.
4.  Recursive forecasts must use previous forecasts when future actuals
    are unavailable.
5.  `List.Accumulate` makes recursive forecasting practical in Power
    Query.
6.  MAE measures accuracy; ME measures directional bias.
7.  DAX filter context can cause row-level measures to produce
    unexpected grand totals.
8.  On this backtest, the **7-day Rolling Average was the
    best-performing method**.

## Repository Structure

``` text
retail-demand-forecasting-powerbi/
│
├── README.md
├── data/
│   └── online_retail_10sku_extended_july25.csv
├── power-query/
│   └── WMA3_WMA5_Recursive_Backtest.m
├── dax/
│   ├── Rolling_Average_Measures.dax
│   └── Forecast_Error_Metrics.dax
├── powerbi/
│   └── Retail_Demand_Forecasting.pbix
└── documentation/
    └── Forecasting_Project_Steps_Taken.docx
```

## Limitations and Next Steps

This is a short demonstration backtest, so the result should not be
treated as proof that Rolling Average will always outperform the WMA
methods.

Next steps could include:

-   Additional historical holdout periods
-   Testing different rolling windows
-   SKU-level model selection
-   RMSE
-   Seasonal effects
-   Promotions and holidays
-   Additional statistical or machine-learning forecasting models

## Tools

-   Microsoft Power BI
-   Power Query / M
-   DAX
-   Excel/tabular data concepts

## Project Outcome

This project demonstrates an end-to-end retail forecasting workflow:

**transaction data → daily demand → complete SKU-date grid → forecasting
models → recursive backtesting → error metrics → model comparison**

For the evaluated period, the **7-day Rolling Average was selected as
the best-performing method based on MAE**.
