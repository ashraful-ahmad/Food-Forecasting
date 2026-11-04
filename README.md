# Food Demand Forecasting
 
## Overview
This repository documents a five-week data analysis internship project (Yuva Intern) on a food delivery/meal-kit demand dataset. The project covers the full workflow: exploratory analysis, preprocessing, feature engineering, model building, and model optimization and validation, with the goal of forecasting weekly order demand across meal categories, cuisines, pricing, promotions, and fulfilment centers.
 
**Final result:** a tuned HistGradientBoosting model reaching an RMSLE of **0.4786** on 10 unseen hold-out weeks (baseline: 0.4823).
 
## Dataset
- **Source**: [Food Demand Forecasting](https://www.kaggle.com/datasets/kannanaikkal/food-demand-forecasting) (Kaggle, uploaded by kannanaikkal)
- **Files used**: `train.csv`, `fulfilment_center_info.csv`, `meal_info.csv`
- **Structure**: A star-schema join — `train.csv` (fact table of weekly orders) merged with the two dimension tables on `center_id` and `meal_id`
- **Combined size**: 456,548 rows × 15 columns after merging
## Project Timeline
 
| Week | Focus | Status |
|---|---|---|
| 1 | Exploratory Data Analysis | Complete |
| 2 | Data Preprocessing | Complete |
| 3 | Feature Engineering | Complete |
| 4 | Model Building | Complete |
| 5 | Model Optimization and Validation | Complete |
 
## What's in this repo
```
├── Notebooks/
│   ├── EDA.ipynb                              # Weeks 1-2: EDA + preprocessing
│   ├── Week3_Feature_Engineering.ipynb        # Week 3: feature engineering
│   ├── Week4_Model_Building.ipynb             # Week 4: model building
│   └── Week5_Model_Optimization.ipynb         # Week 5: tuning and validation
├── Data/
│   ├── fulfilment_center_info.csv
│   ├── meal_info.csv
│   └── train.csv
├── Report/
│   ├── Food_Demand_EDA_Report.docx            # Week 1: EDA write-up
│   ├── Week2_Preprocessing_Report.docx        # Week 2: Preprocessing write-up
│   ├── Week3_Feature_Engineering_Report.docx  # Week 3: Feature engineering write-up
│   ├── Week4_Model_Building_Report.docx       # Week 4: Model building write-up
│   └── Week5_Model_Optimization_Report.docx   # Week 5: Optimization and validation write-up
├── Charts/
│   └── *.png                                  # Charts referenced in the reports
└── README.md
```
 
Note: the generated datasets (`preprocessed_food_demand.csv`, 456,548 rows × 42 columns, and `feature_engineered_food_demand.csv`, 456,548 rows × 49 columns) are produced by running the notebooks rather than stored in the repo — they are large and easily reproduced by running the notebooks in order.
 
---
 
## Week 1: Exploratory Data Analysis
 
**Data cleaning**
- No missing values and no duplicate rows found
- Flagged data quality issue: ~25% of rows show a negative discount (checkout price exceeds base price), concentrated at -1.00 and -2.00 values, spread evenly across the dataset
**Univariate analysis**
- Distribution checks on `num_orders`, `checkout_price`, `base_price`
- Category breakdowns for `category`, `cuisine`, `center_type`, `region_code`
- Outlier detection on `num_orders` via IQR method (~7.2% of rows flagged)
**Bivariate & multivariate analysis**
- Price vs. demand relationship
- Average orders by category
- Promotion impact on demand
- Correlation heatmap across numeric features
- Category × promotion and region × promotion interactions
- Discount vs. orders, colored by category
**Key findings**
- Promotions nearly **triple** average demand overall
- **Rice Bowl** (~3.7x) and **Sandwich** (~3.6x) show the strongest promotional lift; **Beverages** show the weakest (~1.4x)
- Frequency of a meal being listed is distinct from its per-listing demand performance — high listing frequency doesn't necessarily mean high average orders per listing
## Week 2: Data Preprocessing
 
Preprocessing addressed every data quality issue surfaced in Week 1 EDA, prioritizing transformation and flagging over deletion — the row count is unchanged (456,548) throughout.
 
**Data cleaning**
- Re-confirmed 0 missing values, 0 duplicates after merge
- `num_orders` outliers (7.2%, right-skewed, skewness ≈ 6.93): identified as genuine promotion-driven demand spikes rather than errors, so treated with a `log1p` transform (`num_orders_log`) instead of capping/removal — resulting skewness ≈ -0.02
- Negative discount rows (~25%, 116,101 rows): flagged via a new `has_negative_discount` boolean column rather than removed, since no evidence confirmed they were errors
**Data transformation**
- `StandardScaler` applied to `checkout_price`, `base_price`, `op_area` (mean 0, std 1)
- One-hot encoding applied to `category`, `cuisine`, `center_type`, and `region_code` (treated as nominal despite being numeric)
**Data reduction**
- Dropped non-predictive ID columns: `id`, `center_id`, `meal_id`, `city_code`
**Result**: 456,548 rows × 42 columns (up from 15), saved as `preprocessed_food_demand.csv`
 
## Week 3: Feature Engineering
 
**Input:** `preprocessed_food_demand.csv` → **Output:** `feature_engineered_food_demand.csv` (456,548 rows × 49 columns)
 
Seven features were added across four categories:
 
| Category | Features |
|---|---|
| Time-based | `orders_last_week_log`, `rolling_avg_3wk_log` |
| Price-based | `discount_pct`, `price_vs_category_avg` |
| Interaction | `discount_x_promo` |
| Aggregation | `center_avg_orders_log`, `meal_avg_orders_log` |
 
**Key decisions**
- Feature code runs before the ID-drop step so `center_id` and `meal_id` can be used as grouping keys, then they are dropped before saving.
- Boolean columns (from `pd.get_dummies()` and `has_negative_discount`) are converted to 0/1 in the Week 3 output only; `preprocessed_food_demand.csv` is left unchanged.
- `discount_x_promo` was motivated by the Week 1 finding that promotional lift varies by category (Rice Bowl ~3.7x, Sandwich ~3.6x, Beverages ~1.4x).
**Known limitation:** the mean-encoded aggregation features were computed on the full dataset, which is a data leakage risk. This is documented in the Week 3 report and addressed with a leak-free check in Week 5.
 
## Week 4: Model Building
 
**Setup**
- **Split:** time-based. Train on weeks 1–135, test on weeks 136–145 (32,821 rows), so the model is always evaluated on future weeks.
- **Target:** `num_orders_log`
- **Excluded from features:** `num_orders` and `num_orders_log` (target leakage) and `week`. Dropping `week` improved HistGradientBoosting RMSLE from 0.5084 to 0.4823.
- **Scaling:** applied only inside the Ridge pipeline (continuous columns, fitted on training weeks). Tree models are unscaled.
**Result:** best model is HistGradientBoosting with **test RMSLE 0.4823**.
 
**What drives the model**
- Top features by permutation importance: `meal_avg_orders_log`, `rolling_avg_3wk_log`, `orders_last_week_log`, `center_avg_orders_log`.
- Feature-group ablation (RMSLE from 0.4823): removing price & promotion features → 0.4980, recent-history features → 0.5159, mean-encoded features → 0.4943. Every group contributes.
**Negative discounts and outliers**
- Negative discounts: MAE (log) is 0.3605 on the 8,999 negative-discount test rows vs 0.3761 on the rest, so the model handles them no worse. Removing `has_negative_discount` or clipping negative discounts to 0 left RMSLE unchanged (0.4823 / 0.4822).
- Outliers: capping the training target at the 1st/99th percentile gave 0.4822, essentially no change. An IQR rule on the log target flags only 0.08% of training rows and none in the test set, confirming that the `log1p` transform already tamed the extreme values.
**Known behaviour:** the model over-predicts very low demand and under-predicts demand spikes.
 
**Multicollinearity note:** the engineered file contains scaled copies of some columns (`checkout_price_scaled`, `base_price_scaled`, `op_area_scaled`, correlation 1.0 with the raw columns) and other highly correlated pairs (e.g. `checkout_price` vs `base_price` 0.95, `orders_last_week_log` vs `rolling_avg_3wk_log` 0.93). This does not hurt tree-based models but matters for the linear model.
 
Charts are saved in the `Charts/` folder.
 
## Week 5: Model Optimization and Validation
 
**Approach**
- **Same hold-out as Week 4:** weeks 1–135 for training and tuning, weeks 136–145 kept untouched for final evaluation (baseline RMSLE 0.4823).
- **Cross-validation:** week-based expanding-window CV, so each fold trains on earlier weeks and validates on later ones, with no future data leaking into training.
- **Hyperparameter tuning:** `RandomizedSearchCV` on HistGradientBoosting.
- **Leakage check:** a leak-free version of the mean-encoded features, addressing the Week 3 limitation.
**Best parameters:** `learning_rate=0.05`, `max_leaf_nodes=127`, `max_depth=12`, `min_samples_leaf=20`, `l2_regularization=1.0`
 
**Hold-out results (weeks 136–145)**
 
| Metric | Baseline | Tuned |
|---|---|---|
| RMSLE | 0.4823 | **0.4786** |
| MAE (log) | 0.3718 | **0.3680** |
| R² (log) | 0.8239 | **0.8265** |
| MAE (orders) | 62.73 | **61.85** |
 
Tuning improved every metric, with RMSLE down by 0.0037. The gain is modest, which suggests the Week 3 features, rather than hyperparameters, were the main driver of performance.
 
## Tools used
- Python (pandas, scikit-learn, matplotlib, seaborn) in Jupyter notebooks
- Git / GitHub for version control
- Microsoft Word for report write-ups
## How to reproduce
1. The three CSVs are included in `Data/`; alternatively download fresh from the [Kaggle dataset](https://www.kaggle.com/datasets/kannanaikkal/food-demand-forecasting)
2. Run the notebooks in order: `EDA.ipynb` → Week 3 → Week 4 → Week 5, running all cells top to bottom in each
3. Generated charts save to the `Charts/` folder; intermediate datasets (`preprocessed_food_demand.csv`, `feature_engineered_food_demand.csv`) save to the repo root
