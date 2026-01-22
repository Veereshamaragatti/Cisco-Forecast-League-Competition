# Cisco Forecast League Competition

## Overview

The **Cisco Forecast League Competition (CFL)** is a demand forecasting competition organized by Cisco, where participants compete to predict future sales/booking units for various Cisco hardware products. The competition challenges teams to provide accurate forecasts for Cisco's product portfolio including networking equipment like switches, routers, access points, transceivers, and more.

## Competition Objective

Teams submit forecasted demand units for specific product categories for a target quarter (e.g., FY25 Q2). The forecasts are then compared against actual booking data to determine accuracy and rank teams.

## Repository Contents

### 1. Data Pack (`Cisco Forecast League Data Pack - Phase 2(Data Pack).csv`)

This file contains the **historical data and benchmark forecasts** that teams use to make their predictions:

#### Data Structure:
| Column | Description |
|--------|-------------|
| Cost Rank | Priority ranking based on product cost/importance |
| Product Name | Cisco product category name |
| Product Life Cycle | Product stage (Sustaining, Decline, etc.) |
| FY22 Q2 - FY25 Q1 | Historical actual unit sales by quarter |
| Your Forecast FY25 Q2 | Column where participants enter their forecast |
| Demand Planners Forecast | Cisco's internal demand planning team forecast |
| Marketing Teams Forecast | Cisco's marketing team forecast |
| Statistical and ML Forecast | Machine Learning/Statistical model forecasts |

#### Benchmark Team Accuracy Data:
The data pack also includes historical accuracy and bias metrics for Cisco's internal forecasting teams across previous quarters (FY24 Q3, FY24 Q4, FY25 Q1), helping participants understand:
- **Demand Planning Team's Forecast Accuracy**
- **Marketing Team's Forecast Accuracy**  
- **Statistical/ML Team's Forecast Accuracy**

#### Product Categories Covered:
1. **SWITCH Enterprise** (High 1, High 2, Mid 1, Mid 2, Low)
2. **SWITCH Data Center** (Low 1, Low 2)
3. **ACCESS POINT** (Mid 1, Mid 2)
4. **Wireless Controller** (1, 2)
5. **TRANSCEIVER MODULE**
6. **ROUTER Enterprise**
7. **POWER SUPPLY**
8. **SERVER**
9. **MEMORY**
10. **PROCESSOR**

### 2. Accuracy Results (`Accuracy Results Calculation CFL India(AccuracyCalculation).csv`)

This file contains the **scoring and accuracy calculation template** for evaluating submissions:

#### Key Metrics:
| Metric | Description |
|--------|-------------|
| Cisco Rank | Final ranking of the team |
| Team Name | Participant team identifier |
| PRODUCT_TYPE | Product category being forecasted |
| Forecast Submitted | Team's submitted forecast for the quarter |
| Booking Actuals | Actual units booked (ground truth) |
| PLID level Accuracy | Product Line ID level accuracy percentage |
| Bias | Over/under forecasting indicator (-100% = under-forecast, +100% = over-forecast) |
| Booked Cost | Dollar value of booked units |
| Cost Weight | Product's weight in overall score (based on cost importance) |
| Cost Weighted Accuracy | Accuracy × Cost Weight (contribution to overall score) |
| Overall Accuracy Team | Final weighted accuracy score |

### 3. Methodology Presentation (`Methedology.pptx`)

> **Note**: The filename contains a typo ("Methedology" instead of "Methodology")

A PowerPoint presentation detailing the forecasting methodology, likely containing:
- Competition rules and guidelines
- Scoring methodology
- Data descriptions
- Submission requirements

## How the Competition Works

### 1. Data Collection Phase
- Teams receive historical sales data spanning multiple fiscal years (FY22-FY25)
- Reference forecasts from Cisco's internal teams are provided as benchmarks

### 2. Forecasting Phase
- Participants analyze historical trends, seasonality, and product life cycles
- Teams can use various approaches:
  - **Statistical Methods**: Time series analysis, exponential smoothing, ARIMA
  - **Machine Learning**: Neural networks, ensemble methods, gradient boosting
  - **Hybrid Approaches**: Combining statistical and ML methods
  - **Domain Knowledge**: Understanding product life cycles and market conditions

