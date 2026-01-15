# Cisco Forecast League Competition

## Overview

This project is a demand forecasting solution developed for the Cisco Forecast League Competition. The goal is to predict product demand (units sold) for Cisco products in FY25 Q2 (Fiscal Year 2025, Quarter 2) based on historical quarterly sales data spanning from FY22 Q2 to FY25 Q1.

## Table of Contents

1. [Problem Statement](#problem-statement)
2. [Dataset Description](#dataset-description)
3. [Methodology Overview](#methodology-overview)
4. [Models Used](#models-used)
5. [Forecasting Process](#forecasting-process)
6. [Scenario Examples](#scenario-examples)
7. [Accuracy Evaluation](#accuracy-evaluation)
8. [Project Structure](#project-structure)

---

## Problem Statement

Cisco needs accurate demand forecasts to optimize:
- **Inventory Management**: Prevent overstocking or stockouts
- **Supply Chain Planning**: Ensure timely production and delivery
- **Resource Allocation**: Optimize manufacturing and logistics

The challenge involves predicting FY25 Q2 demand for various product categories including:
- Switches (Enterprise, Data Center)
- Access Points
- Transceiver Modules
- Wireless Controllers

---

## Dataset Description

### Input Data Columns

| Column | Description |
|--------|-------------|
| `Cost Rank` | Product ranking by cost/importance (1-20) |
| `Product Name` | Product category (e.g., "SWITCH Enterprise High") |
| `Product Life Cycle` | Product stage: Sustaining, Decline, or NPI (New Product Introduction) |
| `FY22 Q2 - FY25 Q1` | Historical quarterly sales (units) spanning 12 quarters |
| `Your Forecast FY25 Q2` | Target column for predictions |

### Product Life Cycle Categories

1. **Sustaining**: Mature products with stable demand patterns
2. **Decline**: Products phasing out with decreasing demand
3. **NPI (New Product Introduction)**: New products with limited historical data

---

## Methodology Overview

The forecasting approach uses an **ensemble methodology** that combines multiple forecasting techniques based on product characteristics:

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA PREPROCESSING                            │
│  • Handle missing values (interpolation)                        │
│  • Reshape data for time series analysis                        │
│  • Feature engineering (seasonality, trends)                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│               SEASONAL DECOMPOSITION                            │
│  • Additive Decomposition                                       │
│  • Multiplicative Decomposition                                 │
│  • Extract: Trend, Seasonal, Residual components               │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                 MODEL SELECTION                                  │
│  Based on Product Life Cycle:                                   │
│  • Sustaining → Weighted MA + Holt-Winters                      │
│  • Decline → YoY Growth + MA                                    │
│  • NPI → Holt-Winters + Recent Performance                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  ENSEMBLE FORECAST                              │
│  Weighted combination of multiple models                        │
│  Final Forecast = Σ(weight_i × model_i_forecast)               │
└─────────────────────────────────────────────────────────────────┘
```

---

## Models Used

### 1. Simple Moving Average (MA)

**What is it?**
Simple Moving Average is one of the most fundamental forecasting techniques. It calculates the average of a fixed number of recent observations to predict the next value. By averaging past data points, it "smooths" out random fluctuations and noise to reveal the underlying trend.

**How it works:**
1. Select a window size (n) - typically 4 quarters for quarterly data
2. Sum the last n observations
3. Divide by n to get the average
4. Use this average as the forecast for the next period

**Formula**:
```
MA(t) = (1/n) × Σ(Sales from t-n+1 to t)

Example with n=4:
MA = (Q1_Sales + Q2_Sales + Q3_Sales + Q4_Sales) / 4
```

**Implementation**:
```python
# Last 4 quarters moving average
ma_forecast = np.mean(valid_values[-4:])

# Example: If last 4 quarters are [22084, 21830, 29404, 24518]
# MA = (22084 + 21830 + 29404 + 24518) / 4 = 24,459
```

**Advantages**:
- Simple to understand and implement
- No assumptions about data distribution
- Works well for stable, non-trending data
- Resistant to outliers (smoothing effect)

**Limitations**:
- Lags behind trends (slow to react to changes)
- Gives equal weight to all observations in the window
- Cannot capture seasonality
- Not suitable for rapidly changing demand

**When to use**: Best for products with relatively stable demand patterns, mature products in the "Sustaining" phase with minimal trend.

---

### 2. Weighted Moving Average (WMA)

**What is it?**
Weighted Moving Average improves upon Simple MA by assigning different weights to each observation. More recent data points receive higher weights, making the forecast more responsive to recent changes while still considering historical data.

**How it works:**
1. Assign weights to each period (higher weights for recent periods)
2. Multiply each observation by its weight
3. Sum the weighted values
4. Divide by the sum of weights

**Formula**:
```
WMA(t) = Σ(w_i × Sales_i) / Σ(w_i)

With weights [0.1, 0.2, 0.3, 0.4] for 4 quarters:
- Oldest quarter (t-3): 10% weight
- Quarter (t-2): 20% weight  
- Quarter (t-1): 30% weight
- Most recent (t): 40% weight
```

**Implementation**:
```python
weights = np.array([0.1, 0.2, 0.3, 0.4])
wma_forecast = np.sum(weights * valid_values[-4:]) / np.sum(weights)

# Example: If last 4 quarters are [22084, 21830, 29404, 24518]
# WMA = (0.1×22084 + 0.2×21830 + 0.3×29404 + 0.4×24518) / 1.0
# WMA = (2208 + 4366 + 8821 + 9807) / 1.0 = 25,202
```

**Advantages**:
- More responsive to recent trends than Simple MA
- Flexible weight assignment based on domain knowledge
- Still provides smoothing effect
- Better for trending data

**Limitations**:
- Weight selection can be subjective
- Still cannot capture complex seasonality
- May overreact to recent anomalies if weights are too aggressive

**When to use**: When recent trends are more indicative of future demand, especially for products showing gradual changes in demand patterns.

---

### 3. Year-over-Year (YoY) Growth Rate

**What is it?**
Year-over-Year analysis compares the same period (quarter) across different years to capture annual seasonality. It calculates the growth rate between corresponding periods and applies this rate to project future demand.

**How it works:**
1. Compare current quarter with the same quarter from last year
2. Calculate the growth rate (ratio of current to previous year)
3. Apply this growth rate to project the next corresponding quarter

**Formula**:
```
YoY_Rate = Sales(current_Q2) / Sales(previous_year_Q2)
Forecast_next_Q2 = Sales(current_Q2) × YoY_Rate

Example:
- FY24 Q2 Sales: 22,084
- FY23 Q2 Sales: 27,114
- YoY Rate = 22,084 / 27,114 = 0.814 (18.6% decline)
- FY25 Q2 Forecast = 22,084 × 0.814 = 17,976
```

**Implementation**:
```python
# Compare with same quarter last year (4 quarters back for quarterly data)
yoy_rate = valid_values[-1] / valid_values[-5] if valid_values[-5] > 0 else 1
yoy_forecast = valid_values[-1] * yoy_rate
```

**Advantages**:
- Captures annual seasonality patterns
- Intuitive for business stakeholders
- Works well for products with strong yearly cycles
- Accounts for seasonal peaks and troughs

**Limitations**:
- Requires at least 2 years of data
- Assumes patterns repeat exactly
- Sensitive to anomalies in the comparison period
- Cannot handle structural changes in demand

**When to use**: Products with strong seasonal patterns (e.g., networking equipment with fiscal year-end buying cycles, holiday-driven demand).

---

### 4. Holt-Winters Exponential Smoothing

**What is it?**
Holt-Winters (also called Triple Exponential Smoothing) is a sophisticated method that decomposes time series into three components and forecasts each separately. It can handle data with both trend and seasonality, making it powerful for complex demand patterns.

**The Three Components:**
1. **Level (α - alpha)**: The base value or average of the series
2. **Trend (β - beta)**: The direction and rate of change over time
3. **Seasonality (γ - gamma)**: Repeating patterns within a fixed period

**How it works:**
```
The model updates three equations at each time step:

Level:     L(t) = α × (Y(t)/S(t-s)) + (1-α) × (L(t-1) + T(t-1))
Trend:     T(t) = β × (L(t) - L(t-1)) + (1-β) × T(t-1)
Seasonal:  S(t) = γ × (Y(t)/L(t)) + (1-γ) × S(t-s)

Forecast:  F(t+h) = (L(t) + h×T(t)) × S(t+h-s)

Where:
- Y(t) = Actual value at time t
- s = Seasonal period (4 for quarterly)
- h = Forecast horizon
```

**Two Variants:**
- **Additive**: Seasonal variation is constant (e.g., +5000 units in Q4)
- **Multiplicative**: Seasonal variation is proportional (e.g., 20% higher in Q4)

**Implementation**:
```python
from statsmodels.tsa.holtwinters import ExponentialSmoothing

model = ExponentialSmoothing(
    filled_series, 
    trend='add',           # Additive trend (constant growth)
    seasonal='add',        # Additive seasonality
    seasonal_periods=4     # Quarterly data (4 periods/year)
).fit()
hw_forecast = model.forecast(1)[0]

# The model automatically optimizes α, β, γ parameters
```

**Advantages**:
- Handles both trend and seasonality simultaneously
- Automatically adapts to changing patterns
- Parameters are optimized from data
- Provides confidence intervals for forecasts

**Limitations**:
- Requires sufficient historical data (at least 2 seasonal cycles)
- Assumes stable seasonal patterns
- Can be sensitive to outliers
- More complex to interpret than simple methods

**When to use**: Products with clear trend and seasonal patterns, especially when both components are significant.

---

### 5. SARIMA (Seasonal ARIMA)

**What is it?**
SARIMA (Seasonal AutoRegressive Integrated Moving Average) is one of the most powerful classical time series models. It extends ARIMA by adding seasonal components, making it capable of modeling complex patterns including trends, seasonality, and autocorrelation.

**Model Components Explained:**

**ARIMA(p,d,q)** - Non-seasonal part:
- **p (AR - AutoRegressive)**: How many past values influence the current value
- **d (I - Integrated)**: How many times to difference the data to make it stationary
- **q (MA - Moving Average)**: How many past forecast errors influence the current value

**Seasonal (P,D,Q,s)** - Seasonal part:
- **P**: Seasonal autoregressive order
- **D**: Seasonal differencing
- **Q**: Seasonal moving average order
- **s**: Seasonal period (4 for quarterly, 12 for monthly)

**How it works:**
```
The general SARIMA equation:

Φ(B^s)φ(B)(1-B)^d(1-B^s)^D × Y(t) = Θ(B^s)θ(B) × ε(t)

Where:
- B is the backshift operator (B×Y(t) = Y(t-1))
- φ(B) is the non-seasonal AR polynomial
- Φ(B^s) is the seasonal AR polynomial
- θ(B) is the non-seasonal MA polynomial
- Θ(B^s) is the seasonal MA polynomial
- ε(t) is white noise

Simplified: Current value depends on:
1. Past values (AR terms)
2. Past forecast errors (MA terms)
3. Seasonal patterns from previous years
```

**Implementation**:
```python
from statsmodels.tsa.statespace.sarimax import SARIMAX

model = SARIMAX(
    time_series,
    order=(1, 1, 1),            # (p, d, q) - ARIMA parameters
    seasonal_order=(1, 1, 1, 4)  # (P, D, Q, s) - Seasonal parameters
)
fitted_model = model.fit(disp=False)
sarima_forecast = fitted_model.forecast(1)

# Common orders for quarterly data:
# order=(1,1,1) - Simple model with AR(1), differencing, MA(1)
# seasonal_order=(1,1,1,4) - Seasonal AR, differencing, MA with period 4
```

**Advantages**:
- Very flexible - can model complex patterns
- Handles non-stationarity through differencing
- Captures autocorrelation in residuals
- Well-established theoretical foundation

**Limitations**:
- Requires careful parameter selection (p,d,q,P,D,Q)
- Needs stationary data (or differencing)
- Computationally more intensive
- Sensitive to outliers and missing data

**When to use**: Products with complex seasonal patterns, strong autocorrelation, and when simple methods underperform.

---

### 6. XGBoost (Machine Learning Approach)

**What is it?**
XGBoost (eXtreme Gradient Boosting) is a powerful machine learning algorithm that builds an ensemble of decision trees. Unlike traditional time series methods, it treats forecasting as a supervised learning problem where we predict the target based on engineered features.

**How it works:**
1. **Feature Engineering**: Transform time series into features
   - Lagged values (sales from previous quarters)
   - Rolling statistics (mean, std, min, max)
   - Seasonal indicators (quarter of year)
   - Product characteristics (life cycle stage)

2. **Gradient Boosting**: Build trees sequentially
   - Each tree corrects errors from previous trees
   - Final prediction is sum of all tree predictions

```
Prediction Process:
┌──────────────────┐
│ Time Series Data │
└────────┬─────────┘
         ↓
┌──────────────────┐
│Feature Engineering│
│ - Lag features    │
│ - Rolling stats   │
│ - Season flags    │
└────────┬─────────┘
         ↓
┌──────────────────┐
│  XGBoost Model   │
│  (Ensemble of    │
│   Decision Trees)│
└────────┬─────────┘
         ↓
┌──────────────────┐
│   Prediction     │
└──────────────────┘
```

**Features Used**:
```python
# Feature engineering for XGBoost
features = {
    'lag_1': sales[t-1],           # Previous quarter
    'lag_2': sales[t-2],           # 2 quarters ago
    'lag_4': sales[t-4],           # Same quarter last year
    'rolling_mean_4': mean(sales[t-4:t]),  # 4-quarter average
    'rolling_std_4': std(sales[t-4:t]),    # 4-quarter std
    'quarter': quarter_of_year,     # 1, 2, 3, or 4
    'lifecycle_sustaining': 1 or 0, # Binary encoding
    'lifecycle_decline': 1 or 0,
    'trend': linear_trend_coefficient
}
```

**Implementation**:
```python
import xgboost as xgb
from sklearn.model_selection import TimeSeriesSplit

# Prepare features and target
X = create_features(sales_data)  # Feature matrix
y = sales_data['target']         # Next quarter sales

# Train model
model = xgb.XGBRegressor(
    n_estimators=100,
    max_depth=5,
    learning_rate=0.1,
    objective='reg:squarederror'
)
model.fit(X_train, y_train)

# Predict
xgb_forecast = model.predict(X_test)
```

**Advantages**:
- Captures non-linear relationships
- Handles multiple features naturally
- Robust to outliers
- Can incorporate external factors (promotions, holidays)
- Provides feature importance rankings

**Limitations**:
- Requires more data for training
- Feature engineering is crucial and labor-intensive
- May overfit with limited data
- Less interpretable than statistical methods
- Doesn't extrapolate trends well

**When to use**: Products with complex, non-linear demand patterns, when you have external features to include, or when statistical methods consistently underperform.

---

### 7. Hybrid Model (Ensemble)

**What is it?**
The Hybrid Model combines predictions from multiple forecasting methods to create a more robust and accurate forecast. By leveraging the strengths of different models and averaging out their individual weaknesses, ensemble methods typically outperform any single model.

**Why Ensemble Works:**
- Different models capture different patterns
- Errors from different models often cancel out
- Reduces risk of model selection error
- More stable predictions

**How it works:**
```
┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
│ Simple MA   │   │ Weighted MA │   │ YoY Growth  │   │Holt-Winters │
│ Forecast    │   │ Forecast    │   │ Forecast    │   │ Forecast    │
└──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘
       │                 │                 │                 │
       └────────────────┴────────┬────────┴─────────────────┘
                                 ↓
                    ┌────────────────────────┐
                    │   Weighted Average     │
                    │   Based on Product     │
                    │   Life Cycle           │
                    └───────────┬────────────┘
                                ↓
                    ┌────────────────────────┐
                    │   Final Hybrid         │
                    │   Forecast             │
                    └────────────────────────┘
```

**Formula**:
```
Hybrid_Forecast = w1×MA + w2×WMA + w3×YoY + w4×HW

Where weights (w1, w2, w3, w4) depend on product characteristics
```

**Weighting Strategy by Product Life Cycle**:

| Life Cycle | MA | WMA | YoY | Holt-Winters | Rationale |
|------------|----|----|-----|--------------|-----------|
| **Sustaining** | 30% | 30% | 20% | 20% | Stable products → rely on recent averages |
| **Decline** | 20% | 20% | 40% | 20% | Declining trend → YoY captures the decline pattern |
| **NPI** | 10% | 30% | 10% | 50% | New products → Holt-Winters captures growth trends |

**Implementation**:
```python
def hybrid_forecast(ma, wma, yoy, hw, life_cycle):
    if life_cycle == 'Sustaining':
        weights = [0.3, 0.3, 0.2, 0.2]
    elif life_cycle == 'Decline':
        weights = [0.2, 0.2, 0.4, 0.2]
    else:  # NPI
        weights = [0.1, 0.3, 0.1, 0.5]
    
    forecasts = [ma, wma, yoy, hw]
    return sum(w * f for w, f in zip(weights, forecasts))

# Example for a Sustaining product:
# MA=24,459, WMA=25,202, YoY=17,976, HW=23,500
# Hybrid = 0.3×24,459 + 0.3×25,202 + 0.2×17,976 + 0.2×23,500
# Hybrid = 7,338 + 7,561 + 3,595 + 4,700 = 23,194
```

**Why Different Weights for Different Life Cycles:**

1. **Sustaining Products (MA & WMA emphasized)**:
   - Demand is relatively stable
   - Recent averages are good predictors
   - Less need for complex trend modeling

2. **Decline Products (YoY emphasized)**:
   - Clear declining trend
   - YoY captures the rate of decline
   - Prevents over-forecasting by using trend information

3. **NPI Products (Holt-Winters emphasized)**:
   - Rapid growth phase
   - Need to capture upward trend
   - Limited historical data makes averaging less reliable

**Advantages**:
- More robust than individual models
- Reduces forecasting variance
- Adapts to product characteristics
- Handles model uncertainty

**Limitations**:
- Weight selection can be challenging
- May underperform if one model is clearly superior
- More complex to maintain and explain

**When to use**: Always recommended as the final forecasting approach, especially when dealing with diverse product portfolios with different characteristics.

---

## Forecasting Process

### Step 1: Data Loading and Cleaning
```python
import pandas as pd
import numpy as np

# Load data
data = pd.read_csv('Cisco Forecast League Data Pack.csv')

# Handle missing values
data = data.fillna(method='ffill').fillna(method='bfill')
```

### Step 2: Data Reshaping for Time Series
```python
# Transform wide format to long format
time_cols = [col for col in df.columns if col.startswith('FY')]

products_df = pd.DataFrame()
for idx, row in df.iterrows():
    product_data = {
        'Cost_Rank': row['Cost Rank'],
        'Product_Name': row['Product Name'],
        'Life_Cycle': row['Product Life Cycle']
    }
    for col in time_cols:
        products_df = pd.concat([products_df, pd.DataFrame({
            **product_data,
            'Quarter': col,
            'Units': row[col]
        }, index=[0])], ignore_index=True)
```

### Step 3: Seasonal Decomposition
```python
from statsmodels.tsa.seasonal import seasonal_decompose

# Decompose time series
decomposition = seasonal_decompose(time_series, model='additive', period=4)

# Components
trend = decomposition.trend
seasonal = decomposition.seasonal
residual = decomposition.resid
```

### Step 4: Model Training and Forecasting
```python
# Apply multiple models and combine
forecasts = {}

for name, group in product_groups:
    time_series = group['Units'].values
    
    # Calculate forecasts from each model
    ma_forecast = calculate_ma(time_series)
    wma_forecast = calculate_wma(time_series)
    yoy_forecast = calculate_yoy(time_series)
    hw_forecast = calculate_holt_winters(time_series)
    
    # Combine based on product life cycle
    combined_forecast = weighted_ensemble(
        ma_forecast, wma_forecast, yoy_forecast, hw_forecast,
        life_cycle=group['Life_Cycle'].iloc[0]
    )
    
    forecasts[name[0]] = max(0, round(combined_forecast))
```

### Step 5: Validation and Output
```python
# Save predictions
results_df = pd.DataFrame({
    'Cost_Rank': list(forecasts.keys()),
    'FY25_Q2_Forecast': list(forecasts.values())
})
results_df.to_csv('FY25_Q2_Predictions.csv', index=False)
```

---

## Scenario Examples

### Scenario 1: Sustaining Product (SWITCH Enterprise High)

**Historical Data**:
| Quarter | FY22 Q2 | FY22 Q3 | FY22 Q4 | FY23 Q1 | FY23 Q2 | FY23 Q3 | FY23 Q4 | FY24 Q1 | FY24 Q2 | FY24 Q3 | FY24 Q4 | FY25 Q1 |
|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
| Units | 57,147 | 52,873 | 52,870 | 38,833 | 27,114 | 21,823 | 31,813 | 27,302 | 22,084 | 21,830 | 29,404 | 24,518 |

**Analysis**:
- **Trend**: Declining trend from 57K to 24K over 3 years
- **Seasonality**: Q4 typically shows higher demand than Q2
- **Life Cycle**: Sustaining product

**Calculation**:
```
MA (last 4 quarters) = (22,084 + 21,830 + 29,404 + 24,518) / 4 = 24,459
WMA = (0.1×22,084 + 0.2×21,830 + 0.3×29,404 + 0.4×24,518) / 1.0 = 25,137
YoY (Q2 over Q2) = 24,518 × (22,084/27,114) = 19,973
Holt-Winters = 23,500 (from model)

Combined Forecast (Sustaining):
= 0.3×24,459 + 0.3×25,137 + 0.2×19,973 + 0.2×23,500
= 7,338 + 7,541 + 3,995 + 4,700
= 23,574 units
```

**Predicted FY25 Q2**: ~23,574 units

---

### Scenario 2: Declining Product (ACCESS POINT Mid)

**Historical Data**:
| Quarter | FY22 Q2 | FY22 Q3 | FY22 Q4 | FY23 Q1 | FY23 Q2 | FY23 Q3 | FY23 Q4 | FY24 Q1 | FY24 Q2 | FY24 Q3 | FY24 Q4 | FY25 Q1 |
|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
| Units | 131,048 | 90,440 | 99,442 | 67,630 | 84,349 | 67,670 | 55,237 | 65,237 | 60,000 | 55,000 | 52,000 | 54,959 |

**Analysis**:
- **Trend**: Strong declining trend (~60% reduction)
- **Life Cycle**: Decline phase - product being phased out

**Calculation (Decline weighting)**:
```
YoY Growth Rate = 54,959 / 65,237 = 0.842 (15.8% decline)
Projected = 54,959 × 0.842 = 46,276

Combined Forecast (Decline):
= 0.2×MA + 0.2×WMA + 0.4×YoY + 0.2×HW
= 0.2×55,490 + 0.2×54,200 + 0.4×46,276 + 0.2×52,000
= 11,098 + 10,840 + 18,510 + 10,400
= 50,848 units
```

**Predicted FY25 Q2**: ~50,848 units

---

### Scenario 3: New Product Introduction (NPI)

**Historical Data** (Limited data starting FY22 Q4):
| Quarter | FY22 Q4 | FY23 Q1 | FY23 Q2 | FY23 Q3 | FY23 Q4 | FY24 Q1 | FY24 Q2 | FY24 Q3 | FY24 Q4 | FY25 Q1 |
|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|---------|
| Units | 1,227 | 24,186 | 7,680 | 16,772 | 17,554 | 16,095 | 26,125 | 24,337 | 21,988 | 32,768 |

**Analysis**:
- **Trend**: Strong growth trajectory (from 1.2K to 32.7K)
- **Pattern**: High volatility typical of new products
- **Life Cycle**: NPI - limited historical data

**Calculation (NPI weighting - favors Holt-Winters)**:
```
Holt-Winters captures the growth trend better for new products

Combined Forecast (NPI):
= 0.1×MA + 0.3×WMA + 0.1×YoY + 0.5×HW
= 0.1×26,305 + 0.3×27,500 + 0.1×28,000 + 0.5×35,000
= 2,631 + 8,250 + 2,800 + 17,500
= 31,181 units
```

**Predicted FY25 Q2**: ~31,181 units

---

## Accuracy Evaluation

### Metrics Used

1. **MAPE (Mean Absolute Percentage Error)**:
```
MAPE = (1/n) × Σ|Actual - Forecast| / Actual × 100%
```

2. **RMSE (Root Mean Square Error)**:
```
RMSE = √[(1/n) × Σ(Actual - Forecast)²]
```

3. **Forecast Accuracy**:
```
Accuracy = 1 - |Actual - Forecast| / Actual
```

4. **Forecast Bias**:
```
Bias = (Forecast - Actual) / Actual
```
- Positive bias = Over-forecasting
- Negative bias = Under-forecasting

### Team Performance Comparison

The project compares forecasts from three teams (D, M, S) based on historical accuracy:

| Product | Best Team | Accuracy |
|---------|-----------|----------|
| SWITCH Enterprise High 1 | M | 91.56% |
| ACCESS POINT Mid 1 | D | 95.27% |
| SWITCH Enterprise Mid 1 | S | 93.91% |
| Wireless Controller 1 | D | 92.17% |

---

## Project Structure

```
Cisco-Forecast-League-Competition/
├── README.md                                       # This documentation
├── Methodology.pptx                                # Presentation of methodology
├── Cisco Forecast League Data Pack - Phase 2.csv  # Main input dataset
├── Accuracy Results Calculation.csv               # Accuracy metrics
├── Cisco0/
│   └── Cisco/
│       ├── cisco.ipynb                            # Main forecasting notebook
│       ├── cis.ipynb                              # Alternative analysis
│       ├── dataset.csv                            # Processed dataset
│       ├── forecasted_values.csv                  # Model predictions
│       └── FY25_Q2_Predictions.csv                # Final predictions
└── Cisco1/
    └── Cisco1/
        ├── cis.ipynb                              # SARIMA & decomposition analysis
        ├── Untitled1.ipynb                        # Team comparison analysis
        ├── FY25_Q2_SARIMA_Predictions.csv         # SARIMA predictions
        ├── FY25_Q2_Hybrid_Predictions.csv         # Hybrid model predictions
        └── forecasted_values.csv                  # Ensemble forecasts
```

> **Note**: Some original file names in the repository may vary slightly (e.g., `dataste.csv`, `Methedology.pptx`).

---

## Key Takeaways

1. **Ensemble methods outperform single models** for diverse product portfolios
2. **Product life cycle** is a crucial factor in model selection
3. **Seasonal decomposition** helps identify underlying patterns
4. **Cross-validation** on historical quarters ensures model reliability
5. **Business context** should guide final forecast adjustments

---

## References

- [Statsmodels Documentation](https://www.statsmodels.org/)
- [Holt-Winters Method](https://otexts.com/fpp2/holt-winters.html)
- [SARIMA Tutorial](https://machinelearningmastery.com/sarima-for-time-series-forecasting-in-python/)
- Cisco Forecast League Competition Guidelines

---

## Authors

Participants in the Cisco Forecast League Competition - India

---

*Last Updated: January 2025*
