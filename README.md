# Food Demand Forecasting — Exploratory Data Analysis

## Overview
This repository contains an exploratory data analysis (EDA) of a food delivery/meal-kit demand dataset, completed as part of a data analysis internship. The goal is to understand demand patterns across meal categories, cuisines, pricing, promotions, and fulfilment centers to surface insights that could inform demand forecasting.

## Dataset
- **Source**: [Food Demand Forecasting](https://www.kaggle.com/datasets/kannanaikkal/food-demand-forecasting) (Kaggle, uploaded by kannanaikkal)
- **Files used**: `train.csv`, `fulfilment_center_info.csv`, `meal_info.csv`
- **Structure**: A star-schema join — `train.csv` (fact table of weekly orders) merged with the two dimension tables on `center_id` and `meal_id`
- **Combined size**: 456,548 rows × 15 columns after merging
- Raw CSVs are not included in this repo due to dataset licensing — download them directly from the Kaggle link above to reproduce the analysis.

## What's in this repo
```
├── notebooks/
│   └── food_demand_eda.ipynb        # Full analysis: cleaning, univariate, bivariate/multivariate
├── Data/
│   └── fullfilment_center_info.csv
│   └── meal_info.csv
│   └── train.csv  
├── report/
│   └── Food_Demand_EDA_Report.docx  # Formatted write-up with charts and findings
├── Charts/
│   └── *.png                        # 18 charts referenced in the report
└── README.md
```

## Analysis summary

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

## Key findings
- Promotions nearly **triple** average demand overall
- **Rice Bowl** (~3.7x) and **Sandwich** (~3.6x) show the strongest promotional lift; **Beverages** show the weakest (~1.4x)
- Frequency of a meal being listed is distinct from its per-listing demand performance — high listing frequency doesn't necessarily mean high average orders per listing

## Tools used
- Python (pandas, matplotlib, seaborn) in Jupyter notebooks
- Microsoft Word for the final report

## How to reproduce
1. Download the three CSVs from the [Kaggle dataset](https://www.kaggle.com/datasets/kannanaikkal/food-demand-forecasting) into a local `data/` folder
2. Open `notebooks/food_demand_eda.ipynb` and run all cells
3. Generated charts save to the `images/` folder for reference in the report
