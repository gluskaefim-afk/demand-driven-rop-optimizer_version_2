# Demand-Driven ROP Optimizer

Inventory policy optimization for a single SKU (`SKU-1001`): demand forecasting, order quantity (EOQ) with constraints, and a Monte Carlo (bootstrap) simulation that calibrates the Reorder Point (ROP) to a target service level.

## Overview

The project answers two questions for the upcoming month:

1. **How much to order at once?** (Q, order quantity)
2. **When to place an order?** (ROP, reorder point)

Pipeline: `Prophet forecast → constrained EOQ → period selection → bootstrap simulation → ROP calibration`

## Data

- `demand_dataset.xlsx`: daily demand history, filtered to `SKU-1001` (Jan 2023 - Sep 2025).
- The dataset is **synthetic (AI-generated)**, so the results demonstrate the methodology, not real-world performance.

## Method

### 1. Demand forecast (Prophet)
Prophet (linear growth, yearly + weekly + custom monthly seasonality) forecasts the next 31 days. The forecast is **not** used for day-by-day simulation. It is used to:
- estimate total demand for the horizon (**D = 3,698 units**), the input for EOQ;
- choose historical months with a similar demand level for resampling.

### 2. Order quantity (constrained EOQ)
Instead of only applying the classic formula, the total cost (ordering + holding) is minimized numerically with SLSQP (`scipy.optimize`) under explicit constraints:

| Parameter | Value |
|---|---|
| Ordering (transport) cost | €500 per order |
| Holding cost | €1.55 per unit per month (€0.05 per unit per day) |
| Warehouse capacity | Q ≤ 2,000 units |
| Supplier limit | Q ≤ 5,000 units per delivery |
| Minimum replenishments | R ≥ 1 per month |

**Result: Q = 1,545 units.** The constraints are non-binding, so the solution matches the classic formula `√(2·D·b/h)`, which confirms the model. The per-unit shipping discount (−€0.429) is a constant term in the objective and does not affect the optimum.

### 3. Period selection for bootstrap
Six historical months with total demand closest to the forecast are used as the resampling pool: `2023-02, 2023-07, 2023-08, 2023-12, 2024-08, 2025-08`.

### 4. Monte Carlo simulation
For each of 1,000 simulated months (31 days):
- daily demand is drawn with replacement from the pooled days of the selected months;
- when on-hand stock falls to the ROP, an order of Q = 1,545 is placed;
- the order arrives after the supplier Lead Time (5 days in the model's counter; the order day is counted as day 1);
- unmet demand is treated as lost sales;
- costs are tracked: storage €0.05 per unit per day, €500 per order.

### 5. ROP calibration
The ROP starts at 400 and increases in steps of 10 until the average **Full-Coverage Days** (share of days without a stockout) falls into the 95-100% corridor.


## Model algorithm
![Uploading Group 18 (1).png…]()


## Results

| Metric | Value |
|---|---|
| Optimal ROP | **500 units** |
| Full-Coverage Days, average | 95.05% |
| Full-Coverage Days, worst case (99.9th percentile) | 93.55% |
| Fill Rate (monthly average) | 97.88% |
| Expected cost (storage + ordering) | €2,374.74 / month |
| Peak inventory (Q Max) | 1,680 units (within the 2,000 limit) |

At ROP = 500 a stockout occurs on about 5% of days (roughly 1-2 days per month), and 97.88% of demand units are served directly from stock.

## Limitations

- **Synthetic data**: results are illustrative.
- **Deterministic Lead Time**: supplier delays are not simulated (can be extended by resampling Lead Time from historical deliveries).
- **Independent daily demand**: days are resampled independently, so weekly seasonality and autocorrelation are ignored. The pool is small (~186 days), so the tail of the distribution is limited by observed values.
- **Single-month horizon**: each simulated month starts with full stock and no open orders, which makes the service level optimistic.
- **Service metric**: calibration targets the average share of stockout-free days, and the worst-case value is reported but not guaranteed. Cycle-level service is not measured.
- **Monte Carlo noise**: the result is close to the lower bound of the corridor (95.05% vs 95%), so neighbouring ROP values are statistically hard to distinguish.
- **Costs**: only storage and ordering costs are optimized. Stockout costs and production downtime are not included.
- **Forecast**: no backtesting of the Prophet forecast is included.

## Possible extensions

- Stochastic Lead Time from historical deliveries
- Warm-up period and common random numbers (fixed seed per run)
- Cycle Service Level and Fill Rate as calibration targets
- Day-of-week or block bootstrap
- Stockout cost and a cost-optimal ROP
- Backtest on historical data and multi-SKU scaling (ABC/XYZ)