### 3. Submission Phase
- Teams submit their forecasted units for each product category
- Forecasts are submitted for a specific target quarter (e.g., FY25 Q2)

### 4. Evaluation Phase
Accuracy is calculated using the formula:

```
Accuracy = 1 - |Forecast - Actual| / Actual
```

**Bias** indicates forecast direction:
- **Negative Bias (-X%)**: Under-forecasting (predicted less than actual)
- **Positive Bias (+X%)**: Over-forecasting (predicted more than actual)

### 5. Scoring Phase
The final score uses **Cost-Weighted Accuracy**:
- Higher-cost/more important products carry more weight
- Final score = Σ (Individual Product Accuracy × Cost Weight)

## Key Considerations for Participants

### Product Life Cycle Stages
- **Sustaining**: Mature products with stable demand patterns
- **Decline**: Products being phased out with decreasing demand

### Forecasting Challenges
1. **Seasonality**: Quarterly demand fluctuations
2. **Trend Analysis**: Long-term growth or decline patterns
3. **Product Transitions**: New products replacing old ones
4. **Market Dynamics**: External factors affecting demand

### Success Factors
1. Understand historical patterns for each product category
2. Consider product life cycle stage in your forecasts
3. Balance accuracy across all products (cost-weighted scoring)
4. Avoid systematic bias (over/under forecasting)
5. Benchmark against provided team forecasts

## Models Used in This Repository

The Jupyter notebooks in the `Cisco0/` and `Cisco1/` folders contain implementations of various forecasting models:

### 1. SARIMA/ARIMA (Seasonal AutoRegressive Integrated Moving Average)
**Why Used**: ARIMA models are well-suited for time series data with trends and seasonality, which is common in quarterly sales data.

```python
sarima_model = sm.tsa.statespace.SARIMAX(
    product_data["Sales"], 
    order=(1, 1, 1), 
    seasonal_order=(1, 1, 1, 4),  # Quarterly seasonality (4 periods)
    enforce_stationarity=False, 
    enforce_invertibility=False
).fit()
```

**Strengths**:
- Captures quarterly seasonality patterns
- Works well with limited historical data
- Handles trends effectively

**Performance**: Used as a baseline model and for products with clear seasonal patterns.

### 2. XGBoost (Extreme Gradient Boosting)
**Why Used**: XGBoost excels at capturing complex non-linear relationships and can incorporate multiple features including team forecasts and historical data.

```python
xgb_model = XGBRegressor(
    objective="reg:squarederror", 
    n_estimators=100, 
    learning_rate=0.1, 
    random_state=42
)
```

**Strengths**:
- Can use multiple features (past sales, team forecasts, trends)
- Handles non-linear patterns
- Robust to outliers

**Performance**: Generally provides good accuracy when combined with feature engineering (rolling means, time indices).

### 3. Holt-Winters Exponential Smoothing
**Why Used**: Excellent for data with both trend and seasonal components, which is typical for product demand forecasting.

```python
model = ExponentialSmoothing(
    filled_series, 
    trend='add',
    seasonal='add', 
    seasonal_periods=4
).fit()
```

**Strengths**:
- Automatically handles trend and seasonality
- Good for short-term forecasts
- Simple to implement

### 4. Prophet (Facebook's Forecasting Library)
**Why Used**: Prophet is designed for business time series with strong seasonal effects and multiple seasons of historical data.

**Strengths**:
- Handles missing data well
- Robust to outliers and shifts in trends
- Easy to tune with domain knowledge

### 5. Simple & Weighted Moving Averages
**Why Used**: Provides stable baseline forecasts, especially for products with limited data or erratic patterns.

```python
# Simple Moving Average (last 4 quarters)
ma_forecast = np.mean(valid_values[-4:])

# Weighted Moving Average (higher weight to recent quarters)
weights = np.array([0.1, 0.2, 0.3, 0.4])
wma_forecast = np.sum(weights * valid_values[-4:]) / np.sum(weights)
```

### 6. Linear Regression
**Why Used**: For products with clear linear trends, simple regression can be effective.

### 7. SVR (Support Vector Regression)
**Why Used**: SVR can capture non-linear relationships while being robust to outliers.

---

## Model Comparison & Accuracy Analysis

The notebooks compare model performance using metrics like:
- **MAE (Mean Absolute Error)**
- **RMSE (Root Mean Square Error)**

### Which Model Gave More Accuracy?

Based on the code analysis, the repository uses a **Hybrid Ensemble Approach** that combines multiple models:

| Model | Best For | Typical Accuracy |
|-------|----------|------------------|
| **SARIMA** | Products with clear seasonality | Good for baseline |
| **XGBoost** | Products with complex patterns | Often best performer |
| **Holt-Winters** | Sustaining products | Competitive accuracy |
| **Hybrid** | Overall | Best combined accuracy |

**Key Findings**:

1. **XGBoost tends to perform best** when combined with trend features (time index, rolling means) because it can capture complex relationships between multiple features including the internal team forecasts.

2. **SARIMA excels for products with clear quarterly seasonality** but may struggle with products that have erratic demand patterns.

3. **The Hybrid approach** (combining ARIMA + XGBoost predictions) often achieves the lowest RMSE by leveraging the strengths of both statistical and ML methods:
   ```python
   hybrid_pred = 0.5 * arima_forecast + 0.5 * xgb_forecast
   ```

4. **Product Life Cycle matters**: The code applies different model weights based on product stage:
   - **Sustaining products**: Higher weight on Moving Averages (more stable)
   - **Declining products**: Higher weight on Year-over-Year trends (captures decline)
   - **New products**: Higher weight on Holt-Winters/Exponential Smoothing

### Best Team Selection Logic
The notebooks also implement logic to select the best-performing internal team (D=Demand Planning, M=Marketing, S=Statistical/ML) based on historical bias:

```python
for team in ['D', 'M', 'S']:
    team_errors = np.mean([abs(row[f"{team}_FY2024_Q3_BIAS"]), 
                           abs(row[f"{team}_FY2024_Q4_BIAS"]), 
                           abs(row[f"{team}_FY2025_Q1_BIAS"])])
best_team = min(team_errors, key=team_errors.get)
```

This allows using the most accurate team's forecast for each product when team benchmarks are available.

---

## File Structure

```
Cisco-Forecast-League-Competition/
├── README.md                                                    # This documentation
├── Cisco Forecast League Data Pack - Phase 2(Data Pack).csv    # Historical data & benchmarks
├── Accuracy Results Calculation CFL India(AccuracyCalculation).csv  # Scoring template
├── Methedology.pptx                                             # Competition methodology (note: filename typo)
├── Cisco0/Cisco/                                                # Jupyter notebooks with model implementations
│   ├── cisco.ipynb                                              # Holt-Winters, Moving Average, YoY models
│   ├── cis.ipynb                                                # SARIMA, XGBoost, Prophet, Hybrid models
│   ├── Untitled.ipynb - Untitled2.ipynb                         # Additional experiments
└── Cisco1/Cisco1/                                               # Additional model implementations
    ├── cis.ipynb                                                # SARIMA implementation
    └── Untitled1.ipynb                                          # Team accuracy analysis
```

## Getting Started

1. **Review the Methodology**: Start with `Methedology.pptx` to understand competition rules
2. **Analyze Historical Data**: Study the Data Pack CSV for trends and patterns
3. **Build Your Model**: Develop forecasting models using historical data
4. **Validate Against Benchmarks**: Compare your forecasts with internal team benchmarks
5. **Submit Forecasts**: Fill in the "Your Forecast" column for each product
6. **Review Accuracy**: Use the Accuracy Calculation template to evaluate your submission

## Competition Insights

Based on the benchmark data, Cisco's internal forecasting teams achieve:
- **Demand Planning Team**: Generally 70-98% accuracy depending on product
- **Marketing Team**: Variable performance (0-98% accuracy)
- **Statistical/ML Team**: Typically 51-98% accuracy

This suggests that a well-tuned forecasting approach can compete with or exceed Cisco's internal methods.

## License

This repository contains competition data provided by Cisco for the Forecast League Competition.

---

*For more information about the Cisco Forecast League Competition, please refer to the official Cisco competition guidelines.*
